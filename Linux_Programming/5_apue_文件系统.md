# 文件系统

目的：类ls的实现，如myls
 ```text
  ls -n
  ls -l 
  对应 /etc/passwd 和 /etc/group两个函数
 ```

# 1.目录和文件

## 1.1 获取文件属性

- stat：通过文件路径获取属性，面对符号链接文件时获取的是所指向的目标文件。
- fstat：通过文件描述符获取属性
- lstat：面对符号链接文件时获取的是符号链接文件的属性

- stat 、fstat、lstat函数

  ```c
  NAME
         stat, fstat, lstat, fstatat - get file status
  
  SYNOPSIS
         #include <sys/types.h>
         #include <sys/stat.h>
         #include <unistd.h>
  
         int stat(const char *pathname, struct stat *statbuf);
         int fstat(int fd, struct stat *statbuf);
         int lstat(const char *pathname, struct stat *statbuf);
  DESCRIPTION
         These  functions return information about a file, in the buffer pointed to by statbuf.  No permissions are required on the file itself, but—in the case of stat(), fstatat(), and lstat()—execute (search) permission is required on all of the directories in pathname that lead to the file.
         stat() retrieve information about the file pointed to by pathname;
         lstat() is identical to stat(), except that if pathname is a symbolic link, then it returns information about the link itself, not the file that it refers to.
         fstat() is identical to stat(), except that the file about which information is to be retrieved is specified by the file descriptor fd。
  
  The stat structure           
  	struct stat {
                 dev_t     st_dev;         /* ID of device containing file */
                 ino_t     st_ino;         /* Inode number */
                 mode_t    st_mode;        /* File type and mode */
                 nlink_t   st_nlink;       /* Number of hard links */
                 uid_t     st_uid;         /* User ID of owner */
                 gid_t     st_gid;         /* Group ID of owner */
                 dev_t     st_rdev;        /* Device ID (if special file) */
                 off_t     st_size;        /* Total size, in bytes */
                 blksize_t st_blksize;     /* Block size for filesystem I/O */
                 blkcnt_t  st_blocks;      /* Number of 512B blocks allocated */
  
                 /* Since Linux 2.6, the kernel supports nanosecond
                    precision for the following timestamp fields.
                    For the details before Linux 2.6, see NOTES. */
  
                 struct timespec st_atim;  /* Time of last access */
                 struct timespec st_mtim;  /* Time of last modification */
                 struct timespec st_ctim;  /* Time of last status change */
  
                 #define st_atime st_atim.tv_sec      /* Backward compatibility */
                 #define st_mtime st_mtim.tv_sec
                 #define st_ctime st_ctim.tv_sec
             };
  ```

**小功能**：利用stat结构体重的 st_size来得到一个文件的大小

```c
#include<stdio.h>
#include<stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

/**
 *  通过stat structure，返回文件的长度
*/
static off_t flen(const char *fname)
{
    struct stat statres;

    if(stat(fname,&statres) < 0)
    {
        perror("stat()");
    }
    
    return statres.st_size;
}
int main(int argc,char** argv)
{   
    if(argc < 2)
    {
        fprintf(stderr,"Usage:%s <filename>...\n",argv[0]);
        exit(1);
    }

    printf("%ld\n",flen(argv[1]));
    exit(0);
}
```

![2023-03-29_22-25-56.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-03-29_22-25-56.png)

- 通过stat命令也可以得到文件的信息：

  ![2023-03-29_22-21-51.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-03-29_22-21-51.png)

  size是st_size的数值，我们的小功能函数也验证了这一点。**值得注意的是**，st_size仅仅一个参数，类似st_ino这样，文件的实际存储大小是由

  st_blocks*st_blksize的大小决定的，这和文件系统有关系（linux是这样的，win不太一样）。

  文件系统块大小：

  ![2023-03-29_22-31-36.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-29_22-31-36.png)

  

**小功能：**用程序生一个st_size非常大，但是磁盘空间非常小的文件。

