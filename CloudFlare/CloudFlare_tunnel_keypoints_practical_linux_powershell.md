# Cloudflare Tunnel 暴露私有服务：重点难点驱动实操教程（Linux + PowerShell）

> 版本：2026-05-23  
> 适用对象：需要把本地、WSL、VPS 或内网服务通过 Cloudflare Tunnel 安全暴露到 Cloudflare 域名的开发者。  
> 目标：不直接开放公网端口，通过 Cloudflare Tunnel + Access 实现 HTTP / SSH 私有服务访问。  
> 脱敏说明：本文不包含真实域名、邮箱、Token、IP、用户名。所有敏感信息统一使用占位符。

---

## 0. 占位符与统一约定

| 占位符 | 含义 | 示例 |
|---|---|---|
| `<YOUR_DOMAIN>` | 已接入 Cloudflare 的域名 | `example.com` |
| `<API_HOST>` | HTTP/API 测试域名 | `api-dev.<YOUR_DOMAIN>` |
| `<SSH_HOST>` | SSH 访问域名 | `ssh.<YOUR_DOMAIN>` |
| `<TUNNEL_NAME>` | Named Tunnel 名称 | `dev-tunnel` |
| `<TUNNEL_TOKEN>` | Dashboard 生成的 Tunnel token | 不要提交到 Git |
| `<YOUR_EMAIL>` | Cloudflare Access 允许访问的邮箱 | `user@example.com` |
| `<LINUX_USER>` | Linux / WSL 用户名 | `devuser` |
| `<WINDOWS_USER>` | Windows 用户名 | `devuser` |
| `<SERVER_PUBLIC_IP>` | VPS 公网 IP | 仅用于端口暴露验证 |
| `<SSH_ORIGIN_IP>` | SSH 服务所在内网 IP | `192.168.1.10` |

本文统一使用 HTTP 测试端口：

```text
18080
```

原因：Windows 上 `8080` 可能被 Hyper-V、WSL、Docker、代理软件、安全软件或系统保留端口占用。Python HTTP Server 绑定 `8080` 时可能出现：

```text
PermissionError: [WinError 10013]
```

因此本文统一使用：

```text
http://127.0.0.1:18080
```

---

## 1. 本章重点与实操映射

| 重点 | 必须掌握的实操动作 | Linux 指令 | PowerShell 指令 |
|---|---|---|---|
| Tunnel 链路模型 | 先确认本地 origin，再接 Tunnel | `curl -i http://127.0.0.1:18080` | `curl.exe -i http://127.0.0.1:18080` |
| Quick Tunnel vs Named Tunnel | 分清临时域名和正式域名 | `cloudflared tunnel --url http://127.0.0.1:18080` | 同左 |
| Public Hostname 映射 | 正式域名指向本地服务 | Dashboard 配置 | Dashboard 配置 |
| Access 访问控制 | 只允许指定人员访问 | Dashboard 配置 | Dashboard 配置 |
| SSH over Tunnel | 区分 `ssh` 客户端和 `sshd` 服务端 | `ssh -V` / `sudo service ssh status` | `ssh -V` / `Get-Service sshd` |
| 服务生命周期 | 启动、停止、禁用、卸载 | `systemctl` | `Get-Service` / `Stop-Service` |
| 分层排错 | 本地 → Quick Tunnel → Named Tunnel → Access | `curl` / `systemctl` | `curl.exe` / `Get-Service` |

---

## 2. 本章难点与排错入口

| 难点 | 典型现象 | 直接排查命令 | 常见修正 |
|---|---|---|---|
| `localhost` 指向错误环境 | Windows 跑 cloudflared，服务在 WSL，502 | `curl.exe -i http://127.0.0.1:18080` | 让服务和 cloudflared 跑在同一环境，或填写 WSL IP |
| Quick Tunnel 成功，正式域名 502 | `trycloudflare.com` 成功，`<API_HOST>` 失败 | `Get-Service cloudflared` / Dashboard route | 修改 Named Tunnel 的 Service URL 为 `http://127.0.0.1:18080` |
| PowerShell `curl` 解析问题 | `curl` 参数异常 | `curl.exe -i ...` | 显式使用 `curl.exe` |
| Windows 8080 绑定失败 | `WinError 10013` | `netsh interface ipv4 show excludedportrange protocol=tcp` | 改用 `18080` |
| Windows 无 `ssh` 命令 | `ssh is not recognized` | `where.exe ssh` | 安装 OpenSSH Client、使用 Git SSH 或 WSL SSH |
| Windows `sshd` 安装失败 | `Add-WindowsCapability 0x80240023` | `Get-WindowsCapability -Online \| Where-Object Name -like 'OpenSSH*'` | 用 WSL sshd 代替，或修复 Windows Update/FoD |
| SSH `websocket: bad handshake` | SSH 连接立刻断开 | 检查 WebSockets / Access / SSH route | SSH route 必须是 `ssh://127.0.0.1:22` |

