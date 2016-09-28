---
title: "进程内存管理"
data: "2016-09-28"
tags： ["linux"]
---

---

我们知道, 进程有自己独立的虚拟地址空间, 在 64-bit linux 系统下，理论上最大寻址空间达到了 2^64。一个程序执行的并不是随便使用这么大的空间，而是涉及到程序的内存分配策率。从内存分配的角度来分，有 `static allocation`, `automatic allocation`, `dynamic allocation`。 从内存布局的角度大致分为：数据段，代码段，栈段，堆。


---
<H4>静态分配 (static allocation)</H4>
---
静态分配指在程序开始执行前，为程序的代码，全局变量等分配内存。这种分配在编译时就已经确定了大小并且在整个程序执行过程中是固定不变的。

程序运行所需的代码都放在一起，叫做代码段，并且由于这段内存就是用来存放代码的，所以操作系统可以限定其为只读段加以保护；同样，程序在编译期可知的各种全局变量的数据也存放在一个区域，叫做数据段，这个区域的内存属性为可读/可写。

---
<H4>自动分配</H4>
---
一个程序的运行，除了代码段和数据段之外，在程序运行过程中还需要一些存放临时数据的地方，由于程序的运行总是以函数调用的形式进行的，且函数调用为典型的 `后进先出(LIFO)`，所以，我们把这个用来保存程序执行过程(函数调用)产生的临时局部变量的区域叫做 `栈(Stack) 区`。并且，在同一个进程中，每一个线程有自己独立的 `Stack Segement`。

这种在程序执行过程中随着函数的调用自动分配内存叫做 `automatic allocation`。

---
<H4>动态分配</H4>
---
我们知道，自动分配虽然可以在运行时分配内存，但自动分配的内存只能在当前函数栈帧中使用，无法将数据传递至函数外部，也就是说仅仅有静态分配和自动分配无法在程序运行期间产生全局内存，只能在编译的时候定义全局变量，在很多情况下缺乏表现力。动态内存分配就是提供一种在程序运行期间动态的分配内存的方式。与 `automatic allocation` 为局部变量分配内存不同的是，动态内存的分配需要进行 `系统调用`。

我们先来看一看一个进程的整个虚拟地址空间分布情况：

<img src="http://7xtdq2.com1.z0.glb.clouddn.com/CvITh.png"/>

如上图，从低地址 到 高地址 依次为: `Read-only code and data`， `Read/Write data`(这两部分内存是静态分配得到的，并且在整个程序执行期间将是固定的)， `Run-time heap (create at run time by brk syscall)`（堆分配），`memory mapped region`, `User stack`（栈区）。

实验：

