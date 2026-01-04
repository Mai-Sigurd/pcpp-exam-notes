# Lock-free Data Structures: 
## Define and motivate lock-free data structures.

motivate ???? TODO
**Pros** 
- A CAS operation is faster than acquiring a lock 
- An unsuccessful CAS operation does not cause thread de-scheduling (blocking) 
**Cons** 
- CAS operations result in high memory overhead


 **val.compareAndSwap(a,b)** •
 - Compares the value of val and a, and, if they are equal val is set to b, otherwise it does nothing. In either case, it returns the current value in val, i.e., the value when CAS was executed 
CAS is implemented at an architecture level. 

Instead of complete blocking other threads, we can now try this optimistic approach of trying until we succeed

```java
public int addAndGet(int delta) { 
	int oldValue, newValue; 
	do { 
		oldValue = get(); 
		newValue = oldValue + delta; 
	} while (!compareAndSet(oldValue, newValue)); 
	return newValue; 
} 

public int getAndAdd(int delta) { 
	int oldValue, newValue; 
	do { 
		oldValue = get(); 
		newValue = oldValue + delta; 
	} while (!compareAndSet(oldValue, newValue)); 
return oldValue; 
}
```


**There exist 3 main notions of progress in non-blocking data structures/computation** 
1. *Wait-free*: A method of an object implementation is wait-free if every call finishes its execution in a finite number of steps 
	- My operations are guaranteed to complete in a bounded number of steps (no matter what other threads do) 
2. *Lock-free*: A method of an object implementation is lock-free if executing the method guarantees that some method call (including concurrent) finishes in a finite number of steps 
	- Somebody’s operations are guaranteed to complete in a bounded number of my steps 
	- Most non-blocking data structures are lock-free, e.g., Treabor’s stack 
3. *Obstruction-free*: A method of an object implementation is obstruction-free if, from any point after which it executes in isolation, it finishes in a finite number of steps; 
	- My operations are guaranteed to complete in a bounded of steps (if I get to execute them)

$$wait_{free} \implies lock_{free} \implies obstruction_{free}$$
#### non-blocking $\neq$ busy-wait
- Threads cannot wait forever because other thread finished incorrectly 
	- Even in obstruction-free, completion must be guaranteed when the thread runs in isolation 
- All non-blocking progress definitions set the conditions for termination based on interactions among multiple threads 
	- Threads collaborate with each other
## Explain how *compare-and-swap* (CAS) operations can be used to solve concurrency problems. 
concurrentcy problems can both be in terms of threads thinking they have succesfully written something or simply never being able to do so. 

Cas offers a non blocking solution. In that a thread can try to do a write action and if it fails, it will simply try again. 

this offers less overhead as the scheduler does not have to do much to restart threads, they will just continue trying. In scenarios with lots of threads we might reindotroduce this overhead if all of them are trying to write. 

so instead of locking a variable, we can use CAS. 

TODO revisit. 
## Show some examples of code from your solutions to the exercises in week 6.


### 6.1.1

i) No references to the inner reference types are exposed
ii) We add `final`, which ensures safe publication

```java
import java.util.concurrent.atomic.AtomicInteger;

class CasHistogram implements Histogram {
    private final AtomicInteger[] counts;

    public CasHistogram(int span) {
        this.counts = new AtomicInteger[span];
    }

    public void increment(int bin) {
        int oldValue, newValue;
        do {
            oldValue = counts[bin].get();
            newValue = oldValue + 1;
        } while (!counts[bin].compareAndSet(oldValue, newValue));
    }

    public int getCount(int bin) {
        return counts[bin].get();
    }

    public int getSpan() {
        return counts.length;
    }

    public int getAndClear(int bin) {
        while (true) {
            int oldValue = getCount(bin);
            int newValue = 0;

            if (counts[bin].compareAndSet(oldValue, newValue)) {
                return oldValue;
            }
        }
    }
}
```

### 6.1.2

- Yes. The second operation only completes if the value has not changed since it was read.
- This is primarily because of `compareAndSet`
- If the update was unsuccessful, we simply retry

