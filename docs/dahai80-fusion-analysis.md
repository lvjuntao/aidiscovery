# dahai80「Fusion 本地优先 Apple Silicon AI 生态」架构图项目分析报告

> 分析对象：[dahai80](https://github.com/dahai80) GitHub 主页「Fusion 本地优先 Apple Silicon AI 生态」架构图（一核九端、六层、约 40 个 `fusion-*` 仓库）
> 信息来源：dahai80 公开仓库的主页 README 架构图、各仓库 README / 描述 / 语言构成（经 GitHub 公开 API 采集）
> 信息截止时间：2026-10-24
> 报告语言：中文

---

## 一、生态总体定位

**Fusion** 是 dahai80 个人主导构建的「本地优先（Local-first）Apple Silicon AI 生态」，核心主张写在其主页顶端：

> **One inference core, nine surfaces. 100% local, zero cloud dependency.**（一核九端，100% 本地，零云端依赖）

| 维度 | 内容 |
|------|------|
| **一核** | `fusion-mlx`——运行在 `localhost:11434` 的 OpenAI 兼容 MLX 推理引擎，为整个生态提供统一推理服务 |
| **九端** | 桌面（Studio）、CLI、Web、编码（Code）、文档（Doc）、科学（Science）、金融（Finance）、健康（Health）、教育（K12）九类应用面 |
| **平台约束** | 仅 macOS / Apple Silicon（M1–M5），MLX 原生、Metal 加速、统一内存（UMA）优化 |
| **隐私红线** | 模型、记忆、知识全部留在本机或内网；除 `127.0.0.1` 外无外发流量、零遥测 |
| **规模** | 主页架构图收录 33 个生态卡片（30 个 Live 仓库 + 2 个 Planned + 1 个 Web Portal）；「Full sub-project list」共列出 41 个子项目 |

生态各服务遵循统一的 `114xx` 端口约定（如 gateway 11432、artifacts 11451、doc 11449 等），统一以 `~/.fusion/config.toml` 为配置中心，并配套 `port-registry.yaml` 端口注册表——这是典型的「个人维护的分布式单机微服务」架构：每个仓库是一个独立服务/应用，通过 UDS（Unix Domain Socket）JSON-RPC 或 localhost HTTP 组合起来。

**技术栈全景**（按仓库主语言统计）：Python（领域应用与平台服务，约 20+ 个）、Rust（高性能核心组件：CLI/executor/memory/guard/store/design/supervisor）、Swift（macOS 原生组件：studio/browser/event）、Go（网关）、TypeScript/JavaScript（开发者工具前端：code/doc/drama）。多语言按需选型：推理和业务生态用 Python，性能敏感的守护进程用 Rust，原生体验用 Swift，网关用 Go「避免与 fusion-mlx 争抢 UMA 内存」。

---

## 二、六层架构总览

主页架构图自上而下分为六层：**Portal → Applications → Ecosystem → Developer Tools → Core Framework → Infrastructure**。图例约定：**● Live**（已落地，每个 ● 卡片链到其仓库）· **○ Planned**（规划中，尚无仓库）。

| 层 | 定位 | 项目（● Live / ○ Planned） |
|----|------|--------------------------|
| **Portal 门户层** | 用户入口 | ● bench.dpdns.org（Web Portal）· ● fusion-studio（macOS 桌面客户端） |
| **Applications 应用层** | 领域垂直应用 | ● fusion-k12-teacher · ● fusion-finance · ● fusion-health · ○ Fusion-Ecommerce · ● fusion-science |
| **Ecosystem 生态层** | 生态支撑设施 | ● fusion-plugins-ecosystem · ● fusion-bench · ● fusion-browser · ○ Fusion-Skills · ● fusion-simulation · ○ Fusion-Robot · ● fusion-comfyui |
| **Developer Tools 开发者工具层** | 面向开发者的生产力工具 | ● fusion-doc · ● fusion-code · ● fusion-design · ● fusion-security · ● fusion-cowork · ● fusion-agent-studio · ● fusion-cli · ● fusion-code-modelization |
| **Core Framework 核心框架层** | 统一网关 + 运行时 + 通用件 | ● fusion-gateway（统一 API 网关）＋ ● fusion-core / executor / memory / event / guard / store / osagent ＋ ● fusion-mlx / models-hub / artifacts-engine / multi-nodes |
| **Infrastructure 基础设施层** | 底层推理基座 | ● MLX-LM（ml-explore，语言模型推理）· ● MLX-VLM（Blaizzy，视觉模型推理）· ● Apple Silicon（M 系列统一内存） |

> 说明：架构图中 Core Framework 层实际画了三行——网关（gateway）一行、通用运行时件（core/executor/memory/event/guard/store/osagent）一行、推理与平台底座（mlx/models-hub/artifacts-engine/multi-nodes）一行；Infrastructure 层则引用第三方上游（Apple 的 MLX-LM、Blaizzy 的 MLX-VLM）与 Apple Silicon 硬件本身。主页「Project Layers」表还给出另一套按职责的七层视图（Core / Entry / Platform / Governance / Foundation / Applications / Domain），两者互为补充：六层架构图面向「用户视角的产品栈」，Project Layers 表面向「工程视角的职责分层」，本报告按验收要求以六层架构图为主线展开。

---

## 三、逐层逐项目详解

### 3.1 Portal 层（门户层）

#### ● bench.dpdns.org — Web Portal（Live，站外入口）
- **用途定位**：Fusion 生态的 Web 门户与基准测试榜单站点，`fusion-bench` 支持直接向其数据库写入结果，同时也是生态对外发放 Fusion API Key 的入口（fusion-drama 的 README 指向该站获取密钥）。
- **技术栈**：独立部署的 Web 站点（非 GitHub 仓库卡片），与 fusion-bench 的「Direct DB write」集成。
- **生态角色**：唯一的浏览器入口——把「本机优先」生态的成果（模型基准数据）以公网只读形式发布，兼顾演示与服务发现。

#### ● fusion-studio — macOS 统一桌面客户端（Live）
- **仓库**：https://github.com/dahai80/fusion-studio ｜ 主语言 Swift ｜ 仓库体量在生态中最大（约 76 MB）
- **用途定位**：把「一核九端」产品矩阵整合进**一个原生 macOS App** 的统一桌面客户端，官方口号「One app to rule them all」。当前 V0.1.64 已整合 **27 个模块**，覆盖 Design、Code、Simulation、MultiModal、Training、Data、Agent、KB、Bench、Desk 等场景，目标是消灭「5+ 终端、一堆浏览器标签页、散落目录」的碎片化使用体验。
- **技术栈**：Swift 5.9 + SwiftUI（macOS 14+），混合 Rust/Python 组件，内嵌 React 视图与 SQLite；通过 gRPC/REST/WebSocket 调用生态后端服务。
- **生态角色**：桌面端总壳（Desktop Shell）。按其 README 的生态位描述：`fusion-mlx（推理）→ Fusion Studio（统一桌面 App）→ Design/Code/Simulation/…（各功能模块）`。提供环境健康检查（自动探测 Xcode CLI/Homebrew/Python/MLX/Rust）、一键修复依赖、硬件监视器（CPU/GPU/内存/MLX 指标）、统一任务系统（立即/定时/一次性触发，Kanban 看板）、插件系统与无障碍支持。

### 3.2 Applications 层（领域应用层）

五个领域应用共同的特征：**Python + FastAPI/httpx + fusion-mlx 推理，100% 离线、零数据上传**，且每一个都自我定位为「某云服务的国产/本地替代品」（README 中普遍附有与对应 Claude 云产品的逐项对比表）。

#### ● fusion-k12-teacher — K-12 智能教育助手（Live）
- **仓库**：https://github.com/dahai80/fusion-k12-teacher ｜ Python ｜ Apache-2.0 ｜ 591 测试通过
- **用途定位**：本地 AI K-12 教育助手，定位「Claude K-12 Teacher 的国产替代」。提供备课、自动出题、作文评分、学科答疑、个性化学习路径、课件/工作纸/游戏内容生成、家校沟通模板、PBL 项目式学习设计等。
- **技术栈**：Python 3.11+、FastAPI、fusion-mlx HTTP 推理；内置**2022 版课程标准知识图谱**、三层差异化教案、班级/学生画像与错题分析、多层内容安全过滤与年龄校验、姓名匿名化与字段脱敏；支持 Docker Compose 与 K8s 部署。
- **生态角色**：教育端（Education）。九端之一，演示了「同一推理内核 + 领域提示词/知识库」即可长出一个垂直行业应用的生态复制模式。

#### ● fusion-finance — 本地金融分析平台（Live）
- **仓库**：https://github.com/dahai80/fusion-finance ｜ Python ｜ Apache-2.0 ｜ 530 测试
- **用途定位**：本地 AI 金融分析平台，定位「Claude Financial (CFS) 的国产替代」：财务建模、财报分析、风险管理、报告生成，全部离线零上传。
- **技术栈**：Python 3.11+、FastAPI、httpx、fusion-mlx；v0.5.5 完成生产化加固（AI 失败显式 `ai_status`、审计链 fcntl 文件锁 + 服务端绑定 actor 防伪造、`/ready` 在 MLX 降级时返回 503、鉴权关闭时拒绝非回环绑定等 65 项整改）；含 DCF 估值、压力测试、投资组合、审计链校验端点与 xlsx 导出。服务端口 11466（v0.5.4 从 11446 迁出以避开 multi-node MCP 网关的端口冲突——侧面印证生态统一的端口注册表治理）。
- **生态角色**：金融端（Finance）。对数据不出域要求最强的场景，是「本地优先」卖点的最佳注脚。

#### ● fusion-health — 本地医疗健康助手（Live）
- **仓库**：https://github.com/dahai80/fusion-health ｜ Python ｜ Apache-2.0 ｜ 148 测试
- **用途定位**：本地 AI 医疗健康助手，定位「Claude Health 的本地替代」：病历处理、临床摘要生成、ICD-10/CPT 编码建议、临床文献检索、合规审计。
- **技术栈**：Python 3.12+、fusion-mlx（默认模型 Qwen3.5-9B-4bit @ localhost:11434）；主打 PHI（受保护健康信息）不出设备、PIPL 合规、原生中文医疗支持；提供 CLI 与 API 两种形态。
- **生态角色**：健康端（Health）。医疗隐私合规是云 AI 的天然短板，本仓库用「PHI 永不离开设备」作为核心差异点。

#### ● ○ Fusion-Ecommerce — 电商平台（Planned，尚无仓库）
- **架构图标注**：○ Planned，E-commerce Platform。
- **现状**：GitHub 搜索不到 `fusion-ecommerce` 仓库（API 404），属规划中项目。值得注意的是，`fusion-projects` 的 v0.3.1 变更日志提到「douyin e-commerce AgentGraph 可获取商品主图并真实 CDP 发布」，说明电商自动化能力已在 cowork/projects 侧以 Agent 工作流形式预研，独立电商平台是后续形态。
- **生态角色**：商业端预留位——补齐「九端」叙事中的电商场景。

#### ● fusion-science — 本地科研 AI 工作台（Live）
- **仓库**：https://github.com/dahai80/fusion-science ｜ Python ｜ Apache-2.0
- **用途定位**：面向科研全流程的本地 AI 工作台，定位「Claude Science 的国产替代」：文献综述、数据计算、可视化、论文写作、结果可溯源统一在一个界面里完成。
- **技术栈**：Python、FastAPI + SSE 流式、httpx 网关客户端；**8 个科学数据库连接器**（PubMed/UniProt/PDB/Ensembl/ChEMBl + 国内的 NGDC/CNKI/ScienceDB，支持国内镜像）；基于 MCP 的多智能体自动运行 Python/R/Jupyter 做统计分析、组学处理、分子模拟；2D 统计图 + 3D 分子/蛋白结构可视化；全链路审计（每个图表/数据点/论文片段保留查询源、执行代码、参数与计算日志）。
- **生态角色**：科学端（Science）。是「领域应用 = 推理内核 + 领域工具链 + 领域数据源」模式的最完整示范。

### 3.3 Ecosystem 层（生态层）

七个卡片中五个 Live、两个 Planned，职责是「让生态可扩展、可评测、可感知」。

#### ● fusion-plugins-ecosystem — 插件生态（Live）
- **仓库**：https://github.com/dahai80/fusion-plugins-ecosystem ｜ Python ｜ Apache-2.0 ｜ 634 测试、89% 覆盖率
- **用途定位**：插件注册中心 + 生命周期管理器 + **Claude 全链路原生适配层**。建立在 fusion-cowork 之上的上层子模块（非独立项目）：声明式插件清单、参数 Schema、能力声明；加载/卸载/热重载、超时熔断、自动重启、INLINE/PROCESS 两种沙箱模式。
- **技术栈**：Python 3.11+；每个插件**零适配代码**自动转换为 Claude Code Skill / Agent / Plugin Bundle，并以 MCP Tool 形态暴露（stdio/SSE/HTTP 三种传输）；独立的 Token 计量（区分 Claude 模型消耗与插件本地计算）。
- **生态角色**：生态扩展性的关键——把 cowork 的自动化能力以 Claude/MCP 兼容格式对外输出，实现「一次开发、Claude 生态即用」。

#### ● fusion-bench — 性能基准工作台（Live）
- **仓库**：https://github.com/dahai80/fusion-bench ｜ Python ｜ Apache-2.0 ｜ 336 测试
- **用途定位**：MLX 模型性能基准与自动调参工作台：跑分、基准、自动调优，全部本地。
- **技术栈**：Python 3.12+、fusion-mlx API、Metal 监视器；兼容 **2082 个 lm-eval 任务**；4/8/16-bit 量化对比、自动参数调优、三级质量门（实验/业务/生产）、安全探针（注入/有害/PII）、SQLite 持久化 Trace、五维雷达图/趋势图、多格式导出（JSON/MD/PDF/Excel/HTML）、GitHub Action CI 集成、HTTP 远程分发、fusion-identity 租户鉴权。
- **生态角色**：评测中枢，且与 **bench.dpdns.org** 门户直连（Direct DB write）——门户层那张「Web Portal」卡片的数据发动机。

#### ● fusion-browser — 受控浏览器引擎（Live）
- **仓库**：https://github.com/dahai80/fusion-browser ｜ Swift ｜ Apache-2.0
- **用途定位**：macOS 原生**受控 Agent 浏览器**：为 agent-studio 提供 Web 视觉 + 结构化交互，为 cowork 复用 CDP 自动化能力。
- **技术栈**：Swift + 原生 WebKit（WKWebView）；双协议栈（UDS 长度前缀 JSON 主通道 + CDP-over-WS 兼容层）；**AXTree 词元化**与**防注入净化器**（anti-injection sanitizer）；Phase 4 生产加固已落地：按工作流运行的持久化 Profile（登录态跨会话/跨重启存活）、Cookie/登录态导入（Firefox 明文、Chrome/Edge AES-128-CBC + Keychain 解密，含 Chrome 127+ SHA256 前缀校验修复）、截图推流（200ms 节流）。
- **生态角色**：Web 感知与操作执行器——osagent 操作「像素级桌面」，browser 操作「DOM 级网页」，两者互补构成完整的 GUI 自动化感知层。

#### ● ○ Fusion-Skills — 技能库（Planned，尚无仓库）
- **架构图标注**：○ Planned，Skills Library。GitHub 上尚无对应仓库。
- **生态角色**：预计将沉淀可复用的技能资产（与 plugins-ecosystem 的 Skill 转换能力衔接）。

#### ● fusion-simulation — 机器人虚拟仿真平台（Live）
- **仓库**：https://github.com/dahai80/fusion-simulation ｜ Python ｜ Apache-2.0
- **用途定位**：Apple Silicon 上的机器人虚拟仿真训练与测试平台。
- **技术栈**：Python + **PyBullet** 物理引擎（MuJoCo 为 stub），**六层架构借鉴 NVIDIA Isaac Sim**：L1 核心（SimClock/ECS/EventBus）→ L2 物理 → L3 传感器（RGB/深度/IMU/接触/语义）→ L4 Agent（PolicyClient 经 fusion-mlx HTTP 推理）→ L5 训练/评估（BCTrainer/FusionGymEnv）→ L6 服务（gRPC + JSON-RPC + OpenAI 兼容 REST）。**从不直接 import torch/CUDA/MLX**，所有神经网络推理经 HTTP 委托给 fusion-mlx。
- **生态角色**：具身智能的训练场，也是「Planned 的 fusion-robot 的前置底座」——先仿真后实机。

#### ● ○ Fusion-Robot — 机器人控制（Planned，尚无仓库）
- **架构图标注**：○ Planned，Robot Control。GitHub 上尚无仓库。
- **生态角色**：与 fusion-osagent 互为「具身智能双生子」（osagent README 原话：robot 做体素级物理机器人控制，osagent 做像素级桌面 GUI 自动化）。仿真平台（fusion-simulation）先行落地，实机控制待规划。

#### ● fusion-comfyui — MLX 原生 ComfyUI 后端（Live）
- **仓库**：https://github.com/dahai80/fusion-comfyui ｜ Python ｜ GPL-3.0 ｜ 2 stars（生态中最高之一）
- **用途定位**：Apple Silicon 上**纯 MLX 的 ComfyUI 服务器，运行时零 PyTorch**；支持 FLUX.2、Wan2.2、SkyReels-V3、LTX-2 等图像/视频生成模型经 Metal/MLX 出图出片。
- **技术栈**：Python、FastAPI、fusion-mlx 引擎；四阶段演进：Phase 1 ComfyUI 自定义节点（PyTorch 宿主 + MLX 计算，含原生节点桥接覆盖）→ Phase 2 独立 FastAPI 服务器（ComfyUI 协议、零 PyTorch）→ Phase 3 规格去噪机制（已落地但默认关闭——诚实地记录了「加速被证伪」的负结果）→ Phase 4 Swift 桌面 App + DMG 安装包（首启动 0.15s，无终端端到端 T2V）。
- **生态角色**：多模态生成端——把生态从「文本推理」扩展到图像/视频创作，也是「九端」之外的重要多模态支柱。

### 3.4 Developer Tools 层（开发者工具层）

八个卡片全部 Live，是生态中密度最高的一层，覆盖编码、文档、设计、安全、协作、Agent 编排、命令行与遗留代码现代化。

#### ● fusion-doc — 智能文档平台（Live）
- **仓库**：https://github.com/dahai80/fusion-doc ｜ JavaScript ｜ Apache-2.0
- **用途定位**：本地优先 AI 文档平台（AI-First Document OS）：写作、编辑与知识感知的文档工作流。
- **技术栈**：React 18 + Vite + **TipTap 2.x** 富文本编辑器、D3.js 知识图谱力导向布局、SSE 流式 AI 面板、Office 导入导出、模板系统、Tailwind CSS 暗色主题；服务端模块化 MVC + SQLite/JSON 双存储；**零外部依赖**（无需 PostgreSQL/Redis/Nginx）；服务端口 11449。
- **生态角色**：文档端（Docs）。AI Ghost Text 行内补全（Cmd+J 触发）等 AI 原生编辑体验，向 Claude Docs 类产品看齐。

#### ● fusion-code — 终端原生 AI 编码智能体（Live）
- **仓库**：https://github.com/dahai80/fusion-code ｜ TypeScript ｜ AGPL-3.0（衍生自 Claude-Code-SourceMap，README 明示尊重 Claude 原作）
- **用途定位**：终端原生 AI 编码智能体：本地优先、零遥测、单二进制；深度本地 MLX 集成，云端后端可选。
- **技术栈**：TypeScript + **Bun** 运行时；本地推理对接 `127.0.0.1:11432`（经 fusion-gateway）自动发现本地模型；云后端可插 Anthropic/OpenAI Codex/Azure Foundry（429/529 自动降级链）；**88 个特性开关**（ULTRAPLAN 多智能体、ULTRATHINK 深度推理、语音输入、IDE 桥接等）；智能权限（Auto 模式自动放行安全操作）、上下文管理（自动压缩/硬压缩/MLX 内存安全）。
- **生态角色**：编码端（Coding）。三大旗舰产品之一（Code 写能力 / Studio 编排工作流 / Cowork 全员自动化），是生态中面向程序员的门面。

#### ● fusion-design — AI 辅助设计工作台（Live）
- **仓库**：https://github.com/dahai80/fusion-design ｜ Rust ｜ Apache-2.0 ｜ 495 测试
- **用途定位**：全本地离线的 AI 对话式 UI/原型设计工作台，对标 Claude Design（云端闭源）与 Figma/Penpot。
- **技术栈**：Rust workspace（**13 个 crate**）+ 二次封装 OpenPencil + fusion-mlx 本地多模态推理；内嵌于 Fusion-Desk 的 WKWebView。V0.2 能力：无限矢量画布、对话式 AI 生成 UI（文生 UI/图生 UI，SSE 流式）、内置设计系统（Apple HIG / Admin Dashboard 两套规范）、设计 Lint + 自动修复（13 条规则）、**与 Fusion-Code 双向同步**（画布一键导出 React/HTML/Tailwind，样式反向同步）、PNG/SVG/PDF/HTML 交付、50 级撤销 + 节点级 Diff、全功能 CLI（NDJSON 流式输出）。WASM 构建通过，CI 含 deny-unwrap-expect/deny-external-http 等严格门禁。
- **生态角色**：设计端（Design）。「设计稿 ⇄ 可运行代码」一键转换是其与 Fusion-Code 协同的核心竞争力。

#### ● fusion-security — 本地代码安全审计工具（Live）
- **仓库**：https://github.com/dahai80/fusion-security ｜ Python ｜ MIT ｜ 847 测试
- **用途定位**：本地 AI 代码安全审计工具，定位「Claude Security 的国产替代」：企业级漏洞扫描、AI 语义分析、自动修复生成，100% 离线零代码上传。
- **技术栈**：Python 3.12+、FastAPI、React 18 + Ant Design 仪表盘、fusion-mlx 推理；跨文件数据流漏洞扫描、对抗式验证降误报、Markdown/JSON/HTML 审计报告、CLI + GitHub Actions + GitLab CI 集成、飞书/钉钉通知、断点续跑 + 熔断器、K8s Helm Chart 部署；自身安全加固含 SSRF 防护（DNS 钉扎防重绑定）与日志密钥脱敏。
- **生态角色**：安全端。与 guard（系统层安全）分工：security 管「代码资产」的漏洞，guard 管「运行时行为」的越权。

#### ● fusion-cowork — 零代码桌面自动化平台（Live）
- **仓库**：https://github.com/dahai80/fusion-cowork ｜ Python ｜ Apache-2.0 ｜ v0.5.4
- **用途定位**：本地优先、**零代码**桌面自动化平台：用自然语言描述需求即得自动化工作流，面向全部办公用户（非程序员）。
- **技术栈**：Python 3.11+、fusion-mlx MLX 原生推理；深度集成 macOS Desktop/Downloads/Documents 目录、AI 语义理解（理解内容而非仅文件名）；与 Code/Agent-Studio/KB/Model-Hub 无缝协同。官方生态位口号：**「Studio builds workflows, Desk runs them, Code writes capabilities, KB stores knowledge, Hub manages models.」**
- **生态角色**：三大旗舰产品中最「平民化」的一端——把 Agent 能力从开发者扩散到普通办公人群。

#### ● fusion-agent-studio — 本地 Agent 开发平台（Live）
- **仓库**：https://github.com/dahai80/fusion-agent-studio ｜ Python ｜ Apache-2.0 ｜ v0.13.0 ｜ 3135 测试（生态中最大测试规模之一）
- **用途定位**：Apple Silicon 本地 Agent 开发平台：在 Mac 上运行、构建、编排 AI 智能体，无云、无 API 费用。
- **技术栈**：Python 3.11+，可选 sqlite-vec + FTS5 + RRF 混合本地向量检索（rag 扩展）、fusion-plugins-ecosystem 桥（plugins 扩展）、共享 harness 内核（funnels/hooks/permissions/compactor，缺省自动回退内置内核）；与 multi-nodes 打通跨节点会话检查点（v0.20 共享 session store）。
- **生态角色**：智能体编排中枢——面向开发者/架构师的「Agent IDE」，是 fusion-browser/osagent/plugins 等执行器的编排上游。

#### ● fusion-cli — 统一命令行入口（Live）
- **仓库**：https://github.com/dahai80/fusion-cli ｜ Rust ｜ Apache-2.0 ｜ v0.3.5
- **用途定位**：整个 Fusion-MLX 生态的统一 CLI 入口：「One CLI, Control All」——单个二进制管理模型、推理、知识库、基准、自动化与服务。
- **技术栈**：Rust 1.80+ 单二进制（macOS Apple Silicon，无 Python/Node 依赖）；**唯一后端锁定 fusion-mlx**（明确排他：不用 ollama/omlx/OpenAI）、100% 离线零遥测；模型管理/聊天（SSE）/知识库/RAG 服务/基准/服务管控/桌面自动化模板/AI Agent（sandbox/ask/auto 三档权限）/TUI 交互仪表盘；全部命令支持 `--format=json` 可脚本化；服务 URL 统一读 `~/.fusion/config.toml`；全局 reqwest 连接池复用。
- **生态角色**：Entry 层（主页 Project Layers 表中的「Entry」）。运维与脚本的统一操作面，bash/zsh/fish 补全齐备。

#### ● fusion-code-modelization — 遗留代码现代化平台（Live）
- **仓库**：https://github.com/dahai80/fusion-code-modelization ｜ Python ｜ Apache-2.0 ｜ 727+ 测试
- **用途定位**：遗留代码现代化与跨语言迁移平台：现代化、重构、迁移遗留代码库，全程本地 fusion-mlx 驱动，定位「Claude Code Modernization 的本地替代」。
- **技术栈**：Python 3.12+、fusion-mlx；支持 COBOL→Java、VB6→C# 等跨语言迁移；测试先行 + 双运行验证的安全增量重构、安全扫描集成。
- **生态角色**：开发者工具层的「存量市场」入口——面向企业遗留系统的现代化需求，与 fusion-code（增量开发）形成新旧代码全覆盖。

### 3.5 Core Framework 层（核心框架层）

架构图上该层为三行：**统一网关 → 通用运行时件 → 推理与平台底座**。这一层是「一核九端」的「核」所在，也是工程质量最重的部分（Rust/Swift 集中于此）。

#### ● fusion-gateway — 统一 API 网关（Live）
- **仓库**：https://github.com/dahai80/fusion-gateway ｜ Go ｜ Apache-2.0 ｜ 2 stars
- **用途定位**：本地 AI 模型编排的轻量高性能 API 网关——**混合推理网关**：统一 Apple Silicon 本地推理 + 云端 LLM 的多模型接入、推理路由与资源调度，是 Agent-Studio / fusion-mlx / fusion-code 的核心流量入口。
- **技术栈**：Go（选型理由：**避免与 fusion-mlx 争抢 UMA 内存**）；七层管线：接入层（鉴权/解析/限流）→ 预处理（tokenizer 计数/prompt 校验/参数默认）→ 路由引擎（**规则引擎 + 硬件负载感知**，核心差异化能力）→ 适配器池（统一后端接口）→ MCP 网关（工具注册/路由/Token 预算）→ 流转发（SSE/取消/重试/KV 缓存释放）→ 可观测（日志/指标/热配置重载）。后端池覆盖本地（fusion-mlx :11434、llama.cpp）、私有（vLLM-ascend/cuda）与十余家云端（OpenAI/Claude/DeepSeek/Bedrock/Vertex/Foundry + 国内的 DashScope/Moonshot/智谱/Minimax/百川/混元/StepFun/Yi）。默认端口 11432，可自动拉起 fusion-mlx 并等其健康；容器镜像 ~12.6MB（alpine 多阶段构建，仅转发业务面，MLX 留裸机）。
- **生态角色**：生态的「总闸」——所有上层组件的推理请求汇聚点，也是「本地优先但可按需上云」这一弹性边界的管理者。

#### 通用运行时件（7 个）

| 项目 | 语言 | 用途定位 | 技术栈与要点 | 生态角色 |
|------|------|---------|-------------|---------|
| **● fusion-core** | Python | 生态共享技术底座：纯技术、零业务逻辑 | 统一 `FusionMLXClient`（chat/embedding/stream，默认 base_url 解析自 `FUSION_MLX_URL`）、LLM JSON 解析三件套（失败可见绝不静默）、配置加载（mtime 缓存失效）、幂等日志、httpx 按事件循环 LRU 连接池 + 网关熔断探测、FastAPI 工厂、prompt 模板管理、零信任 guard 客户端；`import` 零 I/O | 消灭了生态内 7+ 个重复 LLM 客户端与 10+ 份 `_parse_json` 拷贝，Python 项目的公共地基 |
| **● fusion-executor** | Rust | 受控执行沙箱 + macOS OS 级控制中枢 | 12-crate Cargo workspace + PyO3/maturin Python 绑定（生态首个 maturin 项目）；Security Guard（正则黑名单 + tokenizer + 白名单）、PTY 子进程沙箱（超时/截断/OOM 上限）、**Git 快照回滚**（原生替代 Docker 沙箱，init <5ms）、Computer Use（AXUIElement + CoreGraphics + CGEvent 合成，20 种 GuiAction）、UDS JSON-RPC IPC、8 语言诊断切片器（tree-sitter）、10Hz 活体遥测、macOS seatbelt、guard 零信任授权；528 Rust + 269 Python 测试 | Agent 的「手」：所有需要真实执行 shell/GUI/文件操作的上层组件都经它，且带安全围栏与回滚 |
| **● fusion-memory** | Rust | 系统级长短期记忆 + 认知图谱中枢 | 11 crate、v1.4.1；SQLite/Kuzu + HNSW 向量（经 fusion-store 适配器桥接，距离语义 1-cos_sim 归一）；**v1.0.0 起 SemVer 冻结线协议**（UDS JSON-RPC 方法集 + HTTP `/v1/memory/*`）；分级记忆命名空间（user/project/local）、多租户隔离、identity 集成；解决 Agent 跨会话状态丢失、重复提问、上下文窗口爆炸 | 生态的「海马体」：让生态越用越懂用户 |
| **● fusion-event** | Swift | macOS 原生反应式事件守护进程（感知层） | Swift 6 严格并发（actor 隔离）；监听 FSEvents/NSWorkspace/Pasteboard/网络事件，规范化后经规则引擎（防抖/节流/glob）触发 Agent 任务；SQLite WAL 规则存储 + 滚动 JSONL 事件日志；全部 UDS 回环 | 把生态从「被动等提示」升级为「感知系统事件 → 主动触发智能体」 |
| **● fusion-guard** | Rust | 零信任动作授权守护（TCC/DLP 防火墙） | 14-crate workspace + UDS JSON-RPC + SQLite WAL 审计；拦截高危副作用（rm -rf、静默外传）、动态脱敏（API key/密码/身份证/私钥/邮箱/IPv4/银行卡）、macOS TCC 权限审计聚合（Swift 桥）、审计链哈希防篡改、tree-sitter 语义扫描、PyO3 绑定、跨节点联邦审计（HKDK 域分离 MAC）；v0.2.0-rc.3 | 生态的「免疫系统」：本地 AI OS 的安全底线，防止 Agent 越权与数据泄漏 |
| **● fusion-store** | Rust | 单机零拷贝存储与索引引擎（L1 持久化基座） | mmap + heed KV（崩溃安全）、**单层 NSW 图向量索引**（README 诚实纠正：并非 HNSW，O(N^log M)，生产建议每命名空间 ≤1-2M 向量，1M×768 实测 p99 1544μs）、Apache Arrow 定宽列存；段 append-only 不可变 + WAL 唯一崩溃同步点；零拷贝仅限 Rust 进程内（C-ABI 强制拷贝，公共定位如实声明）；axum + Prometheus | 向量/KV/列三合一的嵌入式存储底座，供 memory/RAG 等复用 |
| **● fusion-osagent** | Python | 桌面具身智能——像素级 OS Agent 屏障层 | 复用兄弟包而非重造：AX+视觉双轨感知（AXUI 树优先，视觉 grounding 兜底）、Set-of-Mark 标注、快慢双核推理（小 VL 提议、大 VL 校验规划）、动作后帧断言 + 自愈、类人贝塞尔轨迹 + 敏感区遮罩；API 对齐 Claude Computer Use `computer` 工具并扩展 `assert/heal/som_view/replay`；经 executor(AXUI/CGEvent)/mlx(推理)/browser(Web)/agent-studio(编排) 四适配器 | 桌面 GUI 自动化的「可靠层」——把原始 GUI 访问变成可断言、可自愈的 Agent 能力，是 fusion-robot（○ Planned）的桌面镜像 |

#### 推理与平台底座（4 个）

| 项目 | 语言 | 用途定位 | 技术栈与要点 | 生态角色 |
|------|------|---------|-------------|---------|
| **● fusion-mlx** | Python | **生态内核**：Apple Silicon 统一本地模型服务 | Ollama/vLLM 的 Drop-in 替代，MLX 原生 Metal 运行；431 测试文件 / 8742 测试项；专注 UMA 独有能力：**UMA Radix 文本-KV 缓存**（radix 树 + LRU + pin/unpin，扩散模型文本编码器零拷贝复用）、**DSpark 投机解码**（14B 实测 1.47×）、**DFlash2 块扩散投机解码**（Qwen3.8-27B-4bit 实测 2.47×，接受率均值 3.56）、**DSA 共享专家激活缓存**（MoE 重复行去重，零输出漂移）；对「投机去噪」的负结果诚实记录并默认关闭；发布 PyPI 包 + 桌面 App + bench.dpdns.org 榜单 | 「一核」本体：localhost:11434 的 OpenAI 兼容推理服务，全生态共享 |
| **● fusion-models-hub** | Python | 生态统一模型仓库与管理中心 | FastAPI 异步服务；模型 CRUD/搜索/推荐、版本管理（SHA256 校验、5MB 分块上传）、HuggingFace 镜像导入、量化任务（2/4/6/8-bit 经 fusion-mlx）、LoRA 合并、灰度发布（流量比例 + 推理代理路由）、副本伸缩、评测跟踪、多实例差分同步、离线导入导出、安全扫描、水印、AES-256 静态加密、多级审批、RBAC + 多租户 | 生态的「模型资产部」：模型全生命周期治理（注：架构图卡片名 Fusion-Model Hub，实际仓库名为复数 `fusion-models-hub`；主页 Project Layers 表中的 `fusion-model-hub` 单数名与仓库列表的 404 结果为笔误，以实际仓库为准） |
| **● fusion-artifacts-engine** | Python | 结构化产物 CRUD 中间件（渲染与产物执行引擎） | 物理分离「生成产物」与「聊天消息」：长内容独立存储、对话中替换为 ~30 token 轻量引用、按需回读——**10 轮千行代码迭代约 15k token（原 ~120k）**；stdlib HTTP 服务（无 FastAPI 依赖）端口 11451、Docker 容器化、原子写、路径校验、RBAC、审计 | 长对话上下文治理件：让长程 AI 会话不至于上下文爆炸，与 memory 的「记忆治理」互补 |
| **● fusion-multi-nodes** | Python | 分布式 Apple Silicon MLX 推理的集群调度核心 | v0.20.0、1533 测试；流水线并行 + 数据并行，把多台 Mac 池化为统一 AI 集群；常驻节点/家庭服务器模式（笔记本关机不中断调度）、共享会话存储 + 沙箱运行委托（agent-studio 跨节点恢复）、集群状态 Web UI、MCP 网关、可观测性；100% 局域网/离线 | 「一核」的多机扩展形态：从单机本地优先走向内网集群，仍不依赖云 |

### 3.6 Infrastructure 层（基础设施层）

三个卡片回答「推理基座从哪来」：

#### ● MLX-LM（第三方上游，Live）
- **仓库**：https://github.com/ml-explore/mlx-lm （Apple 官方 ml-explore 组织）
- **用途定位**：Apple 官方 MLX 框架上的大语言模型微调与推理库。
- **生态角色**：fusion-mlx 的直接技术上游——语言模型推理能力的源头依赖。架构图将其列入自家居基础设施层，如实标注了生态的「站在 Apple 官方库之上」的构建方式。

#### ● MLX-VLM（第三方上游，Live）
- **仓库**：https://github.com/Blaizzy/mlx-vlm （社区作者 Blaizzy）
- **用途定位**：MLX 生态的视觉语言模型（VLM）推理库。
- **生态角色**：多模态视觉推理的上游——osagent 的快慢双核视觉推理、design 的图生 UI、comfyui 的视觉生成链路都依赖此类 VLM 能力。

#### ● Apple Silicon（硬件基座，Live）
- **说明**：M 系列芯片统一内存（Unified Memory）架构。这不是仓库而是硬件前提：CPU/GPU 共享同一内存池，使「大模型常驻 + 多服务并存 + 零拷贝复用」成为可能。fusion-mlx 的 UMA Radix KV 缓存、fusion-store 的 mmap 零拷贝、gateway 的「Go 选型避免争抢 UMA」均围绕这一硬件特性展开。
- **生态角色**：整个生态的性能与隐私承诺的物质基础——「100% 本地」之所以可行，是因为推理、存储、记忆全部装得进一台 Mac。

---

## 四、主页隐藏彩蛋：Project Layers 职责视图与全量项目清单

主页 README 在架构图之外还给出两个补充视图，本报告一并收录以便交叉验证：

**① Project Layers 七层职责表**（工程视角）：

| 职责层 | 项目 | 职责 |
|--------|------|------|
| Core | fusion-mlx | MLX 推理引擎、OpenAI 兼容 API、自定义 GEMM 内核 |
| Entry | fusion-cli | Rust 单二进制 `fusion` 命令 |
| Platform | fusion-model-hub · fusion-kb · fusion-bench · fusion-plugins-ecosystem | 模型中心 · 知识库 · 基准 · 插件 |
| Governance | fusion-memory · fusion-identity · fusion-supervisor · fusion-gateway | 跨会话记忆 · 身份 · 进程监管 · API 网关 |
| Foundation | fusion-core · fusion-cowork | 公共技术库 · DAG 工作流引擎 |
| Applications | fusion-code · fusion-agent-studio · fusion-doc · fusion-design · fusion-studio | 编码 · 智能体 · 文档 · 设计 · 桌面壳 |
| Domain | fusion-science · fusion-finance · fusion-health · fusion-simulation · fusion-k12-teacher · fusion-security · fusion-multi-nodes · … | 垂直领域应用 |

> 交叉验证发现：该表中 `fusion-model-hub`（单数）与 `fusion-kb` 实际对应真实仓库 `fusion-models-hub`（复数）与 `fusion-rag`（描述自称 Fusion-KB）——主页存在少量命名不一致，属正常的个人项目文档漂移，本报告已按实际仓库勘正。

**② Full sub-project list（41 项全量清单）**：fusion-agent-studio · fusion-artifacts-engine · fusion-autotest · fusion-bench · fusion-browser · fusion-cli · fusion-code · fusion-comfyui · fusion-core · fusion-cowork · fusion-design · fusion-doc · fusion-drama · fusion-ecommerce · fusion-event · fusion-executor · fusion-finance · fusion-game · fusion-gateway · fusion-guard · fusion-health · fusion-identity · fusion-k12-teacher · fusion-memory · fusion-mlx · fusion-model-hub · fusion-multi-nodes · fusion-musetalk · fusion-ninfer · fusion-plugins-ecosystem · fusion-projects · fusion-rag · fusion-robot · fusion-science · fusion-security · fusion-simulation · fusion-store · fusion-studio · fusion-supervisor · fusion-trainer · fusion-web-shell

**清单中未上架构图、但已实际存在的仓库**（采集时核实存在且有近期推送，属「图表未收录的在建项目」）：

| 仓库 | 主语言 | 用途（据仓库描述/README） |
|------|--------|--------------------------|
| fusion-identity | Python | 多租户身份服务：生态唯一 JWT 签发者与租户注册中心（FastAPI :11470 + Postgres），四角色 RBAC、配额、哈希链审计、fail-closed 三红线；Project Layers 表将其归入 Governance 层 |
| fusion-supervisor | Rust | 本地优先服务监管守护：长驻 `fusion-sv` 按 port-registry.yaml 监管 53 个服务，崩溃自动恢复（滑动窗口 + 指数退避）、健康历史（SQLite）、零停机 rollout、Postgres/Qdrant 备份；Project Layers 表 Governance 层 |
| fusion-projects | Python | 本地优先 AI 项目资产容器：项目 = 全局指令 + 持久知识库 + 隔离会话 + 绑定 Agent（对标 Claude Projects）；76 RPC + 9 MCP 工具 + 65 REST 路由 |
| fusion-rag | Python | Fusion-KB：Apple Silicon 原生离线向量知识库后端（LanceDB + MLX、BM25+向量 RRF 混合检索、Contextual Retrieval、GraphRAG、AST 感知分块、MCP Server） |
| fusion-drama | TypeScript | AI 短剧生成平台：剧本生成 → 角色设计 → 分镜 → 视频合成的全流程自动化（Nuxt3/Vue3 + Hono 后端，本地 LTX-2.5/MiniMax-H3/Wan2.2 可全离线） |

**清单中 404（不存在或已改名/删除）的项**：fusion-autotest、fusion-ecommerce（架构图 Planned）、fusion-game、fusion-model-hub（单数名，实为 fusion-models-hub）、fusion-musetalk、fusion-ninfer、fusion-robot（架构图 Planned）、fusion-trainer、fusion-web-shell——其中 supervisor README 提到 fusion-autotest 曾作为一次性测试工具存在，推测部分为已删除或未创建的历史/规划名。

---

## 五、技术选型与架构评价

**选型逻辑高度自洽**，可总结为四条原则：

1. **语言按角色分工**：推理生态与领域应用用 Python（迭代快、MLX 绑定原生）；性能与安全敏感的守护进程（executor/guard/memory/store/event/supervisor/CLI）用 Rust；原生体验（studio/browser）用 Swift；网关用 Go 并明确给出「不与推理引擎争抢统一内存」的理由。每种语言都用在刀刃上，没有为了统一技术栈而牺牲适配性。

2. **本地优先被贯彻到协议层**：所有服务间通信用 UDS JSON-RPC 或 localhost HTTP，配置统一 `~/.fusion/config.toml`，端口统一 114xx 注册表，supervisor 统一监管——「本地」不是宣传语而是一套可治理的单机微服务工程规范。

3. **复用优先于重造**：osagent 明确「复用兄弟包而非重建」，core 的存在理由就是消灭重复实现，plugins-ecosystem 干脆做成了 cowork 的上层子模块而非独立运行时。这种克制是个人维护 40+ 仓库仍能运转的关键。

4. **诚实的工程文化**：多处 README 记录了负结果与勘误——fusion-mlx 把「投机去噪被证伪（0% 接受、0.42× 更慢）」写进文档并默认关闭；fusion-store 主动纠正「向量索引是单层 NSW 而非 HNSW」并给出容量边界；fusion-comfyui 区分「已落地/被证伪/被阻断（NVFP4 依赖上游 mlx#2962）」。这种如实的工程沟通在开源项目里相当少见。

**风险与局限**（基于公开信息的客观观察）：

- **单人维护 40+ 仓库**：fusion-mlx README 自述「single-maintainer project, seeking contributors」，Bus Factor = 1 是生态最大的可持续性风险；多数仓库 star 数 0–4，社区反馈回路很弱。
- **成熟度参差**：核心组件（mlx/memory/guard/executor）已进入 SemVer/RC 阶段且测试规模可观（数千测试项），但领域应用多为 v0.x beta；架构图将它们统一标为 ● Live，实际生产可用性应以各仓库自述的版本与状态为准。
- **命名漂移**：架构图卡片名、Project Layers 表名与实际仓库名存在不一致（model hub 单复数、rag vs kb、code-modelization vs 图上「Code Modernization」），对接时需以 GitHub 实际仓库为准。
- **生态耦合**：组件间依赖以本地路径/monorepo 安装为主（多个包未上 PyPI），脱离作者的 `~/fusion` 目录完整复现生态需要自行拼装。

---

## 六、总结

dahai80 的 Fusion 生态是一份罕见完整的「**个人级本地 AI 操作系统**」蓝图的实物化：以 fusion-mlx 为唯一推理内核，以 gateway/cli/studio 为三类入口，以 core/executor/memory/event/guard/store 为运行时七件套，再向外辐射出教育、金融、健康、科研、仿真、多模态生成等九端应用与一整层开发者工具。六层架构（Portal → Applications → Ecosystem → Developer Tools → Core Framework → Infrastructure）从上到下职责清晰：上层做产品与体验，中层做生态能力（插件/基准/浏览器/仿真），下层做引擎与治理（网关/运行时/安全/存储/集群），最底层诚实地承认自己站在 Apple MLX 生态的肩膀上。

它的独特价值不在单项技术，而在**把「本地优先」当作一等工程约束来做系统性设计**：零拷贝存储、UDS 全互联、零信任授权、跨会话记忆、产物引用化、集群化扩展——每一层都有对应「云端依赖」的本地替代方案。对于关注 Apple Silicon 本地 AI、隐私计算、个人 AI OS 方向的开发者，这套仓库群是一个高密度的参考样本。

---

*报告由自动化分析流程生成于 2026-10-24，数据采集自 GitHub 公开 API（dahai80 主页 README 与各仓库元数据/README），仅覆盖公开信息。*
