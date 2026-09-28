# 报告模板（Reporting）

占位符约定：TARGET、RESULT_DIR、FINDING_ID、SEVERITY、EVIDENCE。

## 单条发现格式（findings.jsonl 行）
```json
{
  "id": "FINDING_ID",
  "target": "TARGET",
  "url": "http://TARGET/ENDPOINT",
  "class": "sql-injection|xss|ssrf|idor|rce|auth-bypass|...",
  "severity": "critical|high|medium|low|info",
  "status": "verified|suspected",
  "evidence": "EVIDENCE（可复现的请求/响应包或 PoC）",
  "fix": "参数化查询|输入白名单|服务端校验|...",
  "timestamp": "ISO8601"
}
```

## 覆盖矩阵（Coverage Matrix）
| 资产/端点 | recon | 目录 | SQLi | XSS | SSRF | IDOR | 上传 | RCE | 认证 | 状态 |
|---|---|---|---|---|---|---|---|---|---|---|
| http://TARGET/login | ✅ | - | ⬜ | ⬜ | - | ✅ | - | - | ✅ | 已测 |
| http://TARGET/api/v1/orders | ✅ | ✅ | ⬜ | - | - | ✅ | - | - | ⬜ | 待测 |

- ✅ = 已覆盖无发现；⬜ = 已覆盖有发现（标 FINDING_ID）；- = 不适用；待测 = 未测。
- 门禁：recon 完成才进入测试阶段；每条 verified 发现必须有 EVIDENCE。

## 汇报顺序
1. 执行摘要（3-5 条最重要发现 + 影响）。
2. 覆盖矩阵。
3. 按严重度排序的发现明细（ID / 描述 / 复现步骤 / 证据 / 修复建议）。
4. 未覆盖项与后续建议。

## 输出
按 目标占位流程： 开头，产出覆盖矩阵表格 + findings.jsonl 行模板 + 分级修复清单。
