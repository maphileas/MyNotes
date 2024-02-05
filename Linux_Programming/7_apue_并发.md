# 并发

并发涉及的概念

```text
- 同步：
- 异步：

异步时间的处理：查询法（发生频率高）、通知法（发生频率低）

```





并发实现方式：

```
一.信号
	1. 信号的概念
		信号是软件终端
		信号的响应依赖于中断
	2. signal()函数
		void (*signal(int signum,void (*func)(int)))(int);
	3. 信号的不可靠
	4. 可重入函数
		所有的信号调用都是可重入的，一部分库函数也可重入，如：memcpy。
	5. 信号的响应过程
	6. 信号常用函数
		kill()、raise()、alarm()、
		pause()、abort()、system()、
		sleep()、nanosleep()、usleep()
		使用单一计时器，利用alarm或setitimer构造一组函数，实现任意数量的计时器。
	7. 信号
		信号集类型：sigset_t
		sigemptyset()、sigfillset()、sigaddset()、sigdelset()、sigismember()
	8. 信号屏蔽字/pending集的处理
	9. 扩展
		sigsuspend();
		sigaction();  -> signal();//sigaction替代signal函数，signal函数存在很多设计上的缺陷，无法保留之前的信号。
		setitimer();
	10. 实时信号 
二.线程
	
```

# 1.信号的概念

信号是软件中断。

信号的响应依赖于中断。



![2023-05-14_20-21-37.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-14_20-21-37.png)

从信号34开始，信号没有特定的名字。

SIGRT: 代表实时信号。他的范围为 SIGRTMIN~SIGRTMAX



![2023-05-29_14-27-10.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-29_14-27-10.png)

# 2.signal函数

man手册上是

```c
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

但是比较推荐：

```c
#include<signal.h>
void (*signal(int signo,void (*func)(int)))(int);
```



**信号会打断阻塞的系统调用**，比如write、read等阻塞函数

**案例**：signal的使用

```c
#include<stdio.h>
#include<stdlib.h>
#include<signal.h>
#include<unistd.h>

int main()
{
    for (int i = 0; i < 10; i++)
    {
        /* code */
        write(1,"*",1);
        sleep(1);
    }
    
    exit(0);
}
```

终端上会输出10个"*"，按ctrl+c会打断输出。



实现中断时，输出其他信息

```c
#include<stdio.h>
#include<stdlib.h>
#include<signal.h>
#include<unistd.h>

static void int_handler(int s)
{
    write(1,"!",1);
}
int main()
{
    //signal(SIGINT,SIG_IGN);//忽略到中断信号(ctrl+c)
    // signal响应的条件：1.程序没结束；2.信号来了
    
    /*void (*ptr)(int s);
    ptr = int_handler;
    signal(SIGINT,ptr);*/

    signal(SIGINT,int_handler);
    for (int i = 0; i < 10; i++)
    {
        /* code */
        write(1,"*",1);
        sleep(1);
    }
    printf("\n");
    
    exit(0);
}
```

![2023-05-14_21-25-25.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-14_21-25-25.png)

当按下ctrl+c的时候，会输出 "!"。

![2023-05-14_21-36-50.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-14_21-36-50.png)

当打印的时候，一直按着ctrl+c不放，其执行时间要不到10s。是**因为信号会打断阻塞的系统调用**。



# 3.信号的不可靠

信号的不可靠是指行为不可靠，行为不可靠是指的是信号在处理行为的同时，又来了一个相同的信号。（信号的现场一般是由内核布置的，就很有可能不是在同一个位置，第二次的现场可能就把第一次的现场冲了）。

所以信号的不可靠，不是指信号的丢失，而是信号的行为不可靠（第一次信号没结束的时候，就来了第二个信号）。



# 4.可重入函数（reinterview）

- 用途：解决信号不可靠的问题。

- **可重入函数：第一次调用还没结束，就发生了第二次调用，但是不会出错。**

**所有的系统调用都是可重入的**，一部分库函数也是可重入的。**只要xx函数有一个xx_r版本的函数，这个xx函数一定不能用在信号处理中，防止重入这种现象。**

![2023-05-17_23-37-13.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-17_23-37-13.png)

像memcpy这种函数，只要src和dest不相同，就是可以重入的。



![2023-05-17_23-42-10.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-17_23-42-10.png)





# 5.信号的响应过程



- 内核为每个进程维护了两个位图，一个位图叫mask（信号屏蔽字），一个叫pending（英文意思：未决定的，即将发生的） 。理论上，mask和pending都是32位的。 linux的标准信号就是32位的。



- 进程和线程并不分家，从内核的角度来说只有进程，从开发者的角度来说是进程。从整个角度来说，进程是容器，线程是具体的体现。



```text
信号从收到到响应有一个不可避免的延迟。
思考: 如何忽略掉一个信号？
	 标准信号为什么要丢失？
	 标准信号的响应没有严格的顺序。
	 不能从信号处理函数中随意的往外跳。（setjmp，longjmp）
