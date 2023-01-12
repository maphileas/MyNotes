## mini_databases;
### part1 简介和设置REPL(Read-Execute-Print Loop)

作为一个数据开发者，我每天都会使用关系型和非关系型数据库。但是数据库对我而言，就像一个黑盒子。我有一些问题：

- 数据以哪种格式保存的？（在内存还是在磁盘）
- 它们什么时候从内存移动到磁盘？
- 每个表为什么只能有一个主键？
- 回滚事务是如何生效的？
- 索引的格式是怎么样的？
- 何时以及如何进行全表扫描?
- 预处理语句以什幺格式保存？

换句话说，数据库是如何工作的？

#### 1.1 Sqlite

![sqlite](https://cdn.jsdelivr.net/gh/maphileas/blog_album@main/img/arch.gif)

一条查询语句，会经历一系列的程序，旨在恢复或修改数据。这个前后（front-end）由下列部分组成：

- tokenizer
- parser
- code generator

front-end的输入是一个sql查询，输出是sqlite虚拟机字节码（必要的编译程序，能够在数据库上运行）。



back-end由下面几部分组成：

- virtual machine
- B-tree
- pager
- os interface

**virtual machine**将front-end生成的字节码作为指令。然后，它可以对一个或多个表或索引执行操作，每个表或索引都存储在称为 B 树的数据结构中。VM 本质上是关于字节码指令类型的大开关语句。

每个 **B-tree**都由许多节点组成。每个节点的长度为一页。B-tree可以从磁盘检索页面，也可以通过向pager发出命令将其保存回磁盘。

**Pager**接收用于读取或写入数据页的命令。它负责在数据库文档中以适当的偏移量读取/写入。它还在内存中保留最近访问的页面的缓存，并确定何时需要将这些页面写回磁盘。

**os interface**是因编译 sqlite 所针对的操作系统而异的层。在本教程中，我不打算支持多平台。

--------------

### part2 最简单的SQL编译器和虚拟机

