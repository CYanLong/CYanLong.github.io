---
title: "Race Condition"
data: "2016-09-14"
tags: ["concurrency"]
---

首先： 线程是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

同一进程中的多条线程将共享该进程中的全部系统资源，如 `虚拟地址空间`，`文件描述符` 和 `信号处理` 等。当同一进程中的多个线程有各自的 `调用栈（call stack）`，自己的 `寄存器环境（register context）`，自己的线程本地存储。

---
<H4>一：状态（State）</H4>
---

由于一个进程中的线程会共享进程空间的 `数据段`，`堆` 等 数据存储内存，并且多个线程的执行具有随机性。所以，对于这些存储在 `共享内存` 的数据的操作就显得格外重要，稍有不慎，就会出现 `数据不一致` 的情况，也就是线程安全性问题。

特别的，在OOP中，一个对象会有 `成员变量和类变量`，我们也叫这些变量为对象的 `状态`。在这里，线程安全性即对这些 `共享的(shared)` 和 `可变的(Mutable)` 的状态的管理。

---
<H4>二：竞态条件(Race Condition)问题</H4>
---

最常见的一种线程安全出现在对 `共享可变` 状态的 `Check-Then-Act`（先检查再执行）操作。出现问题的原因是基于一种可能已经过时的结果来执行一些操作。

例一：
下面这段代码演示了线程安全性问题导致的对单例模式的破坏。

	1.public class RaceCondition1{
	2.	
	3.	private static RaceCondition1 con = null;
	4.	
	5.	public static RaceCondition1 getInstance(){
	6.		if(conn == null){  //Check-Then-Act 
	7.			System.out.println("new RaceCondition1");
				con = new RaceCondition1();
			}
			return con;
		}

		public static void main(String[] args){
			for(int i = 0; i < 1000; i++){
				new Thread(() -> {
					RaceCondition1.getInstance();
				}).start();
			}
		}
	}


1000个线程几乎同时并随机的执行第6行开始的 Check-Then-Act，一种情况：`thread1` 执行完 `if(conn == null)` 进入代码块之后执行 `new` 语句之前，`thread2` 也可以进入 `if` 代码块并执行完，此时 `thread1` 开始执行，在一次执行了 new 语句。这样，单例模式就被破坏了。很显然，问题出在 `thread1` 的 `Check`（检查）已经过时了。

**一个线程对某个 `状态` 执行 `check-then-act` 过程中，另一个线程在它 `Check` 和 `Act` 之间改变了这个状态，导致它的 `Check` 结果失效并最终使得 `Act` 的执行出现了不一致问题。 ** 我们把它叫做 `Race Condition`。

例二： Read-Modify-Write

一种隐蔽的，不易发现的Race Condition 问题。

	1.public class RaceCondition2{
	2.	private int count = 0;
	3.	
	4.	public void increase(){
	5.		count++;
		}

		public static void main(String[] args){
			RaceCondition2 r = new RaceCondition2();

			for(int i = 0; i < 100000; i++){
				Thread t = new Thread(
					for(int j = 0; j < 20; j++){
						r.increase();
					}
				);
				t.start();
			}
		}
	}

这个看起开有些复杂的程序，真正对 `共享可变状态` 操作的只出现在第5行 `count++`。 然而，一句简单的  `count++` 涉及的字节码指令如下：

	 0: aload_0
     1: dup
     2: getfield      #12                 // Field count:I
     5: iconst_1
     6: iadd
     7: putfield      #12                 // Field count:I
     10: return

我们看到，2-7行是执行对 `count` 加1的操作。至少，它又分为：读(getfield)，修改（iadd），写回（putfield）。很明显，我们又看到了 `Race Condition`。
这里，Check 变成了读取，这里的 Act（加一） 依赖于 Check（读取）的结果。当一个线程在另一个线程的 `getfield` 和 `putfield` 之间执行了一次 count++，就会导致 getfield 得到的值 “失效”，最终会导致出现看起来少加的情况。上面的程序得到的结果小于 2000000。


---
<H4>三:同步</H4>
---

解决这类 `Race Condition` 问题的方法是使 `Check-Then-Act` 的执行变成**原子的**。即一个线程在 `Check` 和 `Act` 之间不会有别的线程 "干扰" 它。在线程中，我们使用 `同步` 来实现对多个顺序操作的原子访问。

java 中提供关键字 synchronized 来实现同步。
	
	#例一的Solve
	private static Object o = new Object();//充当锁.
	private static RaceCondition1Solve rc = null;
	
	public static RaceCondition1Solve getInstance(){
		if(rc != null)
			return rc;
		synchronized(o){//对象为空时,不能用作内置锁.
			if(rc == null){
				System.out.println("new object");
				rc = new RaceCondition1Solve();
			}
		}
		return rc;
	}

	
对于例2这种原子性的自增减，我们有更好的选择，即使用原子类(AtomicXxx)。
	
	private AtomicInteger count = new AtomicInteger(0);
	
	public void increase(){
		count.incrementAndGet();
	}


总结：由于同一个进程的多个线程会共享进程的 数据区(.data)，堆内存。所以，在编写多线程程序时由于线程调度算法对于程序员的随机性导致对存放在这些共享内存区的数据的管理尤为重要。常见的一种线程安全性问题是 Race Condition，由于 `Check-Then-Act` 可能会被互相干扰，从而导致 `Check` 的结果已经失效，进而依赖与 `Check` 的结果的 `Act` 使共享可变的状态出现不一致。解决办法是将`Check-Then-Act` 的复合操作通过同步变成一个原子(Atomic) 操作。
 

