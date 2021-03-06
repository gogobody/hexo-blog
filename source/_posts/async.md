---
title: 小议async/await和coroutine
date: 2018-05-21 19:37:14
tags: 
    - javascript
    - kotlin
    - go
---

> Being happy doesn't mean that everything is perfect. It means that you decided to look beyond the imperfections.

后端编程，涉及最多的就是并发。简单理解就是：

> 并发是同时管理多个任务去执行，并行是针对多核处理器，同时执行多个任务。可以理解为一个是manage，一个是run。

并发一般特指IO，IO是独立于CPU的设备，IO设备通常远远慢于CPU，所以我们引入了并发的概念，让CPU可以一次性发起多个IO操作而不用等待IO设备做完一个操作再做令一个。原理就是非阻塞操作+事件通知。

硬件底层上我其实不关心，主要就是在写程序上，如何简单的去写并发的代码。在语法层面上对并发做的比较好的，很适合做服务端，比如go，比如node，又比如某些函数式语言。我最近最近主要使用的是node和kotlin。

那么在写并发代码的时候，就会时不时的想这样一个问题：

## 一个问题

当代码遇到一个“暂时不能完成”的流程时（例如建立一个tcp链接，可能需要5ms才能建立），他不想阻塞在这里睡眠，想暂时离开现场去干点别的事情（例如看看另外一个已经建立的链接是否可以收包了）。问题是：离开现场后，当你回来的时候，上下文还像你走的时候吗？

 跳转离开，在任何语言里都有2种最基本的方法：1）从当前函数返回； 2）调用一个新的函数。 前者会把上下文中的局部变量和函数参数全部摧毁，除非他返回前把这些变量找个别的地方保存起来；后者则能保护住整个上下文的内存（除了协程切换后会摧毁一些寄存器），而且跳转回来也是常规方法：函数返回。 

在写node的时候，基本上是无脑上async/await。每次看到回调函数的时候，强迫症就犯了，总是想方设法将那个方法转成promise，然后使用await获得结果。无脑尝试了bluebird和node的util，虽然有些是很好用的，但是有的还是无法达到我预期的。靠着无脑的async/await，实现了很多功能，代码写起来也是快的飞起，但是只顾着做业务而不深入思考的话，是一个不好的表现，所以我就停下来搜了很多async/await的东西，特别是从阮一峰老师那里收获了很多。

## js异步编程

因为js是单线程，所以异步编程对js特别重要。

实现异步主要有如下几种：

- 回调函数

  callback，英语直译就是重新调用。

  所谓的回调函数就是把任务的第二段单独写在一个函数里面，等到重新执行这个任务的时候，直接调用这个函数。

  回调本身没问题，但是就怕多重嵌套。

- promise

  promise是一种新的写法，把回调函数的横向嵌套，用then的形式改成纵向的加载。

- 协程

  协程就是比线程更小的单位。

  执行过程大致如下：

  第一步，协程A开始执行。

  第二步，协程A执行到一半，进入暂停，执行权转移到协程B。

  第三步，（一段时间后）协程B交还执行权。

  第四步，协程A恢复执行。

  后面再展开说协程。

很明显，在go火起来之后，很多编程语言都在往协程上靠，因为协程很好的将异步的写法转化成了同步的写法，降低了心智负担。js当然也不落后。

js的异步写法的演进

- generator

  es6增加了generator函数，就是协程的一种实现，最大特点就是使用yield关键字就是用来交出函数的执行权。

  ```
  function* gen(x){
    var y = yield x + 2;
    return y;
  }
  ```

  不同于普通函数的地方在于调用generator函数的时候，不返回结果，而是会返回一个内部的指针。调用指针的next方法，会移动内部指针（即执行异步任务的第一段），遇到的yield语句就交出执行权，执行别的代码。下次再调用该函数指针的next方法，就继续执行到该函数的下一个yield语句。

  

  虽然 Generator 函数将异步操作表示得很简洁，但是流程管理却不方便（即何时执行第一阶段、何时执行第二阶段），这样看来其实generator函数就是一个异步操作的容器，需要有一个触发它自动执行的机制。

- Thunk函数

  说到thunk函数，就得先了解一下参数的求值策略。

  ```
  let m=1;
  function f(x){
      return x*2
  }
  
  f(m+5)
  ```

  - 传值调用

    先计算出来m+5的值6，然后再将值传给函数f，即6*2

  - 传名调用

    把m+5传入到f中，在用到的时候再计算，即(x+5)*2。

  传值调用比较简单，但是对参数求值的时候，实际上还没用到这个参数，有可能造成性能损失。

  编译器的"传名调用"实现，往往是将参数放到一个临时函数之中，再将这个临时函数传入函数体。这个临时函数就叫做 Thunk 函数。

  js是传值调用。他的thunk函数是将多参数的函数，替换成了单参数的版本，而且只接受回调函数作为参数。

  这样就可以很方便的实现了基于thunk函数的generator自动执行器。

  具体的实现和如何使用，参考http://www.ruanyifeng.com/blog/2015/05/thunk.html。

