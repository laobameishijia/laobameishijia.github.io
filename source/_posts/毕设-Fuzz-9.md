---
title: 毕设-Fuzz-9
date: 2021-12-6 09:25:00
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

# NoteBook阅读

## Probabilistic Grammar Fuzzing

我们可以根据我们的需要去生成测试数据, 可以给某个`expansion`很高的权重, 这样的话, 在生成的数据过程中, 这个`expansion`被选择的概率就很高. 最终数据中的占比也就比较高.

> 下面原文的意思也就是上面所说的

### Synopsis

To [use the code provided in this chapter](Importing.ipynb), write

```python
>>> from fuzzingbook.ProbabilisticGrammarFuzzer import <identifier>
```

and then make use of the following features.


A _probabilistic_ grammar allows to attach individual _probabilities_ to production rules.  To set the probability of an individual expansion `S` to the value `X` (between 0 and 1), replace it with a pair

```python
(S, opts(prob=X))
```

If we want to ensure that 90% of phone numbers generated have an area code starting with `9`, we can write:

```python
>>> from Grammars import US_PHONE_GRAMMAR, extend_grammar, opts
>>> PROBABILISTIC_US_PHONE_GRAMMAR: Grammar = extend_grammar(US_PHONE_GRAMMAR,
>>> {
>>>       "<lead-digit>": [
>>>                           "2", "3", "4", "5", "6", "7", "8",
>>>                           ("9", opts(prob=0.9))
>>>                       ],
>>> })
```
A `ProbabilisticGrammarFuzzer` will extract and interpret these options.  Here is an example:

```python
>>> probabilistic_us_phone_fuzzer = ProbabilisticGrammarFuzzer(PROBABILISTIC_US_PHONE_GRAMMAR)
>>> [probabilistic_us_phone_fuzzer.fuzz() for i in range(5)]
['(918)925-2501',
 '(981)925-0792',
 '(934)995-5029',
 '(955)999-7801',
 '(964)927-0877']
```
As you can see, the large majority of area codes now starts with `9`.

### Learning Probabilities from Samples

对于`expasion`的权重来讲, 必须要通过人工设定. 我们可以通过学习样本数据来设定这样的权重.

Probabilities need not be set manually all the time.  They can also be _learned_ from other sources, notably by counting _how frequently individual expansions occur in a given set of inputs_.  This is useful in a number of situations, including:

1. Test _common_ features.  The idea is that during testing, one may want to focus on frequently occurring (or frequently used) features first, to ensure correct functionality for the most common usages.
2. Test _uncommon_ features.  Here, the idea is to have test generation focus on features that are rarely seen (or not seen at all) in inputs.  This is the same motivation as with [grammar coverage](GrammarCoverageFuzzer.ipynb), but from a probabilistic standpoint.
3. Focus on specific _slices_.  One may have a set of inputs that is of particular interest (for instance, because they exercise a critical functionality, or recently have discovered bugs).  Using this learned distribution for fuzzing allows us to _focus_ on precisely these functionalities of interest.

### Lessons Learned

* By specifying probabilities, one can steer fuzzing towards input features of interest.
* Learning probabilities from samples allows one to focus on features that are common or uncommon in input samples.
* Learning probabilities from a subset of samples allows one to produce more similar inputs.