### 6.1.3

We implemented the test using the same techniques as last week, including:

- Repeating the test. We only do 20 times due to time (it takes about 15 seconds to run).
- Using barriers to maximize contention

```java
@RepeatedTest(value = 20)
public void TestGoBrrr() throws Exception {
    // We run only 0-3999 due to hardware issues - we cannot run more than 4000 threads.
    // See:
    //  [warning][os,thread] Failed to start thread "Unknown thread" - pthread_create failed (EAGAIN) for attributes: stacksize: 2048k, guardsize: 16k, detached.
    //  [warning][os,thread] Failed to start the native thread for java.lang.Thread "Thread-4068"
    var numThreads = 4000;

    var barrier = new java.util.concurrent.CyclicBarrier(numThreads+1);

    var histogram = new CasHistogram(30);

    for (int i = 0; i < numThreads; i++) {
        final int j = i;
        new Thread(() -> {
            try {
                barrier.await();

                int factors = countFactors(j);
                histogram.increment(factors);

                barrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
    }

    barrier.await();
    barrier.await();

    var seqHistogram = new Histogram1(30);
    for (int i = 0; i < numThreads; i++) {
        int factors = countFactors(i);
        seqHistogram.increment(factors);
    }

    for (int i = 0; i < 30; i++) {
        assertEquals(seqHistogram.getCount(i), histogram.getCount(i));
    }
}
```

All tests pass:

```sh
$ gradle cleanTest test --tests exercises06.TestHistograms

TestHistograms > TestGoBrrr() > repetition 1 of 20 PASSED
TestHistograms > TestGoBrrr() > repetition 2 of 20 PASSED
...
TestHistograms > TestGoBrrr() > repetition 19 of 20 PASSED
TestHistograms > TestGoBrrr() > repetition 20 of 20 PASSED
```

## 6.2

See `exercises06/app/src/main/java/exercises06/ReadWriteCASLock.java`. For the presentation's sake, we insert it below too. Explanations follow afterwards.

```java
import java.lang.Thread;
import java.util.concurrent.atomic.AtomicReference;

class ReadWriteCASLock implements SimpleRWTryLockInterface {
    private final AtomicReference<Holders> holders = new AtomicReference<>(null);

    public boolean readerTryLock() {
        while (true) {
            var thread = Thread.currentThread();
            var current = holders.get();

            if (current instanceof Writer) {
                return false;
            }

            var newReaders = new ReaderList(thread, (ReaderList) current);

            if (holders.compareAndSet(current, newReaders)) {
                return true;
            }
        }
    }

    public void readerUnlock() {
        while (true) {
            var thread = Thread.currentThread();
            var current = holders.get();

            if (!(current instanceof ReaderList)) {
                throw new RuntimeException("No readers to unlock");
            }

            var readers = (ReaderList) current;

            if (!readers.contains(thread)) {
                throw new RuntimeException("Current thread does not hold a read lock");
            }

            var newReaders = readers.remove(thread);

            if (holders.compareAndSet(current, newReaders)) {
                return;
            }
        }
    }

    public boolean writerTryLock() {
        var writer = new Writer(Thread.currentThread());
        return holders.compareAndSet(null, writer);
    }

    public void writerUnlock() {
        var thread = Thread.currentThread();
        var holder = holders.get();

        if (holder instanceof Writer w && w.thread == thread) {
            holders.compareAndSet(holder, null);
        } else {
            throw new IllegalMonitorStateException("Current thread does not hold the write lock");
        }
    }

    private static abstract class Holders { }

    private static class ReaderList extends Holders {
        private final Thread thread;
        private final ReaderList next;

        public ReaderList(Thread thread, ReaderList next) {
            this.thread = thread;
            this.next = next;
        }

        public boolean contains(Thread thread) {
            var cur = this;
            while (cur != null) {
                if (cur.thread == thread) return true;
                cur = cur.next;
            }
            return false;
        }

        public ReaderList remove(Thread thread) {
            if (this.thread == thread) {
                return this.next;
            }

            if (this.next == null) {
                return null;
            }

            return new ReaderList(thread, this.next.remove(thread));
        }
    }

    private static class Writer extends Holders {
        public final Thread thread;

        public Writer(Thread thread) {
            this.thread = thread;
        }
    }
}
```

