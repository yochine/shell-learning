> **来自公众号：小姐姐味道**作者简介：小姐姐味道 \(xjjdog\)，一个不允许程序员走弯路的公众号。聚焦基础架构和Linux。十年架构，日百亿流量，与你探讨高并发世界，给你不一样的味道。

在对[类的命名这篇长文](https://mp.weixin.qq.com/s?__biz=MzA4MTc4NTUxNQ==&mid=2650524593&idx=1&sn=99d335849dc1fc92f65dc618106aa844&chksm=8780cc75b0f745637ee03a63d7ea640ad1d4638478e8af5d811e92e985b1509219957408ec27&token=1556116051&lang=zh_CN&scene=21#wechat_redirect)
中，我们提到了`Future`和`Promise`。Future相当于一个占位符，代表一个操作将来的结果。一般通过get可以直接阻塞得到结果，或者让它异步执行然后通过callback回调结果。但如果回调中嵌入了回调呢？如果层次很深，就是回调地狱。Java中的CompletableFuture其实就是Promise，用来解决回调地狱问题。Promise是为了让代码变得优美而存在的。有多优美？这么说吧，一旦你使用了CompletableFuture，就会爱不释手，就像初恋女友一样，天天想着她。

## 一系列静态方法

从它的源代码中，我们可以看到，CompletableFuture直接提供了几个便捷的静态方法入口。其中有`run`和`supply`两组。![](https://mmbiz.qpic.cn/mmbiz_png/cvQbJDZsKLpsgAkLzXYicDoxwahBdiaaHM0ZpSk3Zk1GHA3tib55XlXYOibBiatlVcDPYLFoKDlF9JdqB4icJoOyC5rw/640?wx_fmt=png)
run的参数是Runnable，而supply的参数是Supplier。前者没有返回值，而后者有，否则没有什么两样。这两组静态函数，都提供了传入自定义线程池的功能。如果你用的不是外置的线程池，那么它就会使用默认的ForkJoin线程池。默认的线程池，大小和用途你是控制不了的，所以还是建议自己传递一个。典型的代码，写起来是这个样子。

```
CompletableFuture<String> future = CompletableFuture.supplyAsync(()->{ 
  return "test";
});
String result = future.join();
```

拿到CompletableFuture后，你就可以做更多的花样。

## 这些花样有很多

我们说面说了，CompletableFuture的主要作用，就是让代码写起来好看。配合Java8之后的stream流，可以把整个计算过程抽象成一个流。前面任务的计算结果，可以直接作为后面任务的输入，就像是管道一样。

```
thenApply
thenApplyAsync
thenAccept
thenAcceptAsync
thenRun
thenRunAsync
thenCombine
thenCombineAsync
thenCompose
thenComposeAsync
```

比如，下面代码的执行结果是99，并不因为是异步就打乱代码执行的顺序了。  

```
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> 10)
                  .thenApplyAsync((e) -> {   
                      try {          
                          Thread.sleep(10000);            
                      } catch (InterruptedException ex) {         
                          ex.printStackTrace();               
                      }               
                        return e * 10;      
                      }).thenApplyAsync(e -> e - 1);
cf.join();
System.out.println(cf.get());
```

同样的，函数的作用还要看then后面的动词。

- apply 有入参和返回值，入参为前置任务的输出
- accept 有入参无返回值，会返回CompletableFuture
- run 没有入参也没有返回值，同样会返回CompletableFuture
- combine 形成一个复合的结构，连接两个CompletableFuture，并将它们的2个输出结果，作为combine的输入
- compose 将嵌套的CompletableFuture平铺开，用来串联两个CompletableFuture

## when和handle

上面的函数列表，其实还有很多。比如：

```
whenComplete
```

when的意思，就是任务完成时候的回调。比如我们上面的例子，打算在完成任务后，输出一个`done`。它也是属于只有入参没有出参的范畴，适合放在最后一步进行观测。

```
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> 10)                .thenApplyAsync((e) -> {                    try {                        Thread.sleep(1000);                    } catch (InterruptedException ex) {                        ex.printStackTrace();                    }                    return e * 10;                }).thenApplyAsync(e -> e - 1)                .whenComplete((r, e)->{                    System.out.println("done");                })                ;cf.join();System.out.println(cf.get());
```

handle和exceptionally的作用，和whenComplete是非常像的。

```
public CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn);

public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
```

CompletableFuture的任务是串联的，如果它的其中某一步骤发生了异常，会影响后续代码的运行的。exceptionally从名字就可以看出，是专门处理这种情况的。比如，我们强制某个步骤除以0，发生异常，捕获后返回-1，它将能够继续运行。

```
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> 10)              
                  .thenApplyAsync(e->e/0)               
                  .thenApplyAsync(e -> e - 1)            
                  .exceptionally(ex->{               
                   System.out.println(ex);             
                      return -1;     
                  });
cf.join();
System.out.println(cf.get());
```

handle更加高级一些，因为它除了一个异常参数，还有一个正常的入参。处理方法也都类似，不再赘述。当然，CompletableFuture的函数不仅仅这些，还有更多，根据函数名称很容易能够了解到它的作用。它还可以替换复杂的CountDownLatch，这要涉及到几个比较难搞的函数。

## 替代CountDownLatch

考虑下面一个场景。某一个业务接口，需要处理几百个请求，请求之后再把这些结果给汇总起来。如果顺序执行的话，假设每个接口耗时100ms，那么100个接口，耗时就需要10秒。假如我们并行去获取的话，那么效率就会提高。使用CountDownLatch可以解决。

```
ExecutorService executor = Executors.newFixedThreadPool(5);
CountDownLatch countDown = new CountDownLatch(requests.size());
for(Request request:requests){    
    executor.execute(()->{     
      try{       
        //some opts     
      }finally{   
        countDown.countDown();    
      }   
    });
}

countDown.await(200,TimeUnit.MILLISECONDS);
```

我们使用CompletableFuture来替换它。

```
ExecutorService executor = Executors.newFixedThreadPool(5);
List<CompletableFuture<Result>> futureList = requests   
              .stream()   
              .map(request->    
                  CompletableFuture.supplyAsync(e->{       
                  //some opts       
                },executor))
              .collect(Collectors.toList());

CompletableFuture<Void> allCF = CompletableFuture.allOf(futureList.toArray(new CompletableFuture[0]));

allCF.join();
```

我们这里用到了一个主要的函数，那就是allOf，用来把所有的CompletableFuture组合在一起；类似的还有anyOf，表示只运行其中一个。常用的，还有三个函数：

- thenAcceptBoth 处理两个任务的情况，有两个任务结果入参，无返回值
- thenCombine  处理两个任务的情况，有入参有返回值，最喜欢
- runAfterBoth 处理两个任务的情况，无入参，无返回值

## End

自从认识了CompletableFuture，我已经很少硬编码Future了。相对于各种回调的嵌套，CompletableFuture为我们提供了更直观、更优美的API。在“多个任务等待完成状态”这个应用场景，CompletableFuture已经成了我的首选。唯一的问题是，它的函数有点多，你需要熟悉一小段时间。另外，有一个小小的问题，个人觉得，这个类如果叫做`Promise`的话，就能够和JS的统一起来，算是锦上添花吧。

  