---

# Part A：Cloudflare Tunnel 的链路模型

## A1. HTTP 链路

```text
Browser / curl
    ↓
https://<API_HOST>
    ↓
Cloudflare Edge
    ↓
Cloudflare Tunnel
    ↓
cloudflared connector
    ↓
http://127.0.0.1:18080
```

关键判断：

```text
外部用户访问的是 Cloudflare 域名，不是直接访问你的公网 IP:18080。
```

所以安全目标是：

```text
公网 18080 不开放
Cloudflare 域名可访问
cloudflared 停止后外部访问失败
```

## A2. SSH 链路

```text
ssh client
    ↓
ProxyCommand: cloudflared access ssh --hostname <SSH_HOST>
    ↓
Cloudflare Access
    ↓
Cloudflare Tunnel
    ↓
cloudflared connector
    ↓
sshd: ssh://127.0.0.1:22
```

SSH 比 HTTP 多了几个组件：

```text
ssh 客户端
sshd 服务端
ProxyCommand
Access Application
WebSocket
SSH route
```

因此 SSH 排错必须分层，不要直接从 Cloudflare 开始猜。

---

# Part B：Linux 实操流程

## B1. 安装 cloudflared

### Ubuntu / Debian

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings

curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update
sudo apt install -y cloudflared
cloudflared --version
```

## B2. 启动本地 HTTP origin：127.0.0.1:18080

```bash
mkdir -p ~/cf-tunnel-test
cd ~/cf-tunnel-test

echo "hello cloudflare tunnel on port 18080" > index.html
python3 -m http.server 18080 --bind 127.0.0.1
```

保持该终端不关闭。

新开一个终端验证：

```bash
curl -i http://127.0.0.1:18080
```

期望看到：

```text
HTTP/1.0 200 OK
...
hello cloudflare tunnel on port 18080
```

如果这里失败，不要继续配置 Cloudflare。先修本地服务。

## B3. Quick Tunnel 临时验证

```bash
cloudflared tunnel --url http://127.0.0.1:18080
```

成功后会输出：

```text
https://random-name.trycloudflare.com
```

新开终端验证：

```bash
curl -i https://random-name.trycloudflare.com
```

结论：

```text
Quick Tunnel 成功 = 本地服务 + 前台 cloudflared 链路正常。
Quick Tunnel 不会绑定 <API_HOST>。
```

退出 Quick Tunnel：

```text
Ctrl + C
```

## B4. 创建 Named Tunnel：Dashboard 方式

Cloudflare Dashboard：

```text
Cloudflare Dashboard
→ Networking
→ Tunnels
→ Create Tunnel
→ Select Cloudflared
→ Tunnel name: <TUNNEL_NAME>
```

选择 Linux 环境后，Dashboard 会给出安装命令。典型形式：

```bash
sudo cloudflared service install <TUNNEL_TOKEN>
```

检查服务：

```bash
sudo systemctl status cloudflared
```

启动 / 重启：

```bash
sudo systemctl start cloudflared
sudo systemctl restart cloudflared
```

## B5. 配置 Public Hostname：HTTP

Dashboard：

```text
Tunnel
→ <TUNNEL_NAME>
→ Public Hostname / Published applications
→ Add a public hostname
```

配置：

```text
Hostname: <API_HOST>
Service Type: HTTP
Service URL: http://127.0.0.1:18080
```

不要写：

```text
http://127.0.0.1:8080
```

验证：

```bash
curl -i https://<API_HOST>
```

如果启用了 Access，可能返回登录页或重定向，这是正常现象。

## B6. 配置 Access：限制访问人员

Dashboard：

```text
Zero Trust
→ Access controls
→ Applications
→ Add an application
→ Self-hosted
```

配置：

```text
Application name: api-dev
Subdomain: api-dev
Domain: <YOUR_DOMAIN>
```

Policy：

```text
Policy name: allow-myself
Action: Allow
Selector: Emails
Value: <YOUR_EMAIL>
```

验收：

```text
授权邮箱：可以访问
非授权邮箱：拒绝访问
隐身窗口：需要重新认证
```

## B7. Linux SSH origin

安装 OpenSSH Server：

```bash
sudo apt update
sudo apt install -y openssh-server
```

启动：

```bash
sudo service ssh start
sudo service ssh status
```

确认监听：

```bash
ss -lntp | grep ':22'
```

本机测试：

```bash
ssh <LINUX_USER>@127.0.0.1
```

如果本地 SSH 失败，不要继续配置 Cloudflare SSH。

## B8. 配置 Public Hostname：SSH

Dashboard：

```text
Tunnel
→ <TUNNEL_NAME>
→ Public Hostname / Published applications
→ Add a public hostname
```

如果 SSH 服务和 cloudflared 在同一台 Linux：

```text
Hostname: <SSH_HOST>
Service Type: SSH
Service URL: ssh://127.0.0.1:22
```

如果 SSH 服务在另一台内网机器：

```text
Hostname: <SSH_HOST>
Service Type: SSH
Service URL: ssh://<SSH_ORIGIN_IP>:22
```

协议必须是：

```text
ssh://
```

不要写：

```text
http://127.0.0.1:22
https://127.0.0.1:22
```

## B9. Linux SSH 客户端配置

确认客户端：

```bash
ssh -V
cloudflared --version
```

写入 SSH config：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh

cat > ~/.ssh/config <<'EOF'
Host <SSH_HOST>
    HostName <SSH_HOST>
    User <LINUX_USER>
    ProxyCommand cloudflared access ssh --hostname %h
EOF

chmod 600 ~/.ssh/config
```