```



# 6.常用函数

```text
kill()
raise()
alarm()：set an alarm clock for delivery of a signal
pause()：wait for signal
abort()
system()
sleep()
```

## 6.1 alarm：

向被调用的进程，以秒为单位发送SIGALRM信号。让进程/线程有一个时间的概念。

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

int main()
{
    alarm(5);
    while(1);
    exit(0);
}
```

![2023-05-27_16-04-27.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-27_16-04-27.png)

5秒后程序回被信号中断而结束。

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

int main()
{
    alarm(10);
    alarm(5);
    alarm(1);
    while(1);
    exit(0);
}
```

此时1秒后，程序就会结束。

**在很多平台sleep是由alarm+pause实现的，所以如果有源码中使用了sleep，可以会与你的alarm()函数冲突，那么结果就不确定。**

**注：**alarm没法实现多任务的计时器。



- 案例：定时循环

  ```c
  #include<stdio.h>
  #include<stdlib.h>
  #include<time.h>
  
  //用time来控制时钟计数
  
  int main()
  {
      time_t end;
      int64_t count = 0;
   
      end = time(NULL) + 5;
  
      while(time(NULL) <= end)
          count++;
  
      printf("%ld\n",count);
      exit(0);
  }
  ```

  ```c
  #include<stdio.h>
  #include<stdlib.h>
  #include<unistd.h>
  #include <signal.h>
  
  // 用alarm来控制时钟计数
  
  static int loop = 1;
  
  
  static void alrm_handler(int s)
  {
      loop = 0;
  }
  int main()
  {
      int64_t count = 0;
  
      alarm(5);
  
      signal(SIGALRM,alrm_handler);
      while(loop)
          count++;
      printf("%ld\n",count);
      exit(0);
  }
  
  ```

  

  分别用time、alarm来实现

  ![2023-05-27_16-38-54.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-05-27_16-38-54.png)

  time中，5.999s和5.001s都认为是5秒，所以用time来计时，其误差比较大，从上面两个运行结果对比也可以知道。

​		**注**：在signal处理函数和alarm函数的顺序，建议将处理函数放在前。

```c
int main()
{
    int64_t count = 0;
    signal(SIGALRM,alrm_handler);
    alarm(5);

    while(loop)
        count++;
    printf("%ld\n",count);
    exit(0);
}
```



​	

## 6.2 setitimer

# 7. 信号集
		信号集类型：sigset_t
		sigemptyset();
		sigfillset();
		sigaddset();
		sigdelset();
		sigismember()



# 8.信号屏蔽字/pending集的处理

## 8.1 sigpromask



- sigpromask：将mask置为0，那么信号在此期间无法响应。（我们不能决定信号什么时候来，但是可以决定信号什么时候响应，sigpromask就是这样的函数）。

  ```c
  #include<stdio.h>
  #include<stdlib.h>
  #include<signal.h>
  #include<unistd.h>
  
  // void (*sighandler_t)(int);
  //希望在打印5个"*"之间不会响应，希望信号在两行之间响应。
  static void int_handler(int s)
  {
      write(1,"!",1);
  }
  int main()
  {
      sigset_t set;
      signal(SIGINT,int_handler);
      sigemptyset(&set);
      sigaddset(&set,SIGINT);
  
  
      for(int j = 0; j < 1000; j++)
      {
          sigprocmask(SIG_BLOCK,&set,NULL);
          for (int i = 0; i < 5; i++)
          {
              write(1,"*",1);
              sleep(1);
          }
          write(1,"\n",1);
          sigprocmask(SIG_UNBLOCK,&set,NULL);
      }
      
      printf("\n");
      exit(0);
  }
  ```

  ```c
  #include<stdio.h>
  #include<stdlib.h>
  #include<signal.h>
  #include<unistd.h>
  
  // void (*sighandler_t)(int);
  //希望在打印5个"*"之间不会响应，希望信号在两行之间响应。
  static void int_handler(int s)
  {
      write(1,"!",1);
  }
  int main()
  {
      sigset_t set,oset;
      signal(SIGINT,int_handler);
      sigemptyset(&set);
      sigaddset(&set,SIGINT);
  
  
      for(int j = 0; j < 1000; j++)
      {
          sigprocmask(SIG_BLOCK,&set,&oset);
          for (int i = 0; i < 5; i++)
          {
              write(1,"*",1);
              sleep(1);
          }
          write(1,"\n",1);
          sigprocmask(SIG_SETMASK,&oset,NULL);
      }
      
      printf("\n");
      exit(0);
  }
  ```


## 8.2 sigpending

用途不大。

```c
#include <unistd.h>
#include <stdio.h>
#include <signal.h>
#include <errno.h>

