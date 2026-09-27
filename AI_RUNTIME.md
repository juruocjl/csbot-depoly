# AI 运行时改造与发布

本文件维护架构、权限、数据迁移和 AI 专项发布步骤。通用发布约束仍以 [AGENTS.md](AGENTS.md)、[DEPLOY.md](DEPLOY.md)、[SERVER_SERVICES.md](SERVER_SERVICES.md) 为准。代理实际可读的数据字典在 [DATA.md](csbot/ai_runtime/DATA.md)。

## 已确定的行为

| 场景 | 会话与行为 |
| --- | --- |
| QQ 群普通聊天 | 归档、建立检索索引；不开口、不启动模型、不提炼 Mneme 记忆。 |
| `/ai`、明确 @ 机器人、回复机器人 | 进入该群唯一 DSH 会话，保留真实 QQ 发言者；只对这次调用回复。@全体成员与机器人自发消息不触发 AI。 |
| 同群多用户 | 使用同一会话与群记忆；服务端按群串行处理。角色命令只改变本轮语气。 |
| 网页 | 按群权限、用户和个人 conversation 三者隔离；可查该用户有权访问的群数据，聊天和 Mneme 不写入群会话。 |
| 日报、周报 | 每次独立报告会话，不向群长期记忆沉淀自动报告；报告实际发送后仍进入群消息索引。 |
| 旧 AI 回复记录 | 无可靠归属的返回 404；管理员确认并导入映射后开放，不推测历史权限。 |

DSH 负责会话存储、恢复、工具循环和压缩。业务层只维护消息摄取游标、授权、发送归档与任务状态，没有另一套自写的聊天轮次截断器。每次被调用时启动 DSH，结束后退出，避免为每个群常驻一个 Node 进程。

### 上下文摄取

- 上次已处理游标之后至本轮截点，不超过 40 条且 JSON 不超过 12,000 字节：按顺序提供原消息。
- 超过阈值：提供约 10,000 字节近期消息及最近 1,200 条消息的块目录，每块 50 条、最多 24 块。单条长消息给预览并标记截断。
- 更早资料仍可按关键词、时间、用户、原始 record_id 检索；目录明确表示是否省略更老的块。块 ID 绑定原始消息编号，重建检索 span 不使引用变指向。
- 普通聊天以插件资料注入，真正叫到机器人的消息以用户消息进入 DSH。只有成功完成的 QQ 轮次推进摄取游标。中断不会自动重发请求。
- DSH 采用已声明的模型上下文容量（默认 65,536）触发自身压缩；实际压缩、退出、恢复已做集成验收。

### Mneme 和回复语气

固定 `@deepseek-ai/dsh=0.1.5-rc.3`、`@modusensus/dsh-mneme=0.8.8`，npm lock 随代码提交。使用 lightMode、关闭自动 dream、实体抽取和外部 API。每个群或个人会话有独立物理 memory 目录，避免 Mneme 按工作目录识别作用域时串群。

Mneme 从被叫到后的公开对话提炼，不把插件注入的旁听原文加入提炼输入。公开工具结果和回答可以作为这次对话的证据；供应商返回的思考仅在下文规定的会话权限范围内展示，不发送到 QQ 正文。QQ 输出检查在 DSH 落盘和 Mneme 提炼之前执行。`mneme.mjs` 等待上游异步 turn/end 写入完成，解决 DSH 不等待观察者导致进程提前退出的问题；回归测试刻意给提炼加延迟。

默认身份是直爽犀利、有自己判断、爱接梗和吐槽的群友，也愿意帮忙查数据。欣赏配合和坦诚，反感甩锅与硬装；闲聊、抱怨先接话，不自动变成分析任务。幽默跟随话题，不硬塞梗、追着羞辱人或虚构亲身经历。需要查证时可以自然说“我查一下”，不固定输出计划/步骤/总结或客服结束语。原有 `/aitb`、`/aixmm`、`/aixhs`、`/aitmr` 仍指定本轮风格，下一轮恢复默认个性。

mid、record_id、block_id 等仍保留在内部资料中用于准确查询、关联与追溯，日常回复用昵称、地图、比分、原话片段指代。时间按服务端当前时间及 Asia/Shanghai 转成“昨晚”、月日/时段等必要精度；不照搬秒、毫秒和原始时间戳。明确索要原始编号/精确时间或排障、区分记录需要时仍能提供。数值统计和内部计算保留真实精度。通过模型提示及 DATA.md 约束表达，不对最终文本粗暴删除数字，因此不承诺每次生成都绝无偏差，应继续按实际对话样例调整。

