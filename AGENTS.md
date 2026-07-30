# AGENTS.md

## 项目定位

本仓库是 `ArgoFusion`，不是原版 `sba`。它是面向 Debian/Ubuntu、systemd、固定 Cloudflare Argo Token 的中文轻量管理脚本。

主要文件：

- `argofusion.sh`：唯一安装和管理入口。
- `argofusion.env.example`：配置格式示例，不会被脚本自动读取。
- `argofusion.sh.sha256`：GitHub 模式安装时用于校验入口脚本，修改脚本后必须同步更新。
- `README.md`：面向用户的安装、更新和运维说明。
- `sba/`：原版 SBA 本地对照树。除非任务明确要求更新对照版本，否则不得修改、删除或提交该目录。

安装后的主要文件：

- `/etc/argofusion/argofusion.sh`
- `/etc/argofusion/bin/sing-box`
- `/etc/argofusion/bin/xray`
- `/etc/argofusion/bin/cloudflared`
- `/etc/argofusion/config/argofusion.env`
- `/etc/argofusion/config/nodes.conf`
- `/etc/argofusion/config/sing-box.json`
- `/etc/argofusion/config/xray.json`
- `/etc/argofusion/data/nodes.txt`
- `/etc/argofusion/subscriptions/subscription.*`
- `/etc/argofusion/backup/`
- `/etc/nginx/conf.d/argofusion.conf`
- `/etc/systemd/system/argofusion-core.service`
- `/etc/systemd/system/argofusion-tunnel.service`
- `/usr/local/bin/af`

## 配置与数据格式

`/etc/argofusion/config/argofusion.env` 由脚本生成并通过 Bash `source` 读取，保存 UUID、Argo 域名、优选入口、Token、回源端口、WARP 配置和当前 `CORE` 选择。必须使用安全转义、`600` 权限和同目录临时文件原子替换；不得把未经验证的用户输入直接拼入该文件或 systemd unit。

`/etc/argofusion/config/nodes.conf` 每行格式为：

```text
标签|协议|WS路径|本地端口|SOCKS5
```

其中协议仅允许 `vless`、`vmess`、`trojan`；SOCKS5 留空表示 direct，否则格式为 `主机:端口:用户名:密码`。标签、WS 路径和本地端口必须全局唯一。改变该格式时必须同时迁移旧文件，不能静默破坏已有节点。

生成文件包括明文节点、Base64、Clash/Mihomo、Clash Provider、sing-box、Shadowrocket 订阅和自动适配订阅 QR。它们是派生数据，应由 `generate_nodes()` 统一重建，不应成为独立配置源。`config/` 与 `data/` 必须保持 `700`，但 `/etc/argofusion/subscriptions/` 必须为 `755`，否则 Nginx 工作进程无法读取 `alias` 订阅文件而返回 403。Clash/Mihomo 三种 WS 节点必须显式启用 UDP，并使用 Chrome 指纹和 `http/1.1` ALPN；VLESS、VMess 订阅必须保留 XUDP（`packetEncoding=xudp`、`packet-encoding: xudp`、`packet_encoding: "xudp"`）。sing-box TLS 保持核心默认版本协商，WS `headers.Host` 必须是字符串。

## 关键函数职责

- `load_env()` / `save_env()`：读取和原子保存项目环境配置。
- `ensure_nodes_config()` / `validate_nodes_config()`：创建默认节点并校验动态节点数据。
- `write_sing_box_config()` / `write_xray_config()`：从同一份 `nodes.conf` 分别生成并校验两套核心配置；`write_all_core_configs()` 用于首次安装，`write_available_core_configs()` 用于配置事务。两者都必须保持 `WARP → 节点 SOCKS5 → direct` 路由顺序。
- `write_nginx_config()`：生成本地 WS 反代和 UUID 订阅入口，并执行 `nginx -t`。
- `write_services()`：只写项目专属 systemd unit；包含 Token 的文件必须仅 root 可读。
- `generate_nodes()`：从环境配置和 `nodes.conf` 生成全部节点及订阅文件。
- `apply_runtime_config()`：配置修改事务；验证失败必须恢复快照。
- `sync_versions()`：核心版本比较、确认、暂存、校验、原子替换和失败回滚。
- `backup_project()` / `restore_project()`：仅归档与恢复节点配置 `nodes.conf`；解压前必须拒绝路径穿越、符号链接和特殊文件，恢复不得替换脚本、核心二进制或整个 `/etc/argofusion`。
- `uninstall_project()`：所有权检查后的项目范围卸载，不得扩大删除边界。