void printsigset(sigset_t *set)
{
    int i;
    //64个信号太多了，只观察前32个意思意思.进程号从１开始
    for (i = 1; i <=32; i++)
    {
        //判断信号i是否在集合中
        if(sigismember(set,i)==1)
            printf("1 ");
        else
            printf("0 ");
    }
    printf("\n");
}

int main(int argc, char const *argv[])
{
    sigset_t myset,oldset;
    int recv = 0;
    //清空myset信号集
    recv |= sigemptyset(&myset);
    //将要屏蔽的信号添加到信号集中
    recv |= sigaddset(&myset,SIGQUIT);      //ctrl+/产生
    recv |= sigaddset(&myset,SIGINT);       //ctrl+c产生
    recv |= sigaddset(&myset,SIGTSTP);      //ctrl+z产生
    recv |= sigaddset(&myset,SIGKILL);      //kill信号是不能被屏蔽的，这里验证是否能屏蔽
    if(recv!=0)
    {
        perror("sidaddset fun error\n");
        exit(-1);    
    }
    //获取当前未决信号集
    sigpending(&oldset);
    printsigset(&oldset);
    //设置当前进程的屏蔽信号集
    sigprocmask(SIG_BLOCK,&myset,&oldset);
    sleep(9);
    while(1)
    {
        sleep(2);
        //获取当前未决信号集
        sigpending(&oldset);
        printsigset(&oldset);
    }
    return 0;
}
```





# 9.sigsuspend

对于信号驱动需求来说，

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<signal.h>
#include<errno.h>

#define BUFSIZE 2

char buf[BUFSIZE]="!";

static void sig_handler(int signo)
{
    write(1,buf,BUFSIZE);
}
int main()
{
    int i,j;
    sigset_t set,oset;// 信号集
    signal(SIGINT,sig_handler);
    setvbuf(stdout,NULL,_IONBF,0);
    sigemptyset(&set);
    sigaddset(&set,SIGINT);  

    sigprocmask(SIG_BLOCK,&set,&oset);
    for(i=0;i<100;i++)
    {
        for(j=0;j<5;j++)
        {
            write(1,"*",1);
            sleep(1);
        }
        fputs("\n",stdout);
        sigset_t tmpset;
        sigprocmask(SIG_SETMASK,&oset,&tmpset);
        pause();
        sigprocmask(SIG_SETMASK,&tmpset,NULL);
    }

    printf("\n");
    exit(0);
}
```



![2024-01-18_22-18-50](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2024-01-18_22-18-50.png)

