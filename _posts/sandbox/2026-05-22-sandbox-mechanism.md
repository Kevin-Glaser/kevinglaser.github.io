---
title: Agent沙箱机制解析
date: 2026-05-22
description: 浅析Claude Code中沙箱机制
tags: [agent, ai, architecture]
layout: post
---

# Claude Code 沙箱（Sandbox）机制技术解析

## 一、概述

Claude Code 是 Anthropic 推出的 AI 编程助手命令行工具。该工具在执行过程中会代理用户运行 Shell 命令，这引入了一个核心安全命题：如何确保 AI 代理执行的命令不会对宿主操作系统造成不可控的影响。为此，Claude Code 实现了一套完整的沙箱隔离机制，从文件系统、网络访问、命令执行三个维度对 AI 代理的行为进行约束。

该沙箱机制的核心依赖是外部包 `@anthropic-ai/sandbox-runtime`，项目通过适配层（Adapter Layer）`sandbox-adapter.ts` 将其与 Claude Code 自身的设置系统、权限系统和工具链进行桥接。

## 二、底层沙箱技术

Claude Code 的沙箱并非自行实现操作系统级别的隔离原语，而是基于各平台已有的成熟沙箱技术构建：

**macOS 平台**使用 Apple 原生的 `sandbox-exec` 工具。该工具是 macOS 内置的沙箱执行器，基于 Seatbelt 框架（Apple 的强制访问控制实现），通过声明式的策略配置文件（Scheme）来约束进程的文件系统访问、网络连接和进程间通信。

**Linux 和 WSL2 平台**使用 Bubblewrap（`bwrap`）配合 `socat`。Bubblewrap 是 Flatpak 项目开发的轻量级沙箱工具，基于 Linux 内核的 Namespace（命名空间）和 Seccomp-BPF（安全计算模式-伯克利包过滤器）实现进程隔离。`socat` 在此用于网络代理的端口转发。在 Linux 上安装依赖的命令为 `apt install bubblewrap socat`。

**Windows 原生平台和 WSL1 不支持沙箱**。在 `PowerShellTool.tsx` 中，Windows 原生平台的 `shouldUseSandbox` 直接返回 `false`。WSL1 由于缺少必要的内核特性支持，同样无法运行沙箱。

平台支持检测由 `BaseSandboxManager.isSupportedPlatform()` 完成，依赖检测由 `BaseSandboxManager.checkDependencies()` 完成。两者均使用 `memoize` 进行缓存，避免重复计算。

## 三、隔离维度

### 3.1 文件系统隔离

文件系统隔离是沙箱最核心的防护维度。Claude Code 通过白名单与黑名单相结合的方式控制沙箱内进程对文件系统的读写访问。

**写入控制**方面，`allowWrite` 列表定义允许写入的路径，默认包含当前工作目录（`.`）和 Claude 临时目录。`denyWrite` 列表定义禁止写入的路径，系统自动将以下路径加入拒绝写入列表：

- 所有设置文件（`settings.json`、`settings.local.json` 等）。阻止写入设置文件是防止沙箱逃逸的关键措施：如果沙箱内的进程能够修改设置文件，就可以将 `sandbox.enabled` 设为 `false`，从而在后续命令中绕过沙箱。
- `.claude/skills` 目录。Skills 与 Commands 具有同等的权限级别（自动发现、自动加载、完整的 Claude 能力），因此需要与 `.claude/commands` 和 `.claude/agents` 一样受到操作系统级别的保护。
- Git bare repository 相关文件（`HEAD`、`objects`、`refs`、`hooks`、`config`）。这是一项针对 Git 仓库逃逸攻击的防护措施，后文将详细说明。

**读取控制**方面，`denyRead` 列表定义禁止读取的路径，`allowRead` 列表在 `denyRead` 区域内重新开放读取权限，实现细粒度的访问控制。

路径解析遵循 Claude Code 的约定：`//path` 表示从文件系统根目录开始的绝对路径，`/path` 表示相对于设置文件所在目录的路径，`~/path` 和相对路径则直接传递给 `sandbox-runtime` 处理。

### 3.2 网络隔离

网络隔离控制沙箱内进程对外部网络的访问。配置项包括：

