---
title: 汇编语言学习-int指令
date: 2021-10-23 09:25:00
author: 美食家李老叭
img: 
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: 中断信息可以来自CPU的内部和外部，int指令引发的中断是一种重要的内中断
categories: 研究生预备学习
tags:
  - 汇编
---

# int指令

## 思维导图

![思维导图](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211023100523.png)

## BIOS和DOS中断例程的安装过程

- 开机后，CPU一加电，初始化CS=0FFFFH，IP=0，自动从FFFF：0单元开始执行程序。FFFF：0处有一条跳转指令，CPU执行该指令后，转去执行BIOS中的硬件检测系统和初始化程序
- 初始化程序将建立BIOS所支持的中断向量。
  > 注意， 对于BIOS所提供的中断例程，只需要将入口地址登记在中断向量表中即可，因为他们是固化到ROM中的程序，一直在内存中存在
- 硬件系统检测和初始化完成后，调用int 19h进行操作系统引导。从此将计算机交由操作系统控制。
- DOS启动后，除完成其他工作外，还将它所提供的中断例程装入内存，并建立相应的中断向量。


## 实验13

### 1

>编写并安装int 7ch中断例程，功能为显示一个用0结束的字符串，中断例程安装在0：200处
dh 行号，dl 列号， cl 颜色， ds:si指向字符串首地址

#### 思路

- 编写能显示字符串的中断处理程序
- 将终端服务程序移动到0000:0200处
- 修改中断向量表中的7cH号中断的入口地址,使其指向0000:0200

#### 代码

>一些注意事项和问题就写到注释里面了,后面忘了的话记得看看

```text
assume cs:code

data segment
    db 'welcome to masm!',0
data ends

code segment

start:
    mov ah,15
    int 10h
    mov ah,0
    int 10h

    mov ax,cs
    mov ds,ax
    mov si,offset interupt; set the ds:si point to the source address
    mov ax,0
    mov es,ax
    mov di,200h; set the es:di point to the destination address
    mov cx,offset interuptend-offset interupt; cx the length of transmisson
    cld ; the sequence of the transmit
    rep movsb

    mov ax,0
    mov es,ax
    mov word ptr es:[7ch*4],200h
    mov word ptr es:[7ch*4+2],0
    
    mov dh,10
    mov dl,10
    mov cl,2
    mov ax,data
    mov ds,ax
    mov si,0
    int 7ch
    mov ax,4c00h
    int 21h

;dh 行号
;dl 列号
;cl 颜色
;ds:si 指向字符串首地址
interupt:
    mov ax,0b800h
    mov es,ax
    ;找行号对应的内存地址
    mov ah,00
    mov al,160
    mul dh
    mov bp,ax
    ;找列对应的内存地址
    mov ah,00
    mov al,2
    mul dl
    mov di,ax
    s:
        mov al,ds:[si]
        cmp al,0
        je ok
        mov es:[bp+di],al
        mov es:[bp+di+1],cl
        inc si
        add di,2
        jmp s
    ok:
        iret
interuptend:
    nop

code ends
end start
```

#### 实现效果

![实现效果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211023102716.png)

### 2

> 编写并安装int 7ch中断例程，功能为完成loop指令的功能
> cx为循环次数，bx为位移

#### 思路

- 编写能实现loop循环的中断处理程序
- 将终端服务程序移动到0000:0200处
- 修改中断向量表中的7cH号中断的入口地址,使其指向0000:0200

#### 代码

```text
assume cs:code

data segment
    db 'welcome to masm!',0
data ends

code segment

start:
    mov ah,15
    int 10h
    mov ah,0
    int 10h

    mov ax,cs
    mov ds,ax
    mov si,offset interupt; set the ds:si point to the source address
    mov ax,0
    mov es,ax
    mov di,200h; set the es:di point to the destination address
    mov cx,offset interuptend-offset interupt; cx the length of transmisson
    cld ; the sequence of the transmit
    rep movsb

    mov ax,0
    mov es,ax
    mov word ptr es:[7ch*4],200h
    mov word ptr es:[7ch*4+2],0

    mov ax,0b800h
    mov es,ax
    mov di,160*12
    mov bx, offset s - offset se
    mov cx,80
s:
    mov byte ptr es:[di],'!'
    add di,2
    int 7ch
se: 
    nop
    mov ax,4c00h
    int 21h
;采用中断方式实现的loop,转移的范围要更大因为时16位的
;正常情况下的loop是8位的,范围相对来说要小一些
interupt:
    push bp
    mov bp,sp
    dec cx
    jcxz interuptret;就是加不加bx的区别,当cx为零的时候,这个时候就不加bx也就是不会再跳回去了
    add ss:[bp+2],bx;注意这个bx是个负数！
interuptret:
    pop bp
    iret
interuptend:
    nop

code ends
end start
```

#### 实现效果

![实现效果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211023102946.png)

### 3

> 下面的程序，分别在屏幕的第2、4、6、8行显示4句英文诗，补全程序.

#### 代码

这里面有个写法挺奇妙的, `ds:[ds:[si]]`是可以这样嵌套着写的

```text
assume cs:code

code segment

s1: db 'Good,better,best,','$'
s2: db 'Never let it rest,','$'
s3: db 'Till good is better,','$'
s4: db 'And better,best.','$'
s:  dw offset s1, offset s2, offset s3,offset s4
row: db 2,4,6,8

start:
    mov ah,15
    int 10h
    mov ah,0
    int 10h

    mov ax,cs
    mov ds,ax
    mov bx,offset s
    mov si,offset row;行号
    mov cx,4
ok:
    mov bh,0;第0页
    mov dh,ds:[si];这个行号怎么不起作用呢
    mov dl,0;列号
    mov ah,2
    int 10

    mov dx,ds:[ds:[bx]];可以这样嵌套着写！我真是个大聪明！哈哈哈哈哈~~~
    mov ah,9
    int 21h
    
    ;直接在这里加就行了,不用非得跑到mov指令那里加
    ;不能直接加 标号,得加寄存器,你个憨憨
    add si,1
    add bx,2
    loop ok

    mov ax,4c00h
    int 21h

code ends
end start
```

#### 效果

有个疑惑就是为什么这个地方的行号和列号的改变不起作用呢??

![效果](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211023103301.png)