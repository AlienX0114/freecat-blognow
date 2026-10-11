# Windows 使用 acme.sh 申请 Cloudflare 泛域名证书并自动续期、部署到 Nginx

## 1. 适用场景与工作原理

本教程适用于以下环境：

- 操作系统：Windows 10/11 或 Windows Server
- Web 服务：本地或同机运行的 Nginx
- 域名解析：域名托管在 Cloudflare
- 证书目标：同时保护主域名和所有一级子域名，例如：
  - `example.com`
  - `*.example.com`
- 证书生命周期：自动续期、自动复制到 Nginx 目录、自动重载 Nginx

工作流程如下：

1. acme.sh 使用 Cloudflare API 自动添加 ACME 验证所需的 DNS TXT 记录。
2. 证书颁发机构验证域名控制权。
3. acme.sh 签发同时包含主域名和泛域名的证书。
4. 证书成功签发或续期后，acme.sh 将私钥和完整证书链复制到指定目录。
5. 复制完成后自动执行 Nginx 重载命令。
6. Windows 计划任务每天检查证书有效期，接近到期时自动续期。

> 泛域名证书只能保护一级子域名，例如 `www.example.com`、`api.example.com`。  
> 它不能保护多级子域名，例如 `dev.api.example.com`；也不能自动覆盖二级域名。

acme.sh 是基于 Bash 的 ACME 客户端，官方支持在 Windows 的 Cygwin、Git Bash 等环境中运行。[citation:20]

---

## 2. 环境准备

### 2.1 安装 Git for Windows

acme.sh 需要 Bash 环境。推荐安装 **Git for Windows**，安装时建议选择：

- Git from the command line and also from 3rd-party software
- 默认使用 Git Bash
- 勾选 curl
- 勾选 OpenSSL（如安装包提供）

Git for Windows 运行脚本时依赖 MSYS2/Cygwin 风格的 POSIX 兼容层。[citation:30]

### 2.2 安装 Nginx

确保 Nginx 可以正常启动，并确认以下信息：

- Nginx 可执行文件路径，例如：

```text
C:\nginx\nginx.exe
```

- 运行 Nginx 的用户对证书目录具有读取权限。
- 后续重载命令能够停止、验证和重新加载配置。

### 2.3 准备域名

将域名 DNS 服务器设置为 Cloudflare 提供的服务器，并确认域名已经在 Cloudflare 面板正常解析。

泛域名证书验证的是 DNS 区域，不要求每个子域名预先存在解析记录。

---

## 3. 创建 Cloudflare API Token

不要使用 Cloudflare 全局 API Key。推荐创建最小权限的 API Token。

### 3.1 创建流程

1. 登录 Cloudflare 控制台。
2. 打开：`我的个人资料 → API 令牌 → 创建令牌`。
3. 选择 **Create Custom Token**。
4. 权限设置：

| 权限 | 范围 |
|---|---|
| Zone / Zone / Read | 区域读取 |
| Zone / DNS / Edit | DNS 编辑 |

5. 区域资源建议选择：

```text
包含 / 特定区域 / example.com
```

6. 设置适当的有效期。
7. 如有固定公网出口，可配置客户端 IP 限制。
8. 创建后保存 Token。**Token 只完整显示一次。**

官方 Cloudflare 集成支持通过 `CF_Token`、`CF_Account_ID` 和可选的 `CF_Zone_ID` 自动完成 DNS-01 验证。[citation:2]

### 3.2 获取账户 ID 和区域 ID

- Account ID：Cloudflare 主页右上角或域名概览页。
- Zone ID：目标域名概览页右侧。

---

## 4. 安装 acme.sh

关闭所有普通命令行窗口，只打开 **Git Bash**，然后执行：

```bash
curl https://get.acme.sh | sh -s email=admin@example.com
```

安装完成后关闭 Git Bash，重新打开，然后检查版本：

```bash
acme.sh --version
```

如果提示找不到命令，可使用完整路径：

```bash
~/.acme.sh/acme.sh --version
```

官方还支持通过 Git 克隆后执行 `./acme.sh --install` 进行安装。[citation:31]

### 4.1 目录说明

Windows 默认目录通常为：

```text
C:\Users\你的用户名\.acme.sh
```

acme.sh 在该目录中保存：

- 账户配置
- Cloudflare 凭据
- 证书私钥
- 证书文件
- 续期配置
- 运行日志

---

## 5. 配置 Cloudflare 凭据

在 Git Bash 中配置环境变量：

