在 Disruptor 3.4.4 中，实现了 8 种消费者等待策略，不同的等待策略对 cpu 的使用率有不同程度的影响，下面展示了有哪些实现类：

![等待策略](https://github.com/notayessir/blog/blob/main/images/disruptor/WaitStrategy.png)

下面依次看看它们的内部实现：

**BlockingWaitStrategy**

BlockingWaitStrategy 内部使用了 ReentrantLock 和 Condition await 实现等待，源码及注释如下：

```java
@Override
public long waitFor(long sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier) throws AlertException, InterruptedException
{
    long availableSequence;
  	// 生产者当前指针 cursorSequence 小于当前消费者指针，加锁并等待
		if (cursorSequence.get() < sequence)
		{
        lock.lock();
        try {
        		while (cursorSequence.get() < sequence) {
            		barrier.checkAlert();
                processorNotifyCondition.await();
            }
        }
        finally {
            lock.unlock();
        }
    }
  	// 判断当前消费者，所依赖的消费者指针是否大于当前消费者指针，如果小于，则进行忙循环
		while ((availableSequence = dependentSequence.get()) < sequence) {
				barrier.checkAlert();
      	// 若 JDK 版本支持调用 Thread.onSpinWait()，则进行忙循环等待，
        // 否则就循环调用 ThreadHints.onSpinWait()
        ThreadHints.onSpinWait();
    }
		return availableSequence;
}
```

从 JDK 1.9 开始，Thread 类新增了 onSpinWait() 方法，告诉 CPU 进入忙循环等待某些条件的发生（通常这个条件很快被唤醒），详见 stack overflow 上一个解释：https://stackoverflow.com/questions/44622384/onspinwait-method-of-thread-class-java-9

**LiteBlockingWaitStrategy**

对 BlockingWaitStrategy 的优化，当锁无竞争时，在官方测试中有提升。源码及说明暂时忽略。

**TimeoutBlockingWaitStrategy**

TimeoutBlockingWaitStrategy 使用 ReentrantLock 和 Condition awaitNanos 来实现等待策略，源码及注释如下：

```java
@Override
public long waitFor(final long sequence, final Sequence cursorSequence, final Sequence dependentSequence, final SequenceBarrier barrier) throws AlertException, InterruptedException, TimeoutException
{
		long nanos = timeoutInNanos;
		long availableSequence;
    if (cursorSequence.get() < sequence) {
				lock.lock();
        try {
						while (cursorSequence.get() < sequence) {
								barrier.checkAlert();
              	// 超时等待
                nanos = processorNotifyCondition.awaitNanos(nanos);
								if (nanos <= 0) {
										throw TimeoutException.INSTANCE;
                }
            }
        }
        finally {
        		lock.unlock();
        }
    }
		while ((availableSequence = dependentSequence.get()) < sequence) 
				barrier.checkAlert();
    }
		return availableSequence;
}
```

**LiteTimeoutBlockingWaitStrategy**

对 TimeoutBlockingWaitStrategy 的优化，当锁无竞争时，在官方测试中有提升。源码及注释暂时忽略。

**SleepingWaitStrategy**

SleepingWaitStrategy 使用 Thread.yield() 与 LockSupport.parkNanos() 方法进行等待，源码与注释如下：

```java
// 不同数值使用不同策略
int retries = 200;
// 睡眠时间
long sleepTimeNs = 100;
@Override
public long waitFor(final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier) throws AlertException {
		long availableSequence;
		int counter = retries;
		while ((availableSequence = dependentSequence.get()) < sequence) {
      	// 睡眠策略
				counter = applyWaitMethod(barrier, counter);
    }
		return availableSequence;
}

private int applyWaitMethod(final SequenceBarrier barrier, int counter) throws AlertException {
		barrier.checkAlert();
  	if (counter > 100) {	// 大于 100 时使用自减
    		--counter;
    } else if (counter > 0) {	// 0 - 100 时，使用 yield，让出 CPU
				--counter;
        Thread.yield(); 
    } else {								// 小于 0，睡眠
    		LockSupport.parkNanos(sleepTimeNs);
    }
  	return counter;
}
```

**YieldingWaitStrategy**

YieldingWaitStrategy 即使用 yield 方法进行等待，源码与注释如下：

```java
@Override
public long waitFor(final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier) throws AlertException, InterruptedException {
		long availableSequence;
		int counter = 100;
		while ((availableSequence = dependentSequence.get()) < sequence) {
    		counter = applyWaitMethod(barrier, counter);
		}
		return availableSequence;
}

private int applyWaitMethod(final SequenceBarrier barrier, int counter) throws AlertException {
		barrier.checkAlert();
		if (0 == counter) {	// 等于 0 时，使用 yield
				Thread.yield();
		} else {						// 大于 0 时使用自减
				--counter;
		}
		return counter;
}
```

**BusySpinWaitStrategy**

BusySpinWaitStrategy 使用 onSpinWait 忙循环实现等待策略，源码与注释如下：

```java
@Override
public long waitFor(final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier) throws AlertException, InterruptedException {
		long availableSequence;
		while ((availableSequence = dependentSequence.get()) < sequence) {
				barrier.checkAlert();
      	// 忙循环，参考 BlockingWaitStrategy
				ThreadHints.onSpinWait();
    }
		return availableSequence;
}
```

**PhasedBackoffWaitStrategy**

PhasedBackoffWaitStrategy 使用忙循环、yield 并结合上面几种实现的等待策略，流程是，首先进行忙循环、再进行 yield、再根据构造参数传入的等待策略进行等待，源码与注释如下：

```java
@Override
public long waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier) throws AlertException, InterruptedException, TimeoutException {
		long availableSequence;
		long startTime = 0;
		int counter = 10000;
		do {
				if ((availableSequence = dependentSequence.get()) >= sequence) {
                return availableSequence;
				}
				if (0 == --counter) {		// 1、不等于 0 时，使用自减
  					if (0 == startTime) {
            		startTime = System.nanoTime();
            } else {						
            		long timeDelta = System.nanoTime() - startTime;
                if (timeDelta > yieldTimeoutNanos){		// 3、yield 超过一定时间，使用其他策略
                		return fallbackStrategy.waitFor(sequence, cursor, dependentSequence, barrier);
                } else if (timeDelta > spinTimeoutNanos) {	// 2、忙循环超过一定时间，使用 yield
                    Thread.yield();
                }
            }
          	// 重置 counter，继续自减
            counter = 10000;
        }
    } while (true);
}
```

