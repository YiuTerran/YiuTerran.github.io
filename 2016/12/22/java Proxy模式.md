# java Proxy模式
tags: [design_pattern,java,proxy,reflect]

---

java也可以使用反射生成动态代理，从而完成面向切面编程，这是spring框架的基础。
由于java的第一元素只有object，所以函数处于第二阶梯，这导致在其他语言中很容易(函数是第一类的语言中）实现的动态代理，在java中就必须以对象的形式实现。

在Python中使用`getattr(object, method)`可以轻易完成反射；最简单的可以这么做：
```py
import inspect

class Proxy(object):
    def __init__(self):
        self._real = Real()
        for method in inspect.getmembers(_real, predicate=inspect.ismethod):
            setattr(self, method[0], self._call_real(m[0])

    def _call_real(self, method):
        def _real_proxy(*args, **kargs):
            return getattr(self._real, method)(*args, **kargs)
        return _real_proxy
```

java生成动态代理的主要使用了`java.lang.reflect.Proxy`类的`newProxyInstance()`方法，其原型如下：
```
public static Object newProxyInstance(ClassLoader loader,
                                        Class<?>[] interfaces,
                                        InvacationHandler h)throws IllegalArgumentException;
```
参数：
loader - 定义代理类的类加载器
interfaces - 代理类要实现的接口列表
h - 指派方法调用的调用处理程序
返回：
一个带有代理类的指定调用处理程序的代理实例，它由指定的类加载器定义，并实现指定的接口
抛出：
IllegalArgumentException - 如果违反传递到 getProxyClass 的参数上的任何限制
NullPointerException - 如果 interfaces 数组参数或其任何元素为 null，或如果调用处理程序 h 为 null

动态代理类：在运行时生成的class，在其生成过程中，你必须提供一组接口给它，然后该class就声称实现了这些接口。可以把该class的实例当做这些接口中的任何一个来用。其实，这个Dynamic Proxy就是一个Proxy，他不会替你做任何实质性的工作。在生成它的实例时，必须提供一个Handler，由它接管实际的工作。
在使用动态代理类时，必须实现InvocationHandler接口。

`InvaocationHandler`接口必须实现的方法是
```
Object invoke(Object proxy,
              Method method,
              Object[] args)
              throws Throwable
```
此处控制流翻转，该方法被回调，传入用户尝试调用的方法和参数，以及代理本身的实例。


