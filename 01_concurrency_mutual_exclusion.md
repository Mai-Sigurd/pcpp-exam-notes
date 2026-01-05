# Intro to concurrency and the mutual exclusion problem

## Define and motivate concurrency and mutual exclusion. 
### motivate concurrency
to start with, like the turing machine, we had sequential running machines. Meaning it did one job at time in a given order.
we then found we had the hardware to do several tasks, which have been defined into three categories
**Concurrency is and abstraction for all of these:**
- *Exploitation*
	- Hardware capable of simultaneously executing multiple streams of statements
	- several proccessing units sitting in the same piece of hardware
	- exploing the properites of the hardware
	- like using the cpu to the fullest extend. 
- *Hidden (Virtual)*
	- Enabling several programs to share some resources in a manner where each can act as if they had sole ownership
	- the user does not experience the other users.
	- like a website 
- *Inherent*
	- User interfaces and other kinds of input/output
	- the are many things going on, its the whole point of the device that many things can go on at the same time
	- like a operating system

>[!What kind of concurrency is requesting a web-page from a server?]
> hidden 

### mutual exclusion
to define mutual exclusion we must understand the critical sections of a program. IE sections that we want to ensure only thread enters at a time. 

**An ideal solution to the mutual exclusion problem must ensure the following properties:**
- *Mutual exclusion:* at most one thread executing the critical section at the same time
- *Absence of deadlock*: threads eventually exit the critical section allowing other threads to enter
- *Absence of starvation*: if a thread is ready to enter the critical section, it must eventually do so
	- In practice, we will see that it is not always possible to achieve absence of starvation

## Explain data races, race conditions, and critical sections. 

### Race conditions vs data races

**Race conditions**
A race condition occurs when the result of the computation depends on the interleavings of the operations
The interleaving being 

**Data races**
 A data race occurs when two concurrent threads:
 - Access a shared memory location
 - At least one access is a write

*Not all race conditions are data races*
- Threads may not access shared memory
- Threads may not write on shared memory
*Not all data races result in race conditions*
- The result of the program may not change based on the writes of threads

**Extras**
- *Atomicity* : Atomic statements are executed as a single (indivisible) operation
	- ![[Pasted image 20260102123008.png]]
- *Interleaving*
	- The statements in a thread are executed when the thread is in its “running” state
	- An interleaving is a possible sequence of operations for a concurrent program
	- Note this: a sequence of operations for a concurrent program, not for a thread. Concurrent programs are composed by 2 or more threads.
	- ```<thread>(<step>), <thread>(<step>), …``` = ```t1(1), t2(2)```
> [!Are there any other interleavings?]
> There are as many interleavings as possible ways to order the operations in the program

- *States of a thread*
	- ![[Pasted image 20260102131707.png|500]]
	- The statements in a thread are executed when the thread is in its “running” state
### Critical sections
A critical section is a part of the program that only one thread can execute at the same time • Useful to avoid race conditions in concurrent programs
 ![[Pasted image 20260102132104.png]]
 - Simple protocol: call **lock()** before entering the critical section, and **unlock()** after exiting
- Each critical section must have a lock associated to it, but many critical sections may use the same lock.

## Show some examples of code from your solutions to the exercises in week 1.

### 1.1
```java
    class LongCounter {
        private final ReentrantLock lock = new ReentrantLock();
        private long count = 0;

        public void increment() {
            lock.lock(); /// new
            count = count + 1;
            lock.unlock(); //// new
        }

        public long get() {
            return count;
        }
    }
}
```

1. We get 19065332. Non-deterministic and changes per run, as expected.

2. after change from millions to 200, -We consistently get 200. But it is not guaranteed to be the same every time.

   We believe that 200 is below the number of operations before a "context switch" (i.e. before control is yielded to another thread).

3. No. We believe that `count++` and `count += 1` are essentially syntactic sugar for `count = count + 1`. None of the three are atomic operations, and all of them have separate read and write steps.

   We don't see any difference in the results.

4. We have introduced a critical section, covering the single `count = count + 1` statement, guarded by a mutex that ensures mutual exclusion for all concurrent threads. The output is guaranteed to be correct and consistent because the previously possible race conditions are now prevented by the mutex.

5. Yes, it is the least lines of code. Only the implicit read and write steps of the increment operation are in the critical section, and these are the only steps that we want to guarantee happen atomically.


### 1.2
```java
    class Printer {
        private final ReentrantLock lock = new ReentrantLock();

        public void print() {
            lock.lock();
            System.out.print("-"); // (1)
            try { Thread.sleep(50); } catch (InterruptedException exn) { }
            System.out.print("|"); // (2)
            lock.unlock();
        }
    }
}
```
- We have added comments `// (1)` and `// (2)` to the code in `PrinterProgram.java` to indicate steps 1 and 2 used below.
    
    The interleaving `t1(1), t2(1), t1(2), t2(2)` results in the output `--||`.
    
- We have added the lock to the same program as above.
    
    We have introduced a critical section consisting of the two print statements, whereby both `-|` are printed as an atomic operation. An interleaving of `t1(1), t2(1)` is no longer possible due to the mutual exclusion guaranteed by the lock.
### 1.3
1. Modify the behaviour of the Turnstile thread class so that that exactly 15000 enter the park; no less no more. To this end, simply add a check before incrementing counter to make sure that there are less than 15000 people in the park. Note that the class does not use any type of locks. You must use ReentrantLock to ensure that the program outputs the correct value, 15000.
	```java
    long counter = 0;
    final long PEOPLE  = 10_000;
    final long MAX_PEOPLE_COVID = 15_000;
    
    ...
    public class Turnstile extends Thread {
        private final ReentrantLock lock = new ReentrantLock();

        public void run() {
            for (int i = 0; i < PEOPLE; i++) {
                lock.lock();

                if (counter >= MAX_PEOPLE_COVID) {
                    continue;
                }

                counter++;

                lock.unlock();
            }
        }
    }
}
	```
2. *Explain why your solution is correct, and why it always output 15000.*
	- We added a critical section, wherein both the maximum capacity check and the increment happen. Since this is now an atomic operation, the counter cannot be modified inbetween the check and the increment, making a race condition between threads between the two steps impossible.
## Blue questions
### Two threads
>[!What value of counter will this program print]
> less than 20_000, larger than 10_000

![[Pasted image 20250825093525.png]]

>[! What is the minimum value of counter that this program can print]
>10_000 or 10_001 ?