## 产品边界

保持以下范围：

- 仅支持 Debian/Ubuntu + systemd。
- 仅支持 amd64 和 arm64。
- 仅支持固定 Argo Token，不加入临时隧道、Argo JSON 或 Cloudflare API 建隧道。
- 支持 Sing-box 与 Xray 二选一；`af -c` 可在两者间切换，两个私有二进制与 `sing-box.json`、`xray.json` 可同时保留并从统一配置重建。Xray 仅适配既有的 VLESS、VMess、Trojan WS + TLS 节点，不加入 Reality、Hysteria2、XHTTP 或其他协议。
- 默认保留 VLESS、VMess、Trojan 各一个 WS 节点，并允许通过 `af -c` 动态添加、修改或删除这三种协议的 WS 节点。
- 允许节点按 inbound tag 与 WS 路径绑定独立 SOCKS5 出站。
- 允许用户指定目标网址优先通过 Cloudflare 官方 WARP 客户端的本地 SOCKS5 proxy 出站；未命中时仍遵循节点 SOCKS5 或 direct。
- 首次安装和缺省环境配置的 Cloudflare 优选入口为 `bestcf.cdn.fiatnorm.us.kg:443`；用户仍可在安装或 `af -c` 中修改。
- 保持中文交互，不加入英文模式。
- 日常管理必须使用 VPS 本地安装脚本，不得改成每次执行都从 GitHub 拉取仓库脚本。
- BBR 不是项目内置能力，只能作为明确标注的第三方外部工具保留。

不要因为“对齐原版 SBA”而扩大产品范围。只对齐可靠性、回退、版本选择和运行语义。

## 命名规则

- 仓库入口必须使用 `argofusion.sh`，不要重新创建 `sba.sh`。
- 管理命令保持为 `af`。
- systemd 服务保持为 `argofusion-core.service` 和 `argofusion-tunnel.service`。
- 核心二进制必须放在 `/etc/argofusion/bin/`，不得写入或删除 `/usr/local/bin/sing-box`、`/usr/local/bin/xray`、`/usr/local/bin/cloudflared`。
- 新增项目文件优先使用 `argofusion` 或 `argofusion` 前缀，避免与原版 SBA 文件混淆。

## 安全约束

安装、更新和卸载修改必须满足：

- 不得无条件删除整个 `/etc/argofusion`。
- 修改已有配置前必须保留备份，仅重建本项目管理的文件。
- 不得覆盖第三方同名 systemd 服务。只有带本项目 `Description=SBA ...` 或 `Description=ArgoFusion ...` 标记的旧服务可以迁移。
- 卸载必须检查 `/etc/argofusion/managed` 所有权标记。
- 卸载只删除本项目私有核心、服务、Nginx 配置、`af` 链接和节点文件。
- 核心更新必须先比较版本并请求确认。
- 下载必须包含连接超时、总超时、重试、GitHub 代理回退和 SHA256 校验。
- 更新流程必须遵循：下载到临时文件 → SHA256 与可执行性校验 → 所选内核配置检查 → 备份 → 原子替换 → 重启验证 → 失败回滚。
- 不得在脚本或示例文件中加入固定公共 UUID、Token 或未经项目明确指定的公共域名；当前约定的默认优选入口为 `bestcf.cdn.fiatnorm.us.kg:443`，且必须允许用户覆盖。
- 不得嵌入公共 WARP WireGuard 私钥、固定 WARP 账户或非官方 WARP 注册凭据；WARP 使用用户 VPS 上安装的 Cloudflare 官方客户端。
- Argo 域名必须由用户输入；首次安装 UUID 应自动随机生成。
- Argo Token 必须拒绝空白、换行和 systemd 控制字符；包含 Token 的配置和 unit 权限不得宽于 `600`。
- 恢复用户指定的归档前必须先检查 gzip、成员路径和文件类型，并使用 `--no-same-owner --no-same-permissions` 解压。

