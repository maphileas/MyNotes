# 1.进程基本知识

**已经进入多进程阶段**，本节所涉及的内容如下：

```text
1. 进程标识符pid
2. 进程的产生
	fork()/vfork()
3. 进程的消亡及释放资源    
4. exec函数族
5. 用户权限及组权限
6. 观摩课：解释器文件
7. system()
8. 进程会计
9. 进程时间
10. 守护进程
11. 系统日志 
```



## 1.1 进程标识符pid

类型：pid_t（传统来讲是一个有符号的整形数）

命令：ps

进程号是顺次向下使用的

getpid()：获取当前进程号

getppid()： 获取父进程的号



## 1.2 进程的产生

### 1. 2.1 fork();

```text
fork() creates a new process by duplicating the calling process.
duplicate意味着copy、克隆、一模一样的意思。

fork后父子进程的区别：
	fork的返回值不一样，pid不同，ppid也不同，
	未决（悬而未决）信号和文件锁不继承，资源利用量归零。
	
init进程：1号，是所有进程的祖先进程。（因为其他进程不一定是由init进程fork得到的，可能是由init产生的进程得到的）

调度去的调度策略来决定那个进程先运行

fflush的重要性
```

- **测试**：

```c
#include<sys/types.h>
#include<unistd.h>
#include<errno.h>

int main()
{
    pid_t pid;
    printf("[%d]:Begin!\n",getpid());
    pid = fork();
    if(pid < 0)
    {
        perror("fork()");
        exit(1);
    }

    if(pid == 0) // child 子进程
    {
        printf("[%d]:Child is working!\n",getpid());
    }
    else // parent
    {
        printf("[%d]:Parent is working!\n",getpid());
    }


    printf("[%d]:End!\n",getpid());
    exit(0);
}
```

![2023-04-20_22-31-15.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_22-31-15.png)

执行两次的结果，**<u>但是值得一提的是</u>：** **父子进程运行的先后顺序是由调度器的策略来决定的。** 

这个程序有一下结论：

1. 如果想要子进程限制性，可以通过sleep来实现。

```c
if(pid == 0) // child 子进程
{
    printf("[%d]:Child is working!\n",getpid());
}
else // parent
{
    sleep(1);
    printf("[%d]:Parent is working!\n",getpid());
}
```

![2023-04-20_22-37-26.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_22-37-26.png)

2. 如果想程序暂停未结束的话，加上getchar，

```c
    if(pid == 0) // child 子进程
    {
        printf("[%d]:Child is working!\n",getpid());
    }
    else // parent
    {
        sleep(1);
        printf("[%d]:Parent is working!\n",getpid());
    }

    printf("[%d]:End!\n",getpid());

    getchar();
```

然后使用 "ps axf"，可以查看进程关系，是一个阶梯状的进程关系。这个阶梯状的进程关系这样来理解：



![2023-04-20_22-47-57.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_22-47-57.png)

当前的shell创建了fork1进程，在fork1进程的执行过程中，又产生了一个进程。

![2023-04-20_22-43-47.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_22-43-47.png)

像 “sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups”这种顶格的进程，其父进程都是init（1号进程）。**但不是1号进程fork出所有的进程的。**

所以1号进程是所有进程的祖先进程，而不能认为是所有进程的父进程。

3. 问题：begin只打印一次，但是但我们重定向的时候，begin会出现两次

   ![2023-04-20_22-56-51.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_22-56-51.png)

当我将源程序改成这样的时候，将Begin后面的换行符去掉，得到的打印结果

```c
pid_t pid;
printf("[%d]:Begin!",getpid());
pid = fork();
```

![2023-04-20_23-02-18.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_23-02-18.png)

![2023-04-20_23-04-12.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_23-04-12.png)

**解决思路： 在fork之前，应该刷新所有成功打开的流**。

```c
pid_t pid;
printf("[%d]:Begin!",getpid());
fflush(NULL); // ！！！这句话非常重要，实际生产经验。
pid = fork();
if(pid < 0)
{
    perror("fork()");
    exit(1);
}
```

**为什么终端打印一个begin，重定向到文件就是两个begin？：**

终端是标准输出，标准输出设备默认是行缓冲模式，所以”\n“会刷新缓冲区。而文件默认是全缓冲模式，”\n“只是换行符。Begin在放到缓冲区中，还没来得及刷新的时候就fork了，这样父子进程的缓冲去中就各自有了一个Begin，而Begin的进程号已经固定了。

**小功能：输出质数**

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

#define LEFT 30000000
#define RIGHT 30000200

