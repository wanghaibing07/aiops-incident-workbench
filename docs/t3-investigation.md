# T3：Gateway 上游端口故障调查

## 实验环境
- Kubernetes namespace: incident-lab
- Agent: HolmesGPT 0.43.0
- Model: openai/gpt-5.6-luna
- 故障类型：Nginx 上游端口错误

## 故障注入
- 正常上游端口：80
- 注入上游端口：81
- Gateway /healthz：200
- Gateway /api/：502

## Agent 对照结果

| 能力 | 原版（2 个工具） | 增强版（3 个工具） |
|---|---|---|
| 读取 Pod 状态 | 成功 | 成功 |
| 读取 Service / EndpointSlice | 成功 | 成功 |
| 读取 Gateway 日志 | 未提供 | 成功 |
| 定位故障范围 | 成功 | 成功 |
| 识别错误端口 81 | 失败 | 成功 |
| 与 Service 端口 80 对比 | 未完成 | 成功 |

## 关键证据
- 原版 Agent 定位到 Gateway 上游链路，但未确定具体错误。
- 增强版 Agent 自主调用 3 个只读工具。
- Nginx 日志显示请求的上游地址使用端口 81。
- Kubernetes Service 实际暴露端口 80。
- Agent 根据证据判断上游端口不匹配。
- Agent 未直接读取 Nginx 配置文件。

## 恢复
- 恢复 ConfigMap 中的正确上游端口 80。
- 重新创建 Gateway Pod。
- /healthz：200。
- /api/：200。

## 结论
新增一个针对性日志取证工具，使 Agent 从故障范围定位提升到具体错误端口定位。首版暂不开发重复功能的 Python 工具。
