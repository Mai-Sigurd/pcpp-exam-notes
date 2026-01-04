# Testing: 
*“Program testing can be used to show the presence of bugs, but never to show their absence!”*
## Explain the challenges in ensuring the correctness of concurrent programs. 
**A (concurrent) program is correct if and only if it satisfies its specification**

Specifications are often stated as a collection of program properties
A property can be seen as a single statement of a program specification 


Testing concurrent programs is about writing tests to find undesired *interleavings* (if any)
	An interleaving is a possible sequence of operations for a concurrent program 
- These are commonly known as counterexamples 
- They show an interleaving that violates a property
Since concurrent execution is non-deterministic, it is not guaranteed that tests will trigger undesired interleavings

### Concurrency properties
#### Safety – “Something bad never happens” 
A *counterexample* is a **finite** interleaving where the property does not hold

- Ex. 1: Two intersection traffic lights are never green at the same time
	- Counterexample: a finite interleaving (finite sequence of traffic light states) where the two traffic lights are green at the same time
	-
- Ex. 2: The field size of a collection is never less than 0
> [!Can you give a counterexample for this property?]
> - Two threads decrementing at the same time, class not thread safe, therefore it can go to -1

#### Liveness – “Something good will eventually happen” 
A *counterexample* is an **infinite** interleaving where the property never holds.

- Ex. 1: The traffic light will eventually switch to red
	- *Counterexample*: an infinite interleaving (infinite sequence of traffic light states) where the traffic light is never red. For instance, a traffic light that is always green.
	
- Ex. 2: It should always be possible to eventually add elements to the collection
	- *Counterexample*: an infinite interleaving when a thread can never add an element to the collection. For instance, if writer threads trying to access an unfair reader-writer monitor.

## Describe different testing strategies for concurrent programs, and their advantages and disadvantages. 

Some strategies to take into account when developing a test: 
1. Precisely define the property you want to test 
2. If you are going to test multiple implementations, it is useful to define an interface for the class you are testing 
3. Concurrent tests require a setup for starting and running multiple threads 
	+ Maximize contention to avoid a sequential execution of the threads 
	+ You may need to define thread classes 
4. Run the tests multiple times and with different setups to try to maximize the number of interleavings tested

#### 1. Precisely define the property you want to test 

![[Pasted image 20250922085329.png]]
> [! Is this a safety or liveness property?]
> I think safety, because it ensures that a certain threshold gets met 
> Lecturer: Safety because the counter example is finite. N*X is finite

#### 3. Maximize thread contention 
- Maximizing the number of threads running concurrently
A cyclic barrier may be used to decrease the chance that threads are executed sequentially
### Limitations of testing
- Testing is an extremely useful technique, which is the de-facto approach in industry 
	- You should extensively test all your programs! 
- However, it cannot be used to prove the absence bugs (remember the first slides) 
- Tests can be seen as interleaving generators ☺ 
	- They stimulate the system to produce different interleavings 
	- For most systems, it is virtually impossible to write a set of tests that cover all possible interleavings in the system

#### Deadlocks
A deadlock occurs when a thread does not eventually leave the critical section; thus it prevents other threads from executing the critical section 
- For instance, if all threads are in a critical section waiting for a lock held by another thread

Testing for deadlocks is complicated and often not possible

> [!  Is deadlock-freedom a safety property or a liveness property? Why?]
>  
> Answer: **Safety**, difference between counter example and what happens in the test. The overservable behavoir is infinte waiting, but the counter example is a finite interleaving. 

> [! Hard: Whats the minimum value of counter that his program can print]
> answer: 2
> reason, first thread almost gets done, then second thread sets counter to 1, first thread reads that, second thread finishes, first thread increments 1 to 2 and finishes its last iteration. 
![[Pasted image 20250922093100.png]]
## Show some examples of code from your solutions to the exercises in week 5.
### 5.1

#### 1
Here i have used a cyclic barrier, such that we know we decrease the chance of sequintal runs. 
cyclic barrier is the minions bike, that needs everybody to get onboard before we start. 
```java
@RepeatedTest(5000)
    public void TestConcurrentIntegerSetBuggyAdd() throws Exception {
        var barrier = new java.util.concurrent.CyclicBarrier(16+1);

        for(int i=0; i<16; i++){
            new Thread(()->{
                try {
                    barrier.await();
                    set.add(1);
                    barrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        //Above we started 16 threads. Since the barrier is size 16+1, we need the main threads to also await in order for all threads to run
        barrier.await(); //All threads are starting now 
        barrier.await(); //Await for all threads to finish

        assertEquals(1, set.size());
    }

```
We have written a test where 16 threads try to insert the same number, and then we check whether this results in a size of 1. When repeating the test 5000 times, we see an error where the size is not equal to 1.

The interleaving that results in this error is eg. t1 and t2 both reads from the collection at the same time, and conclude that the element is not there and therefore they both add it and increment the size. This is because the add operation is not atomic.

#### 2
We have written a test where 16 threads try to insert the same number and remove it again, and then we check whether this results in a size of 0. When repeating the test 5000 times, we see an error where the size is not equal to 0.

The interleaving that results in this error is a race condition on the size variable. The interleaving is the same as in 5.1.1. This is because the add operation is not atomic.

```java
    
    @RepeatedTest(5000)
    public void TestConcurrentIntegerSetBuggyRemove() throws Exception {
        var barrier = new java.util.concurrent.CyclicBarrier(16+1);

        for(int i=0; i<16; i++){
            new Thread(()->{
                try {
                    barrier.await();
                    set.add(1);
                    set.remove(1);
                    barrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }

        //Above we started 16 threads. Since the barrier is size 16+1, we need the main threads to also await in order for all threads to run
        barrier.await(); //All threads are starting now 
        barrier.await(); //Await for all threads to finish

        assert(set.size() == 0);
    }
```
#### 3
We have added the synchronized keyword to the add and remove methods, thus we make sure that the operations are atomic and the interleavings described in the previous exercises can not happen. The size is already final, thus we haven't done anything with this. The tests pass now.

#### 4
We find no errors when running the tests on the ConcurrentIntegerSetLibrary. The tests pass because the collection is already declared thread safe and we only use single atomic operations from the class.

#### 5
Yes, because if we found a failure we would have a counterexample and then we could prove that the collection is not thread safe.

#### 6
No, running succesfull tests would not let us conclude that the collection is thread safe. There might be a possible interleaving leading to a failure, that our tests did not trigger.

### 5.2

#### 1

- Problem: no condition on `state > 0` in `release`, so `state` can become negative.
- Assume the semaphore has `c = 1`, then the following is possible: `m(release), t1(acquire), t2(acquire)`.
  - Now two threads are in the critical section, but the capacity is only one.

#### 2

- Below is a functional test:
  - Expected: times out after 2 seconds because one of the threads is blocked on `acquire`
  - Actual: no blocking and hits `assert(false)` instead
- The issue is that the initial `release` makes `state = -1`, so two `acquire` calls can succeed without blocking despite `capacity = 1`.

```java
    @Test
    @Timeout(2)
    public void TestBadSemaphore() throws Exception {
        var barrier = new java.util.concurrent.CyclicBarrier(2+1);

        semaphore.release(); // state = -1

        for(int i = 0; i < 2; i++){
            new Thread(()->{
                try {
                    barrier.await();

                    semaphore.acquire();

                    barrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }


        barrier.await();
        barrier.await();

        assert(false);
    }
```