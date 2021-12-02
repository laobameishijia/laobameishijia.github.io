---
title: 毕设-Fuzz-5
date: 2021-11-30 09:25:00
author: 美食家李老叭
img: https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211119142134.png
top: false
hide: false
cover: false
coverImg: 
password: 
toc: true
mathjax: false
summary: 毕设是模糊测试相关的,提前了解
categories: 毕设
tags:
  - Fuzz
---

# Notebook阅读

## Grammars

首先就是要讲, 为什么我们需要语法来规范测试数据的生成. 在之前的讲解中, 相比我们也已经很清楚了, 依靠随机生成的测试数据几乎没有几个符合程序输入(假设程序对输入的数据有严格的限制的话). **为了提高生成测试数据的效率, 必须要采用语法对生成测试数据的过程进行限制.**
> Compilers and Web browsers, of course, are not only domains where grammars are needed for testing, but also domains where grammars are well-known. Our claim in this book is that grammars can be used to generate almost any input, and our aim is to empower you to do precisely that.--浏览器和编译器是比较常见的需要用语法进行规范测试的两种领域. 但是我们的目的是要用语法精准的生成任何你想要的输入数据.

### 如何构建语法

语法是一个非终止符和代替扩张list的一个映射
**A grammar is defined as a mapping of nonterminal symbols to lists of alternative expansions**

```python
>>> US_PHONE_GRAMMAR: Grammar = {
>>>     "<start>": ["<phone-number>"],
>>>     "<phone-number>": ["(<area>)<exchange>-<line>"],
>>>     "<area>": ["<lead-digit><digit><digit>"],
>>>     "<exchange>": ["<lead-digit><digit><digit>"],
>>>     "<line>": ["<digit><digit><digit><digit>"],
>>>     "<lead-digit>": ["2", "3", "4", "5", "6", "7", "8", "9"],
>>>     "<digit>": ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
>>> }
```

根据语法生成数据就是首先从`start`标签开始, 将`start`标签用后面的`phone-number`替换--(**如果start对应多个非终止符号的话, 那就是随机挑一个非终止符号进行替换**), 再用`(<area>)<exchange>-<line>`替换`phone-number`(**如果phone-number对应多个非终止符号的话, 那就是随机挑一个非终止符号进行替换**), 重复操作, 直到后面没有非终止符号为止. (**当然, 像digit这种并没有对应非终止符号的, 也是同样随机挑一个就行**)

```python
>>> [simple_grammar_fuzzer(US_PHONE_GRAMMAR) for i in range(5)]
['(692)449-5179',
 '(519)230-7422',
 '(613)761-0853',
 '(979)881-3858',
 '(810)914-5475']
```

大体的思路上就是这些, 后面讲述的内容. 是**如何更好更快地构建语法**, 因为不可能说所有的语法规则都依靠人工构建. 需要让语法变得非常方便添加\方便加入一些更符合输入数据的限制.


### Lessons Learned

* Grammars are powerful tools to express and produce syntactically valid inputs.
* Inputs produced from grammars can be used as is, or **used as seeds for mutation-based fuzzing.**
* Grammars can be extended with character classes and operators to make writing easier.


### py高阶用法

#### 1 zip_longest

1、zip_longest需要导入itertools模块，且使用的时候需要指定一个填充值fillvalue。

2、当有可迭代对象遍历完，但其他对象还没有的时候，缺少的相应元素就会使用填充值进行填充。

```python
from itertools import zip_longest
a = [i for i in range(10)]
b = [i for i in range(1, 9)]
for num1, num2 in zip_longest(a, b, fillvalue=-1):
    print(num1, num2)
# 0 1
# 1 2
# 2 3
# 3 4
# 4 5
# 5 6
# 6 7
# 7 8
# 8 -1
# 9 -1
```

#### 2 **kwargs

`**kwargs` 允许你将不定长度的键值对, 作为参数传递给一个函数。 如果你想要在一个函数里处理带名字的参数, 你应该使用`**kwargs`。

```python
def greet_me(**kwargs):
    for key, value in kwargs.items():
        print("{0} == {1}".format(key, value))


>>> greet_me(name="yasoob")
name == yasoob
```

#### 3 作用域和命名空间

