# DeepSeek Harness (DSH) 批量部署指南

> 适用环境：Windows 10/11，需管理员权限（装 Git、全局 npm 包）
> 网络假设：国内环境，GitHub 直连受阻，需走镜像/代理

---

## 1. 准备工作

| 项目 | 版本/来源 | 说明 |
|------|-----------|------|
| Node.js | ≥22.19.0 或 ≥24.0.0 | 官网或 `winget install OpenJS.NodeJS.LTS` |
| Git | 2.55+ | 华为镜像离线包（见下） |
| pnpm | 11.7.0 | 与仓库 `packageManager` 一致 |
| 目标目录 | `D:\DSH` | 必须**已存在且为空** |

---

## 2. 离线/半离线安装包准备（可选，用于完全隔网环境）

| 文件 | 下载地址（需外网机器预下） | 备注 |
|------|---------------------------|------|
| Git-2.55.0.3-64-bit.exe | `https://mirrors.huaweicloud.com/git-for-windows/v2.55.0.windows.3/Git-2.55.0.3-64-bit.exe` | 65 MB |
| pnpm 11.7.0 tarball | `https://mirrors.huaweicloud.com/repository/npm/pnpm/-/pnpm-11.7.0.tgz` | 由 `npm pack pnpm@11.7.0` 得到 |
| DSH 源码 zip | `https://codeload.github.com/deepseek-ai/deepseek-harness/zip/refs/heads/master` | 180 MB+，或用 `git clone`（需代理） |

> 完全隔网时：把以上三个文件拷贝到目标机，按“3.1/3.2/3.3 离线版”执行。

---

## 3. 标准部署步骤（有外网、可走镜像/代理）

### 3.1 安装 Git（华为镜像，静默）
```bat
REM 需管理员 PowerShell
$url = "https://mirrors.huaweicloud.com/git-for-windows/v2.55.0.windows.3/Git-2.55.0.3-64-bit.exe"
$out = "$env:TEMP\Git-2.55.0.3-64-bit.exe"
curl.exe -L -o $out $url
Start-Process -FilePath $out -ArgumentList "/VERYSILENT","/NORESTART","/NOCANCEL","/SP-" -Wait
```
验证：`git --version` → `2.55.0.windows.3`

### 3.2 克隆仓库（走 gh-proxy，校验哈希）
```bat
REM 普通 PowerShell 即可
$env:GIT_TERMINAL_PROMPT = "0"
git clone --progress "https://gh-proxy.com/https://github.com/deepseek-ai/deepseek-harness.git" "D:\DSH"
```
校验（可选但推荐）：
```powershell
$local = git -C D:\DSH rev-parse HEAD
$api   = (irm "https://api.github.com/repos/deepseek-ai/deepseek-harness/commits/master").sha
if ($local -ne $api) { throw "HASH MISMATCH" }
```
把 origin 改回官方并配置自动走代理：
```bat
git -C D:\DSH remote set-url origin https://github.com/deepseek-ai/deepseek-harness.git
git -C D:\DSH config url."https://gh-proxy.com/https://github.com/".insteadOf "https://github.com/"
```

### 3.3 安装 pnpm 11.7.0（华为 npm 镜像）
```bat
npm install -g pnpm@11.7.0 --registry=https://mirrors.huaweicloud.com/repository/npm/ --force
```
验证：`pnpm --version` → `11.7.0`

### 3.4 安装依赖 & 构建
```bat
cd /d D:\DSH
set "npm_config_registry=https://mirrors.huaweicloud.com/repository/npm/"
set "ELECTRON_MIRROR=https://mirrors.huaweicloud.com/electron/"
set "ELECTRON_BUILDER_BINARIES_MIRROR=https://mirrors.huaweicloud.com/electron-builder-binaries/"

pnpm install
pnpm run build
```
预计 6–15 分钟（视网络/磁盘）。成功标志：`apps/cli/lib/bin.js` 存在、`pnpm dsh --help` 正常输出。

### 3.5 验证可运行
```bat
pnpm dsh web --no-open
```
日志会打印 `dsh web: http://127.0.0.1:3080/?token=xxx`，浏览器访问该 URL 即可。

---

## 4. 一键启动脚本（随仓库分发）

已生成 `D:\DSH\start-dsh-web.bat`，双击即可启动（含镜像环境变量、报错暂停）。

```bat
@echo off
chcp 65001 >nul
title DeepSeek Harness Web
cd /d "%~dp0"
set "PATH=%APPDATA%\npm;C:\Program Files\Git\cmd;%PATH%"
set "npm_config_registry=https://mirrors.huaweicloud.com/repository/npm/"
set "ELECTRON_MIRROR=https://mirrors.huaweicloud.com/electron/"
echo Starting DeepSeek Harness Web UI ...
echo.
call pnpm dsh web %*
if errorlevel 1 pause
```

---

## 5. 批量部署自动化（PowerShell 脚本）

