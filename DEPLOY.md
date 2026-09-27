# 部署手册

本手册整合原 `csbot/scripts/DEPLOY.md`、后端 README 的部署说明与 Steam Monitor README 的 Docker 部署说明。服务器现状核查日期：2026-09-27（Asia/Shanghai）。

部署前阅读 [执行约束](AGENTS.md)。目录、启动恢复和迁移清单见 [SERVER_SERVICES.md](SERVER_SERVICES.md)，本地验收见 [DELIVERY_CHECK.md](DELIVERY_CHECK.md)。本文中的发布命令供实际发布时使用，本次文档整理未执行发布。

## 部署结构

| 组件 | 发布来源 | 生产位置与运行方式 |
| --- | --- | --- |
| 后端 | `csbot` 的 `main` | `/home/ubuntu/csbot`；`csbot.service` |
| 前端 | `csbot-front/main` 经 GitHub Actions 生成 `build-output` | `/home/ubuntu/csbot/dist` 是独立 Git 仓库；由 `csbot-nginx` 提供静态文件 |
| Steam Monitor | `steam_monitor_js/main` | `/home/ubuntu/steam_monitor_js`；Docker Compose 构建镜像 |
| PostgreSQL / Nginx / NapCat / Adminer | `csbot/docker-compose.yml` | `/home/ubuntu/csbot` 的 Compose 项目 |
| Mihomo / TeamSpeak | 服务器既有配置 | 见服务清单；TeamSpeak 未包含在三个子模块中 |

服务器当前没有改为部署本父仓库。父仓库子模块固定源码版本，现行脚本仍拉取子仓库分支；因此不能把父仓库 commit 当成已经发布的版本凭据。

## 修改完成后的同步与部署顺序

代码、配置和文档修改通过必要检查后，及时提交并同步 GitHub。先 commit/push 有改动的子仓库，再 commit/push 父仓库的文档和子模块指针；核对远端 commit，不能只留下本地修改。用户明确要求暂不推送时按其要求处理，推送受阻时说明原因。

生产发布统一走：**本地修改 → commit → push GitHub → 服务器 `git pull --ff-only` → 按需构建/重启 → 验收**。前端在 push 与服务器 pull 之间还须等待对应 GitHub Actions 构建完成。代码不通过 scp/rsync 覆盖或直接在服务器修改后发布；环境密钥和持久化数据使用各自的运维流程。

同步 GitHub 不自动触发本手册中的服务器发布操作；服务器更新须属于用户当前授权的部署范围。

## SSH 与发布前检查

默认沿用本机 SSH 配置：

```bash
ssh -o BatchMode=yes ubuntu@cgserver 'id -un && hostname'
```

从部署仓库根目录检查本地子模块：

```bash
git submodule status
git -C csbot status --short
git -C csbot-front status --short
git -C steam_monitor_js status --short
```

按本次变更范围检查对应子仓库，确认提交已推送。Windows 旧脚本会检查后端已跟踪改动、fetch 目标分支并拒绝落后或分叉的版本；手工发布也应遵守这一要求。

在服务器记录本次发布前的 commit，并检查已跟踪文件：

```bash
ssh ubuntu@cgserver
git -C /home/ubuntu/csbot status --short --untracked-files=no
git -C /home/ubuntu/csbot rev-parse HEAD
git -C /home/ubuntu/csbot/dist status --short --untracked-files=no
git -C /home/ubuntu/csbot/dist rev-parse HEAD
git -C /home/ubuntu/steam_monitor_js status --short --untracked-files=no
git -C /home/ubuntu/steam_monitor_js rev-parse HEAD
systemctl is-active mihomo docker csbot
```

服务器存在未跟踪的数据和辅助脚本，不能为“清理工作区”删除它们。发布前备份本次可能变更的配置和数据；数据库结构变更还须有单独的迁移与恢复方案。

## 后端发布

以下在服务器执行，前提是本次后端提交已推送到 `main`：

