# Java8 CompletableFuture使用

## 什么是CompletableFuture

Java8新增了CompletableFuture这个类用于异步编程，其实现了Future API和CompletableStage，拥有比Future更为强大的功能，可以实现多个Future的链式调用和组合调用。支持Future的回调处理，支持异常处理。

### Future的缺陷

1. 没有回调函数，无法让其在执行完成后自动调用回调处理逻辑
2. 不能灵活的组合多个Future并行执行的结果，需要配合其他同步器如CountDownLatch
3. Future没有异常处理机制

以上缺陷CompletableFuture都能弥补，因此本篇文章简单了解下CompletableFuture是怎么使用的。



## 使用CompletableFuture

### 使用runAsync()运行无返回值的异步任务

如果只是想异步地运行一个任务，并且没有返回值，使用CompletableFuture方法如下：

```java
// 第一个参数是Runnable，这里直接使用lambda表达式
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("正在刷牙");
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    System.out.println("刷牙完成");
});
// 阻塞等待异步任务完成
future.get();
```

在main函数中运行后输出如下结果：

```
Main thread is [main]
Current thread is [ForkJoinPool.commonPool-worker-1]
正在刷牙
刷牙完成
```

上述代码的runAsync方法没有指定线程池，会默认使用系统的ForkJoinPool。不推荐这样做，最好是根据业务类型定义不同的线程池来执行。

想要使用指定的线程池执行异步任务，只需在调用runAsync方法时加入第二个线程池参数executor即可

```java
// 自定义线程池
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("正在刷牙");
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    System.out.println("刷牙完成");
}, executor);
// 阻塞等待异步任务完成
future.get();
```



### 使用supplyAsync()运行有返回值的异步任务

supplyAsync方法支持返回封装结果的CompletableFuture，使用方式如下：

```java
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    System.out.println("Current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("正在刷牙");
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "刷牙完成";
}, executor);
// 阻塞等待异步任务完成，获取结果
System.out.println(future.get());
```

输出如下：

```
Main thread is [main]
Current thread is [pool-1-thread-1]
正在刷牙
刷牙完成
```



### 链式调用

上面的两种使用其实Future也能够做到，在调用CompletableFuture.get()时都是阻塞的。重点在于CompletableFuture提供了异步任务完成后的链式调用操作。

回调方法主要包括thenApply()，thenAccept() 和thenRun()：

#### thenApply()

thenApply()接收上一步异步任务的结果，处理后返回结果

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
```

比如说刷牙完成后马上洗脸，示例如下：

```java
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    System.out.println("刷牙 thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("正在刷牙");
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "刷牙完成";
}, executor).thenApply(result -> {
    System.out.println("洗脸 thread is [" + Thread.currentThread().getName() + "]");
    System.out.println(result + ", 正在洗脸");
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "洗脸完成";
}).thenApply(result -> {
    System.out.println("梳头 thread is [" + Thread.currentThread().getName() + "]");
    System.out.println(result + ", 正在梳头");
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "梳头完成";
});
// 阻塞直到获取最终结果
System.out.println(future.get());
```

输出如下：

```
Main thread is [main]
刷牙 thread is [pool-1-thread-1]
正在刷牙
洗脸 thread is [pool-1-thread-1]
刷牙完成, 正在洗脸
梳头 thread is [pool-1-thread-1]
洗脸完成, 正在梳头
梳头完成
```

可以看到刷牙和洗脸梳头都是在同一个线程执行，等待刷牙完成后，马上执行洗脸的操作。

thenApply()方法接收上一步异步处理的结果，并最终也会返回一个CompletableFuture结果，实现异步的链式调用



#### thenAccpt()和thenRun()

thenAccpt()接收上一步异步任务的结果，处理后没有返回结果。

```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action)
```

thenRun()没有接收上一步异步任务结果也没有处理后的返回结果

```java
public CompletableFuture<Void> thenRun(Runnable action)
```

这两个方法一般都用于异步调用链的尾端，使用示例：

```java
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("正在刷牙");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "刷牙完成";
}, executor).thenApply(result -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println(result + ", 正在洗脸");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "洗脸完成";
}).thenAccept(result -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println(result + ", 正在梳头");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    System.out.println("梳头完成");
}).thenRun(() -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("完成所有工作");
});
// 阻塞等待执行完成
future.get();
```

输出：

```
Main thread is [main]
current thread is [pool-1-thread-1]
正在刷牙
current thread is [pool-1-thread-1]
刷牙完成, 正在洗脸
current thread is [pool-1-thread-1]
洗脸完成, 正在梳头
梳头完成
current thread is [pool-1-thread-1]
完成所有工作
```

可见thenApply、thenAccept、thenRun的执行都是与上步异步任务的线程相同。

如果想要使用其他线程来执行thenApply、thenAccept、thenRun，CompletableFuture也提供了方法如下：

```java
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn);
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor);
    
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action);
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action,Executor executor);
    