详见菜鸟教程 [https://www.runoob.com/python3/python3-namespace-scope.html](https://www.runoob.com/python3/python3-namespace-scope.html)

```python
def reachable_nonterminals(grammar: Grammar,
                           start_symbol: str = START_SYMBOL) -> Set[str]:
    reachable = set()

    def _find_reachable_nonterminals(grammar, symbol):
        nonlocal reachable # 这里用到了nonlocal
        reachable.add(symbol)
        for expansion in grammar.get(symbol, []):
            for nonterminal in nonterminals(expansion):
                if nonterminal not in reachable:
                    _find_reachable_nonterminals(grammar, nonterminal)

    _find_reachable_nonterminals(grammar, start_symbol)
    return reachable
```

#### 4 set()

set() 函数创建一个无序不重复元素集，可进行关系测试，删除重复数据，还可以计算交集、差集、并集等。

```python
>>>x = set('runoob')
>>> y = set('google')
>>> x, y
(set(['b', 'r', 'u', 'o', 'n']), set(['e', 'o', 'g', 'l']))   # 重复的被删除
>>> x & y         # 交集
set(['o'])
>>> x | y         # 并集
set(['b', 'e', 'g', 'l', 'o', 'n', 'r', 'u'])
>>> x - y         # 差集
set(['r', 'b', 'u', 'n'])
>>>
```

#### 5 typing.Optional

[csdn](https://blog.csdn.net/qq_44683653/article/details/108990873#:~:text=Python%20%E5%A4%A9%E7%94%9F%E4%B8%8D%E6%94%AF%E6%8C%81%20Option%20%E7%B1%BB%E5%9E%8B%EF%BC%8C%20typing%20%E6%9C%89%E4%B8%AA%20Optional%20%2C,%5B%20%28int%29%20-%3E%20Any%5D%27%20%28matched%20generic%20type%20%27Optiona)

![typing.Optional](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211202112803.png)


> 我觉得这几个函数写的非常好,简洁明了. 有些地方看的不是很懂,看来py的很多高级的用法并没有掌握到.

### 代码

说实在的, 这一部分的思想是比较简单的, 但是在代码实现上, 确实用到了很多之前在写python程序时没有用到的写法,而且从代码书写的思路和简洁性上面来说, 比之前写的代码高了不知道几个档次. 再一次让我感觉到了, 原来写出来的代码可以这么写. 所以我在前面把这些方法都总结了下来, 包括后面的这些代码, 希望以后你可以多多的借鉴.  当然这也只是其中的一部分而已.

```python

def def_used_nonterminals(grammar: Grammar, start_symbol: 
                          str = START_SYMBOL) -> Tuple[Optional[Set[str]], 
                                                       Optional[Set[str]]]: 
  
    """Return a pair (`defined_nonterminals`, `used_nonterminals`) in `grammar`.
    In case of error, return (`None`, `None`)."""

    # 这里为什么用元组呢? 是因为元组没有办法被删除嘛?

    defined_nonterminals = set()
    used_nonterminals = {start_symbol}

    for defined_nonterminal in grammar:
        defined_nonterminals.add(defined_nonterminal)
        expansions = grammar[defined_nonterminal]
        if not isinstance(expansions, list):
            print(repr(defined_nonterminal) + ": expansion is not a list",
                  file=sys.stderr)
            return None, None

        if len(expansions) == 0:
            print(repr(defined_nonterminal) + ": expansion list empty",
                  file=sys.stderr)
            return None, None

        for expansion in expansions:
            if isinstance(expansion, tuple):
                expansion = expansion[0]
            if not isinstance(expansion, str):
                print(repr(defined_nonterminal) + ": "
                      + repr(expansion) + ": not a string",
                      file=sys.stderr)
                return None, None

            for used_nonterminal in nonterminals(expansion):
                used_nonterminals.add(used_nonterminal)

    return defined_nonterminals, used_nonterminals

def reachable_nonterminals(grammar: Grammar,
                           start_symbol: str = START_SYMBOL) -> Set[str]:
    reachable = set()

    def _find_reachable_nonterminals(grammar, symbol):
        nonlocal reachable
        reachable.add(symbol)
        for expansion in grammar.get(symbol, []):
            for nonterminal in nonterminals(expansion):
                if nonterminal not in reachable:
                    _find_reachable_nonterminals(grammar, nonterminal)

    _find_reachable_nonterminals(grammar, start_symbol)
    return reachable

def unreachable_nonterminals(grammar: Grammar,
                             start_symbol=START_SYMBOL) -> Set[str]:
    return grammar.keys() - reachable_nonterminals(grammar, start_symbol)

def opts_used(grammar: Grammar) -> Set[str]:
    used_opts = set()
    for symbol in grammar:
        for expansion in grammar[symbol]:
            used_opts |= set(exp_opts(expansion).keys())
    return used_opts

def is_valid_grammar(grammar: Grammar,
                     start_symbol: str = START_SYMBOL, 
                     supported_opts: Set[str] = set()) -> bool:
    """Check if the given `grammar` is valid.
       `start_symbol`: optional start symbol (default: `<start>`)
       `supported_opts`: options supported (default: none)"""

    defined_nonterminals, used_nonterminals = \
        def_used_nonterminals(grammar, start_symbol)
    if defined_nonterminals is None or used_nonterminals is None:
        return False

    # Do not complain about '<start>' being not used,
    # even if start_symbol is different
    if START_SYMBOL in grammar:
        used_nonterminals.add(START_SYMBOL)

    for unused_nonterminal in defined_nonterminals - used_nonterminals:
        print(repr(unused_nonterminal) + ": defined, but not used",
              file=sys.stderr)
    for undefined_nonterminal in used_nonterminals - defined_nonterminals:
        print(repr(undefined_nonterminal) + ": used, but not defined",
              file=sys.stderr)

    # Symbols must be reachable either from <start> or given start symbol
    unreachable = unreachable_nonterminals(grammar, start_symbol)
    msg_start_symbol = start_symbol

    if START_SYMBOL in grammar:
        unreachable = unreachable - \
            reachable_nonterminals(grammar, START_SYMBOL)
        if start_symbol != START_SYMBOL:
            msg_start_symbol += " or " + START_SYMBOL

    for unreachable_nonterminal in unreachable:
        print(repr(unreachable_nonterminal) + ": unreachable from " + msg_start_symbol,
              file=sys.stderr)

    used_but_not_supported_opts = set()
    if len(supported_opts) > 0:
        used_but_not_supported_opts = opts_used(
            grammar).difference(supported_opts)
        for opt in used_but_not_supported_opts:
            print(
                "warning: option " +
                repr(opt) +
                " is not supported",
                file=sys.stderr)

    return used_nonterminals == defined_nonterminals and len(unreachable) == 0
```