测试：

```bash
ssh -v <SSH_HOST>
```

---

# Part C：PowerShell / Windows 实操流程

## C1. 安装 cloudflared.exe

管理员 PowerShell：

```powershell
$CloudflaredDir = "$env:ProgramFiles\cloudflared"
$CloudflaredExe = "$CloudflaredDir\cloudflared.exe"

New-Item -ItemType Directory -Path $CloudflaredDir -Force | Out-Null

Invoke-WebRequest `
  -Uri "https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.exe" `
  -OutFile $CloudflaredExe

& $CloudflaredExe --version
```

加入 PATH：

```powershell
$MachinePath = [Environment]::GetEnvironmentVariable("Path", "Machine")

if ($MachinePath -notlike "*$CloudflaredDir*") {
    [Environment]::SetEnvironmentVariable(
        "Path",
        "$MachinePath;$CloudflaredDir",
        "Machine"
    )
}

$env:Path += ";$CloudflaredDir"
cloudflared --version
```

## C2. 启动本地 HTTP origin：127.0.0.1:18080

```powershell
$TestDir = "C:\cf-tunnel-test"
$ApiPort = 18080

New-Item -ItemType Directory -Path $TestDir -Force | Out-Null
Set-Content -Path "$TestDir\index.html" -Value "hello cloudflare tunnel on port 18080"

cd $TestDir
python -m http.server $ApiPort --bind 127.0.0.1
```

保持窗口不关闭。

新开 PowerShell 验证：

```powershell
curl.exe -i http://127.0.0.1:18080
```

期望：

```text
HTTP/1.0 200 OK
...
hello cloudflare tunnel on port 18080
```

## C3. Quick Tunnel 临时验证

```powershell
cloudflared tunnel --url http://127.0.0.1:18080
```

成功后输出随机地址：

```text
https://random-name.trycloudflare.com
```

验证：

```powershell
curl.exe -i https://random-name.trycloudflare.com
```

说明：

```text
Quick Tunnel 只验证临时 trycloudflare.com 链路。
它不会自动绑定 <API_HOST>。
```

退出：

```text
Ctrl + C
```

## C4. 创建 Named Tunnel：Dashboard 方式

Cloudflare Dashboard：

