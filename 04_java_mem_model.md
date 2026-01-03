# Java memory model
## Motivate the need for the Java memory model. 
Memory models formally describe the valid executions behaviour of concurrent programs (including visibility and reordering)

the need for comes from the fact that different hardware would have different set instructions and differetn way of handeling them. This would therefore create inconsistency in programs depedent on which computer it was run on

**Java memory model** 
+ Set of programming language level rules that determine *valid* executions of a program 

The JVM is designed to produce Java bytecode that only produces executions that are valid in the memory model 
+ This is enforced for all hardware and runtime environments 

This gives programmers the ability to predict the behaviour of concurrent programs


## Explain the elements of the Java memory model including program order, happens-before order, synchronization order, and data races. 
Java memory model 
- Set of programming language level rules that determine valid executions of a program
**There are 3 types of actions:**
• *Variable access*
	• Read/write accesses on program variables
• *Synchronization actions*
	• Synchronization primitives such as acquiring and releasing a lock or monitor, starting a thread, waiting for a thread to terminate,…
• *Other (any action not in the previous two categories)*
### program order

+ Program order defines the intra-thread order of execution of the actions of a thread 

+ *Given two actions a, b that are executed by the same thread t, a occurs before b according to program order iff a would be performed before b when t is executed sequentially* 

+ Program order is a total order among the actions executed by a thread 
	+ Recall the definition of total order: https://en.wikipedia.org/wiki/Total_order 
	+ Intuitively, it means that each pair of actions must be related
this is total for an indivual thread. 
### happens-before order
Happens-before order is a partial order among the actions of an execution 
+ Recall the definition of partial order: https://en.wikipedia.org/wiki/Partially_ordered_set 
+ Intuition: Similar to total order, but not all pairs of elements need to be related'
**Program order rule**. Each action in a thread happens-before every action in that thread that comes later in the program order.

**Monitor lock rule**. An unlock on a monitor lock happens-before every subsequent lock on that same monitor lock.

**Volatile variable rule**. A write to a volatile field happens-before every
subsequent read of that same field.

**Thread start rule.** A call to Thread.start on a thread happens-before
every action in the started thread.

**Thread termination rule.** Any action in a thread happens-before any other thread detects that thread has terminated, either by successfully return from *Thread.join* or by *Thread.isAlive* returning false.

**Interruption rule.** A thread calling interrupt on another thread happens-before the interrupted thread detects the interrupt (either by having InterruptedException thrown, or invoking isInterrupted or interrupted).

**Finalizer rule.** The end of a constructor for an object happens-before
the start of the finalizer for that object.

**Transitivity**. If A happens-before B, and B happens-before C, then A happens-before C.

#### Visibility
*Given two actions a,b in an execution, if it holds that a -> b, then the effect of a is visible by b* 
- In other words, we simply need to ensure that actions are ordered by happens-before to ensure visibility 
- Using this property we can also predict the value that a read access on a variable will read

### synchronization order
+ Synchronization order is a total order among the synchronization actions of an execution 

+ Synchronization order must satisfy two properties 
	+ Consistency with mutual exclusion (locking operations must be correctly nested) 
	+ Consistency with happens-before order
	
+ Multiple synchronization orders may be associated to a program 
	+ Due to non-determinism in the execution of synchronization operations 
	+ Each execution has a synchronization order associated to it
### data races

The definition of data race in the Java memory model is slightly different than the one in lecture 1 

*“Given actions a,b in an execution, there exist a data race between a, b iff* 
+ Actions a,b are conflicting 
+ Actions a,b are not ordered by happens-before”

**Conflicting actions** 
define the pairs of actions that may (potentially) lead to concurrency issues 

*“Given actions a, b in an execution, we say that actions a, b are conflicting iff they are accesses on the same (non-volatile) variable and at least one of them is a write access”* 

By definition, accesses on volatile variables are never conflicting

**From lecture 1**
 A data race occurs when two concurrent threads:
 - Access a shared memory location
 - At least one access is a write
## Define what a correctly synchronized program is according to the Java memory model. 

*A program is correctly synchronized iff none of its executions contains data races*
![Pasted image 20260103142002](Pasted%20image%2020260103142002.png)

## Show some examples of code from your solutions to the exercises in week 3 and illustrate the use of the Java memory model to reason about their correctness.
### 3.1
#### 1
```java
    public CountingThreads() throws InterruptedException {
        count = 0; // (Init(count))
        l = new ReentrantLock(); // (init(l))

        CountingThread t1 = new CountingThread();
        CountingThread t2 = new CountingThread();

        t1.start(); // (Start(t1))
        t2.start(); // (Start(t2))

        t1.join(); // (Join(t1))
        t2.join(); // (Join(t2))

        System.out.println("count="+count); // (1)
    }

    public class CountingThread extends Thread {
        public void run() {
            l.lock();         // (lock) (added in later exercise)
            int temp = count; // (2)
            count = temp + 1; // (3)
            l.unlock();       // (unlock) (added in later exercise)

        }
    }
```