这个程序有一个问题，就是在程序停住后，按ctrl+c，能够驱动程序继续执行，但是在屏蔽信号的过程中，按下ctrl+c，程序会响应信号处理函数，但是没有继续驱动程序往下执行，是因为信号在还没有执行到pause的时候就被响应了（在sigprocmask和pause函数之间被响应），所以必须要在按一次ctrl+c，才能继续驱动程

```。c
sigprocmask(SIG_SETMASK,&oset,&tmpset);
pause();
sigprocmask(SIG_SETMASK,&tmpset,NULL);
```

序，原因是因为这个3句话不是原子操作。**要达到这个效果，就可以使用sigsuspend函数。**



```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<signal.h>
#include<errno.h>

// 信号驱动程序
// 在第一次输出5个'*'之后，程序暂定住，直到再次有SIGINT信号才继续输出下一行

static void sig_handler(int signo)
{
    write(1,"!",1);
}
int main()
{
    int i,j;
    sigset_t set,oset;// 信号集
    signal(SIGINT,sig_handler);
    setvbuf(stdout,NULL,_IONBF,0);
    sigemptyset(&set);
    sigaddset(&set,SIGINT);  

    sigprocmask(SIG_BLOCK,&set,&oset);
    for(i=0;i<100;i++)
    {
        for(j=0;j<5;j++)
        {
            write(1,"*",1);
            sleep(1);
        }
        fputs("\n",stdout);
        sigsuspend(&oset);     
    }

    printf("\n");
    exit(0);
}
```

![2024-01-18_22-26-46.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2024-01-18_22-26-46.png)

使用sigsuspend后，在信号屏蔽期间按下ctrl+c，就可以在基础屏蔽后响应，继续驱动程序输出。



# 10.sigaction

用于替换signal函数，signal函数在使用上是存在缺陷的。

```c
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,struct sigaction *oldact);

struct sigaction {
    void     (*sa_handler)(int); // 信号处理函数
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t   sa_mask;// 在信号函数执行期间，指定需要阻塞住的一些信号
    int        sa_flags;
    void     (*sa_restorer)(void);
};
```



1） signal函数在执行信号处理函数的时候，阻塞其他信号的响应。

```c
#include<stdio.h>
#include<stdlib.h>
#include<errno.h>
#include<sys/types.h>
#include<unistd.h>
#include<fcntl.h>
#include<syslog.h>
#include<errno.h>
#include <string.h>

#define FNAME "/tmp/out"

// 守护进程
// 打开一个文件，不停的往文件中写入数字
// 目前还存在缺陷，就是报错都会往标准输出上走。但是守护进程已经脱离控制终端了

int daemonize(void)
{   
    int fd;
    pid_t pid = fork();
    if(pid<0)
        return -1;

    if(pid > 0) // 父进程
    {
        exit(0);
    }

    fd = open("/dev/null",O_RDWR);
    if(fd < 0)
    {
        perror("open()");
        return -2;
    }

    dup2(fd,0);
    dup2(fd,1);
    dup2(fd,2);

    if(fd > 2)
        close(fd);

    setsid();

    chdir("/");//把当前的路径设置到根目录

    return 0;
}

int main()
{
    FILE *fp = NULL;
    int i = 0;

    openlog("mydaemon",LOG_PID,LOG_DAEMON);

    if(daemonize())// 0为成功，非0表示失败
    {
        syslog(LOG_ERR,"daemonize() failed!");
        exit(1);
    }
    else
    {
        syslog(LOG_INFO,"daemonize() successed!");
    }

    fp = fopen(FNAME,"w");
    if(fp == NULL)
    {
        // 这里有点问题，守护进程回脱离控制终端的
        //perror("fopen()");
        syslog(LOG_ERR,"fopen:%s",strerror(errno));
        exit(1);
    }

    syslog(LOG_INFO,"%s was opened.",FNAME);

    while(1)
    {
        fprintf(fp,"%d\n",i++);//fprintf是一个行缓冲模式，但是文件是一个全缓冲模式
        fflush(fp);
        syslog(LOG_DEBUG,"%d is printed.",i);
        sleep(1);// 1s输入一次
    }

    fclose(fp);
    closelog();

    exit(0);
}
```

