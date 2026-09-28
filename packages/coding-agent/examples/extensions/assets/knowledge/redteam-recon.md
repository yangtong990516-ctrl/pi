# redteam-recon


---

## passive-recon

## 数据源
- 证书透明日志（crt.sh）：`curl -s "https://crt.sh/?q=%25.<domain>&output=json" | jq -r '.[].name_value' | sort -u`
- DNS 记录：`dig +short <domain> A/AAAA/MX/TXT/NS`
- whois：注册人、网段、ASN
- 搜索引擎与代码托管平台：泄露的子域、密钥、内部地址
- 公开端口测绘数据（Shodan/Censys/FOFA 等，需授权 API）

## 落库要求
每条发现调用 （技能记录：asset_upsert），provenance 固定为 `passive`，tool 记录具体数据源。

---

## active-scan

## 前置检查
1. 确认目标 CIDR 在授权范围内。
2. 确认当前时间窗口允许主动扫描。

## 命令
- 快速存活：`nmap -sn <cidr>`
- 全端口（限速）：`nmap -sS -p- --min-rate 1000 -T4 <target> -oX runs/nmap-full.xml`
- 服务版本：`nmap -sV -sC -p <ports> <target> -oX runs/nmap-svc.xml`

## 本地已备好的工具（绝对路径，优先用它们，不要手搓）
| 用途 | 命令 |
| --- | --- |
| 端口扫描 | `nmap` / `masscan` / `~/.dsh/redteam/toolkit/naabu/naabu`（SYN 需 root，否则加 `-scan-type c`） |
| 内网综合扫描 | `~/.dsh/redteam/toolkit/fscan/fscan`（技能 fscan-intranet） |
| 内网测绘/指纹 | `~/.dsh/redteam/toolkit/gogo/gogo`（技能 gogo-intranet） |
| HTTP 探测（ProjectDiscovery） | `~/.local/bin/pd-httpx`（**注意：`/usr/bin/httpx` 是 Python httpx 库的 CLI，不是这个**） |
| 子域/解析 | `~/.dsh/redteam/toolkit/subfinder/subfinder`、`~/.dsh/redteam/toolkit/dnsx/dnsx` |
| POC 扫描 | `nuclei`（模板已在 `~/.local/nuclei-templates`） |
| 目录爆破 | `ffuf`、`~/.dsh/redteam/toolkit/dirsearch/dirsearch` |
| 隧道 | `~/.dsh/redteam/toolkit/suo5/suo5-linux-amd64`、`toolkit/chisel/chisel`、`toolkit/frp/{frpc,frps}` |
| 完整清单 | `~/.dsh/redteam/toolkit/清单.md`、`技能工具清单.md` |

## 落库要求
provenance = `active`，tool = `nmap`/`masscan`，记录 scan_run。

---

## recon-pipeline

# 外部信息收集流水线（ProjectDiscovery）

把"一个域名"变成"一份带标题、技术栈、端口的存活资产清单"，是信息收集阶段的标准动作。
四个工具**有固定顺序**：先枚举子域 → 再解析 → 再探端口 → 再判存活 → 最后抓页面与接口。

## 本机工具

| 工具 | 路径 | 版本 | 作用 |
| --- | --- | --- | --- |
| subfinder | `$DSH_HOME/redteam/toolkit/subfinder/subfinder` | v2.16.0 | 被动子域枚举（多源聚合） |
| dnsx | `$DSH_HOME/redteam/toolkit/dnsx/dnsx` | v1.3.1 | 批量解析、DNS 爆破、泛解析过滤 |
| naabu | `$DSH_HOME/redteam/toolkit/naabu/naabu` | v2.6.1 | 高速端口扫描（SYN 需 root，否则 `-scan-type c`） |
| httpx | `$DSH_HOME/redteam/toolkit/httpx/httpx`（包装器 `~/.local/bin/pd-httpx`） | v1.12.0 | HTTP 存活/标题/技术栈/状态码 |
| katana | ❌ 未安装（如需抓取用技能 `browser-automation`） | — | 爬虫抓页面与接口 |
| ksubdomain | `$DSH_HOME/redteam/toolkit/ksubdomain/ksubdomain` | v0.7 | 无状态子域爆破（比 dnsx 爆破快） |
| OneForAll | `$DSH_HOME/redteam/toolkit/oneforall/OneForAll-0.4.5/`（源码，需 `.venv`） | v0.4.5 | 子域收集全家桶（字典大，慢但全） |

