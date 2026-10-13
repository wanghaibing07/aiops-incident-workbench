# 项目讲解草稿（求职面试用）

> 依据已完成的 T1/T3 实验记录整理。请先用自己的话复述，遇到不能解释的命令或概念再补练习。不要将实验环境描述为生产环境。

## 简历项目经历（选择其中 3～4 条）

- 在三节点 Kubernetes v1.36.4 实验集群中搭建隔离的 `incident-lab` 故障场景，部署 Nginx 网关、下游 Service 与 ConfigMap 配置。
- 接入 HolmesGPT 0.43.0 和模型 API，配置固定命名空间的只读取证工具，调查 Pod、Service、EndpointSlice 与网关日志。
- 复现 Service Selector 不匹配故障：Pod Ready，但 Service 无可用后端；Agent 根据标签与 EndpointSlice 证据定位问题。
- 复现 Nginx 错误上游端口故障：`/healthz` 返回 200、`/api/` 返回 502；首次 Agent 仅缩小故障范围，补充日志工具后识别 81 与 Service 端口 80 不一致；人工恢复后 API 返回 200。

## 1 分钟讲解

我做了一个 Kubernetes 故障调查实验，重点不是自己重新写一个 AI Agent，而是验证现成的 HolmesGPT 能不能通过真实证据定位问题。我在独立命名空间部署了 Nginx 网关和下游服务，设计了两个故障：一个是 Service 选择器不匹配，一个是网关上游端口错误。第一个问题，Agent 能靠 Pod 标签、Service 和 EndpointSlice 定位；第二个问题，原有工具只能判断上游链路有问题，无法指出具体端口。我补充了一个只读日志工具后，Agent 从 Nginx 日志里识别出访问了 81 端口，而下游 Service 实际是 80 端口。最后由人工恢复业务并验证 HTTP 200。这个项目让我理解了 AI 排障的关键是证据获取和权限边界，而不只是模型回答。

## 3 分钟展开思路

**背景和目标：** 在已有 Kubernetes 集群上构造两个小型、可控的业务故障，验证 HolmesGPT 的独立取证和判断能力。Agent 只读，人工负责注入及恢复。

**场景一（T1）：** `t1-web` Pod 仍 Ready，但把 Service Selector 改成无法匹配的标签后，EndpointSlice 不再包含可用后端。Agent 读取资源后将 Service Selector 与 Pod 标签进行对比，定位故障。人工恢复选择器后 HTTP 200。

**场景二（T3）：** Nginx Gateway 的 `/healthz` 自身正常，并不代表 `/api/` 的下游链路正常。我把代理端口从 80 改为 81 并重建 Gateway Pod，观察到健康检查 200、API 502。最初两个工具只能确认 Gateway 与下游的 Pod、Service 和 EndpointSlice 正常，Agent 判断故障可能在上游配置或连通性，但无法证明是哪个端口。

**改进和结果：** 没有另起一套 Agent，而是在 HolmesGPT 自定义 YAML 工具集中增加固定的 Gateway 日志读取工具。第二轮相同故障下，Agent 自主调用三个工具，通过 Nginx 日志中的上游目标 `:81` 与 Service `:80` 对比，定位具体错误端口。人工恢复 ConfigMap 并重建 Gateway Pod 后，API 恢复 200。

**边界和反思：** Agent 并未直接读取网关配置文件，也未执行自动恢复；一次对照实验能证明这次任务中的能力改善，不能据此声称在所有故障上都更准确。项目重点是面向真实故障的最小工具扩展和证据链。

## 面试官可能追问

**Pod Running / Ready，为什么仍可能无法访问 Service？**  
Service 使用 Selector 选择 Pod，真正被路由到哪些地址要看 EndpointSlice；Pod Ready 不代表 Selector 一定匹配。

**为什么网关健康检查 200，API 仍然 502？**  
`/healthz` 在 Gateway 内部直接返回，`/api/` 还依赖代理到下游服务；两条路径检查的范围不同。

**如何证明 AI 真的有改进，而不是提前知道答案？**  
两轮使用同一故障描述，不向 Agent 提供错误端口；前一轮只读两个工具，后一轮新增日志工具，观察工具调用与具体错误端口结论。

**为什么不让 Agent 自动修复？**  
此阶段只验证独立取证和判断。故障注入与恢复使用人工管理身份，避免把读与写的权限混在一起。
