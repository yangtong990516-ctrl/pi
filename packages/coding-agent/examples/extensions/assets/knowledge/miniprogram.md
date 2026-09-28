# 小程序逆向方法论（WeChat Mini Program）

占位符约定：WXAPKG_TOOL、SAMPLE（.wxapkg）、ENDPOINT_LIST、CHECK_FN、PAYLOAD。

## 定位与解包
1. 获取 wxapkg：PC 微信缓存 `%APPDATA%\Tencent\WeChat\XPlugin\Plugins\RadiumWMPF` 或手机 `/data/data/com.tencent.mm/MicroMsg/.../appbrand/pkg/`。
2. 解包：`wxappUnpacker` / `unveilr` / `wxapkg` 工具 → `WXAPKG_TOOL SAMPLE.wxapkg -o OUT_DIR`。
3. 产物：app.json（页面路由）、app-service.js（业务逻辑）、WXML/WXSS（模板样式）、pages/。

## 静态分析
- 提取接口：`grep -oE "https?://[a-zA-Z0-9./_-]+" app-service.js` → ENDPOINT_LIST；再 `grep -oE "(api|v[0-9])/[a-zA-Z0-9_/?=&.-]+"`。
- 定位 CHECK_FN：搜索 `encrypt|sign|token|aes|rsa|md5|secret|wx.request` 相关函数，还原加密/签名参数构造。
- 明文密钥：常硬编码在 JS 里；`--follow` 反混淆（AST 还原）后搜索 hex/base64 字符串。

## 动态分析
- hook 网络层：`wx.request` / `wx.connectSocket` 包装函数打日志，抓明文请求与响应。
- 本地存储：`wx.getStorageSync` 数据、`wx.getUserInfo` 凭据。
- 开发者工具：微信开发者工具导入解包目录（需补 appid），`--inspect` 远程调试。

## 重打包与验证
- 修改 JS 逻辑（如 `isVip` 返回值）→ 重新打包 wxapkg → 回灌目录覆盖。
- 服务端校验优先：所有客户端绕过以服务端接口实测为准，构造 PAYLOAD 直连 ENDPOINT_LIST。

## 输出
按 目标占位流程： 开头：wxapkg 解包 → 反编译提接口 ENDPOINT_LIST → 定位 CHECK_FN → hook wx.request 抓明文 → 重打包回灌验证。
