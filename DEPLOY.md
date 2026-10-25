# 部署手册

本手册整合原 `csbot/scripts/DEPLOY.md`、后端 README 的部署说明与 Steam Monitor README 的 Docker 部署说明。服务器现状核查日期：2026-09-27（Asia/Shanghai）。

部署前阅读 [执行约束](AGENTS.md)。目录、启动恢复和迁移清单见 [SERVER_SERVICES.md](SERVER_SERVICES.md)，本地验收见 [DELIVERY_CHECK.md](DELIVERY_CHECK.md)。本文中的发布命令供实际发布时使用，本次文档整理未执行发布。

## 部署结构

| 组件 | 发布来源 | 生产位置与运行方式 |
| --- | --- | --- |
| 后端 | `csbot` 的 `main` | `/home/ubuntu/csbot`；`csbot.service` |
| 前端 | `csbot-front/main` 经 GitHub Actions 生成 `build-output` | `/home/ubuntu/csbot/dist` 是独立 Git 仓库；`https://cs.ave-mujica.top/`，原 `csbot-nginx` 1234 入口保留 |
| Steam Monitor | `steam_monitor_js/main` | `/home/ubuntu/steam_monitor_js`；Docker Compose 构建镜像 |
| PostgreSQL / Nginx / NapCat / Adminer | `csbot/docker-compose.yml` | `/home/ubuntu/csbot` 的 Compose 项目 |
| HTTPS 入口 | `csbot/docker-compose.yml`、`csbot/assets/homepage.conf` | `csbot-homepage`，80 跳转 HTTPS，443 按域名提供个人主页或 dashboard |
| 主域与通配符证书 | `csbot/deploy/systemd/derper-acme-*`、`csbot/scripts/ave_mujica_cert_deploy.py` | 既有 lego + 阿里云 DNS-01；个人主页与 DERP 共用证书 |
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

服务器存在未跟踪的数据和辅助脚本，不能为“清理工作区”删除它们。发布前备份本次可能变更的配置和数据；数据库结构变更还须有单独的迁移与恢复方案。 后端 AI preflight 要求至少 800 MiB 可用磁盘；发布前必须把本次备份增量计入，确认**备份完成后**仍满足门槛。空间紧张时先准备压缩备份并逐文件校验其可恢复性，不要等停服后才触发磁盘门槛，也不要降低门槛或擅自删除历史数据。

## 安全更新与管理端口

管理 SSH 使用 Tailscale 的 `ubuntu@cgserver`。腾讯云防火墙拒绝公网 TCP 22、1234、10011；无规则的入站流量默认拒绝。保留 TCP 80/443（网页）、8443（DERP）、30033（TeamSpeak 文件传输），UDP 3478（STUN）、41641（Tailscale）、9987（TeamSpeak 语音）及既有 ICMP。控制台的防火墙和自动化助手可用于应急恢复，不将管理端口长期向所有来源开放。

Compose 将 PostgreSQL 5432、Adminer 8081、NapCat 6099 和旧 Nginx 1234 仅映射到 `127.0.0.1`。NapCat 另通过 Tailscale Serve 的 TCP 6099 转发到回环 6099，连接 Tailscale 后可直接访问 `http://cgserver:6099/webui/`；其余管理页面仍使用 SSH 本地隧道。后端 8888 仍供 Nginx 容器经宿主机网关访问。`csbot-management-firewall.service` 对公网网卡 `eth0` 限制管理端口，并过滤 Docker 转发的原始目的端口，不覆盖 Tailscale 或原有规则。迁移机器时先确认公网网卡名称。

在后端脚本经 Git 发布后启用 NapCat 内网入口（不需要重建容器）：

```bash
cd /home/ubuntu/csbot
sudo bash scripts/enable_napcat_tailnet.sh
sudo tailscale serve status
```

