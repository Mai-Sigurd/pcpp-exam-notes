
# Exam questions for the examination of Practical Concurrent and Parallel Programming (PCPP) F2025

Below is a list of 12 questions for the exam. The exam will start with a question that you draw (at random) from the list below. As the exam proceeds, teachers and examiners may transition to any of the other questions, or ask questions about any of the topics in the syllabus. All questions have the same structure. First, you are supposed to start by defining, motivating and explaining the concepts introduced in a particular week of the course. Prepare for talking 7-10 minutes. In your presentation you may use the white/black-board in the exam room. After your presentation, you should show some samples of the code that you made for exercises related to concepts in the question (you may bring paper copies of this code). Teachers and examiners may also show code examples and ask you to explain and discuss these examples. While you are answering a question, teachers and examiners might ask about specific details of the question you are answering.

1. **Intro to concurrency and the mutual exclusion problem**: Define and motivate concurrency and mutual exclusion. Explain data races, race conditions, and critical sections. Show some examples of code from your solutions to the exercises in week 1.
	[01_concurrency_mutual_exclusion](01_concurrency_mutual_exclusion.md) 
	- [x] 

2. **Synchronization**: Explain and motivate how locks, monitors, and semaphores can be used to address the challenges caused by concurrent access to shared memory. Show some examples of code from your solutions to the exercises in week 2.
		[02_Synchronization](02_Synchronization.md)
	- [x] 

3. **Visibility**: Explain the problems of visibility and reordering in shared memory concurrency. Motivate and describe the use of volatile variables and locks to tackle these problems. Show some examples of code from your solutions to the exercises in week 2.
		[03_Visibility](03_Visibility.md)
	- [x] 

4. **Java memory model**: Motivate the need for the Java memory model. Explain the elements of the Java memory model including program order, happens-before order, synchronization order, and data races. Define what a correctly synchronized program is according to the Java memory model. Show some examples of code from your solutions to the exercises in week 3 and illustrate the use of the Java memory model to reason about their correctness.
		[04_java_mem_model](04_java_mem_model.md)
	- [x] 

5. **Thread-safe classes**: Define and explain what makes a class thread-safe. Explain the issues that may make classes not thread-safe. Show some examples of code from your solutions to the exercises in week 4.
	[05_thread_safe_classes](05_thread_safe_classes.md)
	- [x] 

6. **Testing**: Explain the challenges in ensuring the correctness of concurrent programs. Describe different testing strategies for concurrent programs, and their advantages and disadvantages. Show some examples of code from your solutions to the exercises in week 5.
	[[]]
	- [ ] 

7. **Performance measurements**: Motivate and explain how to measure the performance of Java code. Illustrate some of the pitfalls there are in doing such measurements. Show some examples of code from your solutions to the exercises in week 9.
	[[]]
	- [ ] 

8. **Performance and Scalability**: Explain how to increase the performance of Java code exploiting concurrency. Illustrate some of the pitfalls there are in doing this. Show some examples of code from your solutions to the exercises in week 10.
   	[[]]
	- [ ] 

9. **Lock-free Data Structures**: Define and motivate lock-free data structures. Explain how *compare-and-swap* (CAS) operations can be used to solve concurrency problems. Show some examples of code from your solutions to the exercises in week 6.
   	[[]]
	- [ ] 

10. **Linearizability**: Explain and motivate linearizability. Explain how linearizability can be applied to reason about the correctness of concurrent objects. Show some examples of code in your solutions to the exercises in week 7 where you used linearizability to reason about correctness.
    [[]]
	- [ ] 

11. **Streams**: Explain and motivate the use of streams to parallelize computation. Discuss issues that arise in operations executed by parallel streams. Show some examples of code from your solutions to the exercises in week 11.
    [[]]
	- [ ] 

12. **Message Passing**: Explain and motivate the actor model of concurrent computation. Discuss advantages and disadvantages of approaches to distribute computation in actor systems. Show some examples of code from your solutions to the exercises in week 12 and 13.
	[[]]
	- [ ] 