- `allowedDomains`：允许访问的域名白名单。域名来源包括 `sandbox.network.allowedDomains` 设置和 `WebFetch(domain:...)` 权限规则。
- `deniedDomains`：禁止访问的域名黑名单。
- `allowManagedDomainsOnly`：企业策略开关。启用后，仅 `policySettings` 中定义的域名白名单生效，用户级、项目级、本地级设置中的域名配置将被忽略。这确保了企业环境下的网络策略不被用户配置绕过。
- `allowLocalBinding`：是否允许本地端口绑定。
- `httpProxyPort` / `socksProxyPort`：HTTP 代理和 SOCKS 代理端口配置，用于企业网络环境下的流量审计。
- `allowUnixSockets`：macOS 上的 Unix Domain Socket 路径白名单。Linux 上由于 Seccomp-BPF 无法按路径过滤 Unix Socket，此配置被忽略。
- `allowAllUnixSockets`：完全放行所有 Unix Socket 连接，同时禁用两个平台上的 Socket 阻断逻辑。
- `enableWeakerNetworkIsolation`：macOS 专用，允许沙箱访问 `com.apple.trustd.agent`。Go 语言编写的 CLI 工具（如 `gh`、`gcloud`、`terraform`）在使用 HTTP 代理和自定义 CA 进行 TLS 证书验证时需要此访问权限。启用此选项会降低安全性，因为它打开了通过 `trustd` 服务进行数据外泄的潜在通道。

当沙箱内的命令尝试访问未授权的网络域名时，系统会通过 `SandboxPermissionRequest` 组件向用户弹出交互式权限请求，用户可以选择"允许本次"、"允许且不再询问"或"拒绝"。在 `allowManagedDomainsOnly` 启用时，"允许且不再询问"选项被隐藏，防止用户将非托管域名持久化到设置中。

### 3.3 命令执行控制

命令执行控制决定哪些命令需要在沙箱内运行，哪些可以豁免。核心逻辑位于 `shouldUseSandbox.ts`。

判定流程如下：首先检查 `SandboxManager.isSandboxingEnabled()`，如果沙箱功能未启用，直接返回 `false`。然后检查 `dangerouslyDisableSandbox` 参数，如果调用方显式禁用沙箱且策略允许非沙箱命令（`areUnsandboxedCommandsAllowed()` 返回 `true`），则返回 `false`。接着检查命令是否包含用户配置的排除命令（`excludedCommands`），如果匹配则返回 `false`。其余情况返回 `true`，命令将在沙箱内执行。

`excludedCommands` 是用户便利功能而非安全边界。代码注释中明确指出："It is not a security bug to be able to bypass excludedCommands -- the sandbox permission system (which prompts users) is the actual security control." 即真正的安全控制是沙箱权限系统（向用户弹出确认提示），排除命令列表仅为减少误拦截的便利机制。

排除命令的匹配支持三种模式：前缀匹配（`bazel:*` 匹配 `bazel` 及其子命令）、精确匹配（`npm` 仅匹配 `npm` 本身）和通配符匹配（`docker*` 匹配所有以 `docker` 开头的命令）。匹配过程会对复合命令进行拆分（如 `docker ps && curl evil.com` 会被拆分为两个子命令分别检查），并对环境变量前缀和安全包装器（如 `timeout`、`nice`）进行迭代剥离，确保 `FOO=bar timeout 30 bazel run` 也能正确匹配 `bazel:*` 规则。

## 四、安全策略层级

Claude Code 的设置系统具有优先级层级，从高到低为：

```
flagSettings > policySettings > projectSettings > localSettings
```

`flagSettings` 来自命令行标志，`policySettings` 由企业管理员配置，`projectSettings` 存储在项目目录的 `.claude/` 下，`localSettings` 存储在用户主目录下。

与沙箱相关的关键策略控制项包括：

- `sandbox.enabled`：是否启用沙箱。
- `sandbox.autoAllowBashIfSandboxed`：启用后，在沙箱内执行的 Bash 命令自动获得许可，无需用户逐次确认。这是安全性与可用性的平衡：既然命令已在沙箱约束下运行，其影响范围已被限制，因此可以减少交互提示。
- `sandbox.allowUnsandboxedCommands`：是否允许通过 `dangerouslyDisableSandbox` 参数绕过沙箱。设为 `false` 时，该参数被完全忽略，所有命令必须在沙箱内运行。这是企业部署中的硬性安全门控。
- `sandbox.failIfUnavailable`：当沙箱已启用但无法启动时（缺少依赖、不支持的平台），是否报错退出。设为 `true` 时，系统会在启动阶段直接失败，而非静默降级为无沙箱运行。这修复了一个安全陷阱：用户配置了 `allowedDomains` 期望得到网络隔离保障，但由于依赖缺失导致沙箱未生效，用户对此毫不知情。
- `sandbox.enabledPlatforms`：未公开文档化的企业设置，限制沙箱仅在指定平台上生效。例如设置为 `["macos"]` 时，Linux 和 WSL 上的沙箱功能将被禁用。该设置最初为 NVIDIA 的企业部署而添加，他们希望在 macOS 上先启用 `autoAllowBashIfSandboxed`，待 Linux/WSL 的沙箱支持更成熟后再扩展。

