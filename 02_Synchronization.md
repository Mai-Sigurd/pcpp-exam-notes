 **Synchronization**: 
 Explain and motivate how locks, monitors, and semaphores can be used to address the challenges caused by concurrent access to shared memory. Show some examples of code from your solutions to the exercises in week 2.

## Explain and motivate how locks, monitors, and semaphores can be used to address the challenges caused by concurrent access to shared memory.


### Locks
locks or mutexes ensure that only one thread access something at a time. 


-  **lock()**
	- Acquires the lock if available, otherwise it blocks
	- It is blocking

- **unlock()**
	- Releases the lock, if there are other threads waiting for the lock it signals one of them
	- It is not blocking

- These are synchronization operations
- They established an execution order among the operations of different threads

reentrant lock, can be locked multiple times by the same thread, requiring the same amount of unlocks to be free again. 
### Monitors
- A monitor is a structured way of encapsulating data, methods and synchronization in a single modular package
- A monitor consists of:
	- Internal state (data)
	- Methods (procedures)
		- All methods in a monitor are mutually exclusive (ensured via locks)
		- Methods can only access internal state
	- Condition variables (or simply conditions)
		- Queues where the monitor can put threads to wait
+ In Java (and generally in OO), monitors are conveniently implemented as classes
#### Conditions
NOT BOOLEAN
- Conditions are used when a thread must wait for something to happen, e.g.,
- A writer thread waiting for all readers and/or writer to finish
	- A reader waiting for the writer to fin

- Queues in condition variables provide the following interface:
	- **await()**– releases the lock, and blocks the thread (on the queue)
	- **signal()**– wakes up a thread blocked on the queue, if any
	- **signalAll()**– wakes up all threads blocked on the queue, if any

- When threads wake up they acquire the lock immediately (before the execute anything else)
### Semaphores
Semaphores are synchronization primitives that allow at most c number of threads in the critical section where c is called the capacity

The inituiton is the bouncer really into fire safety, only so many people can be in the building at the same time

**A semaphore consists of:**
+ An integer capacity (c), permits in Java
	+ Initial number of threads allowed in the critical section
+ A method acquire()
	+ Checks if c > 0, if so, it decrements capacity by one (c--) and allows the calling thread to make progress, otherwise it blocks the thread
	+ It is a blocking call
+ A method release()
	+ It checks whether there are waiting threads, if so, it awakes one of them, otherwise it increases the capacity by one (c++)
	+ It is non-blocking

*Synchronization primitives that only allow one thread in the critical section are calledmutex (which is short for mutual exclusion)*
## Show some examples of code from your solutions to the exercises in week 2.

In exercise 2 we encounted the reader - writer problem which states that
+ Many threads can read from the structure as long as no thread is writing
- At most one thread can write at the same time

this cannot be solved by mutex locks. 
	 NO!, because of when someone aquires the lock, then they need to check about writers or readers are existing.
### ex1 reader writer problem
#### read writer
```java
public class ReadWriteMonitor {
    int readers = 0;
    boolean writer = false;

    public synchronized void readLock() {
        while (writer) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        readers++;
    }
    
    public synchronized void readUnlock() {
        readers--;
        if (readers == 0) {
            notifyAll();
        }
    }

    public synchronized void writeLock() {
        while (readers > 0 || writer) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        writer = true;
    }

    public synchronized void writeUnlock() {
        writer = false;
        notifyAll();
    }
    
}
```
this yielded an unfair read writer monitor as all the readers would go first and then the writers would go last. 

The current monitor is not fair, since all the reads happen at first and then the writes at last. We have implemented a fair monitor in file `FairReadWriteMonitor.java`. We have now modified the `writeLock()` method such that it first waits for writer to become `false`, and then mark that there is a writer waiting. This prevents new readers from acquiring the lock. Then, the writer waits for all the current readers to release the lock.

