
## 什么是 Thread

当我们提及多线程的时候会想到 Thread 和 Threadpool，这都是异步操作，Threadpool 其实就是 Thread 的集合，具有很多优势，不过在任务多的时候全局队列会存在竞争而消耗资源。Thread 默认为前台线程，主程序必须等线程跑完才会关闭，而 Threadpool 相反。

总结：Threadpool 确实比 Thread 性能优，但是两者都没有很好的api区控制，如果线程执行无响应就只能等待结束，从而诞生了 Task 任务。

## 什么是 Task

Task 简单地看就是任务，那和 Thread 有什么区别呢？Task 的背后的实现也是使用了线程池线程，但它的性能优于ThreadPoll,因为它使用的不是线程池的全局队列，而是使用的本地队列，使线程之间的资源竞争减少。同时 Task 提供了丰富的API来管理线程、控制。但是相对前面的两种耗内存，Task 依赖于CPU对于多核的CPU性能远超前两者，单核的CPU三者的性能没什么差别。Task 有Wait、ContinueWith、Cancel等操作，有返回值。

![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/task和thread_什么是task.PNG)

## Task 与 Thread 的区别
`Thread` 类主要用于实现线程的创建以及执行。

`Task` 类表示以异步方式执行的单个操作。

### 1. `Task` 是基于 `Thread` 的，是比较高层级的封装，`Task` 最终还是需要 `Thread` 来执行

### 2. `Task` 默认使用后台线程执行，`Thread` 默认使用前台线程
```C#
public static void Main(string[] args)
{
    Thread thread = new Thread(obj => { Thread.Sleep(3000); });
    thread.Start();
}
```
上面代码，主程序在3秒后结束。
```C#
public static void Main(string[] args)
{
    Task<int> task = new Task<int>(() => 
    {
        Thread,Sleep(3000);
        return 1;
    });
    task.Start();
}
```
而这段代码，会瞬间结束。

### 3. `Task` 可以有返回值，`Thread` 没有返回值
虽然 `Thread` 可以通过 Start 方法参数来进行返回值处理，但十分不便。
```C#
public static void Main(string[] args)
{  
    Task task = new Task(LongRunningTask);
    task.Start();
    Console.WriteLine(task.Result);
}
private static int LongRunningTask()
{
    Thread.Sleep(3000);
    return 1;
}
```

### 4. `Task` 可以执行后续操作，`Thread` 不能执行后续操作
```C#
static void Main(string[] args)
{
    Task task = new Task(LongRunningTask);
    task.Start();
    Task childTask = task.ContinueWith(SquareOfNumber);
    Console.WriteLine("Sqaure of number is :"+ childTask.Result);
    Console.WriteLine("The number is :" + task.Result);
}
private static int LongRunningTask()
{
    Thread.Sleep(3000);
    return 2;
}
private static int SquareOfNumber(Task obj)
{
    return obj.Result * obj.Result;
}
```

### 5. `Task` 可取消任务执行，`Thread` 不行
```C#
static void Main(string[] args)
{
    using (var cts = new CancellationTokenSource())
    {
        Task task = new Task(() => { LongRunningTask(cts.Token); });
        task.Start();
        Console.WriteLine("Operation Performing...");
        if(Console.ReadKey().Key == ConsoleKey.C)
        {
            Console.WriteLine("Cancelling..");
            cts.Cancel();
        }                
        Console.Read();
    }
}
private static void LongRunningTask(CancellationToken token)
{
    for (int i = 0; i < 10000000; i++)
    {
        if(token.IsCancellationRequested)
        {
            break;
        }
        else
        {                  
            Console.WriteLine(i);
        }               
    }          
}
```

### 6. 异常传播

`Thread` 在父方法上获取不到异常，而 `Task` 可以
