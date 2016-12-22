---
titile: "java对象头"
data: "2016-12-19"
tags : ["jvm"]
---

在 `HotSpot` 虚拟机中，对象在内存中存储的布局可以分为3块区域：`对象头（Header）`, `实例数据(Instance Data)`，`对齐填充(Padding)`。

<H4>对象头(Header)</H4>

&emsp;&emsp;The header consists of a **"mark word"**, **a pointer to the object's class**, **array size in case of an array**, and **padding to reach the next 8-byte boundary**. (In HotSpot).
	 
	+-------------+--------------- +-----------------+---------+
	|  mark word  |	 klass pointer | array size(opt) | padding |
	+-------------+--------------- +-----------------+---------+

<H4>1. mark word </H4>

The **mark word** has **word size** (`4 byte` on 32 bit architectures, `8 byte` on 64 bit architectures)。

The **mark word** is actually used for many things.

**32 bit JVM**
	
	|------------------------------------------------------|
	|			Object Header (64 bytes)	  			   |  State	   
	|------------------------------------------------------|
	|		 		Mark Word(32 bits)		  			   |
	|------------------------------------------------------|	
	| identity_hashcode:25 | age:4 | biased_lock:1 | lock:2|   Normal  
	|------------------------------------------------------+
	| thread:23 | epoch:2 | age:4 | biased_lock:1 | lock:2 |   Bised   
	|------------------------------------------------------+
	| 		ptr_to_lock_record:30                 | lock:2 | Light Lock    
	|------------------------------------------------------+
	|       ptr_to_heavyweight_moniter:30		  | lock:2 |Weight Lock
	|------------------------------------------------------+


1. **identity_hashcode** identity hashcode of the object which is assigned lazily.If System.identityHashCode(obj) is called, it is calculated and written into the object header. When object is locked the identity hashcode value is moved into the moniter object.

2. **age** number of garbage collections the object has survived.It is incremented every time an object is copied within the young generation.
When the age field reaches the value of max-tenuring-threshold, the object is promoted to the old generation.

3. **biased_lock** contains 1 if the biased locking is enabled for the class. 0 if the biased locking is disabled for the class.

4. **lock** the lock state of the object. 00 - Lightweight Locked, 01- Unlocked or Biased, 10-Heavyweight Locked, 11-Marked for Garbage Collection.

5. **thread** when an object is biased towards specific thread instead of identity hash code mark word contains threadid.

6. **epoch** which acts as a timestamp that indicates the validity of the bias.

7. **ptr\_to\_lock_record** When lock acquisitions are uncontended JVM uses atomic operations instead of OS mutexes. This technique is known as **Lightweight Locking**. In case of lightweight locking JVM sets a pointer to the lock record in the object's header word via a CAS operation.

8. **ptr\_to\_heavyweight_monitor** If two different threads concurrently synchronize on the same object, the lightweight lock must be inflated to a Heavyweight Monitor for the management of waiting threads. In case of heavyweight locking JVM sets a pointer to the moniter in the object's header word.


注意:
1. 当一个对象已经计算过 identity hashcode,它就无法进入偏向锁状态。
2. 当一个对象当前正处于偏向锁状态，并且需要计算其identity hashcode时，则它的偏向锁会被撤销，并且锁会膨胀为重量锁。


<H4>2. klass pointer </H4>
The **klass pointer** has word size on 32 bit architectures. On 64 bit architectures the klass pointer either has word size, **but can also have 4 byte if the heap address can be encoded in these 4 bytes.**

The optimization is called "compressed oopes"。

<H4>3.array length (opt)<H4>
Array has an extra 32 bit word to store array size. 



