
## 概述

在代码中遇到了网络请求编程时，往往需要异步编程才能给用户带来良好的体验，不会导致程序完全阻塞。
其中的关键在于：
- 异步方法：再执行完成前就立刻返回调用方法，在调用方法执行过程中完成任务
- async/await 结构主要分为三个部分：
  1. 调用方法：该方法调用异步方法，然后在异步方法回调后继续执行后续程序
  2. 异步方法：该方法异步执行程序，在被调用后立即返回到调用方法
  3. await 表达式：用于异步等待，指出需要异步执行的任务，且需要等待其完成。一个方法可以包含多个await

## 什么是异步

启动程序时，系统会在内存中创建一个新的进程。进程是构成运行程序资源的集合。在进程内部，有称为线程的内核对象，它代表的是真正的执行程序。系统会在 Main 方法的第一行语句就开始线程的执行。

线程：
- 默认情况，一个进程只包含一个线程，从程序的开始到执行结束
- 线程可以派生自其它线程，所以一个进程可以包含不同状态的多个线程，来执行程序的不同部分
- 一个进程中的多个线程，将共享该进程的资源
- 系统为处理器执行所规划的单元是线程，而非进程

下面我们来看一个简单的例子
```C#
class Program
{
    // 创建计时器
    private static readonly Stopwatch Watch = new Stopwatch();

    private static void Main(string[] args)
    {
        // 启动计时器
        Watch.Start();

        const string url1 = "http://www.cnblogs.com/";
        const string url2 = "http://www.cnblogs.com/liqingwen/";

        // 两次调用 CountCharactersAsync 方法（异步下载某网站内容，并统计字符的个数）
        Task<int> t1 = CountCharactersAsync(1, url1);
        Task<int> t2 = CountCharactersAsync(2, url2);

        // 三次调用 ExtraOperation 方法（主要是通过拼接字符串达到耗时操作）
        for (var i = 0; i < 3; i++)
        {
            ExtraOperation(i + 1);
        }

        // 控制台输出
        Console.WriteLine($"{url1} 的字符个数：{t1.Result}");
        Console.WriteLine($"{url2} 的字符个数：{t2.Result}");

        Console.Read();
    }

    // 统计字符个数
    private static async Task<int> CountCharactersAsync(int id, string address)
    {
        var wc = new WebClient();
        Console.WriteLine($"开始调用 id = {id}：{Watch.ElapsedMilliseconds} ms");

        var result = await wc.DownloadStringTaskAsync(address);
        Console.WriteLine($"调用完成 id = {id}：{Watch.ElapsedMilliseconds} ms");

        return result.Length;
    }

    // 额外操作
    private static void ExtraOperation(int id)
    {
        // 这里是通过拼接字符串进行一些相对耗时的操作，如果对字符串拼接有性能要求的话应该使用 StringBuilder
        var s = "";

        for (var i = 0; i < 6000; i++)
        {
            s += i;
        }

        Console.WriteLine($"id = {id} 的 ExtraOperation 方法完成：{Watch.ElapsedMilliseconds} ms");
    }
}
```

同步情况下的调用顺序为：
![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/async和await_正常调用时序图.PNG)

异步情况下的调用顺序为：
![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/async和await_异步调用时序图.PNG)

从上面两种情况可以看出异步编程优势，现在我们来分析一下程序的步骤：

1. 从 Main 方法执行到 `CountCharactersAsync(1, url1)` 方法时，该方法会立即返回，然后才会调用它内部的方法开始下载内容。该方法返回的是一个 `Task<int>` 类型的占位符对象，表示计划进行的工作。这个占位符最终会返回 int 类型的值
2. 这样就可以不必等 `CountCharactersAsync(1, url1)` 方法执行完成就可以继续进行下一步操作。到执行 `CountCharactersAsync(2, url2)`  方法时，跟 步骤① 一样返回 `Task<int>` 对象
3. 然后，Main 方法继续执行三次 ExtraOperation 方法，同时两次 CountCharactersAsync 方法依然在持续工作
4. `t1.Result` 和 `t2.Result` 是指从 `CountCharactersAsync` 方法调用的 `Task<int>` 对象取结果，如果还没有结果的话，将阻塞，直有结果返回为止

## async/await 结构
async/await 结构主要分为三个部分：
  1. 调用方法：该方法调用异步方法，然后在异步方法回调后继续执行后续程序
  2. 异步方法：该方法异步执行程序，在被调用后立即返回到调用方法
  3. await 表达式：用于异步等待，指出需要异步执行的任务，且需要等待其完成。一个方法可以包含多个await

示例结构分析如下：
![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/async和await_结构.PNG)

## await 都做了些什么
为了了解程序 await 时，C#都做了些什么工作，我们就必须了解以下内容

### 方法的状态
首先，方法内所有的本地变量的值都会被记住，包括

- 方法的参数
- 在方法的作用域内定义的任何变量
- 任何其它变量，比如循环中使用到的计数变量
- 如果你的方法不是static的，则还要包括this变量。只有记住了this，当方法恢复执行(resume)时，才可以使用当前类的成员变量。

**上述的这些都会被存储在.NET垃圾回收堆里的一个对象中**。因此，当你使用await时，.NET就会创建这样一个对象，虽然它会占用一些资源，但在大多数情况下并不会导致性能问题。

C#还要记住在方法内部await执行到了哪里——可以通过使用一个数字来表示当前方法中执行到了哪一个await关键字。

具体如何使用await表达式？这其实没有限制，例如，await可以被用做一个大表达式的一部分，一个表达式也可能包含多个await.

`int myNum = await AlexsMethodAsync(await myTask, await StuffAsync());`

这样就对.NET运行时提出了额外的需求——当await一个表达式时，需要记住表达式剩余部分的状态。在上面的例子中，当程序执行await StuffAsync()时，await myTask的结果就需要被记录下来。.NET IL会将这类子表达式存储在栈上，因此当使用了await关键字时就需要把这个栈存储下来。

在这之上，当程序执行到第一个await时，当前方法会返回——除非方法是async void，否则这时就会返回一个Task，因此调用者可以通过某种方式等待任务完成。C# 还必须把操作该返回Task的方法存储下来，这样当我们的方法完成后，前面返回的Task才会变为完成的状态，这样程序才会向上返回一层，回到方法的异步链中去继续执行。我们会在第14章探讨这些额外的机制。

### 上下文（Context）
C#在使用await时会记录各种各样的上下文，目的是当要继续执行方法时能够恢复这个上下文，这样就尽可能地将await的处理过程变得透明。这些上下文中最重要的就是同步上下文（sychronization context），通过它的帮助可以在指定类型的线程上恢复方法的执行。

## 异步方法的结构

1. 关键字：方法头使用 async 修饰。

2. 要求：包含 N（N>0） 个 await 表达式（不存在 await 表达式的话 IDE 会发出警告），表示需要异步执行的任务。

3. 返回类型：只能返回 3 种类型（void、Task 和 Task<T>）。Task 和 Task<T> 标识返回的对象会在将来完成工作，表示调用方法和异步方法可以继续执行。

4. 参数：数量不限，但不能使用 out 和 ref 关键字。

5. 命名约定：方法后缀名应以 Async 结尾。

6. 其它：匿名方法和 Lambda 表达式也可以作为异步对象；async 是一个上下文关键字；关键字 async 必须在返回类型前。

> 本文参考自：
> 
> https://www.cnblogs.com/tuyile006/p/12605523.html
> https://www.cnblogs.com/tuyile006/p/12605523.html
