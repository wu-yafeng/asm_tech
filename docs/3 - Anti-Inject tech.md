# 反作弊的对抗(反注入)

目前主流的游戏，都有反作弊。假设我们是一个反作弊提供商，如何来对抗作弊程序？

> 本文不针对具体的反作弊程序，而是几乎所有反作弊程序的一般性的通用解决方案。

假设现在有一款作弊程序，使用了APC注入了我们的游戏进程。现在我们需要对抗这类恶意代码。如何实现？

### OpenProcess降权

很显然，所有作弊程序都需要使用到 OpenProcess(...) 函数来获取对指定进程访问的句柄，我们是否可以通过hook这个函数，来避免作弊程序工作？

hook后我们该做什么。

要知道，除了作弊程序以外，许多应用程序依然需要获取游戏进程句柄等信息。设想一个开发商开发了一款检测进程内存占用量的程序，用于提示用户计算机的运行状态。所以我们的hook函数应该考虑到兼容性
问题。

下面是OpenProcess的原型

``` cpp
HANDLE OpenProcess(
DWORD dwDesiredAccess,
BOOL bInheritHandle,
DWORD dwProcessId
);

```

第一个参数是目标进程的访问权限，第三个是进程ID。通常，大部分程序都不会需要极高的权限。我们可以通过判断 `dwProcessId` 是否是游戏进程，然后通过重写 `dwDesiredAccess` 来避免作弊
程序拿到过高的权限。

### 作弊升级

OpenProcess的调用链是

用户层代码: kernel32.dll!OpenProcess->ntdll!NtOpenProcess->内核层
内核层：nt!KiSystemCall64->nt!NtOpenProcess->nt!PsOpenProcess->nt!PspOpenProcess->nt!ObOpenObjectByPointer->.....

由于反作弊是在用户层的kernel32.dll!OpenProcess进行了降权，我们可以通过调用更底层的ntdll!NtOpenProcess方法来获取到正确权限的句柄。

### 对抗螺旋升级

反作弊很快意识到这种问题，便开始在 ntdll!NtOpenProcess 实现了hook。而反作弊也开始升级，通过开发内核驱动程序调用内核的nt!NtOpenProcess的函数实现获取句柄。

而反作弊继续升级，hook内核函数nt!NtOpenProcess来进行降权。然后作弊程序继续升级，使用更底层的函数PsOpenProcess。但是请等等，反作弊当hook PsOpenProcess的时候遇到了问题——部分玩
家反馈游戏启动后会崩溃甚至电脑蓝屏。

为什么会出现这种情况？因为nt!NtOpenProcess后的所有调用，已经属于微软未公开的函数了，这意味着微软不总是保证这些api的兼容性，也不建议使用这些api。在某些系统版本里，这些api的行为、参
数都可能发生break-changes。至此，反作弊已经无法对抗作弊程序（因为游戏需要保证稳定性）。但是作弊程序的开发难度，已经提升了许多，同样因为作弊程序使用了未公开的函数，在一些系统下
作弊程序变得无法工作。

### 反作弊继续升级

使用同样的方法，反作弊尽可能hook了所有作弊程序使用的关键api。这些api包括：

- 针对打开进程的NtOpenProcess的保护
- 针对读内存的NtReadVirtualMemory的保护
- 针对写内存的NtWriteVirtualMemory的保护
- 针对打开线程的NtOpenThread的保护


### 作弊恢复hook

作弊程序除了调用更底层的函数以外，还可以通过移除反作弊的hook实现作弊。对此，反作弊可以通过hook后，建立多个线程CRC校验这些函数的机器码，确保这些机器码在hook后没有被恢复。同样，
反作弊程序可以通过代码混淆、花指令、多重保护来避免作弊程序攻击反作弊程序自身。