之前写的mydaemon案例中，下面两句话无法执行到，没法正常做一个收尾的工作，无法释放资源。守护进程只能用kill命令来杀掉。

```c
    fclose(fp);
    closelog();
```

现在规定可以用SIGINT、SIGQUIT、SIGTERM来将进程杀掉。在信号这里，可以通过信号处理函数来进行程序的收尾工作。

```c
static FILE *fp;
static void daemon_exit(int s)
{

    fclose(fp);
    closelog();
    exit(0);
}

int main()
{
    fp = NULL;
    int i = 0;
    signal(SIGINT,daemon_exit);
    signal(SIGQUIT,daemon_exit);
    signal(SIGTERM,daemon_exit);
    
    ...
}
```



**分析**：这三句信号处理的函数，可能会发生重入的可能，会出现这样的现象，信号是可以嵌套来调用的，类似于递归函数，当SIGINT函数进行daemon_exit时，在执行fclose函数时，此时SIGQUIT信号被捕捉到，也会进入daemon_exit函数，这样fclose函数可能会执行多次，会出现内存泄漏的报错。希望在响应信号的时候，把其他信号阻塞住。而signal函数不具备这个功能。





所以在多个信号共用同一个信号处理函数的时候，在响应某一个信号期间，可以吧其他信号block住。而这个功能signal不具备，就可以使用sigaction。



改进后的mydaemon案例：

```c
#include<stdio.h>
#include<stdlib.h>
#include<errno.h>
#include<sys/types.h>
#include<unistd.h>
#include<fcntl.h>
#include<syslog.h>
#include<errno.h>
#include <string.h>
#include<signal.h>

#define FNAME "/tmp/out"

// 守护进程
// 打开一个文件，不停的往文件中写入数字
// 目前还存在缺陷，就是报错都会往标准输出上走。但是守护进程已经脱离控制终端了。
static FILE *fp;

int daemonize(void)
{   
    int fd;
    pid_t pid = fork();
    if(pid<0)
        return -1;

    if(pid > 0) // 父进程
    {
        exit(0);
    }

    fd = open("/dev/null",O_RDWR);
    if(fd < 0)
    {
        perror("open()");
        return -2;
    }

    dup2(fd,0);
    dup2(fd,1);
    dup2(fd,2);

    if(fd > 2)
        close(fd);

    setsid();

    chdir("/");//把当前的路径设置到根目录

    return 0;
}

static void daemon_exit(int s)
{

    fclose(fp);
    closelog();
    exit(0);
}

int main()
{
    //FILE *fp = NULL;
    fp = NULL;
    int i = 0;

    struct sigaction sa;
    sa.sa_handler = daemon_exit;
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask,SIGQUIT);
    sigaddset(&sa.sa_mask,SIGTERM);
    sigaddset(&sa.sa_mask,SIGINT);
    sa.sa_flags = 0;
    
    sigaction(SIGINT,&sa,NULL);
    sigaction(SIGQUIT,&sa,NULL);
    sigaction(SIGTERM,&sa,NULL);

    openlog("mydaemon",LOG_PID,LOG_DAEMON);

    if(daemonize())// 0为成功，非0表示失败
    {
        syslog(LOG_ERR,"daemonize() failed!");
        exit(1);
    }
    else
    {
        syslog(LOG_INFO,"daemonize() successed!");
    }

    fp = fopen(FNAME,"w");
    if(fp == NULL)
    {
        // 这里有点问题，守护进程回脱离控制终端的
        //perror("fopen()");
        syslog(LOG_ERR,"fopen:%s",strerror(errno));
        exit(1);
    }

    syslog(LOG_INFO,"%s was opened.",FNAME);

    while(1)
    {
        fprintf(fp,"%d\n",i++);//fprintf是一个行缓冲模式，但是文件是一个全缓冲模式
        fflush(fp);
        syslog(LOG_DEBUG,"%d is printed.",i);
        sleep(1);// 1s输入一次
    }

    fclose(fp);
    //closelog();

    exit(0);
}
```



2）信号处理函数无法分辨信号来源，即区分信号来源于用户态还是kernel态，无法做到只响应特定途径的信号。

