# Performance measurements: 
## Motivate and explain how to measure the performance of Java code.
one of the motivations for concurrency is the :
**Exploitation**: Hardware capable of simultaneously executing multiple streams of statements

But just creating threads for every small task is not gonna end up going well. 
Thread creation is "expensive" (time consuming) !!

still when working with large amounts of data, running computations sequentially is not the way to go. 

so performance measurements help us find a balance in how to tackle thread creation. 

## Illustrate some of the pitfalls there are in doing such measurements. 

### JIT!!
If youre forexample testing how long ti takes to create htreads, or do simple tasks, then you might have to work around the JIT.
Just in time compiler, will optimize code, and if it sees compution not used it will remove it. 
We had examples of that in our exercises, where we had to come up with some "difficult " math that would ensure the code would actually be run and we could gain meaningful insigths.

### but also what else is running your computer.
Some pitfalls include not being aware of how your computer affect the numbers you are seeing. 
Like running a lot of other stuff, like IDEs or programs whille you measure will weigh on the performance,

another pitfall is simply using a timer to measure a couple of times. or using the mean of those, it will lead to inaccurate readings 

so its good to use a normal distribtuin and thereby look at the standard deviation along side the mean

in order to get a good read you also need a certain amount of numbers to make a mean out off. For some programs you can generate plenty if they are quick but for some, getting one mark might take some time. In this course we have been given a measure ment tool that tries to get as much data as possible within a reasnable time. In mark 5 the amount of times run is doubles until it takes 25 seconds to complete the tests. 

with fewer iterations you see a higher standard deviation which offcourse lessens the learned value of the mean. 

## Show some examples of code from your solutions to the exercises in week 9.

#todo something about how in the exercises we saw that more threads does not necesailiry equals better. 
### 9.1.1

- `Point creation` has outliers where the std. dev. is significantly larger. Fx 137ns +/- 129ns. We assume that some GC activity is causing this.
- `Thread's work` is inconsistent in std. dev. as the number of iterations increase, becoming larger and smaller again.
- `Thread create` has some outliers as well, but less significant. It also reaches std. dev. of <1 at +1M iterations, which is good.
- `Thread create start join` has inconsistent std. dev. which might indicate that starting and joining threads is inconsistent.

The other measurements seem to (mainly) decrease in std. dev. as the number of iterations increase.

### 9.1.2

#### System information

All code has been benchmarked on the following system:

- OS: macOS 15.5 24F74 arm64
- Host: MacBookPro18,3
- Kernel: 24.5.0
- CPU: Apple M1 Pro
- GPU: Apple M1 Pro
- Memory: 16384MiB

The output is as follows:

```txt
$ gradle -PmainClass=exercises09.TestTimeThreads run
# OS:   Mac OS X; 15.5; aarch64
# JVM:  Homebrew; 24.0.2
# CPU:  null; 8 "cores"
# Date: 2025-10-27T10:33:35+0100
Mark 7 measurements
Point creation                       56.9 ns       0.18    8388608
Thread's work                      6985.2 ns      20.17      65536
Thread create                        86.0 ns       0.28    4194304
Thread create start               48931.3 ns    3074.25       8192
Thread create start join          53790.4 ns    4715.92       8192
ai value = 1638340000
Uncontended lock                      9.1 ns       1.00   33554432
```

- `Point creation` is very consistent with low std. dev. This makes sense, since it just creates a single object.
  - Research online shows that allocating small objects is very fast in Java
- `Thread's work` also has very low std. dev., which is expected since it performs the same straight-forward computation repeatedly, without heavy syscalls or similar.
- `Thread create` is also very consistent with low std.dev. It has slightly higher latency than allocating a simple object, but we assume that the constructor has to do a little more work than set two fields.
  - We don't start the thread at all, so since we only create the object, it is still very fast.
- `Thread create start` is significantly slower than just creating the thread, since we now have to start it as well.
  - Starting a thread is very expensive, as outlined in the lecture, so this makes sense.
  - The std. dev. is much higher than the previous measurements (also relatively speaking), but this is expected as the number of iterations is smaller, and the work from the OS is more complicated.
- `Thread create start join` is a little slower than just starting the thread, since we also wait for the thread to finish the work.
  - This is expected, but it is only slightly slower than just starting the thread.
  - Indicates that starting the thread is by far the most expensive part of the process, not the actual work being done.

### 9.1.3

Using the results from 9.1.2, we can estimate the time taken for each operation:

- Time to create a thread object is ~86 ns
- Time to *start* a thread is ~49,000 ns, much more expensive

## Exercise 9.2

### 9.2.1

We iterated a bit on the benchmark due to surprising results.

#### Initial results

We used a benchmark as follows:

```java
Benchmark.Mark7("volatileCounter", i -> {
  TestVolatile tv1 = new TestVolatile();
  for (int j = 0; j < 1_000_000; j++) {
    tv1.vInc();
  }
  return tv1.vCtr;
});

Benchmark.Mark7("standardCounter", i -> {
  TestVolatile tv2 = new TestVolatile();
  for (int j = 0; j < 1_000_000; j++) {
    tv2.inc();
  }
  return tv2.ctr;
});
```

Results:

```
volatileCounter                       3.5 ns       0.07  134217728
standardCounter                       3.7 ns       0.40  134217728
```

This would indicate that incrementing 1 million times takes 3.5ns on average. This is not plausible, as a single addition of two numbers should take a few nanoseconds itself. So something is off.

We suspect that the compiler or the JIT compiler somehow optimizes away the loop.

#### Second attempt

To attempt to fix the above, we move the creation of the counter objects to outside the lambda functions.
This way, we don't benchmark the creation of the object either, but rather only the increments.

```java
TestVolatile tv1 = new TestVolatile();
Benchmark.Mark7("volatileCounter", i -> {
  for (int j = 0; j < 1_000_000; j++) {
    tv1.vInc();
  }
  return tv1.vCtr;
});

TestVolatile tv2 = new TestVolatile();
Benchmark.Mark7("standardCounter", i -> {
  for (int j = 0; j < 1_000_000; j++) {
    tv2.inc();
  }
  return tv2.ctr;
});
```

We now get the following results:

```txt
volatileCounter                 5997492.5 ns   18402.08         64
standardCounter                       0.9 ns       0.01  268435456
```

The `volatileCounter` results now seem more plausible (very slow, but it does make 1M increments per iteration).

However, the `standardCounter` results are now even more implausible, as it indicates that 1M increments take less than 1ns! This makes zero sense, so there must still be an optimization going on.

We hypothesize that since the standard `ctr` field is not volatile, the compiler can guarantee that (visible) increments only happen on the current thread, so it may be able to optimize away the loop completely. In the volatile case, the compiler cannot make this assumption, so it has to actually perform the increments.

We assume that the time is faster because object allocation is not made inside the benchmark this time.

### Third attempt

To avoid the optimization, we add a "dummy" conditional check that uses the fields during the iteration.

```java
TestVolatile tv1 = new TestVolatile();
Benchmark.Mark7("volatileCounter", i -> {
  for (int j = 0; j < 1_000_000; j++) {
    if (tv1.vCtr > 0) tv1.vInc();
    tv1.vInc();
  }
  return tv1.vCtr;
});

TestVolatile tv2 = new TestVolatile();
Benchmark.Mark7("standardCounter", i -> {
  for (int j = 0; j < 1_000_000; j++) {
    if (tv2.ctr > 0) tv2.inc();
    tv2.inc();
  }
  return tv2.ctr;
});
```

Both benchmarks now also include the work of evaluating an if-statement, but since both of them do it, it should correspond to the same amount of work.

Now, the results look like:

```txt
volatileCounter                11953364.5 ns   46658.68         32
standardCounter                 1604014.4 ns 1338670.13        512
```

This seems way more plausible - clearly both experiments are performing actual work now. However, the number of iterations is quite low, so we have large std. dev.

### Final attempt

We decrease the number of iterations per benchmark run to 1000.

```java
TestVolatile tv1 = new TestVolatile();
Benchmark.Mark7("volatileCounter", i -> {
  for (int j = 0; j < 1_000; j++) {
    if (tv1.vCtr > 0) tv1.vInc();
    tv1.vInc();
  }
  return tv1.vCtr;
});

TestVolatile tv2 = new TestVolatile();
Benchmark.Mark7("standardCounter", i -> {
  for (int j = 0; j < 1_000; j++) {
    if (tv2.ctr > 0) tv2.inc();
    tv2.inc();
  }
  return tv2.ctr;
});
```

Now, we get the following results:

```txt
volatileCounter                   11971.1 ns      77.38      32768
standardCounter                    1669.6 ns    1288.81     524288
```

There is still work being done, and it seems that using a volatile variable is significantly slower (~7x).

