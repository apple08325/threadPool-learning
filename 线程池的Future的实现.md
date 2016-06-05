# 线程池的Future的实现
[TOC]
##AbstractExecutorService.submit(Callable&lt;T&gt; callable)实现
* 在submit的时候会把Callable包装成RunnableFuture&lt;T&gt;的对象，RunnableFuture继承了Runnable接口
``` java
	public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        //RunnableFuture继承了Runnable接口,ftask是FutureTask的一个实例
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```

```java
	public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
```


## FutureTask详解
### 构造函数
构造函数将callable对象赋给成员变量，并且设定当前的状态为NEW。

### run方法
FutureTask重写了run方法，然后内部调用callable对象的call，最终把call的返回值赋值给成员变量outcome，完成任务的执行，具体步骤如下：
1. 调用成员变量callable的call方法
2. 设置call的返回（调用set方法）
 * 状态设为COMPLETING
 * 返回值赋给outcome
 * 状态设为NORMAL
 * 在finishCompletion方法里面唤醒所有取值线程

### get方法
get方法提供了无参和有timeout时间两种方法，先判断内部state的状态，没有完成的话就等待，直到timeout时间到或者有返回后继续执行。
1. 判断state方法，如果小于等于completing，则抛timeout异常
2. 通过awaitDone判断state状态
 * 如果当前线程被中断，则抛中断exception
 * 如果当前状态>completing则直接返回当前状态
 * 如果状态处于completing的状态，暂时先让出时间片
 * 如果没有压栈，就把当前线程包装成WaitNode压入waiters栈里。
 * 如果有timeout时间，时间到了后就把当前线程出栈，返回当前状态，如果时间没有到，则通过带堵塞时间参数的LockSupport.parkNanos方法堵塞住当前线程
 * 如果没有timeout时间，则 通过无参的LockSupport.parkNanos堵塞当前线程。
3. 通过report方法校验当前状态
 * 如果当前状态为NORMAL直接返回outcome的值
 * 其他情况抛CancellationException或者ExecutionException

```java
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