```java
public class FairReadWriteMonitor {
    int readers = 0;
    boolean writer = false;

    public synchronized void readLock() {
        while (writer) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        readers++;
    }
    
    public synchronized void readUnlock() {
        readers--;
        if (readers == 0) {
            notifyAll();
        }
    }

    public synchronized void writeLock() {
        while (writer){
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        writer = true;
        
        while (readers > 0) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public synchronized void writeUnlock() {
        writer = false;
        notifyAll();
    }
    
}
```
now we get that if a writer wants access, it needs to wait for the current readers, but no new readers will be allowed in. They will have to wait. 

> [!Blue question] 
> Is it necessary to check whether **readers == 0**
>
>  -  it might just wake up threads that need to go to sleep immeadite anyway
>  - theres less overhead, as if the readers are not zero then whoever is waiting to aqquire entrance is asleep anyway, will still neeed to be if readers is not zero


**Condition variables?**
Yes, we use one condition variable on the intrinsic lock used by synchronized. We call wait() and notifyAll() to make the threads wait until some condition evaluates to true and then to notify all other threads that state has changed.


Is it possible to ensure absence of starvation using ReentrantLock or intrinsic java locks (synchronized)?
	they would need to have a fair mode, wich reeantrantlock has, but intrincsic java lock does not
*Absence of starvation: if a thread is ready to enter the critical section, it must eventually do so*


rest of exercises 
https://github.itu.dk/raw/jst/PCPP2025-Public/main/week02/exercises02.pdf?token=GHSAT0AAAAAAAAAHM3VKNYFMGMRUW26I2BA2LBEKNQ 

### 2.2 
https://github.itu.dk/adbo/pcpp-exercises/blob/main/Assignment1/Exercise2/week02exercises/app/src/main/java/exercises02/TestLocking0.java

1. We observe that the thread t loops forever. We assume it's because the main thread running on one CPU writes to the variable which is only written to local CPU cache, but thread t never sees this because it is running on another CPU. Thus, this is a visibility problem.
2. We have added the synchronized keyword to the methods in the MutableInteger class, and this ensures that the thread always terminates. This is because when the implicit lock and unclock calls are invoked, these flush the CPU caches to main memory and synchronize local caches. We see output:
```
mi set to 42, waiting for thread ...
I completed, mi = 42
Thread t completed, and so does main
```

3. No we assume not since we both need the value to be flushed out from the set and into the get. Synchronized set makes the values be synchronized to main memory, but we also need the get method to be synchronized, such that the changes in main memory are pulled into the local CPU cache. 
   
4. Yes, thread t always terminates when value is declared volatile. This is because volatile variables are only saved in shared memory. Thus, volatile variables can solve visibility issues.

### 2.3
https://github.itu.dk/adbo/pcpp-exercises/blob/main/Assignment1/Exercise2/week02exercises/app/src/main/java/exercises02/TestLocking0.java

1. Yes, there are race conditions. We get the sums:
```
Sum is:
1069312,000000 
1036185,000000
1032732,000000
1038614,000000
1118299,000000
```
2. The problem is that the two threads each have a seperate lock, but they both mutate the same shared data. This happens because the addInstance method is not static (not mutually exclusive on sum), whereas the addStatic is declared static. addStatic and addInstance can run concurrently, because they lock two different objects.
addStatic uses the lock on the Mystery.class object, whereas addInstance uses the lock on the this object.
   
3. We have added the following to the addInstance method:
```java
    public synchronized void addInstance(double x) {
        synchronized(Mystery.class) {
            sum += x;
        }
    }
```
Here we declare that the addInstance method should use the lock on the Mystery.class object, and not only this. Now there is mutual exclusion on the sum field, because when we access it we use the same lock.

4. The synchronized keyword is not neccesary for the sum method in this particular program, because sum is only called after both threads have finished running. Also, even if the sum method was called during the execution of the threads, it would still be okay since it's just a single atomic read step.