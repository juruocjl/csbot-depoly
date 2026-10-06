# 本地交付验收

## 2026-10-05 AI 对话短链接

- QQ 进度和补充回复使用固定短 ID，默认 UUID 末 12 位，碰撞时延长新记录的 ID；前端解析后替换为完整 UUID，再订阅原 SSE。旧长链接兼容，访问权限与对话列表一致。
- 后端 `scripts/check_ai_short_ids.py` 6 项通过：12/16/20 位冲突分配、旧链接不改向、重启稳定、旧 SQLite 补齐、映射失败事务回滚、群/个人权限、大小写兼容，以及实际 FastAPI 路由鉴权先于解析、越权/缺失同为 404。仅临时 SQLite 和模拟登录依赖，不连接业务数据库。现有对话列表 1 项、边界 4 项、QQ 图片/补充通知与源码 2 项回归通过，平台发送仍为模拟 OneBot。
- 前端短 ID 4 项与过程显示 4 项检查、生产构建通过；新解析工具的独立严格 TypeScript 检查通过。现有全量 `vue-tsc` 在执行前因旧版工具与当前 TypeScript 不兼容报 `Search string not found`，不计为通过。
- 本地 Chrome 实际运行构建页面，模拟 API 验证：未登录保留短链接到登录回跳、规范化保留其他 query/hash、解析后只打开一次完整 UUID SSE、长链接跳过解析、404 不开 SSE、解析期间返回列表后迟到响应不跳回。无真实模型调用、业务库访问或 QQ 测试消息。
- 后端 `f4a27c329a06ad8ad27a0631d0c372f2cc578bfb`、前端源码 `82000a3f7efbe27ca8fd29f3af33e32013f6f567` 已推送并核对 GitHub main 一致；对应 [Actions 构建](https://github.com/juruocjl/csbot-front/actions/runs/37304939856) 成功，产物 `a24f8dd83982a96f0ff30c4683d9a0f8c4ec3c99` 的提交信息确认来源为该源码版本。新增运维 SQLite `run_aliases` 表的备份与发布顺序见 [AI_RUNTIME.md](AI_RUNTIME.md#对话短链接)。
- 用户随后授权部署。独立 Linux 验收目录经 GitHub `pull --ff-only` 更新到 `f4a27c3`，上述后端检查和代理抓取 9 项通过；21:57 生产完成后端及前端产物快进更新。现有令牌通过真实生产 API 验证长短 ID/大小写解析、未登录拒绝、缺失/无效 ID 返回 404；全部 267 条旧记录已有固定映射，静态 JS HTTP 200、类型为 JavaScript 且包含解析接口。线上没有其他群或其他用户的记录可作越权样本，跨群/个人权限依靠独立 Linux 夹具验证；未新建令牌、生产测试记录或 QQ 消息。生产代理工具通过本机 7890 读取 example.com，HTTP 200、未截断，正文含 Example Domain。服务、版本及备份见 [服务清单](SERVER_SERVICES.md#2026-10-05-ai-短链接与代理抓取发布)。

## 2026-10-05 AI 可选择代理抓取

- 新增 `web_fetch_proxy(url)`，通过预设回环 HTTP 代理抓取匿名公网页面，保留直接 `web_fetch`。默认端口按服务器只读配置核实为 Mihomo HTTP 7890；模型不接收代理地址或认证参数，`CS_AI_FETCH_PROXY_URL` 空值可关闭代理工具。配置与访问边界见 [AI_RUNTIME.md](AI_RUNTIME.md)。
- `node --test scripts/check_ai_proxy_fetch.mjs` 本地9项通过：固定代理配置、CONNECT目标为校验后的公网IP、原Host和TLS SNI/证书校验保留、私网IP/localhost/认证URL拒绝、同源与跨域重定向、响应限额、超时/取消和代理故障不自动直连。TLS使用临时自签测试证书和模拟代理，密钥已随临时目录清理。
- `scripts/check_ai_web.py` 使用模拟模型/搜索和回环代理完成真实DSH工具协议验收：模型schema含新工具、代理HTML正常解析、内网/认证URL拒绝、只发生一次预期CONNECT，直连/代理/搜索共用预算，搜索凭据不进入模型请求。`check_dsh_runtime.py` 现有会话/记忆/文件/图片/输出审查回归、源码白名单和语法/差异检查通过；不连接业务库或发送QQ测试消息。
- 独立 Linux 验收工作区经 GitHub `pull --ff-only` 更新到后端 `4409b60eb2db17a13f81581f7c014eb11b67df63`，Node 24.14.0 下代理9项、DSH网页工具集成与源码2项检查通过。新 provider 从该工作区实际通过 `http://127.0.0.1:7890` 读取 `https://example.com/`，HTTP 200、HTML正文含 Example Domain、未截断，单次约1.04秒；未调用真实模型、业务库或QQ。生产后端仍为 `e1af04a`，本轮实现已推送，尚未生产发布。

## 2026-10-03 斗鱼开播与回放监控修复

- 后端修复提交 `e1af04ad26e129bfa397a462acfa7eacd6287d70` 已推送 GitHub。移除已停服的“在看直播”（`doseeing.com`）第三方 HTML 抓取，改读斗鱼官方 `betard` 房间信息；靓号失败后只从官方移动页面解析真实 ID，再读取带回放字段的房间接口。未采用会将 `6657` 返回成另一房间的旧 `open.douyucdn.cn` 接口。
- `checks/live_watcher_test.py` 在本地 Python 3.12 和服务器独立临时 Git 工作区均通过 14 项合成测试：开播/下播/平台回放、标题重播、缺失或未知字段、房间身份匹配、靓号解析、超时、移动 `isLive` 不充当备用状态、回放不通知、重复直播不重复提醒、查询失败保持上次状态并继续其他房间、取消保留状态快照。无数据库访问，OneBot 与状态写入均为模拟。
- 本地 `checks/runtime_config_logic_test.py` 13 项通过，修改文件语法及差异检查通过。未新增依赖、修改数据库配置或运行迁移。
- 在生产服务器的独立验收工作区运行 `checks/douyu_live_probe.py`，通过实际公开网络读取 5 个房间：`dy_6979222` / `dy_6657` 均为玩机器、offline；`dy_2267291`（17shou丷）为 replay、`islive=0`；`dy_9999`（yyfyyf）为 live、`islive=1`；`dy_100`（斗鱼吃鸡赛事）标题为“【PUBG鱼乐赏金赛】DAY3重播”，判为 replay、`islive=0`。当次预期均通过 `--expect` 断言；房间状态可能随后变化。
- 回放样本 `2267291` 实际字段为 `show_status=1, videoLoop=1`，移动页面却为 `isLive=1`；赛事房间 `100` 实际字段为 `show_status=1, videoLoop=0`，需由明确重播标题过滤。未仅靠合成回放宣称通过真实回放验收。
- 标题过滤是保守策略，任何标题包含“回放/重播/录播/轮播”均不提醒；手工播旧视频若既无标志也无标题说明，无法从这两个字段可靠辨认。用户明确授权后已上线，15:11 再次只读实测上述 5 个房间的当次预期仍全部通过。未向真实 QQ 群发送测试消息，版本、备份与依赖检查见 [SERVER_SERVICES.md](SERVER_SERVICES.md#2026-10-03-斗鱼监控修复上线)。
- 15:12 首轮生产自然调度实际读取现有四个房间，日志含“玩机器丶Machine 0”，无抓取错误；对应数据库状态为 0，后端稳定且无自动重启。未调用调度函数制造测试通知，也未修改名单加入回放验收房间；回放通知抑制由真实接口探测与模拟 OneBot 的实际监控函数测试共同验证。

## 2026-10-01 @ 对象历史显示名（atv2）

- 新归档片段为 `['atv2', QQ号, 记录时群内显示名或null]`，只涉及被 @ 对象，不保存发言者昵称；旧 `at` 原样可读。显示文本同时保留 QQ 身份标记，取名失败不借当前昵称补历史。
- `scripts/check_ai_outgoing.py` 仅使用 `csbot_backup`、合成群号与模拟 OneBot：验证发送前快照经延迟发件箱索引仍为旧名，接收消息在记录时捕获名片，改名后原记录与 AI 提问格式仍显示旧名，@ 元数据不进入发给 OneBot 的载荷；旧格式和 `@全体` 可读。测试合成行已清理，未发真实 QQ 消息。
- `scripts/check_ai_image_reply.py`、`scripts/check_ai_source.py` 和语法/差异检查通过；源码白名单按审查后哈希扩为 14 个文件。AI 对话列表和详情使用已持久化的请求文本，因此无需前端改动即可显示新记录里的名字。旧记录没有该快照，不能通过重建索引恢复。
- 生产发布后在实际后端工作区运行 `scripts/check_ai_source.py` 通过（2 项）；服务、容器、HTTP 和 Steam 就绪检查见 [SERVER_SERVICES.md](SERVER_SERVICES.md)。生产未注入合成 @ 记录或给 QQ 群发送测试消息，因此真实 OneBot 群名片查询的端到端结果仍待新消息自然出现后观察。

## 2026-09-29 群成员头像查询纠错

- 生产历史会话`1a11308d-9ad3-4186-a607-5e645d771cfe`只调用了`execute_python`探查SDK和`read_image(https://q1.qlogo.cn/...)`，未调用已存在的`member_avatar`；文件工具拒绝URL后模型误称无法读取头像。没有据此断定该成员头像可用或不可用。
- 新系统提示与DATA.md给出`member_avatar → full_path → read_image`的明确顺序；DSH对URL误用给具体修复提示。`check_dsh_runtime.py`协议回归及`check_group_knowledge.py`五项群查询/头像边界测试通过。
- `check_ai_avatar_route.py`在独立Linux验收副本以服务身份、现有DeepSeek测试配置、合成群成员和合成头像运行，不访问生产QQ或业务库。最终回放实际调用`member_avatar`和`read_image`，读图工具无错误，回答正确描述白底蓝圆且未输出内部状态字段。初版回放虽然成功读图但泄露`full_status`字段，收紧日常回复规则后重新验收才计通过。该合成案例不能证明所有真实QQ头像/CDN时刻都可达，也不能保证模型永不选错工具。

## 2026-09-29 自我话题的轻微傲娇语气

- 系统提示将轻微傲娇限定在聊AI自己、收到夸奖或调侃时；模型和能力等事实问题仍直接回答，普通数据查询不套该语气。第一次真实模型回放把虚拟尾巴说成能“掀翻”对方，超出预期，因此增加不以虚拟身体威胁/吓唬的边界后重新回放；第二次回答简短接下夸奖，未作暴力夸张。另一次直接询问模型时准确答出DeepSeek及`deepseek-flash`。
- `check_dsh_runtime.py`完整通过，覆盖真实DSH系统消息、续接会话、Mneme提炼、工具权限等协议；真实回放使用临时隔离会话，不查业务库、不写群记忆、不发QQ。语气有生成波动，单次回放不能保证每次都一样。

## 2026-09-29 DeepSeek 身份与虚拟形象

- `check_dsh_runtime.py` 本地模拟提供方完整通过；实际首次请求包含 DeepSeek 群友身份、蓝色长发/蓝白女仆装/鲸鱼尾巴形象和本轮模型标识，续接会话仍保留完整系统提示。没有改 DSH/Mneme 依赖或上游代码。
- 临时独立会话调用现有 DeepSeek 测试配置和 `deepseek-flash`，提问模型与形象；回答确认当前标识为 `deepseek-flash`，并准确描述蓝色长发、蓝白女仆装、鲸鱼尾巴。未连接业务数据库、保存群记忆或发送 QQ 消息。单次回答验证这一配置可生效，不保证每次措辞相同。

## 2026-09-27 统一图片语义归档验收

- `scripts/check_image_archive.py` 本地/Linux四项通过：描述符校验与128KiB超限回退、图库文件更改/删除/符号链接拒绝、页面参数白名单、截图期间内容变化回退原图。
- `scripts/check_image_capture.py` 在独立Linux验收目录运行实际网页和HTML截图函数，使用合成HTTP页面，验证发图PNG、文字/图片标签快照和cookie不进入metadata；不访问业务数据库或发送QQ消息。
- `scripts/check_ai_outgoing.py` 在本地和Linux使用 `csbot_backup`、模拟OneBot验证：普通图片仍归档；快照可检索和按群读取；QQ线上载荷保留真实像素且无内部描述符；快照原图/缩略图未创建；回流不下载；合并转发和索引故障后metadata仍在outbox，补索引不重发。合成记录已清理。
- 现有QQ AI图片/补充回复、本地DSH原生协议、文件/SQL边界、源码映射和测试配置插件加载验收通过。图库只是引用，缺失/被替换后不会拿新文件冒充旧图；独有图与未知图保留原有LRU。
- 生产发布完成后，源码映射13文件校验、现有令牌只读受保护API及越权拒绝通过，服务状态记录见SERVER_SERVICES.md。未向真实QQ群发送验收图片；真实OneBot群发仍由正常使用验证。

以下 PowerShell 命令从部署仓库根目录开始；后端和前端分别在独立终端运行。
先阅读 [执行约束](AGENTS.md)；真实测试令牌由现有安全渠道提供，不写入文档。


Use the remote test database `csbot_backup` and the existing test token supplied securely as `CS_TEST_TOKEN`.
Do not create a new token for local delivery checks. Provide the real remote database URL through
`CS_DATABASE`; it should point at `csbot_backup`.
If the remote PostgreSQL is reached through SSH, forward it locally first and rewrite the host/port
in `CS_DATABASE` to the forwarded address.

Backend:

```powershell
cd ./csbot
# Example shape only; use the real testing credential.
$env:CS_DATABASE="postgresql+asyncpg://<user>:<password>@<host>:5432/csbot_backup"
$env:CS_SERVER_SKIP_STARTUP_CACHE="1"
$env:CS_SKIP_SCHEMA_CHECK="1"
$env:CS_DISABLE_BACKGROUND_JOBS="1"
$env:CS_WATCH_STAGE_ENABLE_PROFILE_REFRESH="0"
uv run python bot_minimal.py
```

Remote watch-stage test instances should keep background crawling disabled. Only set
`CS_WATCH_STAGE_ENABLE_PROFILE_REFRESH=1` for a run where the explicit purpose is to verify
watch-stage-triggered player profile crawling.

Frontend:

```powershell
cd ./csbot-front
$env:VITE_API_BASE_URL="http://127.0.0.1:1234"
cmd /c npm run dev -- --host 127.0.0.1 --port 5173
```

Smoke-test a protected API with:

```powershell
curl.exe -H "Authorization: Bearer $env:CS_TEST_TOKEN" -H "Content-Type: application/json" `
  -d "{\"steamId\":\"76561199388088405\"}" `
  http://127.0.0.1:1234/api/watch-stage/snapshot
```

For watch-stage delivery, verify that the response is not `401`, contains a `status`, and returns
running match data when the tested Steam ID is currently in a watch-stage match. The profile fields
such as `legacyScore` may be `null` on the first response because player base data is crawled
asynchronously and rate-limited.

## AI 改造专项

架构、查询文档入口、凭据边界与发布/回退步骤见 [AI_RUNTIME.md](AI_RUNTIME.md)。以下从后端仓库执行；所有数据库验收只用 `csbot_backup`，平台发送使用模拟 OneBot，不向真实群发测试消息。

```bash
uv run --frozen python scripts/check_ai_boundaries.py
uv run --frozen python scripts/check_streaming_bm25.py
# 需 Node 和 npm ci，使用本机模拟模型，不使用真实密钥：
uv run --frozen python scripts/check_dsh_runtime.py
# Linux + Docker，纯合成数据：
uv run --frozen python scripts/check_ai_sandbox.py
# 从安全环境提供 CS_DATABASE=.../csbot_backup；测试会清理合成行：
uv run --frozen python scripts/check_ai_outgoing.py
# Linux 端到端：安全配置从 stdin 读入，不放在命令参数中；不得用生产库。
uv run --frozen python scripts/check_ai_live.py < /安全路径/test-config.json
```

最后一项 JSON 仅需 CS_DATABASE、CS_AI_URL、CS_AI_API_KEY、CS_AI_MODEL、CS_TEST_GROUP，另设 CS_AI_GAME_DB 为只读游戏库路径。调用模型只传合成指令与聚合数量；本地 SSH 隧道端口和服务器本地端口不同，运行前必须确认连接仍指向 csbot_backup。

2026-09-27 验收记录：

| 项目 | 已验证的内容 |
| --- | --- |
| SQL / 隔离 | 12 个主库模板对真实测试库成功；游戏库真实 schema/联表只读成功；SQL 拒绝写入、系统函数及越权物理表，游戏查询按绑定成员过滤。 |
| 长聊天 | 24,939 条真实测试库候选完成流式 BM25；大量补入上下文约 10.5 KiB、24 个块；DSH 真正压缩后可跨进程恢复。 |
| 记忆 | 模拟与真实模型均完成 Mneme 写入；旁听 sentinel 不进入提炼输入；旧手工记忆重复导入仅保留一份；最终 QQ 审查先于写入和提炼。 |
| 会话 | 同群稳定恢复，网页独立；真实模型跨进程记住合成测试信息；Node 22.22.0 和开发机 Node 26 均通过协议检查。 |
| 图片 | 原图 LRU、租约、缩略图保留、缺失状态和跨群图片拒绝；真实视觉模型用 read_image 识别合成蓝色图片。 |
| 发出消息 | 文本、图片、引用、合并转发进入检索；自发回流去重；索引故障重试不重发；发送结果不确定时保留 unknown。 |
| 隔离执行 | Linux 实测无网络、只读根、无凭据/Docker socket、cgroup 内存/PID 限制；超量输出拒绝并清理。 |
| 完整链路 | 真实模型 → DSH → Docker 脚本 → 测试主库和实际只读游戏库 → 正确返回 1..100 求和结果。 |
| API | 现有测试令牌完成 ask/recordids/record；跨群、跨用户及无归属旧记录 404，同群授权记录 200；中断状态终止轮询，个人历史不串用户。 |
| 触发 | 普通聊天、@全体与机器人自发消息不触发；明确 @ 与回复机器人触发。此项为实际规则函数测试，真实 QQ 受控群的收发仍属上线验收。 |
| 发布前置 | 独立验收目录中，以 ubuntu 加 Docker 组权限通过 Node/依赖/真实容器/两库 schema/健康状态/磁盘检查。未替换生产后端。 |
| 前端 | npm run build 成功；既有大 bundle 告警仍在，不影响构建。 |

测试库原先缺少 chat_token_lexicon、runtime_config 及 span 的 token_text/token_count；补齐前已保存只含结构的私有备份 `.deploy/csbot_backup-schema-before-ai.sql`，未改生产库。临时日志、测试配置和密钥均在忽略目录，0600，不纳入提交。

上线后仍应按 DEPLOY.md 核对服务及实际 commit、前端构建 commit，并在受控群做人工收发验收。自动化测试通过不表示生产已切换，也不表示历史未归档的机器人消息已被补回。

首次交付版本（已核对 GitHub 远端）：后端 `4a8f1cdb75523b12033b8ebbaa7185a3d5f62f1b`；前端源码 `bbc325cb213975c0e6278104d32cb3c830269d69`；前端产物 `3f39da006f755d516c03d4263782485a159ae41e`，其提交消息明确对应上述源码。[前端 CI](https://github.com/juruocjl/csbot-front/actions/runs/36302978629) 成功。

用户随后授权上线，2026-09-27 15:48 CST 已完成以上版本生产切换。受保护生产 API 用独立个人会话完成真实 DSH → Docker Python → 5050 验收，个人历史可取、未知归属记录拒绝；没有向 QQ 群发送测试消息。备份、服务状态、图片数量、内存及后续版本见 [线上发布记录](SERVER_SERVICES.md#2026-09-27-ai-改造上线记录)。

默认人设调整：真实模型使用合成聊天验证吐槽、观点、比赛记录与追问原始值；日常比赛回答保留比分和击杀/死亡数，省略 mid/record_id/秒级时间，明确追问时给 mid 和精确时间。模拟供应商额外确认 DSH 请求实际包含完整的新系统提示。语气为人工检查，模型仍可能追问或展开，不能把一次样例视为每轮都符合主观交流感的保证。

### 实时过程和图片输出验收

新增可重复检查：

```bash
# 本机模拟慢模型，运行真正的 DSH，无真实密钥：
uv run --frozen python scripts/check_ai_realtime.py
# 实际 QQ 回复函数 + 合成图片，模拟生成，不发送 QQ：
uv run --frozen python scripts/check_ai_image_reply.py
# Linux/Docker 原有检查已增加中文 matplotlib 与 send=True 提交：
uv run --frozen python scripts/check_ai_sandbox.py
```

### 回答依据与系统提示传递（2026-09-27）

- 修复 `system-prompt.config` 补丁覆盖 SDK 默认配置而丢失 `personaPrefix` 的问题。模拟供应商实际请求曾只收到通用 Harness/文件工具提示；修复后，本地及 Linux 原生回归均确认首次请求和恢复会话包含完整 `SYSTEM_PROMPT`，记忆上下文和既有隔离/压缩/审查回归仍通过。
- `scripts/check_ai_answer_evidence.py` 从 stdin 读取既有模型配置，使用临时会话、合成记忆及脚本结果夹具，不连接业务库、不运行容器、不发 QQ。打印真实回答供人工检查，不用简单关键词匹配宣称模型语义正确。
- 最终配置下：未知“致敬”检索后明确回答不知道，没有拼出比分前后关系；已确认“老钟”黑话正确解释；点数查询失败不返回零值或伪造数值；闲聊仍能接话。测试中查询失败仍有重复尝试及多余收尾，这些样例不能证明所有生成都满足语气和效率要求。
- 线上发布完成后，现有令牌只读 API（scopes/list/detail/search/archive/pagination）及未认证/越权拒绝通过。首次探测发生在重启窗口，502 不计为通过；重启完成后重新验证成功。

2026-09-27 本轮本地验收：web/QQ 两条路径均在回复完成前收到 reasoning 和工具开始事件；慢工具期间 steer 及时确认；原问题、补充答案保留；全局执行槽串行；结束后拒绝插入。实际原生事件经过前端投影后，工具调用与结果能正确合并。现有 DSH/Mneme 七项协议回归与图片/权限四项边界检查通过。

使用 csbot_backup 及现有令牌，本地受保护 API 已验证：同群其他成员可看群过程/图片，跨群及他人个人会话不可看；游标续读不重复，失效游标 409；断开 SSE 不取消任务；原图缺失 410、授权缩略图仍可读、未认证拒绝。实际 QQ 回复函数可组装生成图片；模拟 OneBot 验证发送后图片可检索、回流去重及故障只重试索引。未向真实 QQ 群发验收消息。

前端 `npm run build` 通过（原有大 bundle 提示仍存在）。本次 AI 页面及引用文件使用临时兼容 vue-tsc/TypeScript 检查通过；仓库原有 vue-tsc 1.x 与当前 TypeScript 不兼容，全仓检查另外发现既有 MatchGPHistory.vue:388、MatchHistory.vue:404 语法错误，不将全仓类型检查记为通过。本次不修改这些无关页面。

真实模型专项脚本 `scripts/check_ai_live_stream.py` 从 stdin 接收仅 CS_AI_MODEL/CS_AI_URL/CS_AI_API_KEY 的 JSON，使用候选镜像 `csbot-ai-python:realtime-candidate` 和临时 DSH 会话；仅合成数据，无数据库访问能力。验证供应商真实 reasoning 在完成前可见、脚本期间原生 steer 成功、中文图显式提交、两个问题均回答。凭据由忽略目录的安全配置经 SSH stdin 提供，不写命令参数或日志。

服务器真实验收结果：候选隔离镜像通过中文字体绘图、图片显式提交、只读根/无网络/无 Docker socket/cgroup 限额及输出超限清理。真实模型专项脚本通过思考流、工具期间补充、原问题与补充共同回答、PNG 提交。

生产经 Nginx 的受保护验收：首个模型事件约 **0.92 秒**，原问题与排队问题共约 **14.89 秒**；观察到真实 reasoning、工具开始时及时插入补充、普通第二请求为 queued、最终两轮 completed。生成 PNG **25,284 字节**可鉴权读取，个人历史携带图片索引，事件游标续读无重复。只使用独立个人会话和合成数据，没有向 QQ 群发送测试消息。该耗时为一次验收样本，不是延迟保证。

QQ 入口补充回归：插入成功仅链接原任务，不发不存在的新记录链接；错过当前轮先登记新任务再通知；模型启动前故障将记录标为 interrupted。`/ai`、角色命令、@/回复触发共用同一逻辑，实际发送依然由 OneBot 包装器归档。

## 旧记忆导入与接口退役验收（2026-09-27）

新增 `scripts/check_legacy_memory_import.py`，使用真正的 DSH/Mneme 和临时目录，模型地址指向不可用回环端口以验证全程无需模型。已通过：精确原文导入、原生检索、重复执行不新增、私有备份、已归档/忘记内容拒绝复活、未知归属拒绝、群/个人物理隔离、后端持有状态锁时拒绝迁移。服务器独立验收目录同样通过。

现有 DSH/Mneme 七项协议回归、QQ 图片/补充回复回归通过；测试库及既有令牌下的 SSE/图片权限、游标续读和淘汰检查通过。AI、日报、帮助插件在禁用后台运行的测试配置中可成功加载；旧记忆命令与 DataManager 读写接口不再存在。测试不运行日报调度、不发送真实 QQ 消息。

生产原表有手工记忆 554 字、旧问答摘要 2914 字、日报摘要 1792 字。离线预览逐字核对发现手工记忆已由此前首次调用迁移写入 Mneme，因此本次不是新增一份记忆，而是核验原文、原生检索及保存回执；旧问答/日报摘要未提升为长期记忆，原表和私有备份保留全部三行。

上线核验已完成：生产 Mneme 原生检索成功并落回执，原 PostgreSQL 三行与备份相同、手工导入条目仍仅一条。现有令牌经线上 AI API 创建独立个人会话，真实模型调用 memory_search，不能检索群导入记忆；AI/日报/帮助插件在生产启动日志中均确认加载。备份和版本见 SERVER_SERVICES.md 的 17:49 发布记录。

### 群资料与规则查询验收（2026-09-27）

- `scripts/check_group_knowledge.py` 5项本地及Linux验收通过：昵称重名分页、QQ角色与竞选状态分离、群外成员与额外参数拒绝、平台失败不伪造数据、头像固定地址/私有缓存/JPEG转PNG、23:55业务日、SQL状态键白名单。`check_ai_boundaries.py` 4项、`check_dsh_runtime.py` 7项回归通过，AI/report/help插件可加载，旧记忆接口未恢复。
- 只读连接`csbot_backup`验证3个新SQL模板（缓存成员40条、竞选键3条、指定用户点数流水50条）；点数聚合与物理表精确群过滤计数一致，未知群无法读到点数与管理员状态。没有在生产业务库运行该测试，也没有写入测试数据。
- 线上使用已有认证令牌和新的独立个人会话，真实模型→隔离Python查询成员、QQ管理员/竞选记录、规则和点数；头像下载后原生`read_image`实际返回image内容。最终版本只需1次execute_python+1次read_image，均成功；未向QQ群发验收消息。
- 首次线上测试发现QQ返回JPEG而缓存扩展名PNG导致DSH拒读，模型靠缩略图恢复。已修复为下载后规范化最长640像素PNG，并用真实模型复验直接读取成功；不能把首次调用动作当作读取成功。

### 业务源码映射验收（2026-09-27）

- `scripts/check_ai_source.py` 通过：13文件逐字快照、函数索引、只读文件权限、退出清理；哈希变化、符号链接、manifest追加环境文件均拒绝。`check_ai_boundaries.py` 4项通过。
- `check_dsh_runtime.py` 当前8项通过，新增真实原生read先读SOURCE.md，再读授权快照中的calc_roll_point/roll_admin源码；宿主文件拒绝、工具数量、群/个人/Mneme及压缩恢复回归保持通过。
- 服务器独立验收目录使用ubuntu用户与docker组运行`check_ai_sandbox.py`通过：/source可搜索函数，修改现有源码与新建文件均失败，无.env.prod/.git/data；原有无网络、无密钥/宿主Docker socket、cgroup限制、中文绘图、产物导出和输出超限清理检查通过。没有重建镜像。
- 最终线上个人会话使用真实模型与原生read/隔离Python完成验收：VERSION.json与部署commit一致且matches_head=true，实际读取setcard_function，并验证只读挂载拒写。模型根据finish之后的控制流指出“设置昵称实际不会加20点”，没有仅复述帮助。测试未查询业务数据库、未向QQ发送消息。

### AI只读列表页验收（2026-09-27）

- 后端`check_ai_conversations.py`本地及Linux通过：本群QQ/report与本人个人记录可见、跨群/他人个人/未知类型排除；预览300字符封顶、30条分页、翻页无重叠、新记录不干扰已有游标、非法游标拒绝。生产使用已有令牌只读验证接口200、分页无重叠、未认证401/403、非法游标400，未创建问题或调用模型。
- 前端Vite build通过；使用既有临时兼容vue-tsc检查AIChat.vue及依赖通过（未宣称修复历史全仓类型检查问题）。对应GitHub Actions运行`36312797928`成功，构建产物提交对应源码`0bfcc3f64b78cfd04f8e3b618a5637f74662dac6`。
- 真实Chrome已登录页面验收：入口“对话列表”，textbox和“新对话”按钮均0；加载更多由30项增至40项；点击既有个人记录读取最终回答“排队验收完成。”，返回列表正常。原chatId详情仍用既有实时过程链路，无新增提问动作。

## 2026-09-27 执行过程顺序修复

- 前端过程由两个分类循环改为单一事件时间线，按 SSE seq 到达顺序插入思考、模型输出、工具和补充；结果更新原工具卡片，连续同块 delta 合并。原生 block-end 不重复追加，多个完整块不丢失；中断尝试保留标记。
- `node --test csbot-front/checks/aiTrace.test.mjs` 四项通过：多轮实时/回放一致、同尝试跨工具分段与并行结果、完整块与重试、补充/历史孤立结果。AI 页面相关 vue-tsc 检查及生产构建通过（保留既有 bundle 大小提示）。
- 使用现有令牌只读取得一条已完成、有权限的真实事件记录（2286 个事件），回放得到 reasoning/tool/reasoning/tool/reasoning/tool/reasoning/text；线上 Chrome DOM 验证顺序一致，三个工具结果及生成图片保留。未调用模型或发送 QQ 测试消息。
- GitHub Actions `36314402109` success，前端源码 `6b25ed3bc43e05bb22966698443b18477fe02086`，产物 `cb1e193c01c86fac8e2aab6230ab0d0babb82f86`；仅静态前端发布，后端无需重启。

## 2026-09-27 记忆浏览验收

- 新增 `scripts/check_ai_memory_browser.py`：四项合成库测试覆盖群/个人范围授权、字面搜索与游标分页、归档/遗忘过滤、数据库内容不变、缺失库不创建、符号链接拒绝、正文长度与截断标记。本机Python3.12及独立Linux验收目录均通过。
- 前端AI页面相关类型检查、生产构建通过（只有既有bundle大小提示）。新增 `/ai-memory`，侧栏/对话页提供入口，无新增、修改、删除控件。
- 线上使用已有测试令牌，只读验证scope/list/detail/search/archive及分页；群有效记忆190条、本人个人scope7个；未登录401/403，未知scope列表/详情404，坏游标400。未调用模型、未发QQ消息、未新增测试令牌。
- Chrome实际操作验证：群列表20→40条分页、详情正文加载、关键词“旧群记忆迁移”命中1条、切换本人个人会话并清空搜索后显示6条；与API隔离结果一致。
- 前端源码 `e860decf1606537904982cd12aa72a730227b77a`，GitHub Actions `36315452646` success，产物 `628b6fe6b7f34b31232b4cc6b41ce529e89973a7`；后端 `abe04873edaf6a5203176ac3817a9e48f6b223c2`。

## 2026-09-27 群聊记忆对象纠正验收

- 本地及独立Linux `check_dsh_runtime.py` 通过：真实DSH/Mneme对模拟模型请求含群聊身份规则，旁听资料仍排除；原生离线save/forget/search可用且不调用模型；会话恢复、文件权限、源码、输出审查、旧导入幂等、压缩恢复回归均通过。Linux首次因SSH缺少Node PATH失败，补齐生产Node路径后重跑通过。
- `check_ai_memory_repair.py` 合成库验证旧内容哈希、群隔离、唯一标题冲突、重复计划无操作、拒绝复活被遗忘确认条目。
- 使用既有模型配置在临时独立scope做真实模型验收，仅发送合成称呼与QQ：普通“小茶是谁”触发原生memory_search，回答正确目标QQ222、与提问者QQ111分开；原生后台提炼完成。模型另尝试了一个脚本调用，验收dispatch拒绝业务数据访问。无生产群数据外发、无QQ消息。
- 生产纠正采用已审查哈希计划：新增1条用户明确确认映射，原生forget停用2条误归身份/游戏数据的条目；memory_search验证成功。重启后授权网页API能找到正确映射且不再返回两条旧错记；未向QQ发送修正或测试消息。

## 2026-09-27 分层记忆验收

- 本地和独立Linux：DSH原生回归通过，覆盖进程重启、上下文压缩、旁听排除、主体分离、自动提升、旧手工记忆原文真正进入模型输入、离线维护不调用模型、权限及输出审查。升级了证据索引后再次检查。
- `node scripts/check_layered_memory.mjs`：超过10000条低层资料不挤占基础层、引用伪造/错主体拒绝、幂等、先保存后纠正旧条目、忘记后缓存失效、低层保留未确认候选、持久化故障不伪装成有效保存、旁听和memory_search结果不能自证。合成库5项测试覆盖tier/category筛选、旧手工确认来源、详情与权限；旧纠正脚本回归通过。
- 真实模型仅使用临时合成scope：明确确认小茶=22222，提问者11111；重启直接召回；44444疑问猜测不提升；明确改为33333后旧映射停用。最终精简证据索引版也通过四项。最初发现主体格式不明确、核验输出预算不足，修复后才判定通过；未向QQ群发验收消息。
- 前端AI页面相关类型检查、构建、4项事件时间线回归通过。GitHub Actions对应最终源码success。线上现有令牌验证层级/分类/详情、主体3024182971、越权及未认证拒绝，并逐字段比对旧手工记忆与发布前备份一致。
- Chrome页面实际显示“基础知识”默认层级、人物称呼与“旧版手工确认”两条；打开手工记忆详情展示完整原文。初始静态资源权限问题已修正，不能只用入口HTML 200判断前端正常。


## 2026-09-27 原生网页查询与语境/记忆修复验收

- DSH 0.1.5-rc.3原生`web_search`和`web_fetch`接入。`check_ai_web.py`本地及Linux通过：真实工具schema、Anthropic结构化来源/引文解析、密钥不进入profile或模型消息、内网/认证URL拒绝、每轮搜索预算、原生过程事件。没有增加联网Python或修改上游依赖。
- `check_layered_memory.mjs`通过低层证据保留、审核拒绝不降层、失败脚本不作证据、存储故障仍报错；原有万条低层不挤占基础、主体纠正等保持通过。`check_dsh_runtime.py`本地和Linux的完整提示传递、恢复/压缩、旧手工原文注入、旁听排除、基础层晋升、输出审查回归通过；边界4项、源码2项、定向记忆维护1项通过。
- 真实模型回放使用临时scope、合成QQ身份、空业务查询夹具及真实公开搜索。夹具会先编译模型脚本，语法错误如实返回非零退出；不连接业务库，不发送QQ。答案由人工审阅，不用词命中宣称语义通过。
- 初版回放未通过：实验flash查到偏题结果仍猜邀请人为群友；线上`deepseek-flash`查到术语后又补成“自行排课蹭课”；`deepseek-v4-pro`也漏掉zdbk，并在另一个Jira/Jenkins隐喻案例补出“包加班餐”。因此没有切换生产模型。删除固定吐槽示例后，明确旅游计划的回复恢复为直接确认，但未知术语的字面误读仍可能出现；提示重写不能被当作通用理解能力已解决。
- 加入有公开来源的基础词义（只定义zdbk/Celechron，不提供本条笑点或私人身份）后，线上模型的回放能把“邀请”关联到课表安排，理解包装成参观/旁听的语气；没有新增真实旅游行程记忆。未知词义回放明确找不到，未新增猜测记忆；明确要求记住的10月3日出发日期仍可保存。存在重复摘要和多余措辞的样本，未声称自然度或去重全面解决。
- 中间无词义回放还出现过把引用行程存为“未证实”资料，因而进一步收紧引用个人资料的提炼：只有本轮当事人确认或明确要求记住该事实才可存为个人行程/经历/偏好，其他引用保留会话原文。既有手工记忆不重新审核或清洗。
- Linux真实搜索与fetch首轮HTTP成功但HTML截断在导航，未按全文成功验收。原生`fetchMaxOutputChars`同时限制转换输入，因此调至300K字符以读到GitHub README正文，保持500KB字节和15秒底层超时。
- 最终无预置词义的负例回放也没有写入被引用人的行程；这次能识别课表邀请的调侃，但把在校身份直接说出，仍有推断过实的边界问题。不能据此宣称模型从此不会误读或臆测。
- 用原生HTTP提供方与原生HTML转换函数定位正文位置：页面约264K字符，200K转换结果只有6999字符导航且无DDL，300K得到10742字符并包含README的DDL功能。未通过改写上游解析器解决。
- 最终Linux真实模型→DSH原生search→fetch复验通过，并断言工具结果实际含README中的DDL功能文本，不只检查HTTP状态。模型根据正文正确列出日程、课表、DDL与成绩查询。最终修复没有重建Docker镜像或改前端。
- 上线后现有令牌只读API通过群/个人scope、列表/详情/搜索/分页以及未认证/越权拒绝；新3条内容和旧条目forgotten状态均与计划一致，旧手工确认原文与备份相同。后端/依赖恢复正常；备份挤压磁盘导致的启动中断和处理记录见SERVER_SERVICES.md。
# 2026-09-30 历史原图读取修复验收

- 生产只读回查会话 `fada27b9-b4b9-4fa1-ac1f-3e840fe4024e`：`csdata.image` 报告 `full_status=available`，原图为 1170×2532 的 JPEG（204901 字节），历史路径却以 `.png` 结尾；缩略图为 118×256 的 PNG。模型先读缩略图，随后读原图时 DSH 报文件扩展名与实际格式不符，于是误称只剩缩略图。原图文件仍在，未被 LRU 淘汰。
- `scripts/check_dsh_runtime.py` 本地 mock 协议回归通过：合成 JPEG 字节保存在 `.png` 路径且无 MIME 元数据时，本轮临时 `.jpg` 视图能被原生 `read_image` 成功读取，归档字节不变；其余 DSH/Mneme 协议检查通过。
- `scripts/check_ai_image_route.py` 在独立 Linux 验收副本用现有模型测试配置、合成图片与模拟图片查询运行。真实模型调用 `catalog → image → read_image`，读图结果 `isError=false`，回答正确描述白底蓝色矩形。没有访问生产 QQ、业务数据库或发送群消息；合成图验收不等于已重新识别该历史真实图片的文字。
- 生产更新至后端 `ba596d7fd4f0df0ce3a9dc757435253b6161cdcc` 后，针对该历史原图验证临时 `.jpg` 视图与归档同 inode，可识别为 JPEG、1170×2532，原归档未改动；服务状态与入口检查见 [SERVER_SERVICES.md](SERVER_SERVICES.md)。

## 2026-10-05 完整图片 metadata 与本轮上下文验收（待生产部署）

- 后端 `1583edcb9b9f712ed5b2f470f957f3c88f4427b4`、前端源码 `f3e63eb5f7da0187b02a55430495e2541e506dc4` 已推送并核对远端。前端 [Actions 37329704087](https://github.com/juruocjl/csbot-front/actions/runs/37329704087) 为 success，对应 build-output `46631aa299731c0f09c34a83ec0b755f085ddc4e`。本轮仅更新独立验收目录，没有更新生产代码、重启服务或发送真实 QQ 消息。
- 覆盖网页截图、HTML 排行/战绩/比赛记录/队友/报价、词云、既有图表与图库入口。新索引与旁听资料仅带 metadata 短引用，完整生成输入由 `csdata.image` 按群权限读取。网页 API 响应在前端 schema 过滤前收集，比赛与玩家页另附全部原始模型列；完整 metadata 超过128KiB或不可取得时保留原图，不截断后冒充完整数据。没有迁移、索引重建或历史快照改写。
- 本地图片归档6项、上下文4项（含原生嵌套 system/message、同轮匹配、缺失/歧义、展示截断、群与网页发起人鉴权）、查询边界4项、源码2项、短链接6项及 QQ 图片/补充提问回归通过。真实 DSH mock 协议在本地和 Linux 通过，检查系统提示和实际被动上下文均产生 context 事件；既有会话恢复、压缩、工具边界、Mneme 与输出审查保持通过。
- 实际 Chrome/pyppeteer 截图函数在本地和 Linux 使用合成页面，验证真实 PNG、完整响应/原始统计字段、认证值排除、缺失 metadata 保留像素。Linux `check_ai_outgoing.py` 仅连接 `csbot_backup`、模拟 OneBot，验证短索引不展开比分而完整 metadata 仍可授权读取、发件箱/合并转发/延迟索引保留 metadata，合成行已清理。本机15432测试隧道未开放，因此该数据库验收在独立 Linux 目录完成。
- `csbot_backup` 的受群权限限制逻辑查询实际返回完美45列、官匹44列，包含首杀、爆头、残局、KAST、下包等网页外统计。未查询生产业务库验收这些新逻辑。
- 前端11项 metadata/上下文/事件排序/短链接测试、工具模块 strict tsc 和 `npm run build` 通过。实际 Chrome 验证响应中的未渲染字段在 Zod 处理后仍留于 metadata；上下文面板显示两条合成原始资料与归档来源，无伪造工具行，登录/短链接回归通过。完整 vue-tsc 仍受既有 TypeScript 工具兼容问题影响（supportedTSExtensions 搜索失败），不能报告完整 Vue 类型检查通过。
- 用户提供的真实旧记录 `8f51c136-ef0f-4fee-9582-7f4f06d73f44` 使用新恢复函数只读核验，取得系统提示、本轮身份/风格/群资料、运行时资料与记忆三部分；包含被回复消息505332、原比赛ID及21/6/2数据，未截断。没有重放模型或补造当时输入。旧图片未归档的统计字段仍无法恢复；面板展示本轮实际注入，续用会话历史不是完整 provider 请求转储。

## 2026-10-06 图片成员身份关联验收（待生产部署）

- 后端 `13103fd6dd76cb09d193b153d3ac24d9375cddb1` 补齐图片工具返回的 `identity_context`，只在实际读取 snapshot 时查询当前授权群的 SteamID→QQ 绑定，并取得当前群名片/QQ昵称。保留多重绑定和截断标志；读取失败明确状态，QQ不可用明确缓存来源，当前群外QQ不给当前名字。不修改历史图片 metadata，不把当前绑定伪装成发图时绑定。前端未改，继续在实际工具结果中展示该资料；本轮没有生产更新/重启。
- 本地及独立 Linux 验收 `check_image_identity.py` 7项通过：游戏重名按SteamID分别对应QQ和群名片、改群名片、多重绑定、截断、当前QQ不在群、QQ/SQL不可用、普通图片不查绑定、群SQL隔离、原metadata不变、实际image工具派发、队友主体ID、实际matplotlib分数图的明确QQ/SteamID字段。首次队友修复的测试发现头像变量提前使用，已修正后全部通过。
- 队友卡来源ID不会再被最后一位队友覆盖；分数趋势metadata直接保存QQ/SteamID/游戏昵称，不再仅用组合标签表达身份。更新已审查CS源码哈希，源码2项、查询边界4项及Linux DSH mock完整回归通过，没有真实模型或QQ发送验收。
- 历史仅归档昵称而没有可靠ID的图片无法凭当前同名可靠反推身份；本次补齐新metadata的身份链与当前身份查询，不能报告旧记录的历史身份已恢复。

- 独立 Linux 的新版本实际连接 `csbot_backup`，受群限制的身份SQL返回37条 SteamID/QQ 绑定，QQ分支为模拟离线，确认缓存来源及无群名片，不调用生产OneBot、不写数据库。仅输出数量与校验状态，未输出凭据或成员身份列表。
