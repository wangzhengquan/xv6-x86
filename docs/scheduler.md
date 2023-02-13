# CPU调度线程

从 系统启动加载（boot loader）--> entry.S --> main 里的一系列初始化 ，最后到scheduler（）这是整个系统的主干。scheduler是一个单独的线程，有独自的栈，每个CPU都运行一个scheduler。scheduler每运行一次就会尝试找到一个状态为RUNNABLE的进程，然后让出CPU给这个进程运行，具体就是调用swtch把当前的scheduler的线程状态保存到它的栈中，然后把CPU的寄存器设置为要运行的进程之前在内核栈中保存的状态。当这个进程运行一段时间又要决定让出CPU，比如因为时间中断的原因调用yield -> sched -> swtch 这一次置换与前面那一次正好相反，先把当前进程的状态保存到该进程内核栈中，然后把前面保存到scheduler线程栈中的CPU寄存器状态恢复到CPU寄存器中，scheduler再次获得CPU继续在上一次让出CPU的地方执行，也就是继续for循环寻找下一个状态为RUNNABLE的进程，然后重复前面的过程再次调用swtch把CPU让给这个进程运行。


这里面的重点就是swtch方法了，下面详细分析swtch方法。

## Context switching

如上所述 switch 的功能就是**把当前线程的寄存器状态保存到旧线程的栈中，把新线程栈中之前保存的寄存器状态恢复到CPU寄存器中,以实现线程的切换**。switch的代码如下：

```
# Context switch
#
#   void swtch(struct context **old, struct context *new);
# 
# Save the current registers on the stack, creating
# a struct context, and save its address in *old.
# Switch stacks to new and pop previously-saved registers.

.globl swtch
swtch:
  movl 4(%esp), %eax  // eax = old
  movl 8(%esp), %edx  // edx = new

  # Save old callee-saved registers
  pushl %ebp
  pushl %ebx
  pushl %esi
  pushl %edi

  # Switch stacks
  movl %esp, (%eax)  // *old = esp
  movl %edx, %esp    // esp = new

  # Load new callee-saved registers
  popl %edi
  popl %esi
  popl %ebx
  popl %ebp
  ret
```
 
在调用swtch函数的时候按照x86的惯例会先把%eip保存到当前栈中,然后跳转到swtch这里。接着swtch的四个pushl命令继续保存callee-saved registers到当前栈中，然后后把保存位置也就是当前栈指针%esp记录在在*old 中。 接着把新的栈指针赋给%esp, 然后四个popl指令把新栈中保存的状态值恢复到寄存器中，ret指令恢复%eip寄存器。 %esp 和 %eip的恢复意味着CPU切换了栈和正在执行的代码。