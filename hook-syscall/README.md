一、系统调用

1、系统调用是用户空间程序访问内核空间的一种方式，系统调用通过软中断0x80陷入内核，跳转到系统调用处理程序（中断处理程序）system_call函数，
     并执行相应的服务例程（内核函数）；比如系统调用dup()的服务例程是内核函数sys_dup()。

2、x86-32架构下分析dup()为例


![image](https://github.com/wanglinhuiyong/linux/blob/master/hook-syscall/image.png)

过程：应用程序代码调用库函数dup(oldfd)，该函数是一个包装系统调用的库函数 ；
     库函数dup(oldfd){}负责准备向内核传递的参数，并触发软中断，将进程切换到内核态；
     CPU被软中断打断后，执行中断处理函数，即系统调用处理函数 ( system_call )；
     系统调用处理函数调用系统调用服务例程sys_dup()，真正开始处理该系统调用；

（1）	用户应用程序在某些时候可以直接通过系统调用来访问内核；但更多时候， 应用程序是通过操作系统提供的应用编程接口（API——C库的函数）而不是直接通过系统调用来编程。

（2）	内核提供的每个系统调用在C库中都具有相应的封装函数。系统调用与其C库封装函数的名称常常相同，比如，read系统调用在C库中的封装函数即为read函数。当然，也会有挺多C库封装函数和系统调用名称不同。

（3）	系统调用和C库函数之间并不是一一对应的关系。可能几个不同的函数会调用到同一个系统调用，即多对一关系，比如C库函数malloc和free都是通过brk系统调用来扩大或缩小进程的堆栈，execl、execlp、execle、execv、execvp和execve这些C库函数都是通过execve系统调用来执行一个可执行文件。也有可能一个函数调用多个系统调用，即一对多关系。

（4）	strace工具可以跟踪命令的执行，并显示出该命令执行过程中所使用到的所有系统调用。

二、系统调用表
    1、系统调用表
      （1）系统调用表（System call Table），是一张由指向实现各种系统调用的内核函数的函数指针组成的表，该表可以基于系统调用编号进行索引，来定位函数地址，完成系统调用。
      （2）在sys_call_table.c中定义
             
          void *sys_call_table[__NR_syscalls] = {
	                       #include <asm/unistd.h>
          };
        《1》sys_call_table是个数组，一般有三百多个元素(在linux-4.4.131中__NR_syscalls为379)；
        《2》#include <linux/***.h> 是在/usr/src/kernels/3.10.0-693.21.1.ns7.007.mips64el/include/linux下面寻找源文件。
            #include <asm/***.h> 是在/usr/src/kernels/3.10.0-693.21.1.ns7.007.mips64el/arch/arm（mips/x86/arm64）/include/asm下面寻找源文件。
            可以通过find / -name xxx.h寻找此文件路径
        《3》此数组维护者一张表
             #define __NR_io_setup 0
             __SC_COMP(__NR_io_setup, sys_io_setup, compat_sys_io_setup)
             #define __NR_io_destroy 1
             __SYSCALL(__NR_io_destroy, sys_io_destroy)
             #define __NR_io_submit 2
            __SC_COMP(__NR_io_submit, sys_io_submit, compat_sys_io_submit)
             #define __NR_io_cancel 3
              __SYSCALL(__NR_io_cancel, sys_io_cancel)
               ......................
             #define __NR_fchown 55
	        __SYSCALL(__NR_fchown, sys_fchown)     //相当于[__NR_fchown] = (sys_fchown),
	        #define __NR_openat 56
	        __SC_COMP(__NR_openat, sys_openat, compat_sys_openat)
	        #define __NR_close 57
	        __SYSCALL(__NR_close, sys_close)
               注：#define  __SYSCALL(nr, call)  [nr] = (call)，
          相当于sys_call_table[] = { sys_io_setup, sys_io_destroy, sys_io_submit, ......,sys_fchown, sys_openat,....}
     可以看出这些内核函数的地址都存在一个表里面，而系统调用号（数组下标），其实是地址的偏移。所以，内核函数的地址就等于起始地址 + 系统调用号 * 表中每个地址的字节数
     注：mips架构下，系统调用号存在5000的偏移，可能是为了区分不同的架构，所以每个内核函数在系统调用表的位置要-5000。比如sys_close函数在sys_call_table[__NR_close-5000]处。
           #if _MIPS_SIM == _MIPS_SIM_ABI64

          /*
          * Linux 64-bit syscalls are in the range from 5000 to 5999.
          */
          #define __NR_Linux			5000
          #define __NR_read			(__NR_Linux +	0)
          #define __NR_write			(__NR_Linux +	1)
          #define __NR_open			(__NR_Linux +	2)
          #define __NR_close			(__NR_Linux +	3)
          。。。。。。。。。。
          
          
   2、在Linux内核2.6之后，不能直接导出sys_call_table的地址后，我们要如何获得系统调用表的地址，从而实现系统调用的截获呢？
           命令：cat /proc/kallsyms | grep sys_call_table
           或cat /boot/System.mapXXXX | grep sys_call_table
              某个具体的系统调用的地址：cat /proc/kallsyms | grep sys_close



