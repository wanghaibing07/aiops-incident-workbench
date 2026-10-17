# AI SRE 技术复盘：Kubernetes 故障调查与 Agent 工具扩展

本项目在独立 Kubernetes 实验命名空间中，通过 HolmesGPT 0.43.0 调查两个受控故障，使用固定范围的只读工具获取证据，最终由人工恢复服务。

## T1：Service Selector 不匹配

Pod Ready，但 Service Selector 与 Pod 标签不一致，EndpointSlice 无可用后端。Agent 读取 Kubernetes 资源后正确定位问题；人工恢复 Selector，HTTP 恢复 200。

[详细记录](t1-investigation.md)

## T3：Nginx 网关上游端口错误

网关健康检查返回 200，但业务 API 返回 502。下游 Service 正常暴露端口 80，注入错误后网关使用端口 81。

| 调查能力 | 原版两工具 | 增加日志工具后 |
|---|---|---|
| Pod 和 Service 状态检查 | 成功 | 成功 |
| 判断故障范围 | 成功 | 成功 |
| 读取 Nginx 错误日志 | 不支持 | 成功 |
| 识别 81 与 80 不匹配 | 未成功 | 成功 |

增强版 Agent 自主调用三个只读工具，依据日志里的上游端口与 Service 端口对照给出判断。恢复网关配置后两个接口均返回 200。

[详细对照实验](t3-investigation.md) · [自定义工具](../toolsets/incident-lab-readonly.yaml)

## 技术取舍与边界

- 复用成熟 Agent，仅补充真正缺少的日志取证能力。
- 故障注入和恢复由人工执行；Agent 无自动恢复功能。
- Agent 没有直接读取 Nginx 配置，判断依据是日志与 Service 信息。
- 这是有限案例的实验观察，不能据此推断生产环境可靠性或统计精度。
- [最小 Kubernetes 资源文件](../k8s/incident-lab.yaml) 已通过客户端解析，未完成新集群部署验收。

[架构说明](architecture.md)
