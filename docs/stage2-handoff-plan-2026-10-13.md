# 项目二 Stage 2 规划与下一对话交接书（v1.0）

**日期**：2026-10-13  
**项目**：AIOps Incident Response Lab / [GitHub: aiops-incident-workbench](https://github.com/wanghaibing07/aiops-incident-workbench)  
**执行状态**：**Stage 1 已封板；Stage 2 尚未开始实操验收。**  
**主要依据**：用户《项目二 v3.0：AIOps 成熟方案集成与工程验证路线》（2026-10-08）、本次会话 LOG-001～LOG-054、GitHub 已公开文件、HolmesGPT/Kubernetes/GitHub 官方文档。  
**执行优先级**：真实求职价值 > 本人真正理解 > 复现与证据可信度 > 展示形式 > 新增功能。

> 给新对话的定位：**不是从零搭环境，不是重做 T1/T3，不是继续 Stage 1，也不是写第二套 Agent。** 应以既有仓库和封板事实为基线，用 2～3 步/轮带做 Stage 2。

## 一、决策来源和阶段命名

《项目二 v3.0》的原路线是 **P0 现场体检 → P1 接入成熟 Agent → P2 受控任务 → P3 真缺口才写 Python 增量 → P4 交付与止损**。P0～P4 是任务与决策闸门，**不等同于**会话采用的 Stage 编号；不能将 v3.0 误说成预先规定了 Stage 0～8。

### 与已完成工作的映射

| v3.0 路线 | 已有事实 | 处理决定 |
|---|---|---|
| P0 环境核验 | 已在旧集群和独立开发机完成基础接入；用户曾主动关闭一个 worker 节约资源 | 不重做安装 |
| P1 HolmesGPT 与只读 Kubernetes | HolmesGPT 0.43.0、模型工具调用、namespace 限定的 ServiceAccount/工具成功 | 已满足 |
| P2 受控业务故障 | T1 Selector；T3 端口 81；HTTP 验收与人工恢复 | 已满足 |
| P3 按实际缺口补工具 | 第三项 Gateway 日志只读 YAML 工具使 Agent 能定位 :81 ≠ :80 | 有实际增量；**无必要不编写 Python 重复工具** |
| P4 项目交付 | GitHub/中文 README/架构/T1/T3 对照/最小 YAML/面试草稿已发布 | 仍缺“别人如何运行”、本人独立理解验收；归 Stage 2 精简解决 |

**Stage 1**：Agent 集成和真实受控故障验证，已正式封板；[封板报告](stage1-closeout-2026-10-13.md)。  
**Stage 2**：独立掌握、最小复现说明和求职成果转化；**本规划是首版范围，不强制向 Stage 3～8 拓展**。

## 二、实际现状：已经有什么

### 机器与权限

- **k8s-master**：现有 Kubernetes v1.36.4 管理节点，已有 `/etc/kubernetes/admin.conf`；**只有人工故障注入和恢复**使用管理员身份。
- **aiops-dev01**：CentOS Stream 9；日常用户 `aiops`，项目目录 `~/aiops-incident-lab`；`uv` / Python 3.12，`kubectl` 与 HolmesGPT 0.43.0；本地启动封装 `~/.local/bin/holmes-lab`。
- Agent 只读身份：`system:serviceaccount:incident-lab:holmes-investigator`；开发机 kubeconfig `~/.kube/incident-lab.kubeconfig`。Token 为会过期的临时凭据，不在仓库里、不在聊天中公开。
- 现场隔离：项目二使用 `incident-lab`，与 Retail 项目隔离；用户曾让 Codex 缩容 Retail，实际命令误触及 `incident-lab`，造成两个 Deployment 为 0；其后已恢复。未来禁止跨 namespace 的批量修改。
- 过去 GitHub SSH 22 端口不通，已经改用 `ssh.github.com:443`，可用 SSH 密钥推送。**不要再次安装 `gh` 或重建 Git 仓库**。

### 应用调用链

`HTTP → Service/t3-gateway:8080 → Nginx 网关 → Service/t1-web:80 → Web Pod`

- 两个 Nginx Deployment 各 1 副本（历史成功验收，不代表新对话实时在线状态）。
- `t1-web` 标签/Selector `app=t1-web`，Service 端口 80。
- `t3-gateway` 标签/Selector `app=t3-gateway`，Service 端口 8080。
- Gateway `/healthz` 独立返回健康结果；`/api/` 经 Nginx `proxy_pass` 依赖下游。
- `t3-gateway-config` ConfigMap 经 `subPath` 挂载，修改后旧容器不会自动获得新内容；过去实验使用 `rollout restart`。
- 工具文件：[incident-lab-readonly.yaml](../toolsets/incident-lab-readonly.yaml)，三项工具分别读取 Pod 摘要、Service/EndpointSlice 路由、网关最近 15 分钟日志；固定 namespace，不接收自由命令参数。

### 已验证的证据链

| 记录 | 已核实结论 |
|---|---|
| LOG-034～036 | T1：Pod Ready、Service Selector 不匹配、EndpointSlice 无后端；Agent 识别，人工恢复 HTTP 200 |
| LOG-038～046 | 部署副本误缩容导致的额外事故已调查并恢复，**不计入受控 T3 成功实验** |
| LOG-048 | T3：网关 /healthz=200、/api/=502；原两个工具仅定位上游链路，无法确定具体端口 |
| LOG-050 | 加入第 3 个日志工具后，Agent 从 Nginx 日志中读到上游 :81，对照 Service :80，正确定位错误端口；并未直接读取配置文件 |
| LOG-051 | ConfigMap 与 Gateway 重建恢复，/healthz=200、/api/=200 |
| LOG-052～054 | README、T1 文档、复现 YAML 等推送到 GitHub；六对象 YAML 客户端解析通过，未做全新集群部署 |

证据入口：[T1](t1-investigation.md)｜[T3 对照](t3-investigation.md)｜[架构](architecture.md)｜[面试草稿](interview-guide.md)｜[Stage 1 封板](stage1-closeout-2026-10-13.md)。

## 三、未完成之处：真实缺口，不偷换成新需求

**必须诚实写清的限制：**

1. **仓库不能端到端开箱即用**：虽然有六对象业务清单和 YAML 工具集，公开仓库还缺可信的最小运行说明、完整非秘密 RBAC、启动封装说明；安装与模型 API 凭据仍是本地配置。
2. **复现 YAML 验证级别有限**：`kubectl create --dry-run=client --validate=false` 只证明能解析；没有新集群部署、服务端 Schema 和与原环境逐字段一致性验证。
3. **Python 项目目前偏骨架**：`pyproject.toml` 有依赖与 CLI 声明，但 GitHub 上没有已证明可工作的独立 Python Agent/脚本与测试；**不把它写成已经实现的 Python 运维平台**。
4. **独立操作能力未验收**：已有的 [interview-guide.md](interview-guide.md) 是面试表达草稿，用户是否能脱稿解释 Selector、EndpointSlice、readiness、Nginx 502、Agent 证据来源仍未知。
5. **实验数量有限**：没有统一的 Codex/K8sGPT 横向公平对照、统计测评、生产长期稳定性或自动修复；不据此虚构准确率和 MTTR 数据。
6. **现场实时状态会变**：后续只在实际需要在线演示时读取最新状态；历史的 HTTP 200 不代表当前总是 200。短时 Token 可能过期。

**不是本阶段必须补齐的项目**：新的 T2/T5、多故障基准库、FastAPI、数据库、RAG、Operator、多 Agent、Grafana、自动修复、长稳压力测试、云上重建、全新集群重新安装、完整可观测平台。

## 四、Stage 2 工作包与最小验收

### S2-A：本人独立理解与排障能力验收（先做）

**目的**：从“跟着 AI 执行”转到能自主解释、检查、判断。以 2～3 个很短的步骤为一批，一次只引入一个关键概念。

- **S2-A01**：同步 GitHub 最新 `main`，阅读 [架构](architecture.md) 与 [Stage 1 封板](stage1-closeout-2026-10-13.md)；用自己的话回答两个问题：① Pod Ready 为什么 Service 仍不可用？② 原 Agent 和增强 Agent 对 T3 的证据差在哪里？
- **S2-A02～**：根据真实回答缺口，最多补少量 `kubectl get pods,svc,endpointslices`、`logs`、`describe` 的只读演练；理解 Selector → EndpointSlice → HTTP → Nginx 错误日志的因果链；无需重新故障注入。
- **验收**：不看稿说明 T1/T3 的症状、证据、排除过程、Agent 功能边界和恢复原则；能解释关键命令为什么用，并能纠正给出的一个错误判断。
- **拒收情形**：照念文档而不能解释“就绪与业务可用不同”“日志 :81 对比 Service :80”或把 AI 的推断当作直接读取配置。

### S2-B：最小可解释交付（按真实需求精简）

**目的**：让审阅者理解**怎样复用**这个集成项目，而非追求生产级安装器。

- **S2-B01**：写一页 `docs/runbook.md`，说明既有环境上的运行入口、三个工具来源、调查示例、授权边界、演示/恢复流程和常见失败（Token 到期、两个 Deployment 被缩容、GitHub SSH 22 端口）；**只写已经实测过的命令**，不复制 API Key/Token。
- **S2-B02（条件触发）**：如果公开仓库必须能让别人独立复现，再选择性添加最小命名空间 Role/RoleBinding/ServiceAccount 清单与脱敏示例，或修复失效的 Python CLI 声明；否则公开写清“实验环境已验证、公开复现仍有限”即可。
- **验收**：陌生读者从 README 找到架构、T1/T3 证据、运行前提和限制；用户能解释哪些配置可公开，哪些凭据必须本地提供。
- **不做**：新建完整集群、复制管理员 kubeconfig、自动安装脚本、打包镜像和不必要 CI 套件。除非面试/协作者确实需要，不为“看起来完整”强行部署新环境。

### S2-C：成果转化、求职应用与 Stage 2 收口

**目的**：服务深圳 Linux / 云平台运维、NOC/IDC、运维开发等实际岗位；项目一 Retail 可靠性实验室仍是 Linux/K8s/发布监控主项目，项目二用于补充 Agent 工具集成的差异化。

- **S2-C01**：根据 [interview-guide.md](interview-guide.md) 产出简历最有说服力的 **3～4 条事实约束项目经历**，避免写“生产自动修复”“完全自主构建 Agent”“统计提升 xx%”。
- **S2-C02**：练习 **1 分钟和 3 分钟**讲解，优先问答“为什么先查 Service/EndpointSlice”“为什么 readiness=200 不等于 API=200”“第三工具究竟提供了什么额外证据”“为什么不写 Python Agent”。
- **S2-C03**：检查 GitHub 页面、材料和本人理解一致；列出未证实内容，完成 Stage 2 封板或保留真正的短板清单。
- **验收**：用户能独立讲清案例及边界，简历陈述每条可回溯至 GitHub 和日志；不用编造性能、业务收益或生产资历。

## 五、执行优先级、停机规则和资源取舍

**必须做（现在）**：S2-A 独立理解；基于已有证据整理能讲明白的项目经历；README 保持与真实结论一致。

**条件做**：S2-B 运行手册/RBAC/最小可复现；只有在当前材料无法满足审阅或真实演示时补缺；出现“已实现入口实际上跑不通”则先修复误导。

**暂不做**：Python 通用 Agent、第二套工具循环、复杂基准、T1/T3 反复重新注入；同一个问题已有证据就停止继续搜索和测试。

**快速决策规则**：
1. 先找真实阻塞或能力缺口，不预设要开发。
2. 能通过配置、日志、文档或现成工具解决，就不写重叠代码。
3. 需要在线验证时，使用最小可逆操作和精准 namespace；不动已封板 Retail。
4. 每轮最多 2～3 步，命令标明 **机器 / 用户 / 关键参数 / 预期输出**。
5. 失败即记录 `NOT VERIFIED`，不为了赶进度虚构 PASS；Stage2 编号 `S2-A01`、`S2-B01` 等，LOG 从 **LOG-055** 接续。

## 六、交付与完成标准

首版 Stage 2 满足以下条件即可停止：
- 已有 T1/T3 实际实验链路保持可核查；
- 用户独立回答关键排障原理，能面对至少一次非照稿追问；
- README / 证据 / 架构 / 运行限制 / 简历陈述互相一致；
- 未完成的复现、Python CLI、Token 时效、模拟环境限制如实标注；
- 不以项目继续打磨为由推迟真实求职。

成功时保存最终短报告、当前 GitHub commit、技术总结及面试要点；如无法独立解释，先补相应知识而非增加组件。

## 七、给新对话的启动步骤（严格 2 步）

**第 1 步，aiops-dev01 / aiops**（仅当用户准备实际动手）：同步本轮在 GitHub 上发布的文档，不要重新 `git init` 或安装 HolmesGPT。

```bash
cd ~/aiops-incident-lab
git pull --ff-only origin main
git status -sb
git log -1 --oneline
```

**第 2 步，理解检查**：阅读 [architecture.md](architecture.md) 和 [stage1-closeout-2026-10-13.md](stage1-closeout-2026-10-13.md)，脱稿回答：

- “`t1-web` Pod 已 Ready，为什么 Service 仍可能访问失败？如何用证据证明？”
- “T3 原版与增强版 HolmesGPT 分别实际看到了什么？为什么只有增强版能指出错误端口？”

根据回答针对性练两步，不强行进入下一工作包。

## 八、可直接复制到下一对话的交接指令

> 继续「AIOps Incident Response Lab / aiops-incident-workbench」项目。Stage 1 已正式封板，不要重建环境或重跑 T1/T3。请先阅读 GitHub 仓库 `wanghaibing07/aiops-incident-workbench` 的 `docs/stage1-closeout-2026-10-13.md` 和 `docs/stage2-handoff-plan-2026-10-13.md`，结合 README、T1/T3 和工具 YAML，以 GitHub 最新 main 与 LOG-054 为基线。现在开始 Stage 2，先做 S2-A01 本人独立理解与排障能力验收，每轮只安排 2～3 步；实操命令必须写主机、用户、目的和预期现象。按 `Stage → S2-A / S2-B / S2-C → S2-A01` 编号，LOG 从 055 接续。首先解释当下最值得做的两步，而不是重新规划所有组件。对已验证、未验证和判断明确区分；优先求职和理解，拒绝过度设计。每次答复最后用“说白了就是：……”收束。

## 九、资料与来源

- 项目内部：[Stage 1 封板](stage1-closeout-2026-10-13.md) · [T1](t1-investigation.md) · [T3](t3-investigation.md) · [架构](architecture.md) · [面试草稿](interview-guide.md) · [业务清单](../k8s/incident-lab.yaml)
- 本规划基线：用户《项目二 v3.0：AIOps 成熟方案集成与工程验证路线》（2026-10-08）；不是可公开链接，不臆造其中没有的 Stage 2～8 详细定义。
- [HolmesGPT 0.43.0：Custom Toolsets](https://holmesgpt.dev/0.43.0/data-sources/custom-toolsets/)
- [GitHub README 编写建议](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)
- Kubernetes 参考：[Service](https://kubernetes.io/docs/concepts/services-networking/service/) · [EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) · [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) · [kubectl](https://kubernetes.io/docs/reference/kubectl/)

**说白了就是：Stage 1 已把 Agent 排障功能跑通并用真实故障证明价值；Stage 2 主要要让人能读懂、能复核、你自己能讲会用，并最终服务于实际求职。**
