# Thread-safe classes
*A (concurrent) program is correct if and only if it satisfies its specification*
## Define and explain what makes a class thread-safe. 
**Thread safe class:**
	A class is said to be thread-safe if and only if no concurrent   execution of method calls or field accesses (read/write) result in data races on the fields of the class
	
> [! What is the specification in this definition?]
> "no concurrent execution of method calls or field accesses (read/write) result in data races on the fields of the class"
> = There is no data race in any of the class fields

**To analyse whether a class is thread-safe**, we must simply ensure that for any concurrent execution of field access and methods calls—where at least one write access is executed— the actions are related by the happens-before relation 

To analyse thread-safe in a class, we must identify/consider: 
- Identify the *class state* 
- Make sure that class state does not *escape* 
- Ensure *safe publication* 
- Whenever possible define class state as *immutable (Immutability)* 
- If class state must be mutable, ensure *mutual exclusion*

*When asked to reason about the thread-safety of a class, you must always cover these elements ^^^*
## Explain the issues that may make classes not thread-safe. 

### class state 

Methods should only manipulate class state or parameters 
+ For instance, avoid the use of variables from parent classes 

Methods should avoid using object references as parameters 
+ We cannot guarantee happens- before order on actions involving the referenced object 

That said, our definition of thread- safe class focuses on data races on the fields of the class
+ Therefore, these problems do not violate the definition
### Escaping
It is important to not expose class fields 

Otherwise, threads may use them without ensuring proper synchronization 
+ Thus, we cannot enforce that the operations are related by happens-before

Defining all (primitive) class fields as private ensures that these variables will only be accessed through public methods. 
+ Thus, it is easier to control and reason about concurrent access

```java
class Counter { 
	// class state (variables) 
	int i=0; 
	// class methods 
	public synchronized void inc(){i++;}
}

Counter c = new Counter(); 
new Thread(() -> { 
	c.inc(); 
}).start(); 

new Thread(() -> { 
	c.i++; // escaped the lock in inc() 
}).start();
```
![[Pasted image 20250915085128.png]]
> [! Can list `a` escape in `IntArraylist`]
> The get method returns a reference to a, so whoever recieves it can change it without locks
> Problem is showed below

![[Pasted image 20250915085340.png]]
Remember that when a method returns an object, we get a reference to that object 
Therefore, even if we obtain the reference using locks, later we can modify the content of the object without locks 
To prevent escaping, avoid methods that return references to objects in the class state
### Safe publication
Safe publication requires that **initialization happens-before publication**

*Visibility issues* may appear *during initialization* of objects
```java
public class UnsafeInitialization { 
	private int x; // add voliatile to fix it
	private Object o;  // add final to fix it
	public UnsafeInitialization() { 
		x = 42; 
		o = new Object(); 
	}
}
```
For the thread executing the constructor, there are no visibility issues, but if a reference to an instance of UnsafeInitialization object is accessible to another thread, it might not see `x==42` or `o` completely initialized

For primitive types, we can: 
- Declare them as volatile 
- Declare them as final (only works if the content is never modified) 
- Initialize them as the default value: 0. (only works if the default value is acceptable) 
- Declare them as static (only works if the field must be static in the class) 
- Use corresponding atomic class from Java standard library: AtomicInteger

For complex objects, we can: 
- Declare them as final 
- Initialize them as the default value: null. (only works if the default value is acceptable)
- Declare them as static (only works if the field must be static in the class) 
- Use the AtomicReference class

### Immutability
An immutable object is one whose state does not change after initialization 
- You can think of it as a constant 
- The final keyword in Java prevents modification of fields 
	- Remember that variables assigned to an object only hold a reference to the object 

Since immutable objects do not change the state after initialization, data races can only occur during initialization 

An immutable class is one whose instances are immutable objects

**To ensure thread-safety of immutable classes**, we must ensure that: 
- No fields can be modified after publication 
- Objects are safely published 
- Access to the object’s state does not escape

> [!Why is this class thread-safe? (tip: there are 3 main reasons)]
> 1. final in the field 
> 2. method is not retuning the reference , therefore prevents escaping
> 3. final class, ensures that the public initiliation becomes done first
> ![[Pasted image 20250915092001.png]]
> since i cannot write any where else there can be no conflicting actions
> (also not having to aqquire the lock will improve program speed)

### Mutual exclusion
Whenever shared mutable state is accessed by several threads, we must ensure mutual 

## Show some examples of code from your solutions to the exercises in week 4.

