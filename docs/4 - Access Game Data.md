# 访问游戏数据

还记得我们第一部分所讲述的,修改目标进程的全局变量吗。一旦我们注入，我们便可以访问游戏进程的变量。比如一般游戏都有一个 World (世界) 信息变量，用于访问当前世界
里的角色，怪物，道路，树木，建筑等等信息。假设我们需要访问一个游戏里当前玩家的角色的生命值信息。应该怎么做？

## 正向开发

假设我们是一个游戏开发者，我们使用到了某一个游戏引擎来开发我们的游戏。一般的，我们需要定义一个 class 来表示角色信息

``` csharp

class Player
{
    double HP;
    double MP;
    string Name;
    unsigned int Id;
}

```

当然，我们还需要定义一个全局的变量。用于在内存里保存我们的场景上下文

``` csharp
class World
{
   IList<Player> Players;
   Player Master;
   
   // other props.
   
   void Tick(int deltaMs)
   {
   // ...
   }
}


// 游戏启动的初始化函数

World g_world;

void initialize(...)
{
    g_world = new World();
    // ....
}

```

正向的去开发游戏很容易理解，即使你不是一个游戏开发者，你也能明白这些代码的含义和思路。但是如何在逆向的过程中找到 HP 在内存中所在的位置，并不容易。所幸的是我们有一些工具。

在介绍工具之前，需要理解这些代码最终会编译成什么样子。

对于c++引擎而言，一个class最终被编译的样子是：

``` cpp

struct World::vftable
{
 void (__thiscall* Tick)(void*, int deltaMs);

}

struct World
{
  World::vftable* vftable;
  // ...
  
  Player* Master;
}

```
一般而言，一个类型会被编译成struct。首字节是 一个虚函数结构的指针。随后就是他的字段数据。

当我们要访问HP时，需要访问全局变量 g_world的地址，然后通过g_world+若干偏移量，得到 master 的指针，随后master所在地址+偏移量得到 HP 的位置。

由于g_world是全局变量，所以，他的地址也是基地址。基地址的值，就是g_world，g_world是指向一个类型为 World 的指针。

## 从一个程序开始

接下来，我们将从一个简单的程序入手，来确定访问HP的方式。


