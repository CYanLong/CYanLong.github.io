---
title: "事务和锁"
date: "2016-05-21"
tag: [MySQL]
---

---
&emsp;&emsp;事务(Transaction)和锁机制(locking)是数据库区别于文件系统的重要特性之一，也是确保数据一致性最重要的保护机制。

<hr/>
<h3>一:什么是锁:</h3>


&emsp;&emsp;锁机制用于管理对共享资源的并发访问。InnoDB储存引擎实现了如下两种标准的行级锁:

&emsp;&emsp;共享锁(shared lock) 和 排他锁(exclusive lock)，也叫读锁(read lock) 和 写锁(write lock)。

&emsp;&emsp;读锁是共享的，或者说是相互不阻塞的。多个客户在同一时刻可以读取同一资源，而互不干扰。写锁则是排他的，也就是说一个写锁会阻塞其他的写锁和读锁，这是出于安全策略的考虑。

&emsp;&emsp;一个用户在对表进行写操作(插入，删除，更新等)前，需要先获得写锁(exclusive lock)，这会阻塞其他用户对该表的所有读写操作。只有没有写锁时，其他读取的用户才能获得读锁，读锁之间是不相互阻塞的。

&emsp;&emsp;此外，InnoDB存储引擎支持多粒度(granular)锁定。

---
<h3>二:锁粒度</h3>

&emsp;&emsp;一种提高共享资源并发性的方式就是让锁定对象更有选择性。尽量只锁定需要修改的部分数据，而不是所有的资源。更理想的方式是，只对会修改的数据片进行精确的锁定。任何时候，在给定的资源上，锁定的数据量越少，则系统的并发程度越高。

&emsp;&emsp;所谓的锁策略，就是在锁的开销和数据的安全性之间寻求平衡。

<h4>表锁(table lock)</h4>

&emsp;&emsp;表锁是MySQL中最基本的锁策略，并且是开销最小的策略。它会锁定整张表。

<h4>行级锁(row lock)</h4>

&emsp;&emsp;行级锁可以最大程度地支持并发处理(同时也带来了最大的锁开销)，在InnoDB中实现了行级锁。行级锁只在存储引擎层实现，而MySQL服务器层没有实现。

<h4>隐式和显式锁</h4>

&emsp;&emsp;InnoDB采用的是两阶段锁定协议(tow-phase locking protocol)。在事务执行过程中，随时都可以执行锁定。锁只有在执行COMMIT或者ROLLBACK的时候才会释放，并且所有的锁是在同一时刻被释放。InnoDB会根据隔离级别在需要的时候自动加锁。

&emsp;&emsp;另外，InnoDB也支持通过特定的语句进行显示锁定。

	SELECT ... LOCK IN SHARE MODE
	SELECT ... FOR UPDATE

&emsp;&emsp;MySQL也支持LOCK TABLES 和 UNLOCK TABLES语句，这是在服务器层实现的，和存储引擎无关。

---
<h3>三:一致性非锁定读.</h3>

&emsp;&emsp;MySQL的大多数事务型存储引擎实现的都不是简单的行级锁。基于提升并发性能的考虑，它们一般都同时实现了多版本并发控制(Multiversion Concurrency Control)。

&emsp;&emsp;可以认为MVCC是行级锁的一个变种，但是它在很多情况下避免了加锁操作。 因此开销更低。大多数MVCC都实现了非阻塞的读操作，写操作也只锁定必要的行。

&emsp;&emsp;MVCC的实现，是通过保存数据在某个时间点的快照来实现的。也就是说，不管需要执行多长时间，每个事务看到的数据都是一致的。根据事务开始的时间不同，每个事务对同一张表，同一操作看到的数据可能是不一样的。

&emsp;&emsp;InnoDB的MVCC，是通过在每行记录后面保存两个隐藏的列来实现的。这两个列，一个保存了行的创建时间，一个保存行的过期时间(或删除时间)。当然存储的并不是实际的时间值，而是系统版本号(system version number)。每开始一个新的事务，系统版本号都会自动递增。事务开始时刻的系统版本号会作为事务的版本号，用来和查询到的每行记录的版本号进行比较。下面在REPEATABLE READ隔离级别下，MVCC的具体操作.

###1.SELECT

&emsp;&emsp;InnoDB会根据以下两个条件检查每行记录。