## 交互和运行语义

- 优选入口使用单行 `域名/IP:端口` 输入；IPv6 使用 `[地址]:端口`。
- `af -n` 必须输出当前全部订阅链接、唯一一张自动适配订阅 QR 和明文节点；不得为其他订阅或单个节点重复输出 QR。自动适配订阅 QR 同时显示在网页订阅面板中，并作为单独的 `/auto-qr.svg` 订阅面板资源提供。
- 节点连接地址使用优选入口，WebSocket Host 与 TLS SNI 使用 Argo 域名。
- 修改 Token 或优选入口后，应重新生成节点并执行健康检查。
- 健康检查必须测试 `/etc/argofusion/config/nodes.conf` 中的全部 WS 路径，默认包括 `/argo-vl`、`/argo-vm`、`/argo-tr`。
- `af -c` 必须集中管理 Token、Argo 域名、优选入口、本地端口、UUID、动态节点、节点 SOCKS5 出站和 WARP 目标网址。
- WARP 域名规则必须位于节点 SOCKS5 规则之前，保持 `目标网址 WARP → 节点 SOCKS5 → direct` 的优先级。
- `af -x` 必须检查配置、Token、服务、动态端口、全部公网 WS 路径、核心版本、WARP（启用时）和最近日志。
- `af -k/-l` 必须校验 `/etc/argofusion/managed`；只备份和恢复 `/etc/argofusion/config/nodes.conf` 节点配置，恢复失败必须自动回滚，不得用旧归档覆盖当前脚本、核心或项目目录。
- 终端配色必须在非 TTY、`TERM=dumb` 或 `NO_COLOR` 环境自动关闭，不得向日志和管道写入 ANSI 控制符。
- 状态诊断保持简洁，并包含公网 IP、脚本/核心版本、内存、systemd 状态、监听端口和最近错误。
- 普通启停、查看节点、修改配置和卸载不得执行 `git pull` 或重新下载仓库脚本。
- `af -i` 必须提供“本地重装”和“在线更新安装”两种模式；仅在线更新安装联网更新项目脚本，校验通过后必须先替换 `/etc/argofusion/argofusion.sh`，再切换到新版脚本进程继续安装，其他日常命令仍运行 VPS 本地脚本。

## 修改要求

