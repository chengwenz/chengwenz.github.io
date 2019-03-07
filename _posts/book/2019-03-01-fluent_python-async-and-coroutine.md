---
layout: post
title: 流畅的python之协程
category: 读书笔记
tags: [python]
keywords: python,coroutine,yield
---

## Coroutine

#### 字典为动词“to yield”给出了两个释义：产出和让步。对于 Python 生成器中的 yield 来说，这两个含义都成立。 yield item 这行代码会产出一个值，提供给 next(...) 的调用方；此外，还会作出让步，暂停执行生成器，让调用方继续工作，直到需要使用另一个值时再调用 next()。调用方会从生成器中拉取值。
#### 从句法上看，协程与生成器类似，都是定义体中包含 yield 关键字的函数。可是，在协程中， yield 通常出现在表达式的右边（例如， datum = yield），可以产出值，也可以不产出——如果 yield 关键字后面没有表达式，那么生成器产出 None。协程可能会从调用方接收数据，不过调用方把数据提供给协程使用的是 .send(datum) 方法，而不是next(...) 函数。通常，调用方会把值推送给协程。
#### yield 关键字甚至还可以不接收或传出数据。不管数据如何流动， yield 都是一种流程控制工具，使用它可以实现协作式多任务：协程可以把控制器让步给中心调度程序，从而激活其他的协程。从根本上把 yield 视作控制流程的方式，这样就好理解协程了。

- Generator作为Coroutine使用时的行为与状态
- 使用装饰器自动预激协程
- 调用方如何使用Generator对象的.throw(...)和.close(...)方法控制协程
- 协程终止时如何返回值
- yield from新句法的用途和语义


### 1. 协程使用时的行为与四个状态。
> 当前状态可以使用inspect.getgeneratorstate(...) 函数确定
- 'GEN_CREATED'
　　等待开始执行。
- 'GEN_RUNNING'
　　解释器正在执行。
- 'GEN_SUSPENDED'
　　在 yield 表达式处暂停。
- 'GEN_CLOSED'
　　执行结束。

    ##### 因为 send 方法的参数会成为暂停的 yield 表达式的值，所以，仅当协程处于暂停状态时才能调用 send 方法，例如 my_coro.send(42)。不过，如果协程还没激活（即，状态是'GEN_CREATED'），情况就不同了。因此，始终要调用 next(my_coro) 激活协程——也可以调用 my_coro.send(None)，效果一样。

##### 示例：使用协程计算移动平均值
    def averager():
        total = 0.0
        count = 0
        average = None
        while True: ➊
            term = yield average ➋
            total += term
            count += 1
            average = total/count

###### ➊ 这个无限循环表明，只要调用方不断把值发给这个协程，它就会一直接收值，然后生成结果。仅当调用方在协程上调用 .close() 方法，或者没有对协程的引用而被垃圾回收程序回收时，这个协程才会终止。
###### ➋ 这里的 yield 表达式用于暂停执行协程，把结果发给调用方；还用于接收调用方后面发给协程的值，恢复无限循环。
##### 使用协程的好处是， total 和 count 声明为局部变量即可，无需使用实例属性或闭包在多次调用之间保持上下文。

### 2. 预激协程的装饰器


    from functools import wraps
    def coroutine(func):
    """装饰器：向前执行到第一个`yield`表达式，预激`func`"""
        @wraps(func)
        def primer(*args,**kwargs): ➊
            gen = func(*args,**kwargs) ➋
            next(gen) ➌
            return gen ➍
        return primer

- ❶ 把被装饰的生成器函数替换成这里的 primer 函数；调用 primer 函数时，返回预激后
的生成器。
- ❷ 调用被装饰的函数，获取生成器对象。
- ❸ 预激生成器。
- ❹ 返回生成器。

