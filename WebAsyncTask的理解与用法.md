## WebAsyncTask的作用

传统的Restful请求，都是阻塞的模型，即一个请求如果一直没有返回的话，则会导致阻塞。系统吞吐量会降低。

而WebAsyncTask实现了异步处理，则可以立即返回，通过异步回调线程去处理请求。释放当前请求资源，提高吞吐量。



异步调用变成。可以进行灵活的异步任务处理：

* 异步回调
* 超时处理
* 一场处理



### 看个正常操作的例子

``` java
@GetMapping("/completion")
public WebAsyncTask<String> asyncTaskCompletion() {
    // 打印处理线程名
    System.out.println(format("请求处理线程：%s", currentThread().getName()));

    // 模拟开启一个异步任务，超时时间为10s
    WebAsyncTask<String> asyncTask = new WebAsyncTask<>(10 * 1000L, () -> {
        System.out.println(format("异步工作线程：%s", currentThread().getName()));
        // 任务处理时间5s，不超时
        sleep(5 * 1000L);
        return asyncService.generateUUID();
    });

    // 任务执行完成时调用该方法
    asyncTask.onCompletion(() -> System.out.println("任务执行完成"));
    System.out.println("继续处理其他事情");
    return asyncTask;
}
```

### 抛出异常的例子

``` java
@GetMapping("/exception")
public WebAsyncTask<String> asyncTaskException() {
    // 打印处理线程名
    System.out.println(format("请求处理线程：%s", currentThread().getName()));

    // 模拟开启一个异步任务，超时时间为10s
    WebAsyncTask<String> asyncTask = new WebAsyncTask<>(10 * 1000L, () -> {
        System.out.println(format("异步工作线程：%s", currentThread().getName()));
        // 任务处理时间5s，不超时
        sleep(5 * 1000L);
        throw new Exception(ERROR_MESSAGE);
    });

    // 任务执行完成时调用该方法
    asyncTask.onCompletion(() -> System.out.println("任务执行完成"));
    asyncTask.onError(() -> {
        System.out.println("任务执行异常");
        return ERROR_MESSAGE;
    });

    System.out.println("继续处理其他事情");
    return asyncTask;
}
```

> 注意：WebAsyncTask.onError(Callable<?>) ：当异步任务抛出异常的时候，onError()方法即会被调用。

### 出现超时响应的例子



``` java
@GetMapping("/timeout")
public WebAsyncTask<String> asyncTaskTimeout() {
    // 打印处理线程名
    out.println(format("请求处理线程：%s", currentThread().getName()));

    // 模拟开启一个异步任务，超时时间为10s
    WebAsyncTask<String> asyncTask = new WebAsyncTask<>(10 * 1000L, () -> {
        out.println(format("异步工作线程：%s", currentThread().getName()));
        // 任务处理时间5s，不超时
        sleep(15 * 1000L);
        return TIME_MESSAGE;
    });

    // 任务执行完成时调用该方法
    asyncTask.onCompletion(() -> out.println("任务执行完成"));
    asyncTask.onTimeout(() -> {
        out.println("任务执行超时");
        return TIME_MESSAGE;
    });

    out.println("继续处理其他事情");
    return asyncTask;
}
```

> 注意：WebAsyncTask.onTimeout(Callable<?>) ：当异步任务发生超时的时候，onTimeout()方法即会被调用。

### 线程池任务

> 注意： 上面介绍的例子，默认是没有采用线程池管理的，以为这线程释放就会重新创建新的异步线程。开销会多一些。一般都会采用线程池进行管理，所以一般注册一个@Bean进行一下线程池的配置，然后在相应代码处引入该线程池即可。

``` java
@Configuration
public class TaskConfiguration {
    @Bean("taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(5);
        taskExecutor.setMaxPoolSize(10);
        taskExecutor.setQueueCapacity(10);
        taskExecutor.setThreadNamePrefix("asyncTask");
        return taskExecutor;
    }
}
```

``` java
@Autowired
@Qualifier("taskExecutor")
private ThreadPoolTaskExecutor executor;

@GetMapping("/threadPool")
public WebAsyncTask<String> asyncTaskThreadPool() {
    return new WebAsyncTask<>(10 * 1000L, executor,
            () -> {
                out.println(format("异步工作线程：%s", currentThread().getName()));
                return asyncService.generateUUID();
            });
}

```