```c
#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<unistd.h>
#include<fcntl.h>

/**
 * 
 * 做一个totalsize非常大，但是磁盘空间非常小的文件
 * 做一个5G大小的文件
*/
int main(int argc,char **argv)
{
    int fd;
    off_t res;
    if(argc < 2)
    {
        fprintf(stderr,"Usage:%s <filename>\n",argv[0]);
        exit(0);
    }

    fd = open(argv[1],O_WRONLY|O_CREAT|O_TRUNC,0600);
    if(fd < 0)
    {
        perror("open");
        exit(1);
    }

    res = lseek(fd,5L*1024L*1024L*1024L-1L,SEEK_SET);
    if(res< 5L*1024L*1024L*1024L-1L)
    {
        perror("lseek");
        exit(1);
    }

    write(fd,"",1);
    close(fd);
    exit(0);
}
```

![2023-03-29_23-01-27.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-03-29_23-01-27.png)

这样就生成了一个size很大，但是占用内存仅4k的一个文件。

这个程序的注意点：

```text
输入5*1024*1024*1024-1 会报数据溢出的警告，所以将数据扩张到Long类型，所以就需要在后面跟上L
5L*1024L*1024L*1024L-1L
```

**<u>总结</u>**：linux环境下，size值只是一个属性而已。

## 1.2 st_mode

__st_mode是一个16位的位图，用于表示文件类型、文件访问权限及特殊权限位。__

![2023-03-30_20-28-33.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-03-30_20-28-33.png)

前面的权限就是由st_mode来决定的。

文件类型：dcb-lsp

```text
d:directory 
c:字符设备文件
b:block，块设备文件
-：普通文件（regular fil
l:link，符号链接(symbol link)文件
s:socket file，网络套接字文件
p:pipe，管道文件，在这里特指的是匿名管道文件，匿名管道文件在磁盘上看不到。
```

<img src="https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-03-30_20-47-21.png" alt="2023-03-30_20-47-21.png" style="zoom:50%;" />



![2023-04-05_14-32-10.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-05_14-32-10.png)



![2023-04-05_14-27-58.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-05_14-27-58.png)



## 1.3 umask

​	作用：为了防止产生权限过松的文件。

## 1.4 文件权限的更改/管理

 - chmod
 - fchmod

## 1.5 粘住位

最原始的定义：给某一个可执行的二进制文件设置t位，在内存中保留它使用的痕迹。

t位：(了解)

![2023-04-05_14-47-25.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-05_14-47-25.png)

## 1.6 文件系统：FAT，UFS

文件系统：文件或数据的存储和管理。



## 1.7硬链接，符号链接

![2023-04-05_23-10-23.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-05_23-10-23.png)

- 硬链接

```text
ln [参数] [源文件或目录] [目标文件或目录]
命令的功能是为某一个文件在另外一个位置建立一个同步的链接
```

使用 “ln bigfile bigfile_link” 创建连接

![2023-04-05_23-13-30.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-05_23-13-30.png)



![2023-04-05_23-15-08.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-05_23-15-08.png)

通过ln bigfile bigfile_link,删除bigfile源文件，bigfile_link依然可以正常使用。

**硬链接：**两个指针指向同一个文件。是**目录项**的同义词，且建立硬链接有限制，不能给分区简历，不能给目录简历

- 符号（symbol）连接

  符号链接：可以跨分区，可以给目录简历。

  ```text
  ln -s [源文件或目录] [目标文件或目录]
  ```

  ![2023-04-06_22-03-19.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-06_22-03-19.png)

  ![2023-04-06_22-06-04.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-06_22-06-04.png)

​		删除源文件后，符号链接变得不可用。

​		![2023-04-06_22-07-57.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-06_22-07-57.png)



​	

涉及函数：



- link
- unlink

- remove

- rename

## 1.8 utime

可以更改文件最后读的时间和最后修改的时间

## 1.9 目录的创建和销毁

涉及函数：

- mkdir()
- rmdir()



## 1.10 更改当前工作路径

