---
    layout: post
    title: CompletableFuture使用
---

## 1. CompletableFuture.supplyAsync()

- 用AsyncSupply包装了提交supply，默认用ForkJoinPool.commonPool()

- f.get() 运行用户的supply，异常的话completeThrowable 设置result为AltResult()，d.postComplete();去触发之前whenComplete,thenApply等stack中的
```java
@SuppressWarnings("serial")
    static final class AsyncSupply<T> extends ForkJoinTask<Void>
            implements Runnable, AsynchronousCompletionTask {
        CompletableFuture<T> dep; Supplier<T> fn;
        AsyncSupply(CompletableFuture<T> dep, Supplier<T> fn) {
            this.dep = dep; this.fn = fn;
        }

        public final Void getRawResult() { return null; }
        public final void setRawResult(Void v) {}
        public final boolean exec() { run(); return true; }

        public void run() {
            CompletableFuture<T> d; Supplier<T> f;
            if ((d = dep) != null && (f = fn) != null) {
                dep = null; fn = null;
                if (d.result == null) {
                    try {
                        d.completeValue(f.get());
                    } catch (Throwable ex) {
                        d.completeThrowable(ex);
                    }
                }
                d.postComplete();
            }
        }
    }
```

- 这个stack是一个abstract Completion，其实就是whenComplete,thenApply ...等操作组装的stage，然后tryFire()进行链式的调用  
```java
/**
 * Pops and tries to trigger all reachable dependents.  Call only
 * when known to be done.
 */
final void postComplete() {
    /*
     * On each step, variable f holds current dependents to pop
     * and run.  It is extended along only one path at a time,
     * pushing others to avoid unbounded recursion.
     */
    CompletableFuture<?> f = this; Completion h;
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        if (f.casStack(h, t = h.next)) {
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                h.next = null;    // detach
            }
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}
```


## 2.CompletableFuture.get()
- 主要对result的while()判断，超时和中断的处理封装了Signaller
```java
/**
 * Returns raw result after waiting, or null if interrupted, or
 * throws TimeoutException on timeout.
 */
private Object timedGet(long nanos) throws TimeoutException {
    if (Thread.interrupted())
        return null;
    if (nanos <= 0L)
        throw new TimeoutException();
    long d = System.nanoTime() + nanos;
    Signaller q = new Signaller(true, nanos, d == 0L ? 1L : d); // avoid 0
    boolean queued = false;
    Object r;
    // We intentionally don't spin here (as waitingGet does) because
    // the call to nanoTime() above acts much like a spin.
    while ((r = result) == null) {
        if (!queued)
            queued = tryPushStack(q);
        else if (q.interruptControl < 0 || q.nanos <= 0L) {
            q.thread = null;
            cleanStack();
            if (q.interruptControl < 0)
                return null;
            throw new TimeoutException();
        }
        else if (q.thread != null && result == null) {
            try {
                ForkJoinPool.managedBlock(q);
            } catch (InterruptedException ie) {
                q.interruptControl = -1;
            }
        }
    }
    if (q.interruptControl < 0)
        r = null;
    q.thread = null;
    postComplete();
    return r;
}
```

