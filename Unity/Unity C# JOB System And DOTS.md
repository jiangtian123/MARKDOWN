# Unity C# JOB System And DOTS

## 概述

设计目的：简单安全地使用多线程，随便就能写出高性能代码。

## Job System

简化多线程:job system通过创建jobs来实现多线程，而不是直接创建thread

job 概念：完成特定任务的一个小的工作单元。job接收参数并不操作数据，类似函数调用。job和job之间可以有依赖关系。

job system 管理一组worker threads，并且**保证一个logical CPU core 一个 worker thread**，避免上下文切换。

job system 将一个job 放在一个jobqueue里面，worker threads 从 job queue里面获取job然后执行

job依赖性： job system管理 job依赖关系，并保证执行时序的正确性

## C# Job System 的Safety System

- Race conditions

  写多线程时，很容易出现多个线程操作同一个数据的现象，导致一个输出结果不对，很难Debug。

- Safety system ：

  为了解决这个问题，每个job的数据都来自主线程数据的副本，而不是操作原数据。

## Native Container

NativeContainer实际上是native memory的一个wrapper，包含一个指向非托管内存的指针。

**不需要拷贝：**使用NativeContainer可以让一个job和main thread共享数据，而不用拷贝。（copy虽然能保证Safety System，但每个job的计算结果也是分开的）。

可使用C#类型定义：

| 数据结构           | 说明                              | 来源  |
| ------------------ | --------------------------------- | ----- |
| NativeArray        | 数组                              | Unity |
| NativeSlice        | 可以访问一个NativeArray的某一部分 | Unity |
| NativeList         | 一个可变长的NativeArray           | ECS   |
| NativeHashMap      | key value pairs                   | ECS   |
| NativeMultiHashMap | 一个key对应多个values             | ECS   |
| NativeQueue        | FIFO的queue                       | ECS   |

Safety System安全策略：   

　　Safety System**内置于**所有的NativeContainer，**会自动跟踪**NativeContainer的读写状态。

  注意：所有的safety checkes都只在Editor和PlayMode模式下生效：bounds checks、deallocation checks、race condition checks。

  还有一部分安全策略：

​     **DisposeSentinel**：自动检测memory leak并报错。依赖宏定义ENABLE_UNITY_COLLECTIONS_CHECKS。

​     **AtomicSafetyHandle**：用来转移NativeContainer的控制权。比如当2个jobs同时写一个NativeContainer，Safety System就会抛出一个error，并描述如何解决。异常会在产生冲突的job调度时抛出。依赖宏定义ENABLE_UNITY_COLLECTIONS_CHECKS。

​     这种情况下，可以使用job依赖，让其中一个job依赖另外一个job的完成。

**规则：Safety System允许多个job同时read同一块数据。**

**规则：Safety System不允许一个job正在writing数据时，调度激活另一个“拥有write权限”的job（不是不让同时write）。**

**规则：手动指定job对数据的只读：（默认是可读写，会影响性能）**

**NativeContainer Allocator分配器：**

- Allocator.Temp

  最快，维持1 frame，job不能用（只能在创建它的线程中使用），需要手动Dispose()，比如可以在native层的callback调用时使用。

- Allocator.TempJbo

  稍微慢一点，最多维持4 frames，thread-safe，如果4 frames内没有Dispose()，会有warning。大多数small jobs都会使用这个类型的分配器.

- Allocator.Persistent

  最慢，但是可持久存在，就是malloc的wrapper。Longer jobs使用这个类型，但在性能敏感的地方不应该使用。

## 创建Job

三要素：

- 创建一个struct实现接口IJob；
- 添加数据成员：要么是blittable类型， 要么是NativeContainer；
- 添加Execute()方法实现。

执行job时，job.Execute()方法会在一个cpu core上执行一次。

注意：job操作数据是基于拷贝的，除非是NativeContainer类型。那么，一个job访问main thread数据的唯一方式就是使用NativeContainer。

```c#
public struct TestJob : IJob
{
    public float a;
    public float b;
    public NativeArray<float> result;
    public void Execute()
    {
        result[0] = a + b;
    }
}
```

## 调度Job

三要素：

- 实例化job；
- 设置数据；
- 调用job.Schedule()方法。

调用Schedule方法会将job放到job queue里面等待执行。一旦开始schedule，就没法中断job了。（疑问：这个once scheduled，是job.Schedule方法，还是从job queue里面拿出来开始执行？）

```c#
private void TestScheduleJob()
{
    // Create a native array of a single float to store the result. This example waits  for the job to complete for illustration purposes
    NativeArray<float> result = new NativeArray<float>(1, Allocator.TempJob);

    // Set up the job data
    MyJob jobData = new MyJob();
    jobData.a = 10;
    jobData.b = 10;
    jobData.result = result;

    // Schedule the job
    JobHandle handle = jobData.Schedule();

    // Wait for the job to complete
    handle.Complete();

    // All copies of the NativeArray point to the same memory, you can access the  result in "your" copy of the NativeArray
    float aPlusB = result[0];

    // Free the memory allocated by the result array
    result.Dispose();
}
```

## JobHandle和Job依赖

设置job依赖关系：

```c#
JobHandle firstJobHandle = firstJob.Schedule();
secondJob.Schedule(firstJobHandle);
```

secondJob依赖firstJob的结果。

组合依赖项：

```C#
NativeArray<JobHandle> handles = new NativeArray<JobHandle>(numJobs, Allocator.TempJob);
// Populate `handles` with `JobHandles` from multiple scheduled jobs...
JobHandle jh = JobHandle.CombineDependencies(handles);
```

```c#
public struct MyJob : IJob
{
    public float a;
    public float b;
    public NativeArray<float> result;
    public void Execute()
    {
        result[0] = a + b;
    }
}
public struct AddOneJob : IJob
{
    public NativeArray<float> result;
    
    public void Execute()
    {
        result[0] = result[0] + 1;
    }
}


private void TestScheduleJob()
{
    NativeArray<float> result = new NativeArray<float>(1, Allocator.TempJob);
    MyJob jobData = new MyJob();
    jobData.a = 10;
    jobData.b = 10;
    jobData.result = result;
    JobHandle firstHandle = jobData.Schedule();
    AddOneJob incJobData = new AddOneJob();
    incJobData.result = result;
    JobHandle secondHandle = incJobData.Schedule(firstHandle);
    secondHandle.Complete();
    float aPlusB = result[0];
    result.Dispose();
}
```

# Dots