脚本仅设置 `--tcp=6099` 到 `tcp://127.0.0.1:6099`，保留 NapCat 原认证、Docker 回环绑定和公网过滤，不启用 Funnel。`--bg` 配置由 Tailscale 持久保存并在重启后恢复，见 [Tailscale Serve 官方说明](https://tailscale.com/docs/reference/tailscale-cli/serve#effects-of-rebooting-and-restarting)。回退只执行 `sudo tailscale serve --bg --tcp=6099 off`，不要使用会清空其他入口的 `serve reset`。

按 Git 发布后安装版本控制中的模板：

```bash
sudo install -m 0644 /home/ubuntu/csbot/deploy/apt/52csbot-security-upgrades /etc/apt/apt.conf.d/
sudo install -m 0755 /home/ubuntu/csbot/scripts/management_firewall.sh /usr/local/lib/csbot-management-firewall.sh
sudo install -m 0644 /home/ubuntu/csbot/deploy/systemd/csbot-management-firewall.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now csbot-management-firewall.service
```

自动更新必须允许 `${distro_id}:${distro_codename}-security`；只有 `jammy` 基础仓库时，任务虽成功运行也不会安装安全仓库补丁。模板开启每日检查和安全更新，关闭自动整机重启。Docker CE 第三方仓库、容器镜像、Python 锁文件和专用 Node 需单独维护；操作系统更新不会自动刷新它们。

内核与 Docker 升级前准备数据库可读归档、SQLite 一致性备份、会话与私有配置备份、原版本和镜像回退标签，并检查备份后的空间。预下载软件包和镜像，暂停守护、停止后端及会话容器，再升级、按需重启。跨重启维护时记录并临时关闭后端/守护的自动启动，依照服务清单恢复 Mihomo、Docker、各 Compose 项目和后端后，重新启用原有服务和 timer。不要反复启动 Steam 触发登录退避。

Steam 构建使用完整 `node:22.23.4-bookworm` 中自带的编译工具。服务器下载慢时可限定本次构建使用 `--network=host --build-arg http_proxy=http://127.0.0.1:7890 --build-arg https_proxy=http://127.0.0.1:7890`；代理参数不写入运行环境。验收包含 Node/sqlite3 原生模块、严格 AI preflight、受保护 API、QQ/Steam 就绪与公网 HTTPS，不只检查静态页面。

## 磁盘与日志维护

系统 journal 使用后端仓库模板 `deploy/systemd/journald.conf.d/60-csbot-retention.conf`，持久日志上限 512 MiB、单文件 64 MiB、最长保留 14 天，并尽量为文件系统留出 2 GiB。容量限制先到时，实际保留时间会短于 14 天；活跃日志和轮转时会有少量额外占用。这些设置不会删除其他目录以保证空闲空间。

模板按上述 Git 发布流程同步到服务器后安装，只重启日志服务，无需重启业务：

```bash
sudo install -d -m 0755 /etc/systemd/journald.conf.d
sudo install -m 0644 /home/ubuntu/csbot/deploy/systemd/journald.conf.d/60-csbot-retention.conf /etc/systemd/journald.conf.d/60-csbot-retention.conf
sudo systemctl restart systemd-journald
sudo journalctl --rotate
sudo journalctl --vacuum-size=512M --vacuum-time=14d
sudo journalctl --disk-usage
df -h /
```

清理 Docker 时先核对运行容器和镜像标签。无标签旧镜像可用 `docker image prune`，旧构建缓存可用 `docker builder prune --filter until=24h`；不执行包含卷的整体清理，也不把尚未运行容器的 `csbot-ai-python:1` 隔离镜像当作废弃镜像。有标签回滚镜像、数据库、QQ 会话、小图及历史备份须分别确定用途和保留范围。安装缓存若被运行中的进程占用，跳过清理，不强行抢锁。

硬链接图片备份可能继续持有被 LRU 淘汰的磁盘块。多次 `du` 会因遍历顺序把共享块算在不同目录，不能将各次结果直接相加；评估回收量应按 inode 去重并核实链接数。不要把图片备份存在解释为 LRU 未执行。

## NapCat 镜像更新

NapCat 由后端 Compose 的 `napcat` 服务管理。`latest` 标签不会随容器重启自动更新；用户授权升级后，先保留当前运行镜像的独立回滚标签，再执行 `docker compose pull napcat`，记录拉取摘要和实际运行版本。仅拉取镜像不会切换现有容器。

重建前核对生产已跟踪文件与 Compose，检查磁盘余量；停止 NapCat 后将 `napcat/config` 和 `ntqq` 制作成私有一致性归档，逐文件比较并保存 SHA256。备份失败应先恢复原容器，不能继续升级。不要输出 WebUI/OneBot 令牌或丢弃 QQ 会话。

检查新镜像 entrypoint 和包内默认配置：首次解包可能覆盖持久卷里的通用 `napcat.json`，升级前须比较并保留已有设置。2026-10-05 的镜像包只含默认 `config/napcat.json`，它与生产内容一致；不能据此假设未来镜像不会覆盖其他文件。准备就绪后在 `/home/ubuntu/csbot` 运行 `sudo -n docker compose up -d --no-deps --pull never --force-recreate napcat`，仅重建 NapCat；如需修改受版本控制的 Compose，仍按 Git 发布流程先提交、推送和服务器 `pull --ff-only`。

验收须包括运行日志中的 NapCat/QQ 版本、实际镜像 ID、配置比对、QQ 在线状态、OneBot 连接及自然群消息处理，并区分接收验证与发送验证。旧镜像与会话备份保留；回滚前评估新版 QQ 数据兼容性，需恢复会话时先停 NapCat，再从已校验归档恢复。具体版本与备份见 [服务清单](SERVER_SERVICES.md#2026-10-05-qq-收发恢复与-napcat-升级)。

## 后端发布

涉及 DSH/Mneme AI 改造时，先完成 [AI 专项发布前置步骤](AI_RUNTIME.md#发布前准备)，包括 Node、npm 固定依赖、隔离镜像、服务身份权限与前置检查。仅 pull 并重启不足以完成该版本部署。

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

## 后端内存保护

适用当前约 3.32 GiB、无 swap 的生产机。仅管理 `csbot.service`，不重启数据库、NapCat、Steam Monitor 或整机。脚本与模板在后端仓库：`scripts/memory_guard.py`、`deploy/systemd/csbot-memory-guard.{service,timer}`、`deploy/systemd/csbot.service.d/memory.conf`。

保护策略优先避免 OOM，允许短暂不可用，不等待聊天、AI 请求或观战任务结束：

| 条件 | 动作 |
| --- | --- |
| 后端 cgroup 内存达到 1600 MiB | 重启后端 |
| 整机 `MemAvailable` 不高于 384 MiB，且后端至少占 1024 MiB | 重启后端 |
| 北京时间每日 05:10（允许在 05:10–05:50 窗口补执行），且后端连续运行至少 6 小时 | 重启后端 |
| 后端 cgroup 达到 1800 MiB 硬上限 | 内核限制该服务内存；发生 cgroup OOM 时 systemd 清理整个服务并按失败策略重启 |

守护任务每次结束后 60 秒再次执行。主动重启要求后端运行至少 10 分钟，防止频繁重启；硬上限在这 10 分钟内仍有效。重启会重置运行时长，日常低峰重启不会在同一窗口重复。后端已停止、正在启动或停止时跳过，不自动拉起人为停用的后端。守护任务用文件锁避免与手工执行重叠。低内存由其他服务造成且后端不足 1024 MiB 时，不反复重启后端；本方案不能保证整机永不 OOM。

每次采样输出时间、后端内存、整机可用内存和运行时长到 journal，重启后再采样。停止最多等待 30 秒，超时由 systemd 清理服务进程。启动后等待最多 45 秒确认 systemd active 且 8888 HTTP 可响应；专用不存在路径返回 404 也视为 HTTP 就绪，这不代替受保护 API 和依赖验收。探测失败会使 guard 任务失败并留下日志，不在一次执行中循环重启。

安装前完成本地检查、提交推送和线上 `pull --ff-only`，记录目标 commit，确认该版本已通过验收。不要将其他任务未验收的提交顺带发布。以下安装会立即重启后端，并启用持续保护：

```bash
set -e
cd /home/ubuntu/csbot
python3 checks/memory_guard_test.py
python3 scripts/memory_guard.py --dry-run
sudo -n install -m 0644 scripts/memory_guard.py /usr/local/lib/csbot-memory-guard.py
sudo -n install -m 0644 deploy/systemd/csbot-memory-guard.service /etc/systemd/system/
sudo -n install -m 0644 deploy/systemd/csbot-memory-guard.timer /etc/systemd/system/
sudo -n install -d /etc/systemd/system/csbot.service.d
sudo -n install -m 0644 deploy/systemd/csbot.service.d/memory.conf /etc/systemd/system/csbot.service.d/
sudo -n systemd-analyze verify /etc/systemd/system/csbot-memory-guard.service /etc/systemd/system/csbot-memory-guard.timer /etc/systemd/system/csbot.service
sudo -n systemctl daemon-reload
sudo -n systemctl restart csbot.service
sudo -n systemctl enable --now csbot-memory-guard.timer
sudo -n systemctl start csbot-memory-guard.service
systemctl show csbot.service -p ActiveState -p MemoryCurrent -p MemoryMax -p OOMPolicy
systemctl list-timers csbot-memory-guard.timer --no-pager
sudo -n journalctl -u csbot-memory-guard.service -n 30 --no-pager
```

同时执行本手册的发布验收。比较重启后、30 分钟、数小时及次日的采样，区分基线、缓存增长和持续累积；重启释放内存本身不能证明泄漏。后端若正常任务已需要超过 1800 MiB，应优化或扩容后调整阈值，不能把反复触顶重启当成正常运行。守护任务无外部通知渠道，失败和内存趋势需查看 journal。

停用主动重启：`sudo systemctl disable --now csbot-memory-guard.timer`，并等待已启动的 guard 任务完成；这不会移除后端硬上限。需要完全回退时，另删除本节安装的 `/etc/systemd/system/csbot.service.d/memory.conf`，执行 `sudo systemctl daemon-reload`，在已授权维护窗口重启后端，再核对 `MemoryMax`。不要删除其他 drop-in 或生产数据。

## 独立 QQ 健康告警

后端仓库的 `scripts/health_alert.py` 与 `deploy/systemd/csbot-health-alert.{service,timer}` 每分钟检查一次服务器。它使用系统 Python 标准库，不加载机器人、不依赖业务数据库取得收件人，通过 NapCat 4.18.30 现有 WebUI 的认证 `/api/Debug/call` 直接调用 `send_private_msg`。仅使用回环 6099，不新增 OneBot 监听端口或改动 NapCat 配置，也不重启业务服务。

收件人必须在服务器私有文件 `/etc/csbot-health-alert.json` 中显式设置，例如 `{"recipients":["管理员QQ数字字符串"]}`，属主 root、权限 0600。不会自动扩展到 `SUPERUSERS`、群管理员或群聊。凭据只从既有 `napcat/config/webui.json` 读取，不写入命令行、告警或日志；HTTP 不使用系统代理。

| 检查 | 预警 | 严重 |
| --- | --- | --- |
| 根分区 | 使用率 ≥90% 或可用 <2 GiB | 使用率 ≥97% 或可用 <800 MiB |
| 根分区 inode | 剩余 ≤2% | 剩余 ≤1% |
| 整机 MemAvailable | ≤512 MiB | ≤256 MiB |
| 后端 cgroup 内存 | ≥1400 MiB | ≥1600 MiB |
| app.log / err.log 单文件 | ≥100 MiB | ≥500 MiB |
| 两个日志合计增速 | ≥5 MiB/分钟 | — |
| 已发消息归档积压 | 最老 ≥30 分钟 | 最老 ≥2 小时 |
| Steam 就绪 | 连续 3 次未就绪 | API 检查失败 |

另外检查 csbot、Mihomo、Docker、内存守护 timer，数据库/NapCat/Steam/两处 Nginx 容器、PostgreSQL 接收连接、后端 HTTP 响应、QQ 在线状态，以及一次采样间隔内至少两次的后端自动重启。HTTP 路由响应只代表事件循环可响应，不代替数据库或业务功能验收。归档仅只读统计，不重发群消息或修改 outbox。

通常连续两次异常才通知；严重资源压力与超大日志立即通知。预警每 6 小时、严重每小时提醒一次，升级立即通知；连续两次正常发送一次恢复通知。发送失败至少间隔 5 分钟重试，按收件人记录成功状态；QQ 不可用期间已确认的异常即使随后恢复，也保留到能发出补报。状态、锁和冷却写在 `/run/csbot-health-alert/` 的 tmpfs，根盘满时仍可工作；重启整机会清空状态。不会自动清理磁盘或重启服务。

先运行 `python3 checks/health_alert_test.py`。按 Git 发布流程提交、推送并在服务器 `pull --ff-only` 后，准备好私有收件配置，再安装：

```bash
cd /home/ubuntu/csbot
sudo -n python3 scripts/health_alert.py --dry-run
sudo -n install -m 0644 scripts/health_alert.py /usr/local/lib/csbot-health-alert.py
sudo -n install -m 0644 deploy/systemd/csbot-health-alert.service /etc/systemd/system/
sudo -n install -m 0644 deploy/systemd/csbot-health-alert.timer /etc/systemd/system/
sudo -n systemd-analyze verify /etc/systemd/system/csbot-health-alert.service /etc/systemd/system/csbot-health-alert.timer
sudo -n systemctl daemon-reload
sudo -n systemctl enable --now csbot-health-alert.timer
sudo -n systemctl start csbot-health-alert.service
systemctl list-timers csbot-health-alert.timer --no-pager
sudo -n journalctl -u csbot-health-alert.service -n 10 --no-pager
```

`--dry-run` 只检查，不发送消息或写状态。真实验收须查看 `notification_delivered`（平台确认 message_id）及收件情况，不能把配置存在或 dry-run 成功当作通知已送达。停用使用 `sudo systemctl disable --now csbot-health-alert.timer`，必要时等待正在运行的 oneshot 完成；不会影响内存守护。告警依赖主机和 NapCat/QQ 通道，整机掉电、断网或 QQ 故障期间无法即时送达；需要外部独立通道才能覆盖这些情况。NapCat 升级时须复核 WebUI 认证与 Debug/call 返回结构。

## 前端发布

1. 在 `csbot-front` 提交并推送 `main`。
2. 等待该 commit 对应的 GitHub Actions `Node.js CI` 完成。工作流使用 Node.js 22，执行 `npm ci`、`npm run build`，将 `dist` 推送到 `build-output`。
3. 在服务器更新产物仓库：

```bash
set -e
cd /home/ubuntu/csbot/dist
umask 022  # 私有备份的077不能沿用到公开静态文件/目录
git fetch origin build-output
git pull --ff-only origin build-output
git rev-parse HEAD
curl -fsS --max-time 10 http://127.0.0.1:1234/ai-chat -o /dev/null
```

静态文件通过 bind mount 直接供 Nginx 读取，无单独前端进程。只更新静态文件通常无需重启后端；涉及后端契约的变更应协调两者发布。修改 Nginx 模板或 Compose 配置时需另行验证配置并应用到容器，单纯拉取文件不能保证运行中配置已经更新。

## 80 端口个人主页

`csbot` Compose 的 `homepage` 服务使用独立 Nginx 容器 `csbot-homepage`，映射宿主机 `80:80` 和 `443:443`。配置源为 `csbot/assets/homepage.conf`，在生产只读挂载至 `/etc/nginx/conf.d/default.conf`；`csbot/assets/homepage/` 只读挂载至 `/usr/share/nginx/html/`，由 Nginx 直接提供本地文件。证书目录 `/var/lib/ave-mujica-tls` 只读挂载至 `/etc/nginx/tls`，HTTPS 使用 `current/fullchain.pem` 和 `current/privkey.pem`。原 CSBot 前端仍由 `csbot-nginx` 的 1234 端口提供。

主页源自 `https://juruocjl.github.io/`，2026-10-02 下载并保存为 `csbot/assets/homepage/index.html`；当前页面的样式全部内联，无外部脚本或图片。访问时不请求 GitHub Pages，主域 HTTPS 未知路径返回本地 404。个人主页不需要 `csbot-front` 构建产物，域名入口为 `https://ave-mujica.top/`。80 端口返回 308，默认跳转到 `https://ave-mujica.top$request_uri`，保留路径和查询参数；IP、主域或其他 Host 的 HTTP 访问均使用固定主域。`cs.ave-mujica.top` 单独跳转到同名 HTTPS dashboard。证书覆盖主域和一级子域，不覆盖 IP 地址。

`www.ave-mujica.top` 使用 CNAME 指向 `ave-mujica.top`，默认解析线路、TTL 600 秒。HTTP 沿用统一跳转，HTTPS 使用单独的 `server_name www.ave-mujica.top` 块返回同一 308 目标。现有通配符证书覆盖 `www`，无需重新签发。

DNS 恢复时先核对现有记录；`scripts/ensure_www_dns.py` 默认只读检查 `www`，`--label cs` 检查 dashboard 子域。加上 `--apply --backup-dir <私有目录>` 仅在所选标签没有记录时新增指向固定主域的 CNAME。已有正确记录则跳过，冲突记录拒绝覆盖；标签仅允许 `www`、`cs`。通过 transient systemd service 的 `EnvironmentFile=/etc/derper-acme/aliyun.env` 沿用现有凭据，不把凭据放入参数或日志；脚本只输出记录状态及固定错误码，DNS 前后状态保存在私有目录。修改 DNS 后须检查权威解析和 HTTP/HTTPS 跳转，客户端可能暂时保留此前的负缓存。

主页页脚保留 `浙ICP备2026084445号`，链接到 `https://beian.miit.gov.cn/`。原站内容更新后，先下载到临时文件，比较并合并至 `csbot/assets/homepage/index.html`，保留本地备案页脚，再按 Git 发布流程更新服务器。在此 Mac 上沿用现有代理下载：

```bash
curl --proxy socks5h://127.0.0.1:7895 -fsS --connect-timeout 5 --max-time 30 https://juruocjl.github.io/ -o /tmp/csbot-homepage-origin.html
```

先按本手册完成本地检查、提交推送、线上已跟踪文件与分支检查，并确认共享证书已经签发及发布，再在服务器执行：

```bash
set -e
cd /home/ubuntu/csbot
git pull --ff-only origin main
sudo -n docker compose config --quiet
sudo -n docker compose run --rm --no-deps --pull never homepage nginx -t
sudo -n docker compose up -d --no-deps --pull never --force-recreate homepage
sudo -n docker exec csbot-homepage nginx -t
curl -fsSI --max-time 10 http://127.0.0.1/  # 应为308，Location为https://ave-mujica.top/
curl -fsS --resolve ave-mujica.top:443:127.0.0.1 --max-time 10 https://ave-mujica.top/ -o /dev/null
curl -fsSI --resolve www.ave-mujica.top:443:127.0.0.1 --max-time 10 https://www.ave-mujica.top/  # 应为308并跳转主域
```

上述命令沿用服务器已安装的 `nginx:alpine` 镜像；首次恢复须先准备该镜像。配置文件被 Git 替换后，须重建 `homepage` 以刷新单文件 bind mount；仅更新静态目录内文件通常无需重建。仅操作 `homepage`，并执行本手册的业务验收；另从公网检查 HTTP 308 的 Location、路径/查询参数保留、跟随跳转后 HTTPS 200 和首页内容。2026-10-02 发布时公网已可达，无需调整主机防火墙或云安全组。

停用入口可执行 `sudo -n docker compose stop homepage`；配置回退通过 revert 提交及正常 Git 发布流程完成。此服务没有业务数据卷，不涉及数据库迁移。

## Dashboard HTTPS 与网址设置

`cs.ave-mujica.top` 的默认线路 CNAME 指向 `ave-mujica.top`，TTL 600 秒；使用现有主域/通配符证书。`homepage` 的同名 HTTPS server 直接读取只读挂载的 `dist/`（容器内 `/usr/share/nginx/dashboard/`）和 `imgs/`，SPA 路由回退至 `index.html`。`/api/` 直接代理到宿主机 `8888`，传递认证请求头；关闭代理缓冲、读取超时 660 秒，以支持 AI SSE 事件流。该入口与原 1234 端口读取同一构建产物，不增加页面上游请求或新的前端构建流程。

生产 `/home/ubuntu/csbot/.env.prod` 设置 `CS_DOMAIN="https://cs.ave-mujica.top"`，末尾不带斜杠。该值由分享、AI 对话、日报和周报模块启动时读取，不属于 `/admin/config` 的数据库动态配置。调整时先私有备份环境文件，确认 AI 排队/运行任务及 pending/sent 发件箱均为 0，暂停内存守护 timer，停止后端并用 SQLite backup API 备份 AI 状态；只替换该变量，保留其他环境行及原文件属主/权限，再通过 `csbot.service` 启动前置检查恢复后端和 timer。凭据和环境文件不入库。新服务器恢复须同时准备 `dist/`、`imgs/`、共享证书和 DNS。

验收使用正常公网 DNS：HTTP `cs` 应返回 308 至同名 HTTPS，保留路径与查询参数；HTTPS 首页、登录页和 AI 路由应返回 dashboard，JS/CSS 应是实际资源。用既有令牌只读验证 `/api/auth/verify`、历史对话及 SSE，匿名请求应被拒绝；确认主域个人主页及 www 跳转仍正常。更新 `dist` 的流程仍遵守 GitHub Actions 构建与 `build-output` 发布规则。

## 主域与通配符证书

沿用服务器既有 `/usr/local/bin/lego` 和 `/usr/local/sbin/derper-acme-run`，通过阿里云 DNS-01 签发一张包含 `ave-mujica.top`、`*.ave-mujica.top` 的证书。`/etc/derper-acme/aliyun.env` 保留原 DNS 凭据；注册账户仍位于 `/var/lib/derper-acme/accounts`，原账户邮箱由服务器私有的 `/etc/derper-acme/account.env` 提供，变量名为 `DERPER_ACME_EMAIL`。该配置由 root 持有、权限 0600，不入 Git，也不新建 ACME 账户或 DNS 凭据。

签发源文件为 `/var/lib/derper-acme/certificates/ave-mujica.top.{crt,key}`。部署脚本安装为 `/usr/local/sbin/ave-mujica-cert-deploy`，验证可信证书链、有效期、主域/通配符 SAN、主域及子域匹配和私钥公钥一致性。每版材料保存在 `/var/lib/ave-mujica-tls/releases/<证书链SHA256>/`，通过一次替换 `current` 符号链接发布成套证书与私钥。目录 root:derper 0750、私钥 root:derper 0640；保留旧版材料供回退，不向外输出私钥。

DERP 的 `deploy/systemd/derper.service.d/cert.conf` 将手工证书目录设为 `/var/lib/ave-mujica-tls/current`，其中 `derp.ave-mujica.top.{crt,key}` 指向同一套材料。DERP 仍使用 TCP 8443/UDP 3478，保留原身份、Tailscale 客户端验证和服务加固配置。

`derper-acme-renew.timer` 每日北京时间 02:00–03:00 检查，剩余不超过 30 天时续签。`derper-acme-renew.service` 签发完成后运行部署脚本；每个消费者只在版本变化或上次加载失败时重启/重载，并通过本机可信 TLS 握手验证实际提供的证书。部署失败后，即使 lego 的新证书已保存，下一次检查也会再次部署。签发、续签、部署共用 `/run/lock/ave-mujica-acme.lock`，避免并行修改。

首次切换或授权更换 SAN 时，先备份原证书、ACME 状态、systemd 与 Compose，暂停续签 timer。在独立 Git 检出目录经 `pull --ff-only` 获取已推送版本，安装签发 unit 和脚本，使用已注册账户签发；签发成功后再更新生产检出目录及其他配置。以下命令仅用于已准备好账户配置和备份的首次安装，普通页面发布不重复签发：

```bash
sudo -n install -m 0755 scripts/ave_mujica_cert_deploy.py /usr/local/sbin/ave-mujica-cert-deploy
sudo -n install -m 0644 deploy/systemd/derper-acme-issue.service /etc/systemd/system/
sudo -n systemctl daemon-reload
sudo -n systemctl start derper-acme-issue.service
sudo -n /usr/local/sbin/ave-mujica-cert-deploy --check
sudo -n /usr/local/sbin/ave-mujica-cert-deploy --publish-only
sudo -n install -d /etc/systemd/system/derper.service.d
sudo -n install -m 0644 deploy/systemd/derper.service.d/cert.conf /etc/systemd/system/derper.service.d/
sudo -n install -m 0644 deploy/systemd/derper-acme-renew.{service,timer} /etc/systemd/system/
sudo -n systemctl daemon-reload
sudo -n docker compose run --rm --no-deps --pull never homepage nginx -t
sudo -n docker compose up -d --no-deps --pull never --force-recreate homepage
sudo -n /usr/local/sbin/ave-mujica-cert-deploy
sudo -n systemctl enable --now derper-acme-renew.timer
sudo -n systemctl start derper-acme-renew.service
```

验收包含主域 HTTPS 200、HTTP 308 跳转到固定 HTTPS 主域、备案页脚、DERP 8443 与主页 443 的实际证书一致、续签任务 success、timer enabled，以及原业务服务检查。公网访问须核对 DNS 和云安全组；证书签发成功本身不能证明 443 公网可达。

回退时先停续签 timer；从私有备份恢复原续签 unit/timer，移除本次新增的 DERP `cert.conf` 并 daemon-reload，重启 DERP 以加载保留的旧目录。主页通过 revert 提交及 Git 流程恢复原配置。保留新旧证书和 ACME 状态，不删除凭据、身份或业务数据。新服务器还须从原机安全恢复现有 lego、日志脱敏 wrapper、账户和 DNS 凭据。

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

## 斗鱼直播监控

监控列表由数据库热配置 `live_watch_list` 管理，在 `/admin/config` 保存后下一轮（每两分钟）生效。斗鱼配置推荐使用真实房间 ID，例如玩机器为 `dy_6979222`；靓号 `dy_6657` 可通过官方移动页面解析到同一房间，不需要修改现有数据库名单。

监控读取斗鱼官方 `https://www.douyu.com/betard/<真实房间ID>`，每次请求限时 10 秒。只有 `show_status=1` 且 `videoLoop=0` 才可能算开播；平台回放及标题明确包含“回放、重播、录播、轮播”的房间按非直播处理，在 `/直播状态` 显示“（回放） 0”，不触发开播提醒。标题过滤采用保守策略，直播标题带上述词也会被抑制；手工重播若既没有回放字段也没有标题说明，接口无法据此辨认视频内容。

移动页面的 `isLive=1` 同样包含回放，因此只用于解析真实房间 ID，不能充当开播状态的备用来源。旧 `open.douyucdn.cn` 接口可能把靓号解析到其他房间，未采用。原 `doseeing.com` 已于 2026-09-30 停止直播数据服务，不再使用。

请求失败、缺字段或未知状态不保存为下播，保留数据库上一有效状态，并在状态列表显示对应 ID 的“查询失败”。这避免暂时失败制造虚假的下播→开播提醒；失败期间真正发生的下播/再开播可能无法观测。

本地合成回归及服务器只读实测（不加载机器人、不访问数据库、不发 QQ）：

```bash
cd csbot
uv run --frozen python checks/live_watcher_test.py
uv run --frozen python checks/douyu_live_probe.py dy_6979222 dy_6657 dy_2267291 dy_9999 dy_100
```

实测应当场确认一个正在回放、一个真实直播和一个下播的房间；房间状态会变化，不能永久固定上述房间的预期状态。可用 `--expect dy_房间ID=replay`（或 `live` / `offline`）断言当次预期。部署后还须检查自然调度日志和现有监控状态，不能只凭 HTTP 200 或模拟回放验收。

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