- co函数

  co函数是基于Promise的generator函数的自动执行器。

  源代码只有几十行，tj大神太强👍了。

  http://www.ruanyifeng.com/blog/2015/05/co.html

- async/await

  async函数就是generator函数的语法糖。

  async函数自带执行器，无脑写async和await的时候就是，几乎所有的函数都写成了async函数，只要需要等待的方法，都用await去等待，这样就造成了很多无意义的等待。本来两个不相干的操作，如果每个都是用await等的话，就会很影响性能。

  多个请求并发执行的时候，尽量选用Promise.all方法。

理解了以上的演进过程，感觉自己终于摆脱了java思维的枪，对node终于入门了。然后，同步地写着kotlin项目，又陷入了泥潭中。

## Future、RxJava、Actor和kotlin协程

我理解的也不是很深，求科普。

以前写java的时候，自己都是无脑用线程池，开多线程去处理，一般这种情况下不需要线程的结果。

- future

  因为不能直接从别的线程中得到函数的返回值，所以future就出场了。

  Futrue可以监视目标线程调用call的情况，当你调用Future的get()方法以获得结果时，当前线程就开始阻塞，直接call方法结束返回结果。 

  Future对象本身可以看作是一个显式的引用，一个对异步处理结果的引用。由于其异步性质，在创建之初，它所引用的对象可能还并不可用（比如尚在运算中，网络传输中或等待中）。这时，得到Future的程序流程如果并不急于使用Future所引用的对象，那么它可以做其它任何想做的事儿，当流程进行到需要Future背后引用的对象时，可能有两种情况：

  - 希望能看到这个对象可用，并完成一些相关的后续流程。

    可以通过调用Future.isDone()判断引用的对象是否就绪，并采取不同的处理。

  - 如果实在不可用，也可以进入其它分支流程。

    只需调用get()或 get(long timeout, TimeUnit unit)通过同步阻塞方式等待对象就绪。实际运行期是阻塞还是立即返回就取决于get()的调用时机和对象就绪的先后了。

- rxjava

- actor

- coroutine

跪求科普，等理解了再接着完善。

## 浅谈协程

说到协程，就要说线程。

线程是操作系统的用户态概念，线程本身也依赖中断来进行调度。早期的用户态IO并发处理是用poll(select)模型去轮询IO状态，然后发起相应的IO操作，称之为事件响应式的异步模型，这种方式并不容易使用，所以又发展出了阻塞式IO操作，让逻辑挂起并等待IO完成，为了让阻塞式IO能够并发就必须依赖多线程或者多进程模型来实现。但是线程的开销是非常大的，当遇到大规模并发的时候多线程模型就无法胜任了。所以大规模并发时我们又退回去使用事件响应，epoll在本质上还是poll模型，只是在算法上优化了实现，此时我们只用单线程就可以处理上万的并发请求了。

直到多核CPU的出现，我们发现只用一个线程是无法发挥多核CPU的威力的，所以再次引入线程池来分摊IO操作的CPU消耗，甚至CPU的中断响应也可以由多个核来分摊执行，此时的线程数量是大致等于CPU的核心数而远小于并发IO数的（这时CPU能处理百万级的并发），线程的引入完全是为了负载均衡而跟并发没有关系。所以不管是用select/epoll/iocp在逻辑层都绕不开基于事件响应的异步操作，面对异步逻辑本身的复杂性，我们才引入了async/await以及coroutine来降低复杂性。

coroutine是个很宽泛的概念，async/await也属于coroutine的一种。

而协程在实现模式上又分为：stackful coroutine和stackless coroutine。

所谓stackful是指每个coroutine有独立的运行栈，比如go语言的每个goroutine会分配一个4k的内存来做为运行栈，切换goroutine的时候运行栈也会切换。stackful的好处在于这种coroutine是完整的，coroutine可以嵌套、循环。

 与stackful对应的是stackless coroutine，比如js的generator函数，这类coroutine不需要分配单独的栈空间，coroutine状态保存在闭包里，但缺点是功能比较弱，不能被嵌套调用，也没办法和异步函数配合使用进行控制流的调度，所以基本上没办法跟stackful coroutine做比较。保存这些状态的时候，有的语言就引入了状态机的模型来实现线程。

async/await的出现，实现了基于stackless coroutine的完整coroutine。在特性上已经非常接近stackful coroutine了，不但可以嵌套使用也可以支持try catch。

## 结语

发现自己挺无脑的，学会了一个东西，就无脑的去用，直到碰壁了才会去思考。