```bash
set -e
cd /home/ubuntu/csbot
git pull --ff-only origin main
git rev-parse HEAD
sudo -n systemctl restart csbot.service
systemctl is-active csbot.service
sudo -n systemctl --no-pager -l status csbot.service
```

后端 unit 设置 `ENVIRONMENT=prod`，工作目录 `/home/ubuntu/csbot`，入口为 `/home/ubuntu/.local/bin/uv run python bot.py`。修改依赖或数据库结构时，须结合 `uv.lock` 和已审查迁移安排发布；现有发布脚本不执行显式数据库迁移。不要在生产运行自动生成迁移的 `csbot/scripts/upgrade.sh`。

Git 拉取失败先检查 Mihomo。旧脚本有一次禁用 Git HTTP 代理的重试；手工操作也可在确认直连可用后使用 `git -c http.proxy= -c https.proxy= pull --ff-only origin main`，不要修改全局代理作为临时排障手段。

## 前端发布

1. 在 `csbot-front` 提交并推送 `main`。
2. 等待该 commit 对应的 GitHub Actions `Node.js CI` 完成。工作流使用 Node.js 22，执行 `npm ci`、`npm run build`，将 `dist` 推送到 `build-output`。
3. 在服务器更新产物仓库：

```bash
set -e
cd /home/ubuntu/csbot/dist
git fetch origin build-output
git pull --ff-only origin build-output
git rev-parse HEAD
curl -fsS --max-time 10 http://127.0.0.1:1234/ai-chat -o /dev/null
```

静态文件通过 bind mount 直接供 Nginx 读取，无单独前端进程。只更新静态文件通常无需重启后端；涉及后端契约的变更应协调两者发布。修改 Nginx 模板或 Compose 配置时需另行验证配置并应用到容器，单纯拉取文件不能保证运行中配置已经更新。

## Windows 既有发布脚本

脚本仍位于 [csbot/scripts/deploy_backend.ps1](csbot/scripts/deploy_backend.ps1)。从部署仓库根目录运行：

```powershell
powershell -ExecutionPolicy Bypass -File ./csbot/scripts/deploy_backend.ps1 -Remote ubuntu@cgserver
```

脚本默认推送后端分支、拉取线上后端和前端产物、重启后端并查看日志。前端有变更时，须先完成前端构建等待步骤。它沿用旧 Windows 默认值：目标 IP `42.193.244.178` 和本机 `~/.ssh/id_rsa`；上例显式覆盖 SSH 目标，密钥不同可用 `-IdentityFile`、`-SshConfig`、`-KnownHosts` 指定。

常用开关（加在上述命令后）：

| 参数 | 用途 |
| --- | --- |
| `-SkipFrontend` | 仅发布后端 |
| `-SkipPush -SkipPull -SkipFrontend` | 仅重启并检查后端 |
| `-SkipPush -SkipPull -SkipFrontend -SkipRestart` | 只检查本地状态并 fetch 远端引用；不会连接服务器，因此不能作为 SSH 配置或连接验证 |

## 发布验收与回退

在服务器执行：

```bash
systemctl is-active mihomo.service docker.service csbot.service
sudo -n docker compose -f /home/ubuntu/csbot/docker-compose.yml ps
sudo -n docker compose -f /home/ubuntu/steam_monitor_js/compose.yaml ps
curl -fsS --max-time 10 http://127.0.0.1:1234/major-homework -o /dev/null
curl -fsS --max-time 5 http://127.0.0.1:5555/api/health
sudo -n journalctl -u csbot.service -n 80 --no-pager
```

日志在受信任终端查看，对外分享前脱敏。核对实际 commit 与发布目标一致，并验证本次涉及的受保护 API；测试方法见 [DELIVERY_CHECK.md](DELIVERY_CHECK.md)。Steam 健康需要 HTTP 200、`loggedOn=true`、`friendStatusReady=true`。

回退前使用发布前记录的版本评估应用、静态产物与数据库兼容性。源码优先以 revert 提交恢复并按正常流程发布；前端需重新生成对应构建产物；Steam Monitor 源码回退后需重新构建容器。数据库回退单独处理，不能用旧代码或删除数据卷代替数据恢复。