##### 很多框架都提供了处理协程的特殊装饰器，不过不是所有装饰器都用于预激协程，有些会提供其他服务，例如勾入事件循环。比如说，异步网络库 Tornado 提供了 tornado.gen装饰器（http://tornado.readthedocs.org/en/latest/gen.html）。使用 yield from 句法调用协程时，会自动预激，因此与示例 中的@coroutine 等装饰器不兼容。 Python 3.4 标准库里的 asyncio.coroutine 装饰器（第18 章介绍）不会预激协程，因此能兼容 yield from 句法。

### 3. 终止协程和异常处理

    ##### 协程中未处理的异常会向上冒泡，传给 next 函数或 send 方法的调用方（即触发协程的对象）。
    ##### 未处理的异常会导致协程终止
     
        
    >>> from coroaverager1 import averager
    >>> coro_avg = averager()
    >>> coro_avg.send(40) # ➊
    40.0
    >>> coro_avg.send(50)
    45.0
    >>> coro_avg.send('spam') # ➋
    Traceback (most recent call last):
    ...
    TypeError: unsupported operand type(s) for +=: 'float' and 'str'
    >>> coro_avg.send(60) # ➌
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    StopIteration

- ❶ 使用 @coroutine 装饰器装饰的 averager 协程，可以立即开始发送值。
- ❷ 发送的值不是数字，导致协程内部有异常抛出。
- ❸ 由于在协程内没有处理异常，协程会终止。如果试图重新激活协程，会抛出StopIteration 异常。
##### 出错的原因是，发送给协程的 'spam' 值不能加到 total 变量上。
##### 示例暗示了终止协程的一种方式：发送某个哨符值，让协程退出。内置的 None 和Ellipsis 等常量经常用作哨符值。 Ellipsis 的优点是，数据流中不太常有这个值。我还见过有人把 StopIteration 类（类本身，而不是实例，也不抛出）作为哨符值；也就是说，是像这样使用的： my_coro.send(StopIteration)。

##### 从 Python 2.5 开始，客户代码可以在生成器对象上调用两个方法，显式地把异常发给协程。
##### 这两个方法是 throw 和 close。
##### generator.throw(exc_type[, exc_value[, traceback]])
> 致使生成器在暂停的 yield 表达式处抛出指定的异常。如果生成器处理了抛出的异常，代码会向前执行到下一个 yield 表达式，而产出的值会成为调用 generator.throw方法得到的返回值。如果生成器没有处理抛出的异常，异常会向上冒泡，传到调用方的上下文中

##### generator.close()
> 致使生成器在暂停的 yield 表达式处抛出 GeneratorExit 异常。如果生成器没有处理这个异常，或者抛出了 StopIteration 异常（通常是指运行到结尾），调用方不会报错。如果收到 GeneratorExit 异常，生成器一定不能产出值，否则解释器会抛出RuntimeError 异常。生成器抛出的其他异常会向上冒泡，传给调用方。

* 示例：学习在协程中处理异常的测试代码


    class DemoException(Exception):
    """为这次演示定义的异常类型。 """
    def demo_exc_handling():
        print('-> coroutine started')
        while True:
            try:
                x = yield
            except DemoException: ➊
                print('*** DemoException handled. Continuing...')
            else: ➋
                print('-> coroutine received: {!r}'.format(x))
        raise RuntimeError('This line should never run.') ➌
        
- ❶ 特别处理 DemoException 异常。
- ❷ 如果没有异常，那么显示接收到的值。
- ❸ 这一行永远不会执行。
##### 示例中的最后一行代码不会执行，因为只有未处理的异常才会中止那个无限循环，而一旦出现未处理的异常，协程会立即终止。demo_exc_handling 函数的常规用法如下所示。

* 示例：激活和关闭 demo_exc_handling，没有异常


    >>> exc_coro = demo_exc_handling()
    >>> next(exc_coro)
    -> coroutine started
    >>> exc_coro.send(11)
    -> coroutine received: 11
    >>> exc_coro.send(22)
    -> coroutine received: 22
    >>> exc_coro.close()
    >>> from inspect import getgeneratorstate
    >>> getgeneratorstate(exc_coro)
    'GEN_CLOSED'
    
