# Performance and Scalability: 
## Explain how to increase the performance of Java code exploiting concurrency. 

avoid creating to many threads. This creates a lot of overhead and ends up taking more time as we saw in the exercises where we tested performance. 

### Executor
 we can instead submit the task to  javas **executor**.  which will either create a new thread or use an existing one, or wait till theres room to complete the task
replace:
`(new Thread(r)).start();` with `e.execute(r);`

 dependent on whether or not wee need a return value we can use `runnable` for no return value or If we want to return a result, we can use `Callable`

Executor is much faster and doing many simiarlor jobs compared to threads. 
### thread pool
Most of the executor implementations in `java.util.concurrent` use _thread pools_, which consist of _worker threads_. This kind of thread exists separately from the `Runnable` and `Callable` tasks it executes and is often used to execute multiple tasks.

Using worker threads minimizes the overhead due to thread creation. Thread objects use a significant amount of memory, and in a large-scale application, allocating and deallocating many thread objects creates a significant memory management overhead.
### Fork/Join
The first step for using the fork/join framework is to write code that performs a segment of the work. Your code should look similar to the following pseudocode:

```
if (my portion of the work is small enough)
  do the work directly
else
  split my work into two pieces
  invoke the two pieces and wait for the results
  ```


Wrap this code in a `ForkJoinTask` subclass, typically using one of its more specialized types, either [`RecursiveTask`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveTask.html) (which can return a result) or [`RecursiveAction`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveAction.html).

After your `ForkJoinTask` subclass is ready, create the object that represents all the work to be done and pass it to the `invoke()` method of a `ForkJoinPool` instance.

### Loss of scalability
**Starvation loss** (Amdahl) • 
+ Minimize the time that the task pool is empty 
**Separation loss** (best threshold) 
+ Find a good threshold to distribute workload evenly 
**Saturation loss** (locking common data structure) 
+ Minimize high thread contention in the problem 
**Braking loss** 
+ Stop all tasks as soon as the problem is 


### Amdahls law
#todo 
Amdahl’s law describes how much a program can theoretically be sped up by additional computing resources, based on the proportion of parallelizable and serial components. If F is the fraction of the calculation that must be executed serially, then Amdahl’s law says that on a machine with N processors, we can achieve a speedup of at most:
## Illustrate some of the pitfalls there are in doing this. 
p only accesses thread-safe classes ⇏ p is a thread-safe program

```java
public String getLast(ArrayList<String> l) {  /// getlast is not thread safe !!!
	int last= l.size()-1; 
	return l.get(last);  // get is thread safe
	
} 
public static void delete(ArrayList<String> l) { // delete is not thread safe
	int last= l.size()-1; 
	l.remove(last);      // remove is thread safe
}
```
so adding synchronized on l would ruin the idea that the arralist in it self was okay for concurrent use. 

### Reduce locking -> lock striping 
if we have a large data structure, and many things using it, locking it is an ideal solution.
Here we the notion of lock striping. in this idea we only lock part of the data strucute. Could be buckets of data structure. 

## Show some examples of code from your solutions to the exercises in week 10.


### 10.1.1

We get the following results when running Mark7 and we can conclude that they are proportional to num transactions, since everytime the number of transactions double, the runtime roughly doubles as well:

```
Do transactions 1              53349780,2 ns  421937,63          8
Do transactions 2             106881532,3 ns  872719,96          4
Do transactions 4             212142997,9 ns 2299577,16          2
Do transactions 8             426169800,0 ns 3273327,89          2
Do transactions 16            850812935,5 ns 4414272,83          2
Do transactions 32           1703622816,8 ns 7468675,52          2
```

```java
    for(int i = 1; i <= NO_TRANSACTION; i*=2){
      final int j = i;
      benchmarking.Benchmark.Mark7("Do transactions " + j, x -> {return doNTransactions(j);});
    }
```

### 10.1.2

This piece of code only influences the order in which the locking happens. By always locking the smallest id first, we ensure that we avoid deadlocking. A possible deadlock scenario can be seen here:

```
Transfer 1: A transfers to B
Transfer 2: B transfers to A

Deadlock:
  1) Account A is locked by Transfer 1
  2) Account B is locked by Transfer 2
  3) Transfer 1 tries to lock Account B and blocks to wait forever
  4) Transfer 2 tries to lock Account B and blocks to wait forever
```

When we run the code with min and max, the transactions look fine. However, when we do it without taking min and max, we reach a deadlock.

### 10.1.3

We have edited the following code in ThreadsAccountExperimentsMany:

```java
  public static void main(String[] args){
    new ThreadsAccountExperimentsMany();
    for( int i = 0; i < N; i++){
      accounts[i] = new Account(i);
    }

    final ExecutorService pool = new ForkJoinPool();
    final int range= 50;
    final int threshold= 5;

    Future f = pool.submit(new DoTransactionsTask(range, pool));

    try { f.get(); }
    catch (InterruptedException | ExecutionException e) { e.printStackTrace(); }

    try {
    // Wait for all tasks to complete or timeout after 100 milliseconds
    if (pool.awaitTermination(60, TimeUnit.SECONDS)) {
    //System.out.println("All tasks completed successfully.");
    } else {
    System.out.println("Timeout elapsed before termination.");
    }
    } catch (InterruptedException e) { e.printStackTrace(); }
  }

    public static class DoTransactionsTask implements Runnable{

    int numTransactions;
    private final ExecutorService pool;

    public DoTransactionsTask(int numTransactions, ExecutorService pool){
      this.numTransactions = numTransactions;
      this.pool = pool;
    }
    @Override public void run() {
      if(numTransactions <= 1)
      {
        doNTransactions(1);
        return;
      }

      Future f1 = pool.submit( new DoTransactionsTask(numTransactions/2, pool) );
      Future f2 = pool.submit( new DoTransactionsTask(numTransactions/2, pool) );
      try {
        f1.get();
        f2.get();
      } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
      }
    }
  }
```

### 10.1.4

We have ensured termination with the following code in the main method:

```
try {
  // Wait for all tasks to complete or timeout after 60 seconds
  if (pool.awaitTermination(60, TimeUnit.SECONDS)) {
  //System.out.println("All tasks completed successfully.");
  } else {
  System.out.println("Timeout elapsed before termination.");
  }
  } catch (InterruptedException e) { e.printStackTrace(); }
```

## 10.2

### 10.2.1

We use Amdahl's law and define F to be 1+1/2+1/4+1/8+(n/16)log2(n/16)=1.875+(n/16)log2(n/16), and N to be 16 when we have 16 cores. This gives us (n log(n))/(1,875+(n/16)log2(n/16)) which is approximately 21.07 when n = 100.000.

When we have 32 cores, we define F to be 1+1/2+1/4+1/8+1/16+(n/32)log2(n/32)=1.9375+(n/32)log2(n/32) and N to be 32. This gives us (n log(n))/(1,9375+(n/32)log2(n/32)) which is approximately 45.77 when n = 100.000.

### 10.2.2

1.000.000 numbers on 16 cores: max speed up 19.97 1.000.000 numbers on 32 cores: max speed up 42.71

10.000.000 numbers on 16 cores: max speed up 23.34 10.000.000 numbers on 32 cores: max speed up 49.83

## 10.3

### 10.3.1

```
> Task :app:run
countSequential                       2599614,2 ns   26827,38        128
countParallelN        1               2661288,7 ns   36512,72        128
countParallelNLocal   1               2668214,0 ns   28204,02        128
countParallelN        2               1708533,2 ns   29682,27        256
countParallelNLocal   2               1693893,1 ns    4858,11        256
countParallelN        4               1131091,8 ns   15753,48        256
countParallelNLocal   4               1025703,2 ns    8807,28        256
countParallelN        8                890381,3 ns    5907,17        512
countParallelNLocal   8                757857,1 ns    3079,74        512
countParallelN        16               937691,4 ns   19060,12        512
countParallelNLocal   16               844477,5 ns   21331,94        512
```

On the computer this was run on, we can see that it generally becomes faster up until 8 threads. With 16 threads we see a rise in time.

### 10.3.2

We have included the following class which implements callable as we with isPrime want to return an integer

```java
public TestCountPrimesThreads() {
	    final int range = 100_000;
	    Benchmark.Mark7("countSequential", i -> countSequential(range));
	    for (int c=1; c<=16; c=2*c) {
	      final int threadCount = c;
	      Benchmark.Mark7(String.format("countParallelN %2d", threadCount),
		      i -> countParallelN(range, threadCount));
	      Benchmark.Mark7(String.format("countParallelNLocal %2d", threadCount),
		      i -> countParallelNLocal(range, threadCount));
	    }
		final ExecutorService pool = new ForkJoinPool();
		for (int threshold = 100; threshold <= range; threshold = threshold *10){
			final int t = threshold;
			Benchmark.Mark7("executor with threshold" + t, i -> {

				try {
					Future<Integer> f = pool.submit(new task(0, range, pool, t));
					return f.get().intValue();
				} catch (InterruptedException | ExecutionException e) {
					e.printStackTrace();
				}

					return 0;
			});

		}
		pool.shutdown();
		try {
				// Wait for all tasks to complete or timeout after 60 seconds
			if (pool.awaitTermination(60, TimeUnit.SECONDS)) {
				//System.out.println("All tasks completed successfully.");
			} else {
				System.out.println("Timeout elapsed before termination.");
			}
		} catch (InterruptedException e) { e.printStackTrace(); }

	}
  ...
	public static class task implements Callable<Integer> {
		int low;
		int high;
		int threshold;
    	private final ExecutorService pool;

    	public task(int low, int high, ExecutorService pool, int threshold){
      		this.low = low;
			this.high = high;
      		this.pool = pool;
			this.threshold = threshold;
    	}

		@Override public Integer call() {
			int lc= 0; // Counting primes
			if ((high-low) < threshold) {
				for (int i=low; i<=high; i++)
					if (isPrime(i)) lc= lc+1;
			} else {
				int mid= low+(high-low)/2;
				Future<Integer> f1= pool.submit( new task(low, mid, pool, threshold) );
				Future<Integer> f2= pool.submit( new task(mid+1, high, pool, threshold) );

				try {
					lc= f1.get().intValue() + f2.get().intValue();
				}
				catch (InterruptedException | ExecutionException e){ }
			}
			return lc;
		}

	}
```

