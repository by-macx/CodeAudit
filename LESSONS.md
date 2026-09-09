# LESSONS — 普适问题档案

> 2026-09-05 repo 体系重构与生产收敛中发现并修复的问题立档。"普适"= 剥离具体
> 文件名后仍会在同类工作中复发的模式。操作清单（怎么防）在
> [deploy/README.md「迁移/重构后自检清单」](deploy/README.md)，本文是问题本体
> （是什么、为什么）。每条含：现象 → 根因 → 普适教训 → 防线落点。

| # | 问题模式 | 修复锚点 | 防线 |
|---|---|---|---|
| 1 | 契约迁移后调用点残留，mock 测试测不出真实链路断裂 | engine `bf2d6570`(ADR-202)、`bd6affa3`(ADR-203，并行会话) | e2e 套 + fakeGateway 测试台 |
| 2 | 字符串引用不随 git 移动更新 | umbrella `34541c0`、manager `41fb1a3` | 自检清单第 1、2 步 |
| 3 | gitignored 单一事实源不跟仓走 | manager `26eff4a`；opengrep 恢复见 PROVENANCE.md | 自检清单第 3 步 |
| 4 | 部署配方落后源码一个大版本 | manager `41fb1a3`、dsh-pentest-sse `6841f06` | 配方与源码同原子变更纪律 |
| 5 | 文档与代码失真（未经实测的断言） | gateway `7326e27`、manager `41fb1a3` | README 验收=逐行核对代码+实测 |
| 6 | 手工运维残留使 compose 无法接管 | LXC 侧处置（2026-09-05 收敛） | 自检清单第 4 步 |
| 7 | 叠加式同步使部署目录单调膨胀 | LXC 侧换血（见下） | `check` 确定性哈希 |
| 8 | 结构化配置改动未经 parse 验证 | engine `a8a33c49` | 提交前 parse 一遍 |
| 9 | 多智能体会话并行写同一文件 | web 侧 ADR-202/203 期间互覆写，按裁决恢复 | 按文件归属分工 + 后发指令优先 |
| 10 | 测试架构盲区：部署接线缺陷对全部既有测试隐形 | 伞仓 235f1ce 前后一系列 GUI 实测修复（engine 8de1a4d9/c24fb917 等） | check-wiring 静态审计 + e2e 入口多样化（08/09）+ 真实栈必跑纪律 |
| 11 | 遇到新问题先发明新方案，不查往期实证；手工应急成功后不固化 | 智谱 provider `../` 绕过（往期 llm-config.json 原文在先，nginx shim 在后）；kafka 镜像逐个手试拉取 | 新问题先 grep 往期（ADR/账本/兄弟仓 runtime 配置）再动手；手工步骤成功即入工具链/文档 |
| 12 | 传输地址"看起来对"但缺 scheme，类型化客户端才炸且错误远离配置点 | production.env `OPENSHELL_MANAGER_URL` 缺 `http://`（2026-09-09）；Go `http.NewRequest` 报 `unsupported protocol scheme` 于每个下游调用 | 联动键模板/脚本/文档三处同源生成；错误文案含 `unsupported protocol scheme` 即先查上游 URL 前缀 |
| 13 | 外部系统键名约定未经实测即写入 UI/文档，静默失效 | 网关只认大写环境变量风格 config 键（`OPENAI_BASE_URL`/`ANTHROPIC_BASE_URL`），小写 `base_url` 被静默忽略→回落官方端点 401（2026-09-09 探针实测） | UI 预填键名必须来自端到端实测；配 provider 后必跑一次"切路由开验证"自检 |
| 14 | 声明式覆盖默认值时不知道默认值为何物，覆盖件比被覆盖者更弱 | 引擎 sandboxSpec 携带极简 policy 整体覆盖镜像完整 policy.yaml，supervisor 自举即被 Landlock 拒→ContainerExited（2026-09-09 双沙箱 A/B 实证） | spec 里不复述"镜像已权威携带"的配置；A/B（带/不带）是覆盖类故障的标准定位法 |
| 15 | 多层代理链中每层都"原样透传"，无人负责语义对齐（协议/模型名） | DSH 只说 OpenAI 协议而路由配了 anthropic 型 provider→`no compatible inference route`；模型名默认 `deepseek-v4-flash` 原样透传上游必拒（2026-09-09） | 代理链每个环节把"自己消费的语义"显式注入（dsh-runtime 按路由注入 DSH_MODEL+anthropic 型 fail-loud）；跨层契约靠 e2e 真跑而非各层自述 |

## 10. 测试架构盲区：部署接线缺陷对全部既有测试隐形

