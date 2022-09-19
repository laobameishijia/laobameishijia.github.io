---
title: CTF-PWN-平衡栈帧
date: 2022-9-17 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220917162435.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: 平衡栈帧
categories: CTF
tags:
    - PWN
---

# 栈溢出

## 第一讲--初始

**先说明，以下讨论均在32位机器下进行讨论！**

这个栈溢出的例子,我是从b站up主Innks那里看到的。因为有些细节不理解，所以动手敲了一遍。

### 改进之前

改进之前由于没有平衡栈空间，导致栈空间被破坏，程序无法正确返回。

```c
#include <iostream>
#include <Windows.h>

#pragma optimize("",off)

void MsgBox()
{
    int ary[2];
    ary[4] = ary[3]; //相当于把ret返回地址复制了一遍
    ary[3] = (int)MessageBoxA;
    //ary[4] = ary[3]; //这条指令写到这个位置和写在上面一个位置是完全不一样的哟！
    ary[5] = 0;
    ary[6] = (int)"NO_CALL 恭喜你中毒了";
    ary[7] = (int)"NO_CALL 你中毒了";
    ary[8] = MB_OK;
}
int main()
{

    MsgBox();
    MessageBoxA(0, "恭喜你中毒了", "你中毒了", MB_OK);
}
```

### 改进之后

改进之后，可以正确的让程序结束

```c
#include <iostream>
#include <Windows.h>

#pragma optimize("",off)

void MsgBox()
{
    int ary[2];
    ary[4] = ary[3]; //相当于把ret返回地址复制了一遍
    ary[3] = (int)MessageBoxA;
    //ary[4] = ary[3]; //这条指令写到这个位置和写在上面一个位置是完全不一样的哟！
    ary[5] = 0;
    ary[6] = (int)"NO_CALL 恭喜你中毒了";
    ary[7] = (int)"NO_CALL 你中毒了";
    ary[8] = MB_OK;
}
int main()
{
    __asm push ebp;
    __asm push ebp;
    __asm push ebp;
    __asm push ebp;
    __asm push ebp;
    MsgBox();
    MessageBoxA(0, "恭喜你中毒了", "你中毒了", MB_OK);
}

```


### 改进前后的栈帧对比

由于ipencil长时间不用，现在才发现已经被我摔坏了，以后不常用的东西还是保管好呢，不要满不在意，用的时候才发现坏了。

所以我就直接用手写的图了，懒得画图了。

也确实是该换手机了，手机前置摄像头找出来的照片有点黑。


![栈空间对比](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220917164427.jpg)

### 总结

因为我们采用栈溢出的方式调用了函数，那么应该`push到栈中的参数占的空间`占用了`其他栈帧的空间`。所以会导致后续程序流发生不可控制的变化。

![MessageBoxA的汇编代码](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220917164601.png)

汇编中的`retn 10h`就是为了平衡call函数之前push到栈里面的参数所占的空间。第一开始不理解的地方就在于此，我觉得`retn 10h`平衡的也就是`4个参数--16字节`。但是up主却用了五个`push ebp`。

实际上，`retn 10h`使栈空间减少了20个字节的空间。

>retn操作：先eip=esp，然后esp=esp+4
>retn N操作：先eip=esp，然后esp=esp+4+N

所以是20个字节！也就是五个`push ebp`就可以提前把这20个字节的空间弄出来。而不用影响到后续main函数的栈帧。

除此之外呢，我还发现vs---debug编译模式和release模式，是非常不一样的。

![vs-debug&release](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220917165306.png)

debug简单来说是为了方便分析程序，release模式是发布程序。我使用ida反汇编之后发现，debug生成的exe的汇编代码中添加了很多关于栈空间和一些寄存器的检查工作。而release模式下，是没有这些检查函数的。

## 第二讲--改进

通过前面的第一讲，我们明白了要解决通过栈溢出调用函数而导致的栈平衡问题。

up 还留了一个坑。 就是要采用什么样的方式去平衡栈，而不用写汇编。

### 预知识

函数调用有`__cdecl`、`__stdcall`。

__cdecl 是C Declaration的缩写（declaration，声明），表示C语言默认的函数调用方法：所有参数从右到左依次入栈，这些参数由调用者清除，称为手动清栈。被调用函数不会要求调用者传递多少参数，调用者传递过多或者过少的参数，甚至完全不同的参数都不会产生编译阶段的错误。

_stdcall 是StandardCall的缩写：所有参数从右到左依次入栈，如果是调用类成员的话，最后一个入栈的是this指针。这些堆栈中的参数由被调用的函数在返回后清除，使用的指令是 retnX，X表示参数占用的字节数，CPU在ret之后自动弹出X个字节的堆栈空间。称为自动清栈。函数在编译的时候就必须确定参数个数，并且调用者必须严格的控制参数的生成，不能多，不能少，否则返回后会出错。


MessageBoxA显然属于_stdcall。由被调用函数自己清栈。这也是系统API的特点之一。这样做的好处就是，严格控制了传递参数的个数，或多或少都不行。
![MessageBoxA](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220919153649.png)

MsgBox 属于_cdecl调用方式，由调用者自己清栈，这个过程中你传递参数的个数可以变化，这也是为什么可以定义可变参数的原因把。

![MsgBox](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220919154006.png)

### 思路

通过预知识的学习，我们知道了自定义函数和系统API调用采用的平栈方式不同，那么我们能不能利用这个特性来实现平栈呢？

![改进思路](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220919154545.png)

通过上图可以发现，我们通过给自定义函数增加参数，实现的效果和`push ebp`的效果一致。但是仅仅是这样不能够平栈，因为系统调用的时候还是会`retn 10h`，而由于_cdecl平栈的特性，其还`add esp 14h`。所以也就相当于进行了两次平栈操作。