涉及函数：

	- chdir(),cd函数是有该函数封装得到的。
	- fchdir()
	- getcwd(), 封装出来的命令 pwd



## 1.11分析目录和读取目录内容

- opendir

- closedir

- readdir

- rewinddir

- seekdir

- telldir

- glob：解析模式/通配符

  glob，可以实现上面函数的功能。

**小功能：**

```c
#include<stdio.h>
#include<stdlib.h>

int  main(int argc,int **argv)
{
    printf("argc = %d\n",argc);
    
    exit(0);
}
```



![2023-04-08_09-10-14.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-08_09-10-14.png)



这里有一个问题，统计数据参数个数的函数在输入 "*.c"的时候，其结果是多少？

“*”就是一个通配符。



**小功能：**通过glob函数读取/etc/下的文件

```c
#include<stdio.h>
#include<stdlib.h>
#include<glob.h>
//#define PATTERN "/etc/a*.conf"
#define PATTERN "/etc/*"
// 统计/etc目录下有多少以a*.conf文件

#if 0
int errfunc(cosnt char* epath,int eerrno)
{
    puts(epath);
    fprintf(strerr,"ERR MSG:%s",strerror(eerrno));
    return errno;
}
    
#endif
int main(int argc,char **argv)
{
    glob_t globers;
    int err = glob(PATTERN,0,NULL,&globers);
    if(err)
    {// 出错
        printf("Error code = %d\n",err);
        exit(1);
    }

    for(int i=0;i<globers.gl_pathc;i++)
    {
        puts(globers.gl_pathv[i]);
    }

    globfree(&globers);
    exit(0);
}
```



**小功能：**用opendir、readdir、closedir实现/etc/下文件的读取

```c
#include<stdio.h>
#include<stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>
#include <dirent.h>


int main()
{
    DIR* dir;
    struct dirent* rdd;
    dir = opendir("/etc/");
    if(dir == NULL)
    {
        closedir(dir);
        printf("opendir failed!!!\n");
        fprintf(stderr,"opendir failed,%s",strerror(errno));
        exit(1);
    }

    int count = 0;
    while((rdd = readdir(dir)) != NULL)
    {
        printf("%s\n",rdd->d_name);
        count++;
    }

    printf("=======================\n");
    fprintf(stdout,"total files number is %d\n",count);

    closedir(dir);
    exit(0);
}
```



# 2.系统数据文件和信息

## 2.1 /etc/passwd文件

```text
相关的函数：
	getpwuid()
	getpwnam()
```

```c
NAME
       getpwnam, getpwnam_r, getpwuid, getpwuid_r - get password file entry

SYNOPSIS
       #include <sys/types.h>
       #include <pwd.h>

       struct passwd *getpwnam(const char *name);
       struct passwd *getpwuid(uid_t uid);
通过uid和用户名查询用户的所有信息
```

不同的系统并不一定有/etc/passwd文件，所以这个还是要看具体系统的存放方式，linux系统中是有的。

![2023-04-12_20-50-11.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-12_20-50-11.png)

x代表passwd，是通过加密后的信息。

**小功能**：通过uid来获取用户名

```c
#include<stdio.h>
#include<stdlib.h>
#include <sys/types.h>
#include <pwd.h>

// 从命令行输入uid，打印出username
int main(int argc,char **argv)
{
    struct  passwd *pwdline;
    if(argc < 2)
    {
        fprintf(stderr,"Error,Usage:%s <uid>\n",argv[0]);
        exit(1);
    }

    pwdline = getpwuid(atoi(argv[1]));
    puts(pwdline->pw_name);
    exit(0);
}
```

![2023-04-12_20-51-04.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-12_20-51-04.png)

## 2.2 /etc/group

```text
相关函数
	getgrgid();
	getgrgrnam();
```

```c
NAME
       getgrnam, getgrnam_r, getgrgid, getgrgid_r - get group file entry

SYNOPSIS
       #include <sys/types.h>
       #include <grp.h>

       struct group *getgrnam(const char *name);
       struct group *getgrgid(gid_t gid);
```



