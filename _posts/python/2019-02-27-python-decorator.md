---
layout: post
title: python中装饰器的设计
category: 技术
tags: [python]
keywords: python,decorator
---

## Python装饰器原理

下面是 Python 装饰器的常规写法：

    @decorator
    def func(*args, **kwargs):
        do_something()

这种写法只是一种语法糖，使得代码看起来更加简洁而已，在 Python 解释器内部，函数 func 的调用被转换为下面的方式：

    >>> func(a, b, c='value')
    # 等价于
    >>> decorated_func = decorator(func)
    >>> decorated_func(a, b, c='value')

可见，装饰器 decorator 是一个函数（当然也可以是一个类），它接收被装饰的函数 func 作为唯一的参数，然后返回一个 callable（可调用对象），对被装饰函数 func 的调用实际上是对返回的 callable 对象的调用。

## 自顶而下设计装饰器

从原理分析可见，如果我们要设计一个装饰器，将原始的函数（或类）装饰成一个功能更加强大的函数（或类），那么我们要做的就是要写一个函数（或类），其被调用后返回我们需要的那个功能更加强大的函数（或类）。

### 简单装饰器

简单的装饰器函数就像上面介绍的那样，不带任何参数。假设我们要设计一个装饰器函数，其功能是能使得被装饰的函数调用结束后，打印出函数运行时间，我们来看看使用自顶而下的方法来设计这个装饰器该怎么做。

所谓“顶”，就是先不关注实现细节，而是做好整体设计和分解函数调用过程。我们把装饰器命名为 timethis，其使用方法像下面这样：

    @timethis
    def fun(*args, **kwargs):
        pass

分解对被装饰函数 fun 的调用过程：

    >>> func(*args, **kwargs)
    # 等价于
    >>> decorated_func = timethis(func)
    >>> decorated_func(a, b, c='value')

由此可见，我们的装饰器 timethis 应该接收被装饰的函数作为唯一参数，返回一个函数对象，根据惯例，返回的函数命名为 wrapper，因此可以写出 timethis 装饰器的模板代码：

    def timethis(func):
        def wrapper(*args, **kwargs):
            pass

        return wrapper

装饰器的框架搭好了，接下来就是“下”，丰富函数逻辑。

对被装饰的函数调用等价于对 wrapper 函数的调用，为了使 wrapper 调用返回和被装饰函数调用一样的结果，我们可以在 wrapper 中调用原函数并返回其调用结果：

    def timethis(func):
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            return result
        return wrapper

可以随意丰富 wrapper 函数的逻辑，我们的需求是打印 func 的调用时间，只需在 func 调用前后计时即可：

    import time
    def timethis(func):
        def wrapper(*args, **kwargs):
            start = time.time()
            result = func(*args, **kwargs)
            end = time.time()
            print(func.__name__, end-start)
            return result
        return wrapper

由此，一个可以打印函数调用时间的装饰器就完成了，来看看使用效果：

    @timethis
    def fibonacci(n):
        """
        求斐波拉契数列第 n 项的值
        """
        a = b = 1
        while n > 2:
            a, b = b, a + b
            n -= 1
        return b
    ​
    >>> fibonacci(10000)
    fibonacci 0.004000663757324219
    ...结果太大省略

基本上看上去没有问题了，不过由于函数被装饰了，因此被装饰函数的基本信息变成了装饰器返回的 wrapper 函数的信息：

    >>> fibonacci.__name__
    wrapper
    >>> fibonacci.__doc__
    None
    
>注意这里 fibonacci.__name__ 等价于 timethis(fibonacci).__name__，所以返回值为 wrapper。