We have commented numbers and names for each execution in the file 'CountingThreads.java'

**Variable access**
- (Init(count)), (1), (2), (3)
**synchronization**
- (Start(t1)), (Start(t1)), (Join(t1)), (Join(t2))
- also the lock and unlock but they where added in later exercise

#### 2
$$
HB^{M}_{PO} = m(Init(count)) \to m(start(t1)) \to m(start(t2))  \to m(join(t1)) \to m(join(t2)) \to m(1)
$$
$$
HB^{t1}_{PO} = t1(2) \to t1(3)
$$
$$
HB^{t2}_{PO} = t2(2) \to t2(3)
$$
#### 3

$$HB_{init} = M(start(t1)) \to t1(2)$$
$$HB_{init} = M(start(t2)) \to t2(2)$$
$$HB_{ter} = t1(3) \to m(join(t1))$$
$$HB_{ter} = t2(3) \to m(join(t2))$$

#### 4

**Synchronization order**
`m(start(t1)), m(start(t2)), m(join(t1)), m(join(t2))`

#### 5
Given that in the HB we can not show any relation between t1(2) and t2(3) we can conclude that a data race will occur. Similiarly we can not prove that theres an happens before relation between t2(2) and t1(3). This interleaving will result in dataraces.
#### 6
We have introduced a ReentrantLock which both threads use. The lock locks before execution (2) and unlocks after execution (3).

$$
HB^{1} = HB \cup \{t1(unlock) \to t2(lock)\}
$$
$$
HB^{2} = HB \cup \{ t2(unlock) \to t1(lock) \}
$$

There are two sync orders, either
`m(start(t1)), m(start(t2)), t1(lock),t1(unlock), t2(lock), t2(unlock), m(join(t1)), m(join(t2))` or `m(start(t1)), m(start(t2)), t2(lock),t2(unlock), t1(lock), t1(unlock), m(join(t1)), m(join(t2))`

This shows that multiple happens before relation arrise.
#### 7
We can guarantee that there no dataraces when we use locks because of what we wrote in exercise 6, that is in both the happens before relations either t1 releases the lock before t2 aqquires it or the opposite.
#### 8
Given defintion 5 in the reading(Pardo, The Java Memory Model (for PCPP).), we can conclude that there is no longer data race. This is due the fact that read and write access are only conflictng on non-volatile variables, since count is now a volatile variable there cannot be conflicting actions.

*Definition 5. Given an execution e ∈ E, we say that actions a, b ∈ e are conflicting iff they are accesses on the same (non-volatile) variable and at least one of them is a write access.*
### 3.2
```java
    public TestStringSet() throws InterruptedException {
        s = new StringSet(); // (Init(s))

        Thread t1 = new Thread(() -> {
                s.findOrAdd("PCPP");
        });
        Thread t2 = new Thread(() -> {
                s.find("PCPP");
        });

        t1.start(); // (start(t1))
        t2.start(); // (start(t2))
    }


    public class StringSet {

        private final List<String> list = new ArrayList<String>();

        public synchronized int findOrAdd(String s) {
            int ret = list.indexOf(s); //(1)
            if (ret == -1) {
                list.add(s); // (2)
            }
            return ret;
        }

        public synchronized int find(String s) {
            return list.indexOf(s); // (3)
        }
    }
```
#### 1
A program is correctly synchronised iff none of its executions contains data races
Therefore we need to argue that the program contains data races.
Given that (1) and (3) are read actions, (2) is a write action, and the find method is not synchronized, we cannot define a happens-before relation between (2) and (3). This creates a datarace because (2) and (3) are conflicting.
#### 2
Adding synchronized to the find method ensures that only one of the methods can run at a time. This creates several happens-before relatios, $t1(1) \to t1(2) \to t2(3)$ OR $t2(3) \to t1(1) \to t1(2)$. Since the conflicting actions are now defined by a happens-before relation there is no data race.
### 3.3
```java
    public TestVisibility() throws InterruptedException {
        x=0;
        Thread t1 = new Thread(() -> {
                while(x==0)                     // (3)
                    {/*Do nothing*/}
                System.out.println("t1: x="+x); // (4)
        });
        t1.start();                             // (start(t1))
        x=42;                                   // (1)
        System.out.println("Main: x="+x);       // (2)
    }
```
#### 1
The pair of actions that need to be orderd are $m(1) \to t1(3)$ This ensures that the write to x is visible to thread t1.
#### 2
Given x is non volatile variable we can show that there is not a happens before relation between m(1) and t1(3).