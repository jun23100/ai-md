---
name: windows-github-bootstrap
description: Windows 环境下修复 GitHub 访问、创建并推送 GitHub 仓库、迁移本地项目目录、安装常用工具的操作流程。适用于 GitHub 主站访问不稳定、git 不在 PATH、需要用 ghproxy 拉取、用 Git Credential Manager 授权、通过 API 创建仓库、把项目移动到 C:\project、安装 Chrome 等场景。
---

# Windows GitHub Bootstrap

## 核心原则

- 先诊断，再修改系统配置。
- GitHub API、raw、主站、Git 操作要分别验证，不要把一个通道可用误判为全部可用。
- `ghproxy.net` 适合 `clone/pull/fetch`，不适合认证 `push`。
- 不要在日志、终端输出或文档中保存 GitHub token。
- Windows `hosts` 写入需要管理员权限；优先生成可回滚脚本。

## 诊断 GitHub 访问

优先检查：

```powershell
Resolve-DnsName github.com -Type A
curl.exe -I --connect-timeout 10 --max-time 25 https://github.com
curl.exe -I --connect-timeout 10 --max-time 25 https://api.github.com
curl.exe -I --connect-timeout 10 --max-time 25 https://raw.githubusercontent.com
```

如果 `github.com` 解析到某个 IP 后超时，用 `--resolve` 测试候选 IP：

```powershell
curl.exe -4 -I --http1.1 --connect-timeout 10 --max-time 25 --resolve github.com:443:140.82.114.4 https://github.com
```

这次经验里，`api.github.com` 和 `raw.githubusercontent.com` 可用，但 `github.com` 主站直连有抖动。不能只靠 hosts 保证主站稳定。

## 配置 Git 访问

如果系统没有 `git`，可以使用 Codex 自带 Git，并放一个 `git.cmd` 到 `C:\Users\<user>\.local\bin`：

```bat
@echo off
"C:\Users\<user>\.cache\codex-runtimes\codex-primary-runtime\dependencies\native\git\cmd\git.exe" %*
```

配置 `clone/pull/fetch` 走 ghproxy：

```powershell
git config --global url.https://ghproxy.net/https://github.com/.insteadOf https://github.com/
git config --global url.https://ghproxy.net/https://raw.githubusercontent.com/.insteadOf https://raw.githubusercontent.com/
```

验证 rewrite：

```powershell
git ls-remote --get-url https://github.com/git/git.git
git ls-remote https://github.com/git/git.git HEAD
```

如果要 `push`，不要走 ghproxy。为仓库单独设置 push URL：

```powershell
git remote set-url origin https://github.com/<owner>/<repo>.git
git remote set-url --push origin https://<owner>@github.com/<owner>/<repo>.git
```

## GitHub 授权与创建仓库

优先使用 Git Credential Manager 登录：

```powershell
git credential-manager github login
git credential-manager github list
```

检查当前凭据时不要打印完整 token。可只确认账号：

```powershell
git credential-manager github list
```

如果网页登录不通，但 `api.github.com` 可用，可以读取 GCM 凭据或让用户提供临时 token，然后调用 GitHub REST API 创建仓库。

创建仓库后，常规推送流程：

```powershell
git init -b main
git add .
git commit -m "Initial knowledge base structure"
git remote add origin https://github.com/<owner>/<repo>.git
git remote set-url --push origin https://<owner>@github.com/<owner>/<repo>.git
git push -u origin main
```

如果 Git 提示缺少作者，只在当前仓库配置即可：

```powershell
git config user.name "<owner>"
git config user.email "<owner>@users.noreply.github.com"
```

## hosts 直连

写 hosts 前先备份，并用标记块包裹：

```text
# BEGIN CODEX GITHUB ACCESS
140.82.114.4 github.com
140.82.114.4 www.github.com
140.82.116.6 api.github.com
185.199.108.133 raw.githubusercontent.com
# END CODEX GITHUB ACCESS
```

写入后刷新 DNS：

```powershell
ipconfig /flushdns
```

保留回滚脚本，删除同一个标记块即可。

## 项目迁移到 C:\project

迁移前检查目标路径：

```powershell
Test-Path C:\project\<repo>
git -C <old-path> status --short --branch
```

如果 `Move-Item` 因进程占用失败，先复制到目标路径并验证：

```powershell
Copy-Item -LiteralPath <old-path> -Destination C:\project\<repo> -Recurse -Force
git -C C:\project\<repo> status --short --branch
git -C C:\project\<repo> remote -v
```

旧目录为空但被占用时，不要强行破坏进程；等相关 PowerShell/Codex 进程退出或重启后删除。

## 安装 Chrome

优先用 `winget`：

```powershell
winget install --id Google.Chrome --exact --silent --accept-package-agreements --accept-source-agreements
```

验证：

```powershell
winget list --id Google.Chrome --exact
(Get-Item "C:\Program Files\Google\Chrome\Application\chrome.exe").VersionInfo
```

## 完成标准

- `git ls-remote https://github.com/git/git.git HEAD` 成功。
- 目标仓库在 GitHub API 返回 `200 OK`。
- 本地 `main` 跟踪 `origin/main`。
- 项目实际位置在 `C:\project\<repo>`。
- `git remote -v` 中 fetch 可走 ghproxy，push 直连 GitHub。