// 求质数（30000000~30000200）单机版
int main()
{
    int mark = -1;
    for(int i=LEFT;i<=RIGHT;i++)
    {
        mark = 1;
        for(int j=2;j<=i/2;j++)
        {
            if(i%j == 0)
            {
                mark = 0;
                break;
            }
        }
        if(mark)
            printf("%d is a primer\n",i);
    }
    exit(0);
}
```



![2023-04-21_20-49-27.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-21_20-49-27.png)

**小功能：输出质数(多进程版)**

```c
#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<unistd.h>
#include<errno.h>

#define LEFT 30000000
#define RIGHT 30000200

// 求质数（30000000~30000200）
// 版本1：用201个子进程来计算201个待计算的任务
int main()
{
    int i,j,mark;
    pid_t pid;
    for(i=LEFT;i<=RIGHT;i++)
    {
        pid = fork();
        if(pid < 0)
        {
            perror("fork()");
            exit(1);
        }

        if(pid == 0)// 子进程
        {
            mark = 1;
            for(j=2;j<=i/2;j++)
            {
                if(i%j == 0)
                {
                    mark = 0;
                    break;
                }
            }
            if(mark)
                printf("%d is a primer\n",i);
            exit(0); // 注意这里，每个子进程打印之后都要正常退出。
        } 
    }
    exit(0);
}
```



![2023-04-21_21-09-53.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-21_21-09-53.png)

但是减少的时间并不成比例，这与线程调度与机器的core数相关。

![2023-04-21_21-13-58.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-21_21-13-58.png)

#### 1.2.1.1 进程中的僵尸进程和孤儿进程

- 当父进程sleep(1000)的时候，

```c
#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<unistd.h>
#include<errno.h>

#define LEFT 30000000
#define RIGHT 30000200

// 求质数（30000000~30000200）
// 版本1：用201个子进程来计算201个待计算的任务
int main()
{
    int i,j,mark;
    pid_t pid;
    for(i=LEFT;i<=RIGHT;i++)
    {
        pid = fork();
        if(pid < 0)
        {
            perror("fork()");
            exit(1);
        }

        if(pid == 0)// 子进程
        {
            mark = 1;
            for(j=2;j<=i/2;j++)
            {
                if(i%j == 0)
                {
                    mark = 0;
                    break;
                }
            }
            if(mark)
                printf("%d is a primer\n",i);
            //sleep(1000);
            exit(0);
        } 
    }
    sleep(1000);
    exit(0);
}
```

![2023-04-21_21-34-13.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-21_21-34-13.png)

当前shell中，primer1 创建了201个子进程，父进程的状态是"S+"，可中断的睡眠态，每个子进程的状态是“Z+”，僵尸状态，处于不正常的状态。但是**在进程关系中，出现zombie态是非常正常的。**如果父进程正常退出，这些僵尸进程就会变成孤儿进程。

僵尸进程所占用的资源很少，一个进程的信息就存在一个结构体中，这个结构体中肯定有pid号，而pid号是一个宝贵的资源，因为pid号有限。

- 当子进程sleep(1000)秒的时候

  ```c
  int main()
  {
      int i,j,mark;
      pid_t pid;
      for(i=LEFT;i<=RIGHT;i++)
      {
          pid = fork();
          if(pid < 0)
          {
              perror("fork()");
              exit(1);
          }
  
          if(pid == 0)// 子进程
          {
              mark = 1;
              for(j=2;j<=i/2;j++)
              {
                  if(i%j == 0)
                  {
                      mark = 0;
                      break;
                  }
              }
              if(mark)
                  printf("%d is a primer\n",i);
              sleep(1000);
              exit(0);
          } 
      }
      //sleep(1000);
      exit(0);
  }
  ```

  ![2023-04-21_21-36-56.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-21_21-36-56.png)

这个时候子进程处于可中断的睡眠态。因为父进程会正常退出(exit(0))，所以这些子进程就变成了孤儿进程，孤儿进程将由init进程接管，init进程等子进程exit(0)后进行收尸。所以通过“ps axf”命令查看，./primer1子进程会顶格显示。

#### 1.2.1.2 收尸

**在linux中，始终有一个原则，谁打开谁释放。子进程可以看作父进程创建出来的资源，所以父进程在最后一定要释放申请出来的资源。**所以父子进程之间有一个收尸环节。

上面的程序还缺少收尸的模块，应该是父进程创建完子进程，子进程干活期间或者干完活前，父进程在等待着，等子进程状态终止，然后把子进程回收掉。

**收尸需要关心两件事**：

1. 是否关心子进程退出状态？关心的话，应该冲僵尸进程中把它的状态取过来。
2. 释放pid。因为进程pid_t是一个longlong类型的整数，存在上限。



### 1.2.2 vfork

fork()函数可能存在这样的场景，父进程导入30w条数据，子进程给我打印一个“hello world”然后退出。我们知道，fork创建子进程，子进程会copy一份和父进程完全一样的内容。·

<img src="https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/20230423071055.jpg" alt="20230423071055.jpg" style="zoom:50%;" />

父进程通过fork产生子进程，数据内容之间是通过memcpy来实现的。父进程的指针指向一块虚拟地址内存（也会有一块实际的物理地址），子进程的指针也会指向一块虚拟地址，而子进程虚拟地址的物理地址是通过memcpy父进程的物理地址来实现的。  

而使用vfork最大的区别就是其子进程和父进程共用底层的物理地址。

问题：在父进程中有一个文件描述符fd，这个时候vfork了一个子进程，问：在子进程中执行close(fd)，这个文件会不会关闭？

现在的fork（更改过），**fork的时候，父进程和子进程共用的是同一个数据模块，如果父子进程对这些模块是只读不写的，那些数据模块谁都不会改变。**如果其中一个进程企图通过指针去修改内容，那么会将数据模块memcpy一份（父进程修改，父进程memcpy一份，子进程修改，子进程memcpy一份，**写时拷贝技术**），改自己的copy出来的那一份数据，而子进程的指向还是不变。一句话，谁改谁copy。父进程通过fork出来一个子进程，他们就是独立的两个进程空间，谁也不能访问轻易去对方空间。这就是fork后面加的的**写时拷贝**技术，把vfork的功能加了进来，所以vfork基本不用了。





## 1.3 进程的消亡及释放资源

- 涉及函数

  ```text
  wait()
  waitpid()
  waitid()
  wait3()
  wait4()
  ```

  ```c
  #include<stdio.h>
  #include<stdlib.h>
  #include<sys/types.h>
  #include<unistd.h>
  #include<errno.h>
  #include <sys/wait.h>
  
  #define LEFT 30000000
  #define RIGHT 30000200
  
  // 求质数（30000000~30000200）
  // 版本1：用201个子进程来计算201个待计算的任务
  int main()
  {
      int i,j,mark;
      pid_t pid;
      for(i=LEFT;i<=RIGHT;i++)
      {
          pid = fork();
          if(pid < 0)
          {
              perror("fork()");
              exit(1);
          }
  
          if(pid == 0)// 子进程
          {
              mark = 1;
              for(j=2;j<=i/2;j++)
              {
                  if(i%j == 0)
                  {
                      mark = 0;
                      break;
                  }
              }
              if(mark)
                  printf("%d is a primer\n",i);
              exit(0);
          } 
      }
  
      for(i = LEFT;i<=RIGHT;i++)
          wait(NULL);
      exit(0);
  }
  ```

### 1.3.1 进程分配之交叉分配法的实现

```c
#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<unistd.h>
#include<errno.h>
#include <sys/wait.h>

