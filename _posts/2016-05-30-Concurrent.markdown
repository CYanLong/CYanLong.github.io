---
title: "ConcurrentHashMap"
data: "2016-05-30"
tags: [concurrent]
---

Java平台类库包含了丰富的并发基础构建模块，例如线程安全的容器类以及各种用于协调多个相互协作的线程控制流的同步工具类(Synchronizer)。

一:同步容器类

同步容器类包括Vector和Hashtable，此外还包括有Collections.synchronizedXxx等工厂方法创建的。这些类实现线程安全的方式是:将它们的状态封装起来，并对每个公有方法都进行同步，使得每次只有一个线程能访问容器的状态。

同步容器类都是线程安全的，但在某些情况下可能需要额外的客户端加锁来保护复合操作。

1.迭代操作
在调用size()和get()操作之间，Vector的长度可能会发生变化。

---
	for(int i = 0 ; i < vector.size() ; i++){
		Object o = vector.get(i);	
	}

在有其他线程并发修改Vector时，那么这种迭代方法将抛出ArrayIndexOutOfBoundsException异常.

因此，即使是使用像Vector这样的同步容器，也需要在使用者在必要使进行同步控制。

---
	synchronized(vector){
		for(int i = 0 ; i < vector.size() ; i++){
			Object o = vector.get(i);
		}
	}


二:迭代器和ConcurrentModificationException

&emsp;&emsp;如果其他线程并发修改容器，那么即使是使用迭代器也无法避免在迭代期间对容器加锁。在同步容器的迭代器中，并没有考虑到并发修改的问题，并且它们表现出的行为是"fail-fast".这意味着，当它们发现容器在迭代过程中被修改时，就会抛出一个ConcurrentModificationException.
无论是同步的SynchronizedArrayList还是非同步的ArrayList，在迭代时直接通过容器的remove(),add()等操作改变容器的结构都会发生此异常。还有一种更糟糕的情况是，在多线程下，即使你本身没有修改容器，仅仅是单纯的迭代，也会由于其他线程在你迭代期间由于改变了容器的结构而导致你的迭代失败。

---
	private static List<Integer> list = Collections.synchronizedList(new ArrayList<Integer>());


	public static void main(String[] args){
		for(int i = 0 ; i < 20 ; i++){
			list.add(i);
		}
		
		new Thread(() -> {
			Iterator<Integer> iterator = list.iterator();
			while(iterator.hasNext()){
				int i = iterator.next();
				System.out.println("ThreadOne 遍历" + i);
				try{
					Thread.sleep(10);
				}catch(InterruptedException e){
					e.printStackTrace();
				}
			}
		}).start();
		
		new Thread(() -> {
			for(int i = 0 ; i < 10 ; i++){
				list.add(i);
			}
		}).start();
	}

三:并发容器类。

&emsp;&emsp;
Java5.0提供了并发容器类来改进同步容器的性能。同步容器将所有对容器状态的访问都串行化，以实现它们的线程安全性。
这种方法的代价是严重降低并发性，当多个线程竞争容器的锁时，吞吐量将严重影响。

&emsp;&emsp;
在Java5中增加了ConcurrentHashMap，用来替代同步的基于散列的Map，以及CopyOnWriteArrayList，用来在遍历操作为主要操作的情况下替代同步的List。

&emsp;1.CopyOnWriteArrayList/CopyOnWriteSet
--

&emsp;&emsp;
正如名字所暗示的那样，CopyOnWriteArrayList实现的基本思路是，当需要往容器中写入数据时就会复制一份数据，然后基于新的数据操作，不改变原来的数据。

&emsp;&emsp;1.读操作，不加锁。

&emsp;&emsp;2.永远不会抛出ConcurrentModificationException异常。

&emsp;&emsp;我们来看一看迭代器的实现:

---
	
	private final Object[] snapshot;
	
	private int cursor;
		
	private COWIterator(Object[] elements, int initialCursor){
		this.elements = elements;
		this.initialCursor = cursor;
	}
	
	public boolean hasNext(){
		return cursor < snapshot.length;
	}
	
	public E next(){
		if(! hasNext())
			throw new NoSuchElementException();
		return (E) snapshot[cursor++];
	}

&emsp;&emsp;可以发现，CopyOnWriteList的迭代器实现非常单纯。这种单纯的实现方式得益于snapshot仅为创建此迭代器时的elements，并且此对象不会在变，CopyOnWrite机制使得elements数组成为一个不可变对象。此迭代器具有若一致性(Weakly Consistent),而并非"即时失败"。弱一致性的迭代器可以容忍并发的修改。


2.ConcurrentHashMap
---

&emsp;&emsp;与HashMap一样，ConcurrentHashMap也是一个基于散列的Map，但它使用了一种完全不同的加锁策略来提供更高的并发性和伸缩性。ConcurrentHashMap并不是将每个方法都在同一个锁上同步，这使得每次只有一个线程访问容器。而是使用一种粒度更细的加锁机制来实现更大程度的共享，这种机制称为分段锁(Locking Striping).
在这种机制下，任意数量的读取线程可以并发地访问Map,执行读取操作的线程和执行写入操作的线程可以并发地访问Map，并且一定数量的写入线程可以并发地修改Map。
同样的，ConcurrentHashMap迭代时也不会抛出ConcurrentModificationException.
