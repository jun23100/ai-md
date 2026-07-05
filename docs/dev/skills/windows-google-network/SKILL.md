---
name: windows-google-network
description: Windows 环境下诊断并打通 Google 网络访问的流程。适用于 www.google.com、accounts.google.com、gstatic、Chrome 登录或 Google 服务访问超时、DNS 污染、直连不可用、本机无代理、需要安装并启用 Cloudflare WARP 的场景。
---

# Windows Google Network

## 判断问题类型

先分层验证，不要只测浏览器：

```powershell
curl.exe -I --connect-timeout 10 --max-time 25 https://www.google.com
curl.exe -I --connect-timeout 10 --max-time 25 https://accounts.google.com
curl.exe -I --connect-timeout 10 --max-time 25 https://www.gstatic.com
curl.exe -I --connect-timeout 10 --max-time 25 https://dl.google.com
Resolve-DnsName www.google.com -Type A
```

本次现象：

- `www.google.com` 超时。
- `accounts.google.com` 超时。
- `dl.google.com` 可用，说明 Google 下载源或部分 CDN 不一定受影响。
- `www.google.com` 被解析到非 Google 地址，例如 `157.240.7.20`、`69.171.235.22`，属于 DNS 污染或错误解析。
- 手动 `--resolve` 到常见 Google IP 后仍超时，说明 hosts 不能单独解决主站访问。

## 检查本机代理

先确认是否已有可用代理：

```powershell
netsh winhttp show proxy
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' |
  Select-Object ProxyEnable,ProxyServer,AutoConfigURL
```

检查常见本地端口：

```powershell
$ports=7890,7891,7892,1080,10808,10809,20170,20171,2080,8080,8118
foreach($p in $ports){
  $c=New-Object Net.Sockets.TcpClient
  $a=$c.BeginConnect('127.0.0.1',$p,$null,$null)
  if($a.AsyncWaitHandle.WaitOne(300)){
    try{$c.EndConnect($a); Write-Output "OPEN $p"} catch{}
  }
  $c.Close()
}
```

如果没有代理端口，也没有 Clash、v2rayN、sing-box、mihomo 等进程，直连 Google 通常无法稳定打通。

## 安装 Cloudflare WARP

使用 winget 官方源安装：

```powershell
winget install --id Cloudflare.Warp --exact --source winget --silent --accept-package-agreements --accept-source-agreements
```

如果 winget 的 `msstore` 源报错，但提示 winget 源存在包，指定 `--source winget` 重试。

定位 `warp-cli`：

```powershell
Get-ChildItem 'C:\Program Files','C:\Program Files (x86)' -Recurse -Filter warp-cli.exe -ErrorAction SilentlyContinue
```

注册并连接：

```powershell
& 'C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe' --accept-tos registration new
& 'C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe' --accept-tos mode warp
& 'C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe' --accept-tos connect
Start-Sleep -Seconds 8
& 'C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe' --accept-tos status
```

成功状态应类似：

```text
Status update: Connected
Network: healthy
```

也可用 `ipconfig` 确认出现 `CloudflareWARP` 适配器和 `172.16.0.2` 一类地址。

## 验证 Google

连接 WARP 后重新验证：

```powershell
curl.exe -I --connect-timeout 10 --max-time 25 https://www.google.com
curl.exe -I --connect-timeout 10 --max-time 25 https://accounts.google.com
curl.exe -I --connect-timeout 10 --max-time 25 https://www.gstatic.com
```

通过标准：

- `www.google.com` 返回 `HTTP/1.1 200 OK`。
- `accounts.google.com` 返回 `302` 到登录页，或其他 Google 登录相关正常响应。
- `www.gstatic.com` 返回 Google 服务端响应，根路径 `404` 也可接受，因为代表网络已到达 Google。

## 注意事项

- Google 主站问题通常不是单纯 DNS 问题；hosts 可能改善解析，但无法解决被阻断的 TCP/TLS 链路。
- `dl.google.com` 可用不代表 `www.google.com`、`accounts.google.com` 可用。
- WARP 会创建系统级网络通道，可能影响所有应用的出站网络；出现问题时可断开：

```powershell
& 'C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe' --accept-tos disconnect
```

- 恢复连接：

```powershell
& 'C:\Program Files\Cloudflare\Cloudflare WARP\warp-cli.exe' --accept-tos connect
```
