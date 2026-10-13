# AIOps Incident Investigation Workbench

基于 HolmesGPT 的 Kubernetes 故障调查实验，重点验证
AI Agent 能否利用真实集群证据定位故障，以及工具扩展
能否提升根因分析的准确性。

> **进度**：Stage 1（HolmesGPT 集成、T1/T3 受控故障验证）已封板。Stage 2 从独立理解、运行说明和求职转化开始，不重新搭建 Agent。


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

## 最小复现资源

[查看 Kubernetes YAML](k8s/incident-lab.yaml)

包含 Namespace、两个 Deployment、两个 Service
及 Gateway ConfigMap。

该配置依据实验环境参数重建，尚未在全新集群
完成部署验收。

Agent 的只读 RBAC 与认证凭据不包含在此文件中。

## 架构与面试讲解

- [项目架构、只读调查边界和复现限制](docs/architecture.md)
- [简历项目经历与 1/3 分钟面试讲解草稿](docs/interview-guide.md)

## 阶段归档与后续计划

- [Stage 1 封板报告：已验证事实与未完成边界](docs/stage1-closeout-2026-10-13.md)
- [Stage 2 规划与下一对话交接书](docs/stage2-handoff-plan-2026-10-13.md)
