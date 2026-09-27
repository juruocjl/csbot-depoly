# csbot-depoly

部署仓库，包含以下 Git 子模块：

| 目录 | 仓库 |
| --- | --- |
| `csbot-front` | https://github.com/juruocjl/csbot-front |
| `csbot` | https://github.com/juruocjl/csbot |
| `steam_monitor_js` | https://github.com/juruocjl/steam_monitor_js |

## 获取代码

```sh
git clone --recurse-submodules git@github.com:juruocjl/csbot-depoly.git
```

已有克隆可执行：

```sh
git submodule update --init --recursive
```

子模块版本由本仓库记录的 commit 固定。

## 部署文档

部署文档和约束统一在本仓库根目录维护：

| 文件 | 内容 |
| --- | --- |
| [DEPLOY.md](DEPLOY.md) | 后端、前端、Steam Monitor 发布流程、配置与验收 |
| [SERVER_SERVICES.md](SERVER_SERVICES.md) | 2026-09-27 服务器调查、运行服务、启动顺序、备份与恢复 |
| [DELIVERY_CHECK.md](DELIVERY_CHECK.md) | 本地交付检查与测试库使用规则 |
| [AGENTS.md](AGENTS.md) | 部署操作、凭据、数据和子模块管理约束 |

当前生产连接为 `ssh ubuntu@cgserver`，沿用系统已有 SSH 配置。服务器保持独立的后端、前端产物和 Steam Monitor 工作目录；父仓库尚未取代现行部署方式。

### 文档迁移对应关系

| 原位置 | 新位置 |
| --- | --- |
| `csbot/scripts/DEPLOY.md` | `DEPLOY.md` |
| `csbot/SERVER_SERVICES.md` | `SERVER_SERVICES.md` |
| `csbot/README.md` 的 Deploy / Local delivery check | `DEPLOY.md` / `DELIVERY_CHECK.md` |
| `steam_monitor_js/README.md` 的 Docker 部署 | `DEPLOY.md` |
| 分散在以上文档的操作限制 | `AGENTS.md` |

子仓库 README 保留入口。Compose、systemd 模板、发布脚本仍在子仓库；业务文档 `csbot/scripts/MAJOR_STAGE.md`、数据库迁移模板说明和组件 API/开发文档保留原位置。

修改子模块文档后，须先在对应子仓库提交、推送，再提交父仓库中的子模块版本更新。子仓库指向父仓库的在线文档链接要在本父仓库发布后才可访问。

## 部署服务器 SSH 密钥

系统现有 SSH 认证已验证可连接 `ubuntu@cgserver`。另生成的一对 Ed25519 密钥作为本机备用，尚未确认安装到服务器：

- 私钥：`.deploy/ssh/csbot-deploy-ed25519`
- 公钥：`.deploy/ssh/csbot-deploy-ed25519.pub`

`.deploy/` 已被 Git 忽略，密钥不会随克隆分发。私钥无口令，供自动化连接使用；私钥权限为 `600`，所在目录权限为 `700`。仅将公钥追加到服务器目标用户的 `~/.ssh/authorized_keys`，私钥保留在本机。

服务器安装公钥后，在本仓库目录连接（替换用户名和地址）：

```sh
ssh -o IdentitiesOnly=yes -i .deploy/ssh/csbot-deploy-ed25519 deploy@SERVER_HOST
```

首次连接前应核对服务器主机密钥指纹。
