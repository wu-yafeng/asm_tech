# 作弊软件是如何工作的

排除掉反作弊的情况，一个作弊软件是如何工作的？首先需要知道一个正常的软件程序是如何运作的。

1. 编写业务代码
2. 编译软件
3. 发布给客户运行

而客户手上收到的软件，大致的结构应该如下

- Main.exe
- Core.dll
- Config.ini

一般来说，现代软件启动后，会加载所需的依赖性dll，然后读取配置运行代码。当然，假设这个软件足够简单，仅仅是一个单线程的控制台程序，dll中包含了向用户问好的代码。

## Core.dll中的代码

``` cpp

auto g_Message = (char*)"Hello, I Received your message.";

char* SayHello(const char* message)
{
    return g_Message;
}

```

## main.exe中的代码

``` cpp

typedef char*(__stdcall* SayHelloProc)(const char* message);

void main()
{
    auto handle = LoadLibrary(".\\Core.dll");
    
    auto SayHello = (SayHelloProc)GetProcAddress(handle, "SayHello");
    
    while(getchar())
    {
        printf(SayHello(""));
    }
}

```

现在，我们需要实现一个作弊程序，当调用SayHello("") 的时候，需要返回 "Hello From Cheat.\n"; 如果我们可以修改源代码并重新编译，这很容易做到，但大部分情况我们没办法修改源代码。
一个办法是让Main.exe加载我们恶意的dll，在这个恶意的dll中修改g_Message变量的值。

> 作弊程序的一种类别，就是修改目标程序的内存数据。

## 如何获取g_Message变量

可以通过CE、x64dbg等工具，找到g_Message的地址。对于此类全局变量，地址可能是 0x12345;

访问它的方式是

``` cpp

auto lpMsg = (char**)0x12345;

*lpMsg = "Msg By Cheat";

```

对于复杂的数据，获取我们需要一些新的方法，当然此处足够简单。

## 如何让目标程序加载我们恶意模块

注入的方法有很多，这包含了远程线程注入、劫持注入、APC注入。
