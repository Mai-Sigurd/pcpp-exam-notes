# Linearizability: 
## Explain and motivate linearizability. 
The motivation behind linearizability is to use specifications of sequential objects as a basis for the correctness of concurrent objects
+ Specifications of sequential objects are typically expressed as pre- and post-conditions for method calls

### Sequential consistency
**General**
For executions of concurrent objects, an execution is sequential consistent iff
1. Method calls appear to happen in a one-at-a-time, sequential order
	1. *Requires memory commands to be executed in order* from old times
2. Method calls should appear to take effect in program order
	1. *Requires sequential program order for each thread*



**Specific for an object**
A concurrent execution is sequentially consistent **iff** there exists a reordering of operations producing a sequential execution where:
 1. Operations happen one-at-a-time
 2. Program order is preserved (for each thread)
 3. The execution satisfies the specification of the object

 *To show that a concurrent execution is linearizable, we must define linearization points within each method call, which map to sequential execution that satisfy the specification of the object* 
 + A linearization point defines the instant in the execution when the method call “takes effect”
![](Pasted%20image%2020260104164043.png)
> [!Is this concurrent execution sequentially consistent? (Recall we are working with a FIFO queue)]
> ![[Pasted image 20251006085809.png]]
> probably not since it should dequeue x first since x was queued first?
> y should be enqueued first



## Explain how linearizability can be applied to reason about the correctness of concurrent objects. 
### Sequentially Consistent Concurrent Objects
+ A concurrent object is sequentially consistent iff all concurrent executions of method calls are sequentially consistent 
+ Unfortunately, sequential consistency is not a compositional property 
	+ Concurrent executions involving sequentially consistent objects may not be sequentially consistent 
	+ This is a major weakness, as it does not allow for modular reasoning when implementing concurrent objects
	
> [! Is this concurrent execution linearizable? (Recall we are working with a FIFO queue)]
> ![[Pasted image 20251006091803.png]]
> in order to be lineriasibale you need to find one valid linearization, we can find one here, with enq y first, then enq x,  and then deq y, it therefore is sequentially constistent. 
> to argue its not wee need to find all the cases, which is trickier. 
> ![[Pasted image 20251006092631.png]]

we can use linerasions points in the code, to create interleavings that can possibly break the specifications of the program. 
![](Pasted%20image%2020260104173908.png)
![](Pasted%20image%2020260104173914.png)
## Show some examples of code in your solutions to the exercises in week 7 where you used linearizability to reason about correctness.

### Exercise 7.1

#### 7.1.1

Is this execution sequentially consistent? If so, provide a sequential execution that satisfies the standard specification of a sequential FIFO queue. Otherwise, explain why it is not sequentially consistent.

```
A: ---------------|q.enq(x)|--|q.enq(y)|->
B: ---|q.deq(x)|------------------------->
```

Yes it is sequentially consistent. One possible execution is the below:

```
<q.enq(x),q.deq(x),q.enq(y)>
```

#### 7.1.2

Is this execution (same as above) linearizable? If so, provide a linearization that satisfies the standard specification of a sequential FIFO queue. Otherwise, explain why it is not linearizable.

```
A: ---------------|q.enq(x)|--|q.enq(y)|->
B: ---|q.deq(x)|------------------------->
```

No, it is not linearizable. There is no way to specify linearization points in such a way that the enq(x) happens before deq(x)

#### 7.1.3

Is this execution linearizable?

```
A: ---|        q.enq(x)        |-->
B: ------|q.deq(x)|--------------->
```

Yes this exection is linearizable.

```
<q.enq(x),q.deq(x)>
```

#### 7.1.4

Is this execution linearizable?

```
A: ---|q.enq(x)|-----|q.enq(y)|-->
B: --|       q.deq(y)          |->
```

No it is not linearizable because it is a FIFO queue. In order for us to dequeue y, y should have been enqueued before x, and there is no way for us to specify that linearization.

### Exercise 7.2

#### 7.2.1