现象（2026-09-05 一键重建 + GUI 实测一次暴露四类）：①「新建项目→上传→自动任务」
start 必 409（task→project/storage 服务间地址 env 缺覆盖，容器内回落 localhost 拨
自身）；②SAST 0 发现/沙箱 32 字节空包（任务源目录无共享卷，各容器各扫各的空目录）；
③sim overlay 四服务重复键，整组 env 被 lenient 解析器吞掉；④prod overlay 缺
storage 生产档位（通知恒空、文件不落 MinIO）。单元/契约测试全绿、e2e 07 用例
"通过"——四类缺陷对它们全部隐形。

根因四层叠加：
1. **e2e 写而未跑**：tests/ 套件提交时标注"容器执行待 docker 宿主"，此后 sim 栈
   因 overlay 重复键根本起不来——套件实际是死代码，"e2e 通过"从未发生过；
2. **测试入口单一**：04 用例任务自带 `config.upload_file_id`，永远走"幸运路径"，
   不触项目 config 兜底链（task→project/storage RPC）；GUI 用户的真实入口序列
   （建项目→上传→自动任务）无任何自动化覆盖；
3. **接线无契约**：compose env 覆盖、共享卷挂载、存储档位是"部署层契约"，
   既无单测也不属于任何人的检查清单——yaml 缺省回落 localhost 的错误由降级链
   静默吞掉（fail-silent），与 ADR-137 的 fail-loud 精神相悖；
4. **断言粒度太粗**："到达 COMPLETED"对"SAST 实际扫到东西"零证明力——空目录
   扫描照样能走完状态机。

普适教训：**mock 与真实栈之间的地带（部署接线）必须有它自己的测试层**；测试
入口必须覆盖用户真实路径而非实现方便的路径；"从未真实跑过的测试"比没有测试
更危险（绿灯是假的）。防线落点：`deploy/check-wiring.py`（接线契约静态审计，
挂 sandbox-deploy check 与 production-deploy 预检）+ e2e 08/09（用户路径回归，
断言发现数与可观测面非空）+ 纪律：tests/ 套件入库的同一个变更原子必须含一次
真实执行的证据。

## 11. 先查往期实证，再发明新方案；手工应急成功即固化

现象（2026-09-05 同日两例）：①网关推理 provider 对智谱端点 404/400，在未查
往期的情况下搭了 nginx 路径改写 shim——而往期实证（four-direction-pentest-engine/
runtime/llm-config.json）早已给出正解：base_url 写 `https://open.bigmodel.cn/
v1/../../api/coding/paas/v4`，用 `/v1/../` 前缀抵消网关对 openai 型追加的
`/v1` 段（服务器端归一化归位），shim 属于多余基础设施；②bitnami/kafka 拉取
失败时逐个手动试镜像源，被叫停后才发现该把多源回退写成 pull-images.sh。

根因：外部服务适配类问题（URL 拼接/鉴权/网络出口）几乎必然在历史上出现过——
本工作区跨 7 仓 + 伞仓账本 + 兄弟仓 runtime 配置，实证散落面大；手工应急的
"成功知识"只存在于操作者脑中，不沉淀则下次复发。