&emsp;&emsp;a.InnoDB只查找版本早于当前事务版本的数据行。(也就是说，行的系统版本号小于或等于事务的系统版本号)，这样可以确保事务读取的行，要么是在事务开始前已经存在，要么是事务自身插入或修改过的。

&emsp;&emsp;.行的删除版本要么未定义，要么大于等于当前事务版本号。这可以确定事务读取到的行，在事务开始之前未被删除。由于是REPEATABLE READ隔离级别，后开始的事务对数据的影响不应该被先开始的事务看到，所以该行应该被返回。

&emsp;&emsp;只有符合上述两个条件的记录，才能返回作为查询结果。

###2.INSERT
&emsp;&emsp;InnoDB为新插入的每一行保存当前系统版本号作为行版本号。

###3.DELETE

&emsp;&emsp;InnoDB为删除的每一行保存当前系统版本号作为行删除标识。

###4.UPDATE

&emsp;&emsp;InnoDB为插入一行新纪录，保存当前系统版本号作为行版本号，同时保存当前系统版本号到原来的行作为行删除标识。(在select时，只会返回行删除标识大于当前版本号的数据)。

###MVCC优点总结:

&emsp;&emsp;上述策略的结果就是，在读取数据的时候，innoDB几乎不用获得任何锁，每个查询都通过版本检查，只获得自己需要的数据版本，从而大大提高了系统的并发度。

&emsp;&emsp;MVCC只在REPEATABLE READ 和 READ COMMITTED 两个隔离级别下工作。其他两个隔离级别都和MVCC不兼容，因为READ UNCOMMITTED总是读取最新的数据行，而不是符合当前事务版本的数据行。而SERIALIZABLE则会对所有读取的行都加锁。

---
<h3>四:一致性锁定读:</h3>
&emsp;&emsp;上文中，我们知道，在默认配置下，即事务的隔离级别为REPEATABLE READ模式下，InnoDB存储引擎的SELECT操作使用一致性非锁定读。但是在某些情况下，用户需要显示地对数据库读取操作进行加锁以保证数据逻辑的一致性。而这要求数据库支持加锁语句，即使是对于SELECT的只读操作。InnoDB存储引擎对于SELECT语句支持两种一致性的锁定读(locking read)操作。

	1.SELECT ... FOR UPDATE
	2.SELECT ... LOCK IN SHARE MODE

&emsp;&emsp;`SELECT ... FOR UPDATE `对读取的行记录加一个 `X锁`，其他事务不能对已锁定的行加上任何锁。 `SELECT ... LOCK IN SHARE MODE `对读取的行记录加一个 `S锁`，其他事务可以向被锁定的行加`S锁`，但如果加`X锁`，则会被阻塞。


<h3>五:事务的隔离级别</h3>

&emsp;&emsp;在SQL标准中定义了四种隔离级别。

&emsp;&emsp;用户可以用SET TRANSACTION语句改变单个会话或者所有新进连接的隔离级别。语法如下:
	
	SET [SESSION | GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}

&emsp;&emsp;默认的行为(不带session或global)是为下一个(为开始)事务设置隔离级别。使用SESSION关键字为将来在当前连接上执行的事务设置默认事务级别。

&emsp;&emsp;可以使用下列语句查询全局和会话事务隔离级别。
	
	SELECT @@global.tx_isolation
	SELECT @@session.tx_isolation
	SELECT @@tx_isolation

###READ UNCOMMITTED(未提交读)

&emsp;&emsp;在READ UNCOMMITTED级别，事务中的修改，即使没有提交，对其他事务也都是可见的。事务可以读取未提交的数据，这也被称为脏读(Dirty Read).此级别下的数据库锁情况:事务在读数据的时候并未对数据加锁，事务在修改数据的时候只对数据增加行级共享锁。
	
&emsp;有如下表数据:

	mysql> select * from user;
	+-------+----------+
	| id    | username |
	+-------+----------+
	| sdfds | abcde    |
	+-------+----------+


将两个连入的session都设置为READ UNCOMMITTED隔离级别。

	------------------session1----------------------------------	
	start transaction;

	update user set username='a';

此时session1的事务未提交，session2执行查询。会查询到session1中update过但为提交的数据。(Dirty data).

	----------------session2------------------------
	session2:
	
	select * from user;
	
	+-------+----------+
	| id    | username |
	+-------+----------+
	| sdfds | a    	   |
	+-------+----------+

