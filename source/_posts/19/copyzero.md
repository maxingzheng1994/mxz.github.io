# copyzero
[wiki](https://wikipedia.dx.gugeeseo.com/wiki/零拷贝)
[https://www.cnblogs.com/f-ck-need-u/p/7615914.html](https://www.cnblogs.com/f-ck-need-u/p/7615914.html)
[https://www.jianshu.com/p/193cae9cbf07](https://www.jianshu.com/p/193cae9cbf07)


零复制（英语：Zero-copy；也译零拷贝）技术是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域。这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽。

零复制的概念是避免将数据在内核空间和用户空间进行拷贝。主要目的是减少不必要的拷贝，避免让CPU做大量的数据拷贝任务

注：上面只是说正常情况下，例如某些硬件可以完成TCP/IP协议栈的工作，数据可以不经过socket buffer，直接在app buffer和硬件之间传输数据，RDMA技术就是在此基础上实现的。

上面讲的机制看起来一切都很好，但它还是有个缺点：如果我想在传输时修改数据本身，就无能为力了。不过，很多操作系统也提供了内存映射机制，对应的系统调用为mmap()/munmap()。通过它可以将文件数据映射到内核地址空间，直接进行操作，操作完之后再刷回去。其对应的简要时序图如下。

mmap()函数将文件直接映射到用户程序的内存中，映射成功时返回指向目标区域的指针。这段内存空间可以用作进程间的共享内存空间，内核也可以直接操作这段空间。

在映射文件之后，暂时不会拷贝任何数据到内存中，只有当访问这段内存时，发现没有数据，于是产生缺页访问，使用DMA操作将数据拷贝到这段空间中。可以直接将这段空间的数据拷贝到socket buffer中。所以也算是零复制技术


[https://www.cnblogs.com/f-ck-need-u/category/1311970.html](https://www.cnblogs.com/f-ck-need-u/category/1311970.html)