> ⚠️ **`/usr/bin/httpx` 是 Python httpx 库的 CLI，不是 ProjectDiscovery 的**——必须用 `pd-httpx` 或绝对路径。

## 标准流水线

```bash
mkdir -p runs
TK=$DSH_HOME/redteam/toolkit
D=example.com

# ① 子域枚举（被动，多源）
$TK/subfinder/subfinder -d $D -all -silent -o runs/subfinder-$D.txt
# 补充：无状态爆破（有字典时）
# $TK/ksubdomain/ksubdomain -d $D -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# ② 泛解析过滤 + 解析存活（dnsx 会剔除泛解析噪声，这一步很关键）
cat runs/subfinder-$D.txt | $TK/dnsx/dnsx -silent -a -resp -o runs/dnsx-$D.txt
awk '{print $1}' runs/dnsx-$D.txt | sort -u > runs/alive-domains-$D.txt

# ③ 端口扫描（先 top100 快速过一遍，再对存活主机扫全端口）
$TK/naabu/naabu -l runs/alive-domains-$D.txt -top-ports 100 -silent -o runs/naabu-$D.txt
# 全端口（慢，确认在范围内再做）：-p - -rate 1000

# ④ HTTP 存活与指纹（标题/状态码/技术栈/服务器）
$TK/httpx/httpx -l runs/alive-domains-$D.txt -ports 80,443,8080,8443,8000,8888,9000 \
  -title -status-code -tech-detect -web-server -follow-redirects -silent \
  -json -o runs/httpx-$D.jsonl

# ⑤ 整理成资产清单（URL 清单给目录爆破/漏洞扫描用）
python3 - <<'PY'
import json
out=set()
for line in open('runs/httpx-'+'$D'+'.jsonl'):
    try: j=json.loads(line)
    except: continue
    if j.get('url'): out.add(j['url'])
open('runs/alive-urls.txt','w').write('\n'.join(sorted(out)))
print('存活 URL:', len(out))
PY
```

## 单条命令速查（不想跑全流程时）

```bash
# 只要子域
$TK/subfinder/subfinder -d example.com -all -silent

# 只要存活 URL（已有域名清单）
pd-httpx -l domains.txt -silent -o alive.txt

# 只要端口（单 IP，全端口）
nmap -sS -p- --min-rate 1000 -T4 1.2.3.4 -oX runs/nmap-1.2.3.4.xml   # 技能 active-scan
$TK/naabu/naabu -host 1.2.3.4 -p - -rate 1000 -silent
```

## 与其它技能的分工

| 场景 | 用哪个 |
| --- | --- |
| 单位名/域名 → 互联网资产面（含未备案、同主体） | **`fofa-recon`**（测绘，找得最全） |
| 域名 → 子域 → 存活 → 指纹 | **本技能**（流水线，标准化） |
| 不接触目标（纯被动） | `passive-recon`（crt.sh/dig/whois） |
| 内网大网段测绘 | `gogo-intranet`（**不要用本技能打内网**） |
| SP​A/JS 渲染页面、接口清单 | `browser-automation` / `kimi-webbridge` |
| 存活主机端口/服务版本 | `active-scan`（nmap/masscan） |

**推荐顺序**：`fofa-recon` 铺面（拿单位资产全景）→ 本技能对每个域名做流水线（拿存活与指纹）→ `active-scan` 对重点 IP 定版本。