We get the following result:

```
countSequential                   2626821,9 ns   13877,88        128
countParallelN  1                 2788744,5 ns   89124,20        128
countParallelNLocal  1            2702511,8 ns   15992,16        128
countParallelN  2                 1815971,7 ns   76290,25        256
countParallelNLocal  2            1806404,6 ns   44492,73        256
countParallelN  4                 1167609,6 ns   46412,50        256
countParallelNLocal  4            1085871,8 ns   51420,56        256
countParallelN  8                  950592,7 ns   42686,11        512
countParallelNLocal  8             795868,5 ns   31463,07        512
countParallelN 16                  958434,5 ns   24514,03        512
countParallelNLocal 16             842225,0 ns    3811,80        512
executor with threshold100         507433,1 ns    5911,27        512
executor with threshold1000        490456,6 ns    8611,86        512
executor with threshold10000       545514,8 ns    5169,20        512
executor with threshold100000     1678792,3 ns    3323,65        256
```

We see that the executor runs this faster than the the thread method, and that the threshold with the lowest time is at 1000.

## 10.4

### 10.4.1

What implementation performs better? The monitor implementation or the CAS-based one?
Is the result you got expected? Explain why.

The cas performs better. This is what we expected.

Why?: Lock histogram locks the entire datastructure on each operation which scales poorly. The cas histogram uses striping, meaning synchronization happens on a per bucket level. Which significantly reduces contention.

```
Lock Histogram  1               7878806,3 ns  337271,97         64
Lock Histogram  2              11086010,3 ns  517136,58         32
Lock Histogram  3              10583117,6 ns  578372,20         32
Lock Histogram  4              10835320,4 ns  288215,23         32
Lock Histogram  5              10857784,6 ns  468071,84         32
Lock Histogram  6              11042991,0 ns  977697,05         32
Lock Histogram  7              11143314,6 ns  123017,35         32
Lock Histogram  8              10901767,1 ns  245487,27         32
Lock Histogram  9              11156546,5 ns  369966,28         32
Lock Histogram 10              10971059,4 ns  243728,27         32
Lock Histogram 11              10795241,8 ns  158085,37         32
Lock Histogram 12              11740945,2 ns  649953,73         32
Lock Histogram 13              10807782,3 ns  188161,75         32
Lock Histogram 14              11373144,5 ns  478105,51         32
Lock Histogram 15              11232178,9 ns  376867,62         32
Cas Histogram  1                7639878,8 ns   51836,87         64
Cas Histogram  2                5441390,6 ns   82302,05         64
Cas Histogram  3                4998125,3 ns  216558,45         64
Cas Histogram  4                5770906,1 ns  312328,63         64
Cas Histogram  5                6339272,2 ns   64701,14         64
Cas Histogram  6                6634382,1 ns  305026,54         64
Cas Histogram  7                7233950,9 ns   48258,74         64
Cas Histogram  8                8321559,2 ns  139244,12         32
Cas Histogram  9                9151761,1 ns  214051,94         32
Cas Histogram 10                9372404,7 ns  248264,50         32
Cas Histogram 11                9157703,9 ns  773873,90         32
Cas Histogram 12                9380073,8 ns  359215,05         32
Cas Histogram 13                9211268,1 ns  275766,66         32
Cas Histogram 14                9178345,4 ns  171947,31         32
Cas Histogram 15                9043528,7 ns  191441,14         32
```

```java
    int noThreads= 16;
    int range= 100_000;

    for (int i= 1; i < noThreads; i++) {
      int threadCount= i;
      // Benchmark.Mark7( ... using a monitor
      Benchmark.Mark7(String.format("Lock Histogram %2d", threadCount), j -> {
        HistogramLocks histogramLock = new HistogramLocks(100);
        countParallel(range, threadCount, histogramLock);
        int count = histogramLock.getCount(1);
        return count;
      });
    }

    for (int i= 1; i < noThreads; i++) {
      int threadCount= i;
      // Benchmark.Mark7( ... using CAS
      Benchmark.Mark7(String.format("Cas Histogram %2d", threadCount), j -> {
        CasHistogram histogramCAS = new CasHistogram(100);
        countParallel(range, threadCount, histogramCAS);
        int count = histogramCAS.getCount(1);
        return count;
      });
    }
  }
```