普适教训：**动第一步之前先 grep 往期**（关键词：问题域名 + provider/配置名 +
报错原文片段；检索面=伞仓账本、engine decisions/status 归档、兄弟仓的
runtime/*.json 与 README），有实证照抄实证，无实证才设计新方案；手工应急一旦
验证成功，同一变更原子内把它固化成脚本/文档（本例最终形态：manual-test-guide
「推理 provider 配置方法」+ production-deploy 横幅指引）。

## 1. 契约迁移后调用点残留，mock 测试测不出断裂

现象：ADR-148→ADR-200 接口迁移后，`/tasks/new` 上传路径不再自动填入、
ProjectsPage 上传链路断裂——而前端单测全绿。根因：单测 mock 在 HTTP 边界，
只证明"代码符合 mock"，不证明"链路可用"；迁移时未 grep 全部调用点。
普适教训：**契约/接口迁移的验收 = 全量调用点 grep + 至少一条真实端到端流**；
mock 数字的绿灯对迁移类破坏零证明力。防线：deploy/tests e2e 套（黑盒走
HTTP 面）+ web fakeGateway 测试台（并行会话引入）。

## 2. 字符串引用不随 git 移动更新

现象（一次审计三连）：deploy.sh 默认 `SRC` 指向已归档路径；分发器默认读
`deploy.toml`（实际文件已改名 `sandbox-deploy.toml`——默认值是旧世界说明该链
自迁移后从未运行）；清单四个 `dir` 全部指向不存在目录。根因：移动/改名只改
了文件本体，字符串引用（脚本默认值行、注释、清单）不跟走，且无任何门禁。
普适教训：脚本默认值行是**最高发**位点——跑不到的默认值烂得最久；注释里的
旧仓名次之。防线：自检清单 grep 步 + 只读 `check` 全链。

## 3. gitignored 单一事实源不跟仓走

现象：`manager/deploy/env`（token 单一事实源）在重构迁移中未交接，fresh
clone 即缺，deploy 在 `pct push` 处必败；同类：`opengrep` 二进制（只入库
PROVENANCE+sha256，实物 gitignore）在新检出缺失，sast-adapter 镜像必构建
失败。根因：设计上"密钥/大件不入 git"与运维上"脚本硬依赖该文件"叠加，
缺一个"如何重建"的显式出口。普适教训：**每个 gitignore 的被依赖文件必须有
配套的重建命令写在仓里**（fail-loud 提示 + README/PROVENANCE），不是只写
"不入库"。防线：自检清单第 3 步（含两条重建命令）。

## 4. 部署配方落后源码一个大版本

现象：源码 ADR-174 已 FastAPI 化，`deploy/Dockerfile.manager` 仍停在
stdlib 1.0.0 配方——照此部署构建出起不来的容器；同构问题：compose 镜像
tag 落后（1.0.0 vs 现役 2.0.0）、dsh-pentest-sse 默认 `IMAGE` 落后两个版本。
根因：当时源码仓与部署 overlay 分属两仓，改源码的提交"够不着"另一仓的
配方。普适教训：**配方与源码必须同原子变更**——改运行时形态（依赖/入口/
端口/版本）的提交不得绕过同目录部署配方；结构性解药是同居一仓（本次重构
已完成），纪律防的是同居之后的手滑。防线：评审纪律 + `check` 漂移比对。

## 5. 文档与代码失真（未经实测的断言）

现象：gateway README 称"发布 8081 health"，实测容器内仅绑 loopback、健康
端点无桥接监听，宿主侧不可达；manager README API 表写 `{id}`，实际路由参数
是沙箱名（ADR-173 name→UUID 内解析），且漏掉 `GET providers/{name}` 路由。
根因：文档写于实现之前或转手转述，从未对照代码与运行时验证。普适教训：
**README 的验收标准 = 逐行核对代码 + 关键断言实测**（curl 一下就知道 8081
通不通）；"发布即可用"这类断言最容易想当然。防线：本文档模式——重写即实测
（gateway/manager README 均按此法重做过一轮）。

## 6. 手工运维残留使 compose 无法接管

现象：manager 容器被绕过 deploy.sh 手工 compose 起过（标签 workdir 不符）、
kafka/redis 是无标签 `docker run` 产物——正式 deploy 时容器名冲突或拒绝
接管，链路中断在最后一公里。根因：应急手工操作不留痕、不复位。普适教训：
**生产容器只允许经 deploy.sh 变更**；确需手工应急，事后必须补一次正规
deploy 收敛。处置顺序：先 `compose build` 确认新镜像可建 → `docker rm -f`
旧容器 → 正规 `up`（中断秒级，状态在卷与 .env 不丢）。

## 7. 叠加式同步使部署目录单调膨胀

现象：生产构建上下文 `/root/os-deploy/deploy/codeaudit` 积到 **18,467 个
文件**（本地仓 374）——历次 tar 叠加同步从不删除，旧版已删文件、evidence
截图、误产物全在里面，漂移检查永远报漂移。根因：同步语义是"覆盖+新增"，
无对账删除；且曾长期无人运行 `check`。普适教训：漂移检查"永不过"时优先
怀疑**远端残留**而非漏同步；构建上下文目录是纯派生物（容器跑镜像、状态在
卷），可安全换血——备 `.env` → 目录改名退役 → 验证后删除 → 全新同步。
防线：`check` 的确定性哈希比对（本次正是它把问题逼出来的）。

## 8. 结构化配置改动未经 parse 验证

现象：engine compose 的 redis 服务带两个 `ports` 键（ADR-199 加 6379 发布
时旧同值块未删），compose 严格解析直接失败，生产重部署在 parse 阶段退出。
根因：YAML/TOML 是"看起来对"就能提交的语言，无编译器兜底；该缺陷随 ADR-199
入库多日，直到下一次真实部署才爆。普适教训：**结构化配置（YAML/TOML/JSON）
改动提交前必须 parse 一遍**（compose config / python yaml 查重键 / toml
解析），一行命令的成本低于一次部署失败。

## 9. 多智能体会话并行写同一文件

现象：本会话与并行会话在 ADR-202/203 期间对 ProjectsPage/TaskNewPage 及其
测试互相覆写，一度出现"我恢复的版本又被并行会话重写"的循环。根因：两个
会话各自持有完整工作副本，无文件锁；裁决靠人类后发指令。普适教训：**并行
会话按文件归属分工**（一方认领的文件另一方只审不改）；冲突已发生时以时间
上更后的人类指令为准，机械回滚对方改动会破坏已授权工作；提交前 `git status`
核对文件归属。防线：本档案 + 会话记忆（裁决先例）。

## 12. 传输地址"看起来对"但缺 scheme（2026-09-09 生产实测）

现象：`OPENSHELL_MANAGER_URL=host.docker.internal:18800` 在 Python/curl 语境
无感，Go `http.NewRequest` 却在每个下游调用处报
`unsupported protocol scheme "host.docker.internal"`——dsh-runtime 推理管理面
与沙箱通道全断，而 env 值"肉眼看着是对的"。根因：host:port 形态对部分客户
端是合法 URL（curl 自动补 scheme），对要求绝对 URL 的类型化客户端不是；错误
抛在调用点而非配置读取点，离配置错误隔了一整层。普适教训：**跨语言共享的
地址常量必须显式带 scheme**，且由单一事实源（模板/脚本）生成所有出现点。
防线：production-deploy.sh set_env_kv 写入带 scheme 值；三处文档常量已同步；
错误文案含 `unsupported protocol scheme` 时第一反应该查上游 URL 前缀。

## 13. 外部系统键名约定未经实测即写入 UI/文档，静默失效（2026-09-09 探针实测）

现象：GUI 预填 `config: {base_url: …}`，用户照填后切路由验证报上游 401——
网关只认大写环境变量风格键（`OPENAI_BASE_URL`/`ANTHROPIC_BASE_URL`），小写
`base_url` 被静默忽略、端点回落 api.openai.com/api.anthropic.com，真 key 在
官方端点当然 401。同族：UI 预填的 `zhipu`/`deepseek`/`openai-compatible`
类型根本不在网关支持集（openai/anthropic/nvidia/deepinfra/google-vertex-ai/
aws-bedrock）内。根因：键名/枚举来自想当然而非网关契约实测；"静默忽略+回落
默认"让错误表现为远端鉴权失败，极具误导性。普适教训：**UI 预填与文档示例
里的每个键名/枚举值必须端到端实测过**；外部系统有"验证"入口（如切路由的
连通性验证）时把它当自检标配。防线：ProvidersPage 预填改为实测契约
（类型全集+类型感知大写键）并有 vitest 钉死；manual-test-guide 配方本就是
实证版。

## 14. 声明式覆盖默认值，覆盖件反而比被覆盖者更弱（2026-09-09 双沙箱 A/B 实证）

现象：任务沙箱 100% 于 Provisioning 期 ContainerExited（error phase），而同一
镜像手工创建的沙箱全部 READY。A/B 实验定位：引擎 sandboxSpec 携带的极简
policy（read_only 仅 /skills,/tools,/input）**整体覆盖**了镜像内完整的
/etc/openshell/policy.yaml（read_only 含 /usr /lib /proc /etc …），supervisor
自举阶段自己都读不了基础路径，容器立毙。根因：spec 编写时不知道镜像 policy
才是权威且更完备；覆盖语义是整体替换而非合并，弱覆盖件把强默认值抹掉了。
普适教训：**spec 里不复述"镜像/基座已权威携带"的配置**；出现"同镜像两处创
建一活一死"时，第一怀疑点就是 spec 覆盖面，A/B（带/不带该字段）是标准定位
法。防线：sandboxSpec 已去 policy 并留注释锁死；fakeManager 断言 spec 不得
携带 policy。

## 15. 多层代理链无人负责语义对齐（2026-09-09）

现象：链路每一层都"正常"——DSH 正常发请求、网关正常路由、凭据正常注入——
整链却 404/401/`no compatible inference route`。三个独立实例：①DSH 唯一适配
器说 OpenAI 协议（/v1/chat/completions），路由却配了 anthropic 型 provider
（只收 /v1/messages）→ 网关正确地拒绝；②模型名默认 `deepseek-v4-flash` 原样
透传，上游是智谱 → 必拒；③网关固定拼 `base+/v1/chat/completions`，直拼智谱
openai 端点多出 /v1 → 404（无 key 探测还被"先查鉴权头"的 401 掩护）。
根因：每层契约（协议、模型名、URL 拼接）都隐式依赖下一层"恰好兼容"，没有
一层显式声明并校验自己消费的语义。普适教训：**代理链中每个主动方把自己
消费的语义显式注入/校验**（dsh-runtime 现按路由注入 DSH_MODEL、对 anthropic
型路由 fail-loud）、**上游路径结论必须带真凭据验证**（401≠路径存在）。
防线：session.go 路由前置解析 + external-interfaces/data-flows 文档补 DSH_MODEL
契约；智谱 openai 端点用 `/v1/../` 归一化配方（manual-test-guide 实证版）。