## 纪律

1. **只测授权范围内目标**：naabu 全端口、ksubdomain 爆破都属于**主动**行为，确认在范围内再做。
2. **限速**：`naabu -rate 1000` 起步；`subfinder` 用被动源不加 `-all` 时更安静（配额消耗也少）。
3. **泛解析**：`dnsx` 会过滤；如果结果里出现大量同 IP 的随机子域，说明目标有泛解析，改用 `-wd` 或换爆破字典。
4. **API key（可选增强）**：`subfinder` 配置 `~/.config/subfinder/provider-config.yaml` 可接
   Shodan/Censys/VirusTotal 等源（**需要用户提供 key**，见技能 `redteam-setup`）；没 key 也能用免费源。

## 输出与落库（强制）

1. **逐条 `（技能记录：asset_add）`**：`target`（域名/IP）、`names`（域名）、`ports`（带 `url=host`、`title`、`product=server`）、
   `provenance=passive`（subfinder/dnsx 阶段）或 `active`（naabu/httpx 主动探测）、`tool`（具体工具名）。
2. **按 /24 自动建 C 段**（`（技能记录：asset_add）` 已自动处理）→ 用 `（技能记录：asset_link）` 建立
   `resolves`（域名→IP）、`contains`（C 段→IP）关系，形成图谱（技能 `asset-correlation`）。
3. 每个阶段 `（技能记录：chain_add）`：`stage_code=recon`，`tool` 写实际命令原文，`result` 写数量摘要
   （如 `subfinder 命中 42 个子域 → dnsx 存活 31 → naabu 开放 87 端口 → httpx 存活 19 个站点`）。
4. **重点标注边缘资产**：测试/预发环境（test/dev/uat/pre/staging）、老旧系统、非标准端口、
   非备案主体但同 C 段/IP 的站点——写进 `note`，这些是资产梳理阶段的高优先级目标。
5. 结论给指挥者时**给数字**：子域数 / 存活 IP 数 / 开放端口数 / 存活站点数 / 其中高优先级几条。

---

## fofa-recon

# FOFA 资产测绘（限速安全版）

## 凭据
- API Key：从环境变量 `FOFA_KEY` 读取（**任何 key 都不要写进仓库**；本技能所有脚本只读环境变量）
- 接口：`GET https://fofa.info/api/v1/search/all`
- 参数：`key`、`qbase64`（查询语句的 base64）、`size`、`page`、`fields`

## 官方限额（fofa.info/vip，2026-09 核对）

| 会员 | 查询请求 | 获取数据量 | **查询接口并发** |
|---|---|---|---|
| 注册用户（免费） | 无 API（页面 300 次/月） | 页面 3,000 条/月 | — |
| 个人版 | 10,000 次/月 | 100,000 条/月 | **1 秒 1 次** |
| 专业版 | 80,000 次/月 | 800,000 条/月 | **1 秒 1 次** |
| 商业版 | 900,000 次/月 | 9,000,000 条/月 | **1 秒 2 次** |
| 企业版 V2 | 5,000,000 次/月 | 50,000,000 条/月 | **1 秒 5 次** |

## 实测行为（实测验证）
- 连续快速请求 → **HTTP 429**，响应体 `{"error":true,"errmsg":"[45012] 请求速度过快"}`
- key 错误 → HTTP 200 + `{"error":true,"errmsg":"[-700] 账号无效"}`
- 单页 `size` 到 1000 正常；请求间隔 ≥1 秒时连续请求均返回 200
- 结论：**按 1 请求/秒 保守执行，绝不并发**；遇到 45012 必须退避，不要立即重试

