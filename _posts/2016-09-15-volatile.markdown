---
title: "重排序问题"
data: "2016-09-15"
tags: [concurrency]
---

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。

如下所示的例子，第4，5行分别对两个在顺序上没有逻辑依赖关系变量的操作，处理器可能会将其重新排序，即可能先执行 `str="啦啦啦"` 后执行 `i = 100`。并且，我们可以看到，这种重排序对我们的程序结果没有任何影响。


	1.int i = 1;
	2.String str = "";
	3.public void atomic(){
	4.	i = 100;
	5.	str = "啦啦啦";
	6.}


**在处理器重排序问题上，有一个 `as-if-serial` 语义，即：不管怎么重排序（编译器和处理器为了提高并行度），程序的执行结果不能被改变。** 所以，为了遵守 `as-if-serial` 语义，编译器和处理器不会对存在 `数据依赖关系` 的操作做重排序。

---
<H4>数据依赖性</H4>
---

**如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在`数据依赖性`。**

细分下来，数据依赖可分为下面三种类型：
	
	1.写一个变量后，再读这个变量。  a=1; b=a;
	2.读一个变量后，在写这个变量。  t=a; a=1;
	3.写一个变量后，再次写这个变量。 a=1; a=2;


看一个具体的例子：
	
	double pi = 3.14;  //A
	double r = 1.0;    //B
	double area = pi * r * r;  //C

下面是这三个操作的数据依赖关系：

	A 与 B：没有数据依赖关系，不能保证其顺序。
	A 与 C: 先对pi写(A)，再读pi(C)，有数据依赖关系，A必然先于C执行。
	B 与 C: 同上，B先于C执行。

所以，可能存在的执行顺序有：
	
	1. A -> B -> C
	2. B -> A -> C

我们程序员不能对这两种情况做出确定的假设。

---
<H4>多线程下的重排序</H4>
---

通过上面的 `数据依赖性` 和 `as-if-serial` 原则，我们貌似发现 `重排序` 在用户程序的角度根本不会 “体现” 出来。是的，对于单线程程序，即针对特定的数据只有一条执行流，重排序根本不会对程序本身的正确运行有任何影响。

让我们来分析在多线程下的情况：
	
	1.public class NoVisibility {
	2.
	3.	private static boolean ready = false;
	4.	private static int number = 0;
    5.
	6.	public static void main(String[] args){
	7.		//读线程.
	8.		new Thread(() -> {
	9.			while(!ready){
	10.				Thread.yield();
	11.			}
	12.			System.out.println(number);
	13.		}).start();
	14.		//写线程.
	15.		new Thread(() -> {
	16.			number = 10;
	17.			ready = true;
	18.		}).start();
		}
	}
	

我们先分析第 16，17行的对 `number` 和 `ready` 的写操作，由于这两步操作没有任何数据依赖性，所以不能保证 `number=10` 和 `ready=true` 的执行顺序。现在，我们可以得到一个可能出现的执行情况：
	
	(write-thread: ready=true) -->  
	(read-thread: !read==false) --> 
	(read-thread: print:0)
	(write-thread: number=10;)

很明显，上面的这种执行顺序会导致程序出现不一致错误。在这里，我们说 `write-thread` 的写操作对 `read-thread`线程 **不可见**。

---
<H4>happens-before</H4>
---

在 `java` 中提供了一个叫做 `内存模型` 的概念抽象来描述并解决重排序对多个线程间 “通信” 造成的影响,即用来阐述操作之间的内存可见性。 

如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必然存在 `happens-before` 关系。这里提到的两个操作既可以是在一个线程之内，也可以是不同线程之间的操作。

下面是几个最重要的 happens-before 规则：
	
	1.程序顺序规则：一个线程中的每个操作，happens-before 于该线程中的任意后续操作。 
	
	2.监视器(monitor)锁规则：对一个监视器锁的解锁，happens-before 于随后对这个监视器锁的加锁。
	
	3.volatile变量规则：对一个volatile域的写，happens-before 于任意后续对这个volatile 域的读。
	
	4.传递性：如果 A happens-Before B, B happens-before C，则 A happens-before C。
	
	...

本文将只关注 `volatile` 变量的 happens-before 规则。

---
<H4>volatile</H4>
---

首先，`volatile` 是一个关键字，当我们将一个共享变量声明为 `volatile` 后，对这个变量的 读/写 将会很特别：

**1、当第二个操作为 volatile 写时，不管第一个操作是什么，都不能重排序。这个规则确保 volatile 写之前的任意操作不会被编译器重排序到 volatile写之后**

	例:
	#declaration
	volatile boolean ready = false; //ready is a volatile variable
	int number = 0;
	
	number = 10;
	ready = true;  #write a volatile variable, number=10 happens-before ready=true

**2、当第一个操作是 	`volatile` 读时不管第二个操作是什么，都不能重排序。这个规则保证了 `volatile` 读之后的操作不会被编译器重排序到 `volatile` 读之前。**

	1.new Thread(() -> {
	2.	while(!ready){  #read a volatile variable, so happens-before 3 and 5 line operation 
	3.		Thread.yield();
	4.	}
	5.	System.out.println(number);
	}).start();


**3、当第一个操作是 `volatile` 写，第二个操作是 `volatile` 读时，不能重排序。**


ok，有了这写规则，我们可以对上面那个多线程错误程序进行改正。只要将 `ready` 变量修饰为 volatile 即可。


总结：我们写的程序代码可能会被重新排序执行，当然，这种重排序在单个执行流上会保证运行结果与源程序相同。具体的，当出现 `数据依赖关系` 的操作不会被重排序。
但在多条执行流（多线程）的情况下，即使每个线程在重排序下保证了各自的 `as-if-serial`，程序的运行结果也可能发生不一致。所以，我们必须在源程序级别上提供一种限制重排序的手段。 `volatile variable`, `synchronized`，`final` 等都提供了对这种重排序控制规则，在 `java` 中，叫做 `内存模型`。本文主要关注了 `volatile variable` 的使用。




	








