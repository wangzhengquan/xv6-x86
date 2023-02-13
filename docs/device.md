# 设备管理

xv6根linux一样把所有的设备都当文件管理。这里以键盘和显示器为例，键盘和显示器统称为控制台设备（console）,其中键盘是输入设备，显示器是输出设备。

## 控制台的创建

xv6的控制台的创建代码在init.c中，有如下两句：

```
mknod("console", 1, 1); 
open("console", O_RDWR);
```

mknod是在文件系统里创建一个名字为console的设备文件，该设备的 major=1, minor=1

## 控制台读写方法的注册
在console.c的consoleinit方法中有下面两句对控制台读写方法进行注册
```
devsw[CONSOLE].write = consolewrite;
devsw[CONSOLE].read = consoleread;

```
其中CONSOLE的值是1,与创建方法中的major对应


## 控制台读写方法的系统调用

与文件的读写一样对设备的读写都是通过open返回的文件描述符调用系统调用方法read 和 write。 例如调用read方法后，read通过系统调用的中断流程又调用sys_file.c的sys_read，sys_read又调用file.c的fileread，fileread判断当前文件类型是FD_INODE调用fs.c的readi, readi判断inode的类型是T_DEV调用`devsw[ip->major].read(ip, dst, n)` 也就是前面注册的读方法consoleread。 

## 键盘输入

consoleread是从系统缓存队列里读取的，而系统缓存队列的内容是由键盘输入的。键盘按键被按下时会产生一个中断,如果是主板上的键盘则中断编号是T_IRQ0 + IRQ_KBD，如果是键盘是通过串口连接到主板上的则中断编号为T_IRQ0 + IRQ_COM1，根据这不同的中断编号调用的方法也不同，分别是kbdintr和uartintr，这两个方法都调用consoleintr，只是参数分别是kbdgetc和uartgetc两个不同获取按键ASCII码的方法。consoleintr方法会把获取到的键盘输入字符追加到缓存队列里。




