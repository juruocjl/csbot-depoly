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
