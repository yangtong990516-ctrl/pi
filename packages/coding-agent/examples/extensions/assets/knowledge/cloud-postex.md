# 云 / 后渗透方法论（Cloud & Post-Exploitation）

占位符约定：TARGET、TOKEN、ROLE_B、PIVOT_SLOT、RESULT_LOG、CLEANUP_SLOT。

## 云（Cloud）
1. 云元数据：SSRF 打 `http://169.254.169.254/latest/meta-data/iam/security-credentials/`（AWS）、`http://metadata.google.internal/computeMetadata/v1/`（GCP，需 `Metadata-Flavor: Google`）、`http://100.100.100.200/latest/meta-data/`（阿里云）。
2. 凭据利用：拿到 TOKEN 后枚举 `aws sts get-caller-identity` / `az account list` / `gcloud auth list`；检查权限过大（PassRole、S3 写、KMS 解密）。
3. 存储桶：`s3 ls s3://BUCKET` 未授权列举/写；对象版本泄露。
4. 容器：`/var/run/docker.sock` 未授权、`kubectl get secrets`、镜像层泄露。

## 后渗透（Post-Exploitation）
1. 枚举：`whoami /all`、`net user`、`net localgroup administrators`、`wmic qfe`、环境变量（AWS keys）。
2. 凭据提取：`mimikatz sekurlsa::logonpasswords`、`secretsdump.py`、浏览器密码/会话 cookie、SSH 私钥、云 CLI 缓存。
3. 横向：计划任务/服务二进制替换、`schtasks`、WMI 远程执行、SMB 共享滥用。
4. 持久化：注册表 Run 键、计划任务、服务、SSH authorized_keys（需显式确认）。

## 清理
- 删除上传的临时文件、恢复被改配置、注销创建的账号；输出 CLEANUP_SLOT 清单。

## 输出
按 目标占位流程： 开头：元数据/凭据枚举 → 权限放大 → 横向 PIVOT_SLOT → 清理 CLEANUP_SLOT → 记录 RESULT_LOG。