```text
Cloudflare Dashboard
→ Networking
→ Tunnels
→ Create Tunnel
→ Select Cloudflared
→ Tunnel name: <TUNNEL_NAME>
```

选择 Windows 后，Dashboard 会给出命令：

```powershell
cloudflared.exe service install <TUNNEL_TOKEN>
```

管理员 PowerShell 执行：

```powershell
cloudflared.exe service install <TUNNEL_TOKEN>
```

检查服务：

```powershell
Get-Service cloudflared
```

启动 / 重启：

```powershell
Start-Service cloudflared
Restart-Service cloudflared
```

查看服务配置：

```powershell
sc.exe qc cloudflared
```

## C5. 配置 Public Hostname：HTTP

Dashboard：

```text
Tunnel
→ <TUNNEL_NAME>
→ Public Hostname / Published applications
→ Add a public hostname
```

配置：

```text
Hostname: <API_HOST>
Service Type: HTTP
Service URL: http://127.0.0.1:18080
```

验证：

```powershell
curl.exe -i https://<API_HOST>
```

如果返回 502，优先检查：

```powershell
curl.exe -i http://127.0.0.1:18080
Get-Service cloudflared
```

并检查 Dashboard 中 Service URL 是否还是旧的 `8080`。

## C6. 配置 Access：限制访问人员

Dashboard：

```text
Zero Trust
→ Access controls
→ Applications
→ Add an application
→ Self-hosted
```

配置：

```text
Application name: api-dev
Subdomain: api-dev
Domain: <YOUR_DOMAIN>
```

Policy：

```text
Policy name: allow-myself
Action: Allow
Selector: Emails
Value: <YOUR_EMAIL>
```

验收：

```text
授权邮箱：可以访问
非授权邮箱：拒绝访问
隐身窗口：需要重新认证
```

---

# Part D：PowerShell 下 SSH 重点实操

## D1. 先确认 SSH 客户端 `ssh.exe`

PowerShell：

```powershell
where.exe ssh
ssh -V
```

如果报错：

```text
ssh : The term 'ssh' is not recognized
```

说明 Windows 找不到 SSH 客户端。

### 方案 1：检查系统 OpenSSH Client

```powershell
Test-Path "$env:WINDIR\System32\OpenSSH\ssh.exe"
```

如果返回 `True`：

```powershell
& "$env:WINDIR\System32\OpenSSH\ssh.exe" -V
$env:Path += ";$env:WINDIR\System32\OpenSSH"
ssh -V
```

### 方案 2：安装 OpenSSH Client

管理员 PowerShell：

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
ssh -V
```

### 方案 3：用 Git for Windows 自带 SSH

```powershell
Test-Path "C:\Program Files\Git\usr\bin\ssh.exe"
& "C:\Program Files\Git\usr\bin\ssh.exe" -V
```

临时加入 PATH：

```powershell
$env:Path += ";C:\Program Files\Git\usr\bin"
ssh -V
```

### 方案 4：用 WSL SSH

```powershell
wsl ssh -V
```

如果 WSL 没有 SSH client：

```powershell
wsl sudo apt update
wsl sudo apt install -y openssh-client
```

## D2. 区分 ssh 和 sshd

```text
ssh  = 客户端，用来连接别人
sshd = 服务端，用来接受别人连接
```

Windows 检查服务端：

```powershell
Get-Service sshd
```

Linux / WSL 检查服务端：

```bash
sudo service ssh status
```

## D3. Windows sshd 安装失败时的绕行方式

如果执行：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

报错，例如：

```text
0x80240023
```

不要卡死。可以用 WSL 的 sshd 做 SSH origin。

## D4. WSL 作为 SSH origin

PowerShell 安装并启动 WSL sshd：

```powershell
wsl sudo apt update
wsl sudo apt install -y openssh-server
wsl sudo service ssh start
wsl sudo service ssh status
```

WSL 内测试：

```powershell
wsl bash -lc "ssh $(whoami)@127.0.0.1"
```

如果不知道 WSL 密码，重设：

```powershell
wsl whoami
wsl -u root
```

在 WSL root shell 中：

```bash
passwd <LINUX_USER>
exit
```

查 WSL IP：

```powershell
wsl hostname -I
```

假设输出：

```text
172.25.144.32
```

Dashboard SSH route 应配置为：

```text
Hostname: <SSH_HOST>
Service Type: SSH
Service URL: ssh://172.25.144.32:22
```

注意：WSL IP 重启后可能变化。长期使用更建议把 `cloudflared` 也放进 WSL，这样可以配置：

```text
Service URL: ssh://127.0.0.1:22
```

## D5. Windows SSH ProxyCommand 配置

为避免 `C:\Program Files` 路径带空格问题，建议复制 cloudflared：

```powershell
New-Item -ItemType Directory -Path "C:\cloudflared" -Force | Out-Null