##### 如果把 DemoException 异常传入 demo_exc_handling 协程，它会处理，然后继续运行，如下所示。
* 示例：把 DemoException 异常传入 demo_exc_handling 不会导致协程中止


    >>> exc_coro = demo_exc_handling()
    >>> next(exc_coro)
    -> coroutine started
    >>> exc_coro.send(11)
    -> coroutine received: 11
    >>> exc_coro.throw(DemoException)
    *** DemoException handled. Continuing...
    >>> getgeneratorstate(exc_coro)
    'GEN_SUSPENDED'

##### 但是，如果传入协程的异常没有处理，协程会停止，即状态变成 'GEN_CLOSED'。下面示例 演示了这种情况。
* 示例：如果无法处理传入的异常，协程会终止


    >>> exc_coro = demo_exc_handling()
    >>> next(exc_coro)
    -> coroutine started
    >>> exc_coro.send(11)
    -> coroutine received: 11
    >>> exc_coro.throw(ZeroDivisionError)
    Traceback (most recent call last):
    ...
    ZeroDivisionError
    >>> getgeneratorstate(exc_coro)
    'GEN_CLOSED'

##### 如果不管协程如何结束都想做些清理工作，要把协程定义体中相关的代码放入try/finally 块中，如下面示例。
* 示例： coro_finally_demo.py：使用 try/finally 块在协程终止时执行操作


    class DemoException(Exception):
    """为这次演示定义的异常类型。 """
        def demo_finally():
            print('-> coroutine started')
            try:
                while True:
                    try:
                        x = yield
                    except DemoException:
                        print('*** DemoException handled. Continuing...')
                    else:
                        print('-> coroutine received: {!r}'.format(x))
            finally:
                print('-> coroutine ending')

##### Python 3.3 引入 yield from 结构的主要原因之一与把异常传入嵌套的协程有关。另一个原因是让协程更方便地返回值。

### 4. 让协程返回值
#####    下面示例是 averager 协程的不同版本，这一版会返回结果。为了说明如何返回值，每次激活协程时不会产出移动平均值。这么做是为了强调某些协程不会产出值，而是在最后返回一个值（通常是某种累计值）。
#####    下面示例 中的 averager 协程返回的结果是一个 namedtuple，两个字段分别是项数（count）和平均值（average）。我本可以只返回平均值，但是返回一个元组可以获得累积数据的另一个重要信息——项数。
* 示例 16-13 coroaverager2.py：定义一个求平均值的协程，让它返回一个结果


    from collections import namedtuple
    Result = namedtuple('Result', 'count average')
    def averager():
        total = 0.0
        count = 0
        average = None
        while True:
            term = yield
            if term is None:
                break ➊
            total += term
            count += 1
            average = total/count
        return Result(count, average) ➋
- ➊ 为了返回值，协程必须正常终止；因此，这一版 averager 中有个条件判断，以便退
出累计循环。
- ➋ 返回一个 namedtuple，包含 count 和 average 两个字段。在 Python 3.3 之前，如果
生成器返回值，解释器会报句法错误

##### 下面在控制台中说明如何使用新版 averager，如示例 16-14 所示。
* 示例： coroaverager2.py：说明 averager 行为的 doctest


    >>> coro_avg = averager()
    >>> next(coro_avg)
    >>> coro_avg.send(10) ➊
    >>> coro_avg.send(30)
    >>> coro_avg.send(6.5)
    >>> coro_avg.send(None) ➋
    Traceback (most recent call last):
    ...
    StopIteration: Result(count=3, average=15.5)
- ❶ 这一版不产出值。
- ❷ 发送 None 会终止循环，导致协程结束，返回结果。一如既往，生成器对象会抛出StopIteration 异常。异常对象的 value 属性保存着返回的值。
> 注意， return 表达式的值会偷偷传给调用方，赋值给 StopIteration 异常的一个属性。这样做有点不合常理，但是能保留生成器对象的常规行为——耗尽时抛出StopIteration 异常。

