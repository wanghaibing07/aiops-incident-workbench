# AIOps Incident Investigation Workbench

基于 HolmesGPT 0.43.0 的 Kubernetes 故障调查实验。

## 已验证
- 自定义模型 API 与 Agent 工具调用
- namespace 级 Kubernetes 只读访问
- T1 Service Selector 故障调查及恢复
- T3 Gateway 与下游 HTTP 基线

## 已发现的限制
- HolmesGPT 内置查询工具存在命名空间范围问题
- 使用自定义 YAML 工具进行范围受限的取证
- T3 业务依赖故障调查仍待完成

## 当前工程选择
复用成熟 Agent，只有实验证明存在缺口时才开发 Python 扩展。
