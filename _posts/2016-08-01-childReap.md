---
title: "进程的回收"
data: "2016-08-01"
tags： [linux]
---

---
父进程通过 `fork` 函数可以创建出子进程来，虽然当 `fork` 函数返回时父子进程已经是两个独立的逻辑流，并且有着自己独立的地址空间，但我们仍然提供了一种机制，终止进程能通知其父进程它是如何终止的。

如果子进程在父进程之前终止，父进程又如何得到终止子进程的信息呢？ 对此：内核为每个终止子进程保存了一定量的信息，当终止进程的父进程调用 `wait` 和 `waitpid` 时，可以得到这些信息，包括进程ID，该进程的终止状态，以及该进程使用的CPU时间总量。 在UNIX术语中，一个已终止，但其父进程尚未对其进行善后处理(获取终止子进程的有关信息，释放它仍占用的资源) 的进程被称为 `僵死进程(zombie)`。

---
<br/>
<h3>wait 和 waitpid</h3>
当一个进程正常或异常终止时，内核就向其父进程发送 `SIGCHID` 信号。父进程可以选择忽视该信号，或者提供一个该信号发生时即被调用执行的函数。

    #include<sys/wait.h>
    
    pid_t waitpid(pid_t pid, int *statloc, int options);

`waitpid` 函数有点复杂，默认地（当options = 0 时），`waitpid` 挂起调用进程的执行，直到它的 `等待集合` 中的一个子进程终止。`waitpid` 返回导致它返回的已终止子进程的PID，**并且将这个已终止的子进程从系统中去除。**所以，`waitpid` 函数有 `回收(reap)` 已终止的子进程在内核中遗留的状态信息。

---
<br/>
<h5>1.判定等待集合：</h5>

`waitpid` 函数的 `等待集合` 由它的第一个参数 `pid` 确定：

 - 如果 `pid > 0`，那么等待集合就是一个单独的子进程，它的进程ID等于pid。
 - 如果 `pid = -1`，那么等待集合就是由父进程所有的子进程组成的。

---
<br/>
<h5>2.检查已回收子进程的退出状态:</h5>
 如果 `status` 参数非空，那么 `waitpid` 就会在 `status` 参数中放上关于导致返回的子进程的状态信息。下面由几个 `宏` 可以解释 `status` 参数：
 
- WIFEXITED(status)：如果子进程通过调用 `exit` 或者一个返回(return) 正常终止，就返回真。
- WEXITSTATUS(status)：返回一个正常终止的子进程的退出状态。只有在 `WIFEXITED` 返回为真时，才会返回这个状态。

<br/>
ok，下面的一个 c 程序演示了 `waitpid` 的使用，回收僵死子进程。

    #include<unistd.h>
    #include<sys/wait.h>
    #include<sys/types.h>
    
    int main(){
        int status, i;
        pid_t pid;
        
        for(i = 0; i < N; i++)
            if((pid = fork()) == 0)
                exit(100+i); //每个子进程以唯一的退出状态退出
        
        //会阻塞调用
        while((pid = waitpid(-1, &status, 0)) >  0){
            if(WIFEXITED(status))
                printf("child %d terminated normally 
                        with exit status=%d\n", pid,
                            WEXITSTATUS(status));
            else
                printf("child %d terminated abnormally\n", pid);
        }
        
        exit(0);
    }
    
上面的程序通过 `exit` 为每个子进程设置了唯一的退出状态值，只要子进程是正常退出的， `WEXITSTATUS(status)` 就可以得到其退出的状态值。


ok，我们直到了，一个进程退出后在内核仍会保留一些状态信息，以便父进程可以得到这些信息。为了清空这些信息，父进程必须等待子进程终止并显示的回收它。但如果父进程先于子进程退出又会怎么样呢？

---
<br/>
<h3>init进程：</h3>

对于父进程已经终止的所有进程，它们的父进程都改变为 `init进程`。我们称这些进程由 `init进程` 领养。当一个进程终止时，内核逐个检查所有活动进程，以判断它是否是正要终止进程的子进程，如果是，则将该进程的父进程ID更改为1（init进程的ID）。 这种处理方法保证了每个进程都有一个父进程。

并且， `init进程`会自动回收它的以终止子进程的状态信息。

下面实例展示了 `init进程`的回收：
    
    int main(){
        pid_t pid;
        if(fork() == 0){
            sleep(2);
            printf("child id:%d\n", getppid());
            exit(0);
        }
        exit(0);
    }
    
    //output:
    child id:1
    
上面，子进程通过 `sleep`来等待父进程的终止，并通过 `getppid` 得到 `1`,说明它已经被 `init进程` 领养。