`areSandboxSettingsLockedByPolicy()` 函数检测沙箱设置是否被高优先级配置锁定。当 `flagSettings` 或 `policySettings` 中显式设置了沙箱相关配置时，本地修改将无法生效，`/sandbox` 命令也会提示用户设置被覆盖。

## 五、命令执行流程

当用户或 AI 代理触发一条 Shell 命令时，执行流程如下：

1. **沙箱判定**：`shouldUseSandbox()` 根据前述逻辑判断命令是否需要在沙箱内执行。
2. **命令包装**：如果需要沙箱，`Shell.ts` 调用 `SandboxManager.wrapWithSandbox()` 将原始命令包装为沙箱执行命令。该函数内部委托给 `BaseSandboxManager.wrapWithSandbox()`，由 `sandbox-runtime` 包根据平台生成对应的 `bwrap` 或 `sandbox-exec` 命令行。
3. **临时目录创建**：为沙箱进程创建独立的临时目录，权限设为 `0o700`（仅所有者可访问），防止多用户环境下的权限冲突。
4. **进程启动**：使用 `child_process.spawn()` 启动包装后的命令。对于沙箱内的 PowerShell 命令，由于 `wrapWithSandbox` 会将命令格式化为 `<binShell> -c '<cmd>'`，而 `pwsh` 的 `-NoProfile -NonInteractive` 参数在沙箱内会丢失，因此采用特殊处理：先由 `powershellProvider.buildExecCommand` 将命令编码为 Base64 格式（`pwsh -NoProfile -NonInteractive -EncodedCommand <base64>`），再以 `/bin/sh` 作为沙箱的内层 Shell 来执行该调用。
5. **输出收集**：命令执行过程中的 stdout 和 stderr 被实时收集。`O_NOFOLLOW` 标志防止沙箱内的符号链接跟随攻击。
6. **清理**：命令执行完毕后，`SandboxManager.cleanupAfterCommand()` 执行清理操作，包括删除沙箱临时文件和检查后置安全条件。

## 六、初始化与配置动态更新

沙箱的初始化在 REPL 启动时通过 `SandboxManager.initialize()` 完成。初始化过程包括：

1. 检测当前是否处于 Git Worktree 中，如果是，缓存主仓库路径以供后续配置使用。
2. 调用 `convertToSandboxRuntimeConfig()` 将 Claude Code 的设置格式转换为 `sandbox-runtime` 所需的 `SandboxRuntimeConfig` 格式。
3. 调用 `BaseSandboxManager.initialize()` 启动底层沙箱运行时，并注册网络权限请求回调。
4. 订阅设置变更事件（`settingsChangeDetector.subscribe`），当设置发生变更时，自动重新计算配置并调用 `BaseSandboxManager.updateConfig()` 更新沙箱运行时，无需重启会话。

初始化使用 Promise 缓存模式防止并发问题：`initializationPromise` 在首次调用时同步创建，后续调用直接返回该 Promise。如果初始化失败，Promise 被清除以允许重试。

`refreshConfig()` 函数提供同步的配置刷新能力，用于权限更新后立即生效，避免竞态条件下新请求使用过期配置。

## 七、安全攻防设计

### 7.1 设置文件写入防护

沙箱内进程被禁止写入所有 `settings.json` 文件。这是防止沙箱逃逸的第一道防线。如果攻击者能修改设置文件，就可以关闭沙箱或修改权限规则，使后续命令脱离约束。

### 7.2 Git Bare Repository 逃逸防护

Git 的 `is_git_directory()` 函数会将包含 `HEAD`、`objects/`、`refs/` 的目录识别为 bare repository。攻击者可以在沙箱内创建这些文件，并设置 `core.fsmonitor` 配置指向恶意脚本。当 Claude 在沙箱外执行 Git 命令时，恶意 hook 将以完整权限运行，从而逃逸沙箱。

防护措施分两层：对于已存在的 Git 内部文件，通过 `denyWrite` 列表将其设为只读（`sandbox-runtime` 会使用 `ro-bind` 挂载）。对于不存在的文件，在命令执行后通过 `scrubBareGitRepoFiles()` 清理可能被创建的文件，确保在非沙箱的 Git 命令运行前，恶意文件已被删除。

在 `readOnlyValidation.ts` 中还有额外的应用层防护：检测复合命令中是否同时包含"写入 Git 内部路径"和"执行 Git 命令"的操作，如果存在则强制要求用户权限确认。此外，当沙箱启用且当前目录不是原始工作目录时，Git 命令也需要额外的权限确认，因为原始工作目录受 `denyWrite` 保护，而其他目录可能被利用进行竞态攻击。

### 7.3 Skills 目录写入防护

`.claude/skills` 目录被加入 `denyWrite` 列表。Skills 具有与 Commands 相同的权限级别（自动发现、自动加载、完整的 Claude 能力），如果沙箱内进程能够写入 Skills 目录，就可以注入恶意技能定义，在后续会话中以完整权限执行。