## 使用纪律（重要）
1. **精确查询优先**：`domain=` / `cert=` / `org=` 先缩小范围，再考虑 `body=` / `title=` 等宽泛语句。宽语句返回几十万条没意义，还烧配额。
2. **不要全量翻页**：默认只取前 1–3 页（每页 100 条）拿到测绘入口即可。
3. **单进程单线程**：同一时刻只跑一个采集任务；多个查询串行排队。
4. **结果落盘缓存**：同一查询不要重复拉，先看 `runs/fofa-*.jsonl` 是否已有。
5. **长任务放后台**：用 `run_in_background` 跑，别阻塞对话。
6. 配额耗尽或被限速时改用其他数据源（证书透明、DNS、搜索引擎），不要死磕。

## 限速客户端（用这个，不要手写 curl 循环）
```bash
cat > runs/fofa_client.py <<'PY'
#!/usr/bin/env python3
"""FOFA 采集（限速安全）：单线程 + 最小间隔 + 指数退避 + 结果缓存。

用法:
  python3 runs/fofa_client.py 'domain="example.com"' --pages 2 --size 100 --interval 2.0
输出:
  runs/fofa-<slug>.jsonl   每行一个资产 {host,ip,port,title,domain,server,protocol}
"""
import argparse, base64, hashlib, json, os, sys, time, urllib.error, urllib.parse, urllib.request

KEY = os.environ["FOFA_KEY"]
API = "https://fofa.info/api/v1/search/all"
FIELDS = "host,ip,port,title,domain,server,protocol"
RETRY_BACKOFF = [5, 15, 45]          # 遇 45012 的退避秒数
HARD_BUDGET = 30                      # 单次运行最多请求次数


def fetch(query, page, size, timeout=30):
    qs = urllib.parse.urlencode({
        "key": KEY, "qbase64": base64.b64encode(query.encode()).decode(),
        "size": size, "page": page, "fields": FIELDS,
    })
    req = urllib.request.Request(API + "?" + qs, headers={"User-Agent": "redteam-recon/1.0"})
    try:
        with urllib.request.urlopen(req, timeout=timeout) as r:
            return r.status, json.load(r)
    except urllib.error.HTTPError as e:
        body = e.read().decode("utf-8", "ignore")
        try:
            return e.code, json.loads(body)
        except Exception:
            return e.code, {"error": True, "errmsg": body[:200]}


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("query")
    ap.add_argument("--pages", type=int, default=2, help="最多翻页数（默认 2）")
    ap.add_argument("--size", type=int, default=100, help="每页条数（默认 100，不要调大）")
    ap.add_argument("--interval", type=float, default=2.0, help="请求最小间隔秒（默认 2.0）")
    ap.add_argument("--out", default=None)
    args = ap.parse_args()

    slug = hashlib.md5(args.query.encode()).hexdigest()[:8]
    out = args.out or f"runs/fofa-{slug}.jsonl"
    os.makedirs(os.path.dirname(out) or ".", exist_ok=True)

    if os.path.exists(out) and os.path.getsize(out) > 0:
        print(f"[cache] {out} 已存在（{os.path.getsize(out)} bytes），如需重拉请先删除", file=sys.stderr)
        return 0

    seen, rows, requests_made, last_ts = set(), [], 0, 0.0
    for page in range(1, args.pages + 1):
        # ---- 限速：保证两次请求之间至少间隔 interval 秒 ----
        wait = args.interval - (time.time() - last_ts)
        if wait > 0:
            time.sleep(wait)

        status, data = None, None
        for attempt, backoff in enumerate([0] + RETRY_BACKOFF):
            if attempt:
                print(f"[backoff] 等待 {backoff}s 后重试 (page={page})", file=sys.stderr)
                time.sleep(backoff)
            last_ts = time.time()
            requests_made += 1
            if requests_made > HARD_BUDGET:
                print(f"[stop] 达到请求预算 {HARD_BUDGET}", file=sys.stderr)
                break
            status, data = fetch(args.query, page, args.size)
            if status == 200 and not data.get("error"):
                break
            errmsg = str(data.get("errmsg", ""))
            if "45012" in errmsg or status == 429:
                continue                      # 限速 → 退避重试
            print(f"[stop] 不可重试错误 http={status} {errmsg}", file=sys.stderr)
            data = None
            break
        if data is None:
            break

        results = data.get("results") or []
        print(f"[page {page}] {len(results)} 条 (size={data.get('size')}, 消耗F点={data.get('consumed_fpoint')})", file=sys.stderr)
        for item in results:
            if isinstance(item, str):
                item = [item]
            host = item[0] if len(item) > 0 else ""
            ip = item[1] if len(item) > 1 else ""
            port = item[2] if len(item) > 2 else ""
            key = (host, ip, port)
            if key in seen:
                continue
            seen.add(key)
            rows.append({
                "host": host, "ip": ip, "port": port,
                "title": item[3] if len(item) > 3 else "",
                "domain": item[4] if len(item) > 4 else "",
                "server": item[5] if len(item) > 5 else "",
                "protocol": item[6] if len(item) > 6 else "",
            })
        if len(results) < args.size:
            break                              # 已到末页

    with open(out, "w", encoding="utf-8") as f:
        for row in rows:
            f.write(json.dumps(row, ensure_ascii=False) + "\n")
    print(f"[done] {len(rows)} 条资产 → {out}（共 {requests_made} 次请求）", file=sys.stderr)
    return 0


if __name__ == "__main__":
    sys.exit(main())
PY
```

