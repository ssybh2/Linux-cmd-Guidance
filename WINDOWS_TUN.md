# Windows TUN 网络排错

适用于 Windows + Clash Verge / Mihomo TUN 环境下的常见问题：

- 浏览器可以打开 ChatGPT，但 Codex CLI 一直 `Reconnecting...`
- CLI 的 HTTPS/TLS 失败
- NoMachine / SSH / 局域网设备在开启 TUN 后异常

## 1. 这次问题的根因

本机同时开启了 Clash Verge 的 TUN，并且 Windows 用户环境变量里还残留：

```text
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
```

于是浏览器可以正常通过 TUN 访问网络，但 Codex CLI 会优先读取 `HTTP_PROXY` / `HTTPS_PROXY`，额外走一次 `127.0.0.1:7897`。

实际测试结果是：

- 纯 TUN：HTTPS 正常
- 强制走 `127.0.0.1:7897`：`CONNECT` 成功，但 TLS handshake 失败

所以最终方案是：**保留 TUN，删除多余的 HTTP/HTTPS 代理环境变量，让 Codex 直接走 TUN。**

---

## 2. 最简单的诊断

### 查看代理环境变量

```powershell
Get-ChildItem Env: | Where-Object {$_.Name -match "proxy"}
```

如果看到：

```text
HTTP_PROXY  http://127.0.0.1:xxxx
HTTPS_PROXY http://127.0.0.1:xxxx
```

而你已经开启 Clash Verge TUN，就要怀疑 CLI 正在重复走代理。

### 测试纯 TUN 是否能访问 ChatGPT

```powershell
curl.exe `
  --noproxy "*" `
  --ssl-revoke-best-effort `
  -I https://chatgpt.com `
  --max-time 20
```

如果能收到类似：

```text
HTTP/1.1 403 Forbidden
Server: cloudflare
```

说明 TLS 和 HTTP 链路已经成功。这里的 `403` 是 Cloudflare 对 curl 的 challenge，不代表网络不通。

### 对比显式代理

把端口换成你自己的 Clash HTTP / Mixed Port：

```powershell
curl.exe `
  -x http://127.0.0.1:7897 `
  -I https://chatgpt.com `
  --max-time 20
```

如果纯 TUN 正常，而这里出现：

```text
SSL/TLS connection failed
```

就说明显式代理路径有问题。

---

## 3. Codex CLI 修复方法

### 删除当前终端和用户级代理变量

```powershell
$vars = @(
  "HTTP_PROXY",
  "HTTPS_PROXY",
  "ALL_PROXY",
  "http_proxy",
  "https_proxy",
  "all_proxy"
)

foreach ($v in $vars) {
  Remove-Item "Env:$v" -ErrorAction SilentlyContinue
  [Environment]::SetEnvironmentVariable($v, $null, "User")
}
```

`NO_PROXY` 可以保留。

关闭当前 PowerShell，再重新打开。

### 确认已经清理

```powershell
Get-ChildItem Env: | Where-Object {$_.Name -match "proxy"}
```

正常情况下可以只剩：

```text
NO_PROXY localhost,127.0.0.1,::1
```

### 测试 Codex

```powershell
codex
```

如果可以正常对话，就说明问题已经解决。

推荐的网络路径是：

```text
Codex
  ↓
Windows 网络栈
  ↓
Clash Verge TUN
  ↓
Internet
```

而不是：

```text
Codex
  ↓
HTTP_PROXY / HTTPS_PROXY
  ↓
127.0.0.1:7897
  ↓
TUN
```

---

## 4. NoMachine / 局域网设备排查

例如连接：

```text
10.190.16.26:4000
```

先检查 Windows 实际准备从哪张网卡访问：

```powershell
Find-NetRoute -RemoteIPAddress 10.190.16.26 |
  Format-List InterfaceAlias,InterfaceIndex,NextHop,SourceAddress
```

测试端口：

```powershell
Test-NetConnection 10.190.16.26 -Port 4000
```

如果：

```text
TcpTestSucceeded : True
```

说明 TCP 4000 已经能到达目标设备。

如果路由显示：

```text
InterfaceAlias : Meta
```

说明这个私网地址正在经过 Clash TUN。

如果端口能通但 NoMachine / SSH / 其他私网应用仍异常，可以在 Clash Verge 的 TUN 设置中把这个地址排除：

```text
10.190.16.26/32
```

只排除这一台设备通常比直接排除整个 `10.0.0.0/8` 更安全。

---

## 5. 一键查看 Windows 网络状态

```powershell
Write-Host "=== Proxy ==="
Get-ChildItem Env: | Where-Object {$_.Name -match "proxy"}

Write-Host "`n=== Adapters ==="
Get-NetAdapter | Format-Table Name,InterfaceDescription,Status,ifIndex -AutoSize

Write-Host "`n=== Default Routes ==="
Get-NetRoute -AddressFamily IPv4 |
  Where-Object {$_.DestinationPrefix -eq "0.0.0.0/0"} |
  Format-Table DestinationPrefix,NextHop,InterfaceAlias,RouteMetric -AutoSize

Write-Host "`n=== ChatGPT ==="
curl.exe --noproxy "*" --ssl-revoke-best-effort -I https://chatgpt.com --max-time 20
```

## 最终结论

当 Clash Verge 已经开启 TUN 时，CLI 通常不需要再额外设置 `HTTP_PROXY` / `HTTPS_PROXY`。

如果出现“浏览器正常、CLI 不正常”，优先检查：

```powershell
Get-ChildItem Env: | Where-Object {$_.Name -match "proxy"}
```

很多问题就是 **TUN + 残留显式代理叠加** 导致的。
