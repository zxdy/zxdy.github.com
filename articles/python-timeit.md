其实我本意是就贴一段代码完事，但又觉得确实有点干巴巴，还是稍微灌点水吧。

way1和way2分别代表了两种不同的计时方式，同时也展现了两种不同的用途。

way1是使用了timeit包。Python 社区有句俗语："Python 自己带着电池。" 别自己写计时框架。Python 2.7 具备一个叫做 timeit 的完美计时工具。看demo代码可以发现，它不仅支持对单个方法计时，还支持方法传参，更支持重复计时。我觉得这个计时方法更加适用于测试，特别是性能测试的时候。

way2没什么好说的，一种很简单粗暴傻瓜化的计时方法，换种编程语言也是相同的套路。这个计时方法相比way1来说，更加浅显易懂，操作也简单，就是代码量相对较多，不够简洁。但它也不是没有市场，比较适合在python脚本中写日志。

```python
#!/usr/bin/env python
import time
from timeit import Timer
def func1(x):
    sum=0
    for i in  xrange(x*10000):
        sum=sum+i

def func2(x,y):
    sum=0
    for i in  xrange((x+y)*10000):
        sum=sum+i

if __name__ == '__main__':
    #way1
    t1=Timer("func1(2)","from __main__ import func1")
    t2=Timer("func2(3,4)","from __main__ import func2")
    print(t1.timeit(1))
    print(t2.timeit(1))
    print(t1.repeat(4,10)) //第一个参数是重复整个测试的次数，第二个参数是每个测试中调用被计时语句的次数

    #way2
    start = time.clock()
    func1(2)
    elapsed = (time.clock() - start)
    print("Time used:", elapsed)
```