public CompletableFuture<Void> thenRunAsync(Runnable action);
public CompletableFuture<Void> thenRunAsync(Runnable action,Executor executor);
```

使用示例：

```java
Executor executor = Executors.newFixedThreadPool(10);
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("正在刷牙");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "刷牙完成";
}, executor).thenApplyAsync(result -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println(result + ", 正在洗脸");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "洗脸完成";
}, executor).thenAcceptAsync(result -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println(result + ", 正在梳头");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    System.out.println("梳头完成");
}, executor).thenRunAsync(() -> {
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    System.out.println("完成所有工作");
}, executor);
// 阻塞等待执行完成
future.get();
```

输出每个异步操作的线程都不一样：

```
Main thread is [main]
current thread is [pool-1-thread-1]
正在刷牙
current thread is [pool-1-thread-2]
刷牙完成, 正在洗脸
current thread is [pool-1-thread-3]
洗脸完成, 正在梳头
梳头完成
current thread is [pool-1-thread-4]
完成所有工作
```



### 组合调用

#### thenCombine()

```java
public <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
```

使用thenCombine()方法可以等待两个异步任务执行完成后根据二者返回结果处理一些操作：

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    System.out.println("获取A设备信息");
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "Device data of A";
});
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    System.out.println("获取B设备信息");
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "Device data of B";
});
CompletableFuture<String> getDevDataFuture = future1.thenCombine(future2, (dataOfA, dataOfB) -> {
    System.out.println("根据A设备和B设备信息保存相关数据");
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return dataOfA + ", " + dataOfB;
});
String result = getDevDataFuture.get();
System.out.println(result);
```

输出：

```
Main thread is [main]
获取A设备信息
current thread is [ForkJoinPool.commonPool-worker-1]
获取B设备信息
current thread is [ForkJoinPool.commonPool-worker-2]
根据A设备和B设备信息保存相关数据
current thread is [ForkJoinPool.commonPool-worker-2]
Device data of A, Device data of B
```

获取A设备信息和获取B设备信息的任务是同时执行的，thenCombine的执行线程是这两个异步任务执行线程中的最后执行完成的线程。

类似的，thenCombine也有对应的异步方法，可以另起线程执行，以及指定线程池：

```java
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)
```



#### 组合执行多个异步任务

##### allOf()

使用CompletableFuture.allOf()方法可以同时调用多个异步任务执行，当所有任务完成时CompletableFuture才算完成。

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
```

假设现在业务需要透传获取三个设备的信息并最后统一处理，那么可以如下处理：

```java
...定义获取设备A和设备B的future1和future2
CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
    System.out.println("获取C设备信息");
    System.out.println("current thread is [" + Thread.currentThread().getName() + "]");
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    }
    return "Device data of C";
});
List<CompletableFuture<String>> getDevFutures = new ArrayList<>();
getDevFutures.add(future1);
getDevFutures.add(future2);
getDevFutures.add(future3);
// 由于allOff返回的CompletableFuture封装结果是void，并不能获取到批量结果
// 所以它的作用就只有通知这些所有任务都已经完成了
CompletableFuture<Void> allFuture =
        CompletableFuture.allOf(future1, future2, future3);
