# UNIX 标准及实现

### 1、POSIX 与 SUS 是什么？

   - POSIX(Portable Operating System Interface): 指的是可移植操作系统接口。该标准的目的是提升应用程序在各种UNIX系统环境之间的可移植性。它定义了“符合POSIX”的操作系统必须提供的各种服务。( POSIX 包含了 ISO C 标准库函数)
   - SUS(Single Unix Specification)：是 POSIX 标准的一个超集，他定义了一些附加接口扩展了 POSIX 规范提供的功能。

### 2、UNIX 的主要实现有哪些？

   - SVR4(UNIX System V Release 4)
   - 4.4 BSD(Berkeley Software Distribution)
   - FreeBSD
   - Linux
   - Mac OS X
   - Solaris

### 3、程序限制分为哪几种？
 
    某些限制在一个给定的 UNIX 实现中可能是固定的（由头文件定义），在另一个 UNIX 实现中可能是动
    态的（需要由进程调用一个函数获得限制值）。如文件名的最大字符数在不同的操作系统中，是属于动态/
    静态限制。因此提供了三种限制：
	
	 1、编译时限制（由头文件给定）
	 2、与文件或者目录无关的运行时限制（由 `sysconf`函数给定）
	 3、与文件或者目录相关的运行时限制（由 `pathconf`函数以及`fpathconf`函数给定）
	 
### API 
  
   	long sysconf(int name); 
	long pathconf(const char*pathname,int name);
	long fpathconf(int fd,int name); //fd 为文件描述符