```java
    public void push(T value) {
        Node<T> newHead = new Node<T>(value);
        Node<T> oldHead;
        do {
            oldHead      = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead,newHead));  // PU(6)

    }

    public T pop() {
        Node<T> newHead;
        Node<T> oldHead;
        do {
            oldHead = top.get();                        // PO(4)
            if(oldHead == null) { return null; }        // PO(5)
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead,newHead));  // PO(7)

        return oldHead.value;
    }
```

We have found one linearization point for push and two for pop.

##### Push

If two threads execute push concurrently, then only one of them will succeed. Given that only one of them will get the correct oldHead and newHead and execute compareAndSet. The other one will fail and try again.

##### Pop

We have one linearazation point, PO(4) which determines the outcome of PO(5)
and another linearazation point PO(7) where we try compareAndSet on top.

If two threads concurrently pop, and the queue is not empty (PO(5) is false) then only one of them will succeed compareAndSet (PO(7)). The thread that fails PO(7) will redo the while loop.

#### 7.2.2

We implement the following:

```java
    @RepeatedTest(10000)
    public void TestStackPush() throws Exception {
        var numThreads = 10;
        var barrier = new java.util.concurrent.CyclicBarrier(numThreads+1);

        var stack = new LockFreeStack<Integer>();
        var expected = 0;

        for(int i = 0; i < numThreads; i++){
            expected += i;

            final int ti = i;
            new Thread(()->{
                try {
                    barrier.await();

                    stack.push(ti);

                    barrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        barrier.await();
        barrier.await();

        var sum = 0;
        Integer cur = null;
        do {
            cur = stack.pop();
            if(cur != null) sum += cur;
        } while(cur != null);

        assertEquals(expected, sum);
    }
```

All tests pass:

```
...
TestLockFreeStack > TestStackPush() > repetition 10000 of 10000 PASSED

BUILD SUCCESSFUL in 7s
```

#### 7.2.3

We implement the following:

```java
    @RepeatedTest(10000)
    public void TestStackPop() throws Exception {
        var numThreads = 10;
        var barrier = new java.util.concurrent.CyclicBarrier(numThreads+1);

        var stack = new LockFreeStack<Integer>();
        var expected = 0;

        for(int i = 0; i < numThreads; i++){
            expected += i;
            stack.push(i);
        }

        var result = new AtomicInteger(0);

        for(int i = 0; i < numThreads; i++){
            new Thread(()->{
                try {
                    barrier.await();

                    var res = stack.pop();
                    if(res != null) result.addAndGet(res);

                    barrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        barrier.await();
        barrier.await();

        assertEquals(expected, result.get());
    }
```

All tests pass:

```
...
TestLockFreeStack > TestStackPop() > repetition 10000 of 10000 PASSED

BUILD SUCCESSFUL in 12s
```

#### 7.2.4

No, we don't cover all linearization points since we don't cover the when-the-stack-is-empty case.
Thus, we add the following test:

```java
    @RepeatedTest(10000)
    public void TestStackEmptyPop() throws Exception {
        var numThreads = 10;
        var barrier = new java.util.concurrent.CyclicBarrier(numThreads+1);

        var stack = new LockFreeStack<Integer>();

        stack.push(42);

        var result = new AtomicInteger(0);

        for(int i = 0; i < numThreads; i++){
            new Thread(()->{
                try {
                    barrier.await();

                    var res = stack.pop();
                    if(res != null) result.addAndGet(res);

                    barrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        barrier.await();
        barrier.await();

        assertEquals(42, result.get());
    }
```

### Exercise 7.3

#### 7.3.1

- `readerTryLock` is lock-free since it uses only CAS and no locks, but it uses a loop to wait for a succesful set. Specifically, using the definition, it is lock-free because some method will always finish in a finite number of steps.
- `readerUnlock` is lock-free the same reason as above
- `writerTryLock` is wait-free since it always finishes in a finite number of steps (it has no loops)
- `writerUnlock` is wait-free the same reason as above
