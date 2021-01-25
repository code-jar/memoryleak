# ASP.NET Core 中的内存管理和模式

‎内存管理很复杂‎, 即使在像 .NET 这样的托管框架中. 分析和理解内存问题也很具挑战性.

最近 一个用户在 ASP.NET Core 主存储库中 [提交了一个问题](https://github.com/aspnet/Home/issues/1976) 指出垃圾回收器(GC) "未运行垃圾回收", 那它就失去了存在的意义. 症状如提交者描述那样, 内存在请求后不断增长, 这让他们认为问题出在 GC.

‎我们试图获得有关此问题的更多信息‎, 了解问题出在 GC 还是应用程序本身, 但我们得到的是贡献者提交的一系列类似行为报告: ‎内存不断增长‎. 有了一定的线索后,我们决定把它分成多个问题，并独立跟进.  最后,大多数问题都可以解释为对.NET中‎内存消耗的工作原理‎存在误解, ‎但也存在如何测量的问题‎.

为了帮助 .NET 开发人员更好地了解他们的应用程序，我们需要了解内存管理在 ASP.NET Core 中的工作方式、如何检测内存相关问题以及如何防止常见错误.

## 垃圾回收在 ASP.NET Core 中如何工作

GC按段分配，其中每个段是连续的内存范围. 放在其中的对象分为三代 0, 1, 2. 代决定了GC 尝试在应用程序不再引用的托管对象上释放内存的频率 - 数字越小频率越高.

对象根据其生存期从一代移动到另一代. 随着对象存在周期的延长，它们会被移动到更高的代中, 并减少回收检查次数. 生存期较短的对象 (如Web请求生命周期期间引用的对象)将始终保留在第 0 代中. 而应用程序级别的单例对象很可能移动到第1代，并最终移动到第2代.

当 ASP.NET Core 应用启动时, GC将为初始堆段保留一些内存, 并在加载运行时提交其中的一小部分. 这样做是出于性能原因,因此堆段可以位于连续内存中.

> 重要: ASP.NET Core 进程在启动时将会预先分配大量内存.

### 显式调用GC

‎手动调用GC执行‎ `GC.Collect()`. 将触发第2代和所有较低代回收. ‎这通常仅在调查内存泄漏时使用‎, 确保在测量前GC移除内存中所有悬空对象.

> 注意: 应用程序不应直接调用 `GC.Collect()`.

## ‎分析应用程序的内存使用情况‎

‎专用工具可帮助分析内存使用情况‎:
- 对象引用数量
- ‎测量 GC 对 CPU 的影响‎
- ‎测量每一代使用的空间‎

然而为了简单起见，本文不会使用这些，而是呈现一些应用内实时图表.

要深入分析,请阅读这些文章 其中演示如何使用 Visual Studio .NET:

[不使用调试器情况下的内存使用情况](https://docs.microsoft.com/en-us/visualstudio/profiling/memory-usage-without-debugging2)

[在 Visual Studio 中衡量内存使用情况](https://docs.microsoft.com/en-us/visualstudio/profiling/memory-usage)


### ‎检测内存问题‎

大多数时候，__任务管理__ 中显示的内存‎‎度‎‎量值用于了解ASP.NET应用程序内存量. 此值表示计算机进程使用的内存量, ASP.NET应用程序的生存对象和其他内存使用者，如本机内存使用情况.
此值表示ASP.NET的进程的内存使用量， 其中包括应用程序的活动对象和其他内存使用者（如本机内存）

看到此值无限增加是代码中某处存在内存泄漏的线索，但它无法解释它是什么. 下一节将向您介绍特定的内存使用模式并对其进行解释.

### ‎运行应用程序‎

完整的源代码在 GitHub 上提供 https://github.com/sebastienros/memoryleak

一旦应用程序启动,应用程序显示一些内存和GC统计信息,页面每隔一秒钟刷新一次. 特定的API接口执行特定的内存分配模式. 

‎测试此应用程序‎, ‎只需启动它‎. ‎您可以看到分配的内存不断增加‎, 因为显示这些统计信息就是在分配自定义对象. ‎GC 最终运行并收集它们‎.

此页显示一个包含分配内存和GC集合的图. ‎图例还显示 CPU 使用率和吞吐量（以请求数/秒表示）‎.

‎图表显示内存使用情况的两个值‎:
- Allocated(分配): 托管对象占用的内存量‎
- Working Set(‎工作集‎): 进程使用的总物理内存(RAM) (如任务管理器中显示的)

#### 瞬态对象‎

以下 API 创建一个 10KB  `String` 实例并返回到客户端‎. 每个请求在内存中分配一个新对象，并在响应上写入. 

> 注意: .NET中字符串以UTF-16编码存储,因此每个字符在内存中需要两个字节‎.

```csharp
[HttpGet("bigstring")]
public ActionResult<string> GetBigString()
{
    return new String('x', 10 * 1024);
}
```

下图以相对较小的5K RPS负载生成，以便了解内存分配如何受到GC的影响.

![](images/bigstring.png)

‎在此示例中‎, ‎当分配达到略高于300MB 的阈值时，GC大约每两秒钟收集一次0代实例‎. ‎工作集稳定在 500 MB 左右‎, CPU使用率低.

‎此图显示的是，在相对较低的请求吞吐量时，内存消耗非常稳定，达到 GC 选择的量‎.

‎一旦负载增加到机器可以处理的最大吞吐量，将绘制以下图表‎.

![](images/bigstring2.png)

‎有一些值得注意的点‎:
- ‎回收发生的频率要大得多‎, 每秒多次
- 现在有第一代回收, 这是因为我们在同一时间内分配了更多的资源
- ‎工作集仍然稳定‎

‎我们看到的是，只要CPU没有被过度利用‎, ‎垃圾回收可以处理大量的分配‎.

#### Workstation GC vs. Server GC

.NET 垃圾收集器可以在两种不同的模式下工作‎, 分别为 __Workstation GC__ 和 __Server GC__. 正如名字所述, 它们针对不同的工作负载进行了优化. ASP.NET 应用默认使用Server GC 模式, 而桌面应用使用 Workstation GC 模式.

区分两种模式的影响, 我们可以通过修改项目文件(`.csproj`)中`ServerGarbageCollection`参数,强制Web应用使用 Workstation GC. ‎这需要重新生成应用程序‎.

```xml
    <ServerGarbageCollection>false</ServerGarbageCollection>
```

‎也可以通过在已发布的应用程序的文件 `runtimeconfig.json` 设置‎ `System.GC.Server` 属性来完成.

以下是5K RPS使用Workstation GC下的内存使用情况.

![](images/workstation.png)

差异是巨大的:
- ‎工作集从 500MB 到 70MB‎
- GC每秒执行多次0代回收，而不是每两秒执行一次
- ‎GC 阈值从 300MB 到 10MB‎

‎在典型的 Web 服务器环境中，CPU资源比内存更重要‎, 因此使用Server GC更合适. 然而, 某些服务器可能更适合使用Workstation GC, 例如当一个服务器托管了多个Web应用程序时，内存资源更加宝贵. 

> 注意: 在单核心机器上，GC的模式总是 Workstation.

#### 持久的引用

‎即使垃圾回收器在防止内存增长方面做得很好‎, ‎如果对象由用户代码持续持有,‎ GC就没法释放它. ‎如果此类对象使用的内存量不断增加‎, 这叫做托管内存泄漏.

以下 API 创建一个 10KB  `String` 实例并返回到客户端. 不同于第一个例子的是，此实例由静态成员引用, 这意味着它不会被回收.

```csharp
private static ConcurrentBag<string> _staticStrings = new ConcurrentBag<string>();

[HttpGet("staticstring")]
public ActionResult<string> GetStaticString()
{
    var bigString = new String('x', 10 * 1024);
    _staticStrings.Add(bigString);
    return bigString;
}
```

这是一个典型的用户代码内存泄漏，内存将持续增加直到引发`OutOfMemory`异常导致进程崩溃.

![](images/eternal.png)

通过此图表上可以看到，一旦开始在这个终结点上发起请求工作集不再稳定，且不断增加. 在此期间，随着内存增加GC会尝试调用第2代垃圾回收释放内存, ‎这成功并释放了一些‎, ‎但这并没有阻止工作集增长.

‎某些方案需要无限期地保留对象引用‎, 在这种情况下，缓解此问题的一种方法是使用`WeakReference`类，以便在内存压力下仍可以回收对象上保留引用. 这是在ASP.NET Core中 `IMemoryCache` 的默认实现. 

#### 本机内存

内存泄漏不一定是由对托管对象的持久引用造成的. 有些.NET对象依赖本机内存来运行. GC无法收集此内存，.NET对象需要使用本机代码释放它.

幸运的是 .NET 提供了 `IDisposable` 接口让开发人员主动释放本机内存‎. ‎即使‎ `Dispose()` ‎未及时调用‎, 类通常在终结器运行时自动执行... ‎除非类未正确实现‎.

‎让我们看一下这个代码‎:

```csharp
[HttpGet("fileprovider")]
public void GetFileProvider()
{
    var fp = new PhysicalFileProvider(TempPath);
    fp.Watch("*.*");
}
```

`PhysicaFileProvider` ‎是托管类‎, 因此所有实例将会在请求结束后回收.

‎下面是连续调用此 API 时生成的内存分析.

![](images/fileprovider.png)

这个图表显示了这个类实现的一个明显问题, 它不断增加内存使用量. ‎这是一个已知问题，正在这里跟踪‎ https://github.com/aspnet/Home/issues/3110

‎同样的问题很容易在用户代码中发生‎, ‎不正确地释放类或忘记调用‎需要释放对象的 `Dispose()` 方法. 

#### ‎大型对象堆‎

‎随着内存的连续分配和释放‎, ‎内存中可能发生碎片‎. ‎这是因为对象必须分配在连续的内存块中所导致. 为了‎缓解此问题‎, ‎每当垃圾回收器释放一些内存‎, 将尝试进行碎片整理. 这个过程叫做 __压缩__.

压缩面临的问题是，对象越大‎, ‎移动速度越慢‎. 当到达一定大小后，‎移动它所花费的时间使移动它不再那么有效‎. 因此，GC 为这些大型对象创建一个特殊的‎‎内存‎‎区域‎, 成为 __大型对象堆__ (LOH). 大于 85,000 bytes (非 85 KB)的对象‎被放置在那里‎, 不压缩, 而且仅在2代回收时释放. 但是当LOH满的时候, 将会自动触发2代垃圾回收, 这本质上是较慢的， 因为它触发了所有其他代的回收.

‎下面是一个 API，它说明了此行为‎:

```csharp
[HttpGet("loh/{size=85000}")]
public int GetLOH1(int size)
{
    return new byte[size].Length;
}
```

下图显示了‎在最大负载下‎，调用使用`84,975`字节数组终结点的内存分析 

![](images/loh1.png)

当调用同一个终结点，但只多了一个字节时, i.e. `84,976` bytes (byte[]结构在实际字节序列化的基础上有一些开销).

![](images/loh2.png)

‎在这两种情况下，工作集大致相同‎, 稳定 450 MB. 但需要我们注意的是,并非回收了第0代, 我们回收了第2代, 这需要更多的CPU时间，直接影响吞吐量 从 35K 到 18K RPS, __‎几乎减半‎__.

‎这表明应避免非常大的对象‎. 例如ASP.NET Core __Response Caching__ 中间件,将缓存项拆分为小于85,000字节的块以处理此情况.

下面是处理此行为的特定实现的一些链接‎ 
- https://github.com/aspnet/ResponseCaching/blob/c1cb7576a0b86e32aec990c22df29c780af29ca5/src/Microsoft.AspNetCore.ResponseCaching/Streams/StreamUtilities.cs#L16
- https://github.com/aspnet/ResponseCaching/blob/c1cb7576a0b86e32aec990c22df29c780af29ca5/src/Microsoft.AspNetCore.ResponseCaching/Internal/MemoryResponseCache.cs#L55

#### HttpClient

‎不是具体到内存泄漏问题，更多的是资源泄漏问题‎, 但这在用户代码中已经出现了很多次，值得在这里提及.

有经验的 .NET 开发者实现 `IDisposable`接口释放对象或其他本机资源，如数据库连接和文件处理程序, ‎不这样做可能会导致内存泄漏‎ (参见前面的示例).

`HttpClient`例外, ‎即使它实现‎ `IDisposable`, 应该重用它，而不是在每次使用后释放.

这是一个API终结点，它在每次请求中都创建新的实例而后释放.

```csharp
[HttpGet("httpclient1")]
public async Task<int> GetHttpClient1(string url)
{
    using (var httpClient = new HttpClient())
    {
        var result = await httpClient.GetAsync(url);
        return (int)result.StatusCode;
    }
}
```

当给终结点施加负载后, 一些异常就会被记录下来:

```
fail: Microsoft.AspNetCore.Server.Kestrel[13]
      Connection id "0HLG70PBE1CR1", Request id "0HLG70PBE1CR1:00000031": An unhandled exception was thrown by the application.
System.Net.Http.HttpRequestException: Only one usage of each socket address (protocol/network address/port) is normally permitted ---> System.Net.Sockets.SocketException: Only one usage of each socket address (protocol/network address/port) is normally permitted
   at System.Net.Http.ConnectHelper.ConnectAsync(String host, Int32 port, CancellationToken cancellationToken)
```

当 `HttpClient` 实例被释放, ‎实际网络连接需要一些时间才能由操作系统释放‎. 每个客户端连接都需要自己的客户端端口,通过不断创建新连接，‎‎可用端口最终被耗尽.

解决方式是像这样重用同一个 `HttpClient` 实例:

```csharp
private static readonly HttpClient _httpClient = new HttpClient();

[HttpGet("httpclient2")]
public async Task<int> GetHttpClient2(string url)
{
    var result = await _httpClient.GetAsync(url);
    return (int)result.StatusCode;
}
```

‎当应用程序停止时，此实例最终将被释放‎.

这表明，可释放的资源也不意味着需要立即释放

> ‎注意‎: 从ASP.NET Core 2.1开始有个更好的方式处理 `HttpClient`实例的生命周期  https://blogs.msdn.microsoft.com/webdev/2018/02/28/asp-net-core-2-1-preview1-introducing-httpclient-factory/

#### 对象池

在上一个例子中我们看到 我们看到了如何使`HttpClient`实例静态使用，并由所有请求重用，以防止资源耗尽

类似的模式是使用对象池. 这个想法是,如果一个对象的创建是昂贵的, 我们应该重用它的实例来防止资源分配. 对象池是可跨线程保留和释放的预初始化对象的集合. 对象池可以定义硬限制之类的分配规则, 预定义大小, ‎或增长率‎.

Nuget 包 `Microsoft.Extensions.ObjectPool` ‎包含有助于管理此类池的类‎.

‎展示它是多么有效, 让我们使用一个API终结点来实例化一个`byte`缓冲区, 该缓冲区在每个请求中填充随机数:

```csharp
        [HttpGet("array/{size}")]
        public byte[] GetArray(int size)
        {
            var random = new Random();
            var array = new byte[size];
            random.NextBytes(array);

            return array;
        }
```

在一些负载下,我们看到第0代回收每秒都在进行.

![](images/array.png)

优化这些代码我们,可以使用`ArrayPool<>`,将字节数组放入对象池中. ‎静态实例在请求之间重复使用‎. 

此方案的特殊部分是，我们从 API 返回一个池对象, 这意味着只要我们从方法返回，就失去了对它的控制, 且无法释放它. 为了解决这个问题,我们需要将数组池封装在可释放对象中, 然后将此对象注册到 `HttpContext.Response.RegisterForDispose()`. ‎此方法将负责对目标对象调用‎ `Dispose()`, 所以它只有在HTTP请求完成时才被释放.

```csharp
private static ArrayPool<byte> _arrayPool = ArrayPool<byte>.Create();

private class PooledArray : IDisposable
{
    public byte[] Array { get; private set; }

    public PooledArray(int size)
    {
        Array = _arrayPool.Rent(size);
    }

    public void Dispose()
    {
        _arrayPool.Return(Array);
    }
}

[HttpGet("pooledarray/{size}")]
public byte[] GetPooledArray(int size)
{
    var pooledArray = new PooledArray(size);

    var random = new Random();
    random.NextBytes(pooledArray.Array);

    HttpContext.Response.RegisterForDispose(pooledArray);

    return pooledArray.Array;
}
```

以下是使用与非应用池版本相同负载的请求图表:

![](images/pooledarray.png)

‎您可以看到主要差异是分配的字节‎, 并且第0代的回收也更少.

## ‎结论‎

理解垃圾回收如何与ASP.NET Core协同工作,‎有助于调查内存压力问题‎,最终影响应用程序的性能. 

应用本文中解释的实践应该可以防止应用程序出现内存泄漏的迹象.

### ‎参考文章‎

‎进一步了解内存管理在 .NET 中的工作原理‎, ‎这里有一些推荐的文章‎.

[垃圾回收](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/)

[使用并发可视化工具了解不同的GC模式](https://blogs.msdn.microsoft.com/seteplia/2017/01/05/understanding-different-gc-modes-with-concurrency-visualizer/)