#define LEFT 30000000
#define RIGHT 30000200
#define N 3
//进程之交叉分配法

int main()
{
    int i,j,mark,n;
    pid_t pid;

    for(n=0;n<N;n++)
    {
        pid = fork();
        if(pid < 0)
        {
            perror("fork()");
            //这里应该进行收尸处理，如果第三个fork失败的话，应该对前面的进程收尸。
            for(int m = 0;m<n;m++)
            {
                wait(NULL);
            }
            exit(1);
        }

        if(pid == 0)
        {
            for(i=LEFT+n;i<=RIGHT;i+=N)
            {
                
                mark = 1;
                for(j=2;j<=i/2;j++)
                {
                    if(i%j == 0)
                    {
                        mark = 0;
                        break;
                    }
                }
                if(mark)
                    printf("[%d]:%d is a primer\n",n,i);
                      
            }
            exit(0); 
        }
        
    }
    

    for(n = 0;n<N;n++)
        wait(NULL);
    exit(0);
}
```

![2023-04-25_06-23-52.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-25_06-23-52.png)





## 1.4 exec函数族

```text
execl()
execlp()
execle()
execv()
execvp()
The  exec() family of functions replaces the current process image with a new process image.
```
- 案例：

  ```c
  #include<stdio.h>
  #include<stdlib.h>
  #include <unistd.h>
  
  int main()
  {
      puts("Begin!");
      fflush(NULL);  /*!!!*/
      
      execl("/bin/date","date","+%s",NULL);
      perror("execl()");
      exit(1);
      
      puts("End!");
      return 0;
  }
  ```

  ![2023-12-30_22-38-28.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-12-30_22-38-28.png)

  execl函数，用一个新的进程来替换原来的进程，所以不出错的话，永远不会执行到 "put("End!")"。

  在程序中未使用fflush()函数的情况下，将结果重定向到文件中：

  ![2023-12-30_22-47-25.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-12-30_22-47-25.png)

  "Begin!"不见了。加了fflush函数后，

  ![2023-12-30_22-50-50.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-12-30_22-50-50.png)

- fork+exec+wait

  ```c
  #include<stdio.h>
  #include<stdlib.h>
  #include<sys/types.h>
  #include<unistd.h>
  #include<errno.h>
  #include <sys/wait.h>
  
  // fork exec wait函数的完整使用
  
  int main()
  {
      pid_t pid;
  
      puts("Begin!");
      fflush(NULL);
  
      pid = fork();
      if(pid < 0)
      {
          perror("fork()");
          exit(1);
      }
      if(pid == 0)// 子进程干活， 开一个子进程干活
      {
          execl("/bin/date","date","+%s",NULL);
          perror("execl");
          exit(1);
      }
  
      wait(NULL);
      puts("End!");
      exit(0);
  }
  ```

  





## 1.5 用户权限及组权限

主要是（u+s）和（g+s）怎么实现的

```text
u+s
g+s
getuid();
geteuid();
getgid();
getegid();