1.每个线程有自己独立的栈区
    
    #include <pthread.h>
    #include <stdio.h>
    
    void *threadFunc(){
        int autoVari = 0;
        sleep(1000);
    }
    
    int main(){
        pthread_t pt;
        int ret;
        printf("%d\n", getpid());        
        getchar();

        ret = pthread_create(&pt, NULL, threadFunc, NULL);
        //暂停一下
        getchar();
        getchar();
        return 1;
        
    }
    
    运行程序并利用 `cat /proc/pid/maps 查看其进程地址空间布局:
    
    00400000-00401000 r-xp 00000000 fd:00 4324723                            /home/cyl/heap_mang/thread-stack
    00600000-00601000 r--p 00000000 fd:00 4324723                            /home/cyl/heap_mang/thread-stack
    00601000-00602000 rw-p 00001000 fd:00 4324723                            /home/cyl/heap_mang/thread-stack
    //pt 线程的stack.
    7f3c67b64000-7f3c68364000 rw-p 00000000 00:00 0 [stack:23170]

    ...//省略
    
    #主线程的stack.
    7fffbb769000-7fffbb78a000 rw-p 00000000 00:00 0  [stack]
    7fffbb7a3000-7fffbb7a5000 r-xp 00000000 00:00 0  [vdso]
    ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0 [vsyscall]

我们可以查看到，有两个 `stack`区。并且，我们可以根据段的读写属性判断出最上面的三个段分别为 `代码段`，`只读数据段(常量)`, `可读/可写数据段`。处于低地址，且在 64-bit 下从 `0040000` 开始。

---
<H4>堆内存管理</H4>
---
实际上，我们有两种途径动态的为进程分配 `堆内存(heap)`。涉及到三个系统调用。

 - int brk(void* addr);
 - void* sbrk();
 - void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);

区别在于它们所分配的内存区域是不同的。首先，考虑上图的虚拟地址空间，从低地址数据段到中间的共享库区域 和 从共享库区域到高地址的栈区 都是可用的，所以都可以用来动态分配给进程。具体的， brk 和 sbrk 都是在前者区域分配， mmap 使用后者的区域进行分配。

实验:
    #include <unistd.h>
    
    int main(){
        void *curr_brk, *temp_brk = NULL;
        printf("Welcome to sbrk example:%d\n", getpid());
        //get the current break addr
        temp_brk = sbrk(0);
        printf("Program Break Location: %p\n", curr_brk);
        getchar();
        curr_brk = sbrk(100*1024*1024);
        printf("Program break Location: %p\n", curr_brk):
        getchar();
        getchar();
        return 0;
    }
    
同样，我们利用 `cat` 分析内存布局如下：
   
    00400000-00401000 r-xp 00000000 fd:00 4324728   heap_mang/mm-test
    00600000-00601000 r--p 00000000 fd:00 4324728   heap_mang/mm-test
    00601000-00602000 rw-p 00001000 fd:00 4324728   heap_mang/mm-test
    01b67000-01b88000 rw-p 00000000 00:00 0 [heap]

我们可以看到，当使用sbrk 进行动态内存分配时，在高于数据段但低于共享库段之间出现了 `heap`区。

ok, 我们来看看mmap系统调用。

    #include <unistd.h>
    
    int main(){
        char* addr = NULL
        int ret = -1;
        getchar();
        
        addr = mmap(NULL, (size_t)13*1023, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
        printf("addr location:%p", addr);
        getchar();
        getchar();
    }
    
    Output:
    new addr: 0x7f2a259e6000
    
其内存布局如下:
    
    7f2a257cc000-7f2a257ed000 r-xp 00000000 fd:00  /usr/lib64/ld-2.17.so
    7f2a259d7000-7f2a259da000 rw-p 00000000 00:00 0 
    //匿名内存映射区域
    7f2a259e6000-7f2a259ed000 rw-p 00000000 00:00 0 

我们看到，使用 `mmap` 开辟的内存相比与 `brk` 处于高地址区域。并且，也没有标识为 `heap`,其实，就是一段没有文件映射的匿名空间，在这种情况下，我们可以拿来作为堆空间。

---
<H4>glibc malloc</H4>
---

到目前为止，我们知道了要想在程序运行时动态的产生内存,需要使用 `brk` 或者 `malloc` 系统调用。我们知道，系统调用的开销是非常大的，并且，`malloc` 分配会有内存对齐。所以，如果程序频繁的使用 这两个系统调用，会严重影响程序性能。为此，通常情况下，我们应该使用 C 运行库(GBLIB) 提供的`malloc`函数。

其实，`malloc` 函数并没不是什么很神秘的东西，只是在应用层对 `brk` 和 `mmap` 进行了封装 和 对申请到的内存进行了有效管理。

其基本思想是：当应用程序通过 `malloc` 进行动态内存分配时，对于小于 `128k` 的大小申请，它会在现有的堆空间里面(如果是首次调用malloc或者现有的堆空间不够，就使用 `brk` 进行系统申请)，按照堆分配算法为它分配一块空间；对于大于128kb的请求，它会直接使用 `mmap()` 函数为其分配一块匿名空间，然后在这个匿名空间中为用户分配空间。并且，当我们调用 `free` 释放时，并没有真正的时放掉，而是将其缓存起来以便后续使用。通过这些手段，可以显著减少系统调用和内存碎片的问题。




