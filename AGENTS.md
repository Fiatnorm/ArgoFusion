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

- `/etc/afs/argofusion.sh`
- `/etc/afs/bin/sing-box`
- `/etc/afs/bin/xray`
- `/etc/afs/bin/cloudflared`
- `/etc/afs/config/argofusion.env`
- `/etc/afs/config/nodes.conf`
- `/etc/afs/config/sing-box.json`
- `/etc/afs/config/xray.json`
- `/etc/afs/data/nodes.txt`
- `/etc/afs/data/traffic.db`
- `/etc/afs/subscriptions/subscription.*`
- `/etc/afs/backup/`
- `/etc/nginx/conf.d/argofusion.conf`
- `/etc/systemd/system/afs-core.service`
- `/etc/systemd/system/afs-tunnel.service`
- `/etc/systemd/system/afs-traffic.service`
- `/etc/systemd/system/afs-traffic.timer`
- `/usr/local/bin/af`（`/usr/local/bin/AF` 为等价入口）

## 配置与数据格式

`/etc/afs/config/argofusion.env` 由脚本生成并通过 Bash `source` 读取，保存 UUID、Argo 域名、优选入口、Token、回源端口、WARP 配置、流量统计 API 端口和当前 `CORE` 选择。必须使用安全转义、`600` 权限和同目录临时文件原子替换；不得把未经验证的用户输入直接拼入该文件或 systemd unit。

`/etc/afs/config/nodes.conf` 每行格式为：

```text
标签|协议|传输路径|本地端口|SOCKS5
```

其中协议仅允许 `vless`、`vmess`、`trojan`、`vless-xhttp`、`shadowsocks`；前三类和 Shadowsocks 使用 WS，`vless-xhttp` 使用 XHTTP。SOCKS5 留空表示 direct，否则格式为 `主机:端口:用户名:密码`。标签、传输路径和本地端口必须全局唯一。Shadowsocks 固定使用 `chacha20-ietf-poly1305` 并以全局 UUID 为密码；改变字段格式时必须同时迁移旧文件，不能静默破坏已有节点。

生成文件包括原始节点订阅、Base64、Clash/Mihomo、sing-box 与自动适配订阅 QR；活动订阅入口仅保留自适应、Base64、Clash/Mihomo、Sing-box、原始节点订阅五类。Clash Provider 与 Shadowrocket 文件仅可作为迁移、快照回滚和卸载清理的废弃文件，不得重新生成或公开路由。派生数据应由 `generate_nodes()` 统一重建，不应成为独立配置源。`config/` 与 `data/` 必须保持 `700`，但 `/etc/afs/subscriptions/` 必须为 `755`，否则 Nginx 工作进程无法读取 `alias` 订阅文件而返回 403。自适应订阅的 User-Agent 映射必须在 Clash/Mihomo 规则之前识别 Karing，并为 Karing 返回 Base64 URI 订阅，避免 Clash 转换丢失 Shadowsocks `mux=0`；显式 `/clash` 路由仍返回 YAML。Clash/Mihomo 的 WS 节点必须显式启用 UDP，并使用 Chrome 指纹和 `http/1.1` ALPN；原始/Base64 与 Sing-box 的 VLESS/VMess/Trojan WS 使用 Chrome 指纹但不强制 ALPN，交给 TLS 默认协商。VLESS、VMess 订阅保留 XUDP（`packetEncoding=xudp`、`packet-encoding: xudp`、`packet_encoding: "xudp"`）。XHTTP 链接参考 ArgoX 使用 `mode=auto`、Argo Host 与 `h2,http/1.1`；Shadowsocks 链接使用 `v2ray-plugin`、`mux=0` 和 UoT，`mux=0` 必须位于插件参数首位并由 `generate_nodes()` 校验全部三类订阅。Sing-box 的 WS inbound 不得额外启用 multiplex 或 padding；sing-box TLS 保持核心默认版本协商，WS `headers.Host` 必须是字符串。

双核心流量统计统一使用回环 V2Ray Stats API：Sing-box 必须通过 `experimental.v2ray_api` 统计全部节点 inbound 和有效 outbound，Xray 必须启用 `StatsService`、`stats` 与 system policy 的四项 inbound/outbound 计数。`afs-traffic.timer` 每分钟读取非重置计数并将差值持久化到 `/etc/afs/data/traffic.db`；基线必须区分核心、计数器和进程启动标识，计数回退或进程变化时从当前值重新累计。核心停止前必须补采，切换核心、服务重启和配置重建不得清空历史；重置必须先采集当前增量并更新基线。每节点粗略用量以共享 tag 的 inbound 计数为准，共享 direct/WARP outbound 不得伪装成节点级精确分摊。

## 关键函数职责

