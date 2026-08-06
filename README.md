# ArgoFusion · AFS v2.15.2

面向固定 Argo Token 隧道的中文轻量安装脚本，提供：

- VLESS + WS + TLS：`/argo-vl`
- VMess + WS + TLS：`/argo-vm`
- Trojan + WS + TLS：`/argo-tr`
- VLESS + XHTTP + TLS：`/argo-xh`
- Shadowsocks + WS + TLS：`/argo-sh`
- 安装及 `af -p` 均可切换 Sing-box / Xray 内核，五类节点、订阅与 Argo 接口保持一致
- 自适应订阅、Base64 订阅、Clash/Mihomo 订阅、Sing-box 订阅、原始节点订阅五类入口

TLS 由 Cloudflare 边缘终止；VPS 本机 Nginx 与所选代理内核仅监听回环地址。固定隧道必须在 Cloudflare Zero Trust 添加 Public Hostname，Service 指向 `http://localhost:3010`。Public Hostname 域名必须由安装者输入；Cloudflare 优选入口默认使用 `bestcf.cdn.fiatnorm.us.kg:443`，也可在安装或配置时修改。

## 支持范围

仅支持 Debian/Ubuntu + systemd，以及 amd64/arm64。首次安装自动生成随机 UUID；也可在提示时自行替换。

## 安装

可在本地目录直接安装；安装后的 `af -i` 也可选择从 GitHub 获取脚本后在线更新安装。

```bash
chmod +x argofusion.sh
sudo ./argofusion.sh -i
```