### 用法
```bash
mkdir -p runs
python3 runs/fofa_client.py 'domain="example.com"' --pages 2 --size 100 --interval 2.0
python3 runs/fofa_client.py 'cert="example.com"' --pages 3 --interval 2.0
python3 runs/fofa_client.py 'org="示例科技有限公司"' --pages 2 --interval 2.0
```

### 落库
读 `runs/fofa-*.jsonl`，逐条 `（技能记录：asset_add）`：`provenance="passive"`、`tool="fofa"`，端口带 `url=host`、`title`、`product=server`；域名写进 `names`。

## 常用查询语句
| 目的 | 语句 |
|---|---|
| 某域名全部资产 | `domain="example.com"` |
| 证书包含该域名 | `cert="example.com"` |
| 某 C 段 | `ip="203.0.113.0/24"` |
| 单位名称 | `org="示例科技有限公司"` |
| 按标题找后台 | `title="登录" && domain="example.com"` |
| 按 body 特征 | `body="/seeyon/"` |
| 指定端口 | `ip="203.0.113.0/24" && port="3306"` |

> 宽泛语句（单独用 `body=` / `title=`）结果量大、消耗配额快，务必叠加域名或 IP 限定。

---

## web-fingerprint

## 方法
1. 响应头：`Server`、`X-Powered-By`、`Set-Cookie` 特征。
2. 页面特征：favicon 哈希、静态资源路径、报错页、robots.txt。
3. 主动探测：nuclei 技术识别模板 `nuclei -u <url> -tags tech`；HTTP 存活与技术栈汇总用 `~/.local/bin/pd-httpx -l urls.txt -tech-detect -title -status-code -web-server`（**ProjectDiscovery 版必须用 pd-httpx，`/usr/bin/httpx` 是 Python 库的 CLI**）。
4. 大网段批量指纹用技能 `gogo-intranet`（`gogo -i <cidr> -p top2 -v --af`，主动指纹要加 `-v`）。
5. 版本比对：从指纹推断产品与版本，为漏洞检测做准备。

## 落库要求
写入 fingerprint 表：category（框架/中间件/CMS/组件）、vendor、product、version、evidence。

---

## asset-correlation

## 关联规则
- 域名解析 → `resolves` 边（domain → asset）
- C 段归属 → `contains` 边（segment → asset）
- 端口开放 → `exposes` 边（asset → port）
- 证书 SAN 共享 → `shares_cert` 边

## 落库要求
通过 （技能记录：asset_link） 写入 edge 表；重复关系幂等。