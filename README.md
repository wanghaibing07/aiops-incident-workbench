# AI SRE Incident Investigation Workbench

**让 AI Agent 协助 SRE 排查 Kubernetes 服务故障。** Agent 自动读取工作负载、服务路由和网关日志，分析故障证据、定位错误；工程师根据调查结果执行恢复并验证业务。

基于 HolmesGPT 的 AI SRE 故障调查工作台。重点解决的问题：**Pod 看起来正常，业务接口却失败时，如何从 Kubernetes 资源和日志中找到真正的故障点？**

## 故障调查流程

```text
业务请求异常 → Agent 只读取证 → 对照服务路由与日志 → 定位故障 → 人工恢复 → HTTP 验证
```

AI SRE 在本项目中的实际能力是**辅助故障诊断**：自动调查、证据关联和原因判断；变更与恢复仍由工程师执行。

## 架构

```text
业务请求 → Service/t3-gateway:8080 → Nginx Gateway
                                   → Service/t1-web:80 → Web Pod

HolmesGPT → namespace 限定的只读取证工具 → Kubernetes API / 网关日志
```

实验环境：Kubernetes v1.36.4、HolmesGPT v0.43.0、Nginx 1.25，命名空间为 `incident-lab`。Agent 仅调查，故障注入和恢复由人工执行。

## 真实故障案例

| 案例 | 故障现象 | Agent 调查结果 |
|---|---|---|
| [T1：Service Selector 不匹配](docs/t1-investigation.md) | Pod Ready，但 Service 没有可用后端 | 根据 Pod 标签、Service Selector、EndpointSlice 定位匹配错误 |
| [T3：Gateway 上游端口错误](docs/t3-investigation.md) | `/healthz` 为 200，`/api/` 为 502 | 原有工具仅定位故障范围；增加日志工具后识别端口 81 与 Service 端口 80 不匹配 |

T3 的两轮调查使用相同故障描述，Agent 自主调用工具。**增强前只判断问题范围，增强后通过日志与 Service 端口对照识别具体错误。** 人工恢复配置后，业务返回 HTTP 200。

[查看技术复盘与对照结果](docs/technical-report.md)

## 仓库内容

- [toolsets/incident-lab-readonly.yaml](toolsets/incident-lab-readonly.yaml)：三个固定范围的只读调查工具，分别读取 Pod、服务路由和 Gateway 日志。
- [k8s/incident-lab.yaml](k8s/incident-lab.yaml)：Namespace、两个 Deployment、两个 Service 与 Gateway ConfigMap。
- [docs/architecture.md](docs/architecture.md)：业务调用链、取证过程与边界。
- [docs/technical-report.md](docs/technical-report.md)：T1/T3 关键证据、实验结果及工程取舍。

## 复现与验证

- Kubernetes 资源 YAML 已通过 `kubectl create --dry-run=client --validate=false` 客户端解析；全新集群部署验收尚未进行。
- 仓库提供业务配置、只读调查工具和案例记录。复现时需配置 HolmesGPT、Kubernetes 访问身份与模型服务。
- 两个案例的调查结果已有记录；Agent 未直接读取 Nginx ConfigMap，也未自动恢复服务。生产级可靠性与长周期指标仍需另行评估。

## 设计重点

复用 HolmesGPT 进行调查；围绕 T3 故障补充 Nginx 日志取证工具，使 Agent 从定位异常链路进一步识别错误端口。调查权限限定为只读，恢复操作由工程师控制。
