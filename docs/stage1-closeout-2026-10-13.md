# Stage 1 封板报告｜AIOps Incident Investigation Workbench

- **封板日期**：2026-10-13
- **项目**：AIOps Incident Response Lab / GitHub 仓库 `aiops-incident-workbench`
- **状态**：**Stage 1 核心目标已达成，正式封板**；这不等于所有计划中的工程能力均已实现。
- **现场证据范围**：LOG-001～LOG-054；本报告重点复核 LOG-034～054 与已公开的 GitHub 文档。
- **封板原则**：保留已核验的事实、清楚指出欠缺；不为凑功能开发通用 Agent，也不重复注入已通过的故障。

## 1. Stage 1 的实际目标与规划映射

原《项目二 v3.0》（2026-10-08）采用 **P0～P4 决策路线**，**并未事先定义 Stage 0～8 全部编号**。会话执行记录采用 Stage / 工作包 / Attempt 编号。这里把已执行的 Agent 集成、T1/T3 故障调查和初步公开展示统合为 **Stage 1**；之后新建 **Stage 2：成果验收与求职转化**，不回头重编号历史日志。

v3.0 的核心取舍：**成熟 HolmesGPT + 小规模真实业务故障 + 只读证据工具 + 增强前后对比**；若成熟工具能完成任务就复用，避免 Python 重复造轮子；实际求职与项目并行。

## 2. 封板验收

| 验收项 | 关键证据 | 结论 |
|---|---|---|
| HolmesGPT / 模型接入 | v0.43.0；OpenAI 兼容模型渠道；实际工具调用成功 | PASS |
| 调查权限边界 | `incident-lab` 的 ServiceAccount `holmes-investigator`；三项固定命名空间只读工具；实际调查未观察到写操作 | PASS（仅验证所述范围） |
| T1 故障调查 | Pod Ready；Service selector `app=t1-web-fault` vs Pod `app=t1-web`；EndpointSlice 无后端；Agent 判断正确；人工恢复 HTTP 200 | PASS |
| T3 原版调查 | 网关 /healthz=200、/api/=502；原版两工具只能定位网关上游链路，不能证明具体错误端口（LOG-048） | PASS（诚实记录缺口） |
| T3 工具增强 | 新增网关日志读取工具；相同故障 80→81；Agent 自主调用三工具，通过日志中的 :81 和下游 Service :80 识别端口不匹配（LOG-050） | PASS |
| 故障恢复闭环 | ConfigMap 恢复端口 80，滚动重建 Gateway，业务 /api/ 与 /healthz 均 HTTP 200（LOG-051） | PASS |
| 项目公开与文档 | 中文 README、T1/T3 文档、工具集 YAML、架构说明、面试材料、GitHub 推送 | PASS |
| 最小 Kubernetes 清单 | 6 个对象；`kubectl create --dry-run=client --validate=false` 解析成功（LOG-054） | PASS（仅客户端解析） |

### 确认的工程增量

- **增强前**：Pod、Service、EndpointSlice 资源证据可以缩小 T3 故障范围，但不能锁定端口。
- **增强后**：在原工具集增加 `incident_lab_gateway_error_logs`，Agent 对照日志与 Service 端口成功指出 **81 ≠ 80**。
- 这是**本实验的一次定性对照**，不是样本量充分的准确率提升统计，也不代表普遍提升 MTTR。

### 对照失败、意外问题与修复经验

- 在 LOG-038～046 的调查中，多次碰到两个 Deployment 副本为 0；用户提供的 Codex 执行记录证实曾对 `incident-lab` 使用 `scale deployments --all --replicas=0`。恢复两服务后 HTTP 200。**这属于外部批量操作误伤，不计入 T3 预设故障成功次数。**
- Deployment rollout 成功不等于 Service HTTP 立即就绪。Gateway 重建后，首次请求可能出现 000/connection refused，后续返回 200/502；实测需要单独观察业务链路。
- 短时 Kubernetes Token 过期曾造成 Agent 认证失败；LOG-045 刷新后 `kubectl auth whoami` 恢复成功。
- GitHub SSH 22 端口不可用，改用官方 SSH 443 后推送成功；本地 `gh` 未安装不是必要阻塞。

## 3. 封板成果与证据入口

- [README / 项目主页](../README.md)
- [T1 故障记录](t1-investigation.md)
- [T3 增强前后调查对照](t3-investigation.md)
- [HolmesGPT 自定义只读工具集](../toolsets/incident-lab-readonly.yaml)
- [Kubernetes 最小业务拓扑 YAML](../k8s/incident-lab.yaml)
- [调查架构与边界](architecture.md)
- [简历与面试讲解草稿](interview-guide.md)

**现场边界**：核心原始终端 Agent 调用记录和完整日志并未全部作为 GitHub 原始附件保存；公开文档是有日志支持的事实摘要。现有 Python 项目主要为 `uv` 初始化骨架；主要本人可说明的实现成果是 **HolmesGPT YAML 工具集、K8s 场景和调查方法**，不能描述成已开发完整 Python Agent。

## 4. 明确没有完成或没有证明的部分

1. **端到端一键复现未完成**：仓库有业务 YAML，但缺可公开复用的完整 RBAC/Agent 安装与启动说明、凭据自动获取流程；环境本地封装命令 `~/.local/bin/holmes-lab` 尚未入库。
2. **新集群部署验证未做**：清单根据 LOG-053 配置重建，不是逐字段导出；`--validate=false` 不等于服务端 Schema 校验。
3. **未证明 Agent 直接读取 Nginx ConfigMap 内容**：T3 的正确判断依据是 Nginx 日志的上游端口与 Service 端口对比。
4. **未实现 Python 独立取证模块 / 自动恢复 / 持续值班 / 生产级安全审计或长稳评测**；当前没有证据证明做这些会带来更高收益。
5. **Codex/K8sGPT 的统一条件横向对照未完成**；v3.0 中属“条件允许时”的参照，并非本轮结论前提。
6. **本人独立讲解尚未验收**：GitHub 面试材料是草稿，不代表已掌握每个命令、配置项和排障原理。

## 5. 正式封板决定

- **Stage 1：CLOSED / PASS（按收敛后的核心实验范围）**。
- T1/T3 受控注入、Agent 调查和人工恢复不再为了“多几次证明”而重跑。
- 不扩大到 FastAPI、Redis、数据库、Operator、多 Agent、RAG、可视化平台、自动恢复、生产化评测或完整自主 Agent runtime。
- 如 Stage 2 为了实际可复现/面试暴露明确缺口，可做**最小、可解释、能验收**的增量，必要时重新开启相应工作包；不得悄悄篡改 Stage 1 结论。
- 下一阶段方案：[Stage 2 规划与新会话交接书](stage2-handoff-plan-2026-10-13.md)。

> **说白了就是：Stage 1 已证明“小范围真实故障 + HolmesGPT 只读取证 + 精准增加日志工具”可帮助定位问题；完整平台化、普遍准确率以及个人独立掌握程度仍未证明。**