那么 评论区大lao 的思路就是跳过 `add esp 14h`。由于这句指令是 3 字节，所以我们要在 `ary[4] = ary[3] + 3`。这样就跳到了下一条指令`push 0`的地址。

```C++
void MsgBox(int a, int b, int c, int d, int e)
{
    int ary[2];
    ary[4] = ary[3] + 3 ; //跳过 esp
    ary[3] = (int)MessageBoxA;
    ary[5] = 0;
    ary[6] = (int)"NO_CALL 恭喜你中毒了";
    ary[7] = (int)"NO_CALL 你中毒了";
    ary[8] = MB_OK;
}
int main()
{
    MsgBox(1,2,3,4,5);
    MessageBoxA(0, "恭喜你中毒了", "你中毒了", MB_OK);
}
```

然后up又对这个思路进行了改进。既然我们传递了一些参数，而且后续我们又把这些参数当作了MessageBoxA这个函数的参数，那么为什么不在传递参数的时候就把该传递的参数传进去呢。

```C++
void MsgBox(void* address,HWND hWnd,LPCSTR lpText,LPCSTR lpCaption,UINT uType)
{
    int ary[2];
    //交换ary[3] 和ary [4] ----也就是把MessageBoxA的地址换上来，把返回地址换下去
    ary[4] = ary[3] ^ ary[4];
    ary[3] = ary[3] ^ ary[4];
    ary[4] = ary[3] ^ ary[4];

    ary[4] = ary[4] + 3;
}
int main()
{
    MsgBox(MessageBoxA,0, "恭喜你中毒了", "你中毒了", MB_OK);
}
```

## 第三讲--完美

通过上述方式，我们知道了是需要跳过`_cdecl`或者是`_stdcall`两种平栈方式中的一种。那我们就可以利用这个欺骗编译器。

在声明的时候不给函数参数，但是在调用的时候，欺骗编译器这是个`_stdcall`类型且带有4个参数的函数，那么编译器会帮助我们将参数压栈，并且消除了`add esp 14h`的影响。

```C++
//完美版本
void MsgBox() {
    int ary[2];
    //交换ary[3] 和ary [4] ----也就是把MessageBoxA的地址换上来，把返回地址换下去
    ary[3] = ary[3] ^ ary[4];
    ary[4] = ary[3] ^ ary[4];
    ary[3] = ary[3] ^ ary[4];
}//ret----_cdecl 方式返回

typedef int* (_stdcall* _hMessageBoxA)(void* address, HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType);
//这样做的好处是，代码可复用性强。后续只需要写写声明就可以了。
typedef int* (_stdcall* _hMessageBoxW)(void* address, HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType);
int main()W
{
    //完美版本
    ((_hMessageBoxA)MsgBox) (MessageBoxA, 0, "恭喜你中毒了", "你中毒了", MB_OK);

    ((_hMessageBoxA)MsgBox) (MessageBoxA, 0, "恭喜你中毒了", "你中毒了", MB_OK);
}
```

## 所有的代码

```C++
#include <iostream>
#include <Windows.h>

#pragma optimize("",off)


// 初始版本
void MsgBox1(int a, int b, int c, int d, int e)
{
    int ary[2];
    ary[4] = ary[3]; //相当于把ret返回地址复制了一遍
    ary[3] = (int)MessageBoxA;
    //ary[4] = ary[3]; //这条指令写到这个位置和写在上面一个位置是完全不一样的哟！
    ary[5] = 0;
    ary[6] = (int)"NO_CALL 恭喜你中毒了";
    ary[7] = (int)"NO_CALL 你中毒了";
    ary[8] = MB_OK;
}

//进阶版本
void MsgBox2(int a, int b, int c, int d, int e)
{
    int ary[2];
    ary[4] = ary[3] + 3; //跳过 esp
    ary[3] = (int)MessageBoxA;
    //ary[4] = ary[3]; //这条指令写到这个位置和写在上面一个位置是完全不一样的哟！
    ary[5] = 0;
    ary[6] = (int)"NO_CALL 恭喜你中毒了";
    ary[7] = (int)"NO_CALL 你中毒了";
    ary[8] = MB_OK;
}
//进阶版本---改进
void MsgBox2_1(void* address, HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType)
{
    int ary[2];
    //交换ary[3] 和ary [4] ----也就是把MessageBoxA的地址换上来，把返回地址换下去
    ary[4] = ary[3] ^ ary[4];
    ary[3] = ary[3] ^ ary[4];
    ary[4] = ary[3] ^ ary[4];

    ary[4] = ary[4] + 3;
}
//完美版本
void MsgBox() {
    int ary[2];
    //交换ary[3] 和ary [4] ----也就是把MessageBoxA的地址换上来，把返回地址换下去
    ary[3] = ary[3] ^ ary[4];
    ary[4] = ary[3] ^ ary[4];
    ary[3] = ary[3] ^ ary[4];
}//ret----_cdecl 方式返回

typedef int* (_stdcall* _hMessageBoxA)(void* address, HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType);

int main()
{
    //完美版本
    ((_hMessageBoxA)MsgBox) (MessageBoxA, 0, "恭喜你中毒了", "你中毒了", MB_OK);

    ((_hMessageBoxA) MsgBox) (MessageBoxA, 0, "恭喜你中毒了", "你中毒了", MB_OK);
}
```

## 总结

不知道怎么说，自己的水平还是差了很多，up主所提到的安全思维也没有。很有可能做一辈子也是个普通人，但那又能怎么样呢? 一直学下去呗。不断丰富自己，最后不会太差哒！
