# Chrome 内网 HTTPS 证书提醒处理

- 日期：2026-07-05
- 标签：Chrome、HTTPS、内网、自签名证书、VMware

## 场景

访问内网地址：

```text
https://192.168.1.80/ui/#/login
```

Chrome 每次提示“不安全”。`curl` 普通访问失败，跳过证书校验后成功：

```powershell
curl.exe -I https://192.168.1.80/ui/#/login
curl.exe -k -I https://192.168.1.80/ui/#/login
```

本次证书信息：

- 证书来源：VMware ESX Server Default Certificate
- 证书链问题：不受信任的根证书
- SAN：缺失
- 证书指纹：`C662A7B98C760BA41BA1FF957942492F980EDC69`

## 判断方法

用 PowerShell 读取服务端证书：

```powershell
$hostName='192.168.1.80'
$tcp=New-Object Net.Sockets.TcpClient($hostName,443)
$ssl=New-Object Net.Security.SslStream($tcp.GetStream(),$false,({$true} -as [Net.Security.RemoteCertificateValidationCallback]))
$ssl.AuthenticateAsClient($hostName)
$cert=New-Object System.Security.Cryptography.X509Certificates.X509Certificate2($ssl.RemoteCertificate)
$ssl.Close(); $tcp.Close()
$cert | Select-Object Subject,Issuer,NotBefore,NotAfter,Thumbprint | Format-List
$cert.Extensions | ForEach-Object {
  if ($_.Oid.FriendlyName -eq 'Subject Alternative Name') { $_.Format($true) }
}
```

如果证书没有 `Subject Alternative Name`，导入证书信任也可能继续报域名/IP 不匹配。

## 推荐做法

不要使用全局 `--ignore-certificate-errors` 作为日常 Chrome 参数。更安全的方式是使用指定证书公钥的 SPKI 放行：

```text
--ignore-certificate-errors-spki-list=<SPKI_SHA256_BASE64>
```

本次设备的 SPKI：

```text
bfUSyjkQILV1hw72wL6dSX3Ag4uq1IbqGsH/BJnqn8Y=
```

已创建当前用户 Chrome 快捷方式：

```text
C:\Users\huangjun\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Google Chrome.lnk
C:\Users\huangjun\Desktop\Google Chrome.lnk
```

快捷方式参数：

```text
--ignore-certificate-errors-spki-list=bfUSyjkQILV1hw72wL6dSX3Ag4uq1IbqGsH/BJnqn8Y=
```

## 生效条件

- 关闭所有已经运行的 Chrome。
- 从开始菜单或桌面的 `Google Chrome` 快捷方式重新打开。
- 之后在正常 Chrome 窗口中访问 `https://192.168.1.80/ui/#/login`。

如果 Chrome 已经以不带参数的方式运行，后来再点带参数快捷方式，可能仍复用旧进程，参数不会生效。

## 更彻底的长期方案

从设备侧重新签发证书，要求：

- 证书被当前系统信任。
- SAN 中包含 `IP:192.168.1.80`，或改用一个固定域名并在 SAN 中包含该域名。

设备证书正确后，不需要 Chrome 启动参数。
