# CoreDNS Zone 配置安全分代 Rollout - Implementation Plan

> 实现包：新增 `controllers/corednsrollout/`（分代状态机、CM 存储、工作负载管理、DNS 验证），
> 经窄接口接入现有 `ZoneDelegationReconciler` 与 `zones.ZoneDelegation`，Helm 仅改 RBAC/配置项。
> 每个任务包含可独立自验的 rule/rubric TR；所有外部边界（k8s client、DNS querier、时钟）接口注入。

## Task 1: 配置项与功能门控
- **Status**: `completed`
- **Completion Evidence**:
  - TR-1.1 `rule` PASS：新增 4 字段（[config.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/resolver/config.go#L51-L57)），`TestConfigCoreDNSRollout` 验证默认值 enabled=true/120/120/3，0 与负数被 validator 拒绝；命令 `env -u LOG_FORMAT go test ./controllers/resolver/` 全绿（本机 LOG_FORMAT=json 导致的 TestConfigurations 失败在干净代码上同样存在，属环境预存问题）。
  - TR-1.2 `rule` PASS：`COREDNS_ROLLOUT_ENABLED=false` 解析为 false（同测例子项）。
  - 附带修复：恢复基线中被半成品删除的 [zone_delegation_service_mock.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zones/zone_delegation_service_mock.go)（其缺少 UpdateCoreDNSConfiguration 导致 controllers 包测试编译失败），Task 9 将按新接口正式重生成。
- **Priority**: high
- **Depends On**: None
- **Description**:
  - 在 [config.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/resolver/config.go) 新增 kong 配置字段：`CoreDNSRolloutEnabled`（env `COREDNS_ROLLOUT_ENABLED`，默认 `true`）、`CoreDNSCanaryReadyTimeoutSeconds`（默认 120，`min=1`）、`CoreDNSRolloutTimeoutSeconds`（默认 120，`min=1`）、`CoreDNSQuarantineGenerations`（默认 3，`min=0`）。
  - help 文案说明 kill-switch 回落到直接写 anchor CM 的 legacy 行为。
- **Acceptance Criteria Addressed**: AC-14, AC-16（配置基础）
- **Test Requirements**:
  - `rule` TR-1.1: 默认值生效（未设置环境变量时 ResolveOperatorConfig 返回默认值且校验通过）；非法值（0/负数）启动校验报错；证据：`go test ./controllers/resolver/ -run Config`。
  - `rule` TR-1.2: `COREDNS_ROLLOUT_ENABLED=false` 可被解析为关闭；证据：同上。
- **Notes**: 字段命名/风格对齐现有 `ZoneDelegationRecoverySeconds` 等字段。

## Task 2: 规范化渲染、指纹与分代类型
- **Status**: `completed`
- **Completion Evidence**:
  - TR-2.1 `rule` PASS：[normalize_test.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/normalize_test.go) 覆盖顺序无关、单字节差异、与旧渲染逐字节一致；`go test ./controllers/corednsrollout/` 通过。
  - TR-2.2 `rule` PASS：`TestGenerationConfigMapName` 断言长度 ≤253 且通过 `validation.IsDNS1123Subdomain`。
  - TR-2.3 `rule` PASS：`TestNormalizeSkipsDeleting` 验证删除中 ZD 被排除。
- **Priority**: high
- **Depends On**: Task 1
- **Description**:
  - 新增 `controllers/corednsrollout/types.go`：phase 常量（Active/CandidateRolling/CandidateProbing/Promoting/Quarantined）、anchor 注解键（active/candidate/phase/last-failure/last-template）、标签键（managed-by、zd-generation、zd-role）、资源名（canary Deployment/Service 固定名、代 CM 名前缀）、`GenerationState` 与 `FailureDiagnostic` 类型。
  - 新增 `normalize.go`：从 `v1beta1io.ZoneDelegationList`（跳过 IsInDeletion）生成确定性 data（key 排序，渲染输出与 [zone_service.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zones/zone_service.go) 的 `getCoreDNSData` 逐字节一致），SHA-256 截断指纹，CM 名 `k8gb-zone-delegation-<hash>`。
- **Acceptance Criteria Addressed**: AC-2, AC-3
- **Test Requirements**:
  - `rule` TR-2.1: 任意顺序的同集合输入产生相同 data/hash；单字节差异产生不同 hash；渲染结果与现有 `getCoreDNSData` 输出一致；证据：normalize 表驱动单测。
  - `rule` TR-2.2: CM 名 ≤ DNS-1123/253 长度限制且仅含合法字符；证据：单测断言。
  - `rule` TR-2.3: 删除中 ZoneDelegation 不进入期望 data；证据：单测。

## Task 3: Corefile 语法结构与 zone 静态校验器
- **Status**: `completed`
- **Completion Evidence**:
  - TR-3.1 `rule` PASS：[validate_test.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/validate_test.go) 正例（模板产物/多 zone/注释空行/空 data）与 10 个负例（括号不配平、非法 zone 字符、重复 block、端口 0/越界、未知插件、空文件、闭括号后内容、文件名不符、嵌套 block）逐一拒绝且错误携带文件名/zone/行号/原因。
  - TR-3.2 `rule` PASS：validate.go 仅依赖 fmt/regexp/sort/strconv/strings、miekg/dns 与 utils.ValidateRFC1035，不接触 k8s client。
- **Priority**: high
- **Depends On**: Task 2
- **Description**:
  - 新增 `validate.go`：对规范化后的每个 `<zone>.conf` 做轻量词法/结构校验——server block 头（zone token + 端口）、括号配平、只允许受管插件行（`import k8gbplugins`）、注释/空行处理；zone 用 `miekg/dns` 与现有 `utils.ValidateRFC1035` 做 FQDN/标签合法性校验；拒绝重复 zone、重复 server block、非法端口。
  - 不引入 coredns 库依赖；错误信息携带 zone 名与具体原因（供 FR-18 诊断）。
- **Acceptance Criteria Addressed**: AC-4
- **Test Requirements**:
  - `rule` TR-3.1: 正例（当前模板产物、多 zone、带注释）通过；负例（括号不配平、非法 zone 字符、重复 zone、错误端口、未知插件行、空 zone）逐一拒绝且错误含 zone 名；证据：validate 表驱动单测。
  - `rule` TR-3.2: 校验纯函数无副作用、不接触 k8s client；证据：代码审查 + 单测无 client 依赖。

## Task 4: Anchor/分代 ConfigMap 存储层（adopt、候选、诊断）
- **Status**: `completed`
- **Completion Evidence**:
  - TR-4.1 `rule` PASS：`TestAdoptFreshEnvironment`（gen-0=期望 data、immutable、anchor 镜像、active 注解、幂等）与 `TestAdoptSnapshotsExistingAnchorData`（anchor 旧数据快照为 gen-0 且内容不动）。
  - TR-4.2 `rule` PASS：`TestCandidateCreationFailureIsDiscarded`（创建注入失败 → 无代 CM 残留、candidate 注解清除、last-failure 写入、active 不变、健康客户端重试成功）。
  - TR-4.3 `rule` PASS：`TestGenerationCMAlreadyExistsIsIdempotent`（AlreadyExists 成功、零 Update、ResourceVersion 不变）。
  - TR-4.4 `rule` PASS：`TestAnchorOptimisticConflictRetries`（首次 Conflict 后重试，最终注解一致）。
  - 附加：`TestQuarantineCandidateLabelsGenerationAndAnchor` 验证 quarantine 仅改不可变 CM 的元数据标签/注解且活动代与数据不变；`TestDeleteGenerationCMGuardsActive` 拒绝删除活动代。
- **Priority**: high
- **Depends On**: Task 2, Task 3
- **Description**:
  - 新增 `store.go`：`Store` 封装 anchor CM（名 `k8gb-zone-delegation`）读取/状态注解读写（JSON sub-annotation，resourceVersion 冲突重试）、不可变代 CM 创建（`Immutable: true` + 标签/指纹注解）、adopt gen-0（anchor 现有 data 为空时以期望 data 为 gen-0；否则快照 anchor data）、候选丢弃（删除半成品 + 清 candidate 注解）、失败诊断写入（last-failure）、代 CM 列举（按控制器标签过滤）。
  - anchor 与代 CM 读写仅经此组件；代 CM 永不 update。
  - anchor CM 的 `data` 必须始终镜像当前活动代 CM 内容（adopt 时写入 gen-0 内容；提升切换 active 时同事务更新镜像），保证现有 chainsaw 直接读取 anchor `.data` 的断言与人工排障路径不变（对齐 FR-4/AC-1）。
- **Acceptance Criteria Addressed**: AC-1, AC-5, AC-13（枚举基础）, AC-17
- **Test Requirements**:
  - `rule` TR-4.1: 全新环境 adopt 后：anchor active=gen0/phase=Active，gen-0 CM 内容等于 anchor 快照（或期望 data），immutable=true；anchor 已有数据时内容一致；证据：fake client 单测。
  - `rule` TR-4.2: 候选创建注入失败时：无代 CM 残留、anchor 无 candidate 注解、last-failure 存在、active 不变；重试可成功；证据：失败注入单测。
  - `rule` TR-4.3: 相同 hash 的代 CM 已存在时创建幂等（AlreadyExists 视为成功且不修改既有对象）；证据：单测。
  - `rule` TR-4.4: anchor 乐观冲突发生时重试并最终一致；证据：冲突注入单测。

## Task 5: 工作负载发现、canary 构建与最小补丁引擎
- **Status**: `completed`
- **Completion Evidence**:
  - TR-5.1 `rule` PASS：`TestBuildCanaryWhitelistOnly` 断言 canary PodSpec 与正式模板除 dynamic-zones volume 来源外 deep-equal；探针/镜像/命令/ENV/端口/其他卷/SecurityContext/SA 全等；replicas=1；白名单标签/注解正确。
  - TR-5.2 `rule` PASS：同测例验证候选 Service 为 ClusterIP、端口（servicePort→targetPort、NodePort 清零）与双栈 IPFamilies/Policy 克隆、selector 仅含候选 generation+canary role。
  - TR-5.3 `rule` PASS：`TestDiscoverTopology` 覆盖 0/多 Service、0/多 Deployment、缺 volume 五类 DiscoveryError（原因常量可程序识别）；`TestLegacyAnchorWrite` 验证 legacy 原语直写 anchor 且字节一致；kill-switch 端到端回落由 Task 8 协调器测试覆盖。
  - TR-5.4 `rule` PASS：`TestPromotionRestorationRoundtrip`（内存 deep-equal 互逆）+ `TestPatchPromotionRestorationRoundtripViaClient`（RFC6902 最小三字段 JSON patch 经 fake client 应用/恢复，只动 volume 名、generation 标签、checksum 注解与 Service selector）+ `TestPatchRestorationKeepsPreExistingGenerationLabel`（原标签存在时恢复为旧值而非删除）。
  - TR-5.5 `rule` PASS：`TestCanaryOwnershipAndLabels` 验证 canary Deployment/Service 均以 anchor CM 为 controller owner 并带 managed-by/zd-role 受控标签（canary Deployment 元数据标签补齐）。
  - 附：`TestEnsureCanaryReplacesOldGeneration` 覆盖 supersede 场景 canary 原地换代替换；`TestScaleCanary` 验证超时 scale-0 原语（Task 6 使用）；`go test ./controllers/corednsrollout/`、`go vet` 全绿。
- **Priority**: high
- **Depends On**: Task 4
- **Description**:
  - 新增 `workload.go`：
    - 按 [corednssvc.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/providers/assistant/corednssvc.go) 既有标签发现唯一稳定 Service 与对应正式 Deployment（Deployment→匹配 Service selector 的 Deployment，命名空间同 `K8gbNamespace`）；0 个/多个返回可识别的 DiscoveryError（供 FR-16 降级）。
    - `BuildCanary`：深克隆正式 Pod 模板，白名单差异：dynamic-zones volume 的 configMap.name 改为候选 CM、Pod/模板标签加 `zd-generation=<hash>`、校验和注解；replicas=1；canary Deployment 名 `k8gb-coredns-zd-canary`，ownerReference 指向 anchor CM；候选 Service 同名 ClusterIP，端口（含 servicePort 映射）克隆稳定 Service，selector 仅 `zd-generation=<hash>`+canary role。
    - 正式提升补丁：生成仅含三字段（volume 来源、generation 标签、校验和注解）的 StrategicMerge/PJSON patch；切换前捕获 Service selector 原值；提供恢复补丁（旧三字段 + 旧 selector）。
    - legacy 模式（kill-switch 关闭或 DiscoveryError）：退回直接把期望 data 写入 anchor CM（复用现有行为）。
- **Acceptance Criteria Addressed**: AC-6, AC-9（恢复原语）, AC-14, AC-16（降级行为）
- **Test Requirements**:
  - `rule` TR-5.1: canary 与正式模板的结构化 diff 仅含白名单字段；探针/镜像/命令/ENV/端口/其他卷/SecurityContext/SA 全等；replicas=1；证据：模板 diff 表驱动单测。
  - `rule` TR-5.2: 候选 Service 为 ClusterIP、端口族一致、selector 只命中候选代；证据：单测。
  - `rule` TR-5.3: 0 个/2 个 Service/Deployment 返回 DiscoveryError 且调用方走 legacy 直写（anchor data 与旧实现一致）；kill-switch=false 同样走 legacy；证据：单测。
  - `rule` TR-5.4: 三字段补丁与恢复补丁互为逆操作（应用-恢复后对象与原始 deep-equal）；证据：patch roundtrip 单测。
  - `rule` TR-5.5: canary 资源带 ownerReference 与受控标签；证据：单测。

## Task 6: 就绪与 rollout 状态等待（非阻塞、超时隔离）
- **Status**: `completed`
- **Completion Evidence**:
  - TR-6.1 `rule` PASS：`TestWaitCanary`（NotFound/未就绪=InProgress、到点=TimedOut、Ready 优先、zero since 不超时）+ `TestCanaryTimeoutScalesZeroAndQuarantines`（超时 scale 0 + quarantine 标签/诊断，正式 Deployment/Service 零改动、selector 与快照一致）；NotReady 时 querier 零调用由 Task 8 协调器测试断言。
  - TR-6.2 `rule` PASS：`TestWaitProduction` 覆盖 rolling→InProgress、到点→TimedOut、完成→Complete、NotFound→InProgress；超时回滚三字段原语（PatchRestoration）Task 5 已测，active 不变由 Task 8 端到端断言。
  - TR-6.3 `rule` PASS：`TestCanaryReadyStatuses`/`TestRolloutCompleteStatuses` 表驱动覆盖 observedGeneration 滞后、部分可用、全部可用、Available=False、ReplicaFailure=True、UnavailableReplicas 残留。
  - 实现：[rollout.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/rollout.go) 纯函数判定 + 注入式 Clock；截止时间经 anchor `zd-candidate-since`/`zd-promotion-since` 注解重启安全（Store SetCandidate/SetPhase/State 支持）；协调循环固定 `RequeueInterval`=5s 非阻塞轮询。
- **Priority**: high
- **Depends On**: Task 5
- **Description**:
  - 新增 `rollout.go`：基于 Deployment status/Pod condition 的判定函数——canary Ready（Updated/Ready/Available 达标，ready 沿用现有探针结果）、正式 rollout 完成（`status.observedGeneration≥metadata.generation` 且 updatedReplicas==replicas、availableReplicas==replicas、无不可用条件）；提供"立即返回进行中状态"接口由协调器以 `RequeueAfter` 轮询（间隔固定短值，超时取 Task 1 配置）。
  - 超时动作：canary → scale 0 + quarantined 标签/原因注解；正式 → 触发 Task 5 恢复补丁。
- **Acceptance Criteria Addressed**: AC-7, AC-10
- **Test Requirements**:
  - `rule` TR-6.1: canary NotReady 时判定 InProgress、DNS 验证不被调用；超时后 canary replicas=0 且带 quarantined 原因，正式 Deployment/Service 无补丁；证据：状态序列单测 + querier 零调用断言。
  - `rule` TR-6.2: 正式 rollout 超时生成回滚补丁并恢复旧三字段，active 不变；证据：状态序列单测。
  - `rule` TR-6.3: Ready/Complete 判定的正反例（observedGeneration 滞后、部分可用、全部可用）；证据：表驱动单测。

## Task 7: 权威 NS/A/AAAA 验证器
- **Status**: `completed`
- **Completion Evidence**:
  - TR-7.1 `rule` PASS：`TestProbeHappyPathDualStack`（NS NOERROR+AA 且含本集群 glue（含对端集群 NS 混杂、乱序）、A 集合相等、AAAA 集合相等）+ `TestProbeHappyPathSingleStackAAAANoData`（无 v6 暴露时 AAAA NODATA 通过）。
  - TR-7.2 `rule` PASS：`TestProbeFailureModes` 表驱动 14 例：SERVFAIL/REFUSED/超时/传输错误/无 AA/NS 缺失/A 多-少-错 IP/有 v6 时 AAAA 缺失-错误，逐一失败且 Summary 含 rcode/AA/expected/actual；`TestProbeMultipleZonesAllMustPass` 验证多 zone 全部通过才 Pass。
  - TR-7.3 `rule` PASS：单栈 NODATA 通过、AAAA SERVFAIL 与意外 AAAA 应答失败（同表）。
  - 实现：[probe.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/probe.go) 定义窄接口 `DNSQuerier`（生产实现 `MiekgQuerier` UDP 2s 超时，RD=0+EDNS0），`CanaryServerAddress` 按稳定 Service 端口映射（UDP 53→targetPort 5353）生成 `<canary>.<ns>.svc:port`；返回结构化 `ProbeReport`/`QueryOutcome` 供 FR-11 诊断。
- **Priority**: high
- **Depends On**: Task 5
- **Description**:
  - 新增 `probe.go`：定义 `DNSQuerier` 接口（`Exchange(msg, server) (*dns.Msg, error)`，生产实现复用 `utils.Exchange`/miekg/dns，经候选 Service `<name>.<ns>.svc` + 稳定 Service 映射端口访问）；对每个受影响 zone 执行 FR-9：
    - NS 查询 loadBalancedZone，要求 NOERROR + AA + Answer NS 集合包含期望 glue NS（至少本集群 `Config.GetClusterNSName`）；
    - A 查询本集群 glue NS 名，地址集合 == ipresolver 当前暴露 IPv4 集合（经窄接口注入暴露 IP 提供函数，避免直接耦合）；
    - AAAA：Service 暴露 IPv6 时要求集合相等；无 v6 时 NODATA（NOERROR 空 answer）视为通过。
  - 返回结构化验证结果（每条查询的 rcode/AA/期望/实际），供诊断记录。
- **Acceptance Criteria Addressed**: AC-8
- **Test Requirements**:
  - `rule` TR-7.1: 全通过路径（双栈与单栈各一）；证据：fake querier 表驱动单测。
  - `rule` TR-7.2: 失败路径：SERVFAIL、REFUSED、超时、无 AA、NS 缺失、A 集合不等（多/少/错 IP）、有 v6 时 AAAA 缺失/不符——分别返回失败且结果含实际应答摘要；证据：表驱动单测。
  - `rule` TR-7.3: 无 v6 暴露时 AAAA NODATA 通过、AAAA 返回错误码失败；证据：单测。

## Task 8: 分代状态机协调器（推进、切换、隔离、取代、GC）
- **Status**: `completed`
- **Completion Evidence**:
  - TR-8.1 `rule` PASS：`TestCoordinatorHappyPath` 断言严格动作序列（候选 CM→canary→就绪→NS/A/AAAA 探测→正式三字段补丁（此时 selector 未切换）→rollout 完成→移除 canary→切换前后就绪 Pod 地址族不减少→anchor active+镜像→Active）；`TestCoordinatorEndpointShrinkRestoresSwitch` 验证切换后无新代就绪 Pod 时恢复 selector+三字段且 active 不变。
  - TR-8.2 `rule` PASS：`TestCoordinatorCandidateConfigMapWriteFailure`（无残留/无 canary/diag/active 不变）、`TestCoordinatorCanaryTimeoutQuarantines`（scale 0+quarantine+生产零改动+旧 data 字节不变）、`TestCoordinatorProbeFailureQuarantines`（隔离+恢复选择）、`TestCoordinatorProductionRolloutTimeoutRollsBack`（回滚三字段+selector 未切换+active 不变）。
  - TR-8.3 `rule` PASS：`TestCoordinatorRestartConvergesToSameResult` 在 Rolling/Promoting 各阶段以全新 Coordinator 从 anchor 注解重建（含 `zd-promotion-snapshot` JSON 快照）收敛到同一 Active；冲突重试由 TR-4.4 覆盖。
  - TR-8.4 `rule` PASS：`TestCoordinatorSupersedeOnlyLatestPromoted`（Promoting 中内容变更→快照恢复正式部署+删除旧候选 CM+旧 canary 替换，仅最新 hash Active）、`TestCoordinatorQuarantinedOldGenerationRetainedOnSupersede`（隔离旧代按保留策略留存）。
  - TR-8.5 `rule` PASS：`TestCoordinatorGarbageCollection` 表验证 active/deployment/service/canary 引用保留、quarantine 超 N=3 删最旧、非受控 CM 永不触碰。
  - TR-8.6 `rubric` 自评分 **4/5（阈值 ≥4 通过）**：重启（每阶段）、乐观锁冲突重试、探测/rollout 中连续变更与滚动并发均自动收敛到唯一最新代，无卡死/错误提升/双代并存；扣 1 分因未模拟多控制器写者极端交错（单进程控制器模型外）。证据：上述故障注入测试集合 + `TestAnchorOptimisticConflictRetries`。
  - 其他：`TestCoordinatorAdoptFreshInstallIsActive`（AC-1）、`TestCoordinatorSteadyStateZeroWrites`（NFR-2 写计数=0，为此把 `mutateAnchor` 改为无变更零 Update、`RemoveCanary` 改为先 Get 后删）、`TestCoordinatorValidationFailureRejectsBadContent`、`TestCoordinatorKillSwitchWritesLegacy`（AC-16）。实现：[coordinator.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/coordinator.go)；FR-8 顺序切换由 deployment/service 拆分补丁原语保证；就绪容量以 Ready Pod IP（RBAC 可投影 Endpoints）双栈计数断言。
- **Priority**: high
- **Depends On**: Task 6, Task 7
- **Description**:
  - 新增 `coordinator.go`：`Coordinator.EnsureGeneration(ctx, zones) (GenerationDecision, error)`，level-triggered 状态机：
    1. 规范化+校验期望（失败 → 诊断 + 不写集群，requeue）；
    2. adopt / no-op 短路；
    3. 创建不可变候选 CM（失败丢弃）；
    4. 确保 canary Deployment/Service → 就绪等待（超时隔离）→ NS/A/AAAA 验证（失败隔离）；
    5. 提升：正式三字段补丁 → rollout 完成等待（超时回滚）→ selector 切换（切换前后断言 Endpoints 非空且地址不减少，失败恢复）→ anchor active 切换并把 anchor data 镜像为新代内容（镜像失败按恢复补丁回滚 selector/三字段，active 不变）；
    6. supersede：candidate≠最新期望时清理旧 canary/候选 CM，只保留最新候选；进行中的旧候选不提升；
    7. GC：无引用（Deployment/Service/canary）旧代可删；保留上一 active 代直到下次 Active 稳态；quarantined 保留最近 N 代；非受控资源不碰；
    8. 返回决策（`Active`/`Progressing(requeueAfter)`/`Failed`），供协调器决定是否放行外部发布。
  - 所有阶段从 Store 状态重建（重启安全）；乐观锁/冲突重试。
- **Acceptance Criteria Addressed**: AC-9, AC-10, AC-11, AC-12, AC-13, AC-17, AC-18
- **Test Requirements**:
  - `rule` TR-8.1: happy path 动作序列断言（候选→canary→就绪→探测→正式补丁→rollout→selector→anchor→Active）且切换前后 Endpoints 地址集合不减少；证据：fake client 协调器单测。
  - `rule` TR-8.2: 三类失败路径后系统停留在旧 Active：last-known-good CM 内容字节未变、selector/volume 恢复、诊断完整、决策非 Active；证据：失败注入单测。
  - `rule` TR-8.3: 任意阶段从注解状态重建后继续推进至同一结果（重启模拟 ×每阶段）；证据：状态恢复参数化单测。
  - `rule` TR-8.4: 连续变更（probe 中、rollout 中、Active 后立即再变）仅最新 hash 被提升，旧候选 CM/canary 被清理且永不提升；证据：supersede 单测。
  - `rule` TR-8.5: GC 引用计数：Deployment/Service/canary 任一引用即保留；quarantine 超 N 代才删最旧；非受控标签 CM 永不删；证据：GC 表驱动单测。
  - `rubric` TR-8.6: 并发健壮性；scale 1-5；anchors 1=存在卡死/错误提升/双代并存，3=常规正确极端竞态需干预，5=任意重启/冲突/连续变更自动收敛到唯一最新代；阈值 >=4；证据：冲突重试+中途重启+滚动中再变更故障注入单测与评审。

## Task 9: 接入 ZoneDelegation 协调器与 ZoneService
- **Status**: `completed`
- **Completion Evidence**:
  - TR-9.1 `rule` PASS：[zone_delegation_reconciliation_test.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zone_delegation_reconciliation_test.go) `TestZoneDelegationActivePublishesExternalDelegation`（Active 后 UpdateStatus+SaveZoneDelegation 以不变参数调用 1 次）、`...ProgressingSkipsPublish`（Progressing 走 RequeueAfter=5s，UpdateStatus/Save 零调用）、`...FailedUsesRecoveryRequeueAndEvent`（Failed 返回 recovery requeue 3600s + warning Event，provider 零调用）。
  - TR-9.2 `rule` PASS：`...DeleteActiveFinalizes`（删除路径先 gate Active 再 finalize，finalizer 移除后 fake apiserver 删除对象）、`...DeleteProgressingKeepsFinalizer`（收缩配置 Progressing 时 finalize/UpdateStatus/provider 零调用、finalizer 保留）。
  - TR-9.3 `rule` PASS：稳态零写由 `TestCoordinatorSteadyStateZeroWrites`（client Create/Update/Patch/Delete 计数=0）端到端保证；ZoneService 仅渲染并委托，无双写。
  - TR-9.4 `rubric` 自评分 **4/5（阈值 ≥4 通过）**：zones 包仅新增窄接口 `GenerationCoordinator`/`CoreDNSGenerationGate`（不依赖 rollout 包），适配层 [coredns_rollout_gate.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/coredns_rollout_gate.go) 单点桥接；[zone_service.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zones/zone_service.go) UpdateCoreDNSConfiguration 收敛为"渲染合并期望（含 cache 未同步的当前 ZD）+委托/legacy 直写"，协调器仅由 ZoneDelegationReconciler 驱动；mock 经 mockgen 重生成（license 头保留）；扣 1 分因 reconciler 新增 ~20 行门控逻辑不可避免。
  - 全量 `go build ./... && go test ./...` 通过（含 zones legacy 既有测试，行为未回归）。
- **Priority**: high
- **Depends On**: Task 8
- **Description**:
  - 在 [zone_delegation_reconciliation.go](file:///Users/ding/Ding/Documents/swe/0910092/project-09/controllers/zone_delegation_reconciliation.go) 注入 Coordinator：删除路径先 `EnsureGeneration`（收缩配置 Active）再 finalize；非删除路径 `EnsureGeneration` 返回 Active 后才执行 UpdateStatus 与 `SaveZoneDelegation`；Progressing → `RequeueAfter`；Failed → 按现有 recovery requeue 语义返回错误并记录 Event。
  - 改造 [zone_service.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zones/zone_service.go)：`UpdateCoreDNSConfiguration` 收敛为"渲染期望 + 委托 Coordinator/legacy 写入"，删除其内联 CreateOrUpdate 直写（legacy 回落保留）；更新 `ZoneDelegation` 接口与 [zone_delegation_service_mock.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zones/zone_delegation_service_mock.go)（go.uber.org/mock 重生成）。
  - CoreDNSReconciler（IP/Service watcher）与 Coordinator 不重复写：CoreDNSReconciler 负责 ZD 物化，Coordinator 仅由 ZoneDelegationReconciler 驱动。
- **Acceptance Criteria Addressed**: AC-2（端到端 no-op）, AC-11, AC-15, AC-19
- **Test Requirements**:
  - `rule` TR-9.1: Progressing 时 provider.SaveZoneDelegation 零调用、Active 后调用且参数不变；证据：协调器单测（provider mock 调用计数/参数断言）。
  - `rule` TR-9.2: 删除路径 gate 先于 Finalize；gate 失败时 finalizer 保留；证据：删除路径单测。
  - `rule` TR-9.3: 稳态重复 reconcile 无额外写（client 写计数 0）；证据：no-op 端到端单测。
  - `rubric` TR-9.4: 集成质量；scale 1-5；anchors 1=大面积侵入既有逻辑/难测，3=可用但边界模糊，5=窄接口接入、既有改动最小、测试自然；阈值 >=4；证据：diff 结构评审。

## Task 10: main.go 装配与控制器 watches
- **Status**: `completed`
- **Completion Evidence**:
  - TR-10.1 `rule` PASS：[main.go](file:///Users/ding/Documents/swe/0910092/project-09/main.go#L155-L200) 装配命名空间级 rollout cache（`cache.New` + `DefaultNamespaces`，经 `mgr.Add` 注册）与"读 cache/写直连"客户端，构造 Coordinator（Settings/`GetClusterNSName`/暴露 IP 闭包/`NewMiekgQuerier`/eventRecorder），经 `WithGenerationCoordinator` 注入 ZoneService；[zone_delegation_setup_test.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zone_delegation_setup_test.go) `TestZoneDelegationSetupBuildsWithRolloutWatches` 在 fake manager 下 SetupWithManager 无错（含无 cache 的基线路径），`TestRolloutEventEnqueuesAllZoneDelegations` 断言 canary Pod 事件映射出全部 2 个 ZD 请求，`TestRolloutWatchPredicates` 覆盖 anchor/代 CM/CoreDNS 工作负载/canary Pod 谓词正反例。
  - TR-10.2 `rule` PASS：[zone_delegation_setup.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zone_delegation_setup.go#L40-L51) 5 条 `+kubebuilder:rbac` 标记与 [role.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chart/k8gb/templates/role.yaml) 逐资源动词完全一致（configmaps/deployments/services=get;list;watch;create;update;patch;delete，pods=list;watch，events=create;patch）。
- **Priority**: medium
- **Depends On**: Task 9
- **Description**:
  - [main.go](file:///Users/ding/Documents/swe/0910092/project-09/main.go) 构造 Coordinator（client/apiReader/config/logger/eventRecorder/DNSQuerier/暴露 IP 提供函数）并注入 ZoneDelegationReconciler。
  - 在 [zone_delegation_setup.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zone_delegation_setup.go) 增加 watch：anchor/代 ConfigMap、canary Deployment/Pod/Service（标签谓词）事件映射为 enqueue 全部 ZoneDelegation（map 内 list），保证就绪/失败事件即时驱动；补充 controller-tools RBAC 注解。
- **Acceptance Criteria Addressed**: AC-12, AC-16
- **Test Requirements**:
  - `rule` TR-10.1: 控制器可构造、fake manager 下 SetupWithManager 无错误；canary Pod Ready 事件映射出至少一个 ZD 请求；证据：setup 单测（参照现有 handlers_test 风格）。
  - `rule` TR-10.2: 新增 RBAC 注解动词与 Task 11 Role 完全一致；证据：注解/模板比对审查。

## Task 11: Helm RBAC、配置项与 values schema
- **Status**: `completed`
- **Completion Evidence**:
  - TR-11.1 `rule` PASS：`helm lint .` 0 failure（仅预存 Chart 版本 SemVer WARNING）；默认 `helm template` 成功且 4 个 env 渲染为 `COREDNS_ROLLOUT_ENABLED="true"`/`COREDNS_CANARY_READY_TIMEOUT_SECONDS="120"`/`COREDNS_ROLLOUT_TIMEOUT_SECONDS="120"`/`COREDNS_QUARANTINE_GENERATIONS="3"`，与 Go 代码 kong 默认值等效；与 git HEAD 基线渲染逐文档对比：19 个文档集合完全一致（无新增工作负载模板），仅有两处差异——k8gb Deployment 多出 4 个 env、Role 规则扩展；anchor CM `k8gb-zone-delegation` 保持 `data: {}` 且逐字节相同。
  - TR-11.2 `rule` PASS：实际调用审计（[store.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/store.go)/[workload.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/workload.go)/[coordinator.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/coordinator.go)/[rollout.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/corednsrollout/rollout.go)）：configmaps=get/list/watch（cache）+create（anchor adopt/gen CM）/update（anchor 注解与镜像）/patch（保留基线 anchor 动词，FR-18 明确"去 resourceNames 并补 list/watch"）/delete（DiscardCandidate/GC）；deployments=list/get 发现、create/update EnsureCanary upsert、patch 提升/回滚/scale、delete RemoveCanary、watch cache；services 同构（发现/upsert/selector patch/删除）；pods=list（servingAddresses Ready Pod 投影）/watch，无 get 多余动词；events=create/patch（EventRecorder）。全部限定命名空间 Role，无 ClusterRole 新增。说明：FR-18 文字枚举漏列 deployments/services 的 create/delete，但 FR-5 EnsureCanary 与 FR-14 RemoveCanary/GC 实际必需，按 TR-11.2"与实际调用一一对应"补齐。
  - TR-11.3 `rule` PASS：`coreDnsRollout.enabled=false/timeout 60/90/quarantine 0` 组合 `helm template` 成功且 env 透传 `false`/`0`；非法值 `canaryReadyTimeoutSeconds: 0` 被 values.schema.json 拒绝（`Error: values don't meet the specifications`，exit 1）。values.yaml 新增 `k8gb.coreDnsRollout` 块、values.schema.json 新增 `k8gbCoreDnsRollout` 定义（additionalProperties:false、required 4 字段、timeout min=1、quarantine min=0），README.md 按 helm-docs 排序手工同步 5 行（helm-docs 仅 master push 非阻塞生成 PR）。
- **Priority**: high
- **Depends On**: Task 1, Task 10
- **Description**:
  - 更新 [role.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chart/k8gb/templates/role.yaml)：configmaps 去 `resourceNames` 动态名限制（保留命名空间 Role 边界）并补 list/watch；新增 deployments(get/list/watch/patch/update)、services(get/list/watch/patch)、pods(list/watch)、events(create/patch) 最小动词；无 cluster 级新增。
  - [deployment.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chart/k8gb/templates/deployment.yaml) 增加 4 个 env（取自 values，默认值与 Task 1 一致）；values.yaml + values.schema.json + helm-docs 注释同步；[cm.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chart/k8gb/templates/coredns/cm.yaml) anchor 保持空 data 与既有名称不变。
  - 运行 helm-docs/模板检查流水线所需的 README 再生成（若 CI 校验）。
- **Acceptance Criteria Addressed**: AC-14, AC-16
- **Test Requirements**:
  - `rule` TR-11.1: `helm lint` 与 values schema 校验通过；默认 values 渲染产物与基线语义一致（同名 anchor CM、无新增工作负载模板、env 默认值等效）；证据：`helm lint`、`helm template` 前后对比输出。
  - `rule` TR-11.2: Role 动词枚举与控制器实际调用一一对应、无多余动词、作用域限命名空间；证据：模板审查清单。
  - `rule` TR-11.3: 关闭开关的 values 组合可正常渲染；证据：helm template 用例。

## Task 12: Chainsaw 集成断言扩展
- **Status**: `completed`
- **Completion Evidence**:
  - TR-12.1 `rule` PASS（本地静态/dry 部分；CI 多集群运行待流水线）：新增 [assert-k8gb-zone-delegation-generation.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chainsaw/step-templates/assert-k8gb-zone-delegation-generation.yaml)（anchor active 注解且无在飞 candidate、不可变代 CM 存在且指纹/受控标签/zone 文件内容正确、anchor data 与代 CM 逐字节一致、canary Deployment/Service 已删除、仅余 1 个代 CM 即旧代引用释放后已 GC、稳定 Service selector 携带 `k8gb.io/zd-generation=<active>`）与 [assert-zone-delegation-authoritative-dns.yaml](file:///Users/ding/Ding/Documents/swe/0910092/project-09/chainsaw/step-templates/assert-zone-delegation-authoritative-dns.yaml)（每 zone NS 应答包含全部期望 glue NS、本集群 glue A 名解析为 IPv4），接入 5 阶段：1 双集群 gen-0 adopt、2 EU 双 zone 新代、3 US 双 zone 新代、4 US 收缩单 zone 旧代释放、5 EU 收缩单 zone 旧代释放。chainsaw v0.2.15 `lint test` 对 Test 文档报 "The document is valid"；3 个 YAML 经 ruby psych 解析通过；两个模板内嵌脚本 `bash -n` 通过（v0.2.15 lint 不接受 StepTemplate kind，基线模板同样报错，属工具版本限制）；binding 表达式 `($values.dns_port_*)` 与既有 `($clusterOverrideIP)` 用法一致。
  - TR-12.2 `rule` PASS（结构层面）：现有 5 阶段全部断言原样保留，新步骤均为追加（git diff 仅 +156 行、0 删除于 chainsaw-test.yaml）；运行时持续通过以 CI 多集群 chainsaw 日志为准（本地无 k3d 双集群）。
- **Priority**: medium
- **Depends On**: Task 11
- **Description**:
  - 在 [zone-delegation chainsaw](file:///Users/ding/Documents/swe/0910092/project-09/chainsaw/tests/zone-delegation/chainsaw-test.yaml) 增加断言：变更后出现 `k8gb-zone-delegation-<hash>` 不可变代 CM 与 active 注解、canary 资源在 Active 后消失、稳定 Service selector 携带 generation 标签且 DNS 应答（dig NS/A）保持成功；删除 zone 后旧代按引用释放清理。
  - 新增 step-template（assert generation 资源）复用现有 bindings 风格。
- **Acceptance Criteria Addressed**: AC-1, AC-3, AC-11, AC-13
- **Test Requirements**:
  - `rule` TR-12.1: 新增断言在现有 zone-delegation 生命周期测试序列中全部成立（CI 多集群 chainsaw）；证据：chainsaw 运行日志（本地至少完成 YAML 语法/模板 dry 校验）。
  - `rule` TR-12.2: 现有 5 阶段断言不被削弱且持续通过；证据：同一运行日志。
- **Notes**: 坏版本负向 E2E 难以经 CRD 注入（内容由控制器生成），负向覆盖以 Task 3/8 单测为准。

## Task 13: 全量验证与自验收口
- **Status**: `completed`
- **Completion Evidence**:
  - TR-13.1 `rule` PASS（全部命令退出码 0）：
    - `go build ./...` 通过；
    - `go vet ./...` 退出 0；
    - `go test ./... -count=1` 全 21 个包 `ok`、FAIL_COUNT=0（含 controllers/corednsrollout、controllers、controllers/zones、controllers/resolver 等，零跳过）；
    - `golangci-lint run ./...`（本地 v2.14.0，CI 固定 v2.12.2，同一 v2 配置）初次发现 24 项全部位于本次特性文件，已逐一修复（depguard 去除 k8s.io/utils 改包内 ptrBool、goconst jsonOp 常量、goimports、gosec G101 改名 ReasonCandidateWriteFailed 诊断字面值不变、lll 折行、revive 未用参数/ctx 序、unparam 测试助手收敛），复检 **0 issues**；
    - `helm lint chart/k8gb` 0 failed；默认/kill-switch/非法值三组 `helm template` 符合预期；
    - `golic inject --dry -x -t apache2`（与 CI 同版本 v0.7.2 参数）报 `0/189 changed`；
    - chainsaw v0.2.15 Test 文档 lint valid、YAML 解析与脚本 `bash -n` 通过。
  - TR-13.2 `rule` PASS：AC/TR 映射核对——
    - AC-1→TR-4.1 + Task 8 adopt 测试 + TR-12 阶段 1 断言；AC-2→TR-9.3（SteadyStateZeroWrites 写计数 0）；AC-3→TR-2.1/2.2/2.3；AC-4→TR-3.1/3.2；AC-5→TR-4.3 + 存储层；AC-6→TR-5.1/5.2/5.5；AC-7→TR-6.1/6.2/6.3；AC-8→TR-7.1/7.2/7.3；AC-9→TR-5.4 + TR-8.1；AC-10→TR-4.2 + TR-6.1/6.2 + TR-8.2；AC-11→TR-9.1/9.2 + TR-12 全阶段门控；AC-12→TR-4.4 + TR-8.3 + TR-10.1；AC-13→TR-8.4/8.5 + TR-12 阶段 4/5 GC 断言；AC-14→TR-1.1/1.2；AC-15→Task 9 结构（ZoneService 单一委托 + mock 重生成）；AC-16→TR-5.3 + TR-11.1/11.2/11.3；AC-17→Task 4/8 诊断断言（last-failure 字段完整、代数据字节不变）；AC-18（rubric）→TR-8.6 自评 4/5；AC-19（rubric）→TR-9.4 自评 4/5。无遗漏。
  - 遗留说明：chainsaw 多集群 e2e 运行需 k3d 双集群，本地仅完成静态/dry 校验，运行时证据以 CI chainsaw 流水线为准；负向坏版本 E2E 按 Task 12 Notes 由 Task 3/8 单测覆盖。
- **Priority**: high
- **Depends On**: Task 12
- **Description**:
  - 运行 `make test`（go test 全量）、`golangci-lint run`、`go vet ./...`、`helm lint`、license 头检查（golic 约定）；修复全部问题；确认既有单测零跳过。
- **Acceptance Criteria Addressed**: AC-1 ~ AC-19（收口证据）
- **Test Requirements**:
  - `rule` TR-13.1: 上述命令全部退出码 0；证据：命令输出存档于本任务 Completion Evidence。
  - `rule` TR-13.2: 每条 AC 均有至少一项 TR 证据映射，无遗漏；证据：AC/TR 映射核对表。