However, it is very surprising that `standardCounter` has a std. dev. that is almost as large as the mean. This indicates that the benchmark is not very stable, and we would ideally want to investigate this further.

The std. dev. for `volatileCounter` is very small, so the measurements seem consistent.

## Exercise 9.3

### 9.3.1

Our results are as follows:

```txt
# OS:   Mac OS X; 15.5; aarch64
# JVM:  Homebrew; 24.0.2
# CPU:  null; 8 "cores"
# Date: 2025-10-27T11:58:55+0100
countSequential                 2611495.8 ns   12837.44        128
countParallelN       1          2680530.3 ns   16288.58        128
countParallelN       2          1910864.1 ns   79431.15        256
countParallelN       3          1534797.4 ns   63145.81        256
countParallelN       4          1764249.4 ns   31134.91        256
countParallelN       5          1898573.8 ns  111546.48        128
countParallelN       6          1891708.0 ns   36249.28        256
countParallelN       7          2179723.5 ns  264345.07        128
countParallelN       8          2080901.1 ns  212157.02        128
countParallelN       9          1996098.9 ns   47607.00        128
countParallelN      10          2029653.8 ns   62573.11        128
countParallelN      11          2197482.6 ns  258046.11        128
countParallelN      12          2157657.9 ns   56483.36        128
countParallelN      13          2062227.9 ns   25366.59        128
countParallelN      14          2092729.0 ns   50434.07        128
countParallelN      15          2104429.3 ns   41568.53        128
countParallelN      16          2117235.9 ns   18954.85        128
countParallelN      17          2140142.2 ns   38182.26        128
countParallelN      18          2258940.8 ns   44536.89        128
countParallelN      19          2176673.1 ns   32313.68        128
countParallelN      20          2271714.7 ns  242726.99        128
countParallelN      21          2222853.4 ns   30199.08        128
countParallelN      22          2208797.1 ns   30371.28        128
countParallelN      23          2266269.4 ns   37239.08        128
countParallelN      24          2264840.2 ns   26812.21        128
countParallelN      25          2266626.7 ns   33469.19        128
countParallelN      26          2360803.3 ns   43378.22        128
countParallelN      27          2323983.3 ns   43242.73        128
countParallelN      28          2359980.9 ns   45903.94        128
countParallelN      29          2376663.1 ns   36652.16        128
countParallelN      30          2371077.5 ns   26350.89        128
countParallelN      31          2421625.4 ns   35937.71        128
countParallelN      32          2413418.9 ns   50264.83        128
```

### 9.3.2

- We observe that using 3 threads is the fastest, at ~1.53 ms.
- The computer being benchmarked has 8 cores, so one could expect that 8 threads would perform optimally. But it doesn't.
  - There might be too much overhead in creating threads for a benefit to be seen.
  - Earlier, we measured that it takes ~49,000 ns to start a thread. In the worst case here, 32 threads should take 32 * ~49,000 ns = ~1,568,000 ns = 1.5 ms. The mean measurement time for 32 threads is ~2.41 ms, so starting the threads takes up a significant portion of the total time compared to the actual work being done.
- The division of work between threads seems to be quite unfair
  - The work is divided by simply splitting the range into contiguous chunks, but some chunks might have more prime numbers than others, leading to some threads doing more work.
  - Thus, more threads may not actually take on more work, but rather just add overhead.
  - A better approach might have been to distribute the work more fairly, for example by using a shared counter that each thread increments to get the next number to check - or something similar.

## Exercise 9.4

### 9.4.1

```java
import java.util.concurrent.atomic.AtomicInteger;

class PrimeCounter {
  private AtomicInteger count = new AtomicInteger(0);

  public void increment() {
    count.incrementAndGet();
  }

  public int get() { 
    return count.get();
  }

  public void add(int c) {
    count.addAndGet(c);
  }

  public void reset() {
    count.set(0);
  }
}
```

### 9.4.2

We get:

```
Array Size: 5697
# Occurences of ipsum :1430
```

### 9.4.3

We add a very simple benchmark, but keep the previous print statements to prepare the data in memory and in caches before the timed benchmarks.

```java
// Keep this to visit the data and perhaps populate cpu caches
System.out.println("Array Size: "+ lineArray.length);
System.out.println("# Occurences of "+target+ " :"+search(target, lineArray, 0, lineArray.length, lc));

Benchmark.Mark7("searchThatSheesh", i -> {
  lc.reset();
  search(target, lineArray, 0, lineArray.length, lc);
  return lc.get();
});
```