Copy-Item `
  -Path "C:\Program Files\cloudflared\cloudflared.exe" `
  -Destination "C:\cloudflared\cloudflared.exe" `
  -Force
```

写入 SSH config：

```powershell
$SshDir = "$env:USERPROFILE\.ssh"
New-Item -ItemType Directory -Path $SshDir -Force | Out-Null

@"
Host <SSH_HOST>
    HostName <SSH_HOST>
    User <LINUX_USER>
    ProxyCommand C:/cloudflared/cloudflared.exe access ssh --hostname %h
"@ | Set-Content -Path "$SshDir\config" -Encoding ascii
```

测试：

```powershell
ssh -v <SSH_HOST>
```

或使用 Git SSH：

```powershell
& "C:\Program Files\Git\usr\bin\ssh.exe" -v <SSH_HOST>
```

---

# Part E：服务停止、退出、禁用与卸载

## E1. Linux Quick Tunnel 退出

Quick Tunnel 是前台进程：

```bash
cloudflared tunnel --url http://127.0.0.1:18080
```

退出：

```text
Ctrl + C
```

## E2. Linux HTTP 测试服务停止

如果 Python HTTP Server 在前台运行：

```text
Ctrl + C
```

如果需要查进程：

```bash
ss -lntp | grep ':18080'
ps aux | grep 'http.server'
```

杀进程：

```bash
pkill -f 'python3 -m http.server 18080'
```

## E3. Linux cloudflared systemd 服务

查看：

```bash
sudo systemctl status cloudflared
```

停止：

```bash
sudo systemctl stop cloudflared
```

启动：

```bash
sudo systemctl start cloudflared
```

重启：

```bash
sudo systemctl restart cloudflared
```

禁止开机自启：

```bash
sudo systemctl disable cloudflared
```

重新启用开机自启：

```bash
sudo systemctl enable cloudflared
```

卸载服务：

```bash
sudo cloudflared service uninstall
```

删除包：

```bash
sudo apt remove -y cloudflared
```

## E4. PowerShell Quick Tunnel 退出

前台运行：

```powershell
cloudflared tunnel --url http://127.0.0.1:18080
```

退出：

```text
Ctrl + C
```

## E5. PowerShell HTTP 测试服务停止

如果 Python HTTP Server 在前台：

```text
Ctrl + C
```

查 18080 端口：

```powershell
Get-NetTCPConnection -LocalPort 18080 -ErrorAction SilentlyContinue
```

查 PID：

```powershell
Get-NetTCPConnection -LocalPort 18080 -ErrorAction SilentlyContinue |
  Select-Object LocalAddress, LocalPort, State, OwningProcess
```

结束进程：

```powershell
Stop-Process -Id <PID> -Force
```

## E6. Windows cloudflared Service

查看：

```powershell
Get-Service cloudflared
```

停止：

```powershell
Stop-Service cloudflared
```

启动：

```powershell
Start-Service cloudflared
```

重启：

```powershell
Restart-Service cloudflared
```

禁用开机自启：

```powershell
Set-Service cloudflared -StartupType Disabled
```

恢复自动启动：

```powershell
Set-Service cloudflared -StartupType Automatic
```

卸载服务：

```powershell
cloudflared.exe service uninstall
```

删除程序目录：

```powershell
Remove-Item -Recurse -Force "C:\Program Files\cloudflared"
```

## E7. WSL sshd 停止

```powershell
wsl sudo service ssh stop
wsl sudo service ssh status
```

重新启动：

```powershell
wsl sudo service ssh start
```

---

# Part F：分层排错方法

## F1. HTTP 502 排错流程

当访问：

```text
https://<API_HOST>
```

返回 502 时，按顺序执行。

### Step 1：本地 origin

Linux：

```bash
curl -i http://127.0.0.1:18080
```

