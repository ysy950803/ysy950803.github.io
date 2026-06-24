---
title: 迁移到PowerShell 7
date: 2026-06-24 16:06:52
tags:
  - 鼓捣折腾
  - Windows
---

Windows 11自带的PowerShell（就是那个默认蓝色背景的）并不是很好用，即便按照[Windows开发者优化配置](https://blog.ysy950803.top/2026/06/18/Windows%E5%BC%80%E5%8F%91%E8%80%85%E4%BC%98%E5%8C%96%E9%85%8D%E7%BD%AE/)优化了以后，还是没有达到macOS上oh-my-zsh的体验。

对于开发者来说，可以直接迁移到更先进的PS7，下面是简要的对比：

|项目|Windows 11 自带 Windows PowerShell|PowerShell 7|
| ----------| -----------------------------------------| ------------------------------------------|
|命令|​`powershell.exe`|​`pwsh.exe`|
|版本定位|Windows 内置旧版，最新是 5.1|现代版 PowerShell|
|运行时|.NET Framework，仅 Windows|新版 .NET，跨 Windows/macOS/Linux|
|更新|不再增加新功能，随 Windows 支持周期维护|持续更新|
|安装方式|系统自带|额外安装，不会替换 5.1|
|兼容性|老 Windows 管理模块最好|大多数模块可用，但极老模块可能要回到 5.1|
|开发体验|能用，但偏旧|更快、更现代、跨平台、补全体验更好|

### 清除

把旧版PS的命令历史记录清掉（强迫症首选）：

```shell
Clear-Content (Get-PSReadLineOption).HistorySavePath
```

### 安装

在旧版PowerShell中执行安装命令：

```shell
winget install --id Microsoft.PowerShell --source winget
```

如果网络有问题导致安装失败，那就到官网手动下载（MSI包的安装体验最好）：[learn.microsoft.com/zh-cn/powershell/scripting/install/install-p...](https://learn.microsoft.com/zh-cn/powershell/scripting/install/install-powershell-on-windows?view=powershell-7.6)

MSI包安装过程注意勾选如下选项，对开发者最友好：

```shell
✅ Add PowerShell to Path Environment Variable
✅ Register Windows Event Logging Manifest
⬜ Enable PowerShell remoting
✅ Disable Telemetry
✅ Add 'Open here' context menu to Explorer
✅ Add 'Run with PowerShell 7' context menu for PowerShell files
```

### 配置

安装成功后打开PS7（可以考虑把它配置成Windows默认终端），执行：

```shell
if (!(Test-Path $PROFILE)) {
    New-Item -ItemType File -Path $PROFILE -Force
}

notepad $PROFILE
```

在记事本中把下面内容加进去：

```shell
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView

Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
Set-PSReadLineKeyHandler -Key UpArrow -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow -Function HistorySearchForward
```

实现效果：

- ​`Tab`：直接显示候选菜单，而不是一个个循环。
- ​`Ctrl + Space`：也可打开补全菜单。
- ​`↑ / ↓`：按当前输入前缀搜索历史命令。
- ​`F2`：切换预测补全的 InlineView / ListView。
- ​`→`：接受内联预测。

到这里其实就已经有很好的体验了，但如果你还有更进阶的需求，请看下文。

### 进阶

> 关于自动补全和命令预测，如何实现像macOS上oh-my-zsh那样可以通过tab来补全命令的功能？而不是仅靠用户自己的历史来补全。举例说就是，当我第一次使用adb的时候，在oh-my-zsh里，就可以在输入adb ins之后直接tab补全为adb install，感觉它读取了adb的所有支持命令，在PowerShell 7中如何实现？

结论：你要的不是 **Predictive IntelliSense/历史预测**，而是 **native command argument completion**。PowerShell 7 可以做到，但需要给 `adb`​ 这类外部命令注册补全器；它不会自动“理解”所有 `.exe`​ 的子命令。微软文档也明确区分了：预测补全主要来自历史或插件，而 `Tab`​ 触发的是 completion；如果你按 `Tab`，预测建议会被忽略，进入补全候选。

#### 最推荐方案：装 [Carapace](https://github.com/carapace-sh/carapace-bin)

它最接近 oh-my-zsh 的体验：一个工具给很多 CLI 提供补全定义。Carapace 官方说明它为多个 CLI 命令提供 argument completion，并支持 PowerShell、Zsh、Fish、Bash 等 shell；它的补全列表里也明确包含 `adb`。

快速安装：

```shell
winget install -e --id rsteube.Carapace
```

如果网络不好就参考它的文档手动装。装好之后再次打开配置文件：

```shell
notepad $PROFILE
```

然后直接全部替换为如下内容，既兼容了**PSReadLine 配置**负责的“交互体验”，也兼容了**Carapace 配置**负责的“外部命令补全数据”：

```shell
# ==============================
# PowerShell 7 Developer Profile
# ==============================

# ---------- PSReadLine ----------
# Tab 显示菜单式补全，接近 zsh/oh-my-zsh 的体验
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete

# 根据当前输入前缀搜索历史命令
Set-PSReadLineKeyHandler -Key UpArrow -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow -Function HistorySearchForward

# 命令预测：主要来自历史记录
# 注意：这不是 adb/git/docker 子命令补全的核心，只是补充体验
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView

# 可选：减少误触 Ctrl+C 后的提示音
Set-PSReadLineOption -BellStyle None


# ---------- Carapace ----------
# 为 adb、git、docker、npm、kubectl 等外部命令提供子命令/参数补全
if (Get-Command carapace -ErrorAction SilentlyContinue) {
    # 允许 Carapace 桥接其他 shell 的补全定义
    $env:CARAPACE_BRIDGES = 'zsh,fish,bash,inshellisense'

    # 改善菜单补全选中项的显示效果
    Set-PSReadLineOption -Colors @{
        Selection = "`e[7m"
    }

    # 注册 Carapace 补全
    carapace _carapace | Out-String | Invoke-Expression
}
```

到这里就全部搞定了，爽用！

#### 题外话：为什么 PowerShell 默认不行？

PowerShell 对自己的 cmdlet 很强，因为它知道参数元数据；例如 `Get-ChildItem -Fi<Tab>`​ 可以补成 `-Filter`​。对外部程序，比如 `adb.exe`​、`git.exe`​、`docker.exe`​，PowerShell 默认只知道“这是个可执行文件”，不知道它内部有哪些子命令。微软提供的机制是 `Register-ArgumentCompleter`​，它可以为指定命令注册动态补全；文档中也专门提到 `-Native`​ 可用于原生命令，例如给 `dotnet` CLI 添加补全。

oh-my-zsh 的 `adb install`​ 那种体验，本质上也是因为有人提前写好了 zsh completion 文件。PowerShell 里对应的东西就是 ​**ArgumentCompleter**；Carapace 则相当于帮你集中维护了很多命令的 completer。