安装时输入 Argo Token、Public Hostname，并以一行 `域名/IP:端口` 的形式输入 Cloudflare 优选入口，默认值为 `bestcf.cdn.fiatnorm.us.kg:443`。IPv6 使用 `[2001:db8::1]:443`。最后选择 `1` 使用默认 Sing-box，或选择 `2` 使用 Xray；安装后可在 `af` 主面板的“核心切换”随时切换，原有节点、订阅地址和 Argo 配置不变。Sing-box 固定使用 [Fiatnorm/argofusion-sing-box](https://github.com/Fiatnorm/argofusion-sing-box/releases) 下游内核，不再下载 SagerNet 官方通用构建。

核心安装在项目私有目录：

```text
/etc/afs/bin/sing-box
/etc/afs/bin/xray
/etc/afs/bin/cloudflared
/etc/afs/config/argofusion.env
/etc/afs/config/nodes.conf
/etc/afs/config/sing-box.json
/etc/afs/config/xray.json
/etc/afs/data/nodes.txt
/etc/afs/subscriptions/subscription.*
/etc/afs/backup/
```

项目核心、配置、节点和订阅数据统一保存在 `/etc/afs/`：`bin/` 为私有二进制，`config/` 为环境、节点和核心运行配置，`data/` 为明文节点，`subscriptions/` 为全部订阅产物，`backup/` 为节点备份。根目录仅保留项目脚本与所有权标记。升级安装会将带所有权标记的旧 `/etc/argofusion/` 或 `/etc/asb/` 迁入此目录，并在新服务验证成功后移除旧目录兼容链接；公网订阅 URL 不变。只有 systemd unit、Nginx 站点配置和 `/usr/local/bin/af`、`/usr/local/bin/AF` 命令入口按 Linux 系统约定保存在对应系统目录。服务使用 `afs-core.service` 和 `afs-tunnel.service`，不会覆盖系统已有的通用 `sing-box.service`、`xray.service` 或 `cloudflared.service`。

升级时仅在旧目录存在 `managed` 所有权标记且 `/etc/afs` 不存在时迁移；`/etc/argofusion` 与 `/etc/asb` 若同时为真实目录，或任一旧目录缺少所有权标记，脚本会停止并要求人工核对。迁移会临时保留旧目录到 `/etc/afs` 的兼容链接；新服务验证通过后才移除旧服务和兼容链接，失败则恢复旧服务。

迁移会先停止旧服务并等待节点端口释放，再启动新服务；若新服务启动失败，会先停用新服务再恢复旧服务，避免两套 sing-box 同时抢占节点端口。重新执行 v2.8.2 安装可修复旧版迁移失败后形成的新旧服务端口冲突。

v2.15.2 修复 Karing 通过自适应订阅导入 SS+WS 时 `mux=0` 丢失的问题。Karing 会发送由多种兼容标识组成的 User-Agent；旧映射先命中 Clash 并返回 YAML，Karing 在 Clash 转换中会丢弃 v2ray-plugin 的 `mux: false`。`/auto` 现在优先识别 Karing 并返回已经包含 `mux=0` 的 Base64 URI 订阅；显式 `/clash` 仍保持 Clash/Mihomo YAML。更新后应在 Karing 中删除旧的 `CL` 类型订阅并重新添加自适应链接，新的订阅应按 Base64/URI 导入。

v2.15.1 修复 Shadowsocks+WS 客户端转换后可能丢失 `mux=0` 的问题：原始/Base64、Clash/Mihomo 和 Sing-box 三类订阅均将禁用 v2ray-plugin multiplex 的字段置于插件参数首位，并在生成阶段强制检查。为进一步贴近 ArgoX，Sing-box 的 VLESS/VMess/Trojan WS 服务端不再额外启用带 padding 的 inbound multiplex；通用 URI 与 Sing-box WS 订阅不再强制 ALPN，由 TLS 默认协商，Clash/Mihomo 仍显式使用 `http/1.1`。early-data、Nginx 关闭缓冲与一小时时限、Cloudflared 参数及 XHTTP `mode=auto` 保持与 ArgoX 一致。实际延迟仍受 VPS、优选入口、Cloudflare 路由和测试时刻影响。

v2.15.0 将 Sing-box 下载源切换为 `Fiatnorm/argofusion-sing-box` 的固定稳定版 `v1.13.14-argofusion.2`，安装时同时校验 GitHub SHA256、下游版本和 `with_v2ray_api` 构建标签。新增 VLESS+XHTTP 与 Shadowsocks+WS 两类节点：新安装默认生成 `/argo-xh`、`/argo-sh`，既有安装保留原 `nodes.conf`，可从 `af -c` 添加；Sing-box/Xray 服务端配置、Nginx 分流、原始/Base64/Clash/Mihomo/Sing-box 订阅、节点列表和公网传输诊断同步支持。XHTTP 使用 ArgoX 的 CDN `mode=auto`、`h2,http/1.1` 与标准 VLESS URI，Nginx 使用边界安全前缀并关闭请求缓冲；Shadowsocks 使用 ArgoX 的 `chacha20-ietf-poly1305`、UUID 密码、`v2ray-plugin` 参数和 UoT，Sing-box 服务端的 UDP 经 UoT v2 承载。

v2.14.21 收紧节点与操作输出：主面板使用“WARP 分流”，节点概览仅显示 `Vless / Vmess / Trojan` 数量；`af -n` 节点标题改为 `标签 · Vmess+WS+TLS`。节点表恢复为不超过 64 列的 13/6/14/5/18 排版，过长字段以 `~` 截断，协议为亮蓝、出站 IP 为亮紫；仅“查看节点”在列表前留空行。核心切换操作后停留在核心配置页，处理/成功文案与服务操作统一。Xray 成功配置检查静默并过滤已知弃用提醒；服务重启先执行 `daemon-reload` 且去除范围后的多余横线。WARP 添加或删除域名留空时不修改、不报错，直接返回 WARP 操作页。

v2.14.20 扩充节点与交互信息：主面板新增按协议计数的节点概览，`af -n` 在标签后标注 `协议+WS+TLS`，节点表加宽并让直连节点显示 VPS 公网 IPv4；节点增改删改为逐项校验和当前步骤重试。所有确认统一为 `[Y/n]`，支持大小写且回车表示确认；服务重启选择后直接执行。表单标题统一显示 `Enter · 默认 | 0 · 取消`，输入项不再重复“按 Enter 保持”。Token 状态按已配置绿色、未配置黄色显示，诊断日志隐藏 Xray 的配置读取和已知协议弃用提醒。

v2.14.19 全局明确回车语义：已有配置的编辑表单标题右端统一显示 `Enter · 保持 | 0 · 取消`，备份/恢复等默认值表单显示 `Enter · 默认 | 0 · 取消`；回车分别保留当前值或使用默认值。主菜单仍为“核心切换”，其页面与合并信息分区统一命名为“核心配置”。功能、配置格式与操作语义保持不变。

v2.14.18 统一终端子页面与输入默认值：核心切换将代理状态、共享范围和保留文件合并为“核心状态与配置”单一分区；节点列表增加独立标题与间隔、保持 14/7/18/5/12 列对齐，并显示 SOCKS5 出站主机/IP:端口而不暴露凭据；运行诊断将组件版本收拢到“系统与组件状态”。所有可保留的已有配置均明确标注“按 Enter 保持”，备份与恢复等可取消表单统一显示 `0 · 取消`。功能、配置格式与操作语义保持不变。

v2.14.17 统一子页面结构：状态、配置范围或风险说明均以亮蓝分区呈现，操作菜单统一置于末段；核心切换现按“当前状态 / 配置范围 / 切换操作”显示，WARP、备份恢复、服务单项启停和服务重启同步采用同一层级。功能、配置格式与操作语义保持不变。

v2.14.16 调整终端层级与配置保留语义：横幅恢复 `Sing-box / Xray`，系统环境摘要左移并与版本信息对齐；代理核心按“运行中 · Xray”显示，WARP 本地代理地址保持亮白，仅公网 IP 与优选入口 IP 使用亮紫。可交互页面将全局 `0` 提示收敛到标题右端，其中 `0` 使用亮黄、说明使用亮白；`af -n` 将节点 URI 区命名为“原始节点”，编号后的标签使用亮紫。订阅中心保持白底蓝字，并将“原始节点订阅”置于指定格式首位。已存在的 Token、域名、优选入口和 UUID 在留空时保持当前值。

v2.14.15 统一终端输入、确认、检查结果和风险提示的排版：输入统一为“对象 `[约束/默认值]`：”，确认统一为“确认动作？`[y/N]`：”；页面提示加 `•` 图标，诊断明确输出“已通过 / 无效 / 未运行”。安装入口更名为“本地重装 / 在线更新”，在线更新附带校验与替换本地脚本的说明；BBR/DD 与卸载使用明确的“风险提示 / 将移除”分区。所有命令、配置格式与运行逻辑保持不变。

v2.14.14 将订阅入口收敛为自适应、Base64、Clash/Mihomo、Sing-box、原始节点链接五类，停止生成和公开 Clash Provider；订阅中心以白底蓝色卡片重新排版，并保留唯一的自适应 QR。`af -p` 的核心切换只重启并验证 Nginx 与目标代理核心，不再因无关的 Argo Tunnel 瞬时状态回滚；实际失败时会输出对应服务状态和最近日志。GitHub 下载直连失败时优先回退到 `github-proxy.fiatnorm.pp.ua`。

v2.14.13 按 `ArgoFusion_AFS_v2.14.12_TERMINAL_UI_DESIGN_v3.md` 重排终端 UI：主菜单统一为“节点订阅、服务启停、核心切换、参数配置、服务重启、运行诊断、项目安装、组件更新、备份恢复、BBR / DD、项目卸载”；状态区固定为 Argo Tunnel、代理核心、WARP、域名、优选入口、Argo 回源与版本；分区使用亮蓝、键名使用亮青，输入提示去除冗余“操作提示：”前缀。`af -n` 同步使用 VLESS、VMess、Trojan 的正确展示大小写。

v2.14.12 精简终端与订阅中心文案：主面板统一使用“节点与订阅、配置管理、重启服务、运行诊断”等短名称；安装方式、服务开关、组件更新和订阅入口同步采用清晰一致的提示语，快捷命令与功能不变。

v2.14.11 合并主面板的服务开关与节点配置备份/恢复：`af -a` 打开服务管理，`af -k` 打开节点配置备份与恢复；子菜单不再拥有独立短命令。切换代理核心新增主页面短命令 `af -p`。

v2.14.10 将运行命名空间迁移为 AFS：项目目录改为 `/etc/afs/`，服务改为 `afs-core.service` 与 `afs-tunnel.service`，保留 ArgoFusion 品牌及 `argofusion.*` 文件名；`af` 与 `AF` 均可作为管理命令。已有受管理的 `/etc/argofusion/` 或 `/etc/asb/` 安装会在更新时安全迁移并在新服务验证后清理旧兼容链接。

v2.14.9 修复订阅中心导致 Nginx 配置校验失败的问题：保留 UUID 精确 `return 200` 路由，但将内联页面压缩到 Nginx 单参数长度限制内，不再触发“too long parameter / missing terminating quote”。

v2.14.8 将持续测试入口固定为 `codex/argofusion-test`，不再使用名称中固化旧版本的测试分支；每次推送该分支均同步递增脚本、README、设计稿与校验哈希版本，便于核算。

v2.14.7 根据 `ArgoFusion_TERMINAL_UI_DESIGN_v2.14.5.md` 统一终端输出：首页改用 ArgoFusion 斜体字标与亮白正文；页面提示按主面板、返回、取消和只读场景显示；主面板状态统一为 Argo Tunnel 与代理核心；节点表采用 64 列布局并隐藏 SOCKS5 凭据；诊断检查 `config/data` 的 `700` 与 `subscriptions` 的 `755`，健康时仅汇总最近日志。

v2.14.6 合并重复的 Shadowrocket 订阅：Shadowrocket 与 V2rayN、NekoBox 共用 `/base64` 通用订阅，停止生成和暴露独立 `/shadowrocket` 文件与 URL。`/raw` 统一命名为“原始订阅”；网页订阅中心重排为推荐自动适配入口与指定格式订阅两层，并改善窄屏布局。

v2.14.5 修复重整目录后的订阅 403：`/etc/argofusion/subscriptions/` 由 Nginx 通过 `alias` 对外提供，目录权限改为 `755`，使 Nginx 工作进程可以读取订阅、二维码和自动适配文件；`config/`、`data/` 仍保持 `700` 私有权限。重新安装或更新安装会自动修正已有目录权限。

v2.14.3 重整 VPS 项目目录：环境、节点定义与两套核心 JSON 迁入 `/etc/argofusion/config/`，明文节点迁入 `/etc/argofusion/data/`，各类订阅和自动适配 QR 迁入 `/etc/argofusion/subscriptions/`。升级安装会先核对同名文件内容，拒绝覆盖冲突或异常文件类型，再迁移并重建 Nginx、核心服务与订阅；外部订阅 URL 保持不变。

v2.14.4 优化终端输出层级与排版：页面标题下统一说明 `0` 的返回/退出规则，移除子页面和确认提示中的重复文字；主面板将“切换代理核心”提升为独立日常操作，集中配置只保留配置、节点与 WARP 分流。同步将终端设计稿、菜单和说明统一为 ArgoFusion 名称与 `af` 命令。

v2.14.2 修复 GitHub Release 压缩 JSON 的 SHA256 解析。Cloudflared、Sing-box 与 Xray 仍必须通过官方 Release 发布的 SHA256 校验；缺少或不匹配时继续拒绝安装。

v2.14.1 将节点与订阅派生文件的生成提前到服务启动之前；即使后续服务同步或健康检查中断，`nodes.txt` 和 `subscription.*` 也不会缺失。若固定隧道日志纠正了 Argo 域名，脚本会重新生成节点和订阅，保证连接域名一致。

v2.14.0 完成 ArgoFusion 本地命名迁移：安装入口为 `argofusion.sh`，配置目录为 `/etc/argofusion/`，服务为 `argofusion-core.service` 与 `argofusion-tunnel.service`，Nginx 配置为 `argofusion.conf`，短命令为 `af`。升级时仅迁移带所有权标记的 Argo-Singbox 安装；不依赖或修改原版 SBA 对照树。移除了三个未调用的函数，并收紧旧命令清理为仅删除符号链接。

v2.8.3 在用户启用 WARP 且系统缺少 `warp-cli` 时，可自动配置 Cloudflare 官方 APT 软件源并安装客户端。

v2.8.4 修正 WARP 注册检测：读取现有注册时自动接受使用条款；若客户端报告残留的无效旧注册，会在用户确认后删除并重新注册。

v2.9.0 增加按客户端区分的订阅文件和自动适配入口，覆盖 V2rayN/NekoBox、Clash/Mihomo、sing-box 与 Shadowrocket；保留逐行明文节点协议文件。节点修改现在可以直接修改标签、协议、WS 路径、监听端口和节点 SOCKS5。WARP 菜单支持单独添加、删除或整体修改目标域名。

v2.9.1 修复从交互菜单卸载后再次进入面板的问题。卸载会清理项目核心、配置、订阅、备份、服务和命令入口，并分别询问是否额外卸载 Nginx、Cloudflare WARP 及通用系统工具。

v2.10.0 首次安装固定使用 sing-box `1.13.0-rc.4`，cloudflared 与原版 SBA 一样下载 latest；固定 Token 日志未输出 hostname 时不再误报，公网 WS 探测超时也不再阻断安装。新增 UUID 文件索引、可输入备份文件夹、WARP 添加前展示已有域名，并调整节点表格和终端输出间距。

v2.10.1 修复 UUID 配置索引的 Nginx 500 错误；节点文件迁入 `/etc/argofusion/nodes.txt`；备份和恢复默认使用 `/etc/argofusion/backup/`；Argo/cloudflared 与 Sing-box 核心改为分别询问是否更新；运行检查、完成信息和明文节点之间增加分区空行。

v2.10.2 调整面板为高亮蓝、青和白色主视觉，明文节点块末尾固定保留一行；`af -i` 会先获取并校验 `Fiatnorm/ArgoFusion` 的最新 `main` 脚本，再由最新脚本继续安装或更新。

v2.10.3 修正安装完成后的明文节点块末尾空行，并将 `af -i` 拆分为“当前 VPS 本地脚本重装”和“GitHub 最新脚本安装”两种明确模式。最终面板配色使用亮紫品牌、亮蓝/亮青分区、亮黄菜单序号、白色功能名及绿/黄/红状态色，增强层级和区分度。

v2.10.4 将节点与订阅索引中的 URL 统一改为白色，保留亮蓝色键名，提升链接区域的可读性。

v2.10.5 对现有功能进行可靠性与安全加固：环境配置改为同目录临时文件原子写入并安全转义，Token 增加字符集校验，包含 Token 的 systemd unit 收紧为仅 root 可读；恢复前拒绝项目目录之外的路径、符号链接和特殊文件，并禁止继承归档所有者与权限；修复 IPv6 优选入口在 VLESS/Trojan URI 与 Clash YAML 中的格式、带前导零端口的解析及节点标签删除的精确匹配；监听端口检查改用 `ss` 的端口过滤器，避免文本匹配误判。

v2.10.6 将 `af -n` 的 QR 二维码扩展到自动适配、明文节点、Base64、Clash/Mihomo、Clash Provider、sing-box 和 Shadowrocket 全部订阅入口；配置索引页改为清晰的客户端卡片布局；终端菜单、配置页和诊断页统一命名，并优化为亮紫品牌、蓝色分区、青色键名、黄色订阅标签和亮白内容的高对比配色。

v2.10.7 将 QR 二维码从终端输出移入网页订阅面板，仅保留自动适配订阅 QR；`af -n` 改为只输出订阅链接和明文节点；网页面板调整为白底蓝字的简约布局，终端输出命名统一为“订阅链接 / 明文节点”，配色调整为亮蓝品牌、亮青键名和亮白内容。

v2.10.8 修复 `af -i` 在线更新安装后仍回到旧脚本进程的问题：GitHub 模式会先原子替换 VPS 本地脚本，再切换到新版继续安装。终端输出改为紧凑分区格式，统一使用短菜单名、`[OK] / [WARN] / [ERR] / [INFO]` 状态前缀，以及蓝、青、紫、黄、绿、红的清晰配色。

v2.10.9 对最终控制面板和订阅面板做 UI 优化：控制面板标题固定为 `ArgoFusion v版本号`，不显示系统发行版代号；网页订阅面板显示 `ArgoFusion 订阅面板`，使用更清晰的推荐、Raw、Base64、YAML、Provider、JSON、iOS 标签；终端菜单、配置页、诊断页和订阅输出进一步压缩文字、强化蓝/青/紫/黄/绿/红的层级区分。

v2.11.0 按 CLI Design System 重排终端 UI：新增统一 `ui_*` 输出层，标题改为 `ArgoFusion  v版本号` + 动态分隔线；主菜单直接显示 Argo、sing-box、WARP 状态和当前域名/优选入口；配置中心、订阅与节点、核心更新、备份、恢复和卸载页统一分区、提示与短文案；长订阅 URL 不再下划线；普通输入统一使用 `>`；诊断页改为概览、配置、服务、版本、网络和结果分区，正常通过时不再默认输出大量 journal 日志。

v2.11.1 将终端 UI、交互文案和文字配色恢复为 v2.10.5 风格，订阅面板恢复经典白底蓝字布局；`af -n` 仅为自动适配订阅输出一张 QR，不再为每个明文节点重复生成二维码。同时保留在线更新安装先替换 VPS 本地脚本、再切换到新版进程继续安装的修复。

v2.11.2 修复 Windows Git Bash 生成的 SHA256 清单使用 `*文件名` 标记时，在线安装无法读取预期哈希并反复校验失败的问题；安装器现在同时兼容文本模式与二进制模式清单，并隐藏正常下载进度条。

v2.11.3 优化安装与启动终端首页：加入紧凑品牌字标、项目能力摘要和系统环境行；主菜单补充服务、域名与优选入口概览；安装结束统一展示运行摘要，保持非 TTY、`TERM=dumb` 和 `NO_COLOR` 环境无 ANSI 控制符。

v2.11.4 恢复 v2.10.8 风格的白底蓝字订阅面板及自动适配订阅 QR；修复 Debian `/etc/os-release` 覆盖脚本版本号的问题；启动概览增加组件版本、Argo 回源和 WARP 状态，并统一全脚本交互提示与颜色层级。`af -n` 在终端保留一张自动适配订阅 QR，网页面板同时通过 `/auto-qr.svg` 展示。

v2.11.5 按 `TERMINAL_UI_DESIGN.md` 统一终端输出：更新启动字标、66 列分隔线、运行概览、菜单和订阅索引列宽，停用服务使用黄色提示，公网 WS 探测超时使用黄色警告。备份与恢复改为只处理 `/etc/argofusion/nodes.conf` 节点配置；恢复旧完整归档时也只提取节点配置，不替换脚本、核心或整个 `/etc/argofusion`，避免新版本降级。

v2.11.6 重新按 `TERMINAL_UI_DESIGN.md` 收敛终端 UI：统一 64 列分隔线、显示宽度对齐、青色下划线订阅链接、蓝色概览标题和独立操作页面标题；备份、恢复、核心更新、重启、BBR 和卸载都补充上下文分区与处理状态。`TERMINAL_UI_DESIGN.md` 中的节点、域名、UUID、IP、SOCKS5 和日志示例已全部替换为占位内容，避免提交真实节点配置。

v2.11.7 根据测试清单继续优化终端 UI：系统 IP 使用紫色、运行分区标题统一为亮青色、组件名统一显示 `ArgoFusion`，订阅索引链接改为白色，WARP 未启用状态使用黄色；各子操作支持返回上级界面。

v2.11.8 收紧终端返回交互为仅输入 `0`；配置文件索引删除标题前空行并在订阅列表末尾增加白色分隔线。所有显示的优选入口 IP、公网 IP 与本机 IP 统一使用亮紫色。

v2.11.9 配置文件索引中的每条订阅链接下方均显示白色分隔线，便于逐项辨识。

v2.13.5 将 VMess 的全部客户端订阅输出统一为 `aes-128-gcm`：Base64 分享链接为 `scy`、Clash/Mihomo 为 `cipher`、sing-box 为 `security`。该具体算法可同时被 Xray 与 Sing-box 的 VMess 入站正确解码，避免部分客户端把 `auto` 直接写入 VMess 握手而被 Xray 拒绝。Base64 链接中的 `host`、`alpn` 均为字符串；sing-box JSON 的 `headers.Host` 保持字符串。`alpn` 则按 sing-box 与 Clash/Mihomo 的标准 schema 保持单元素数组，客户端导入后显示为数组属于其配置规范化，不会改变订阅源的 Host 字段。

v2.13.4 将 Sing-box 的离线回退版本更新到官方最新稳定版 `1.13.14`；联网时仍优先从官方 Releases 查询最新版本。sing-box 订阅生成的 WS `headers.Host` 保持为动态 Argo 域名的字符串；应以订阅源 JSON 为准，不以客户端导入后的显示类型判断生成字段。

v2.13.3 为订阅中的 VLESS、VMess、Trojan 统一设置 Chrome 指纹与 `http/1.1` ALPN；VMess 同时输出 XUDP（`packetEncoding=xudp` / `packet-encoding: xudp` / `packet_encoding: "xudp"`）。sing-box 不再限制 TLS 版本，交由客户端与核心使用默认协商范围；WS `headers.Host` 保持为动态 Argo 域名的字符串而不是数组。

v2.13.2 对齐 ArgoX 的 Xray WS 入站与 Nginx 长连接代理：VLESS 使用 `level: 0`、`decryption: none`，VMess 保持默认无传输安全层，VLESS/Trojan 显式 `security: none`，三个协议继续启用完整 sniffing；WS 反代统一关闭缓冲并设置 1 小时读写超时。订阅对三种协议保持 UDP 与 WebSocket 0-RTT 参数，VLESS 同时输出 XUDP（`packetEncoding=xudp` / `packet-encoding: xudp` / `packet_encoding: "xudp"`）。

v2.13.1 将内核选择加入 `af -c` 集中配置。首次安装同时保存项目私有的 Sing-box、Xray 二进制及 `/etc/argofusion/sing-box.json`、`/etc/argofusion/xray.json`；两个 JSON 都从同一份 `argofusion.env` 与 `nodes.conf` 生成并校验。旧安装首次切换到缺失核心时会下载并校验该核心，然后仅切换 `argofusion-core.service` 的启动目标，不改变 Argo、Nginx、节点或订阅入口。

v2.13.0 增加 Xray 内核适配。安装时可在 Sing-box 与 Xray 间选择，VLESS、VMess、Trojan 的 WS + TLS 节点继续共用 `nodes.conf`、Nginx 反代、固定 Argo Token 隧道和全部订阅入口。Xray 同样按 `WARP 目标域名 → 节点 SOCKS5 → direct` 生成路由，下载、配置检查、更新和回滚沿用项目的 SHA256、原子替换与项目专属服务边界；不引入 Reality、临时隧道或内置 WARP 凭据。
v2.12.2 将首次安装和缺省环境配置中的 Cloudflare 优选入口统一为 `bestcf.cdn.fiatnorm.us.kg:443`，同时保留安装及集中配置时的自定义入口能力；同步更新 README、维护约束和终端 UI 示例。
v2.12.1 在 v2.12.0 的 UI 收敛基础上补强配置事务：派生节点/订阅文件纳入回滚快照，订阅生成或服务验证失败时恢复完整运行面，并集中校验已有环境文件；状态行对 IPv6 优选入口统一显示为 `[地址]:端口`。同时保持 64 列分隔线、白色下划线 URL、唯一自动适配 QR 和节点专用备份边界不变。
v2.12.0 配置文件索引改为白色下划线 URL，不再输出整行链接分隔线；所有页面的分隔线后一级小标题均紧贴显示。Argo 回源地址保持白色，本机公网 IP 与优选入口 IP 保持亮紫色。

下载具有总超时、重试、GitHub 反代回退和 GitHub Release SHA256 digest 校验；直连失败时优先使用 `github-proxy.fiatnorm.pp.ua`，随后才尝试其他反代。二进制还会执行版本和构建能力检查。Sing-box 固定为 ArgoFusion 下游稳定发布，Xray 与 cloudflared 固定为项目验证过的官方稳定发布，组件更新不会自动切换到预发布版本或普通构建标签。

当前固定版本为 ArgoFusion Sing-box `1.13.14-argofusion.2`、Xray `26.3.27`、cloudflared `2026.7.3`；系统依赖继续从当前 Debian/Ubuntu 的官方稳定 APT 源安装，以保持发行版兼容性。后续执行 `af -v` 时，脚本只比较和更新当前选择的代理内核及 cloudflared 至上述固定稳定版本。

## 是否需要反复拉取 GitHub

安装完成后，脚本会保存在 `/etc/afs/argofusion.sh`，并建立等价的本地命令 `/usr/local/bin/af` 与 `/usr/local/bin/AF`。查看节点、修改 Token/优选入口、启停或重启服务、查看状态和卸载都直接使用 VPS 上的本地文件，不会重新拉取仓库。`af -i` 可从 `Fiatnorm/ArgoFusion` 获取、校验并原子替换最新脚本。

以下操作仍会主动访问网络：

- 首次安装或再次执行“安装 / 更新”：获取并校验最新 ArgoFusion 脚本，再下载并校验 ArgoFusion Sing-box 下游发布物、Xray 与 cloudflared 官方发布物。
- `af -v`：比较固定的稳定组件版本，有更新并确认后下载核心。
- 健康检查：访问 Cloudflare 公网入口。
- 状态诊断：尝试访问 `api.ipify.org` 获取公网 IP，失败时自动使用本机地址。
- `af -b`：明确执行第三方的内核升级、BBR 和 DD 系统脚本。

因此，后续管理不需要 `git pull`，但 Argo 隧道本身及核心在线更新显然仍需要 VPS 能访问互联网。

## 完整运行指令

首次从仓库运行：

```bash
cd ArgoFusion
chmod +x argofusion.sh
sudo ./argofusion.sh
```

直接执行安装或更新：

```bash
sudo ./argofusion.sh -i
```

安装完成后统一使用 VPS 本地命令 `af`（`AF` 等价），不需要进入仓库目录：

| 指令 | 功能 |
|---|---|
| `sudo af` | 打开完整中文管理面板 |
| `sudo AF` | 与 `sudo af` 完全等价 |
| `sudo af -i` | 选择使用 VPS 本地脚本重装，或从 GitHub 获取最新脚本后安装 |
| `sudo af -n` | 显示全部节点、所有订阅地址及一张自动适配订阅 QR |
| `sudo af -a` | 打开服务启停，切换 Argo Tunnel、代理核心状态或重启全部服务 |
| `sudo af -p` | 打开核心切换，切换 Sing-box / Xray 代理核心 |
| `sudo af -c` | 修改 Token、域名、优选入口、端口、UUID、节点、SOCKS5 与 WARP 域名 |
| `sudo af -x` | 执行运行诊断、WS 检查并显示最近日志 |
| `sudo af -v` | 比较版本并更新 Argo/cloudflared 与当前选择的代理内核 |
| `sudo af -k` | 打开节点备份与恢复菜单 |
| `sudo af -b` | 启动第三方 Linux-NetSpeed BBR/DD 工具 |
| `sudo af -u` | 彻底卸载本项目，并选择是否卸载共享依赖 |

所有操作都要求 root 权限。也可以在仓库中把 `af` 替换为 `./argofusion.sh` 执行相同参数，但日常管理应使用安装到 `/etc/afs/argofusion.sh` 的本地 `af` 或 `AF` 命令。

不支持组合参数，也没有后台静默卸载参数。`-u` 会要求确认，避免误删正在使用的 Nginx、WARP 或通用系统工具。

## 菜单

```text
1. 节点订阅 (af -n)
2. 服务启停 (af -a)
3. 核心切换 (af -p)
4. 参数配置 (af -c)
5. 运行诊断 (af -x)
6. 项目安装 (af -i)
7. 组件更新 (af -v)
8. 备份恢复 (af -k)
9. BBR / DD (af -b)
10. 项目卸载 (af -u)
0. 退出脚本
```

## 参数配置与分流

`af -c` 用于修改 Token、Argo 域名、优选入口、Argo Tunnel 回源端口和全局 UUID，也可以添加、修改或删除 `vless`、`vmess`、`trojan`、`vless-xhttp`、`shadowsocks` 节点。前三类和 Shadowsocks 使用 WS，`vless-xhttp` 使用 XHTTP；公网均由 Cloudflare 提供 TLS。修改 Tunnel 回源端口时，节点监听端口从“回源端口 + 1”开始依次顺延；添加节点时默认使用当前最大监听端口的下一个端口。节点增改删会在每个输入步骤立即校验，格式、重复值或标签不存在时只要求重试当前步骤，不退出脚本。配置保存在 `/etc/afs/config/nodes.conf`。修改后还必须在 Cloudflare Public Hostname 中把 Service 同步为新的 `http://localhost:端口`。

### 切换 Sing-box / Xray

在 `af` 主面板选择“核心切换”，输入 `1` 为 Sing-box、`2` 为 Xray。两种核心共用 `/etc/afs/config/argofusion.env`、`/etc/afs/config/nodes.conf`、Nginx、Argo Tunnel 和订阅文件；`/etc/afs/config/sing-box.json` 与 `/etc/afs/config/xray.json` 始终分别保留。切换会先检查目标二进制，必要时从对应 Release 下载、校验并原子安装，然后同时重建和校验两套 JSON，最后重启同一个 `afs-core.service`。失败会恢复切换前的环境、运行配置、服务文件和订阅文件。

添加节点时可留空使用直连，也可输入 SOCKS5 出站：

```text
203.0.113.10:1080:proxyuser:proxypass
```

路由按节点 inbound tag 匹配，因此同一种协议的不同传输路径可以使用不同出口。SOCKS5 地址、端口、用户名和密码只写入权限为 `600` 的项目配置；节点分享链接不包含出站凭据。配置变更会重建所有已安装内核的配置并逐一检查、执行 `nginx -t`、重启服务和状态验证，失败时恢复修改前文件。

主面板、可返回菜单与确认/配置表单分别在标题右端显示“`0 · 退出`”“`0 · 返回`”“`0 · 取消`”，只读页不显示提示；标题过长时提示自动换行。基础项、节点操作和 WARP 子菜单均保留该行为，返回时不会写入半成品配置。

### 按网址优先使用 WARP

`af -c` 的 WARP 入口使用 Cloudflare 官方 Linux 客户端的本地 SOCKS5 proxy 模式。菜单可启用或整体修改配置，也可单独添加、删除目标域名；删除最后一个域名时必须明确停用 WARP，避免生成空规则。启用时若系统尚未安装 `warp-cli`，脚本会征得确认后自动配置 Cloudflare 官方 APT 软件源，校验软件源签名密钥，并安装 `cloudflare-warp`。软件源及系统支持范围见 [Cloudflare 官方 Linux 安装说明](https://developers.cloudflare.com/warp-client/get-started/linux/)。输入以逗号分隔的网址或域名，例如：

```text
https://chatgpt.com,api.openai.com,example.com
```

脚本会提取并保存域名、启动 `warp-svc`、注册客户端、切换本地 proxy 模式，再为当前内核生成优先级明确的路由：

代理端口提示只能输入数字或直接回车采用默认值；目标网址应在下一条提示中输入。若客户端存在无法读取的旧注册，脚本会先询问是否删除并重新注册，不会静默覆盖现有注册。

```text
目标域名命中 WARP → WARP
其他流量 → 节点配置的 SOCKS5
没有节点 SOCKS5 → direct
```

WARP 只覆盖匹配的网址，不会替换其他节点的 SOCKS5 配置。`af -x` 会检查 `warp-svc`、本地代理端口，并通过 WARP 访问第一个目标域名。WARP 不提供匿名保证，也不保证指定国家或地区的落地 IP。

终端输出使用高亮配色：亮青字标和键名、亮紫页面标题、输入提示、当前核心、优选入口 IP 与公网 IP，亮蓝分区、分隔线及节点协议，亮黄色菜单序号、停用状态及标题右端的导航 `0`，亮白承载主要内容，绿/黄/红分别表示成功、警告和错误。Token 已配置为绿色、未配置为黄色。`af -n` 的“原始节点”标题按节点显示 `Vless/Vmess/Trojan+WS+TLS`、`Vless+XHTTP+TLS` 或 `Shadowsocks+WS+TLS`；主面板状态键为“WARP 分流”，节点概览固定显示 `Vless / Vmess / Trojan / XHTTP / SS` 数量而不显示总数。节点表按 13/6/14/5/18 排版且总宽度不超过 64 列，过长字段以 `~` 截断；XHTTP 与 Shadowsocks 在六列协议栏显示为 `XHTTP`、`SS`，直连公网 IPv4 与 SOCKS5 主机端口统一为亮紫。仅参数配置中的“查看节点”在列表前留一行，增改删页面不留空行。核心切换成功后停留在核心配置页，输入 `0` 才返回；服务重启先刷新 systemd unit，并去掉重启范围后的额外横线。Xray 成功配置检查不输出配置读取和弃用提醒，失败只显示过滤后的真实错误。WARP 添加、删除域名留空时直接返回，不执行配置事务。

## 诊断、备份与恢复

- `af -x`：执行运行诊断，检查配置与 Token 同步、三个服务、全部动态监听端口、每条公网 WS 握手或 XHTTP OPTIONS 路径、核心版本，并输出最近 30 条项目日志。
- `af -k`：打开节点备份与恢复菜单。备份仅归档 `/etc/afs/config/nodes.conf`；默认保存到 `/etc/afs/backup/argofusion-nodes-backup-时间.tar.gz`，也可在子菜单指定其他绝对路径。恢复默认从 `/etc/afs/backup/` 选择最新节点归档，也可在子菜单指定目录或完整文件；解压前会验证 gzip、成员路径与文件类型，拒绝目录穿越、符号链接和特殊文件，失败自动回滚。传入旧版完整 `/etc/argofusion` 归档时，也只读取其中的 `nodes.conf`，不会恢复旧脚本、旧核心或整个项目目录。

`af -v` 会分别显示 Argo/cloudflared 与当前选择的 Sing-box 或 Xray 的本地、目标版本，并分别询问是否更新。只下载、备份、替换和重启用户确认更新的核心；下载文件会校验 SHA256/可执行性，并在替换前检查当前配置。验证失败时只回滚本次选择的核心。

核心更新的备份只用于本次回滚，验证成功后立即删除；失败时保留，便于核对和恢复。

`af -b` 调用 `ylx2016/Linux-NetSpeed` 外部远程脚本，提供内核升级、BBR 和 DD 系统入口。该工具不属于本项目，执行前会明确提示。

## 节点、订阅和检查

`af -n` 先输出订阅面板链接，再按自适应、原始节点订阅、Base64、Clash/Mihomo、Sing-box 的顺序输出五类入口、自动适配订阅 QR 和全部原始节点 URI。链接标签统一按固定显示宽度对齐；节点标题显示编号、标签及完整协议/传输/TLS 类型，不重复显示传输路径。浏览器访问 `https://你的域名/你的UUID/` 可进入白底蓝字订阅中心，扫描同一自动适配订阅 QR，或打开不同客户端配置：

```text
https://你的域名/你的UUID/
https://你的域名/你的UUID/auto
https://你的域名/你的UUID/raw
https://你的域名/你的UUID/base64
https://你的域名/你的UUID/clash
https://你的域名/你的UUID/sing-box
```

`/你的UUID` 会跳转到文件索引；索引 HTML 由 Nginx 直接返回，避免目录 URL 使用文件 `alias` 导致 500。`/auto` 根据 User-Agent 为 Clash/Mihomo 与 sing-box 返回对应格式，其他客户端（包括 Shadowrocket）返回 Base64 通用订阅。`/raw`（原始节点订阅）以及 `/etc/afs/data/nodes.txt` 保留逐行明文 `vless://`、`vmess://`、`trojan://`、XHTTP `vless://` 与 Shadowsocks `ss://` 节点协议；不再生成或公开 Clash Provider。旧的 `/argofusion-sub` 与 `/argofusion-sub-base64` 入口继续可用。

网页自动适配订阅 QR 由 `generate_nodes()` 生成，并通过订阅面板的 `/你的UUID/auto-qr.svg` 资源展示；终端仅输出这一张自动适配 QR，不为明文节点或其他独立配置重复生成二维码。

Clash/Mihomo 的 WS 节点均显式 `udp: true`，VLESS、VMess、Trojan 使用 Chrome 指纹与 `http/1.1` ALPN；VLESS、VMess 同时带 `packet-encoding: xudp`。VMess 统一使用 `aes-128-gcm`（Base64 的 `scy`、Clash/Mihomo 的 `cipher`、sing-box 的 `security`）；明文 VLESS 链接带 `packetEncoding=xudp`，兼容该扩展字段的 Base64 VMess 分享链接也带 `packetEncoding=xudp`，sing-box 的 VLESS、VMess 出站均带 `"packet_encoding":"xudp"`。XHTTP 参考 ArgoX 固定使用 `mode=auto`、Host 为 Argo 域名、ALPN 为 `h2,http/1.1`；其 Nginx 路由保留完整子路径并关闭响应/请求缓冲。Shadowsocks 固定使用 `chacha20-ietf-poly1305`，密码复用全局 UUID，客户端通过 `v2ray-plugin` 的 WebSocket+TLS 参数连接；Sing-box 订阅显式启用 UoT v2。sing-box TLS 不设置 `min_version` 或 `max_version`，由核心默认协商版本范围；WS `headers.Host` 为动态 Argo 域名的字符串。服务端本机链路由 Cloudflare 边缘终止 TLS，故 Xray / Sing-box 入站继续为回环地址上的明文 WS/XHTTP。

新安装默认标签为 `Argo-Vl`、`Argo-Vm`、`Argo-Tr`、`Argo-Xh`、`Argo-Sh`。升级已有安装时不会自动向用户维护的 `nodes.conf` 插入新节点；可通过 `af -c` 添加 `vless-xhttp` 或 `shadowsocks`。标签同时作为当前内核 inbound tag 和各客户端显示名称。

如 Cloudflare 返回 Challenge/WAF，需为全部动态代理路径和订阅路径建立适当的 Skip 规则。不要把 Public Hostname 手工解析到 VPS IP；应让流量经过 Argo Tunnel。

固定 Token 模式下 cloudflared 日志不保证包含 Public Hostname，因此脚本无法从日志读取 hostname 时会保留用户输入域名，不再显示失败提醒。公网 WS 握手与 XHTTP OPTIONS 探测经过优选入口，单次超时只显示非阻断提示；明确的 HTTP 错误、Cloudflare Challenge 或本地服务/端口异常仍会使健康检查失败。最终连通性应以客户端实测为准。

## 卸载边界

`sudo af -u` 要求 `/etc/afs/managed` 所有权标记存在，并再次确认卸载。确认后固定删除：

- `/etc/afs` 中的项目核心、配置、订阅和项目备份；
- `afs-core.service`、`afs-tunnel.service` 以及确认属于本项目的旧服务；
- `/etc/nginx/conf.d/argofusion.conf`、项目旧 Nginx 配置、`/usr/local/bin/af` 与 `/usr/local/bin/AF`；
- `/etc/afs/data/nodes.txt`、旧版 `/root` 节点文件、项目迁移链接和兼容节点文件。

私有 `/etc/afs/bin/cloudflared`（Argo）、`/etc/afs/bin/sing-box` 和 `/etc/afs/bin/xray` 一定随项目删除。卸载完成后脚本立即退出，不会重新显示管理面板。

Nginx、Cloudflare WARP 和 `curl/ca-certificates/openssl/tar/unzip/qrencode/gnupg` 可能被其他网站或脚本共用，因此分别询问且默认不卸载；明确输入 `y` 后使用 APT purge。选择卸载 WARP 时还会断开连接、删除注册、停止 `warp-svc`，并删除本脚本配置的 Cloudflare APT 软件源和密钥。选择保留 Nginx 时只删除本项目站点配置并重启 Nginx。

只有确认 `/etc/afs/managed` 项目所有权标记后，脚本才递归删除整个 `/etc/afs`，因此项目备份、旧版迁移文件或历史订阅不会残留。请勿把个人文件放入该项目私有目录。脚本不会删除 `/usr/local/bin/sing-box`、`/usr/local/bin/xray`、`/usr/local/bin/cloudflared` 或非本项目 systemd 服务。

本项目不包含 Reality、临时隧道、Argo Json、Cloudflare API 建隧道、英文界面或其他协议脚本；双核心仅适配 VLESS/VMess/Trojan WS、VLESS XHTTP 和 Shadowsocks WS 这五类固定隧道节点。