```c
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,struct sigaction *oldact);

struct sigaction {
    void     (*sa_handler)(int); // 信号处理函数
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t   sa_mask;// 在信号函数执行期间，指定需要阻塞住的一些信号
    int        sa_flags;
    void     (*sa_restorer)(void);
};
当sa_flags=SA_SIGINFO的时候，执行三参数的信号函数sa_sigaction。用三参的信号处理函数，在接到信号，进行信号处理的时候，
可以获取信号的属性，比如从哪里来等等。
    
```



```c
这个结构体里面包含了收到信号的所有信息。
siginfo_t {
               int      si_signo;     /* Signal numbe.3r */ 所有平台都有
               int      si_errno;     /* An errno value */ 所有平台都有
               int      si_code;      /* Signal code */  所有平台都有，记录了当前信号的来源。
               int      si_trapno;    /* Trap number that caused
                                         hardware-generated signal
                                         (unused on most architectures) */
               pid_t    si_pid;       /* Sending process ID */
               uid_t    si_uid;       /* Real user ID of sending process */
               int      si_status;    /* Exit value or signal */
               clock_t  si_utime;     /* User time consumed */
               clock_t  si_stime;     /* System time consumed */
               sigval_t si_value;     /* Signal value */
               int      si_int;       /* POSIX.1b signal */
               void    *si_ptr;       /* POSIX.1b signal */
               int      si_overrun;   /* Timer overrun count;
                                         POSIX.1b timers */
               int      si_timerid;   /* Timer ID; POSIX.1b timers */
               void    *si_addr;      /* Memory location which caused fault */
               long     si_band;      /* Band event (was int in
                                         glibc 2.3.2 and earlier) */
               int      si_fd;        /* File descriptor */
               short    si_addr_lsb;  /* Least significant bit of address
                                         (since Linux 2.6.32) */
               void    *si_lower;     /* Lower bound when address violation
                                         occurred (since Linux 3.19) */
               void    *si_upper;     /* Upper bound when address violation
                                         occurred (since Linux 3.19) */
               int      si_pkey;      /* Protection key on PTE that caused
                                         fault (since Linux 4.6) */
               void    *si_call_addr; /* Address of system call instruction
                                         (since Linux 3.5) */
               int      si_syscall;   /* Number of attempted system call
                                         (since Linux 3.5) */
               unsigned int si_arch;  /* Architecture of attempted system call
                                         (since Linux 3.5) */
}

```



在mytbf函数中，使用sigaction函数对信号的来源进行分辨。

```text
X:\marz\cpp\apue\parallel\signal\mytbf_sa
```



# 11.实时信号

标准信号有两个特点：

​	1）标准信号会丢失

​	2）标准信号的响应没有一个严格的顺序



34~64开始，都为实时信号。**实时信号没有默认的动作。**

![2024-02-04_22-23-29.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2024-02-04_22-23-29.png)



实时信号驱动的案例：

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<signal.h>

#define MYRTSIG (SIGRTMIN+6)  // 定义了一个实时信号

// 用实时信号来实现
static void mysig_handler(int s)
{
    write(1,"!",1);
}


int main()
{   
    int i,j;
    sigset_t set,oset,saveset;//oset = old set

    signal(MYRTSIG,mysig_handler);
    sigemptyset(&set);
    sigaddset(&set,MYRTSIG);

    sigprocmask(SIG_UNBLOCK,&set,&saveset); //移除所有阻塞的信号，之前的信号都会存放在saveset中

    for(i=0;i<1000;i++)
    {
        sigprocmask(SIG_BLOCK,&set,&oset); //将老的信号存在oset中，设置set中的信号为阻塞信号
        for(j=0;j<5;j++)
        {
            write(1,"*",1);
            sleep(1);
        }
        write(1,"\n",1); 
        sigsuspend(&oset);
    }
    sigprocmask(SIG_SETMASK,&saveset,NULL);
    exit(0);
}
```

![2024-02-05_20-51-25.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2024-02-05_20-51-25.png)

![2024-02-05_20-51-57.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2024-02-05_20-51-57.png)



![2024-02-05_20-54-51.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2024-02-05_20-54-51.png)
