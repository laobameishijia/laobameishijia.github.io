---
title: 毕设-Fuzz-4
date: 2021-11-27 09:25:00
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

## Mutation Analyze

关于ast--`Abstract Syntax Trees` 可算是找到一篇讲的非常清楚的博客了. 看了之后自己又调试了一遍代码,清楚多了

[https://www.cnblogs.com/qiulinzhang/p/14258626.html](https://www.cnblogs.com/qiulinzhang/p/14258626.html)

清楚这个之后,再去搞清楚这一小节写的代码逻辑,想必应该要轻松不少了.

### 7 A Simple mutator for Function

总算搞清楚这个代码的逻辑了

- 第一开始初始化的时候,并没有直接修改源代码,只是做了一个统计. 看一下需要变异的节点有多少个

- 后面provoke generate_mutant函数将location传递进来的时候才会发生突变. 换句话说, 这个location的具体含义是要在第几个变异节点发生变异

- 在 generate_mutant函数中`mutant_ast = self.pm.mutator_object(location).visit(ast.parse(self.pm.src))  # copy`中的`ast.parse(self.pm.src)`就是每一次都把源代码解析的ast对象传递进去, 目的就是为了控制变量. 这样就能获取到每一个单个突变位置的突变. 最后再利用diff库中的函数进行与原始版本的比较.

**作者把类中的函数分开来讲解,确实在对单个函数的说明上起到了一定的作用.但是对于类整体的功能理解上, 还是有一定的影响**

我把单独的代码整合到了一起去看代码的逻辑, 终于清晰了不少

#### 整合代码

```python
class MuFunctionAnalyzer:
    
    def __iter__(self):
        print("__iter__")
        return PMIterator(self)
    
    def __init__(self, fn, log=False):
        self.fn = fn
        self.name = fn.__name__
        src = inspect.getsource(fn) # 获取源码
        self.ast = ast.parse(src)
        self.src = ast.unparse(self.ast)  # normalize
        self.mutator = self.mutator_object()
        self.nmutations = self.get_mutation_count()
        self.un_detected = set()
        self.mutants = []
        self.log = log

    def mutator_object(self, locations=None):
        return StmtDeletionMutator(locations)

    def register(self, m):
        self.mutants.append(m)

    def finish(self):
        pass
    
    def get_mutation_count(self):
        self.mutator.visit(self.ast)
        return self.mutator.count
    
    
class StmtDeletionMutator(ast.NodeTransformer):
    
    def __init__(self, mutate_location=-1):
        self.count = 0
        self.mutate_location = mutate_location
    
    def mutation_visit(self, node): return ast.Pass() 
    def mutable_visit(self, node):
        self.count += 1  # statements start at line no 1
        if self.count == self.mutate_location:
            print("进行替换")
            return self.mutation_visit(node)
        return self.generic_visit(node)
    
    def visit_Return(self, node): 
        print("visit_Return")
        return self.mutable_visit(node)
    def visit_Delete(self, node): return self.mutable_visit(node)

    def visit_Assign(self, node): return self.mutable_visit(node)
    def visit_AnnAssign(self, node): return self.mutable_visit(node)
    def visit_AugAssign(self, node): return self.mutable_visit(node)

    def visit_Raise(self, node): return self.mutable_visit(node)
    def visit_Assert(self, node): return self.mutable_visit(node)

    def visit_Global(self, node): return self.mutable_visit(node)
    def visit_Nonlocal(self, node): return self.mutable_visit(node)

    def visit_Expr(self, node): return self.mutable_visit(node)

    def visit_Pass(self, node): return self.mutable_visit(node)
    def visit_Break(self, node): return self.mutable_visit(node)
    def visit_Continue(self, node): return self.mutable_visit(node)
    
class PMIterator(PMIterator):
    
    def __init__(self, pm):
        self.pm = pm
        self.idx = 0
    
    def __next__(self):
        i = self.idx
        if i >= self.pm.nmutations:
            self.pm.finish()
            raise StopIteration()
        self.idx += 1
        mutant = Mutant(self.pm, self.idx, log=self.pm.log)
        self.pm.register(mutant)
        return mutant

class Mutant:
    def __init__(self, pm, location, log=False):
        self.pm = pm
        #print(pm)
        self.i = location
        self.name = "%s_%s" % (self.pm.name, self.i)
        self._src = None
        self.tests = []
        self.detected = False
        self.log = log

    def generate_mutant(self, location):
        mutant_ast = self.pm.mutator_object(
            location).visit(ast.parse(self.pm.src))  # copy
        return ast.unparse(mutant_ast)
    
    def src(self):
        if self._src is None:
            self._src = self.generate_mutant(self.i)
        return self._src
    def diff(self):
        return '\n'.join(difflib.unified_diff(self.pm.src.split('\n'),
                                              self.src().split('\n'),
                                              fromfile='original',
                                              tofile='mutant',
                                              n=3))    
```

>下面是运行代码, 还有一些解释性的语句, 自认为已经比较清楚了.

#### 运行代码

```python
test = MuFunctionAnalyzer(triangle)
print("test.nmutations:",test.nmutations)
for m in test:
    print("=======这是第%d次变异====================" % m.i)
    print("原始代码：")
    print(mutant.pm.src)
    print("在固定节点变异之后的代码")
    print(m.src())
    print("========================================\n")
===================================================================================
运行结果:

visit_Return
visit_Return
visit_Return
visit_Return
visit_Return
test.nmutations: 5
__iter__
=======这是第1次变异====================
原始代码：
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
在固定节点变异之后的代码
visit_Return
进行替换
visit_Return
visit_Return
visit_Return
visit_Return
def triangle(a, b, c):
    if a == b:
        if b == c:
            pass
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
========================================

=======这是第2次变异====================
原始代码：
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
在固定节点变异之后的代码
visit_Return
visit_Return
进行替换
visit_Return
visit_Return
visit_Return
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            pass
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
========================================

=======这是第3次变异====================
原始代码：
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
在固定节点变异之后的代码
visit_Return
visit_Return
visit_Return
进行替换
visit_Return
visit_Return
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        pass
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
========================================

=======这是第4次变异====================
原始代码：
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
在固定节点变异之后的代码
visit_Return
visit_Return
visit_Return
visit_Return
进行替换
visit_Return
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        pass
    else:
        return 'Scalene'
========================================

=======这是第5次变异====================
原始代码：
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        return 'Scalene'
在固定节点变异之后的代码
visit_Return
visit_Return
visit_Return
visit_Return
visit_Return
进行替换
def triangle(a, b, c):
    if a == b:
        if b == c:
            return 'Equilateral'
        else:
            return 'Isosceles'
    elif b == c:
        return 'Isosceles'
    elif a == c:
        return 'Isosceles'
    else:
        pass
========================================

-----------------------------------
```

### 8 Evaluating Mutations

涉及到了两个函数

- `__enter__()`:The __enter__() function is called when the with block is entered. ***It creates the mutant as a Python function and places it in the global namespace***, such that the assert statement executes the mutated function rather than the original.

  ```python
  def __enter__(self):
      if self.log:
          print('->\t%s' % self.name)
      c = compile(self.src(), '<mutant>', 'exec')
      eval(c, globals()) # 创建全局的变量
  ```

- `__exit__()`:The __exit__() function checks whether an exception has occurred (i.e., the assertion failed, or some other error was raised); if so, it marks the mutation as detected. Finally, it restores the original function definition.

  ```python
  def __exit__(self, exc_type, exc_value, traceback):
      if self.log:
          print('<-\t%s' % self.name)
      if exc_type is not None:
          self.detected = True
          if self.log:
              print("Detected %s" % self.name, exc_type, exc_value)
      globals()[self.pm.name] = self.pm.fn # 因为突变把原来函数给改变了嘛,所以后面又重新把它恢复成原来的样子了
      if self.log:
          print()
      return True

  ```

其他的倒是不难理解了

### 9 Mutator for Modules and Test Suites


#### 问题:

- [x] `self.mutator.visit(self.ast)`  这个函数调用的是AdvMutator里面mutable_visit函数???? 为什么会是这样呢?
  
  A: 你懵了吗, 之前不是探讨过这个问题吗. 并不是self.mutator.visit(self.ast)调用的这个函数, 而是其调用的函数调用的.
  
  ```python
    def visit(self, node):
        """Visit a node."""
        method = 'visit_' + node.__class__.__name__
        visitor = getattr(self, method, self.generic_visit) 
        # 这个函数先访问你自定义的节点visit方法,
        # 如果没有的话, 就递归访问子节点, 
        # 也就是说, 是你自定义的节点visit_XXXX方法调用的mutable_visit. 
        # 你可以再去看看上面
        return visitor(node)
  ```
  
- [x] `__dict__` 在py中,到底起什么样的作用呢? 为什么这个可以实现全局调用?

  A: Python 类提供了 __dict__ 属性。需要注意的一点是，该属性可以用类名或者类的实例对象来调用，用类名直接调用 __dict__，会输出该由类中所有类属性组成的字典；而使用类的实例对象调用 __dict__，会输出由类中所有实例属性组成的字典。

- [x] runtest函数中细节还要再理解

  A: 是unittest类里面的内容, 下次的时候可以再去百度看.

- [x] 再了解一下unittest
  
  A: 就是一个测试类, 也记不住, 下次用到在百度看把

#### 合成代码

> 按照我的理解的话, 这一小节的内容, 就是为了**把第8小节中的变异之后函数运行问题做了简化, 其实本质上还是之前的内容.** 过程中用到了`unittest`这个模块

还是一样的做法, 把代码们先弄到一起

```python 

class MuProgramAnalyzer(MuFunctionAnalyzer):
    def __iter__(self):
        return AdvPMIterator(self)

    def __init__(self, name, src):
        self.name = name
        self.ast = ast.parse(src)
        self.src = ast.unparse(self.ast)
        self.changes = []
        self.mutator = self.mutator_object()
        self.nmutations = self.get_mutation_count() 
        self.un_detected = set()

    def mutator_object(self, locations=None):
        return AdvStmtDeletionMutator(self, locations)

    def get_mutation_count(self):s
        self.mutator.visit(self.ast) # 这个函数调用的是AdvMutator里面mutable_visit函数???? 为什么会是这样呢?
        return self.mutator.count
      
class AdvMutator(Mutator):
    def __init__(self, analyzer, mutate_locations=None):
        self.count = 0
        self.mutate_locations = [] if mutate_locations is None else mutate_locations
        self.pm = analyzer

    def mutable_visit(self, node):
        self.count += 1  # statements start at line no 1
        return self.mutation_visit(node)

class AdvStmtDeletionMutator(AdvMutator, StmtDeletionMutator):
    def __init__(self, analyzer, mutate_locations=None):
        AdvMutator.__init__(self, analyzer, mutate_locations)

    def mutation_visit(self, node):
        index = 0  # there is only one way to delete a statement -- replace it by pass
        
        if not self.mutate_locations:  # counting pass
            self.pm.changes.append((self.count, index))
            return self.generic_visit(node)
        else:
            # get matching changes for this pass
            mutating_lines = set((count, idx)
                                 for (count, idx) in self.mutate_locations)
            if (self.count, index) in mutating_lines:
                return ast.Pass()
            else:
                return self.generic_visit(node)

class AdvPMIterator:
    def __init__(self, pm):
        self.pm = pm
        self.idx = 0
    def __next__(self):
        i = self.idx
        if i >= len(self.pm.changes):
            raise StopIteration()
        self.idx += 1
        # there could be multiple changes in one mutant
        print(self.pm.changes)# 
        return AdvMutant(self.pm, [self.pm.changes[i]])


class AdvMutant(Mutant):
    
    def __init__(self, pm, locations):
        self.pm = pm
        self.i = locations
        self.name = "%s_%s" % (self.pm.name,
                               '_'.join([str(i) for i in self.i]))
        self._src = None    
    
    def __getitem__(self, test_module):
        test_module.__dict__[
            self.pm.name] = import_code(
            self.src(), self.pm.name)
        return MutantTestRunner(self, test_module)    
    
    def generate_mutant(self, locations):
        mutant_ast = self.pm.mutator_object(
            locations).visit(ast.parse(self.pm.src))  # copy
        return ast.unparse(mutant_ast)   
    
    def src(self):
        if self._src is None:
            self._src = self.generate_mutant(self.i)
        return self._src      

class MutantTestRunner:
    def __init__(self, mutant, test_module):
        self.mutant = mutant
        self.tm = test_module

    def runTest(self, tc):
        suite = unittest.TestSuite()
        test_class = self.tm.__dict__[tc]
        for f in test_class.__dict__:
            if f.startswith('test_'):
                suite.addTest(test_class(f))
        runner = unittest.TextTestRunner(verbosity=0, failfast=True)
        try:
            with ExpectTimeout(1):
                res = runner.run(suite)
                if res.wasSuccessful():
                    self.mutant.pm.un_detected.add(self)
                return res
        except SyntaxError:
            print('Syntax Error (%s)' % self.mutant.name)
            return None
        raise Exception('Unhandled exception during test execution')
   
```

### 10 The Problem of Equivalent Mutants

在替换的过程中, 有可能会产生这种情况: 替换过后相当于没有替换. 并不会产生错误. 替换掉了一个无关紧要的语句. 把这种情况称为`equivalent mutants`

要解决这个问题, 文章中说了两个方法

#### 10.1 Statistical Estimation of Number of Equivalent Mutants

利用正态分布

![Statistical Estimation](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211130104504.png)

#### 10.2 Statistical Estimation of the Number of Immortals by Chao's Estimator

![Chao's Estimator](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20211130104746.png)

Note that these **immortal mutants** are somewhat different from the traditional equivalent mutants in that the mortality depends on the oracle used to distinguish variant behavior. That is, if one uses a fuzzer that relies on errors thrown to detect killing, *it will not detect mutants that produce different output but does not throw an error*. Hence, the **Chao1** estimate will essentially be **the asymptote value of mutants the fuzzer can detect** if it is given an infinite amount of time.


### Synopsis

This chapter introduces two methods of running *mutation analysis* on subject programs. The first class `MuFunctionAnalyzer` targets individual functions. Given a function `gcd` and two test cases evaluate, one can run mutation analysis on the test cases as follows ---`第7小节`

The second class `MuProgramAnalyzer` targets standalone programs with test suites. Given a program `gcd` whose source code is provided in `gcd_src` and the test suite is provided by `TestGCD`, one can evaluate the mutation score of `TestGCD` as follows

> 个人感觉这两种方式的差距, 并没有很大. 甚至好像没什么区别-可能是我菜吧😥-🤣


The **mutation score** thus obtained is a better indicator of the quality of a given test suite than pure coverage.

### Lessons Learned

为什么做这个, 怎么做, 这种方法有什么局限.又应该怎么改进.
> 不得不说,作者的思路真的很清晰了.

* We have learned why structural coverage is insufficient to evaluate the quality of test suites.
* We have learned how to use Mutation Analysis for evaluating test suite quality.
* We have learned the limitations of Mutation Analysis -- Equivalent and Redundant mutants, and how to estimate them.