## 2.3 /etc/shadow

```texdt
相关函数：
	getspnam()
	getspent()
	crypt()加密函数
	getpass()
```

```c
GETSPNAM(3)                                            Linux Programmer's Manual                                           GETSPNAM(3)

NAME
       getspnam, getspnam_r, getspent, getspent_r, setspent, endspent, fgetspent, fgetspent_r, sgetspent, sgetspent_r, putspent, lckp‐
       wdf, ulckpwdf - get shadow password file entry

SYNOPSIS
       /* General shadow password file API */
       #include <shadow.h>

       struct spwd *getspnam(const char *name);
       struct spwd *getspent(void);
```



## 2.4 时间戳

```text
涉及函数：
	time()
	gmtime
	localtime()
	mktime()
	strftime()
时间戳的类型: time_t、char *、struct tm
上面的时间函数是在这三个类型中切换的函数
```

![2023-04-12_22-09-26.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-12_22-09-26.png)



这里在首次调用./timelog的时候，去查看/tmp/out文件，查看不到任何内容，这是因为：

在IO中，除了终端设备，其他都是全缓冲模式，所以fprintf中的"\n"不能起到刷新缓冲区的作用了。

# 3.进程环境

## 3.1 main函数

```c
int main(int argc,char*argv[] )
```

## 3.2 进程终止

- 正常终止：
  - 从main函数返回
  - 调用exit
  - 调用\_exit或\_EXIT
  - 最后一个线程从其启动例程返回
  - 最后一个线程调用pthread_exit
- 异常终止：
  - 调用abort函数
  - 接到一个信号并终止
  - 最后一个线程对其取消请求作出响应

```text
exit与_exit的区别：
exit是库函数，而_exit是系统函数。
```

- **exit与_exit的区别：**

![2023-04-17_09-48-49.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-17_09-48-49.png)

图片解读：调用\_exit会导致当前进程终止，而调用exit，会先终止处理程序（比如钩子函数）和标准IO清理程序，然后才会调用\_exit函数 。

- exit与_exit调用时机的问题：

```text
当出现错误的时候，不需要或者不敢做任何操作的时候，我们就调用_Exit函数。
当出现错误的时候，我们需要保存数据，刷新文件等等操作的时候，我们就调用exit函数。
```





### 3.2.1 钩子函数atexit(3)

进程正常终止的时候，在钩子函数会被调用。

类似c++中的析构函数
```c
NAME
       atexit - register a function to be called at normal process termination
SYNOPSIS
       #include <stdlib.h>
       int atexit(void (*function)(void));
```

**案例：**

```c
#include<stdio.h>
#include<stdlib.h>

static void f1(void)
{
    puts("f1() is working!!!");
}

static void f2(void)
{
    puts("f2() is working");
}

static void f3(void)
{
    puts("f3() is working");
}
int main()
{
    puts("Begin!");

    atexit(f1);
    atexit(f2);
    atexit(f3);

    puts("End!");
    exit(0);

}
```

![2023-04-17_09-31-06.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-17_09-31-06.png)

**钩子函数的使用场景**：

当打开多个文件的时候，如果文件打开是失败，在fd100处要关闭很多进程。这个时候我们就可以使用钩子函数

```c
fd1 = open();
if(fd1<0)
{
    perror();
    exit(1);
}
fd2 = open();
if(fd2 <0)
{
    close(fd1);
    perror();
    exit(1);
}
...
fd100 = open();
if(fd100<0)
{
    close(fd1);
    close(fd2);
    ...
    close(fd99);
    perror();
    exit(1);    
}
   
```

改进

```c
fd1 = open();
if(fd1<0)
{
    perror();
    exit(1);
}
atexit();     --->   close(fd1)
fd2 = open();
if(fd2 <0)
{
    perror();
    exit(1);
}
atexit();     --->   close(fd2)
...
fd100 = open();
if(fd100<0)
{
    perror();
    exit(1);    
}

atexit();     --->   close(fd2)
```

