---
title: 毕设-Fuzz-AFLGo源码阅读-6
date: 2022-4-21 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211119142134.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: AFL框架的源码基本内容阅读
categories: 毕设
tags:
  - Fuzz
  - AFLGo
---

# AFL种子文件是如何送入目标程序的？

在做实验的过程中，因为要复现漏洞嘛，要把种子文件送入目标程序。这里产生了一个疑问

**如果我用重定向的方式把种子文件(二进制文件)送入目标程序中，那么目标程序会显示相应的报错。但是如果我将二进制文件的内容(使用xxd)复制下来，然后在送入程序中，目标程序就不会报错。**

在小刘的启发下，我开始对照源码，观察AFL是如何把种子文件送入目标程序。

得到的结论就是，AFL是采用重定向的方式将种子文件送入目标程序的。

------------------------

1. 首先是启动参数中含有@@的(目标程序接受**文件名**，作为输入参数)

***`@@`本身代表的就是运行xmllint过程中，需要送进去的文件名***

`$AFLGO/afl-fuzz -m none -z exp -c 45m -i in -o out ./xmllint --valid --recover @@`

AFL会找到`@@`的位置，然后把`@@`替换成**out/.cur_input**。并且指定`out_file`为。`out/.cur_input`。

具体细节可以参照函数 `detect_file_args`


2. 然后是启动参数中不包含@@的(目标程序接受**标准输入**作为输入参数)

`$AFLGO/afl-fuzz -m none -z exp -c 45m -i in -o out binutils/cxxfilt`

AFL会创建`cur_input`文件，然后把`out_fd`作为`.cur_input`的文件描述符

具体细节参照函数`setup_stdio_file`


3. 紧接着会运行`perform_dry_run(..)`执行 input 文件夹下的预先准备的所有测试用例


4. `perform_dry_run(..)`的循环里面会调用`calibrate_case(...)`


5. `calibrate_case(...)`在没有启动fork server的时候，会调用  `init_forkserver(argv)`


6. `init_forkserver(argv)`中的子进程会进行重定向
   
  ```c
    // 如果指定了out_file，则标准输入重定向到dev_null_fd，否则重定向到out_fd
    if (out_file) {

      dup2(dev_null_fd, 0);

    } else {

      dup2(out_fd, 0);
      close(out_fd);

    }
  ```
  
  到此，也就是接受文件名作为参数的，会把标准输入重定向为NULL。接受标准输入作为参数的，回把out_fd指向的文件内容以标准输入的形式输入到程序中。

-------------

特别说明，在每一次fuzz的过程中。AFL都会调用类似于`write_to_testcase`的函数，将新的变异过的文件内容，写入`out_file`中，并清楚原来的内容。

在fuzz的过程中，用到哪个种子文件就打开哪个，然后把种子文件的内容复制到`.cur_input`文件里面, 再对 `.cur_input`文件中内容进行 位翻转等一系列变异操作，同时观察是否可以保留下来。如果能保留下来，就保留。不能保留就进行下面的变异步骤。之后循环往复。 