- 优先小范围修改现有函数，不做无关重构。
- 保持 `set -Eeuo pipefail`。
- 所有变量引用都应正确加引号，临时文件使用 `mktemp`。
- 保持脚本为 LF 行尾；不要引入 CRLF。
- 修改命令、菜单、路径或行为时，同步更新 `README.md`。
- 修改 `argofusion.sh` 后必须同步更新脚本内 `VERSION`、`README.md` 版本记录和 `argofusion.sh.sha256`；纯文档修改且脚本字节未变化时不得伪造版本变更。
- 配置事务的快照必须覆盖环境、节点、运行配置、服务文件和全部派生订阅文件；生成或服务验证失败时必须恢复文件并重新载入环境变量。
- `validate_environment()` 是生成 Sing-box/Xray、Nginx 和订阅文件前的共同边界；已有环境文件中的内核选择、域名、端口、UUID、Token 和 WARP 配置不得绕过校验直接写入运行文件。
- 终端状态行显示 IPv6 优选入口时必须保留 `[地址]:端口` 形式；订阅 URL 按 UI 设计稿使用白色下划线，不输出额外逐条分隔线。
- `TERMINAL_UI_DESIGN.md` 是终端输出的视觉合同；更新终端 UI 时必须同步脚本、该设计稿和 README。当前基准采用 ArgoFusion 斜体字标、64 列分隔线、ANSI `97` 亮白正文、`◆ / ▸ / ✓ / ! / ✗ / • / ›` 图标语义，以及 `main/back/cancel/none` 四种页面提示模式。
- 主面板服务名称使用“Argo Tunnel”和“代理核心”；只读页不显示 `0` 操作提示，普通返回静默，取消配置明确提示但不得写入半成品。节点表固定使用 14/7/18/5/12 显示宽度，并且 SOCKS5 只能展示 `direct`、`SOCKS5` 或主机端口，绝不输出用户名或密码。
- `af -x` 必须诊断 `config`、`data` 的 `700` 与 `subscriptions` 的 `755`；健康时汇总最近日志，出现 ERROR 或诊断失败时才展开相关日志。
- 不要修改用户已有的无关文件或清理未跟踪的 `sba/` 对照树。
- 未在真实 VPS 上验证时，不得宣称 systemd、Nginx、Cloudflare 或公网 WS 已端到端通过。

## 版本与发布

Git 远端可能同时包含原版 SBA 和本项目仓库。发布前必须执行 `git remote -v`，确认目标为 `https://github.com/Fiatnorm/ArgoFusion.git`，不得把本项目改动推送到 `fscarmen/sba`。

向测试分支 `codex/test-directory-layout-v2.14.3` 推送前，必须为该次推送递增并统一 `argofusion.sh` 的 `VERSION`、README 版本记录、终端设计稿适用版本和 `argofusion.sh.sha256`；不得把未标记版本的改动推送到该测试分支，便于逐次核算。

发布范围默认只包含：

- `argofusion.sh`
- `argofusion.env.example`（格式确有变化时）
- `argofusion.sh.sha256`
- `README.md`
- `AGENTS.md`
- 其他明确属于本项目的新文件

不得使用会顺带提交 `sba/` 或用户无关文件的宽泛暂存命令。发布前核对 `git diff --cached --stat` 和 `git diff --cached`。

## 最低验证

在提交修改前执行：

```bash
bash -n argofusion.sh
git diff --check
```

涉及输入解析或节点生成时，还要执行相应函数级测试。涉及安装、更新、卸载、systemd、Nginx 或 Cloudflare 时，应尽可能在 Debian/Ubuntu 测试机验证，并至少检查：

```bash
nginx -t
# Sing-box: /etc/argofusion/bin/sing-box check -c /etc/argofusion/config/sing-box.json
# Xray: /etc/argofusion/bin/xray run -test -c /etc/argofusion/config/xray.json
systemctl is-active nginx argofusion-core argofusion-tunnel
ss -lnt
journalctl -u argofusion-core -u argofusion-tunnel -n 100 --no-pager
```

如果没有 VPS 环境，应明确列出未完成的运行时验证，不得用静态检查替代运行证明。

脚本发生变化时还必须验证发布哈希和行尾：

```bash
sha256sum -c argofusion.sh.sha256
if grep -n $'\r' argofusion.sh; then
  echo "检测到 CRLF"
  exit 1
fi
```

输入解析或安全边界变更至少覆盖对应回归用例：

- IPv4、域名和 `[IPv6]:端口`，包括带前导零端口。
- Token 合法字符，以及空格、换行和控制字符拒绝。
- 节点标签的精确添加、修改和删除。
- 合法备份可恢复；越界路径、符号链接、硬链接和特殊文件必须被拒绝。
- 非 TTY、`TERM=dumb`、`NO_COLOR` 下无 ANSI 控制符。
