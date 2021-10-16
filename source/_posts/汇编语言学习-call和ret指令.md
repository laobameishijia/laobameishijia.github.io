---
title: 汇编语言学习-call和ret指令
date: 2021-10-16 09:25:00
author: 美食家李老叭
img: 
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: call和ret指令都是转移指令,都修改ip或同时修改cs和ip
categories: 研究生预备学习
tags:
  - 汇编
---

# call和ret指令

## 思维导图

![思维导图](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211016111954.png)

关于使用栈来传递参数并用ret返回的实际例子还是需要多看才行。

## 综合实验

### 显示字符串

```text
名称：show_str
功能：在指定的位置,用指定的颜色,显示一个用0结束的字符串
参数：dh 行号(0-24), dl 列号 0-79, cl颜色, ds:si指向字符串的首地址
返回: 无
应用举例：在屏幕的8行3列,用绿色显示出data段中的字符串
```

#### 代码

```text
assume cs:code

data segment
    db 'Welcome to masm!',0
data ends

code segment

start:
    mov dh,8
    mov dl,3
    mov cl,2
    mov ax,data
    mov ds,ax
    mov si,0
    call show_str
    
    mov ax,4c00h
    int 21h

show_str:
    push es
    push bp
    push bx

    mov ax,0B800H
    mov es,ax

    ;找行号对应的内存地址
    mov al,160
    mul dh
    mov bp,ax
    sub bp,160
    ;找列对应的内存地址
    mov al,2
    mul dl
    mov di,ax
    sub di,2
    ;把颜色转移一下
    mov bl,cl
    push cx
s: 
    mov cl,ds:[si]
    mov ch,0
    jcxz ok
    mov es:[bp+di],cl
    mov es:[bp+di+1],bl

    add di,2
    add si,1
    jmp short s

ok:
    pop cx
    pop bx
    pop bp
    pop es
    ret
code ends
end start
```

#### 运行截图

![显示字符串](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211016112701.png)

### 解决除法溢出的问题

用div指令做出发的时候可能产生除法溢出,比如:1000000/10就不能用div指令来算

```text
名称：divdw
功能：进行不会产生溢出的除法运算,被除数为dword,除数为word.结果为dword
参数: ax dword的低16位 | dx dword高16位 | cx除数
返回: dx 结果的高16位, ax 结果的低16位, cx 余数
应用举例：计算 1000000/10(F4240H/0AH)
结果: dx = 0001H  ax = 86A0H cx = 0
```

#### 代码

```text
assume cs:codesg

datasg segment

datasg ends

codesg segment

start:
    mov ax,4240H
    mov dx,000FH
    mov cx,0AH
    call divdw
    mov ax,4c00h
    int 21h
;这里面就是因为 div 被除数默认放在ax | dx(高)和ax(低)中,所以比较麻烦
;除数可以放在寄存器里也可以放在内存单元里，有8/16两种, 8-AL商 AH余数 || 16-AX商 DX余数
;再就是因为数据运算要符合相同的类型,同为16或同为8,在寄存器里面换来换去的就比较麻烦
divdw:
    push ax
    mov ax,dx
    div cl
    mov bl,al
    mov bh,00H
    mov al,ah
    mov ah,00H;bx保留商,ax保留余数

    pop si
    mov dx,si

    mov dx,ax
    mov ax,si
    div cx;32/16 ax余数,dx商

    mov si,bx
    mov bx,dx
    mov dx,si

    mov cx,bx

    ret

codesg ends
end start
```

#### 运行截图

![解决除法溢出的问题](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211016113513.png)

### 数值显示

将12666以字符串的形式显示到显示器上

```text
名称：dtoc
功能：将word型数据转变为十进制的字符串,字符串以0为结尾符
参数：ax word型数据
    ds:si指向字符串的首地址
返回：无
应用举例: 将12666以十进制的形式在屏幕的8行3列,用绿色显示出来
```

#### 改进前代码

