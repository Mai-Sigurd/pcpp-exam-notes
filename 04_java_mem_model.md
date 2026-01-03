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
**There are 3 types of actions:**
• Variable access
	• Read/write accesses on program variables
• Synchronization actions
	• Synchronization primitives such as acquiring and releasing a lock or monitor, starting a thread, waiting for a thread to terminate,…
• Other (any action not in the previous two categories)
### program order

+ Program order defines the intra-thread order of execution of the actions of a thread 

+ *Given two actions a, b that are executed by the same thread t, a occurs before b according to program order iff a would be performed before b when t is executed sequentially* 

+ Program order is a total order among the actions executed by a thread 
	+ Recall the definition of total order: https://en.wikipedia.org/wiki/Total_order 
	+ Intuitively, it means that each pair of actions must be related

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
![[Pasted image 20260103142002.png]]

## Show some examples of code from your solutions to the exercises in week 3 and illustrate the use of the Java memory model to reason about their correctness.