PowerShell：

```powershell
curl.exe -i http://127.0.0.1:18080
```

失败则先启动 HTTP Server。

### Step 2：Quick Tunnel

```bash
cloudflared tunnel --url http://127.0.0.1:18080
```

访问输出的 `trycloudflare.com`。

如果 Quick Tunnel 成功，但 `<API_HOST>` 失败，说明问题在 Named Tunnel / Public Hostname / Access。

### Step 3：Named Tunnel connector

Linux：

```bash
sudo systemctl status cloudflared
```

PowerShell：

```powershell
Get-Service cloudflared
```

### Step 4：Dashboard route

检查：

```text
<API_HOST>
Service Type: HTTP
Service URL: http://127.0.0.1:18080
```

重点确认不是：

```text
http://127.0.0.1:8080
```

### Step 5：Access

临时确认是否 Access 拦截：

```text
未登录：应跳转登录或拒绝
授权邮箱：应访问成功
非授权邮箱：应拒绝
```

## F2. SSH `websocket: bad handshake` 排错流程

### Step 1：SSH client 是否存在

PowerShell：

```powershell
where.exe ssh
ssh -V
```

Linux：

```bash
which ssh
ssh -V
```

### Step 2：sshd 是否可用

Linux / WSL：

```bash
sudo service ssh status
ssh <LINUX_USER>@127.0.0.1
```

Windows：

```powershell
Get-Service sshd
```

### Step 3：Tunnel SSH route

Dashboard 必须是：

```text
<SSH_HOST>
Service Type: SSH
Service URL: ssh://127.0.0.1:22
```

或：

```text
Service URL: ssh://<SSH_ORIGIN_IP>:22
```

### Step 4：Access Application

确认有一个 self-hosted application 绑定：

```text
<SSH_HOST>
```

Policy 允许：

```text
<YOUR_EMAIL>
```

### Step 5：WebSockets / Worker route 干扰

检查：

```text
Cloudflare Dashboard → Network → WebSockets: On
Worker Routes 不要覆盖 <SSH_HOST>/*
Bot / WAF 规则不要拦截 <SSH_HOST>
Binding Cookie 对 SSH/RDP 应关闭
```

---

# Part G：验收矩阵

## G1. HTTP 验收

| 验收项 | Linux | PowerShell | 通过标准 |
|---|---|---|---|
| 本地服务启动 | `python3 -m http.server 18080 --bind 127.0.0.1` | `python -m http.server 18080 --bind 127.0.0.1` | 无报错 |
| 本地服务可访问 | `curl -i http://127.0.0.1:18080` | `curl.exe -i http://127.0.0.1:18080` | 返回测试页面 |
| Quick Tunnel 可用 | `cloudflared tunnel --url ...` | 同左 | `trycloudflare.com` 可访问 |
| Named Tunnel 运行 | `systemctl status cloudflared` | `Get-Service cloudflared` | Running / active |
| 正式域名可访问 | `curl -i https://<API_HOST>` | `curl.exe -i https://<API_HOST>` | 返回页面或 Access 登录 |
| 关闭 cloudflared 后不可访问 | `sudo systemctl stop cloudflared` | `Stop-Service cloudflared` | 外部无法访问 origin |
| 公网端口未开放 | `nc -vz <SERVER_PUBLIC_IP> 18080` | `Test-NetConnection <SERVER_PUBLIC_IP> -Port 18080` | 连接失败 |

## G2. SSH 验收

| 验收项 | Linux / WSL | PowerShell | 通过标准 |
|---|---|---|---|
| SSH client 可用 | `ssh -V` | `ssh -V` 或 `wsl ssh -V` | 输出版本 |
| sshd 可用 | `sudo service ssh status` | `Get-Service sshd` 或 WSL sshd | Running |
| 本地 SSH 可用 | `ssh <LINUX_USER>@127.0.0.1` | `wsl ssh <LINUX_USER>@127.0.0.1` | 可登录 |
| SSH route 正确 | Dashboard | Dashboard | `ssh://127.0.0.1:22` |
| ProxyCommand 正确 | `~/.ssh/config` | `%USERPROFILE%\.ssh\config` | 使用 `cloudflared access ssh` |
| SSH 域名可连 | `ssh -v <SSH_HOST>` | `ssh -v <SSH_HOST>` | Access 认证后进入 SSH |