### 6.2.1

We implement a simple atomic compare-and-set, which sets a new Writer as the holder only if the reference is null.

### 6.2.2

- We check if the current holder is an instance of a `Writer` and that the writer's thread is the current thread
- If that is the case, we can directly update the underlying reference to null
  - We do not need to check if the operation succeeds, since we have mutual exclusion due to the writer lock
- Otherwise, we throw an error as the unlock would be invalid

### 6.2.3

Main logic:
- Fetch value, check if lock is held by writer.
  - If it is, return false
- Try to add reader to list
  - If the compare-and-set fails, something else modified `holders` in the meantime
  - If that "something else" was a reader, we don't want to fail - we retry the operation

### 6.2.4

- Only if the current holder is an instance of a reader list and that list contains the current thread do we proceed to unlock
- We use an atomic compare-and-set to update the reference to a new list, which excludes the current thread
- Our `remove` method always returns a new object reference, so this is valid even when the thread's node is not the head of the list
  - This solves the ABA problem for us, since each modification to the list results in a new object reference
- If the compare-and-set fails, we retry the operation since some other thread has modified the list in the meantime

### 6.2.5

```java
@Test
public void SequentialNoReaderWhileWriter() {
    var lock = new ReadWriteCASLock();
    assert(lock.writerTryLock());
    assert(!lock.readerTryLock());
}

@Test
public void SequentialNoWriterWhileReader() {
    var lock = new ReadWriteCASLock();
    assert(lock.readerTryLock());
    assert(!lock.writerTryLock());
}

@Test
public void SequentialMustLockForUnlock() {
    var lock = new ReadWriteCASLock();

    try {
        lock.readerUnlock();
        assert(false);
    } catch (Exception e) {
        // Expected
    }

    try {
        lock.writerUnlock();
        assert(false);
    } catch (Exception e) {
        // Expected
    }
}
```

```sh
$ gradle cleanTest test --tests exercises06.TestLocks

TestLocks > SequentialNoWriterWhileReader() PASSED
TestLocks > SequentialNoReaderWhileWriter() PASSED
TestLocks > SequentialMustLockForUnlock() PASSED
```

### 6.2.6

- We added a test that spawns 16 threads, which all try to increment a shared (non-thread-safe) counter.
- Each thread uses `ReadWriteCASLock` by acquiring a writer lock before incrementing the counter and unlocking it afterwards.
- We also use barriers similar to before, in order to maximize contention.
- We observe that all test repetitions succeed.

```java
@RepeatedTest(value = 100)
public void ParallelWritersDontInterleave() throws Exception {
    var numThreads = 16;
    var barrier = new java.util.concurrent.CyclicBarrier(numThreads+1);

    var lock = new ReadWriteCASLock();
    Counter counter = new Counter();

    for (int i = 0; i < numThreads; i++) {
        new Thread(() -> {
            try {
                barrier.await();

                for (int j = 0; j < 10000; j++) {
                    while (!lock.writerTryLock());
                    counter.increment();
                    lock.writerUnlock();
                }

                barrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
    }

    barrier.await();
    barrier.await();

    assertEquals(numThreads * 10000, counter.count);
}
```

For fun as an extra verification: if we make `writerTryLock` always return true and make `writerUnlock` a no-op, we get the following test failures as expected:
```sh
TestLocks > ParallelWritersDontInterleave() > repetition 2 of 100 FAILED
    org.opentest4j.AssertionFailedError at TestLocks.java:107

TestLocks > ParallelWritersDontInterleave() > repetition 3 of 100 FAILED
    org.opentest4j.AssertionFailedError at TestLocks.java:107

TestLocks > ParallelWritersDontInterleave() > repetition 4 of 100 PASSED

TestLocks > ParallelWritersDontInterleave() > repetition 5 of 100 PASSED
```