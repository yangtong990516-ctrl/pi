# API / GraphQL / WebSocket 测试方法论

占位符约定：API_ENDPOINT / PARAM / COOKIE / HEADER / TOKEN / OAST_DOMAIN。

## 基础
1. 收集 API 文档：/swagger、/openapi.json、/redoc、/v2/api-docs、/graphql（introspection）。
2. 认证令牌：JWT 解码看 role/exp；OAuth2 流程（auth code / implicit / client credentials / PKCE）。
3. 版本与调试端点：/v1/../v2、/debug、/actuator、/api/internal、X-Forwarded-Host 头。

## 越权与 BOLA/BFLA
- BOLA（对象级授权）：`GET /api/orders/1` → 遍历 `/api/orders/USER_ID`；用 `--vary "id=1,2,3"` 对比响应差异；同长不同内容用 body_similarity 判定。
- BFLA（功能级授权）：ROLE_A 的 Token 调 ROLE_B 的管理端点；对比 200/403。
- 批量赋值（Mass Assignment）：POST 里附加 `role=admin`、`isAdmin=true`、`balance=999999` 字段。
- 请求走私（HTTP Request Smuggling）：CL.TE / TE.CL / TE.TE 变体探测。

## GraphQL
- 内省：`query { __schema { types { name } } }`（开启本身即发现项）。
- Batching 绕过限速：单请求多 query 枚举。
- 别名爆破：同一字段用多个 alias 绕 rate limit。
- 深度/循环：`fragment` 递归拉爆服务端（DoS 需确认）。

## WebSocket
- 握手校验：Origin 头是否校验（CSWSH）。
- 消息注入：客户端消息里拼接 SQL/XSS/命令 PAYLOAD。
- 越权订阅：切换用户身份后是否仍能收到他人数据流。
- 心跳/重连逻辑：重连 token 泄露或可重放。

## 无回显与 OAST
- 参数级注入一律配 DNS 外带：`OAST_DOMAIN` 回连验证；`--marker` 归因到具体 PAYLOAD。

## 输出
按 目标占位流程： 开头，给出 API_ENDPOINT 列表、每类测试的 curl 模板与判定条件（200/403/时长/长度差异）。
