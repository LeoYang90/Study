# UNIX 基础知识

### 1、什么是 UNIX 系统调用？公共函数库？

	UNIX内核的接口称之为系统调用。公用函数库构建在系统调用接口之上。应用程序既可以使用公用函数
	库，也可以使用系统调用。
	
### 2、`man` 帮助手册的参数 1、2、3、4、8 代表什么（`man 1 ls`）？

   - `1`： 可执行程序或者 `shell command`的说明
   - `2`： 系统调用的说明
   - `3`： 库函数的说明
   - `4`： 特殊文件（通常位于`/dev/`）的说明
   - `8`： 系统管理员命令的说明（通常只有 `root`可用）

### 3、UNIX 系统的两种时间是什么？

   - 日历时间：自 UTC 1970年1月1日 00:00:00 以来经历过的秒数累计值。用 `time_t`数据类型来保存这种时间值。
   - 进程时间：也称作CPU时间，用于度量进程使用的CPU资源。进程时间以时间滴答来计算，用`clock_t`数据类型保存这种时间值。

### 4、当度量一个进程的执行时间时，UNIX系统为一个进程维护了3个进程时间值是什么？

   - 时钟时间： 又称作墙上时钟时间，是进程运行的时间总量，其值与系统中同时运行的进程数有关
   - 用户CPU时间：执行用户指令所用的时间量
   - 系统CPU时间：该进程执行内核程序所经历的时间。如进程执行一个`read`系统调用，则内核执行该系统调用的时间计入系统 CPU 时间
	
	 > 用户CPU时间和系统CPU时间之和称作 CPU 时间
	 
	 
### API

```
char *strerror(int errnum);
void perror(const char*msg);
```