setuid();
setgid();

setreuid();
setregid();

seteuid();
setegid();
```



## 1.6 解释器文件

![2024-01-02_20-32-12.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2024-01-02_20-32-12.png)

更改头文件，会自动找相关的软件进行解析。

## 1.7 system函数

system相当于fork+exec+wait的封装。

```c
#include<stdio.h>
#include<stdlib.h>

int main()
{
    system("date +%s > /tmp/out");
    exit(0);
}
```



## 1.8 进程会计

进程会计这个机制只不过是一个方言。无需多关注。

```text
NAME
       acct - switch process accounting on or off

SYNOPSIS
       #include <unistd.h>

       int acct(const char *filename);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):
```





## 1.9 进程时间

times()

```text
NAME
       times - get process times

SYNOPSIS
       #include <sys/times.h>

       clock_t times(struct tms *buf);

DESCRIPTION
       times() stores the current process times in the struct tms that buf points to.  The struct tms is as defined in <sys/times.h>:

           struct tms {
               clock_t tms_utime;  /* user time */
               clock_t tms_stime;  /* system time */
               clock_t tms_cutime; /* user time of children */
               clock_t tms_cstime; /* system time of children */
           };
```

## 1.10 守护进程

- 会话（session），标识 sid

  - 终端：
  
  - 涉及函数
  
    ```text
    setsid()
    getpgrp()
    getpgid()
    setpgid()
    单实例守护进程：锁文件，以.pid结尾，比如/var/run/sshd.pid。
    启动脚本文件： /etc/rc*
    ```
    
    
  

```text
setsid() creates a new session if the calling process is not a process group leader.  The calling process is the leader of the new session (i.e., its session ID is made the same as its process ID).  The calling process also becomes the process group leader of a new process group in the session (i.e., its process group ID is made the same as its  process ID). nitially, the new session has no controlling terminal.
```

父进程在产生子进程的时候，父进程就是这个进程组的leader，setsid函数不能够由父进程来调用，只能由子进程来调用。



![2023-05-07_22-00-53.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-07_22-00-53.png)

PPID：父进程ID

PID：进程标识

PGID：进程组ID

SID：会话ID

TTY：控制终端

TPGID：top值

STAT：状态



守护进程脱离控制终端，所以TTY是个“?”，下图这种是不脱离的进程的终端状态。

![2023-05-07_22-04-25.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-07_22-04-25.png)



一般的进程在运行的时候，其父进程会等待进程运行完毕，然后“收尸”。setsid由子进程调用后，按正常运行，其父进程要等待子进程运行完毕为其“收尸”，但是守护进程是需要一直后台运行的，父进程无法收尸（父进程会等待，父进程的父进程会等待，等等），所以当调用setsid的时候，父进程会退出，也就是**守护进程不需要收尸**。如果父进程退出了，那么当前守护进程的PPID值一定为1。

![2023-05-07_22-14-21.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-07_22-14-21.png)



![2023-05-07_22-17-14.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-07_22-17-14.png)

守护进程，PID、PGID、SID相同，PPID = 1；









## 1.11 系统日志

有必要写系统日志，当时又不能让人人都写（格式怎么样？怎么来写？），所以就做了一个权限分割层。

syslogd服务: 所有要写系统日志的，都把要写的内容提交给syslogd这项服务，由syslogd这项服务来采集所有内容，统一来写系统日志。

![2023-05-09_21-19-02.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-09_21-19-02.png)



- 涉及的函数：

  ```text
  #include <syslog.h>
  
  void openlog(const char *ident, int option, int facility); 打开系统日志的链接
  void syslog(int priority, const char *format, ...); 产生系统日志
  void closelog(void);关闭用来写系统日志的文件
  ```

  

ubuntu中，系统日志文件是 /var/log/syslog，系统日志的配置文件是在/etc路径下，有一个“rsyslog.conf”的文件。







