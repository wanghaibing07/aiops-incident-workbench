# 项目进度

## Stage 1：Agent 故障调查验证

状态：核心实验完成。

### 已完成
- HolmesGPT 模型及工具接入
- incident-lab 命名空间只读调查
- T1 Service Selector 故障调查与恢复
- T3 Gateway 上游端口故障调查与恢复
- T3 增强前后 Agent 能力对比
- GitHub 远程仓库及证据文档

### 已验证的工程结论
- Kubernetes 资源信息可以支持 T1 故障定位。
- T3 原有工具不足以确认具体错误端口。
- 增加日志取证工具后，Agent 识别出 81 与 80 不匹配。
- Agent 保持只读，恢复操作由人工完成。

### 尚未完成
- T1 案例证据整理
- 最小可复现 Kubernetes 资源清单
- 项目最终展示与面试材料

### 首版决策
暂不开发重复功能的 Python Agent 工具。