```bash
export CF_Token="你的Cloudflare_API_Token"
export CF_Account_ID="你的Cloudflare_Account_ID"
export CF_Zone_ID="你的Cloudflare_Zone_ID"
```

如果 Token 已经具备所有区域的读取权限，则 `CF_Zone_ID` 可以省略。为了限制单一域名权限，建议明确配置。

验证凭据：

```bash
echo "$CF_Token"
echo "$CF_Account_ID"
```

执行正式签发后，acme.sh 会保存相关配置，后续续期可以重复使用。[citation:2]

> 不要将包含 Token 的命令粘贴到公开仓库、聊天记录、日志系统或截图。

---

## 6. 申请泛域名证书

以下命令必须在 Git Bash 中执行。

### 6.1 定义变量

将以下内容保存为 `issue-wildcard.sh`：

```bash
#!/usr/bin/env bash

export CF_Token="你的Cloudflare_API_Token"
export CF_Account_ID="你的Cloudflare_Account_ID"
export CF_Zone_ID="你的Cloudflare_Zone_ID"

DOMAIN="example.com"
NGINX_DIR="/c/nginx/ssl/${DOMAIN}"

mkdir -p "${NGINX_DIR}"

~/.acme.sh/acme.sh --issue \
  --dns dns_cf \
  -d "${DOMAIN}" \
  -d "*.${DOMAIN}" \
  --key-file "${NGINX_DIR}/privkey.pem" \
  --fullchain-file "${NGINX_DIR}/fullchain.pem" \
  --reloadcmd "C:/nginx/nginx.exe -s reload" \
  --keylength ec-256 \
  --days 60
```

参数说明：

| 参数 | 作用 |
|---|---|
| `--dns dns_cf` | 使用 Cloudflare DNS API 自动验证 |
| `-d example.com` | 主域名 |
| `-d *.example.com` | 泛域名 |
| `--key-file` | 保存域名私钥 |
| `--fullchain-file` | 保存证书及完整链 |
| `--reloadcmd` | 证书签发或续期后自动重载 Nginx |
| `--keylength ec-256` | 签发 ECDSA P-256 证书 |
| `--days 60` | 接近 60 天有效期时执行续期 |

`--install-cert` 或签发参数中的路径输出应被视为最终部署文件，不要直接复制 acme.sh 内部目录中的源文件。[citation:20]

### 6.2 先使用测试环境

在正式签发前，建议先使用测试环境：

```bash
~/.acme.sh/acme.sh --issue \
  --dns dns_cf \
  -d example.com \
  -d "*.example.com" \
  --test
```

测试环境颁发的是不受浏览器信任的证书，但可以用于验证：

- Cloudflare API 权限
- DNS TXT 记录写入
- Git Bash 环境
- 证书文件复制
- Nginx 重载命令

测试成功后，再执行正式签发：

```bash
bash issue-wildcard.sh
```

不要频繁申请正式证书。Let’s Encrypt 对每个域名每周的注册表与证书数量有限制，正式申请前优先使用测试环境。

---

## 7. 配置 Nginx

### 7.1 单域名示例

```nginx
server {
    listen 80;
    server_name www.example.com;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name www.example.com;

    ssl_certificate     C:/nginx/ssl/example.com/fullchain.pem;
    ssl_certificate_key C:/nginx/ssl/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        root html;
        index index.html;
    }
}
```

### 7.2 多个子域名复用同一证书

由于证书同时包含 `example.com` 和 `*.example.com`，可以在多个 `server` 块中复用：

```nginx
ssl_certificate     C:/nginx/ssl/example.com/fullchain.pem;
ssl_certificate_key C:/nginx/ssl/example.com/privkey.pem;
```

例如：

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;
    ...
}

