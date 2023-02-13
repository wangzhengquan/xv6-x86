# 硬盘中断
ide.c是系统的硬盘驱动器，上层会调用iderw对硬盘进行读写。iderw把读写请求（buf）追加到请求队列（idequeue）里面。
```C
for(pp=&idequeue; *pp; pp=&(*pp)->qnext) 
    ;
*pp = b;
```
如果idequeue里面没有其他的请求，就开起idestart处理当前请求。最后调用sleep等待请求处理完成。

在idestart里根据buf记录的blockno计算出扇区编号，根据每个块的大小（BSIZE）计算读写扇区的数目，根据flags里的B_DIRTY标记判断是读还是写操作。并把这些信息通过outb告诉硬盘控制器，如果是写操作还要把buf里的data通过outsl写入到硬盘控制器的缓存。

当硬盘完成了数据写或数据读的准备后，会产生一个硬盘读写中断，经过系统中断处理流程后进入ideintr方法。ideintr把处理完的请求从请求队列里移除。
```C
idequeue = b->qnext;
```
然后判断如果需要读取硬盘数据就调用insl进行数据读取。然后调用wakeup唤醒等待硬盘读写该数据的进程。最后调用idestart继续处理请求队列里的下一个请求。