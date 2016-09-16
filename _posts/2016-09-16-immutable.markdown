---
title: "不可变类"
data: "2016-09-16"
tags: ["concurrency"]
---

我们知道当多个线程对 **共享可变** 的 `状态` 进行不确定顺序的非原子操作时，那些在逻辑上为原子的非原子操作可能会导致数据出现不一致，即线程安全性问题。

解决线程安全性问题的手段有很多，同步是最常见的一种。但在一些情况下，使用 `不可变类` 来实现线程安全性更为合适，也能带来更好的程序性能。

---
<H4>Immutable</H4>
---

一个对象创建(constructed) 之后其 **状态(state)** 就不能改变，我们称其为不可变的。下面是 Java 规范中关于创建一个不可变类的规则：

1、Don't provide "setter" methods。

2、Make all fields **final** and **private**

3、Don't allow subclass to override methods. This simplest way to do this is to declare the class as final.A more sophisticate(复杂的) approach is to make the constructor private and construct instances in factory methods.

//**这一条非常重要，经常被忽视**  <br/>
4、If the instance fields include references to mutable objects, don't allow those objects to be changed: Don't provide methods that modify the mutable objects. **Don't share reference to the mutable objects.Never store references to external, mutable objects passed to the constructor; if necessary, create copies, and store references to the copies. Similarly, create copies of your internal mutable objects when necessary to avoid returning the originals in your methods.**

ok，我们来具体看一个例子，一个 `PowerServer` 类提供 `int [] get Power(int num)` 方法计算给定数(num)的平方和立方，并且作为 `int[]` 返回。并且，在 `PowerServer` 内部需要缓存上一次的计算结果，这样，对连续相同的值不必每次都计算。 我们用 `OneValueCache` 类来封装缓存数据。 

通常情况下，我们可能按如下方式写：
	
	1.public class PowerServer{
	2.	
	3.	private MutableOneValueCache o = new MutableOneValueCache();
	4.	
	5.	public int[] getPower(int num){
	6.		int[] value = o.getCache(num);
	7.		if(value == null){
	8.			value = new int[]{num*num, num*num*num};
	9.			o.resetCache(num, value);
	10.		}
	11.		return value;
	12.	}
	13.}
	14.
	15.class MutableOneValueCache{
	16.	private int lastNumber;
	17.	private int[] numPower;
	18.	
	19.	public void resetCache(int lastNumber, int[] numPower){	
	20.		this.lastNumber = lastNumber;
	21.		this.numPower = numPower
	22.	}
	23.
	24.	public int[] getCache(int num){
	25.		if(num == lastNumber){ //Check-Then-Act.
	26.			return numPower;
	27.		}	
	28.		return null;
	29.	}
	30.}

我们不难看出，上面的程序存在着严重的线程安全性。由于 `lastNumber` 和 `numpower` 存在着逻辑关联性，而 `resetCache` 和 `getCache` 方法都是非原子的操作，必然会使 `lastNumber` 和 `numPower`出现数据不一致的情况。通常，我们会给 `resetCache` 和 `getCache` 加上 `synchronized` 同步使其变成原子操作，并且保证了内存可见性。

但其实，我们完全可以使用不可变类来代替高消耗的加锁操作。

	1.public class PowerServer{
	2.	
	3.	private ImmutableOneValueCache o = new ImmutableOneValueCache();
	4.	
	5.	public int[] getPower(int num){
	6.		int[] value = o.getCache(num);
	7.		if(value == null){
	8.			value = new int[]{num*num, num*num*num};
	9.			//当我们组合了一个不可变类使，对状态的所有更改操作都需copy一个新值。
	9.			o = new ImmutableOneValueCache(num, value);
	10.		}
	11.		return value;
	12.	}
	13.}
	14.
	15.final class ImMutableOneValueCache{
	16.	  private final int lastNumber;
	17.	  private final int[] numPower;
	18.	  //对于不可变类的构造器，不能直接接受外部的可变reference。
	19.	  public MutableOneValueCache(int lastNumber, int[] numPower){
	20.       this.lastNumber = lastNumber;
	21.       this.numPower = Arrays.copyOf(numPower);
	22.   }
	23.
	24.	  public int[] getCache(int num){
	25.		if(num == lastNumber){ //Check-Then-Act.
	26.			return Arrays.copyOf(numPower);//不能将可变的reference直接发布出去。
	27.		}	
	28.		return null;
	29.	  }
	30.}

	#使用：
	public static void main(String[] args){
		PowerServer server = new PowerServer();
		new Thread(() -> {
			int[] ret = server.getPower(10);
			#当得到值后，我们可能会操作此对象
			ret[0] += 1;
			ret[1] += 1;
		}).start();
	
		new Thread(() -> {
			int[] ret = server.getPower(10);
			#当得到值后，我们可能会操作此对象
			ret[0] += 1;
			ret[1] += 1;
		}).start();
	}


当我们在使用一个线程安全的不可变对象时，任何对此对象所维护的状态的更新都是用包含了新状态的新对象去替换这个旧的对象，而不是在旧的对象的基础上做操作。这样就可以保证其他线程后续对那个旧对象的操作不会出现问题。

程序开发者可能会怀疑使用大量的复制操作会消耗性能，其实这种用户级别的指令级消耗并不比加锁开销大。