改进前,主要是利用在内存中的位置,来对字符串进行逆向的输出。因为算余数的话,顺序是66621得倒过来才行

```text

assume cs:code,ds:data

data segment
    db 10 dup(0)
data ends

code segment

start:
    mov ax,12666
    mov bx,data
    mov ds,bx
    mov si,0
    call dtoc
    

    mov dh,8
    mov dl,3
    mov cl,2
    call show_str

    mov ax,4c00h
    int 21h

dtoc:
    mov dx,00
    mov bh,00
    mov bl,10;感觉逻辑没问题啊,商越界了 得用32/16的
    div bx ;为什么会在这里卡住呢,感觉像是无限循环？？？
    mov ch,00h
    mov cl,dl
    jcxz ok
    add byte ptr cx,0030H
    mov ds:[si],cx
    inc si
    jmp dtoc
ok:
    ;也不是不能操作栈,在之前push进去的,在这里都pop出来就不会有问题,要不然回影响ret指令pop IP
    ;后面你可以再试试,这个程序还是有问题
    sub si,1;运行完之后si=5,然而ds[si]此时刚好是0,所以你得减去个1才行
    ret

show_str:
    push es
    push bp
    push bx

    mov ax,0B800H
    mov es,ax

    ;找行号对应的内存地址
    mov ah,00
    mov al,160
    mul dh
    mov bp,ax
    sub bp,160
    ;找列对应的内存地址
    mov ah,00
    mov al,2
    mul dl
    mov di,ax
    sub di,2
    ;把颜色转移一下
    mov bl,cl
    push cx
s: 
    mov cl,ds:[si]
    mov ch,0
    jcxz ok1
    mov es:[bp+di],cl
    mov es:[bp+di+1],bl

    add di,2
    sub si,1
    jmp short s

ok1:
    pop cx
    pop bx
    pop bp
    pop es
    ret    
    
code ends
end start

```

#### 改进后代码

改进之后,利用了栈的特性,先将算出来的余数入栈,然后再出栈写到内存里.这样就刚好倒过来了。不过需要注意的是,`在子程序中push进去的,在ret之前都要pop出来哦！`

```text
assume cs:code,ds:data

data segment
    db 10 dup(0)
data ends

code segment

start:
    mov ax,12666
    mov bx,data
    mov ds,bx
    mov si,0
    call dtoc
    

    mov dh,8
    mov dl,3
    mov cl,2
    call show_str

    mov ax,4c00h
    int 21h

dtoc:
    mov dx,00
    mov bh,00
    mov bl,10;感觉逻辑没问题啊,商越界了 得用32/16的
    div bx ;为什么会在这里卡住呢,感觉像是无限循环？？？
    mov ch,00h
    mov cl,dl
    jcxz ok
    add byte ptr cx,0030H
    mov ds:[si],cx
    inc si
    jmp dtoc
ok:
    ;也不是不能操作栈,在之前push进去的,在这里都pop出来就不会有问题,要不然回影响ret指令pop IP
    ;后面你可以再试试,这个程序还是有问题
    sub si,1;运行完之后si=5,然而ds[si]此时刚好是0,所以你得减去个1才行
    ret

show_str:
    push es
    push bp
    push bx

    mov ax,0B800H
    mov es,ax

    ;找行号对应的内存地址
    mov ah,00
    mov al,160
    mul dh
    mov bp,ax
    sub bp,160
    ;找列对应的内存地址
    mov ah,00
    mov al,2
    mul dl
    mov di,ax
    sub di,2
    ;把颜色转移一下
    mov bl,cl
    push cx
s: 
    mov cl,ds:[si]
    mov ch,0
    jcxz ok1
    mov es:[bp+di],cl
    mov es:[bp+di+1],bl

    add di,2
    sub si,1
    jmp short s

ok1:
    pop cx
    pop bx
    pop bp
    pop es
    ret    
    
code ends
end start
```

#### 运行截图

![数值显示](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211016114256.png)