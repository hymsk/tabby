# Tabby 开发与 Windows 构建指南

本文件适用于在 Windows x64 上修改、编译、运行和打包本仓库。执行命令前先确认位于仓库根目录。

## 基线与分支

- 上游仓库：`Eugeny/tabby`
- 开发基线：最新 `master`，并同步完整 tags。
- 当前维护分支：`hymsk-ai`
- 搜狗输入法 Shift 上字修复位于：
  `tabby-terminal/src/frontends/xtermFrontend.ts`
- 保持源码提交最小化。不要提交 `dist/`、`builtin-plugins/`、`build/vc_redist.exe`、安装包、portable 压缩包、日志、source map 或本地配置目录。

推荐使用独立 worktree 开发，避免已有构建产物和依赖影响新任务：

```powershell
git remote get-url upstream 2>$null
# 仅在 upstream 尚不存在时执行：
git remote add upstream https://github.com/Eugeny/tabby.git
git fetch upstream master --tags
git worktree add -b <branch> ..\tabby-<branch> upstream/master
Set-Location ..\tabby-<branch>
```

## Windows x64 环境

使用以下工具链：

- Node.js 22 x64
- Yarn Classic 1.x
- Python 3
- Rust stable，目标 `x86_64-pc-windows-msvc`
- Visual Studio 2022 Build Tools
  - Desktop development with C++
  - MSVC v143 x64/x86 build tools
  - Windows 10/11 SDK
  - 对应 MSVC 版本的 Spectre-mitigated libraries
- Git for Windows

开始构建前确认版本：

```powershell
node --version
npm --version
yarn --version
rustc --version
git describe --tags
```

Node.js 必须显示 `v22.x`。在 fork 或新 worktree 中先同步上游 tags，因为构建版本由 `git describe --tags` 生成。

## 初始化干净的构建会话

为当前 PowerShell 会话设置 x64，并移除会改变 Node 模块解析和原生编译目标的继承变量：

```powershell
$env:ARCH = 'x64'
$env:RUST_TARGET_TRIPLE = 'x86_64-pc-windows-msvc'
Remove-Item Env:NODE_PATH -ErrorAction SilentlyContinue
Remove-Item Env:npm_config_arch -ErrorAction SilentlyContinue
Remove-Item Env:npm_config_target_arch -ErrorAction SilentlyContinue
Remove-Item Env:npm_config_target -ErrorAction SilentlyContinue
Remove-Item Env:npm_config_runtime -ErrorAction SilentlyContinue
Remove-Item Env:npm_config_disturl -ErrorAction SilentlyContinue

rustup target add $env:RUST_TARGET_TRIPLE
npm install --global yarn node-gyp@10.2.0
$npmPrefix = npm prefix --global
npm config set node_gyp "$npmPrefix\node_modules\node-gyp\bin\node-gyp.js"
```

## 安装依赖

在仓库根目录执行官方 Windows 构建使用的安装入口：

```powershell
yarn --network-timeout 1000000
```

根目录 `postinstall` 会继续执行：

1. `patch-package`
2. `scripts/install-deps.mjs`
3. `scripts/build-native.mjs`

这些脚本会安装各内置插件依赖，并按仓库使用的 Electron 版本重建原生模块。依赖安装完成后不要从其他 checkout 复制 `node_modules`。

## 编译

完整编译：

```powershell
yarn run build
```

该命令先生成 typings，再依次编译 Electron 主进程、renderer 和各插件。

只验证终端插件修改时，可在完整依赖已安装的前提下执行：

```powershell
yarn run build:typings
yarn --cwd tabby-terminal run build
```

搜狗 Shift 修复的静态检查：

```powershell
yarn eslint tabby-terminal/src/frontends/xtermFrontend.ts
yarn tsc --noEmit --project tabby-terminal/tsconfig.json
git diff --check
```

## 开发运行

先完成完整编译。使用独立配置目录启动，避免读取或改写日常使用的 Tabby 配置：

```powershell
$configDir = Join-Path $PWD '.tabby-dev-profile'
New-Item -ItemType Directory -Force $configDir | Out-Null
$env:TABBY_CONFIG_DIRECTORY = $configDir
yarn start
```

需要生产模式 renderer 时执行：

```powershell
$env:TABBY_CONFIG_DIRECTORY = Join-Path $PWD '.tabby-prod-profile'
New-Item -ItemType Directory -Force $env:TABBY_CONFIG_DIRECTORY | Out-Null
yarn start:prod
```

结束验证后关闭本次启动的 Electron/Tabby 进程，再删除临时配置目录。

## Windows 安装包与 portable 包

先下载 Visual C++ Redistributable 到构建脚本预期位置：

```powershell
$env:ARCH = 'x64'
Invoke-WebRequest `
    -Uri 'https://aka.ms/vs/17/release/vc_redist.x64.exe' `
    -OutFile 'build/vc_redist.exe'
```

然后按顺序执行：

```powershell
yarn --network-timeout 1000000
yarn run build
node scripts/prepackage-plugins.mjs
node scripts/build-windows.mjs
```

输出位于 `dist/`：

- `tabby-<version>-setup-x64.exe`
- `tabby-<version>-portable-x64.zip`

没有配置 Tabby 官方签名密钥时，生成的是未签名构建。不得将其标记为官方 Release。

## 基于官方 portable 验证单个内置插件

当修改仅涉及 `tabby-terminal` 时，可以使用与源码 tag 相同版本、相同架构的官方 portable 作为运行基线：

1. 使用 Node.js 22 完成依赖安装、typings 和 `tabby-terminal` 编译。
2. 运行 `node scripts/prepackage-plugins.mjs` 生成可分发的 `builtin-plugins/tabby-terminal`。
3. 解压同版本官方 Windows x64 portable 到新的候选目录。
4. 将候选目录中的 `resources/builtin-plugins/tabby-terminal` 替换为刚生成的插件目录。
5. 在候选目录旁创建独立配置目录，设置 `TABBY_CONFIG_DIRECTORY` 后启动 `Tabby.exe`。
6. 确认本地 PowerShell 会话、普通键盘输入和目标输入法行为。

不要直接修改官方 portable 原始副本；每次验证都从已校验哈希的原始压缩包解压新的候选目录。

## 搜狗输入法修复验收

在 Windows 搜狗拼音下执行：

1. 打开本地 PowerShell profile。
2. 输入拼音，使候选文字保持在预编辑状态。
3. 按一次 `Shift` 切换到英文输入。
4. 确认预编辑文字提交到终端，没有消失。
5. 继续输入英文、回车和常用快捷键，确认终端输入正常。

同时验证 `event.key === 'Shift'` 与 legacy `event.keyCode === 16` 对应的处理仍保留在源码中。

## 交付前检查

```powershell
git status --short
git diff --check
git diff --stat upstream/master...HEAD
git log --oneline --decorate upstream/master..HEAD
```

交付源码分支时仅包含预期源码、测试和文档。安装包和 portable 包通过 GitHub Actions Artifact 或明确标注为非官方、未签名的 Release asset 分发，不提交到 Git 历史。