旧 `ai_mem` 现作为只读历史归档保留。自动首次调用迁移已退役，改用下文离线迁移/核对脚本，经备份、正文比对和 Mneme 原生检索后保存回执。仅处理有明确群号的手工记忆，排除旧问答/日报摘要，不把个人会话并入群。

## 查询和脚本边界

模型只看到 `read`、`read_image`、`execute_python` 和 Mneme 的保存/检索/忘记工具。文件访问只允许本会话工作目录和服务端确认属于当前群的图片路径，按 realpath 校验；不开放宿主 shell、写文件工具、子进程工具或任意插件安装。

脚本通过 `csdata.call()` 调用预写 SQL，也可以 `csdata.query()` 自写查询；两者走同一个授权编译器。`catalog()` 和工作目录的 DATA.md 给出表、字段、参数、查询样例及统计语义。`search/block/image/status` 也是脚本 SDK 的方法，不额外扩展模型工具列表。

| 边界 | 实现 |
| --- | --- |
| 主库 | PostgreSQL 只读事务，逻辑表编译为带服务端群约束的子查询；主表显式 public，函数解析限制 pg_catalog。不暴露 auth、完整配置、凭据或原始消息二进制。 |
| 游戏库 | SQLite mode=ro、query_only、authorizer，只开放状态历史和名称表；先由主库取得本群 SteamID 再施加过滤。 |
| SQL | 单条 SELECT；禁写入、系统表、物理 schema、递归 CTE、不在白名单的函数/类型。可用聚合、JOIN、窗口、非递归 CTE。 |
| 返回 | 500 行、256 KiB；明确 truncated；数据库超时 8 秒，全局并发 2。必须聚合或分页，不能拿截断结果假装总体统计。 |
| Docker | 无网络、只读根、非 root、cap-drop ALL、no-new-privileges；192 MiB、0.5 CPU、32 PID、45 秒。只挂当轮 Unix 查询 socket 和可信中文字体文件，不挂数据库、仓库目录、Docker socket或模型密钥。 |
| 脚本 | 64 KiB 输入、最多 32 次数据调用、4 MiB 输出；工作目录 32 MiB、临时目录 16 MiB。numpy/matplotlib/Pillow 固定版本。不可用时明确失败，不降级为宿主执行。 |
| 结果文件 | 仅少量 PNG/JPEG/TXT/CSV/JSON，单个 2 MiB，路径由宿主随机化；供本轮只读复核，下轮清理临时文件。 |

SQL 事务使用可信后端现有数据库连接；连接和授权逻辑永远不交给模型/容器。即使是预写模板也不能指定另一个群。游戏切换记录不等于在线心跳：`csdata.status()` 只返回本机 Monitor 的就绪布尔值；查询时间不是更新时间。

## 发送归档和图片

发送通过 OneBot `Bot.call_api` 包装统一处理，包括文本、图片、回复、合并转发、进度提示和报告。先写本地持久 outbox，再发送。拿到 QQ mid 后写 GroupMsg 和聊天索引；索引失败后台每 30 秒重试，**不再次发送 QQ 消息**。平台回流与包装器用群+mid 和事务锁去重。合并转发的展示昵称明确标为非认证身份。历史上未归档的发送不会凭空补齐。

发送超时/连接中断可能已送达，标记 unknown；重启遗留 pending 也变为 unknown，不盲目重发。已确认失败则清空暂存内容，成功索引后也清空。暂存载荷总上限 64 MiB，满时本条发送前明确失败，需管理员核对 unknown。临时发送载荷与原图缓存分别计量；SQLite 可复用空页，文件物理大小不一定随删除缩小。

```bash
uv run --frozen python scripts/ai_outbox.py --state data/ai/state.sqlite3
# 人工核实已经发送及对应 mid 后：
uv run --frozen python scripts/ai_outbox.py --state data/ai/state.sqlite3 --id <outbound-uuid> --confirmed-mid <qq-mid>
# 仅在人工确认没有送达时：
uv run --frozen python scripts/ai_outbox.py --state data/ai/state.sqlite3 --id <outbound-uuid> --confirmed-not-sent
```

原图 `imgs/history/full` 预算固定默认 **1 GiB（1,073,741,824 字节）**，超过后按 LRU 淘汰至 90%，正在读取的图片有租约。`imgs/history/small` 缩略图不清理；保留原图元数据和淘汰状态。历史 `.png` 文件名可能包含 JPEG/GIF 字节，不能只看扩展名判断编码。单张接收上限 32 MiB、64M 像素。

