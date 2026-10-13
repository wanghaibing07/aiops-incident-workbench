# AIOps Incident Investigation Workbench

基于 HolmesGPT 的 Kubernetes 故障调查实验，重点验证
AI Agent 能否利用真实集群证据定位故障，以及工具扩展
能否提升根因分析的准确性。

## 项目概况

- Kubernetes：v1.36.4
- Agent：HolmesGPT 0.43.0
- 模型：GPT-5.6 Luna（OpenAI 兼容接口）
- 调查范围：独立的 `incident-lab` 命名空间
- 工具能力：Pod、Service、EndpointSlice、Gateway 日志
- 操作边界：Agent 只读调查，故障注入和恢复由人工执行

## 已完成案例

### T1：Service Selector 不匹配

Pod 正常运行，但 Service 选择器与 Pod 标签不匹配，
导致 EndpointSlice 没有可用后端。

Agent 根据 Kubernetes 资源证据定位问题，人工修复后恢复 HTTP 200。

[查看 T1 调查记录](docs/t1-investigation.md)

### T3：Nginx 上游端口错误

将 Gateway 的上游端口从 80 修改为 81：

- Gateway `/healthz`：HTTP 200
- Gateway `/api/`：HTTP 502
- 下游 Pod 与 Service：正常

首次调查时，Agent 能定位故障范围，
但无法确定具体错误端口。

补充 Gateway 日志工具后，Agent 自主读取日志，
识别出错误端口 81 与 Service 端口 80 不匹配。

恢复配置后，业务重新返回 HTTP 200。

[查看 T3 对照实验与证据](docs/t3-investigation.md)

## 工具设计

通过自定义 YAML 工具集扩展 HolmesGPT：

1. `incident_lab_pod_snapshot`：读取 Pod 状态
2. `incident_lab_service_route_evidence`：读取服务路由证据
3. `incident_lab_gateway_error_logs`：读取 Gateway 日志

工具定义见：
[incident-lab-readonly.yaml](toolsets/incident-lab-readonly.yaml)

## 工程结论

优先复用成熟 Agent，通过真实故障实验识别工具缺口。

在 T3 案例中，新增一个日志读取工具，
就让 Agent 从故障范围定位提升到了具体错误端口定位。

目前验证的是实验环境下的调查能力，
不代表生产环境自动恢复能力。

本仓库已包含工具定义与 T3 证据；
完整的环境复现清单仍待整理。
