---
title: C++防坑（一）
date: 2018-05-05 15:11:09
tags: CPP
categories: 编程语言
---



### 引言

最近项目开发中遇到一些的小细节。很多时候我们明白导致问题的原理，但是在开发中还是会不小心犯错。这里列一下最近自己遇到的和看到的组内发现的错误。



### 引用失效

在一个模块中使用了vector来模拟队列，模块内一个线程以一定的频率从begin向end扫描，以处理那些达到条件的元素。处理的时候由于首先要判断条件是否满足，于是开发的时候手残的使用了引用。伪代码如下：

<!--more-->

``` c++
mutex.lock();
for (size_t i = 0; i < vec.size(); ++i) {
    Element &ele = vec[i];
    mutex.unlock();
    bool handled = CheckShouldHandle(ele);  // 调用外部模块去检查
    mutex.lock();
    if (handled) {
        Handle(ele);
        vec.erase(vec.begin() + i);
    }
}
mutex.unlock();
```

而调用该模块的地方会有很多，其他模块会将element添加到vector中。基本如下：

```c++
void Add(const Tlement &ele) {
    mutex.lock();
 	  vec.push_back(ele);
    mutex.unlock();
}
```

这就是产生问题的地方。调用外部模块时解开了锁，Add调用push_back，我们知道vector在capcity满了之后重新分配，于是原来的引用就失效了。

这里实际上还有另一个问题，算法复杂度会在erase中间元素时由于移动后续元素成为O(n2)。

### 越界

实际上这个错误大家说了太多了，但是还是不小心就会中招。

我们的一个服务使用是由两个团队开发的：算法策略团队和架构团队。

其中一段代码大概是这样写的：

```c++
std::vector<Element> items = Retrive();
int start_index = -1;
.
.
.

auto ele = items[start_index];
```

策略团队经过各种计算选择来确定start_index。但是实际上计算过程某些分支没有给start_index赋值, 最后使用start_index的默认值-1，于是core了。


### shared_ptr错误使用

简单来说：shared_ptr保存的pointer在多线程下的安全性与shared_ptr没有关系

1. shared_ptr多线程的安全性要靠atomic_load, atomic_store来保证
2. raw pointer自己保证各种成员调用的安全性



以上
