一、系统调用
1、	系统调用是用户空间程序访问内核空间的一种方式，系统调用通过软中断0x80陷入内核，跳转到系统调用处理程序（中断处理程序）system_call函数，
     并执行相应的服务例程（内核函数）；比如系统调用dup()的服务例程是内核函数sys_dup()。
2、  x86-32架构下分析dup()为例

![image](https://github.com/wanglinhuiyong/linux/blob/master/hook-syscall/image.png)