修正方法也很简单，需要使用标准库中提供的一个 wraps 装饰器，将被装饰函数的信息复制给 wrapper 函数

    from functools import wraps
    import time
    ​
    def timethis(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.time()
            result = func(*args, **kwargs)
            end = time.time()
            print(func.__name__, end - start)
            return result
    ​
        return wrapper

至此，一个完整的，不带参数的装饰器便写好了。

### 带参数的装饰器
    
上面设计的装饰器比较简单，不带任何参数。我们也会经常看到带参数的装饰器，其使用方法大概如下：

    @logged('debug', name='example', message='message')
    def fun(*args, **kwargs):
        pass
分解对被装饰函数 fun 的调用过程：

    >>> func(a, b, c='value')
    # 等价于
    >>> decorator = logged('debug', name='example', message='message')
    >>> decorated_func = decorator(func)
    >>> decorated_func(a, b, c='value')

由此可见，logged 是一个函数，它返回一个装饰器，这个返回的装饰器再去装饰 func 函数，因此 logged 的模板代码应该像这样：

    def logged(level, name=None, message=None):
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                pass
            return wrapper
        return decorator

wrapper 是最终被调用的函数，我们可以随意丰富完善 decorator 和 wrapper 的逻辑。假设我们的需求是被装饰函数 func 被调用前打印一行 log 日志，代码如下：

    from functools import wraps

    def logged(level, name=None, message=None):
        def decorator(func):
            logname = name if name else func.__module__
            logmsg = message if message else func.__name__
    ​
            @wraps(func)
            def wrapper(*args, **kwargs):
                print(level, logname, logmsg, sep=' - ')
                return func(*args, **kwargs)
            return wrapper
        return decorator

### 多功能装饰器

有时候，我们也会看到同一个装饰器有两种使用方法，可以像简单装饰器一样使用，也可以传递参数。例如：

    @logged
    def func(*args, **kwargs):
        pass
    ​
    @logged(level='debug', name='example', message='message')
    def fun(*args, **kwargs):
        pass
        
根据前面的分析，不带参数的装饰器和带参数的装饰器定义是不同的。不带参数的装饰器返回的是被装饰后的函数，带参数的装饰器返回的是一个不带参数的装饰器，然后这个返回的不带参数的装饰器再返回被装饰后的函数。那么怎么统一呢？先来分析一下两种装饰器用法的调用过程。

    # 使用 @logged 直接装饰
    >>> func(a, b, c='value')
    # 等价于
    >>> decorated_func = logged(func)
    >>> decorated_func(a, b, c='value')
    ​
    # 使用 @logged(level='debug', name='example', message='message') 装饰
    >>> func(a, b, c='value')
    # 等价于
    >>> decorator = logged(level='debug', name='example', message='message')
    >>> decorated_func = decorator(func)
    >>> decorated_func(a, b, c='value')

可以看到，第二种装饰器比第一种装饰器多了一步，就是调用装饰器函数再返回一个装饰器，这个返回的装饰器和不带参数的装饰器是一样的：接收被装饰的函数作为唯一参数。唯一的区别是返回的装饰器携带固定参数，固定函数参数正是 partial 函数的使用场景，因此我们可以定义如下的装饰器
    
    from functools import wraps, partial

    def logged(func=None, *, level='debug', name=None, message=None):
        if func is None:
            return partial(logged, level=level, name=name, message=message)
    ​
        logname = name if name else func.__module__
        logmsg = message if message else func.__name__
    ​
        @wraps(func)
        def wrapper(*args, **kwargs):
            print(level, logname, logmsg, sep=' - ')
            return func(*args, **kwargs)
        return wrapper

实现的关键在于，若这个装饰器以带参数的形式使用，这第一个参数 func 的值为 None，此时我们使用 partial 返回了一个其它参数固定的装饰器，这个装饰器与不带参数的简装饰器一样，接收被装饰的函数对象作为唯一参数，然后返回被装饰后的函数对象。

### 装饰类

1. 由于类的实例化和函数调用非常类似，因此装饰器函数也可以用于装饰类，只是此时装饰器函数的第一个参数不再是函数，而是类。基于自顶而下的设计方法，设计一个用于装饰类的装饰器函数就是轻而易举的事情，这里不再给出示例。
2. 相比函数装饰器，类装饰器具有灵活度大、高内聚、封装性等优点。使用类装饰器主要依靠类的__call__方法，当使用 @ 形式将装饰器附加到函数上时，就会调用此方法。

        class Foo(object):
        def __init__(self, func):
            self._func = func

        def __call__(self):
            print ('class decorator runing')
            self._func()
            print ('class decorator ending')

        @Foo
        def bar():
            print ('bar')

        bar()

### 练习

最后，以当时今日头条的面试题作为一个练习。现在看来这道题只是一个简单的装饰器设计需求，只怪自己学艺不精，后悔没有早点掌握装饰器的设计方法。

> 题目：
设计一个装饰器函数 retry，当被装饰的函数调用抛出指定的异常时，函数会被重新调用，直到达到指定的最大调用次数才重新抛出指定的异常。装饰器的使用示例如下：

    @retry(times=10, traced_exceptions=ValueError, reraised_exception=CustomException)
    def str2int(s):
    pass
times 为函数被重新调用的最大尝试次数。
traced_exceptions 为监控的异常，可以为 None（默认）、异常类、或者一个异常类的列表。如果为 None，则监控所有的异常；如果指定了异常类，则若函数调用抛出指定的异常时，重新调用函数，直至成功返回结果或者达到最大尝试次数，此时重新抛出原异常（reraised_exception 的值为 None），或者抛出由 reraised_exception 指定的异常。

#### 参考代码
    
要注意实现方式不止一种，以下是我的实现版本：

    from functools import wraps

    def retry(times, traced_exceptions=None, reraise_exception=None):
        def decorator(func):
            
            @wraps(func)
            def wrapper(*args, **kwargs):
                n = times
                trace_all = traced_exceptions is None
                trace_specified = traced_exceptions is not None
                while True:
                    try:
                        return func(*args, **kwargs)
                    except Exception as e:
                        traced = trace_specified and isinstance(e, traced_exceptions)
                        reach_limit = n == 0
    ​
                        if not (trace_all or traced) or reach_limit:
                            if reraise_exception is not None:
                                raise reraise_exception
                            raise
                        n -= 1
            return wrapper
        return decorator

### 总结

总结一下，自定而下设计装饰器分以下几个步骤

##### 1.确定你的装饰器该如何使用，带参数或者不带参数，还是都可以。
##### 2.将 @ 语法糖分解为装饰器的实际调用过程。
##### 3.根据装饰的调用过程，写出对应的模板代码。
##### 4.根据需求编写装饰器函数和装饰后函数的逻辑。
##### 5.完工！