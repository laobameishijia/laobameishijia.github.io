---
title: 毕设-Fuzz-AFLGo源码阅读-5
date: 2022-2-21 09:25:00
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

# AFLGo源码阅读

- [AFLGo源码阅读](#aflgo源码阅读)
  - [按照Main函数中的顺序](#按照main函数中的顺序)
    - [setup_signal_handlers](#setup_signal_handlers)
    - [check_asan_opts()](#check_asan_opts)
    - [fix_up_sync()--不知道具体用途](#fix_up_sync--不知道具体用途)
    - [save_cmdline(argc, argv)](#save_cmdlineargc-argv)
    - [fix_up_banner(argv[optind])](#fix_up_bannerargvoptind)
    - [check_if_tty()](#check_if_tty)
    - [get_core_count()](#get_core_count)
    - [bind_to_free_cpu()](#bind_to_free_cpu)
    - [check_crash_handling()](#check_crash_handling)
    - [check_cpu_governor()](#check_cpu_governor)
    - [setup_post()](#setup_post)
    - [setup_shm()](#setup_shm)
    - [init_count_class16()](#init_count_class16)
    - [setup_dirs_fds()](#setup_dirs_fds)
    - [read_testcases()](#read_testcases)
    - [load_auto()](#load_auto)
    - [pivot_inputs()](#pivot_inputs)
      - [C语言知识](#c语言知识)
    - [load_extras()](#load_extras)
    - [find_timeout()](#find_timeout)
    - [detect_file_args()](#detect_file_args)
      - [C语言知识](#c语言知识-1)
    - [setup_stdio_file()](#setup_stdio_file)
    - [check_binary()](#check_binary)
    - [get_qemu_argv()](#get_qemu_argv)
    - [perform_dry_run()](#perform_dry_run)
      - [calibrate_case()](#calibrate_case)
        - [count_bytes()](#count_bytes)
        - [update_bitmap_score()](#update_bitmap_score)
        - [minimize_bits()](#minimize_bits)
    - [cull_queue()](#cull_queue)
      - [mark_as_redundant](#mark_as_redundant)
    - [show_init_stats()](#show_init_stats)
    - [find_start_position()](#find_start_position)
    - [write_stats_file()](#write_stats_file)
    - [save_auto()](#save_auto)
    - [fuzz_one && while循环](#fuzz_one--while循环)
      - [calculate_score()](#calculate_score)
      - [common_fuzz_stuff()](#common_fuzz_stuff)
      - [run_target()](#run_target)
        - [C 知识](#c-知识)
          - [进程状态](#进程状态)
          - [setitimer](#setitimer)
      - [save_if_interesting](#save_if_interesting)
    - [write_bitmap()](#write_bitmap)
    - [write_stats_file()](#write_stats_file-1)
    - [stop_fuzzing:](#stop_fuzzing)
  - [AFLgo命令行启动参数](#aflgo命令行启动参数)
  - [Linux命令](#linux命令)
  - [参考](#参考)

## 按照Main函数中的顺序

### setup_signal_handlers

初始化各种信号量

终止进程的、超时的等等

### check_asan_opts()

通过检查环境变量中的值来判断--检查ASAN设置

### fix_up_sync()--不知道具体用途

没理解

### save_cmdline(argc, argv)

用`orig_cmdline`保存复制当前命令行

### fix_up_banner(argv[optind])

根据最后一个参数设置标头(banner)?

### check_if_tty()

检查是不是在终端运行

### get_core_count()

从系统文件中获取cpu核的相关信息

### bind_to_free_cpu()

把进程绑定在具体的内核上？

### check_crash_handling()

保证core dumps不会进入程序, 否则会增加将崩溃信息通过waitpid传递给fuzzer的延迟。

### check_cpu_governor()

要把CPU频率调节的算法(可能忽视fuzz产生的短进程)关了，以提高aflgo-fuzz的效率。

### setup_post()

不理解

### setup_shm()

配置共享内存和`virgin_bits`, 并且将共享内存的首地址赋值给`trace_bits`.

### init_count_class16()

之所以用左移是为了加快速度

最终初始化是下面这个样子。16位一个
        0-0 ....      128-0  -256个元素   :0-1-2-4-8-16-32-64-128
      ⬇ 0-1 ....
        0-2 ....
        0-2 ....
        0-4 ....
        0-4 .... 
        .............
        0-128....    128-128         一共 65536个 16bit

### setup_dirs_fds()

1. flock 了 out_dir_fd

2. 创建了跟下面有关的目录
   - queue 
   - crashes
   - hangs
   .... 

3. 还创建其他的fd(/dev/null&/dev/urandom)方便后续使用

### read_testcases()

从`input directory`中读取所有的测试用例，检测测试用例的大小，以及是否已经完成了`deterministic fuzzing`阶段，然后添加到queue中。

初始化

- queued_at_start  Total number of initial inputs
- last_path_time   Time for most recent path (ms)

### load_auto()

加载自动生成的附加组件




### pivot_inputs()

- 首先检查是不是之前跑过的
  - 如果是的话，看一下id是不是一致。
    - id一致, 要改变对应entry的depth
  - 如果不是，就起新名字 `id:%06u,orig:%s`
  - 然后就是重新命名文件，并且更改`q->fname=nfn`

#### C语言知识

strrchr和strchr类似，但是从右向左找字符c，找到字符c第一次出现的位置就返回，函数名中间多了一个字母r可以理解为Right-to-left。

### load_extras()

这个没看懂是干啥的。
跟这些有关，但是不知道具体在fuzz的过程中起到了什么作用

```c++
struct extra_data {
  u8* data;                           /* Dictionary token data            */
  u32 len;                            /* Dictionary token length          */
  u32 hit_cnt;                        /* Use count in the corpus          */
};

static struct extra_data* extras;     /* Extra tokens to fuzz with        */
static u32 extras_cnt;                /* Total number of tokens read      */

static struct extra_data* a_extras;   /* Automatically selected extras    */
static u32 a_extras_cnt;              /* Total number of tokens available */

```

### find_timeout()

只有在Resuming an older fuzzing job的情况下，才会使用。

从状态目录中读取文件名, 并把`exec_timeout :`后面的值复制给`exec_tmout`, 将timeout_given赋值为3.

### detect_file_args()

根据参数@@后面带的东西，更改文件名. 看的也不是很懂。

#### C语言知识

定义函数：char * getcwd(char * buf, size_t size);

函数说明：getcwd()会将当前的工作目录绝对路径复制到参数buf 所指的内存空间，参数size 为buf 的空间大小。

注：
1、在调用此函数时，buf 所指的内存空间要足够大。若工作目录绝对路径的字符串长度超过参数size 大小，则返回NULL，errno 的值则为ERANGE。
2、倘若参数buf 为NULL，getcwd()会依参数size 的大小自动配置内存(使用malloc())，如果参数size 也为0，则getcwd()会依工作目录绝对路径的字符串程度来决定所配置的内存大小，进程可以在使用完次字符串后利用free()来释放此空间。


### setup_stdio_file()

如果没有用-f指定输出文件的话, 那就用默认的`.cur_input`创建

### check_binary()

具体代码没看。。

检查目标二进制文件是否存在，以及它是否是shell脚本。确保可以进行afl的插桩。

### get_qemu_argv()

不知道干啥的

### perform_dry_run()

简单的把所有的测试用例都提前运行一遍，确保程序像预期的那样运行。如果不是的话，会有一些相应的提示。

#### calibrate_case()

测试一个entry，看看是不是有覆盖率、新的路径的添加等等变量是否正常工作啥的。

关于`entry`属性里面的`var_behavior`的理解: 因为在`calibrate`的阶段中，是没有发生变异的，那么如果测试用例在经过不同次数的执行后，产生了不一样的`path`。那么就把这个`entry`标记为`variable`。**这个属性并没有影响到后续的其他步骤**。根据注释，应该只是简单的标注，方便能找到吧。

##### count_bytes()

数一下有多少个字节不为零, 8位代表一个path, 不同的命中次数可能会导致8位中不同位置的bit置1

##### update_bitmap_score()

当某个entry触发了新的path, 我们要与之前的同样触发这个path的"最优"的entry进行一个比较。看看到底谁更优秀。

所谓的`top_rated[]` 就是 `a minimal set of paths that trigger all the bits seen in the bitmap so far.`

##### minimize_bits()

把`trace_bits`压缩为一个占用空间更小的数组。1位代表一个`path`现在。所以刚好是分配了`MAP_SIZE>>3`的空间。


### cull_queue()

`top_rated[i]` 代表的就是发现路径序号为i的最优entry(fav_factor最小的) 而且**关键的是top_rated[i] 指针指向的是queue中的特定的entry**。所以在将`top_rated[i]->favored = 1 `时，原来`queue`中的`entry`的`favored`也同样被设置为1

值得注意的是，并不是说`top_rated[]`中所有的`entry`都是`favored`的。当且仅当你发现的`path`是你之前`entry`都没有发现过的情况下，这个`entry`才会被设置为`favored`

> 我觉得这里有个值得深思的地方，程序这样设计的话，test_case的顺序会影响到其是否会被设置为favored. 这种随机性会不会对框架整体的性能产生一定的影响。

#### mark_as_redundant

把对应的`entry`标记为`redundant`，其间还会创建一些目录，至于什么作用没看懂。



### show_init_stats()

显示统计数据 Total calibration cycles\max_bits\min_bits\exec_us\len等等

根据平均运行时间重新设置一个`timeout_given`

### find_start_position()

当要恢复程序进程的时候，从`fuzzer_stats`目录的文件的文件名中读取相应的位置。

### write_stats_file()

把用到的基本状态信息都写入到状态文件中，这些变量都会在终端页面显示中用到。

### save_auto()

自动保存生成的extras，这个跟token有关系，但没看懂token到底有什么作用。

### fuzz_one && while循环

接下来是循环中的函数

- 首先在进入循环之前, 要先cull_queue, 把favor的entry标记出来

- 判断queue_cur是否为空

  - 如果为空的话，说明是第一次进入循环。进行必要的初始化。

- 然后就是fuzz_one
  
  - 判断在当下的队列中，是否含有 `favored\non-fuzzed` 的`entry`，如果有那么会**以99%的概率跳过那些已经被fuzz过或者不是favored**的`entry`.
  - 如果没有上面所说的那种类型的`entry` 会以75%跳过not fuzzed ，以95%跳过fuzzed的`entry`。 
  - 然后将test case中的内容映射到内存中，这样文件中的位置直接就有对应的内存地址，对文件的读写可以直接用指针来做而不需要`read`\\`write`函数。
  - 如果最初的calibration阶段失败了, 那现在要重新来一遍。
  - `trimming`阶段，**这个阶段的作用，没看懂**。 不明白为什么这个函数会调用run_taget
  - 计算entry分数
  - 看看是否要跳过`deterministic`变异阶段
    - 如果skip_deterministic设置为1、或者entry fuzzed或者entry->passed_det设置为1)
    - 如果执行路径校验将其置于该主实例的范围之外，则跳过确定性模糊处理。
  - 按照以下阶段进行变异 
    - simple bitflip
    - arithmetic
    - interst
    - dictionary
    - havoc
    - splice
    >当然在这些变异阶段中, 大多都是每变异一次就进行`common_fuzz_stuff`。 还有很多为了保证程序效率(比如: 当变异出现的结果在之前的变异阶段已经被运行过的时候可以跳过、当对于某个字节的变异没有出现效果，那在以后的变异阶段就不会变异该字节了-相当于认为该字节对于提高程序效果没有太大的意义)

  循环结束后，回对sync_fuzzer进行一个操作，这个可以后面再看。

#### calculate_score()

计算得分,跟得分有关的因素

- `exec_us` 和`avg_execc_us`的大小关系, `exec_us`相对越小, 得分肯定就越高
- `bitmap_size`(发现的路径数) 和 `avg_bitmap_size`大小关系, `bitmap_size`相对越高, 得分越高
- `handicap` 某个`testcase`可能是在程序运行的末尾才发现, 然后被添加到队列中。而这个时候，队列中前面的`entry`很有可能已经运行了很多`cycle`. 所以，这部分**后来添加到队列中**的`entry`得分更高。
- `depth` 原文 *under the assumption that fuzzing deeper test cases is more likely to reveal stuff that can't be discovered with traditional fuzzers.* `depth`的值越大，得分也就越高。也就是说, 一些变异的`entry`较大可能会是后面才添加进来的。所以假设越往后添加进来的越高。**这个要跟上面的handicap相区别，depth反映的是队列中的entry数量, handicap是整体队列变异的cycle**
- `cooling_schedule` 基于距离的模拟退火算法, 距离越近的随着时间的推移, `power_factor`会越来越高. 相对应的得分也就越高. `perf_score *= power_factor` 

> 具体的得分，跟确定性变异阶段的时间没有关系，得分越高，随机性变异阶段的时间也就越长。



#### common_fuzz_stuff()

把经过变异修改的文件重新写入testcase, 然后在进行`run_target()`。接着运行`save_if_interesting()`判断是否对变异的testcase进行统计或者其他操作

#### run_target()

- 第一种情况: 独自运行exec, 等待子进程结束
- 第二种情况: 通过管道和forkserver通信，forkserver fork出一个子进程进行fuzz，将子进程的状态写入通道。 父进程再通过通道中的信息, 对程序状态进行返回。**当然在子进程进行fuzz的过程中 trace_bits会发生更新**

##### C 知识

###### 进程状态

- WIFSIGNALED(status)为非0 表明进程异常终止 **用来判断crash**
- WIFSTOPPED(status)为非0 表明进程处于暂停状态 **用来判断fork server是否正常进行, 此时因为fork server是处于循环当中，所以对应的状态是处于暂停。**
- WTERMSIG(status) 获取程序退出的信号(比如:`SIGKILL`)

###### setitimer

关于这个，详细的内容网上都有。 但是没搞清楚这个时间定时到底是阻塞的还是非阻塞的。？？

#### save_if_interesting

看一下当前的testcase是否触发了新的路径, 如果触发了新的路径，需要把这个testcase添加到当前的队列里面。并且要在queue中以`("%s/queue/id:%06u,%llu,%s", out_dir, queued_paths, get_cur_time() - start_time ,describe_op(hnb))`这样的形式命名。

根据`run_target()`的返回值，处理timeout、crash、error的情况


### write_bitmap()

把当前共享内存中的bitmap写到文件中去

### write_stats_file()

更新状态文件中的数据

### stop_fuzzing:

程序的终止是需要用户自己按下`ctrl+c` 循环不会自己退出

对占有的内存空间进行释放, 退出程序


## AFLgo命令行启动参数


|参数  |含义  |
|---------|---------|
|i     |    输入目录    |
|o     |    输出目录     |
|M     |    master sync ID     |
|S     |    master sync ID     |
|f     |    目标文件     |
|x     |    字典目录     |
|t     |    超时时间设定     |
|m     |    内存限制     |
|d     |    是否跳过确定性变异阶段     |
|B     |    加载bitmap     |
|C     |    Crash模式     |
|T     |    banner    |
|Q     |    QEMU模式     |
|z     |    模拟退火算法选定     |
|c     |    退火算法的运行时间     |

```bash
Required parameters:

  -i dir        - input directory with test cases
  -o dir        - output directory for fuzzer findings

Directed fuzzing specific settings:

  -z schedule   - temperature-based power schedules
                  {exp, log, lin, quad} (Default: exp)
  -c min        - time from start when SA enters exploitation
                  in secs (s), mins (m), hrs (h), or days (d)

Execution control settings:

  -f file       - location read by the fuzzed program (stdin)
  -t msec       - timeout for each run (auto-scaled, 50-1000 ms)
  -m megs       - memory limit for child process (50 MB)
  -Q            - use binary-only instrumentation (QEMU mode)

Fuzzing behavior settings:

  -d            - quick & dirty mode (skips deterministic steps)
  -n            - fuzz without instrumentation (dumb mode)
  -x dir        - optional fuzzer dictionary (see README)

Other stuff:

  -T text       - text banner to show on the screen
  -M / -S id    - distributed mode (see parallel_fuzzing.txt)
  -C            - crash exploration mode (the peruvian rabbit thing)


```



## Linux命令

- **export**
  
  为shell变量或函数设置导出属性。它们会成为环境变量, 可以在脚本中访问它们，尤其是脚本中调用的子进程需要时。

- **echo**
  
  echo命令 用于在shell中打印shell变量的值，或者直接输出指定的字符串。linux的echo命令，在shell编程中极为常用, 在终端下打印变量value的时候也是常常用到的，因此有必要了解下echo的用法echo命令的功能是在显示器上显示一段文字，一般起到一个提示的作用。

- **mkdir**
  
  创建目录

- **cat**

  连接多个文件并打印到标准输出

- **cut**
  
  cut命令用来显示行中的指定部分，删除文件中指定字段。说明：该命令有两项功能，其一是用来显示文件的内容，它依次读取由参数 file 所指 明的文件，将它们的内容输出到标准输出上；其二是连接两个或多个文件，如cut fl f2 > f3将把文件 fl 和 f2 的内容合并起来，然后通过输出重定向符“>”的作用，将它们放入文件 f3 中。

- **rev**
  
  将文件内容以字符为单位反序输出---也就是每行的字符都到过来

- **cp**

  将源文件或目录复制到目标文件或目录中

- **pushd&&popd**
  
  倒可以简单地把这个命令理解为切换/再换回来目录的命令。

- **chmod**
  
  用来变更文件或目录的权限

- **mv**
  
  mv命令 用来对文件或目录重新命名，或者将文件从一个目录移到另一个目录中。source表示源文件或目录，target表示目标文件或目录。如果将一个文件移到一个已经存在的目标文件中，则目标文件的内容将被覆盖

## 参考

1. http://rk700.github.io/2018/01/04/afl-mutations/
2. https://rk700.github.io/2017/12/28/afl-internals/#%E5%88%86%E6%94%AF%E4%BF%A1%E6%81%AF%E7%9A%84%E5%88%86%E6%9E%90
3. https://paper.seebug.org/496/#_2
4. https://bbs.pediy.com/thread-265936.htm#msg_header_h1_2  
5. https://paper.seebug.org/1732/#afl-afl-asc
6. https://www.anquanke.com/post/id/250540#h2-5
7. https://linux.cmsblogs.cn/    ----查询linux命令的网站