- `load_env()` / `save_env()`：读取和原子保存项目环境配置。
- `ensure_nodes_config()` / `validate_nodes_config()`：创建默认节点并校验动态节点数据。
- `write_sing_box_config()` / `write_xray_config()`：从同一份 `nodes.conf` 分别生成并校验五类节点的两套核心配置；`write_all_core_configs()` 用于首次安装，`write_available_core_configs()` 用于配置事务。两者都必须保持 `WARP → 节点 SOCKS5 → direct` 路由顺序。
- `write_nginx_config()`：生成本地 WS/XHTTP 反代和 UUID 订阅入口，并执行 `nginx -t`；XHTTP 必须使用边界安全前缀、HTTP/1.1、完整路径和关闭请求缓冲。
- `write_services()`：只写项目专属 systemd unit；包含 Token 的文件必须仅 root 可读。
- `traffic_collect()` / `traffic_statistics_menu()`：通过两内核共用的 V2Ray Stats API 采集增量、持久化 SQLite 并提供查看和重置；不得使用 Clash API 代替长期计数。
- `generate_nodes()`：从环境配置和 `nodes.conf` 生成全部节点及订阅文件。
- `apply_runtime_config()`：配置修改事务；验证失败必须恢复快照。核心切换模式只重启并验证 Nginx 与选中核心，避免 Argo Tunnel 的无关状态触发回滚。
- `sync_versions()`：核心版本比较、确认、暂存、校验、原子替换和失败回滚。
- `backup_project()` / `restore_project()`：仅归档与恢复节点配置 `nodes.conf`；解压前必须拒绝路径穿越、符号链接和特殊文件，恢复不得替换脚本、核心二进制或整个 `/etc/afs`。
- `uninstall_project()`：所有权检查后的项目范围卸载，不得扩大删除边界。

## 产品边界

保持以下范围：

- 仅支持 Debian/Ubuntu + systemd。
- 仅支持 amd64 和 arm64。
- 仅支持固定 Argo Token，不加入临时隧道、Argo JSON 或 Cloudflare API 建隧道。
- 支持 Sing-box 与 Xray 二选一；`af -p` 可在两者间切换，两个私有二进制与 `sing-box.json`、`xray.json` 可同时保留并从统一配置重建。双核心适配 VLESS/VMess/Trojan WS、VLESS XHTTP、Shadowsocks WS，不加入 Reality、Hysteria2 或其他协议。
- Sing-box 固定从 `Fiatnorm/argofusion-sing-box` 下载项目验证过的正式稳定 Release，校验 GitHub SHA256、下游版本与 `with_v2ray_api` 构建标签，不得回退到 SagerNet 通用构建或预发布/普通构建标签。
- 双核心统计 API 仅监听 `127.0.0.1`，默认端口 `18085` 必须与回源、节点和 WARP 端口互斥；不得对公网暴露或加入认证凭据。
- 新安装默认保留 VLESS、VMess、Trojan、VLESS XHTTP、Shadowsocks 各一个节点，并允许通过 `af -c` 动态添加、修改或删除这五类节点；升级已有安装不得自动向用户的 `nodes.conf` 插入新节点。
- 允许节点按 inbound tag 与传输路径绑定独立 SOCKS5 出站。
- 允许用户指定目标网址优先通过 Cloudflare 官方 WARP 客户端的本地 SOCKS5 proxy 出站；未命中时仍遵循节点 SOCKS5 或 direct。
- 首次安装和缺省环境配置的 Cloudflare 优选入口为 `bestcf.cdn.fiatnorm.us.kg:443`；用户仍可在安装或 `af -c` 中修改。
- 保持中文交互，不加入英文模式。
- 日常管理必须使用 VPS 本地安装脚本，不得改成每次执行都从 GitHub 拉取仓库脚本。
- BBR 不是项目内置能力，只能作为明确标注的第三方外部工具保留。

不要因为“对齐原版 SBA”而扩大产品范围。只对齐可靠性、回退、版本选择和运行语义。

## 命名规则

- 仓库入口必须使用 `argofusion.sh`，不要重新创建 `sba.sh`。
- 管理命令保持为 `af`，并提供大小写兼容入口 `AF`。
- systemd 服务保持为 `afs-core.service` 和 `afs-tunnel.service`。
- 核心二进制必须放在 `/etc/afs/bin/`，不得写入或删除 `/usr/local/bin/sing-box`、`/usr/local/bin/xray`、`/usr/local/bin/cloudflared`。
- 新增内部项目标识优先使用 `AFS`；`argofusion.*` 为保留的项目文件名，不得改名。

## 安全约束

安装、更新和卸载修改必须满足：

