---
layout:     post
title:      CompletableFuture 异步调用工具
subtitle:   主要替换系统默认线程池
date:       2021-05-12
author:     zj
header-img: img/post-bg-YesOrNo.jpg
catalog: true
tags:
    - 工具整理
---

​    调用CompletableFuture 方法使用默认线程池,该线程池线程数目与操作系统核心数有关，很多业务场景导致我们不能使用默认线程池进行异步操作，例如当操作系统为单核心时，使用默认线程池调用CompletableFuture的方法，由于线程数目为1 ，所以代码执行效率并没有提高，反而有可能导致线程竞争造成卡顿，因此我们有必要自定义CompletableFuture 异步执行框架的线程池，进行异步调用线程的统一管理。

###### 系统默认线程构造代码

```java
// 该方法在jdk 类java.util.concurrent.ForkJoinPool 中
private static ForkJoinPool makeCommonPool() {
        int parallelism = -1;
        ForkJoinWorkerThreadFactory factory = null;
        UncaughtExceptionHandler handler = null;
        try {  // ignore exceptions in accessing/parsing properties
            String pp = System.getProperty
                ("java.util.concurrent.ForkJoinPool.common.parallelism");
            String fp = System.getProperty
                ("java.util.concurrent.ForkJoinPool.common.threadFactory");
            String hp = System.getProperty
                ("java.util.concurrent.ForkJoinPool.common.exceptionHandler");
            if (pp != null)
                parallelism = Integer.parseInt(pp);
            if (fp != null)
                factory = ((ForkJoinWorkerThreadFactory)ClassLoader.
                           getSystemClassLoader().loadClass(fp).newInstance());
            if (hp != null)
                handler = ((UncaughtExceptionHandler)ClassLoader.
                           getSystemClassLoader().loadClass(hp).newInstance());
        } catch (Exception ignore) {
        }
        if (factory == null) {
            if (System.getSecurityManager() == null)
                factory = defaultForkJoinWorkerThreadFactory;
            else // use security-managed default
                factory = new InnocuousForkJoinWorkerThreadFactory();
        }
        if (parallelism < 0 && // default 1 less than #cores
            (parallelism = Runtime.getRuntime().availableProcessors() - 1) <= 0)
            parallelism = 1;
        if (parallelism > MAX_CAP)
            parallelism = MAX_CAP;
        return new ForkJoinPool(parallelism, factory, handler, LIFO_QUEUE,
                                "ForkJoinPool.commonPool-worker-");
    }
```

###### 自定义异步操作工具代码

```java
package com.jd.ins.gravity.common.util;

import com.jd.ins.gravity.common.exception.BizException;
import org.apache.commons.lang3.concurrent.BasicThreadFactory;

import java.util.Objects;
import java.util.concurrent.*;
import java.util.function.Supplier;

/**
 * 异步执行器
 *
 * @author zj
 * @date 2021/5/11 16:24
 */
public class AsyncExecutor {

    private AsyncExecutor() {
    }

    /**
     * 默认线程池
     */
    public static final ThreadPoolExecutor COMMON_POOL =
            new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors() * 2, 1024,
                    15L, TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    new BasicThreadFactory.Builder()
                            .namingPattern("asyncExecutor-common-pool-%d")
                            .daemon(false).build());

    /**
     * 异步执行
     */
    public static CompletableFuture<Void> run(Runnable runnable) {
        return run(runnable, COMMON_POOL);
    }

    /**
     * 异步执行
     */
    public static CompletableFuture<Void> run(Runnable runnable, ExecutorService executorService) {
        return CompletableFuture.runAsync(runnable, executorService);
    }


    /**
     * 异步执行带有返回值
     *
     * @param supplier 获取
     * @param <U>      响应值类型
     */
    public static <U> U runApply(Supplier<U> supplier) throws BizException {
        return runApply(supplier, COMMON_POOL);
    }

    /**
     * 异步执行带有返回值
     *
     * @param supplier 获取
     * @param <U>      响应值类型
     */
    public static <U> U runApply(Supplier<U> supplier, ExecutorService executorService) throws BizException {
        CompletableFuture<U> future = CompletableFuture.supplyAsync(supplier, executorService);
        try {
            return future.get(15L, TimeUnit.SECONDS);
        } catch (InterruptedException | ExecutionException | TimeoutException e) {
            throw new BizException(e);
        }
    }

    /**
     * 异步执行没有执获取执行结果
     *
     * @param supplier 获取
     * @param <U>      响应值类型
     */
    public static <U> CompletableFuture<U> runApplyUnGet(Supplier<U> supplier) throws BizException {
        return runApplyUnGet(supplier, COMMON_POOL);
    }

    /**
     * 异步执行没有执获取执行结果
     *
     * @param supplier 获取
     * @param <U>      响应值类型
     */
    public static <U> CompletableFuture<U> runApplyUnGet(Supplier<U> supplier, ExecutorService executorService) throws BizException {
        return CompletableFuture.supplyAsync(supplier, executorService);
    }

    /**
     * 关闭线程池
     */
    public static void shutDown() {
        COMMON_POOL.shutdown();
    }

    /**
     * 关闭线程池
     */
    public static void shutDown(ExecutorService executorService) {
        if (Objects.nonNull(executorService)) {
            executorService.shutdown();
        }
    }
}

```

###### <font color=red>注意，服务停止时请及时线程池</font>

如果使用spring 框架关闭线程池可以参考以下代码

```java
    @PreDestroy
    public void destroyCommonThreadPoll() {
        AsyncExecutor.shutDown();
    }
```