---

# Part H：推荐学习顺序

## H1. 第一阶段：只做 HTTP

目标：先验证 Tunnel 核心链路。

```text
127.0.0.1:18080
→ Quick Tunnel
→ Named Tunnel
→ Access
```

不要一开始就加入 SSH，否则排错维度太多。

## H2. 第二阶段：加入服务生命周期

掌握：

```text
启动
停止
重启
禁用开机自启
卸载服务
停止后验收不可访问
```

## H3. 第三阶段：加入 SSH

顺序：

```text
ssh client 可用
sshd server 可用
本地 ssh 成功
SSH route 配置
Access app 配置
ProxyCommand 配置
ssh -v 调试
```

## H4. 第四阶段：处理 Windows / WSL 混合环境

重点理解：

```text
cloudflared 跑在哪里，127.0.0.1 就指向哪里。
```

长期建议：

```text
服务在 Windows → cloudflared 也跑 Windows
服务在 WSL     → cloudflared 也跑 WSL
服务在 VPS     → cloudflared 也跑 VPS
```

---

# Part I：一键检查脚本

## I1. PowerShell HTTP 检查脚本

保存为：

```text
check-cloudflare-tunnel-18080.ps1
```

内容：

```powershell
$ApiHost = "<API_HOST>"
$ApiPort = 18080
$LocalApiUrl = "http://127.0.0.1:$ApiPort"

Write-Host "=== 1. cloudflared version ==="
try {
    cloudflared --version
} catch {
    Write-Warning "cloudflared not found in PATH"
}

Write-Host "`n=== 2. cloudflared service ==="
Get-Service cloudflared -ErrorAction SilentlyContinue

Write-Host "`n=== 3. local listening port 18080 ==="
Get-NetTCPConnection -State Listen |
  Where-Object { $_.LocalPort -eq $ApiPort } |
  Select-Object LocalAddress, LocalPort, State, OwningProcess |
  Format-Table -AutoSize

Write-Host "`n=== 4. local API check ==="
curl.exe -i $LocalApiUrl

Write-Host "`n=== 5. public hostname DNS check ==="
Resolve-DnsName $ApiHost -ErrorAction SilentlyContinue

Write-Host "`n=== 6. public hostname HTTP check ==="
curl.exe -I "https://$ApiHost"
```

运行：

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\check-cloudflare-tunnel-18080.ps1
```

## I2. Linux HTTP 检查脚本

保存为：

```text
check-cloudflare-tunnel-18080.sh
```

内容：

```bash
#!/usr/bin/env bash
set -euo pipefail

API_HOST="<API_HOST>"
API_PORT="18080"
LOCAL_API_URL="http://127.0.0.1:${API_PORT}"

echo "=== 1. cloudflared version ==="
cloudflared --version || true

echo
echo "=== 2. cloudflared service ==="
systemctl status cloudflared --no-pager || true

echo
echo "=== 3. local listening port 18080 ==="
ss -lntp | grep ":${API_PORT}" || true

echo
echo "=== 4. local API check ==="
curl -i "${LOCAL_API_URL}" || true

echo
echo "=== 5. public hostname HTTP check ==="
curl -I "https://${API_HOST}" || true
```

运行：

```bash
chmod +x check-cloudflare-tunnel-18080.sh
./check-cloudflare-tunnel-18080.sh
```

---

# Part J：最终结论

本章真正要掌握的不是单条命令，而是以下闭环：

```text
1. 本地服务先通
2. Quick Tunnel 验证 cloudflared 基本链路
3. Named Tunnel 绑定正式域名
4. Public Hostname 指向正确 origin
5. Access 控制访问人员
6. systemd / Windows Service 管理生命周期
7. HTTP 和 SSH 分开排错
8. Windows / WSL 场景避免 localhost 误判
```

最小可用 HTTP 配置：

```text
<API_HOST>
→ HTTP
→ http://127.0.0.1:18080
```

最小可用 SSH 配置：

```text
<SSH_HOST>
→ SSH
→ ssh://127.0.0.1:22
```

最佳排错原则：

```text
先本地，后 Cloudflare。
先 Quick Tunnel，后 Named Tunnel。
先 HTTP，后 SSH。
先 origin 可达，后 Access 策略。
```
