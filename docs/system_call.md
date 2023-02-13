# 系统调用

## 系统调用的用户接口

用户程序系统调用的接口在 user.h 里定义，在usys.S里实现，每个系统调用都用一个宏实现
```
#define SYSCALL(name) \
  .globl name; \
  name: \
    movl $SYS_ ## name, %eax; \
    int $T_SYSCALL; \
    ret
```
`movl $SYS_ ## name, %eax;` 这里的”##“是宏定义里用于拼接变量用的，这段代码是把系统调用编号赋值给%eax，系统调用编号是用来标识将要被调用的系统服务接口。还记得在“中断处理”那一章，中断发生后所有寄存器都会被被保存到trapframe中，而eax作为其中之一会在syscall那里使用到。
`int $T_SYSCALL` 产生一个系统调用的中断，还记得系统调用的中断码即T_SYSCALL是64。


## 系统调用的流程
这里以exec为例，当用户程序执行exec系统调用函数后，会执行usys.S里SYSCALL(exec)宏定义的代码,这段代码在预处理完成后会是下面的样子
```
.globl exec; 
  exec: 
    movl $SYS_exec, %eax;
    int $T_SYSCALL;
    ret
```
`int $T_SYSCALL` 产生一个系统调用的中断，CPU进入中断处理的流程，关于中断处理的流程参见前一章“中断处理”，在那一章里讲了系统如何找到中断处理的入口，保存当前进程状态，最后进入trap函数，这里主要讲在trap函数里关于系统调用的处理过程。

在trap函数里通过中断编号（trapno）判断中断类型是系统调用，然后调用syscall函数，在syscall函数里取出保存在trapframe里的eax，即系统调用编号。通过系统调用编号在系统调用列表中（syscalls[]）索引到相应的系统调用服务即sys_exec，然后调用sys_exec并将返回值赋予trapfraem的eax，在系统调用返回时trapfraem的eax做为系统调用的返回值会被恢复到%eax寄存器中。 

## 获取系统调用参数

内核里面实现系统调用服务的函数比如sys_exec，要获取用户程序传递的参数，而用户程序传递的参数是放在进程的用户栈里的，这里用户栈的栈顶指针保存在trapframe的esp里，栈顶的第一个值是系统调用的返回地址，从第二值开始才是传递的参数，所以第一个参数的地址是`%esp+4`， 第n个参数的地址是`%esp+4+4*n`。

系统实现了一系列的辅助函数如argint, argptr, argstr和argfd 分别去获取整型，指针型，字符串型和文件描述符型的系统调用参数。例如argint首先计算第n个参数的地址，然后把这个地址告诉fetchint，fetchint取出这个地址上的int数据并拷贝到ip指定的变量地址上。