### 7.4 Windows 平台策略阻断

在 Windows 原生平台上，由于沙箱不可用，如果企业策略要求沙箱（`sandbox.enabled: true` 且 `allowUnsandboxedCommands: false`），则 Shell 命令执行将被完全阻断。`isWindowsSandboxPolicyViolation()` 函数检测此条件，返回错误信息："Enterprise policy requires sandboxing, but sandboxing is not available on native Windows. Shell command execution is blocked on this platform by policy."

## 八、违规监控与报告

`SandboxViolationStore` 是一个发布-订阅模式的事件存储，记录所有沙箱违规事件（如尝试访问被拒绝的文件路径或网络域名）。每个违规事件包含时间戳、关联的命令和违规描述。

`SandboxViolationExpandedView` 组件实时订阅违规存储，在终端界面中显示最近 10 条违规记录和总违规计数。该组件仅在 macOS 上显示（Linux 上由于 Bubblewrap 的日志机制不同，违规信息通过 stderr 标注呈现）。

`annotateStderrWithSandboxFailures()` 函数在命令的 stderr 输出中标注沙箱违规信息，使开发者能够从命令输出中直接识别哪些操作被沙箱阻止。

`ignoreViolations` 配置允许用户按类别忽略特定的违规类型，减少噪音输出。

## 九、用户交互界面

Claude Code 提供了 `/sandbox` 命令作为沙箱管理的主入口。该命令提供以下功能：

**模式选择**：三种沙箱运行模式。Auto-allow 模式下，沙箱内的命令自动获得许可，尝试在沙箱外运行的命令回退到常规权限确认。Regular 模式下，所有命令都需要常规权限确认。Disabled 模式完全关闭沙箱。

**依赖检查**：显示当前平台的沙箱依赖状态，包括缺失的依赖项和警告信息。

**覆盖配置**：管理 `excludedCommands` 等覆盖设置。

**配置查看**：显示当前生效的完整沙箱配置。

`/sandbox exclude <pattern>` 子命令可将指定命令模式添加到排除列表，添加后的配置持久化到 `localSettings` 中。

`/doctor` 命令也包含沙箱诊断信息，`getSandboxUnavailableReason()` 函数在启动时检测沙箱是否可用，如果用户已启用沙箱但依赖缺失或平台不支持，会给出明确的错误提示和修复建议。

## 十、接口设计

`ISandboxManager` 接口定义了沙箱管理器的完整公共 API，包含以下方法分组：

**生命周期管理**：`initialize()`、`reset()`、`cleanupAfterCommand()`

**状态查询**：`isSandboxingEnabled()`、`isSandboxEnabledInSettings()`、`isSupportedPlatform()`、`isPlatformInEnabledList()`、`isSandboxRequired()`、`areSandboxSettingsLockedByPolicy()`

**依赖检查**：`checkDependencies()`、`getSandboxUnavailableReason()`

**权限控制**：`isAutoAllowBashIfSandboxedEnabled()`、`areUnsandboxedCommandsAllowed()`

**配置管理**：`setSandboxSettings()`、`refreshConfig()`、`getExcludedCommands()`

**隔离配置查询**：`getFsReadConfig()`、`getFsWriteConfig()`、`getNetworkRestrictionConfig()`

**命令包装**：`wrapWithSandbox()`

**违规处理**：`getSandboxViolationStore()`、`annotateStderrWithSandboxFailures()`

**平台特定**：`getAllowUnixSockets()`、`getAllowLocalBinding()`、`getProxyPort()`、`getSocksProxyPort()`、`getLinuxHttpSocketPath()`、`getLinuxSocksSocketPath()`、`getLinuxGlobPatternWarnings()`

该接口通过 `SandboxManager` 单例对象导出，自定义方法在 `sandbox-adapter.ts` 中实现，通用方法委托给 `BaseSandboxManager`（来自 `@anthropic-ai/sandbox-runtime`）。

## 十一、设计总结

Claude Code 的沙箱机制体现了纵深防御（Defense in Depth）的安全设计原则。操作系统级别的隔离（`sandbox-exec` / Bubblewrap）提供硬性边界，应用层的权限系统提供用户交互式确认，设置文件的写入防护和 Git 逃逸防护封堵已知的逃逸路径，企业策略层级确保管理员的配置不被用户覆盖。

该机制的局限性在于 Windows 原生平台的不支持。在 Windows 上，安全防护完全依赖应用层的权限确认系统，缺乏操作系统级别的隔离保障。对于需要强制沙箱的企业部署，Windows 平台上的 Shell 命令执行会被策略阻断，用户需通过 WSL2 来获得完整的沙箱支持。