将下面脚本保存为 `Deploy-DSH.ps1`，以管理员运行：

```powershell
<#
.SYNOPSIS
    一键部署 DeepSeek Harness 到 D:\DSH
.NOTES
    需管理员权限（装 Git、全局 pnpm）
    网络需能访问：mirrors.huaweicloud.com、gh-proxy.com、api.github.com
#>

$ErrorActionPreference = 'Stop'
$target = 'D:\DSH'
$gitUrl = 'https://mirrors.huaweicloud.com/git-for-windows/v2.55.0.windows.3/Git-2.55.0.3-64-bit.exe'
$gitExe = "$env:TEMP\Git-2.55.0.3-64-bit.exe"

function Ensure-Git {
    if (Get-Command git -ea 0) { return }
    Write-Host "安装 Git..." -Fore Cyan
    curl.exe -L -o $gitExe $gitUrl
    Start-Process -FilePath $gitExe -ArgumentList "/VERYSILENT","/NORESTART","/NOCANCEL","/SP-" -Wait
    $env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ";" + [Environment]::GetEnvironmentVariable('Path','User')
}

function Clone-Repo {
    if (Test-Path "$target\.git") { return }
    Write-Host "克隆仓库..." -Fore Cyan
    $env:GIT_TERMINAL_PROMPT = '0'
    git clone --progress "https://gh-proxy.com/https://github.com/deepseek-ai/deepseek-harness.git" $target
    # 校验并配代理
    $local = git -C $target rev-parse HEAD
    $api   = (irm "https://api.github.com/repos/deepseek-ai/deepseek-harness/commits/master").sha
    if ($local -ne $api) { throw "HASH 校验失败" }
    git -C $target remote set-url origin https://github.com/deepseek-ai/deepseek-harness.git
    git -C $target config url."https://gh-proxy.com/https://github.com/".insteadOf "https://github.com/"
}

function Ensure-Pnpm {
    if (pnpm --version -ea 0) { return }
    Write-Host "安装 pnpm..." -Fore Cyan
    npm install -g pnpm@11.7.0 --registry=https://mirrors.huaweicloud.com/repository/npm/ --force
}

function Install-And-Build {
    Write-Host "安装依赖 & 构建..." -Fore Cyan
    $env:npm_config_registry = 'https://mirrors.huaweicloud.com/repository/npm/'
    $env:ELECTRON_MIRROR     = 'https://mirrors.huaweicloud.com/electron/'
    $env:ELECTRON_BUILDER_BINARIES_MIRROR = 'https://mirrors.huaweicloud.com/electron-builder-binaries/'
    Push-Location $target
    pnpm install
    pnpm run build
    Pop-Location
}

function Verify {
    Write-Host "验证..." -Fore Cyan
    Push-Location $target
    pnpm dsh --version
    Pop-Location
}

# ---- Main ----
Ensure-Git
Clone-Repo
Ensure-Pnpm
Install-And-Build
Verify
Write-Host "`n部署完成！双击 $target\start-dsh-web.bat 即可启动。" -Fore Green
```

---

## 6. 常见问题 & 变通

| 现象 | 原因 | 解决 |
|------|------|------|
| `git clone` 超时/报错 | 直连 github.com 被墙 | 必须用 `gh-proxy.com/` 或其他可用代理前缀 |
| `npm install -g pnpm` 极慢/损坏 | 官方 registry 慢/镜像 tarball 坏 | 改用华为/腾讯 npm 镜像（文中已配置） |
| `electron` postinstall 卡住 | 下载二进制走 GitHub releases | 已设 `ELECTRON_MIRROR` 指向华为镜像 |
| `node-pty` 编译报错 | 缺 VS Build Tools | 仓库已打补丁走预编译 `conpty.dll`，无需 VS |
| 启动后浏览器 401 | 未带 token 访问 | 终端会打印带 token 的完整 URL，复制完整链接访问 |
| 多机器同步配置 | `$DSH_HOME` 默认在 `%USERPROFILE%\.dsh` | 可设 `DSH_HOME=D:\DSH\data` 统一放在仓库目录 |

---

## 7. 目录结构速览（部署后）

```
D:\DSH
├── .git/                  # 含 insteadOf 代理配置
├── apps/cli/lib/bin.js    # CLI 入口（构建产物）
├── node_modules/          # 依赖（~2–3 GB）
├── start-dsh-web.bat      # 一键启动脚本
├── Deploy-DSH.ps1         # 本自动化脚本（可拷贝到其它机器）
└── DEPLOY.md              # 本文件
```

---

## 8. 升级/更新

```bat
cd /d D:\DSH
git pull                    # 自动走 gh-proxy
pnpm install --frozen-lockfile
pnpm run build
```

---

> 维护者：根据实际镜像可用性调整 `gh-proxy.com`、`mirrors.huaweicloud.com` 等域名。
> 如需完全离线，请按第 2 节预下载三个核心文件并在内网分发。