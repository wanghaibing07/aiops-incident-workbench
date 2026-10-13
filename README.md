# AIOps Incident Investigation Workbench

在 Kubernetes 实验环境中，使用 HolmesGPT 调查真实注入的服务故障，重点研究：**当 Kubernetes 资源看起来正常，但业务请求失败时，Agent 能否通过工具获取足够证据定位问题？**

项目复用 HolmesGPT 的调查引擎，围绕实际证据缺口增加一项日志工具；不自称从零开发了 Agent。

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

## 可复现程度与局限

- Kubernetes 资源 YAML 已通过 `kubectl create --dry-run=client --validate=false` 的客户端解析；**未在全新集群做部署验收**。
- 当前仓库公开的是业务配置与 HolmesGPT 工具定义，尚非一键部署的完整 Agent 环境；使用者需要自行配置 HolmesGPT、Kubernetes 访问身份和模型服务。
- Agent 没有直接读取 Nginx ConfigMap 内容，也没有执行自动恢复。该实验仅能支持所述两个案例的观察结果，不代表生产环境中的普遍准确率或故障恢复时间改善。

## 工程取舍

本项目选择优先复用成熟调查框架，用少量受控故障验证其实际能力；在 T3 暴露出日志证据不足后增加一个专用工具，而不是重建整个 Agent 系统。