server {
    listen 443 ssl http2;
    server_name img.example.com;
    ...
}
```

### 7.3 测试配置

在 Git Bash 中执行：

```bash
"/c/nginx/nginx.exe" -t
```

重新加载：

```bash
"/c/nginx/nginx.exe" -s reload
```

如果 Nginx 作为 Windows 服务运行，可使用以下命令：

```bash
"/c/nginx/nginx.exe" -s stop
sleep 2
"/c/nginx/nginx.exe"
```

将实际命令写入 `--reloadcmd`，并确保计划任务账户能够执行。

---

## 8. 验证证书

### 8.1 查看证书信息

```bash
openssl x509 -in /c/nginx/ssl/example.com/fullchain.pem -noout -text
```

重点检查：

- 颁发者
- 有效期
- 使用者备用名称：
  - `DNS:example.com`
  - `DNS:*.example.com`

### 8.2 访问网站

使用以下地址访问：

```text
https://www.example.com
```

也可以创建临时测试子域名，例如：

```text
https://acme-test.example.com
```

只要证书未过期，且对应 `server_name` 已配置，泛域名证书即可生效。

---

## 9. 自动续期

### 9.1 acme.sh 续期机制

acme.sh 会保存每个证书的配置。使用 `--days 60` 后，它会在证书剩余有效期接近 60 天时尝试续期。

证书续期成功以后，会执行：

1. 将新的私钥复制到指定 `--key-file`。
2. 将新的完整证书链复制到指定 `--fullchain-file`。
3. 执行 `--reloadcmd` 中的 Nginx 重载命令。

官方文档说明：

- `--reloadcmd`：证书签发或续期后执行。
- `--renew-hook`：每个成功续期的证书执行一次。
- `--post-hook`：无论签发或续期是否成功都会执行。[citation:12]

本教程使用 `--reloadcmd`，更适合“复制证书并让 Nginx 重新加载”这一场景。

### 9.2 创建 Windows 计划任务

创建每天运行两次的检查任务：

```powershell
$actions = @()

$actions += New-ScheduledTaskAction -Execute "C:\Program Files\Git\bin\bash.exe" -Argument '-l -c "$HOME/.acme.sh/acme.sh --cron --home $HOME/.acme.sh >> $HOME/.acme.sh/renew.log 2>&1"'

$actions += New-ScheduledTaskAction -Execute "C:\Program Files\Git\bin\bash.exe" -Argument '-l -c "$HOME/.acme.sh/acme.sh --renew -d example.com --days 60 >> $HOME/.acme.sh/renew.log 2>&1"'

$trigger1 = New-ScheduledTaskTrigger -Daily -At "02:15"
$trigger2 = New-ScheduledTaskTrigger -Daily -At "14:15"

$principal = New-ScheduledTaskPrincipal -UserId $env:USERNAME -LogonType Interactive -RunLevel Limited

Register-ScheduledTask -TaskName "acme.sh Auto Renew" -Action $actions -Trigger $trigger1,$trigger2 -Principal $principal -Description "自动检查并续期 acme.sh 证书" -Force
```

如果 Nginx 安装目录需要管理员权限，可以将 `-RunLevel` 修改为 `Highest`，并通过任务计划程序改为“以最高权限运行”。

> 请确认 Git 的 Bash 实际路径。常见路径包括：
>
> ```text
> C:\Program Files\Git\bin\bash.exe
> C:\Program Files\Git\usr\bin\bash.exe
> ```

### 9.3 查询与管理任务

```powershell
Get-ScheduledTask -TaskName "acme.sh Auto Renew"

Get-ScheduledTaskInfo -TaskName "acme.sh Auto Renew"

schtasks /query /tn "acme.sh Auto Renew"

schtasks /run /tn "acme.sh Auto Renew"

Unregister-ScheduledTask -TaskName "acme.sh Auto Renew" -Confirm:$false
```

也可以打开：

```text
taskschd.msc
```

查看任务历史记录。

---

## 10. 手动测试续期流程

### 10.1 仅模拟签发流程

```bash
~/.acme.sh/acme.sh --renew -d example.com --days 60
```

如果距离到期时间较长，acme.sh 可能认为不需要续期。

### 10.2 强制续期

```bash
~/.acme.sh/acme.sh --renew -d example.com --force
```

强制续期会真正向 ACME 服务器申请证书，仅在调试时使用。

### 10.3 检查续期结果

```bash
~/.acme.sh/acme.sh --list

~/.acme.sh/acme.sh --info -d example.com
```

检查：

```text
C:\nginx\ssl\example.com\privkey.pem
C:\nginx\ssl\example.com\fullchain.pem
```

确认两个文件均已更新，然后检查 Nginx 是否重新加载成功。

---

## 11. 证书轮换与私钥

默认情况下，acme.sh 续期时不一定每次都生成新的域名私钥。如果需要每次续期都轮换私钥，可以执行：

```bash
~/.acme.sh/acme.sh --renew \
  -d example.com \
  --force \
  --always-force-new-domain-key
