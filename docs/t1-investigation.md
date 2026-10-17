# AI SRE 案例 T1：Service Selector 不匹配

## 故障场景
Kubernetes namespace：incident-lab

- Deployment：t1-web
- Pod 标签：app=t1-web
- 故障 Service Selector：app=t1-web-fault
- Pod 处于 Ready 状态
- EndpointSlice 没有可用后端
- 通过 Service 请求业务失败

## HolmesGPT 调查

Agent 自主调用两个只读工具：

1. incident_lab_pod_snapshot
2. incident_lab_service_route_evidence

调查发现：
- Pod 本身正常
- Service Selector 与 Pod 标签不一致
- EndpointSlice 缺少后端

Agent 正确定位 Service 选择器错误。
调查期间未执行资源修改。

## 人工恢复
- 将 Service Selector 恢复为 app=t1-web
- EndpointSlice 恢复可用后端
- HTTP 请求恢复为 200

## 结论
T1 可以通过 Kubernetes 资源状态完成故障定位，
不需要额外增加日志取证工具。

本文件为实验结果摘要，原始终端调查记录尚未作为附件发布。
