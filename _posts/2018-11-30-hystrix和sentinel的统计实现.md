---
    layout: post
    title: hystrix和sentinel
---

## 先从使用的配置说起吧
 hystrix的动态配置
    > Hystrix uses Archaius for the default implementation of properties for configuration.

    

* java的注解式使用
    [hystrix-javabica](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica)

    - spring的aop配置
        ```java
        @Configuration
        public class HystrixConfiguration {

            @Bean
             public HystrixCommandAspect hystrixAspect() {
                return new HystrixCommandAspect();
             }
        }
        
       ```

    - 代码式的配置设置，可以动态地从其它配置服务设置（diamond，nacos等）
        > Javanica dynamically sets properties using Hystrix ConfigurationManager. For the example above Javanica behind the scenes performs next action:
    ```java
    ConfigurationManager.getConfigInstance().setProperty("hystrix.command.getUserById.execution.isolation.thread.timeoutInMilliseconds", "500");
    
    ```

## HystrixCircuitBreaker和sentinel的指标statistic
- Hystrix 和 Sentinel 的实时指标数据统计实现都是基于滑动窗口的。Hystrix 1.5 之前的版本是通过环形数组实现的滑动窗口，通过锁配合 CAS 的操作每个桶的统计信息进行更新。Hystrix 1.5 开始对实时指标统计的实现进行了重构，将指标统计数据结构抽象成了响应式流（reactive stream）的形式，方便费者去利用指标信息。同时底层改造成了基于 RxJava 的事件驱动模式，在服务调用成功/失败/超时的时候发布相应的事件，通过一系列的变换和聚合最终得到实的指标统计数据流，可以被熔断器或 Dashboard 消费。


- HystrixCircuitBreaker.HystrixCircuitBreakerImpl的RxJava的窗口流订阅
    ```java
    private Subscription subscribeToStream() {
        /*
         * This stream will recalculate the OPEN/CLOSED status on every onNext from the health stream
         */
        return metrics.getHealthCountsStream()
                .observe()
                .subscribe(new Subscriber<HealthCounts>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(HealthCounts hc) {
                        // check if we are past the statisticalWindowVolumeThreshold
                        if (hc.getTotalRequests() < properties.circuitBreakerRequestVolumeThreshold().get()) {
                            // we are not past the minimum volume threshold for the stat window,
                            // so no change to circuit status.
                            // if it was CLOSED, it stays CLOSED
                            // if it was half-open, we need to wait for a successful command execution
                            // if it was open, we need to wait for sleep window to elapse
                        } else {
                            if (hc.getErrorPercentage() < properties.circuitBreakerErrorThresholdPercentage().get()) {
                                //we are not past the minimum error threshold for the stat window,
                                // so no change to circuit status.
                                // if it was CLOSED, it stays CLOSED
                                // if it was half-open, we need to wait for a successful command execution
                                // if it was open, we need to wait for sleep window to elapse
                            } else {
                                // our failure rate is too high, we need to set the state to OPEN
                                if (status.compareAndSet(Status.CLOSED, Status.OPEN)) {
                                    circuitOpened.set(System.currentTimeMillis());
                                }
                            }
                        }
                    }
                });
    }
    
    ```
- metrics.getHealthCountsStream().observe()这个指标信息流，Observable的构造有下面两处

    ```json
    "circuitBreaker": {
                "circuitBreaker.requestVolumeThreshold": "1000",
                "circuitBreaker.errorThresholdPercentage": "80",
                "circuitBreaker.forceOpen": "false"
            },
    
    ```
    1. BucketedRollingCounterStream的构造方法
        ```java
        this.sourceStream = bucketedStream      //stream broken up into buckets
            .window(numBuckets, 1)          //emit overlapping windows of buckets
            .flatMap(reduceWindowToSummary) //convert a window of bucket-summaries into a single summary
            .doOnSubscribe(new Action0() {
                @Override
                public void call() {
                    isSourceCurrentlySubscribed.set(true);
                }
            })
            .doOnUnsubscribe(new Action0() {
                @Override
                public void call() {
                    isSourceCurrentlySubscribed.set(false);
                }
            })
            .share()                        //multiple subscribers should get same data
            .onBackpressureDrop();          //if there are slow consumers, data should not buffer
        
        ```
    2. BucketedCounterStream
        ```java
        this.bucketedStream = Observable.defer(new Func0<Observable<Bucket>>() {
            @Override
             public Observable<Bucket> call() {
                    return inputEventStream
                    .observe()
                    .window(bucketSizeInMs, TimeUnit.MILLISECONDS) //bucket it by the counter window so we can emit to the next operator in time chunks, not on every OnNext
                    .flatMap(reduceBucketToSummary)                //for a given bucket, turn it into a long array containing counts of event types
                    .startWith(emptyEventCountsToStart);           //start it with empty arrays to make consumer logic as generic as possible (windows are always full)
             }
        });
        
        ```

    - ![流程图](../images/hystrix-metrics-event-driven-flow.png)


