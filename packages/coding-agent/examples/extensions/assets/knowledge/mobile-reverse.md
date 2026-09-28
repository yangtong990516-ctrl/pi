# 移动端逆向方法论（Android / iOS）

占位符约定：SAMPLE（apk/ipa）、CHECK_FN、OFFSET、PATCH_BYTE、HOOK_PTR、SIGN_BYPASS_SNIPPET。

## Android（apk）
1. 解包：`apktool d SAMPLE.apk -o OUT_DIR`（资源+smali）；`jadx -d OUT_DIR SAMPLE.apk`（Java 反编译）。
2. 静态分析：
   - 定位 CHECK_FN：搜 `isValid|verify|license|activate|premium|pro` 相关字符串与调用链。
   - 网络层：搜 `okhttp3|retrofit|interceptor`，找加密/签名参数（AES/RSA/MD5）。
   - 壳检测：`apktool` 失败或 smali 异常 → 脱壳（frida-dexdump / BlackDex）。
3. 动态分析：`frida -U -f package.name -l hook.js`。
   - hook CHECK_FN：`Java.perform(() => { const C = Java.use("com.example.Check"); C.isValid.implementation = () => true; })`
   - hook 网络：`Java.use("okhttp3.OkHttpClient")` 抓明文请求。
4. patch：smali 改 `const/4 v0, 0x0` → `0x1`；或回编译 `apktool b` 后重打包签名。
5. 重打包签名：`apksigner sign --ks KEYSTORE SAMPLE-aligned.apk`；绕过校验用 SIGN_BYPASS_SNIPPET（重打包检测：签名校验/完整性校验/包名校验 hook）。

## iOS（ipa）
1. 解包：`unzip SAMPLE.ipa`；`class-dump -H Payload/App.app -o HEADERS`。
2. 静态：`strings` 找 license 逻辑；Mach-O 反汇编（Hopper/Ghidra）定位 CHECK_FN 与 OFFSET。
3. 动态：`frida -U -n AppName`；`ObjC.classes` 枚举；hook `-[Class checkLicense]` 返回 YES。
4. patch：`optool`/`insert_dylib` 注入 dylib；`codesign --force --deep --sign -` 重签。

## 通用
- 证书/SSL pinning 绕过：frida `ssl_pinning_bypass` 脚本或 `frida-server` + objection。
- 数据存储：`sqlite3` 读 SharedPreferences/NSUserDefaults/Keychain 找弱存储凭据。
- 备份：Android `adb backup` / iOS iTunes 备份解密（弱密码）。

## 输出
按 目标占位流程： 开头：SAMPLE → 静态分析 CHECK_FN → 动态 hook HOOK_PTR → patch OFFSET PATCH_BYTE → 重打包 SIGN_BYPASS_SNIPPET → 启动验证。
