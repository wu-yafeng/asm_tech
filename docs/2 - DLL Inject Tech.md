# DLL注入的方法

这些方法近乎是所有作弊程序使用的基础思路。但实际的代码实现方法并非如本文所述，这是因为反作弊也在不断的更新和进步。

## APC注入

在一个进程执行过程中、调用Sleep或者WaitForSingleObject函数时、系统会产生软中断。这时候线程在次被唤醒时、此线程会首先执行apc队列中被注册的函数、
利用QueueUserAPC这个函数、并且去执行我们的dll加载代码 ，进而完成我们注入dll的目的。 


下面是伪代码

``` cpp

// cheat.exe

auto g_shellcode_x86 = new char[xxx] { xxx };

int main()
{
    auto addr = WriteProcessMem(g_shellcode_x86);
    auto hThread = GetGameThreadHandle();

    QueueUserAPC(addr, hThread, NULL);
}

```

上面的代码经过了简化，为了方便你理解这种注入方式的原理。首先我们需要实现一个 `WriteProcessMem` 方法，此方法用于向指定目标进程的内存中写入一段数据 `g_shellcode_x86` 。虽然将
此数据的在目标进程的首地址传给 `QueuUserAPC(...)`

关于 g_shellcode_x86 。这是一段编译后的0x86机器码（取决于目标进程的位数，如果是64位，则是x64的shellcode）。这段shellcode的目的是希望在目标进程执行一段函数，
用于引导作弊模块。shellcode是如下函数编译后的：

``` cpp
void bootstraper(
   ULONG_PTR Parameter
)
{
   LoadLibrary("my_cheat.dll");
}

```

简单来说，我们需要在目标进程里增加一段代码，这段代码用于加载我们的作弊模块到目标进程，这段代码是在目标进程调用Sleep或者WaitForSingleObject函数时调用的，
如果目标进程没有使用到Sleep或者WaitForSingleObject，则我们的作弊引导程序永远不会被执行。

## message注入

message注入其实就是windows消息钩子钩子注入、SetWindowsHook函数是windows消息处理机制的一个平台、应用程序可以指定监视指定窗口的某种消息。
它可以拦截进程的消息到指定的dll中导出函数、这个特性我们就可以将dll注入到程序中。


与APC注入不同，APC是通过目标进程调用Sleep或者WaitForSingleObject函数时，引导作弊模块。而windows消息钩子，则是使用windows的消息机制来引导作弊模块。下面是伪代码