启动时接管原有 hash 文件，按 mtime 初始化旧数据 LRU。旧 `imgs/cache` 只迁移下载器登记过的文件，确认缩略图/元数据已保存才移除重复缓存；未知文件、符号链接或超限文件保留并告警。不会任意删除目录。

模型通过图片 ID 获取 `available/evicted/missing` 与只读路径；实际可读性是最终依据，不自动重下淘汰大图。DSH 保存的是模型预览（至多 1M 像素、长边 2048、编码目标 256 KiB），属于小图，会话恢复可继续看预览，不能宣称原图仍在。原图 1 GiB 不是包含缩略图、日志、会话和数据库在内的整个 data 目录预算；这些持久数据须纳入磁盘监控与备份。

## 发布前准备

本改造的生产切换应另按用户授权执行。独立验收目录 `/home/ubuntu/csbot-ai-validation` 不等于生产 `/home/ubuntu/csbot`。内存保护服务由另一个任务管理，本任务不替换其配置。

1. 按通用发布流程核对并推送后端/前端目标 commit，备份数据库、`.env*`、`data/ai` 和图片。运行时数据不提交 Git。
2. 生产仓库 `git pull --ff-only`。Python 最低 3.11，生产已有 3.12；执行 `uv sync --frozen`。本改造不要求新增 PostgreSQL 表或生成迁移，依赖既有聊天索引、token 字段、runtime_config 和 ai_mem；缺失时先按对应既有迁移文档处理。
3. `bash scripts/install_ai_runtime.sh`：安装 Node **22.22.0 Linux x64** 至忽略的 `.tools/node`，下载官方 tar 并核对内置 SHA256，然后 npm ci 固定 DSH/Mneme。不会改系统 Node。非 Linux x64 应使用适配该平台且通过同一测试的 Node >=22.12。
4. 构建脚本镜像（基础镜像摘要、Python 依赖均固定）：

```bash
sudo -n docker build --memory=384m --cpu-quota=50000 \
  --build-arg PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
  -t csbot-ai-python:1 -f ai_runtime/sandbox.Dockerfile ai_runtime
```

PIP_INDEX_URL 可省略，默认官方源；验收机官方包下载很慢，清华源已成功。基础镜像使用 `registry-1.docker.io` 和摘要，避免服务器既有失效镜像站。构建只包含 Dockerfile 和 csdata.py，不把配置/源码/密钥送入构建上下文。

### 环境与 systemd

| 配置 | 值/含义 |
| --- | --- |
| CS_AI_ENGINE | 仅 dsh；旧引擎及旧记忆读写已移除，legacy 不再可用。 |
| CS_AI_URL / CS_AI_API_KEY | 沿用既有模型接口；从安全配置读取，禁止写入 Git/文档。 |
| cs_ai_model | 仍从数据库动态配置读取，不用旧环境值覆盖已保存选择。 |
| CS_AI_VISION | 默认 true；模型不支持图片时显式 false，不能假装看到了图片。 |
| CS_AI_CONTEXT_WINDOW | 默认 65536，不应大于服务商实际容量。 |
| CS_AI_STATE_DIR | `/home/ubuntu/csbot/data/ai`，目录 0700、状态库 0600。 |
| CS_AI_GAME_DB | `/home/ubuntu/steam_monitor_js/data/friend_game_history.db`，后端须能只读打开其 WAL 数据。 |
| CS_IMAGE_FULL_BUDGET_BYTES | 1073741824。缩略图无清理配置。 |

模板 [ai.conf](csbot/deploy/systemd/csbot.service.d/ai.conf) 设置专用 Node PATH、数据路径和仅后端服务的 SupplementaryGroups=docker。可信后端有 Docker 控制权；这不是把控制权赋予 agent。保留既有 csbot.service 入口及 memory.conf，不覆盖整个 unit。安装模板、daemon-reload 和 restart 属于实际发布操作，须在发布授权后进行。

模板在启动前检查 Node、固定依赖、真实容器执行、两库 schema 与目录权限；Monitor 临时不就绪只警告，避免其登录退避连带阻断整个 bot。正式放行应在服务身份下额外运行严格检查（临时 systemd unit 可指定 SupplementaryGroups=docker 和同样 PATH）：

