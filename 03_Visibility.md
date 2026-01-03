## Explain the problems of visibility and reordering in shared memory concurrency. 
### Visibility 
Visibility refers to when changes made by one thread to shared variables become visible to other threads. Without proper synchronization, a thread might never see updates made by another thread, or see them with significant delay.

• In the absence of synchronization operations the CPU is allowed to keep data in the CPU’s registers/cache
	• Thus, it might not be visible for threads running on a different CPU
	• These are hardware optimizations to increase performance
![[Pasted image 20250901095324.png]]
Blue question answer: Threads running in different CPUs cannot access each other's Core registers

### Reordering 
Both compilers and processors can reorder instructions for optimization purposes. Operations that appear in a certain order in your source code might execute in a different order, which can break concurrent programs.

 **In the absence of data dependences or synchronization operations,** 
+ the processor (CPU) or 
+ Just-In-Time compiler (JIT) in the Java Virtual Machine (JVM) can reorder Java bytecode operations

Thus, write accesses may be perceived as reordered as compared to the order in the definition of the thread

Reordering is intended to increase performance 
+ For instance, parallelizing tasks 
+ Most programming languages incorporate these optimizations

![[Pasted image 20260103130147.png]]
therefore the output can be 0,0       
we cannot fix this with syncrohnized because, the threads do not look up the value of the variables int eh other cpu register. 
#### Happens before

## Motivate and describe the use of volatile variables and locks to tackle these problems. 

### Volatile
+ Java provides a weak form of synchronization via the variable/field modifier volatile 
+ Volatile variables are not stored in CPU registers or low levels of cache hidden from other CPUs 
	+ Writes to volatile variables flush registers and low level cache to shared memory levels
+ Volatile variables cannot be reordered 
+ Volatile variables cannot be used to ensure mutual exclusion! 
	+ Note that neither reads or writes are blocking operations

**difference volatile**
+ *Volatile variables can* 
	+ Ensure visibility 
	+ Prevent reordering 
+ *Locking can* 
	+ Ensure visibility 
	+ Prevent reordering 
	+ Ensure mutual exclusion
	
## Show some examples of code from your solutions to the exercises in week 2.

rest of exercises 
https://github.itu.dk/raw/jst/PCPP2025-Public/main/week02/exercises02.pdf?token=GHSAT0AAAAAAAAAHM3VKNYFMGMRUW26I2BA2LBEKNQ 

### 2.2 
```java
public class TestMutableInteger {
    public static void main(String[] args) {
        final MutableInteger mi = new MutableInteger();
        Thread t = new Thread(() -> {
                while (mi.get() == 0)        // Loop while zero
                    {/* Do nothing*/ }
                System.out.println("I completed, mi = " + mi.get());
        });
        t.start();
        try { Thread.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
        mi.set(42);
        System.out.println("mi set to 42, waiting for thread ...");
        try { t.join(); } catch (InterruptedException e) { e.printStackTrace(); }
        System.out.println("Thread t completed, and so does main");
    }
}

class MutableInteger {
    // WARNING: Not ready for usage by concurrent programs
    private volatile int value = 0;
    public void set(int value) {
        this.value = value;
    }
    public int get() {
        return value;
    }
}
```


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

```java
class Mystery {
    private static double sum = 0;

    public static synchronized void addStatic(double x) {
        sum += x;
    }

    public synchronized void addInstance(double x) {
        synchronized(Mystery.class) {
            sum += x;
        }
    }

    public static double sum() {
        return sum;
    }
}
```
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