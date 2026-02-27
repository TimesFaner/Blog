---
title: JobSystem\UNITY
date: 2026-02-27 13:25:41
tags:
---
一、什么是Job System
Job System是Unity 2018版本引入的一种新的并行计算系统，用于在GPU上执行计算密集型任务。它提供了一种简单、高效的方式来在GPU上并行处理大量数据，从而 <span style="color: yellow;">提高游戏性能和响应速度</span>。Job System可以用于各种场景，包括物理模拟、粒子系统、渲染、AI等。是DOTS中重要一部分。

二、Job System的优势
1. 高效的并行计算：Job System可以在GPU上并行处理大量数据，从而提高计算效率。
2. 灵活的任务调度：Job System支持动态任务调度，可以根据需要灵活地分配和调度任务。
3. 简单易用：Job System提供了一套简单易用的API，使得开发者可以轻松地实现并行计算任务。
4. 支持多种数据类型：Job System支持多种数据类型，包括结构体、数组、矩阵等，可以满足各种计算需求。
5. 与Unity原生的work threads支持，可以集成共享。


三、Job System的原理
1. 创建work thread（s）
2. 获取Job需求的值
3. 在线程中进行计算
4. 同步结果

四、Job System的基础介绍
1. 创建Job：首先需要创建一个继承自IJob的类，该类定义了需要并行执行的任务。例如，创建一个计算向量加法的Job：
```csharp
public struct VectorAddJob : IJob
{
    public NativeArray<float> result;
    public NativeArray<float> a;
    public NativeArray<float> b;

    public void Execute()
    {
        for (int i = 0; i < result.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
    }
}
```
nativeArray是Job System提供的一种数据结构，用于在GPU上存储数据。它是一种托管值类型，为本机内存提供了一个相对安全的 C# 封装器，它包含一个指向非托管分配的指针。
借助nativeContainer，可以允许job与主线程共享数据，而非隔离的同步。创建时有以下枚举：

    1. Allocator.Temp - 具有最快的分配速度。此类型适用于寿命为一帧或更短的分配。不应该使用 Temp 将 NativeContainer 分配传递给Job。在从方法调用返回之前，需要调用 Dispose 方法。

    2. Allocator.TempJob - 的分配速度比 Temp 慢，但比 Persistent 快。此类型适用于寿命为四帧的分配，并具有线程安全性。如果没有在四帧内对其执行 Dispose 方法，控制台会输出警告。大多数逻辑量少的Job都使用这种类型。

    3. Allocator.Persistent - 是最慢的分配，但可以在您所需的任意时间内持续存在，如果有必要，可以在整个应用程序的生命周期内存在。
不过它不受gc，需要手动dispose。

2. 创建JobHandle：使用JobHandle来管理Job的执行。并且shedule进行调度。例如，创建一个JobHandle来执行上述的VectorAddJob：
```csharp
JobHandle handle = new VectorAddJob
{
    result = resultArray,
    a = aArray,
    b = bArray
}.Schedule();
```
注：使用JobHandle来管理多个Job的执行：可以使用JobHandle的DependsOn方法来管理多个Job的执行顺序。例如，创建两个JobHandle，并使用DependsOn方法来确保第一个Job完成后再执行第二个Job：
```csharp
JobHandle handle1 = new VectorAddJob
{
    result = resultArray,
    a = aArray,
    b = bArray
}.Schedule();

JobHandle handle2 = new VectorMultiplyJob
{
    result = resultArray,
    a = aArray,
    b = bArray
}.Schedule(handle1);
```


3. 等待Job完成：使用JobHandle的Complete方法来等待Job完成。例如，等待上述的VectorAddJob完成：
```csharp
handle.Complete();
```
4. 释放资源：使用NativeArray的Dispose方法来释放资源。例如，释放上述的NativeArray：
```csharp
resultArray.Dispose();
aArray.Dispose();
bArray.Dispose();
```

五、Job System进阶（IJobFor、IJobParallelFor、transform）
1. IJobFor的接口里定义的execute与IJob的execute不同，多了index参数，用于指定当前处理的元素索引。通过这个参数我们可以访问Job中的NativeContainer容器，对容器中的元素进行相对独立的操作,且其会自动迭代。
    ```csharp
    public interface IJobFor
    {
        void Execute(int index);
    }

    public interface IJob
    {
        void Execute();
    }
    ```
2. IJobFor的执行方式：
    ```csharp
    public struct SumJobFor : IJobFor
    {
        public NativeArray<float> result;
        public NativeArray<float> a;
        public NativeArray<float> b;

        public void Execute(int index)
        {
            result[index] = a[index] + b[index];
        }
    }
    ```
    ```csharp
    JobHandle handle = new SumJobFor
    {
        result = resultArray,
        a = aArray,
        b = bArray
    }
    /*schedule为IJobFor的单线程（主线程闲置时也会窃取到主线程）调度处理
    第一个参数是需要处理的元素数量，第二个参数是默认的JobHandle， 可替换用于指定Job的执行顺序。例如，创建一个JobHandle来执行上述的SumJobFor：*/
    handle.Schedule(resultArray.Length, default);
    
    /*scheduleParallel为IJobFor的并行处理
    第一个参数是需要处理的元素数量，第二个则是每个线程可分担的自定义最大容量，第三个参数是默认的JobHandle， 可替换用于指定Job的执行顺序。例如，创建一个JobHandle来执行上述的SumJobFor：   
    */
    handle.ScheduleParallel(resultArray.Length, 32,default); 
    ```
    
    而IJobParallelFor与IJobParallelForTransform则是对IJobFor的封装，用于处理更复杂的并行任务。
    前者直接schedule即可并行处理、后者可以对transform进行操作。
3. Burst：Burst是Unity提供的一个用于优化Job System的库，它可以在编译时将Job System的代码转换为高度优化的本机代码，从而提高性能。Burst需要使用C# 7.3或更高版本，并且需要安装Burst Compiler插件。
在jobsysytem中，使用Burst需要将Job标记为BurstCompile特性，例如：
```csharp
[BurstCompile]
public struct VectorAddJob : IJob
{
    public NativeArray<float> result;
    public NativeArray<float> a;
    public NativeArray<float> b;

    public void Execute()
    {
        for (int i = 0; i < result.Length; i++)
        {
            result[i] = a[i] + b[i];
        }
    }
}
```
这会大大提高速率。

六、总结
Job System是Unity提供的一个用于并行处理任务的系统，它可以提高游戏的性能和响应速度。Job System使用C# 7.3或更高版本，并且需要安装Burst Compiler插件。Job System使用NativeArray来存储数据，并使用JobHandle来管理Job的执行。Job System可以处理更复杂的并行任务，例如对transform进行操作。  

七、参考

https://blog.csdn.net/qq_42461824/article/details/113058329
https://docs.unity3d.com/Manual/JobSystemParallelForJobs.html
https://www.cnblogs.com/FlyingZiming/p/17241013.html