```bash
uv run --frozen python scripts/ai_preflight.py --env .env --env .env.prod
```

不要用 root 的检查结果替代 ubuntu 服务身份权限检查。若环境文件由 systemd 直接提供，可省略对应 `--env`。前置检查只读数据库，不调用模型；真实模型与受保护 API 是后续验收。

后端只允许一个进程持有同一 state.sqlite3，文件锁拒绝第二实例。DSH 全局并发 1，队列最多 8；容器受独立 Docker cgroup 限制，不属于后端 MemoryMax。容器挂 `csbot.ai.sandbox=true` 标签，服务停止后/再次启动前清理遗留容器，覆盖后端被强制杀死时来不及 finally 的情况；不得在运行中手工执行 cleanup。

### 旧记录授权映射

管理员逐条确认后准备私有 JSON 文件，数组元素须包含 `chat_id`（原 UUID）、`group_id`、`user_id`（字符串）和 `channel`（qq/web/report）。qq/report 对同群开放，web 仅对应用户可读。不要依据问答内容猜身份。

```bash
uv run --frozen python scripts/ai_import_ownership.py --state data/ai/state.sqlite3 --mapping /安全路径/ownership.json
# 检查 dry run 后才显式应用：
uv run --frozen python scripts/ai_import_ownership.py --state data/ai/state.sqlite3 --mapping /安全路径/ownership.json --apply
```

映射不复制问答、不写群会话或 Mneme；现有归属冲突会拒绝整个操作。错误映射须经管理员核对后恢复状态库备份或修正，不能靠重新导入悄悄覆盖。

## 验收、备份与回退