除了open，还有malloc等等，只要用到申请资源的函数，下面就可以挂上钩子函数。



## 3.3 命令行参数的分析

相关函数：

```text
getopt()
getopt_long()
```

```c
NAME
       getopt, getopt_long, getopt_long_only, optarg, optind, opterr, optopt - Parse command-line options

SYNOPSIS
       #include <unistd.h>
       int getopt(int argc, char * const argv[],const char *optstring);
       extern char *optarg;
       extern int optind, opterr, optopt;
描述：
    optstring是包含合法的选项字符的字符串，如果这样的字符后门跟这":",说明这个选项需要一个参数，所以在optarg中，getopt()函数在argv参数中用指针指向接下来的内容，或者是接下来的argv-element的文本。两个":"意味着一个选项带着一个可选参数。如果当前argv参数中
    
```



## 3.4 环境变量

环境变量类似于全局变量。

- 输出自己系统的环境变量

```c
#include<stdio.h>
#include<stdlib.h>
extern char **environ;
int main()
{

    for(int i=0;environ[i] != NULL;i++)
    {
        puts(environ[i]);
    }

    exit(0);
}

```

相关函数：

```text
getenv()
setenv()
putenv()
```

## 3.5 c程序的存储空间布局

pmap

## 3.6 库

```text
动态库
静态库
手工装载库
dlopen()
dlclose()
dlerror()
dlsym()
```



## 3.7 函数跳转

类似于goto，但是goto无法夸函数跳转。比如c++中，抛出异常，就需要夸函数跳转。

```text
setjmp();  // 设置跳转点
longjmp(); // 从某个位置回到跳转点
// 这两个函数可以实现夸函数跳转
```

模拟实现压栈的功能:

<img src="https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/20230420210324.jpg" alt="20230420210324.jpg" style="zoom: 50%;" />

```c
#include<stdio.h>
#include<stdlib.h>
#include <setjmp.h>
static void d(void)
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Jump now!\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);
}
static void c(void)
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Call d().\n",__FUNCTION__);
    d();
    printf("%s():d() returned.\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);
}
static void b(void)
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Call c().\n",__FUNCTION__);
    c();
    printf("%s():c() returned.\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);

}

static void a(void)
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Call b().\n",__FUNCTION__);
    b();
    printf("%s():b() returned.\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);
}

int main()
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Call a().\n",__FUNCTION__);
    a();
    printf("%s():a() returned.\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);

    exit(0);
}
```

![2023-04-20_20-56-40.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_20-56-40.png)

使用setjmp进行跳转,(在函数a中设置跳转点，在函数d中跳转)

```c
#include<stdio.h>
#include<stdlib.h>
#include <setjmp.h>

static jmp_buf save;

static void d(void)
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Jump now!\n",__FUNCTION__);
    longjmp(save,6);
    printf("%s():End.\n",__FUNCTION__);
}


static void c(void)
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Call d().\n",__FUNCTION__);
    d();
    printf("%s():d() returned.\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);

}

static void b(void)
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Call c().\n",__FUNCTION__);
    c();
    printf("%s():c() returned.\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);

}

static void a(void)
{
    int ret;

    printf("%s():Begin.\n",__FUNCTION__);
    ret = setjmp(save);
    if(ret == 0)
    {
        printf("%s():Call b().\n",__FUNCTION__);
        b();
        printf("%s():b() returned.\n",__FUNCTION__);
    }
    else
    {
        printf("%s():Jumped back here with code %d\n.",__FUNCTION__,ret);
    }
    
    printf("%s():End.\n",__FUNCTION__);

}

int main()
{
    printf("%s():Begin.\n",__FUNCTION__);
    printf("%s():Call a().\n",__FUNCTION__);
    a();
    printf("%s():a() returned.\n",__FUNCTION__);
    printf("%s():End.\n",__FUNCTION__);

    exit(0);
}
```

![2023-04-20_20-58-14.png](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/blog_album/2023-04-20_20-58-14.png)

## 3.8 资源的获取及控制

```text
getrlimit()
setrlimit()
```





