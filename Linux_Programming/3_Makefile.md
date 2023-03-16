# Makefile项目管理

创建makefile文件，命名只能是Makefile或者makefile。

* 基本原则
  * 若想生成目标，检查规则中的依赖条件是否存在，如不存在，则寻找是否有规则用来生成该依赖。
  * 检查规则中的目标是否需要更新，必须先检查它的所有依赖，依赖中有任一 一个被更新，则目标必须更新。
    * 分析各个目标和依赖之间的关系
    * 根据依赖关系自底向上执行命令
    * 根据修改时间比目标新，确定更新
    * 如果目标不依赖任何条件，则执行对应命令，以示更新。

## 1.一个规则

```text
目标：依赖条件
​	（一个tab缩进）命令
```

​		1). 目标的时间必须晚于依赖条件的时间，否则，则更新目录。

​		2). 依赖条件如果不存在，寻找新的规则去产生依赖。

ALL: 指定makefile的终极目标。

- 实例

  ***注：Makefile会将第一组遇到的规则目标，作为终极生成目标****

  ```makefile	
  a.out:sub.o add.o div2.o hello.o
          gcc sub.o add.o div2.o hello.o -o a.out
  sub.o:sub.c
          gcc -c sub.c -o sub.o
  add.o:add.c
          gcc -c add.c -o add.o
  div2.o:div2.c
          gcc -c div2.c -o div2.o
  hello.o:hello.c
          gcc -c hello.c -o hello.o
  ```

  这样的话，会生成a.out文件。

  如果makefiel是这样

  ```makefile
  sub.o:sub.c
          gcc -c sub.c -o sub.o
  add.o:add.c
          gcc -c add.c -o add.o
  div2.o:div2.c
          gcc -c div2.c -o div2.o
  hello.o:hello.c
          gcc -c hello.c -o hello.o
  a.out:sub.o add.o div2.o hello.o
          gcc sub.o add.o div2.o hello.o -o a.out
  ```

  只会生成sub.o文件，Makefile会将第一组遇到的规则目标，作为终极生成目标。可以通过ALL关键字来指定终极生成目标。

  ```makefile
  ALL:a.out
  sub.o:sub.c
          gcc -c sub.c -o sub.o
  add.o:add.c
          gcc -c add.c -o add.o
  div2.o:div2.c
          gcc -c div2.c -o div2.o
  hello.o:hello.c
          gcc -c hello.c -o hello.o
  a.out:sub.o add.o div2.o hello.o
          gcc sub.o add.o div2.o hello.o -o a.out
  ```

## 2.两个函数

```makefile
src=$(wildcard *.c) 找到当前目录下所有后缀为.c的文件，将文件名组成列表，赋值给变量src
obj=$(patsubst %.c,%.o,$(src)) 把src变量里所有的后缀为.c的文件替换成.o
	$()：表示把一个变量取值，脚本语法。
	src代表所有的.c变量，不能代表单独一个。obj变量同理。
	
clean:     clean没有依赖
    -rm -rf $(obj) a.out    "-"的作用是删除不存在文件时不报错
```

通过函数代理makefile中的一些变量

```makefile
src=$(wildcard *.c) #add.c sub.c div2.c hello.c
obj=$(patsubst %.c,%.o,$(src)) #add.o sub.o div1.o hello.o

ALL:a.out
a.out:$(obj)
        gcc $(obj) -o a.out
sub.o:sub.c
        gcc -c sub.c -o sub.o
add.o:add.c
        gcc -c add.c -o add.o
div2.o:div2.c
        gcc -c div2.c -o div2.o
hello.o:hello.c
        gcc -c hello.c -o hello.o
```

在makefile中添加clean函数，可以通过make clean删除掉*.o的文件。

make clean -n 是模拟删除，打印出要删除的文件，没有删除源码，需要进行确认，确认后可以执行make clean，删除文件。

```makefile
src=$(wildcard *.c) #add.c sub.c div2.c hello.c
obj=$(patsubst %.c,%.o,$(src)) #add.o sub.o div1.o hello.o
ALL:a.out
a.out:$(obj)
        gcc $(obj) -o a.out
sub.o:sub.c
        gcc -c sub.c -o sub.o
add.o:add.c
        gcc -c add.c -o add.o
div2.o:div2.c
        gcc -c div2.c -o div2.o
hello.o:hello.c
        gcc -c hello.c -o hello.o
clean:
        -rm -rf $(obj) a.out
```

![2023-03-15_173633.jpg](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/note/2023-03-15_173633.jpg)

## 3.三个自动变量

```text
1. $@: 在规则的命令中，表示规则中的目标
2. $<: 在规则的命令中，表示规则中的第一个条件
3. $^: 在规则命中中，表示规则中的所有依赖条件，组成一个列表，以空格隔开。如果这个列表中有重复的项则消除重复项。
   如果将该变量应用在模式规则中，它可将依赖条件列表中的依赖依次取出，套用模式规则
```

模式规则：

```makefile
%.o:%.c
    gcc -c $< -o $@
```

静态模式规则：

```makefile
$(obj):%.o:%.c
    gcc -c $< -o $@
```

伪目标：

```makefile
.PHONY: clean ALL
```

实例：

```makefile
src=$(wildcard *.c) #add.c sub.c div2.c hello.c
obj=$(patsubst %.c,%.o,$(src)) #add.o sub.o div1.o hello.o
ALL:a.out
a.out:$(obj)
  gcc $^ -o $@
# 目标：依赖条件
# $@:规则中的目标
# $<:规则中的第一个条件
sub.o:sub.c
  gcc -c $< -o $@
add.o:add.c
  gcc -c $< -o $@
div2.o:div2.c
  gcc -c $< -o $@
hello.o:hello.c
  gcc -c $< -o $@
clean:
  -rm -rf $(obj) a.out
```

套用模式规则

```makefile
src=$(wildcard *.c) #add.c sub.c div2.c hello.c
obj=$(patsubst %.c,%.o,$(src)) #add.o sub.o div1.o hello.o

ALL:a.out
a.out:$(obj)
    gcc $^ -o $@
%.o:%.c
     gcc -c $< -o $@
clean:
     -rm -rf $(obj) a.out
```

