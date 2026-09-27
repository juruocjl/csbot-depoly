# 本地交付验收

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
