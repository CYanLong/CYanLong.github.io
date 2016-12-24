---
title: "java对象头"
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
	
	+------------------------------------------------------+-------------+
	|           Object Header (64 bytes)                   |  State	     |
	+------------------------------------------------------+-------------+
	|               Mark Word(32 bits)                     |             |
	+------------------------------------------------------+-------------+
	| identity_hashcode:25 | age:4 | biased_lock:1 | lock:2|   Normal    |
	+------------------------------------------------------+-------------+
	| thread:23 | epoch:2 | age:4 | biased_lock:1 | lock:2 |   Bised     |
	+------------------------------------------------------+-------------+
	|       ptr_to_lock_record:30                 | lock:2 | Light Lock  | 
	+------------------------------------------------------+-------------+
	|      ptr_to_heavyweight_moniter:30          | lock:2 | Weight Lock |
	+------------------------------------------------------+-------------+


1. **identity_hashcode** identity hashcode of the object which is assigned lazily.If System.identityHashCode(obj) is called, it is calculated and written into the object header. When object is locked the identity hashcode value is moved into the moniter object.

2. **age** number of garbage collections the object has survived.It is incremented every time an object is copied within the young generation.
When the age field reaches the value of max-tenuring-threshold, the object is promoted to the old generation.

3. **biased_lock** contains 1 if the biased locking is enabled for the class. 0 if the biased locking is disabled for the class.

4. **lock** the lock state of the object. 00 - Lightweight Locked, 01- Unlocked or Biased, 10-Heavyweight Locked, 11-Marked for Garbage Collection.

5. **thread** when an object is biased towards specific thread instead of identity hash code mark word contains threadid.

6. **epoch** which acts as a timestamp that indicates the validity of the bias.

7. **ptr\_to\_lock_record** When lock acquisitions are uncontended JVM uses atomic operations instead of OS mutexes. This technique is known as **Lightweight Locking**. In case of lightweight locking JVM sets a pointer to the lock record in the object's header word via a CAS operation.

8. **ptr\_to\_heavyweight_monitor** If two different threads concurrently synchronize on the same object, the lightweight lock must be inflated to a Heavyweight Monitor for the management of waiting threads. In case of heavyweight locking JVM sets a pointer to the moniter in the object's header word.


注意:<br/>
1. 当一个对象已经计算过 identity hashcode,它就无法进入偏向锁状态。<br/>
2. 当一个对象当前正处于偏向锁状态，并且需要计算其identity hashcode时，则它的偏向锁会被撤销，并且锁会膨胀为重量锁。


<H4>2. klass pointer </H4>
The **klass pointer** has word size on 32 bit architectures. On 64 bit architectures the klass pointer either has word size, **but can also have 4 byte if the heap address can be encoded in these 4 bytes.**

The optimization is called "compressed oopes"。

<H4>3.压缩指针</H4>


Ordinary Object Pointers(OOPs) 是jvm用来指向对象引用的。当oops仅仅有32-bits长时，只能访问到4G范围的内存空间，也就是说，在32-bit JVM中最大堆内存限制在了4G。

而在64-bit JVM中，我们能访问兆比特(terabytes)级别的内存。但是，大部分时候，我们根本没有这么大的内存。我们的内存可能仅仅不到 32G，但却因为64-bits 的对象引用导致内存使用率比32-bits高了大约1.5倍。

我们现在要做的是用32-bits的oops尽可能大的表示内存范围。我们知道，java对象都是 8-bytes对齐的，也就是说所有java对象的内存地址的后 3-bits 都一定为 0.这样，我们可以利用这个特性尽可能大的表示内存范围。我们将32-bits的oops值表示为从35-bit开始的内存地址。当要使用这个值时(get_field), 左移3位(后3位补0)即可得到原始值。这时我们可以访问大概32G范围的内存。对于大部分的应用来说已经很满足了。

在java7及之后的版本中，当最大堆内存小于 32GB 时，-XX:+UseCompressedOops为默认开启。

<H4>3.array length (opt)</H4>
Array has an extra 32 bit word to store array size. 