验收命令和已验证范围见 [DELIVERY_CHECK.md](DELIVERY_CHECK.md#ai-改造专项)。QQ 真人发送验收必须用受控测试群：普通发言沉默、@/reply//ai 正常响应、机器人答案可检索、群 A 无法读群 B、原图淘汰后缩略图仍能读取。现有测试使用模拟 OneBot 发送，**未向真实 QQ 群发送测试消息**。

备份时停止后端后复制整个 `data/ai`（含 DSH sessions、Mneme SQLite/WAL、state.sqlite3）和图片目录；不要只复制打开状态的 SQLite 主文件。PostgreSQL 继续用现有备份流程。旧记录访问权依赖 state.sqlite3，丢失它应拒绝访问，不能自动放宽。

回退前记录后端/前端源码和 build-output commit，保留新数据。移除 AI 专项 drop-in 后可以回滚已知可用后端 commit；不能用 CS_AI_ENGINE=legacy 回退；旧引擎已移除，必须走已审查的代码 revert 发布。图片已按获准预算淘汰的大图无法通过代码回滚恢复，只有备份能还原；小图不删除。旧 ai_mem 与 PostgreSQL 问答没有被迁移程序删除。

## 实时过程、请求排队与生成图片（2026-09-27）

运行入口仍是 CSBot → DSH 原生 agent → Mneme；网页通过现有后端读取同一次执行的事件。没有另一套网页 agent，也没有 DSH Web 快照观察器。之前提交的 observer 实验已撤回，不需要新监听端口、转发服务或修改 DSH 依赖。

### 请求与补充

- 同一会话按接收顺序串行；跨会话共用一个执行槽，最多 8 个已排队/执行中的请求。等待执行槽时状态保持 `queued`，拿到槽才变成 `running`。网页正常“发送”始终创建排队请求。
- 网页“补充当前问题”只可补充本人当前个人会话。QQ 被明确叫到且正文以 `补充：` 或 `补充:` 开头时，尝试补充同群中该发言者正在执行的问题，不能插入其他人的问题。普通聊天不触发，不更改已有输入图片查看链路。
- 补充调用 DSH 原生 `agent.steer`：在下一处理步骤纳入，不能改写模型已经生成的 token。DSH 接收后可能继续一个步骤/回合；最终回复保留原问题及补充产生的回答，不保证总是合成一句。
- 当前轮未开始、已经结束或明确拒收时，补充作为新请求排队。确认超时/进程断开造成结果不确定时，记录 `unknown` 并告诉用户查看当前回复，**不自动重发**。重启后未完成请求标为 interrupted，不自动重跑或向 QQ 重发。

### 实时展示及权限

- `POST /api/ai/events`：已有 Bearer 认证，JSON `{chatId, after}`，SSE 单调事件序号；80ms 批次转发原生 `agent/assistant-stream` 和 `session/event` 的工具事件。`ready` 带提问、渠道及图片元数据；`done` 带最终状态/回答；空闲每 10 秒心跳。断线带游标重连，不重新提交问题；游标失效返回 409，前端重新读取本轮。
- 工具调用开始、结束配成一张默认折叠卡片；脚本直接显示代码，参数/结果可展开。思考显示供应商实际返回的 reasoning 内容，不生成模拟思考。`CS_AI_ENABLE_THINKING` 默认 true，仍尊重显式配置和模型能力。QQ 最终文本仍先通过已有审查再入 DSH/Mneme；等待审查期间可实时显示真实思考。
- QQ 记录由**同群已认证成员**查看，包括思考和脚本；个人网页记录仅本人且认证群一致时可看。跨群、跨用户、未知归属旧记录返回 404。网页会话、记忆仍独立于群。
- 事件是 UI 投影，不替代 DSH 会话/压缩/记忆。每轮详细 trace 最多 8 MiB；状态、最终回复、补充、图片索引保留。事件 payload 总量超过 256 MiB 后，在下次请求登记时删除最早已结束轮次的 trace，降至 192 MiB；该数值不包含 SQLite 索引及可复用空页。旧记录无思考可回填，界面明确说明；最终回复、图片索引和 DSH 原始持久会话不随 UI trace 清理。
- Nginx `/api/` 禁用响应缓冲、读取超时 660 秒；继续使用已有 HTTP 入口，无新公网服务。

### AI 输出图片

隔离脚本使用以下方式明确提交生成图，临时产物不会自动发送：

```python
import csdata
csdata.configure_plot()  # Agg 后端及只读挂载的中文字体
import matplotlib.pyplot as plt
plt.plot([1, 2, 3], [2, 4, 3])
plt.title("趋势")
plt.savefig("chart.png")
plt.close()
csdata.artifact("chart.png", send=True, caption="趋势图")
```

- 仅成功退出的脚本接收提交；PNG/JPEG，单图 ≤2 MiB、≤1600 万像素，每轮最多 4 张。容器隔离限制不变；只增加仓库可信字体文件的只读挂载，不挂宿主数据目录或 Docker socket。
- 生成图放 `CS_AI_STATE_DIR/media`，不放公开静态目录。网页通过 `GET /api/ai/images/{id}` 按回复归属鉴权，返回原图；原图淘汰返回 410，可用 `?thumbnail=true` 查看永久缩略图。浏览器用认证请求取得 Blob，图片不是公开链接。
- 历史群原图和生成原图**共用 1 GiB LRU 预算**，缩略图暂不清理。页面会说明原图已淘汰。QQ 发送前持有原图租约并复制成 OneBot 图片段，淘汰时不拿缩略图冒充原图；实际发送仍走原有 outgoing 归档、索引与不确定结果处理。
- 生成图已经在网页可见、随后模型失败时，保留已生成的图片及 interrupted 状态，不伪装为整轮成功。

### 发布与回退补充

本次仅自动增加 AI 运维 SQLite 的 `run_events`、`run_images` 表及媒体索引 `private` 列，不生成或执行 PostgreSQL 业务迁移。发布前用 SQLite backup API 保存两个索引；保留既有 DSH/Mneme 状态。必须重新构建 `ai_runtime/sandbox.Dockerfile`，才能让容器中的 csdata SDK 支持 `send=True` 与中文字体配置。

按 DEPLOY.md 先推送代码，经服务器独立验收目录验证，再更新生产、重启后端、加载 Nginx 配置。验证 Nginx 实际配置含 `proxy_buffering off`；若单文件 bind mount 仍指向旧 inode，重建 Nginx 容器加载新文件。旧 observer 单元只在确认 disabled/inactive 后移入本次私有备份，不删除历史快照数据。回退用代码 revert 正常发布并恢复旧镜像标签；新增 SQLite 数据可保留，旧代码不使用这些表。


## 旧记忆接口退役与离线迁移

- 删除 `/ai记忆` 专用命令及帮助条目。现在直接在 `/ai` 或 @/回复机器人的正常对话中说“记住…”或“忘记…”，由 Mneme 工具处理。
- 删除旧 DataManager 的手工记忆、问答摘要和日报摘要读写接口；定时日报/周报照常生成、发送、归档，但不再向旧表写记忆。移除依赖这些接口的旧 AI 引擎回退实现，DSH 为唯一执行入口。
- 群请求不再查询 ai_mem 或触发隐式迁移。AIMemory ORM 定义和原 PostgreSQL 表保留，用于备份、核对及明确的离线迁移；没有删表或删旧内容。

`scripts/ai_import_legacy_memory.py` 从 stdin 接收 `[{"gid":"群号","mem":"原文"}]`。默认 dry-run，输出大小、源 SHA256、目标 scope 和 pending/already_present/conflict 状态，不输出正文。只接受数字群号，`:qa`/`:report` 明确排除，未知键直接拒绝；手工记忆中 `[自动日报周报知识]` 之后内容也排除。

```bash
# 在后端仓库、已激活 .venv 且 Node 22 PATH 就绪时执行；source.json 是服务器私有导出。
python scripts/ai_import_legacy_memory.py --state-dir data/ai < /私有目录/source.json
# 完成代码发布并停后端及内存守护后执行。backup-dir 必须是不存在且位于 state-dir 外的新目录：
python scripts/ai_import_legacy_memory.py --state-dir data/ai --apply \
  --backup-dir /home/ubuntu/backups/legacy-memory-本次日期 < /私有目录/source.json
```

apply 获取与后端相同的独占状态锁，后端运行时拒绝操作；先创建 0700 备份目录，保存源数据（0600）、SQLite backup API 一致性副本和相关群完整 DSH/Mneme 目录，再操作。用固定版本 DSH 的 `memory_save`/`memory_search` 执行导入和检索；不向模型发送数据、不产生用户聊天轮次、不运行脚本容器、不提炼记忆。原文逐字保留，每段最多 4000 字符，上限 64000 字符，超限须另行审查拆分。

已有对应条目时只核对原文和原生检索，不重复写入。发现已迁移内容被修改、遗忘、归档或记录冲突时拒绝覆盖/复活；不得为了通过验收把群友已经删除的记忆重新灌回。校验后在运维 SQLite 的 `memory_import_receipts` 保存源哈希及时间，在备份目录保存 receipt.json。重新执行必须使用新的备份目录，但不增加相同记忆。

回退先停服务与内存守护，用本次备份恢复运维 SQLite 和对应 scope；其余个人或群 scope 不动，PostgreSQL 原表始终保留。不要在线复制正在写入的 SQLite 文件回滚。旧接口的代码回退仍按 Git revert→push→pull→重启流程执行。

## 群资料与群规则查询（2026-09-27）

在原有 `csdata.call()` 中增加 `group_members/member_info/member_avatar/group_admins/election_rules/points_today`，模型工具数量不变；SQL新增 `group_people/points/election_state` 逻辑表及相应模板。调用参数、示例与口径随每轮 DATA.md 提供；查询当前成员不要求绑定Steam。

- OneBot 每轮按授权群拉取一次最新成员快照，明确区分QQ昵称、群名片、实际角色。管理员查询同时给出QQ实际群主/管理员与机器人竞选状态；不同来源不能混为一谈，QQ不可用时不以旧库记录伪造实时结果。
- 头像只接受本群QQ号；后端从固定QQ CDN地址取图，不接受任意URL/平台API。禁止跳转，下载≤2MiB、解码≤400万像素，规范化最长640像素PNG以供DSH直接读取；进入私有图片缓存，共享原图1GiB LRU与永久缩略图。返回路径只对当前轮的read_image开放，持有读取租约，不自动发送头像，不给沙箱开放网络。
- 复读点数按原功能23:55业务日计算，普通点/奖励点/惩罚触发数分别返回。自写SQL中的points按完整group_群号_QQ键限定，election_state只开放本群3个白名单键，不开放任意local_storage或完整user_info。
- 规则来自当前部署代码，返回本群竞选/禁言启用状态。明确按权重随机选拔、资格与转让排除、时间窗口、点数榜不等于竞选概率；惩罚触发记录不保证实际禁言。旧帮助的“后续复读固定3”与执行逻辑不一致，以及昵称加20位于finish之后不可达，知识说明按实际行为写，未擅自修改计分/选拔逻辑。
- 当前管理员、昵称和点数须重新查询，不灌入长期记忆冒充恒定事实。网页版仍只能查询认证群资料，个人记忆不写群会话。新功能没有任免、禁言、改名或加点权限。

不需要新增数据库结构、依赖或重建沙箱镜像；按常规后端Git发布。回归：`python scripts/check_group_knowledge.py` 与 `python scripts/check_ai_boundaries.py`。真实数据库测试仍仅用csbot_backup；线上通过既有认证API验收，不发QQ测试消息。