总结:事务一读取某行记录时，事务二也能对这行记录进行读取，更新。(因为事务一并未对数据增加任何锁)。当事务二对该记录进行更新时，事务一再次读取该记录，能读到事务二对该记录的修改版本。()

###READ COMMITTED(提交读，不可重复读)

&emsp;&emsp;READ COMMITTED解决了上面READ UNCOMMITTED 隔离级别的脏读问题，换句话说，一个事务从开始直到提交之前，所做的任何修改对其他事务都是不可见的。然而，这种隔离级别的问题是会出现同一事务的多次相同查询结果可能会不一致。也就是说，这种隔离级别下:一个事务开始后，不能"看到"另一个事物修改未提交的数据(解决了脏读问题)，但能"看见"已经提交的事务所做的修改(存在了不可重复读问题)。提交读的数据库锁情况:事务对当前被读取的数据加行级共享锁，一旦读完改行，立即释放该行级共享锁；事务在更新某个数据的瞬间，必须先对其加行级排他锁，直到数据结束才释放。

&emsp;&emsp;照例使用上面的测试数据.下面的session1和session都为READ COMMITTED隔离级别。

	-------------session1----------------
	start transaction;
	
	select * from user;
	+-------+----------+
	 id     | username |
	+-------+----------+
	| sdfds | t        |
	+-------+----------+
	
&emsp;&emsp;此时session2进入，改变这一行的数据。
	
	------------session2----------
	start transaction;
	
	update user set username='q';
	
	commit;
	
&emsp;&emsp;session1再次执行相同的查询操作
	
	---------session1-----------
	select * from user;
	+-------+----------+
	 id     | username |
	+-------+----------+
	| sdfds | q        |
	+-------+----------+

&emsp;&emsp;由于session2对user的数据修改已经提交，所以session2在同一事务中执行的两次相同查询得到的结果不同。

&emsp;&emsp;总结:事务1在读取某行记录的整个过程中，事务2都可以对该行记录进行读取。(因为事务1对该行记录增加行级共享锁的情况下，事务二同样可以对该数据增加共享锁来读数据)；事务一读取某行的一瞬间，事务2不能修改该行数据，但是，只要事务1读取完该行数据，事务2就可以对该行数据进行修改。(事务一在读取的瞬间会对数据增加共享锁，任何其他事务都不能对该行数据增加排他锁。)；事务一更新某行记录时，事务二不能对该行记录做更新，直至事务一结束。(事务一在更新数据的时候，会对该行数据增加排他锁，直到事务结束才会释放锁，所以，在事务一没有提交之前，事务二不能对数据增加共享锁进行数据的读取。所以，提交读可以解决脏读的现象)。 

###REPEATABLE READ(可重复读)

&emsp;&emsp;该级别保证了在同一事务中多次读取相同记录的结果是一致的。但理论上，可重复读隔离级别还是无法解决另外一个幻读(Phantom Read)的问题。所谓幻读，指的是当某个事务在读取某个范围内的记录时，另外一个事务又在该范围内插入了新的记录(并以提交)，当之前的事务再次更新该范围的记录时，会产生幻行(Phantom Row).

	------session1------
	start transaction;
	
	select * from user;
	+-------+----------+
	| id    | username |
	+-------+----------+
	| aw    | 123      |
	+-------+----------+

&emsp;&emsp;此时session2进入，插入了一条数据，并立即提交.

	----session2--------
	start transaction;
	
	insert into user values('a', '789');
	
	commit;#提交.

&emsp;&emsp;这是，session1再次执行查找，不会显示session2修改的数据.(此隔离级别下实现了可重复读).
但当session1去更新数据时，就会更新到session2修改的数据.

	----session1-----
	select * from user; #在此查询，可重复读。跟原来一样。
	+-------+----------+
	| id    | username |
	+-------+----------+
	| aw    | 123      |
	+-------+----------+
	
	update user set username='100';
	Query OK, 2 rows affected (0.00 sec)
	#当session1执行update时，令一个事务里提交的数据就出现了。
	select * from user;
	+-------+----------+
	| id    | username |
	+-------+----------+
	| aw    | 123      |
	| sd    | 123      |
	+-------+----------+
	
4.SERIALIZABLE(可串行化).

SERIALIZABLE是最高的隔离级别。它通过强制事务串行执行，避免了前面说的幻读的问题。简单来说，SERIALIZABLE会在读取的每一行数据上都加锁，所以可能导致大量的超时和锁争用的问题。一般不使用此隔离级别。