CompletableFuture<List<String>> allDevFuture =
        allFuture.thenApply(v ->
                // 所有future执行完毕后获取结果并加入List中，最后返回所有结果的数组
                // join()方法功能和get()方法是一样，区别在于join()抛出的是 unchecked Exception
                // 这里是等待allFuture执行完毕后执行的，所以每个future都是执行完毕的，
                // 调用join()无需用户手动catch处理异常
                getDevFutures.stream().map(CompletableFuture::join).collect(Collectors.toList()));
List<String> getDevResult = allDevFuture.get();
getDevResult.forEach(System.out::println);
```

输出：

```java
Main thread is [main]
获取A设备信息
current thread is [ForkJoinPool.commonPool-worker-1]
获取B设备信息
current thread is [ForkJoinPool.commonPool-worker-2]
获取C设备信息
current thread is [ForkJoinPool.commonPool-worker-3]
Device data of A
Device data of B
Device data of C
```



##### anyOf()

anyOf()方法就是只要有任意一个 CompletableFuture 执行完成就返回该任务的结果：

需要注意的是这几个future的返回封装的结果可以是不一样的，所以使用Object来接收。

```java
CompletableFuture cfA = CompletableFuture.supplyAsync(() -> 666);
CompletableFuture cfB = CompletableFuture.supplyAsync(() -> "result");
CompletableFuture cfC = CompletableFuture.supplyAsync(() -> new User());
 
CompletableFuture<Object> future = CompletableFuture.anyOf(cfA, cfB, cfC);
Object result = future.join();
```



### 异常处理

关于CompletableFuture的异常处理，主要是如下两个方法，灵活地实现异常的处理：

```java
public CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn);
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
```

#### exceptionally()

```java
CompletableFuture.supplyAsync(() -> {
     throw new RuntimeException();
}).thenApply(resultA -> resultA + " resultB")
  .thenApply(resultB -> resultB + " resultC")
  .thenApply(resultC -> resultC + " resultD");
```

在如上代码中，第一步出现了异常，那么后面的任务都不会执行，解决方案是可以使用exceptionally()方法对异常处理后，执行后续任务

```java
CompletableFuture.supplyAsync(() -> {
     throw new RuntimeException();
}).exceptionally(e-> "errorResultA")
  .thenApply(resultA -> resultA + " resultB")
  .thenApply(resultB -> resultB + " resultC")
  .thenApply(resultC -> resultC + " resultD");
```

#### handle()

handle方法中的参数result和e，如果e不为null说明上一步出现了异常

```java
int devId = -1;
CompletableFuture<String> checkDevIdFuture = CompletableFuture.supplyAsync(() -> {
    if (devId < 0) {
        throw new IllegalArgumentException("devId invalid");
    }
	return "device data";
}).handle((result, e) -> {
	if(ex != null) {
		System.out.println("occur exception: " + e.getMessage());
		return "error";
	}
	return res;
});
System.out.println(checkDevIdFuture.get());
```



## 总结

本文简单介绍了CompletableFuture的一些使用方法，还有较多方法还未提到，有兴趣的同事可以查阅相关资料了解和使用。

CompletableFuture使用场景的个人思考：

在TUMS系统，存在需要向多个设备透传获取信息的业务，例如RMS透传多个设备获取设备的客控模板配置内容，并同步到系统中，当前使用到异步透传任务+传入countDownLatch，每个异步透传线程获取到设备信息后将countDownLatch-1，主线程等待countDownLatch变为0后即所有信息都获取成功后执行后续配置同步到数据库的工作，这种类似的逻辑可以使用CompletableFuture的allOf方法代替，更为简单。类似的场景还有设备批量绑定等。

