# CoreDNS Zone 配置安全分代 Rollout - 产品需求文档

## Overview
- **Summary**: 为 k8gb 边缘集群的 CoreDNS 动态 zone 委派配置引入"分代安全 rollout"。规范化后的 zone 配置先形成不可变候选 ConfigMap，经 Corefile 语法与 zone 静态校验、独立 canary Deployment/Service rollout、就绪检查，以及从候选 Service 发起的权威 NS/A/AAAA 查询全部通过后，才把活动代切换到新配置，并在此之后才允许 ZoneDelegation 向外部边缘 DNS 发布对应委派地址。写失败丢弃候选、rollout 超时回滚 Deployment、查询失败隔离候选并恢复旧 Service 选择；任何失败都不覆盖最后可用配置且保留诊断。控制器重启、连续变更与 Pod 滚动并发时只推进最新候选，旧代在引用释放前继续应答，安全后再清理。
- **Purpose**: 当前 [zone_service.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zones/zone_service.go#L273-L316) 的 `UpdateCoreDNSConfiguration` 通过 `CreateOrUpdate` 直接重写 anchor ConfigMap `k8gb-zone-delegation`，而 [zone_delegation_reconciliation.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zone_delegation_reconciliation.go#L92-L128) 随后立即调用 `DNSProvider.SaveZoneDelegation`。CoreDNS 以 volume 方式挂载该 CM（`import ../dynamic/*.conf`，reload 30s/15s jitter）：配置换新后一旦 Pod 重启（升级、驱逐、节点维护），坏配置会导致 CoreDNS 无法启动，稳定 Service 没有可用端点；而外部委派可能已经发布，边缘 DNS 会把该 zone 的解析导向坏集群。需要在"配置写入"与"外部发布"之间建立可验证、可回滚、重启安全的分代屏障。
- **Target Users**: 运维 / 平台管理员（更新边缘 DNS 或 zone 委派配置）、多集群 k8gb 使用者（期望 zone 委派发布期间解析不中断）。

## Goals
- 配置变更先在独立 canary 上验证（Corefile 语法/zone 静态校验 → Pod 就绪 → 经 Service 的权威 DNS 查询），通过后才切换活动代。
- 外部委派（NS/glue 记录）的发布以"新代已成为活动代并持续正确应答"为硬门控。
- 任何一类失败都让系统停留在最后可用代：候选丢弃、Deployment 回滚、Service 选择恢复，last-known-good 内容字节不变，诊断可追溯。
- 控制器重启、乐观锁冲突、连续变更与滚动发布并发时，自动收敛到唯一的最新代；旧代在仍被引用期间继续应答，无引用后才清理。
- 首次安装、无内容变化、现有 readiness/liveness 探针、IPv4/IPv6 暴露地址语义、Helm 默认渲染均保持原有可观测行为，并提供 kill-switch 回落到旧路径。

## Non-Goals
- 不修改 ZoneDelegation CRD、GSLB CRD 或外部 DNS provider 的接口与记录语义。
- 不处理应用 GSLB DNSEndpoint（`localtargets-*`/final A 记录）的安全发布，仅覆盖 zone 委派（CoreDNS dynamic-zones）链路。
- 不引入 CoreDNS 代码库依赖、不替换 CoreDNS `reload` 插件、不重定义现有 readiness/liveness 探针。
- 不做跨集群代协调；各集群独立分代、独立收敛。
- 不新增集群级（ClusterRole）RBAC；所有新权限限定在 k8gb 命名空间 Role 内。
- 不构造坏版本的端到端负向用例（配置内容由控制器渲染，无法经 CRD 注入）；负向覆盖以纯函数/协调器单测为准。

## Background & Context
- anchor ConfigMap `k8gb-zone-delegation` 由 Helm 模板创建（空 data、固定名），见 [cm.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chart/k8gb/templates/coredns/cm.yaml#L45-L50)；CoreDNS subchart（1.46.1）以 `dynamic-zones` volume 挂载它，见 values.yaml `coredns.extraVolumes`。
- 现有渲染每个 zone 一个 key `<lbzone-dotted>.conf`，内容固定为 `<zone>:5353 {\n  import k8gbplugins\n}\n`（[zone_service.go getCoreDNSData](file:///Users/ding/Documents/swe/0910092/project-09/controllers/zones/zone_service.go#L318-L326)）；chainsaw 现有 5 阶段断言直接读取 anchor CM 的 `.data` 验证这些键值（[assert-k8gb-zone-delegation-cm.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chainsaw/step-templates/assert-k8gb-zone-delegation-cm.yaml)），因此 anchor CM 必须继续镜像活动代数据。
- 稳定 CoreDNS Service 由标签 `app.kubernetes.io/name=coredns,app.kubernetes.io/component!=metrics` 发现，要求恰好一个（[corednssvc.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/providers/assistant/corednssvc.go#L33-L70)）；servicePort 53 映射容器端口 5353（values.yaml `coredns.servers`）。
- 暴露地址经 ipresolver 从 CoreDNS Service（ClusterIP 双栈 / LoadBalancer ingress）或标注 ingress 发现；外部委派期望 glue NS 名由 `Config.GetClusterNSName(s)` 生成（[ns_names.go](file:///Users/ding/Documents/swe/0910092/project-09/controllers/resolver/ns_names.go#L55-L79)）。
- DNS 查询已有 `utils.Exchange`（miekg/dns）与 `DNSQueryService` 抽象可复用；命名空间由 `Config.K8gbNamespace`（POD_NAMESPACE）提供。
- ZoneDelegation 协调循环顺序：IP 门控 → Get → finalizer → `UpdateCoreDNSConfiguration` →（删除路径 finalize）→ `UpdateStatus` → `SaveZoneDelegation`。bootstrap（CoreDNS）协调器仅把配置型 ZD 物化为 CR（`ZoneService.Save`），不直接写 CM。
- Role 当前仅允许 create configmaps 与对 `k8gb-zone-delegation` 这一 resourceName 的 get/update/patch/delete（[role.yaml](file:///Users/ding/Documents/swe/0910092/project-09/chart/k8gb/templates/role.yaml)）。

## Functional Requirements

### 规范化、候选与校验
- **FR-1（规范化与指纹）**: 从全部非删除中的 ZoneDelegation 集合渲染确定性 ConfigMap data：key 排序、单 zone 输出与现有 `getCoreDNSData` 逐字节一致；对 data 计算 SHA-256 截断指纹；代 ConfigMap 命名 `k8gb-zone-delegation-<hash>`，合法且符合 DNS-1123/253。
- **FR-2（不可变候选）**: 候选代 CM 创建时置 `Immutable: true`，携带控制器标签（managed-by、zd-generation、zd-role）与指纹/模板注解；同指纹已存在时创建幂等（AlreadyExists 视为成功且不修改既有对象）；代 CM 只创建、永不 update。
- **FR-3（Corefile/zone 静态校验）**: 写入集群前对规范化 data 的每个 `<zone>.conf` 做纯函数校验：server block 头（zone token + 端口）、括号配平、仅允许受管插件行（`import k8gbplugins`）、注释/空行；zone 经 miekg/dns 与现有 RFC1035 工具做 FQDN/标签校验；拒绝重复 zone、重复 server block、非法端口、未知插件行、空 zone；错误必须携带 zone 名与具体原因。
- **FR-4（adopt gen-0 与 no-op）**: 首次运行时以期望 data adopt 为 gen-0（anchor 已有非空数据则快照其内容），anchor active 指向 gen-0；期望指纹等于活动代指纹时零写短路返回 Active。anchor CM 的 data 在任何时刻都与活动代 CM 内容一致（镜像），以保持现有 chainsaw 断言与人工排障读取。

### 独立 rollout、就绪与权威验证
- **FR-5（canary 工作负载）**: 按现有标签发现唯一稳定 Service 及同命名空间内 selector 匹配的唯一正式 Deployment；0 个/多个返回可识别的 DiscoveryError。canary = 深克隆正式 Pod 模板，仅允许白名单差异：`dynamic-zones` volume 的 configMap.name 改为候选 CM、Pod/模板标签加 `zd-generation=<hash>` 与 canary role、checksum 注解、replicas=1；探针/镜像/命令/ENV/端口/其他卷/SecurityContext/ServiceAccount 全等。候选 Service 同名固定名 ClusterIP，端口（含 servicePort→targetPort 映射）与端口族克隆稳定 Service，selector 只命中候选代；canary 资源带 ownerReference（指向 anchor CM）与受控标签。
- **FR-6（就绪等待，非阻塞）**: 协调循环不阻塞，立即返回进行中状态并以固定短间隔 `RequeueAfter` 轮询；canary Ready 判定基于 Deployment/Pod status 且沿用现有探针结果（Updated/Ready/Available 达标）；正式 rollout 完成判定要求 `observedGeneration≥generation` 且 updated/available 副本达标、无不可用条件。canary 超时 → scale 0 并标记 quarantined 原因；正式 rollout 超时 → 生成恢复补丁回滚三字段。超时值来自配置。
- **FR-7（权威 NS/A/AAAA 探测）**: 经候选 Service `<name>.<ns>.svc` 与稳定 Service 的端口映射，对每个受影响 zone：NS 查询 loadBalancedZone 要求 NOERROR + AA，Answer 中 NS 集合包含期望 glue NS（至少本集群 `Config.GetClusterNSName`）；A 查询本集群 glue NS 名，地址集合必须等于 ipresolver 当前暴露 IPv4 集合（经窄接口注入）；Service 暴露 IPv6 时 AAAA 集合必须相等，无 v6 暴露时 AAAA NODATA（NOERROR 空 answer）视为通过。SERVFAIL/REFUSED/超时/无 AA/NS 缺失/集合不符均判失败。返回结构化结果（每条查询的 rcode、AA、期望集、实际集）。

### 提升、门控与失败处置
- **FR-8（提升序列与可逆切换）**: 探测通过后：① 对正式 Deployment 打仅含三字段（volume CM 名、generation 标签、checksum 注解）的补丁；② 等待正式 rollout 完成（超时回滚，active 不变）；③ 切换稳定 Service selector 到新代标签，切换前后断言 Endpoints 非空且地址集合不减少，失败则应用恢复补丁（旧三字段 + 旧 selector，二者与正向补丁互为逆操作）；④ 切换 anchor active 注解并把 anchor data 镜像为新代内容。
- **FR-9（外部委派发布门控）**: `EnsureGeneration` 返回 Active 之前，ZoneDelegation 协调器不得调用 `UpdateStatus` 与 `DNSProvider.SaveZoneDelegation`（零调用）；Progressing → RequeueAfter；Failed → 按现有 recovery requeue 语义返回错误并记录 Event。删除路径先把"收缩后配置"推进到 Active 再 finalize；门控未过时 finalizer 保留。
- **FR-10（三类失败处置，不污染 last-known-good）**: ① 候选 CM 写失败 → 删除半成品候选、清除 candidate 注解、记录 last-failure，active 不变，下次 reconcile 可重试；② canary/正式 rollout 超时 → canary scale 0 + quarantine，或正式三字段回滚，active 不变；③ DNS 探测失败 → 候选隔离（quarantine 标签/原因）、恢复旧 Service selector，正式工作负载继续由旧代支撑。三类路径后最后可用代 CM 内容字节不变、Service 仍能正确解析。
- **FR-11（诊断保留）**: 每次失败在 anchor 写入结构化 last-failure 诊断（阶段、原因类别、zone 名、查询 rcode/AA/期望/实际摘要、时间戳）；quarantined 代保留原因标签/注解；诊断写入不得覆盖或截断任何代 CM 数据。quarantined 代保留最近 N 代（默认 3）。

### 重启安全、取代与回收
- **FR-12（重启安全与冲突重试）**: 全部阶段状态保存在 anchor CM 注解（JSON sub-annotation：active/candidate/phase/last-failure/last-template），任意阶段重启后从存储状态重建并继续推进至同一结果；anchor 乐观锁冲突重试至一致。控制器 watch anchor/代 CM、canary Deployment/Pod/Service（标签谓词），事件映射为 enqueue 全部 ZoneDelegation，使就绪/失败即时驱动而非仅等轮询。
- **FR-13（连续变更只推进最新候选）**: 当 candidate 与最新期望指纹不一致时（探测中、rollout 中、Active 后立即再变），清理旧 canary 与旧候选半成品，进行中的旧候选永不提升；任意时刻至多一个候选被推进，最终只有最新指纹成为活动代。
- **FR-14（引用计数 GC）**: 代 CM 仍被正式 Deployment、稳定 Service 或 canary 引用时必须保留并继续应答；上一活动代保留至下一次 Active 稳态达成；quarantined 代超出保留代数后从最旧删除；不带受控标签的 ConfigMap/工作负载永不被删除或修改。

### 配置、装配与 Helm
- **FR-15（配置门控）**: 新增 kong 配置：`CoreDNSRolloutEnabled`（env `COREDNS_ROLLOUT_ENABLED`，默认 true）、`CoreDNSCanaryReadyTimeoutSeconds`（默认 120，min=1）、`CoreDNSRolloutTimeoutSeconds`（默认 120，min=1）、`CoreDNSQuarantineGenerations`（默认 3，min=0）；非法值启动校验失败；help 文案说明 kill-switch 回落语义。
- **FR-16（legacy 回落）**: kill-switch 关闭或工作负载发现失败（DiscoveryError）时，退回直接把期望 data 写入 anchor CM 的现有行为（渲染结果与旧实现一致），不创建 canary/代资源。
- **FR-17（单一写入者）**: `ZoneService.UpdateCoreDNSConfiguration` 收敛为"渲染期望 + 委托 Coordinator 或 legacy 直写"，删除其内联 CreateOrUpdate 之外的双写；Coordinator 仅由 ZoneDelegationReconciler 驱动；bootstrap（CoreDNS）协调器继续只负责 ZD CR 物化，不写 CM。
- **FR-18（Helm/RBAC/默认渲染）**: Role 在命名空间内补最小动词（configmaps 去 resourceNames 动态名限制并补 list/watch；deployments get/list/watch/patch/update；services get/list/watch/patch；pods list/watch；events create/patch），无集群级新增；deployment.yaml 透传 4 个 env 并同步 values.yaml/values.schema.json/helm-docs；anchor 模板保持同名空 data；默认 values 渲染与基线语义一致（同名 anchor、无新增工作负载模板、env 默认等效），关闭开关的 values 组合可正常渲染。
- **FR-19（IPv4/IPv6 与暴露地址一致性）**: canary 与候选 Service 的 IP 族、端口族与稳定 Service 一致；探测期望地址取自 ipresolver 当前暴露集合（v4、v6 分离判定），不改变现有暴露地址发现、`ClusterExposedIPs` 覆盖与双栈行为。

## Non-Functional Requirements
- **NFR-1（可测性）**: 所有外部边界（controller-runtime client、DNS querier、时钟）经窄接口注入；状态判定与校验为纯函数；以 fake client + fake querier 的表驱动单测覆盖 happy path、三类失败、每阶段重启、连续变更与 GC。
- **NFR-2（稳态零写）**: 稳态重复 reconcile 不产生任何集群写操作（client 写计数 0），除周期性 requeue 外无多余动作。
- **NFR-3（安全最小权限）**: 新增 RBAC 仅限 k8gb 命名空间 Role，动词与控制器实际调用一一对应、无多余动词；代 CM 不可变；非受控资源不触碰。
- **NFR-4（向后兼容）**: 默认配置下现有 zone-delegation chainsaw 5 阶段断言不被削弱且持续通过；`helm lint`、values schema、README/helm-docs 检查通过；既有 Go 单测零跳过。
- **NFR-5（可观测）**: 沿用 zerolog 结构化日志与 Kubernetes Event；分代推进/隔离/回滚/GC 均有日志与（适用处）Event，失败诊断可经 anchor 注解离线查看。
- **NFR-6（侵入最小）**: 经窄接口接入现有 `ZoneDelegationReconciler` 与 `zones.ZoneDelegation`，既有协调逻辑改动最小、测试自然。

## Constraints
- **Technical**: Go + controller-runtime client + miekg/dns；不得引入 coredns 代码库依赖；配置字段/风格对齐 kong 既有字段（如 `ZoneDelegationRecoverySeconds`）；license 头遵循 golic 约定。
- **Business**: 任何情况下不得覆盖最后可用配置；外部委派不得在新代 Active 前发布；首次安装与 Helm 默认渲染的外部可观测行为必须与现状一致。
- **Dependencies**: coredns Helm subchart 1.46.1 渲染的 Deployment/Service 标签与 `dynamic-zones` volume 名；CoreDNS servicePort 53→5353 映射；控制器 Pod 具备从集群内经 Service DNS 名访问候选 Service 的网络条件。

## Assumptions
- 正式 CoreDNS Deployment 与稳定 Service 均位于 `Config.K8gbNamespace`，且 canary 与其同命名空间。
- anchor CM data 镜像活动代是"保持原有行为"的一部分（现有 chainsaw 与人工排障直接读 anchor data）。
- 候选 Service 通过 ClusterIP DNS（`<name>.<ns>.svc`）访问，沿用稳定 Service 的 servicePort/targetPort 映射（53→5353）。
- 控制器具备对代 CM 动态名称的 list/watch 与同命名空间 Deployment/Service/Pod 的最小 RBAC（Helm 同步授予）。

## Acceptance Criteria

### AC-1: 首次安装 adopt 行为与现状等价
- **Type**: `rule`
- **Given**: 全新集群，anchor CM `k8gb-zone-delegation` 存在且 data 为空，存在一个默认 ZoneDelegation
- **When**: 首次 reconcile 运行分代流程
- **Then**: 以期望 data adopt gen-0（不可变代 CM 内容等于期望渲染），anchor active 指向 gen-0 且 anchor data 与活动代内容一致；外部委派在 Active 后才发布；现有 zone-delegation 5 阶段 chainsaw 断言全部成立
- **Pass Condition**: fake client 单测中 adopt 后 anchor active=gen0/phase=Active、gen-0 immutable=true、anchor/gen-0 数据与期望逐字节一致；chainsaw 现有断言不削弱并通过
- **Evidence**: Task 4 TR-4.1、Task 12 TR-12.2

### AC-2: 无内容变化零写短路
- **Type**: `rule`
- **Given**: 活动代指纹等于当前 ZoneDelegation 集合的期望指纹
- **When**: 任意 ZoneDelegation 触发 reconcile
- **Then**: 直接返回 Active，不创建/修改任何 CM、Deployment、Service，不调用外部 DNS provider
- **Pass Condition**: 稳态重复 reconcile 的 client 写计数为 0 且 `SaveZoneDelegation` 零调用
- **Evidence**: Task 2/Task 9 TR-9.3

### AC-3: 规范化输出确定且与旧渲染字节一致
- **Type**: `rule`
- **Given**: 任意顺序的同组 ZoneDelegation（含删除中对象）
- **When**: 执行规范化渲染与指纹计算
- **Then**: 同集合产生相同 data 与指纹；单字节差异产生不同指纹；每个 `<zone>.conf` 与现有 `getCoreDNSData` 输出逐字节一致；删除中 ZD 不进入 data；代 CM 名符合 DNS-1123/253
- **Pass Condition**: normalize 表驱动单测全部断言通过（顺序无关、差异敏感、字节一致、删除排除、命名合法）
- **Evidence**: Task 2 TR-2.1/2.2/2.3、Task 12

### AC-4: Corefile 语法与 zone 静态校验在写集群前拦截坏配置
- **Type**: `rule`
- **Given**: 规范化后的候选 data
- **When**: 运行静态校验器
- **Then**: 正例（当前模板产物、多 zone、带注释）通过；括号不配平、非法 zone 字符、重复 zone/block、错误端口、未知插件行、空 zone 被逐一拒绝且错误含 zone 名与原因；校验为不接触 client 的纯函数
- **Pass Condition**: validate 表驱动单测正/负例全部符合预期，且包内无 client 依赖
- **Evidence**: Task 3 TR-3.1/3.2

### AC-5: 候选代 ConfigMap 不可变、内容寻址、创建幂等
- **Type**: `rule`
- **Given**: 一个通过校验的候选
- **When**: 创建代 CM（含重试场景）
- **Then**: 代 CM `Immutable: true`、名为 `k8gb-zone-delegation-<hash>`、带受控标签/指纹注解；同指纹 AlreadyExists 视为成功且不改既有对象；代 CM 永不被 update
- **Pass Condition**: fake client 单测断言不可变、幂等、无 update 路径
- **Evidence**: Task 4 TR-4.3 及存储层单测

### AC-6: canary 是正式工作负载的最小白名单克隆
- **Type**: `rule`
- **Given**: 唯一稳定 Service 与唯一正式 Deployment
- **When**: 构建 canary Deployment/Service
- **Then**: 结构化 diff 仅含白名单字段（dynamic-zones volume 来源、zd-generation/role 标签、checksum 注解、replicas=1）；探针/镜像/命令/ENV/端口/其他卷/SecurityContext/SA 全等；候选 Service 为 ClusterIP、端口与端口族一致、selector 只命中候选代；canary 带 ownerReference 与受控标签
- **Pass Condition**: 模板 diff 表驱动单测 TR-5.1/5.2/5.5 通过
- **Evidence**: Task 5 TR-5.1/5.2/5.5

### AC-7: 就绪门控非阻塞且超时判定正确
- **Type**: `rule`
- **Given**: canary 或正式 Deployment 处于各种 status 序列
- **When**: 协调器轮询就绪/完成状态
- **Then**: canary NotReady 时返回 InProgress 且 DNS 探测零调用；canary 超时后 replicas=0 且带 quarantine 原因、正式资源无补丁；正式 rollout 完成判定覆盖 observedGeneration 滞后、部分可用、全部可用；正式超时生成回滚补丁且 active 不变
- **Pass Condition**: 状态序列单测 + fake querier 零调用断言（TR-6.1/6.2/6.3）
- **Evidence**: Task 6 TR-6.1/6.2/6.3

### AC-8: 经候选 Service 的权威 NS/A/AAAA 验证覆盖双栈与全失败模式
- **Type**: `rule`
- **Given**: 候选 Service 可达，fake querier 可编程应答
- **When**: 对每个受影响 zone 执行 NS/A/AAAA 探测
- **Then**: 双栈与单栈各有全通过路径；SERVFAIL、REFUSED、超时、无 AA、NS 缺失、A 集合多/少/错 IP、有 v6 时 AAAA 缺失/不符分别失败且结果含实际应答摘要；无 v6 时 AAAA NODATA 通过、错误码失败
- **Pass Condition**: probe 表驱动单测 TR-7.1/7.2/7.3 全部通过
- **Evidence**: Task 7 TR-7.1/7.2/7.3

### AC-9: 提升切换有序、可逆且 Endpoints 不减少
- **Type**: `rule`
- **Given**: 候选探测通过
- **When**: 执行三字段补丁 → 正式 rollout → Service selector 切换 → anchor active 切换
- **Then**: 动作严格有序；切换前后 Endpoints 非空且地址集合不减少；正向补丁与恢复补丁互为逆操作（apply-restore 后 deep-equal）；selector 切换失败时恢复旧值且 active 不变
- **Pass Condition**: patch roundtrip 单测 + happy path 协调器单测的动作序列与 Endpoints 断言
- **Evidence**: Task 5 TR-5.4、Task 8 TR-8.1

### AC-10: 三类失败后系统停留在旧 Active 且不覆盖最后可用配置
- **Type**: `rule`
- **Given**: 候选 CM 写失败 / canary 或正式 rollout 超时 / DNS 探测失败
- **When**: 分别注入三类失败
- **Then**: 写失败无代 CM 残留且 candidate 注解清除；rollout 超时 Deployment 回滚三字段；探测失败候选 quarantine 且 Service selector 恢复旧代；三类路径后 last-known-good 代 CM 字节未变、旧代持续应答、决策非 Active、诊断完整；下次 reconcile 可重新推进
- **Pass Condition**: 失败注入单测 TR-4.2/TR-8.2 全部断言（内容字节、selector/volume、诊断、决策）
- **Evidence**: Task 4 TR-4.2、Task 6 TR-6.1/6.2、Task 8 TR-8.2

### AC-11: 外部委派发布与 finalize 均受 Active 门控
- **Type**: `rule`
- **Given**: ZoneDelegation 非删除/删除两种调谐
- **When**: EnsureGeneration 返回 Progressing/Failed/Active
- **Then**: 非 Active 时 `UpdateStatus` 与 `SaveZoneDelegation` 零调用、Progressing 走 RequeueAfter、Failed 走 recovery requeue 并记 Event；Active 后 provider 以不变参数调用；删除路径先把收缩配置推进到 Active 再 finalize，gate 失败时 finalizer 保留
- **Pass Condition**: 协调器 mock 调用计数/参数单测 TR-9.1 与删除路径单测 TR-9.2
- **Evidence**: Task 8/9 TR-9.1/9.2、Task 12

### AC-12: 任意阶段重启/冲突后收敛到同一结果
- **Type**: `rule`
- **Given**: 流程每个阶段（候选创建后、canary 就绪前、探测中、正式 rollout 中、切换前后）
- **When**: 模拟控制器重启（仅从 anchor 注解重建）与 anchor 乐观冲突
- **Then**: 重建后继续推进至与无故障相同的最终状态；冲突重试后一致；canary Pod Ready 等事件能映射出至少一个 ZoneDelegation 调谐请求
- **Pass Condition**: 每阶段重启参数化单测 TR-8.3、冲突注入 TR-4.4、setup watch 映射单测 TR-10.1
- **Evidence**: Task 4 TR-4.4、Task 8 TR-8.3、Task 10 TR-10.1

### AC-13: 连续变更只提升最新代，旧代引用释放后才清理
- **Type**: `rule`
- **Given**: 探测中/rollout 中/Active 后立即发生期望变更
- **When**: 新指纹 candidate 取代旧 candidate 并到达稳态
- **Then**: 仅最新 hash 被提升，旧候选 canary/CM 被清理且永不提升；被 Deployment/Service/canary 引用的代保留并继续应答；上一活动代保留至下次 Active 稳态；quarantine 超 N 代才删最旧；非受控标签 CM 永不删
- **Pass Condition**: supersede 单测 TR-8.4 与 GC 表驱动单测 TR-8.5
- **Evidence**: Task 8 TR-8.4/8.5、Task 12

### AC-14: 配置项默认值与校验正确
- **Type**: `rule`
- **Given**: 配置解析（未设置 env / 显式 false / 非法数值）
- **When**: ResolveOperatorConfig
- **Then**: 默认值为 enabled=true、canary timeout=120、rollout timeout=120、quarantine=3 且校验通过；`COREDNS_ROLLOUT_ENABLED=false` 解析为关闭；超时 0/负数启动校验报错
- **Pass Condition**: `go test ./controllers/resolver/ -run Config`（TR-1.1/1.2）
- **Evidence**: Task 1 TR-1.1/1.2

### AC-15: CoreDNS 配置写入收敛为单一写入者
- **Type**: `rule`
- **Given**: ZoneDelegation 与 bootstrap 两个协调器
- **When**: 发生 ZD 物化与配置更新
- **Then**: `UpdateCoreDNSConfiguration` 仅渲染并委托 Coordinator（或 legacy 直写），不存在第二条代资源写路径；Coordinator 仅由 ZoneDelegationReconciler 驱动；ZoneDelegation 接口与 mock 同步更新
- **Pass Condition**: 代码结构审查 + 协调器单测中 bootstrap 路径不产生代资源写
- **Evidence**: Task 9

### AC-16: kill-switch/发现失败回落 legacy 且 Helm 默认渲染等价、RBAC 最小
- **Type**: `rule`
- **Given**: 开关关闭或 Service/Deployment 发现为 0/多个；默认/关闭两种 values
- **When**: 调谐或 helm template/lint
- **Then**: 回落路径直接写 anchor CM，data 与旧实现一致、不创建 canary/代资源；`helm lint` 与 schema 通过；默认渲染保持同名 anchor（空 data）、无新增工作负载模板、env 默认等效；Role 动词与实际调用一一对应、无多余、限命名空间
- **Pass Condition**: TR-5.3 回落单测、TR-11.1/11.2/11.3（helm lint/template 对比、动词清单、关闭组合渲染）
- **Evidence**: Task 5 TR-5.3、Task 11 TR-11.1/11.2/11.3

### AC-17: 失败诊断被保留且不覆盖任何代数据
- **Type**: `rule`
- **Given**: 任意阶段失败
- **When**: 写 last-failure 与 quarantine 标记
- **Then**: anchor last-failure 注解含阶段/原因/zone/查询摘要/时间戳；quarantine 代保留原因标签且最近 N 代可枚举；诊断写入后代 CM 与 anchor data 内容不变
- **Pass Condition**: 失败注入单测断言诊断字段完整且所有代数据字节未变（TR-8.2/4.2）
- **Evidence**: Task 4、Task 8

### AC-18: 并发健壮性（重启/冲突/连续变更自动收敛）
- **Type**: `rubric`
- **Dimension**: 在重启、乐观锁冲突、连续变更与滚动并发的故障注入下，状态机自动收敛到唯一最新代、无卡死/错误提升/双代并存的能力
- **Scale**: 1-5
- **Anchors**: 1 = 存在卡死、错误提升或双代并存需人工干预；3 = 常规正确，极端竞态需干预；5 = 任意重启/冲突/连续变更均自动收敛到唯一最新代且旧代安全回收
- **Pass Threshold**: >= 4
- **Evidence**: Task 8 TR-8.6（scale 1-5 故障注入单测 + 评审，记录得分与理由）

### AC-19: 集成质量（窄接口、侵入最小、测试自然）
- **Type**: `rubric`
- **Dimension**: 新分代包接入现有 ZoneDelegation 协调链路的边界清晰度与可维护性
- **Scale**: 1-5
- **Anchors**: 1 = 大面积侵入既有逻辑、难以测试；3 = 可用但边界模糊；5 = 窄接口接入、既有改动最小、测试自然、RBAC/装配一致
- **Pass Threshold**: >= 4
- **Evidence**: Task 9 TR-9.4（diff 结构评审，记录得分与理由）

## Open Questions
- 无阻塞性开放问题。候选 Service 经 ClusterIP DNS 与 servicePort 53→5353 映射探测、anchor data 镜像活动代两项已在 Assumptions 中明确；如实现中发现 coredns subchart 标签/端口与假设不符，回落 legacy（FR-16）保证安全。