##### 下面示例展示如何获取协程返回的值。
* 示例：　捕获 StopIteration 异常，获取 averager 返回的值


    >>> coro_avg = averager()
    >>> next(coro_avg)
    >>> coro_avg.send(10)
    >>> coro_avg.send(30)
    >>> coro_avg.send(6.5)
    >>> try:
    ...     coro_avg.send(None)
    ... except StopIteration as exc:
    ...     result = exc.value
    ...
    >>> result
    Result(count=3, average=15.5)
##### 获取协程的返回值虽然要绕个圈子，但这是 PEP 380 定义的方式，当我们意识到这一点之后就说得通了： yield from 结构会在内部自动捕获 StopIteration 异常。这种处理方式与 for 循环处理 StopIteration 异常的方式一样：循环机制使用用户易于理解的方式处理异常。对 yield from 结构来说，解释器不仅会捕获 StopIteration 异常，还会把value 属性的值变成 yield from 表达式的值。可惜，我们无法在控制台中使用交互的方式测试这种行为，因为在函数外部使用 yield from（以及 yield）会导致句法出错。

### 5. yield from

##### 首先要知道， yield from 是全新的语言结构。它的作用比 yield 多很多，因此人们认为继续使用那个关键字多少会引起误解。在其他语言中，类似的结构使用 await 关键字，这个名称好多了，因为它传达了至关重要的一点：在生成器 gen 中使用 yield from subgen() 时， subgen会获得控制权，把产出的值传给 gen 的调用方，即调用方可以直接控制 subgen。与此同时， gen 会阻塞，等待 subgen 终止。
##### yield from 可用于简化 for 循环中的 yield 表达式。例如：
    >>> def gen():
    ...     for c in 'AB':
    ...         yield c
    ...     for i in range(1, 3):
    ...         yield i
    ...
    >>> list(gen())
    ['A', 'B', 1, 2]
##### 可以改写为：
    >>> def gen():
    ...     yield from 'AB'
    ...     yield from range(1, 3)
    ...
    >>> list(gen())
    ['A', 'B', 1, 2]
##### yield from x 表达式对 x 对象所做的第一件事是，调用 iter(x)，从中获取迭代器。因此， x 可以是任何可迭代的对象。
##### 可是，如果 yield from 结构唯一的作用是替代产出值的嵌套 for 循环，这个结构很有可能不会添加到 Python 语言中。 yield from 结构的本质作用无法通过简单的可迭代对象说明，而要发散思维，使用嵌套的生成器。因此，引入 yield from 结构的 PEP 380 才起了“Syntax for Delegating to a Subgenerator”（“把职责委托给子生成器的句法”）这个标题。
##### yield from 的主要功能是打开双向通道，把最外层的调用方与最内层的子生成器连接起来，这样二者可以直接发送和产出值，还可以直接传入异常，而不用在位于中间的协程中添加大量处理异常的样板代码。有了这个结构，协程可以通过以前不可能的方式委托职责。
##### 若想使用 yield from 结构，就要大幅改动代码。为了说明需要改动的部分， PEP 380 使用了一些专门的术语。

> 委派生成器
    - 包含 yield from <iterable> 表达式的生成器函数。

> 子生成器
    - 从 yield from 表达式中 <iterable> 部分获取的生成器。这就是 PEP 380 的标题
（“Syntax for Delegating to a Subgenerator”）中所说的“子生成器”（subgenerator）。

> 调用方
    - PEP 380 使用“调用方”这个术语指代调用委派生成器的客户端代码。在不同的语境
中，我会使用“客户端”代替“调用方”，以此与委派生成器（也是调用方，因为它调用了子
生成器）区分开。

>> PEP 380 经常使用“迭代器”这个词指代子生成器。这样会让人误解，因为委派
生成器也是迭代器。因此，我选择使用“子生成器”这个术语，与 PEP 380 的标题
（“Syntax for Delegating to a Subgenerator”）保持一致。然而，子生成器可能是简单的
迭代器，只实现了 __next__ 方法；但是， yield from 也能处理这种子生成器。不
过，引入 yield from 结构的目的是为了支持实现了 __next__、 send、 close 和
throw 方法的生成器。



    