- sentinel的LeapArray.currentWindow()

    ```java           
    /**
    * Get bucket item at provided timestamp.
    *
    * @param timeMillis a valid timestamp in milliseconds
    * @return current bucket item at provided timestamp if the time is valid; null if time is invalid
    */
    public WindowWrap<T> currentWindow(long timeMillis) {
        if (timeMillis < 0) {
            return null;
        }
        int idx = calculateTimeIdx(timeMillis);
        // Calculate current bucket start time.
        long windowStart = calculateWindowStart(timeMillis);
        /*
        * Get bucket item at given time from the array.
        *
        * (1) Bucket is absent, then just create a new bucket and CAS update to circular array.
        * (2) Bucket is up-to-date, then just return the bucket.
        * (3) Bucket is deprecated, then reset current bucket and clean all deprecated buckets.
        */
        while (true) {
            WindowWrap<T> old = array.get(idx);
            if (old == null) {
                /*
                *     B0       B1      B2    NULL      B4
                * ||_______|_______|_______|_______|_______||___
                * 200     400     600     800     1000    1200  timestamp
                *                             ^
                *                          time=888
                *            bucket is empty, so create new and update
                *
                * If the old bucket is absent, then we create a new bucket at {@code windowStart},
                * then try to update circular array via a CAS operation. Only one thread can
                * succeed to update, while other threads yield its time slice.
                */
                WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
                if (array.compareAndSet(idx, null, window)) {
                    // Successfully updated, return the created bucket.
                    return window;
                } else {
                    // Contention failed, the thread will yield its time slice to wait for bucket available.
                    Thread.yield();
                }
            } else if (windowStart == old.windowStart()) {
                /*
                *     B0       B1      B2     B3      B4
                * ||_______|_______|_______|_______|_______||___
                * 200     400     600     800     1000    1200  timestamp
                *                             ^
                *                          time=888
                *            startTime of Bucket 3: 800, so it's up-to-date
                *
                * If current {@code windowStart} is equal to the start timestamp of old bucket,
                * that means the time is within the bucket, so directly return the bucket.
                */
                return old;
            } else if (windowStart > old.windowStart()) {
                /*
                *   (old)
                *             B0       B1      B2    NULL      B4
                * |_______||_______|_______|_______|_______|_______||___
                * ...    1200     1400    1600    1800    2000    2200  timestamp
                *                              ^
                *                           time=1676
                *          startTime of Bucket 2: 400, deprecated, should be reset
                *
                * If the start timestamp of old bucket is behind provided time, that means
                * the bucket is deprecated. We have to reset the bucket to current {@code windowStart}.
                * Note that the reset and clean-up operations are hard to be atomic,
                * so we need a update lock to guarantee the correctness of bucket update.
                *
                * The update lock is conditional (tiny scope) and will take effect only when
                * bucket is deprecated, so in most cases it won't lead to performance loss.
                */
                if (updateLock.tryLock()) {
                    try {
                        // Successfully get the update lock, now we reset the bucket.
                        return resetWindowTo(old, windowStart);
                    } finally {
                        updateLock.unlock();
                    }
                } else {
                    // Contention failed, the thread will yield its time slice to wait for bucket available.
                    Thread.yield();
                }
            } else if (windowStart < old.windowStart()) {
                // Should not go through here, as the provided time is already behind.
                return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            }
        }
    }
    ```
* 其它
    - 滑动窗口
        1. tcp滑动窗口
            ![tcp](../images/tcp_slide_window.jpg)
        2. sentinel:默认的实现是基于 LeapArray 的滑动窗口
            ![sentinel总体设计](../images/slots.gif)
            
        3. RxJava:Observable<T>#window()