Results:

```txt
Array Size: 5697
# Occurences of ipsum :1430
searchThatSheesh                6311082.6 ns   32901.11         64
```

### 9.4.4

We implement it as the below. We essentially just split the lineArray into `N` even-sized parts with `N` threads.

```java
private static long countParallelN(String target, String[] lineArray, int N, PrimeCounter lc) {
  final int perThread = lineArray.length / N;
  Thread[] threads= new Thread[N];
  for (int t= 0; t<N; t++) {
      final int from = perThread * t, 
      to = (t+1==N) ? lineArray.length : perThread * (t+1); 
      threads[t]= new Thread( () -> {
        search(target, lineArray, from, to, lc);
      });
  }
  for (int t= 0; t<N; t++) 
    threads[t].start();
  try {
    for (int t=0; t<N; t++) 
      threads[t].join();
  } catch (InterruptedException exn) { }
  return lc.get();
}
```

We try it out with a few number of threads to see that it works:

```txt
Array Size: 5697
# Occurences of ipsum :1430
# Occurences of ipsum (4 threads):1430
# Occurences of ipsum (8 threads):1430
# Occurences of ipsum (16 threads):1430
```

It works!

### 9.4.5

We run the above with varying number of threads:

```java
    for (int c= 1; c<=32; c++) {
      var numThreads = c;
      Benchmark.Mark7(String.format("searchThatSheeshParallel %7d", numThreads), i -> {
        lc.reset();
        return countParallelN(target, lineArray, numThreads, lc);
      });
    }
```

Results:

```txt
Array Size: 5697
# Occurences of ipsum :1430
searchThatSheesh                6466649.0 ns   30515.70         64
searchThatSheeshParallel       1       6626604.8 ns  112212.51         64
searchThatSheeshParallel       2       3536105.2 ns   25408.26        128
searchThatSheeshParallel       3       2467733.2 ns   11288.45        128
searchThatSheeshParallel       4       1996976.7 ns   59847.71        128
searchThatSheeshParallel       5       1677745.7 ns   20409.34        256
searchThatSheeshParallel       6       1697761.0 ns  178610.89        256
searchThatSheeshParallel       7       1914339.5 ns   14364.12        256
searchThatSheeshParallel       8       1834851.1 ns   84381.92        256
searchThatSheeshParallel       9       2016801.4 ns  169346.44        256
searchThatSheeshParallel      10       1776063.8 ns   48083.52        256
searchThatSheeshParallel      11       1812312.5 ns   65937.96        256
searchThatSheeshParallel      12       1843036.9 ns   84240.87        256
searchThatSheeshParallel      13       1823826.2 ns   38181.03        256
searchThatSheeshParallel      14       1767833.3 ns   41226.26        256
searchThatSheeshParallel      15       1759590.6 ns   45046.16        256
searchThatSheeshParallel      16       1880172.0 ns  126314.22        128
searchThatSheeshParallel      17       1860613.5 ns   55914.77        256
searchThatSheeshParallel      18       1909299.7 ns  128178.51        128
searchThatSheeshParallel      19       1901459.4 ns   72111.57        256
searchThatSheeshParallel      20       1973835.3 ns   97535.89        256
searchThatSheeshParallel      21       1943991.7 ns  187945.73        128
searchThatSheeshParallel      22       1843578.8 ns   78786.06        256
searchThatSheeshParallel      23       1797758.5 ns   51105.99        256
searchThatSheeshParallel      24       1926443.5 ns  102967.68        256
searchThatSheeshParallel      25       2091533.7 ns  139075.85        128
searchThatSheeshParallel      26       1998466.5 ns   75736.63        256
searchThatSheeshParallel      27       2026846.5 ns   99728.66        256
searchThatSheeshParallel      28       2152286.2 ns   64248.87        256
searchThatSheeshParallel      29       2265299.1 ns  263553.58        128
searchThatSheeshParallel      30       2168277.1 ns  218430.25        128
searchThatSheeshParallel      31       2138778.7 ns   73348.29        128
searchThatSheeshParallel      32       2030661.0 ns   49200.17        128
```

We observe that it is fastest using 5 threads, at ~1.68 ms. It seems that with 5 threads, we obtain a good balance between work done per thread and the cost of starting threads. Above 5, the performance degrades and becomes worse.

However, using 5 threads is ~5x faster than the sequential version, so there is a benefit for sure.

We assume that it is, again, due the cost of starting threads and synchronization overhead.