```

如果启用了私钥轮换，必须确保 Nginx 的新私钥和 `fullchain.pem` 来自同一次签发，避免私钥与证书不匹配。

---

## 12. 更新 acme.sh

### 12.1 查看版本

```bash
~/.acme.sh/acme.sh --version
```

### 12.2 手动更新

```bash
~/.acme.sh/acme.sh --upgrade
```

### 12.3 开启自动更新

```bash
~/.acme.sh/acme.sh --upgrade --auto-upgrade
```

关闭自动更新：

```bash
~/.acme.sh/acme.sh --upgrade --auto-upgrade 0
```

acme.sh 支持自动升级功能，是否启用取决于你的服务器变更管理策略。

---

## 13. 卸载与重新安装

### 13.1 删除证书配置

```bash
~/.acme.sh/acme.sh --remove -d example.com
```

执行前先备份 Nginx 正在使用的证书。

### 13.2 卸载 acme.sh

```bash
~/.acme.sh/acme.sh --uninstall
```

该操作会卸载 acme.sh 并移除默认续期任务。

### 13.3 重新签发

重新安装后需要重新配置环境变量：

```bash
export CF_Token="新的Cloudflare_API_Token"
export CF_Account_ID="你的Cloudflare_Account_ID"
export CF_Zone_ID="你的Cloudflare_Zone_ID"
```

再执行：

```bash
bash issue-wildcard.sh
```

---

## 14. 常见问题排查

### 14.1 Cloudflare API 报权限错误

检查：

- Token 是否已过期。
- 是否同时具有区域读取和 DNS 编辑权限。
- 区域资源是否包含目标域名。
- `CF_Account_ID` 和 `CF_Zone_ID` 是否填写正确。
- 是否存在客户端 IP 限制。

使用调试模式：

```bash
~/.acme.sh/acme.sh --issue \
  --dns dns_cf \
  -d example.com \
  -d "*.example.com" \
  --test \
  --debug
```

### 14.2 TXT 记录超时

DNS 记录可能需要更长时间传播。可以增加等待时间：

```bash
~/.acme.sh/acme.sh --issue \
  --dns dns_cf \
  -d example.com \
  -d "*.example.com" \
  --dnssleep 180
```

### 14.3 证书申请成功但 Nginx 没有重载

依次检查：

1. Git Bash 中能否直接执行 Nginx 重载。
2. 计划任务账户是否有 Nginx 目录权限。
3. `--reloadcmd` 是否使用 Windows 路径。
4. Nginx 是否运行。
5. 计划任务历史记录是否显示成功。
6. `renew.log` 是否包含错误。

### 14.4 提示找不到 acme.sh

不要使用 PowerShell 执行：

```powershell
acme.sh --list
```

应在 Git Bash 中执行：

```bash
~/.acme.sh/acme.sh --list
```

或者重新打开 Git Bash，使别名生效。

### 14.5 Nginx 无法读取新证书

检查：

- 证书目录权限。
- 路径中的斜杠方向。
- 证书文件是否被其他进程锁定。
- 是否使用了 acme.sh 内部目录中的软链接或源文件。

建议始终使用 `--key-file` 和 `--fullchain-file` 输出到固定部署目录。

---

## 15. 安全建议

1. 使用 Cloudflare API Token，不要使用全局 API Key。
2. 将 Token 权限限制到具体区域。
3. 设置 Token 有效期，到期后轮换。
4. 限制能够运行 Nginx 和计划任务的账户。
5. 不要将证书或密钥提交到 Git 仓库。
6. 定期查看 `renew.log`。
7. 保留至少一个可用的旧证书备份，便于紧急回滚。
8. 生产系统不要在短时间内反复执行 `--force` 续期。
9. 如果 Nginx 位于 Cloudflare 代理后面，请确保 Cloudflare 到源站使用有效证书，或根据架构使用 Cloudflare 代理模式。

---

## 16. 推荐生产部署检查表

执行以下检查：

- [ ] Git for Windows 已安装并支持 Bash。
- [ ] Cloudflare API Token 权限最小化。
- [ ] 已使用测试环境完成签发验证。
- [ ] 正式证书同时包含主域名和泛域名。
- [ ] Nginx 使用 `--key-file`、`--fullchain-file` 的输出文件。
- [ ] `nginx -t` 配置检查通过。
- [ ] `--reloadcmd` 能够在 Bash 中手动执行。
- [ ] Windows 计划任务状态为“就绪”。
- [ ] 日志写入 `renew.log`。
- [ ] 已测试手动续期。
- [ ] 已备份当前可用的 Nginx 证书。
- [ ] 已确认证书 SAN 包含 `example.com` 与 `*.example.com`。
