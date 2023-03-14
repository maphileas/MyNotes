# gcc

## 1.gcc常用参数

- 参数：

  ```text
  -v / --V / --version 查看gcc版本号
  -I目录，指定头文件目录，注意-I和目录之间没有空格（也可以有空格）
  -c 只编译，生成.o文件，不进行连接
  -g 包含调试信息
  -o 指定生成的输出文件
  -On n=0~3 编译优化，n越大优化得越多
  -Wall 提示更多警告信息
  -D<DEF> 编译时定义宏，注意-D和<DEF>之间没有空格
  -E 生成预处理文件
  -M 生成.c文件与头文件依赖关系以用于Makefile，包括系统库的头文件
  -MM 生成.c文件与头文件依赖关系以用于Makefile，不包括系统库的头文件
  ```

## 2.gcc编译4步骤


- gcc编译可执行程序4步骤：预处理、编译、汇编、连接。

![2023-03-07_211828.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-07_211828.jpg)

```text
-I:  指定头文件所在目录位置
-c:  只做预处理、编译、汇编得到二进制文件。
-g:  编译时添加调试语句。 
```

# 静态库和共享库

## 1.静态库

<img src="https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-11_102724.jpg" alt="2023-03-11_102724.jpg" style="zoom: 50%;" />

- 加载机制

a.out、b.out、c.out都需要使用静态库文件，那么加载的时候，我们需要将库文件给每个.ou文件复制一份，如果静态库比较大，使用的程序又比较多，那么就非常耗费内存。

- 使用场景

  对空间要求较低，对时间要求较高的核心程序中。

### 1.1 静态库的制作

- add.c

  ```c
  int add(int a,int b)
  {
          return a+b;
  }
  ```


- sub.c

  ```c
  int sub(int a,int b)
  {
          return a-b;
  }
  ```

- div.c

  ```c
  int div2(int a,int b)
  {
          return a/b;
  }
  ```


静态库制作步骤：

1. 将.c文件生成.o文件

   ```gcc
   gcc -c sub.c -o sub.o
   gcc -c add.c -o add.o
   gcc -c div2.c -o div2.o
   ```

   

2. 使用ar工具制作静态库
   ```text
   ar rsc libmymath.a add.o sub.o div2.o
   --  库名为 lib+库名+.a
   ```

3. 编译静态库到可执行文件中：

   ```text
   gcc test.c lib库名.a -o a.out
   ```

   

​	静态库的使用：

```c
// test.c

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<unistd.h>
#include<pthread.h>

int main(int argc,char* argv[])
{
        int a=4,b=6;
        printf("%d+%d=%d\n",a,b,add(a,b));
        printf("%d-%d=%d\n",a,b,sub(a,b));
        printf("%d/%d=%d\n",a,b,div2(a,b));
        printf("hello world\n");
        return 0;
}
```

![2023-03-11_215130.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-11_215130.jpg)

```
gcc test.c libmymath.a -o test
```





- 存在的问题

  gcc test.c libmymath.a -o test  -Wall

  ![](pic2/2023-03-11_220636.jpg)

  这是因为在test.c中没有申明add、sub、div函数。

  **编译器在编译的时候，对于未申明的函数，编译器会根据下边的调用(函数名和参数)，帮你申请一个 返回值为int add(int a,int b)的函数（编译器生成的函数值只会是int类型的）。如果我的函数是void *add(int,int)，这个时候程序调用就不会成功。**

- 解决方法

  手动声明，在与静态库一起编译

  ```c
  ## 1 .申明
  #include<stdio.h>
  #include<stdlib.h>
  #include<string.h>
  #include<unistd.h>
  #include<pthread.h>
  // 在函数中申明函数
  int add(int,int);
  int sub(int,int);
  int div2(int,int);
  
  int main(int argc,char* argv[])
  {
          int a=4,b=6;
          printf("%d+%d=%d\n",a,b,add(a,b));
          printf("%d-%d=%d\n",a,b,sub(a,b));
          printf("%d/%d=%d\n",a,b,div2(a,b));
          printf("hello world\n");
          return 0;
  }
  
  ## 2.编译
  gcc test.c libmymath.a -o test
  ```

  ![2023-03-11_224115.jpg](pic2/2023-03-11_224115.jpg)

  将mymath.h放入inc/文件夹下，libmymath.a放入lib/文件夹下。符合项目的实际开发，在进行编译，如上图。

## 2.动态库

<img src="https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-11_103528.jpg" alt="2023-03-11_103528.jpg" style="zoom:50%;" />

- 加载机制

  使用动态库，运行时，会首先把动态库加载到内存中，a.out、b.out、c.out也在内存中运行，直接将内存所在的地址共享给.out文件，.out文件就可以使用了。

- 使用场景

​	  对空间要求较高，对时间要求较低的场景。

### 2.1 动态库的制作及使用

1. 将.c生成.o文件（生成与位置无关的代码 -fPIC）

   ```text
   gcc -c add.c -o add.o -fPIC
   ```

2. 使用gcc -shared制作动态库

   ```text
   gcc -shared -o lib库名.so add.o sub.o div2.o
   ```

3. 编译可执行程序时，指定所使用的的动态库。使用的参数 -l：指定库名 ， -L：指定库路径。

   ```text
   gcc test.c -o a.out -lmymath -L./lib
   ```

4. 运行可执行程序 ./a.out出错

   解决方法：

   1. 通过环境变量 export LD_LIBRARY_PATH=动态路径  （临时生效，终端重启环境变量失败）

   2. 写入终端配置文件 ./bashrc （永久生效）

   3. 拷贝自定义动态库到/lib（标准C库所在目录位置）

   4.  将动态库所在的路径，添加到 ld.so.conf

      1. vi /etc/ld.so.conf

      2. ![2023-03-12_125103.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-12_125103.jpg)

      3. 保存，然后更新ld.so.conf文件，sudo ldconfig -v

         ![2023-03-12_125238.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-12_125238.jpg)


### 2.2 实操

- 第一步
	![2023-03-12_103125.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-12_103125.jpg)
	
- 第二步
	   ![2023-03-12_103630.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-12_103630.jpg)
	
	   这里 -l 后面是库名，库名是去掉lib和后缀剩余的名字。我们生成的动态库是libmymath.so，所以库名就是mymath。
	
	   这里编译成功之后，但执行错误，

- 第三步 

### 2.3 动态库加载错误的原因及解决方法

2.2中错误的原因：

-  链接器：

  工作于链接阶段，工作时需要 -l 和-L

- 动态链接器：

  工作于程序运行阶段，工作时需要提供动态库所在目录位置。

  通过环境变量：exprot LD_LIBRARY_PATH=动态库的路径 

  ./a.out 成功

  ***注意*** ：环境变量是跟着进程走的，但关掉当前terminal的时候，设置的动态路径就无效了。也可以通过改配置文件，实现永久有效。

![2023-03-12_104753.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-12_104753.jpg)

- 环境变量的配置

  编辑 ~/.bashrc文件，在文件里面添加配置信息，然后刷新一下.bashrc文件即可(source .bashrc/ . .bashrc)。

  ![2023-03-12_105705.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-12_105705.jpg)

- ldd指令

  查看程序在运行起来之后会加载哪些动态库。

  ![2023-03-12_111716.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-12_111716.jpg)
