---
title: "Mutexs and condition variable"
data: "2016-10-01"
tags: ["concurrency"]
---

我们在日常编程中使用并发机制主要解决以下两方面的问题：

 - Race Condition<br/>
    通过提供 `Mutual exclusion` 机制， 确保多个进程或线程不会同时执行某些特定的程序段（代码段），叫做 `critial section`。当进程访问 `critial section` 时，会受到同步机制的控制。这种机制提供了对状态的互斥访问操作，可以用来解决 `Race Condition`，以实现数据的一致性。
 
 - busy-waiting：<br/>
    首先，忙等待发生在当进程频繁的轮询去查看是否可以进入 `critial secion`时，这会造成大量的CUP空转浪费。所以，在等待某个可能长期不会成真的条件时，使进程处于阻塞状态可能会更好一些。通过 `condition veriable` 提供的 `wait` 和 `signal`机制可以使线程在某个条件变量上等待，并可以唤醒等待在某个条件变量上的线程。

在 linux 下, `POSIX` 提供了很多关于线程同步的 `Higher-level API` ，本文主关注下面两个最常用的。

 - mutexs(Mutual exclusion)：
	可使得线程对某段(section)代码互斥执行，用来实现 `critical section`, 解决 `Race condition`。
 
 - condition variables：
 	条件变量,使得一个线程可以在等待一个条件为真时挂起，并由别的线程在条件成立时在将其唤醒，有效的提供了cpu使用率。

---
<H4>Mutex(Mutual exclusion)</H4>
---

`Mutex variable` 被定义为`pthread_mutex_t`。 并且在使用前，必须通 `pthread_mutex_init`初始化。

一个 `pthread_mutex_t` 类型的变量就像一个 锁。在任意时刻，只能有一个线程 `lock(or own)` 这个 `mutex` 变量。`pthread_mutex_lock`:提供了**原子地**获取锁操作，即使多个线程并发执行`pthread_mutex_lock`，也只会有一个线程成功。其他失败的线程将 `block`。

`pthread_mutex_unlock(mutex)` 将释放这个 `mutex`变量。

在这里，我们通过经典的 `producer-consumer problem` 来使用 `mutex`。`Mutex variable` 主要就是解决 `Race Condition` 的。

    #include <pthread.h>
    #include <stdio.h>
    
    #define MAX 10
    
    //struction 
    int data[MAX];
    int itemCount;
    int next = 0;
    int prev = -1;
    
    //define a mutex variable
    pthread_mutex_t lock
    
    void *producer(){
        int item = 3;
        while(1){
            pthread_mutex_lock(&lock);
            if(itemCount < MAX){
                data[(next++)%MAX] =item;
                itemCount++;
            }
            pthread_mutex_unlock(&lock);
        }
    }
    
    void *consumer(){
        int item = -1;
        while(1){
            pthread_mutex_lock(&lock);
            if(itemCount > 0){
                item = data[(++prev)%MAX];
                itemCount--;
                printf("consumer:%d", item);
            }
            pthread_mutex_unlock(&lock);
        }
    }
 

可以看到，使用 `lock/unlock` 提供了对队列的互斥操作。避免了数据不一致的情况。但这里有一个问题，就是存在 `busy-waiting`问题，尤其是当生产/消费数据的事件很长时，这会导致cpu严重的低利用率。

所以，我们需提供一种机制使线程在长时间等待一个条件为真时 `block`,即不会被cpu调度，并且，当确定条件满足时由另一个线程唤醒它。

---
<H4>condition variable</H4>
---

`pthread_cond_t` 表示一个condition variable，同样在使用前必须通过 `pthread_cond_init()`初始化。

`pthread_cond_wait(cond, mutex)`：这个函数将使调用线程阻塞，此时我们说这个线程阻塞在了这个 `cond` 条件变量上了。

注意：**它应该在当前线程已经 `lock` 了这个 `mutex` 时执行，并且将自动释放这个 `mutex`。当收到 `signal` 时，这个线程会  `awakened`，并且会自动的获取 `mutex`。**

`pthread_cond_signal(cond)`: 会释放在当前 `cond` 条件变量上阻塞的一个线程。我们注意到，`signal` 不需要和特定的 `mutex` "绑定"，并且也不没有强制必须 `lock` 了 `mutex` 时才能执行。

ok, 我们可以使用 条件变量来实现上面生产者-消费者的 `busy-waiting` 问题。
    
    pthread_cond_t notFull, notEmpty;
    void *producer(){
        int item = 3;
        while(1){
            pthread_mutex_lock(&lock);
            while(item >= MAX)
                pthread_cond_wait(&notFull, &lock);
            data[(next++)%MAX] =item;
            itemCount++;
            pthread_cond_signal(&notEmpty);
            pthread_mutex_unlock(&lock);
        }
    }
    
    void *consumer(){
        int item = -1;
        while(1){
            pthread_mutex_lock(&lock);
            if(itemCount <= 0)
                pthread_cond_wait(&notEmpty, &lock);
            item = data[(++prev)%MAX];
            itemCount--;
            printf("consumer:%d", item);
            pthread_cond_signal(&notFull);
            pthread_mutex_unlock(&lock);
        }
    }

关于 条件变量 正确的用法：

 - 对于 wait() 端：
1.必须与 `mutex` 一起使用，布尔表达式的读写需受此 mutex 保护
2.在 `mutex` 已上锁的时侯才能调用 `wait`
3.把 判断布尔条件 和 wait() 放在 while 循环中。

 - 对于 signal/broadcast 端:
1.不一定要在 `mutex` 以上锁的情况下调用。
2.在singal之前一般要修改布尔表达式。
3.修改布尔表达式通常要 `mutex` 保护。


---    
<H4>总结</H4>
---
在构建多线程并发同步程序时，`mutex` 和 `condition variable` 是最常使用的同步原语。前者实现了互斥访问，解决了 `Race Condition`，后者解决了 `busy-waiting` 问题。当然，这些原语还是很低级的，尤其是 `condition variable`很容器出错。在实际开发中，应尽量使用更高层的同步构建工具。







    