## 生产配置与响应白名单

以下保留原后端部署文档中的白名单维护步骤，仅在明确需要同步两个列表时执行；示例群号须替换为实际值。


The server reads production settings from `/home/ubuntu/csbot/.env.prod`.
Keep these two lists intentionally separate:

```env
# Groups that receive scheduled pushes and reports.
CS_GROUP_LIST = ["<group-id>"]

# Groups whose events are allowed to reach matchers.
CS_EVENT_GROUP_LIST = ["<group-id>"]
```

`CS_EVENT_GROUP_LIST` is the response whitelist. Events outside this list are
ignored before matchers run. If this list is empty or missing, all incoming
events are blocked.

When the response whitelist should mirror the current push list, run this on
the server before restarting:

```bash
cd /home/ubuntu/csbot
cp .env.prod ".env.prod.bak-event-whitelist-$(date +%Y%m%d%H%M%S)"
python3 - <<'PY'
from pathlib import Path

p = Path(".env.prod")
lines = p.read_text().splitlines()
group_line = next(
    (line for line in reversed(lines) if line.strip().startswith("CS_GROUP_LIST") and "=" in line),
    None,
)
if group_line is None:
    raise SystemExit("CS_GROUP_LIST not found")

event_line = "CS_EVENT_GROUP_LIST=" + group_line.split("=", 1)[1]
for i, line in enumerate(lines):
    if line.strip().startswith("CS_EVENT_GROUP_LIST") and "=" in line:
        lines[i] = event_line
        break
else:
    if lines and lines[-1].strip():
        lines.append("")
    lines.append(event_line)

p.write_text("\n".join(lines) + "\n")
PY
grep -nE '^[[:space:]]*CS_(EVENT_)?GROUP_LIST[[:space:]]*=' .env.prod
```


## Steam Monitor 部署


现有服务器使用 Docker Compose 运行。先提交并推送本次 Steam Monitor 变更，确认代理可用，再在服务器执行：

```bash
set -e
cd /home/ubuntu/steam_monitor_js
git pull --ff-only origin main
mkdir -p data
sudo -n docker compose up -d --build
sudo -n docker compose ps
curl -fsS --max-time 5 http://127.0.0.1:5555/api/health
```

Compose 使用 host 网络访问宿主机上仅监听回环地址的 Clash SOCKS 端口，并将 API 仅监听在宿主机本地：

- `http://127.0.0.1:5555`

以下非 Mihomo 场景属于迁移说明，不应用于当前服务器的日常发布。默认代理地址为 `socks5://127.0.0.1:7891`。如果服务器没有本机 Clash/Mihomo，删除 `compose.yaml` 中的 `STEAM_SOCKS_PROXY`、`STEAM_WEB_COMPATIBILITY_MODE` 和 `network_mode`，恢复端口映射部署。

连接类错误（如 `NoConnection`、`ServiceUnavailable`、连接超时）发生时，服务会通过 Mihomo 控制接口测试候选节点，切换成功后再尝试登录。短重试达到上限后不会退出或永久熔断，而是按 1、2、4、8、16、30、30... 分钟自动恢复；每轮先探测 Steam CM，探测失败时不会发送登录请求，探测成功才尝试一次登录。`RateLimitExceeded` 使用更保守的 1、2、4、6、6... 小时冷却后自动恢复。只有认证拒绝和 Steam Guard 会硬熔断。

容器会挂载宿主机的 `.env` 和 `data/`。程序登录成功后更新的 `STEAM_REFRESH_TOKEN` 会立即用于后续重连并写回宿主机 `.env`；容器进程重启时也会直接读取该文件中的最新 token。SQLite 历史会保存在宿主机 `data/friend_game_history.db`。


自动恢复和健康响应字段的完整说明见 [Steam Monitor README](steam_monitor_js/README.md#4-api-说明)。容器初次启动可能仍处于登录阶段，等待健康检查后再判断发布结果。