- 不得无条件删除整个 `/etc/afs`。
- 修改已有配置前必须保留备份，仅重建本项目管理的文件。
- 不得覆盖第三方同名 systemd 服务。只有带本项目 `Description=SBA ...` 或 `Description=ArgoFusion ...` 标记的旧服务可以迁移。
- 卸载必须检查 `/etc/afs/managed` 所有权标记。
- 卸载只删除本项目私有核心、服务、Nginx 配置、`af`/`AF` 链接和节点文件。
- 核心更新必须先比较版本并请求确认。
- 下载必须包含连接超时、总超时、重试、GitHub 反代回退和 SHA256 校验；直连失败后优先使用 `github-proxy.fiatnorm.pp.ua`。
- 更新流程必须遵循：下载到临时文件 → SHA256 与可执行性校验 → 所选内核配置检查 → 备份 → 原子替换 → 重启验证 → 失败回滚。
- 不得在脚本或示例文件中加入固定公共 UUID、Token 或未经项目明确指定的公共域名；当前约定的默认优选入口为 `bestcf.cdn.fiatnorm.us.kg:443`，且必须允许用户覆盖。
- 不得嵌入公共 WARP WireGuard 私钥、固定 WARP 账户或非官方 WARP 注册凭据；WARP 使用用户 VPS 上安装的 Cloudflare 官方客户端。
- Argo 域名必须由用户输入；首次安装 UUID 应自动随机生成。
- Argo Token 必须拒绝空白、换行和 systemd 控制字符；包含 Token 的配置和 unit 权限不得宽于 `600`。
- 恢复用户指定的归档前必须先检查 gzip、成员路径和文件类型，并使用 `--no-same-owner --no-same-permissions` 解压。

## 交互和运行语义

- 优选入口使用单行 `域名/IP:端口` 输入；IPv6 使用 `[地址]:端口`。
- `af -n` 必须先输出订阅面板链接，再按自适应、原始节点订阅、Base64、Clash/Mihomo、Sing-box 的顺序输出全部订阅链接、唯一一张自动适配订阅 QR 和明文节点；链接标签必须对齐，节点标题显示编号、标签和完整协议/传输/TLS 类型。不得为其他订阅或单个节点重复输出 QR。自动适配订阅 QR 同时显示在网页订阅面板中，并作为单独的 `/auto-qr.svg` 订阅面板资源提供。
- 节点连接地址使用优选入口，WS/XHTTP Host 与 TLS SNI 使用 Argo 域名。
- 修改 Token 或优选入口后，应重新生成节点并执行健康检查。
- 健康检查必须测试 `/etc/afs/config/nodes.conf` 中的全部传输路径：WS 执行公网 101 握手，XHTTP 执行公网 OPTIONS 200；新安装默认包括 `/argo-vl`、`/argo-vm`、`/argo-tr`、`/argo-xh`、`/argo-sh`。
- `af -c` 必须集中管理 Token、Argo 域名、优选入口、本地端口、UUID、动态节点、节点 SOCKS5 出站和 WARP 目标网址。
- WARP 域名规则必须位于节点 SOCKS5 规则之前，保持 `目标网址 WARP → 节点 SOCKS5 → direct` 的优先级。
- `af -x` 必须检查配置、Token、服务、动态端口、全部公网 WS/XHTTP 路径、核心版本、WARP（启用时）和最近日志。
- `af -t` 必须按“统计状态 → 入站统计 → 出站统计 → 统计操作”显示自上次重置以来的 SQLite 持久数据，并提供刷新和确认重置。入站按当前 `nodes.conf` 顺序展示每个节点；出站只按 direct、WARP、节点 SOCKS5 展示出口总量。
- `af -k` 必须打开节点配置备份与恢复菜单并校验 `/etc/afs/managed`；只备份和恢复 `/etc/afs/config/nodes.conf` 节点配置，恢复失败必须自动回滚，不得用旧归档覆盖当前脚本、核心或项目目录。子菜单不得另设短命令。
- 终端配色必须在非 TTY、`TERM=dumb` 或 `NO_COLOR` 环境自动关闭，不得向日志和管道写入 ANSI 控制符。
- 状态诊断保持简洁，并包含公网 IP、脚本/核心版本、内存、systemd 状态、监听端口和最近错误。
- 普通启停、查看节点、修改配置和卸载不得执行 `git pull` 或重新下载仓库脚本。
- `服务启停` 中选择“重启服务”后必须先执行 `systemctl daemon-reload`，再直接重启 Nginx、当前核心与 Argo Tunnel，不再进行二次确认；重启范围与处理提示之间不得插入额外横线。
- `af -i` 必须提供“本地重装”和“在线更新安装”两种模式；仅在线更新安装联网更新项目脚本，校验通过后必须先替换 `/etc/afs/argofusion.sh`，再切换到新版脚本进程继续安装，其他日常命令仍运行 VPS 本地脚本。

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
- `TERMINAL_UI_DESIGN.md` 是终端输出的视觉合同；更新终端 UI 时必须同步脚本、该设计稿、README 和 AGENTS.md。当前基准采用 ArgoFusion 斜体字标、64 列分隔线、ANSI `97` 亮白正文、`◆ / ▸ / ✓ / ! / ✗ / • / ›` 图标语义，以及 `main/back/cancel/default/none` 五种页面提示模式；提示固定在页面标题右端显示为“0 · 退出 / 返回 / 取消”或“Enter · 默认 | 0 · 取消”，其中 `Enter`、`0` 为亮黄、说明为亮白；显示宽度必须以 `C.UTF-8` 计算。输入统一为“对象 `[约束/默认值]`：”，已有配置只展示当前值，不逐项附加回车说明；新安装的必填项仍须拒绝留空。确认统一为“确认动作？`[Y/n]`：”，支持大小写且留空表示确认；诊断行必须明确展示“已通过 / 无效 / 未运行”。
- 分区和 64 列分隔线使用亮蓝，键名与字标使用亮青，页面标题与输入使用亮洋红。主菜单固定为“节点订阅、服务启停、核心切换、参数配置、流量统计、运行诊断、项目安装、组件更新、备份恢复、BBR / DD、项目卸载”，不得改回解释性长句；服务重启属于“服务启停”的子操作，不提供独立短命令；`menu_item()` 的功能列补齐到 28 个显示宽度。
- 主面板状态顺序固定为“Argo Tunnel、代理核心、WARP 分流、节点概览、Argo 域名、优选入口、Argo 回源、组件版本”；节点概览固定显示 `Vless n · Vmess n · Trojan n · XHTTP n · SS n`，不显示总数。只读页不显示 `0` 操作提示，普通返回静默，取消配置明确提示但不得写入半成品。所有交互子页面固定按“状态或说明 → 操作”分区。有现有值或默认值的表单统一使用 `Enter · 默认 | 0 · 取消`。核心切换页面和合并信息分区均命名为“核心配置”，操作后继续停留在该页并使用统一的处理中/成功文案，输入 `0` 才返回主面板。参数配置中的节点表固定使用 13/6/14/5/18 显示宽度并限制为 64 列，过长字段以 `~` 截断；协议显示为 Vless、Vmess、Trojan、XHTTP、SS 并使用亮蓝，直连与 SOCKS5 出站统一使用亮紫。仅“查看节点”在节点列表前留一行，增改删页面不留空行。节点增改删必须逐项校验，选择不存在的节点时原地重试。WARP 添加或删除域名留空时不得报错或应用配置。订阅中心保持白底蓝字，指定格式卡片顺序固定为“原始节点订阅、Base64、Clash/Mihomo、Sing-box”。
- `af -x` 必须诊断 `config`、`data` 的 `700` 与 `subscriptions` 的 `755`；健康时汇总最近日志，出现 ERROR 或诊断失败时才展开相关日志。
- Xray 配置检查成功时必须静默；失败或展示 Xray 日志时必须过滤配置读取行以及 WebSocket、VMess、Trojan 的已知弃用提醒，只保留实际运行过程与真实错误输出。
- UUID 订阅中心仍通过精确 `return 200` 路由提供；内联 HTML 的 UTF-8 单行长度必须低于 `3500` 字节，并由 `write_nginx_config()` 在 `nginx -t` 前强制检查，避免 Nginx 报出 `too long parameter`。
- 不要修改用户已有的无关文件或清理未跟踪的 `sba/` 对照树。
- 未在真实 VPS 上验证时，不得宣称 systemd、Nginx、Cloudflare 或公网 WS/XHTTP 已端到端通过。

## 版本与发布

Git 远端可能同时包含原版 SBA 和本项目仓库。发布前必须执行 `git remote -v`，确认目标为 `https://github.com/Fiatnorm/ArgoFusion.git`，不得把本项目改动推送到 `fscarmen/sba`。

持续测试分支固定为 `codex/argofusion-test`，不得再向名称中固化版本号的 `codex/test-directory-layout-v2.14.3` 推送新测试改动。每次推送 `codex/argofusion-test` 前，必须为该次推送递增并统一 `argofusion.sh` 的 `VERSION`、README 版本记录、终端设计稿适用版本和 `argofusion.sh.sha256`；不得把未标记版本的改动推送到该测试分支，便于逐次核算。

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
# Sing-box: /etc/afs/bin/sing-box check -c /etc/afs/config/sing-box.json
# Xray: /etc/afs/bin/xray run -test -c /etc/afs/config/xray.json
systemctl is-active nginx afs-core afs-tunnel
ss -lnt
journalctl -u afs-core -u afs-tunnel -n 100 --no-pager
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
