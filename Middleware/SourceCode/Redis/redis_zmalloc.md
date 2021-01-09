# Redis内存分配

对于一个C语言程序，内存分配都是编写代码时必须要重视的问题。而对于Redis这样的缓存数据库来说，对内存的分配就显得更为重要。

## C语言内存结构
一般地，由C/C++编译的程序占用的内存空间可分为如下几个部分：
* 栈区：由编译器自动分配释放，存放函数的参数值、局部变量的值等。栈内存分配运算内置于处理器的指令集中，效率高但是分配的内存空间有限。
* 堆区：由程序员手动分配的存储区，如果程序不释放所占用的内存，该内存将会一直占用，直到程序运行结束后由系统自动回收。而本文所说的内存分配也主要是针对堆区的操作。
* 静态存储区：全局变量和静态变量都存储在静态存储区，其中初始化的全局变量和静态变量在一个区域，这块空间当程序运行结束后由系统释放。
* 常量存储区：const修饰的全局变量和常量字符串等都存储在常量存储区，存储在常量存储区的只读不可写。需要注意的是，const修饰的局部变量依然在栈上。
* 程序代码区：存放源程序的二进制代码。

其中，栈区和堆区是程序运行中最常用也是最重要的两个存储区。这里对二者的区别和联系进行简单的说明：
1. 从数据结构上讲，栈是C语言代码底层便支持的数据结构，一般地，有专门的寄存器指向栈所在的地址，有专门的机器指令完成数据的入栈和出栈等。而堆通常是由库函数提供的数据结构。
2. 从申请方式上讲，栈由编译器管理，堆的分配和释放由程序员管理。
3. 从申请大小上讲，栈是向低地址生长的数据结构，是一块连续的内存，能从栈中获得的内存较小，编译器间确定大小；堆是向高地址生长的数据结构，是一个不连续的存储空间，内存获取比较灵活，内存区域也比较大。
4. 从存储内容上讲，栈通常存储函数的参数值、局部变量值、函数调用栈等；堆区的存储内容则由程序员自己决定。
5. 从分配方式上讲，堆都是动态分配的，没有静态分配的堆；而栈有两种分配方式：静态分配和动态分配。静态分配是编译器完成的，如局部变量的分配等，动态分配则是由alloca函数分配的，但是对于栈的动态分配是由编译器实现的，无需程序员手工调用。
6. 从分配效率上讲，栈是由计算机底层提供支持的，其效率高；而堆由库函数提供，它的分配机制复制，分配效率低。

## Redis支持的内存分配器
### libc/glibc库
libc和glibc都是Linux下C语言的标准库，其提供了基本的内存分配与释放实现。libc是Linux下的ANSI C函数库，ANSI C是基本的C语言函数库，包含了C语言最基本的库函数；glibc是Linux下GNU C函数库，GNU C是由GNU开发的C语言库，用于更好的利用C语言开发基于Linux操作系统的程序。目前glibc已经逐渐代替了Linux原生的libc库。
#### malloc函数
函数原型：void *malloc(size_t size);
功能：
1. 开辟一块size大小的连续堆内存；
2. size表示堆上所开辟内存的大小（字节数）；
3. 函数返回值是一个指针，指向刚刚开辟的内存的首地址；
4. 如果开辟内存失败，返回一个空指针，即返回值为NULL；
5. 当内存不再使用时，应使用free()函数将内存块释放；
6. 使用时必须包含头文件<stdlib.h>或<malloc.h>。

#### calloc函数
函数原型：void *calloc(size_t n, size_t size);
功能：
1. 在内存的动态存储区中分配n个长度为size的连续空间；
2. 函数返回一个指向分配起始地址的指针；
3. 如果分配不成功，返回NULL；
5. 当内存不再使用时，应使用free()函数将内存块释放；
6. 使用时必须包含头文件<stdlib.h>或<malloc.h>。

#### realloc函数
函数原型：void \*realloc(void\* mem_address, size_t newsize);
功能：
1. 为已有内存的变量重新分配新的内存大小（可大、可小）；
2. 先判断当前的指针是否有足够的连续空间，如果有，扩大mem_address指向的地址，并将mem_address返回；
3. 如果空间不够，先按照newsize指定的大小分配空间，将原有数据从头到尾拷贝到新分配的内存区域，而后释放原来mem_address所指向的内存区域，同时返回新分配内存区域的首地址，即重新分配的内存块的地址；
4. 如果重新分配成功则返回指向被分配内存的指针；
5. 如果分配不成功，则返回NULL；
6. 当内存不再使用时，应使用free()函数将内存块释放；
7. 使用时必须包含头文件<stdlib.h>或<malloc.h>。

#### free函数
函数原型：void free(void *ptr);
功能：
释放指针变量在堆区上占用的内存空间，不能释放栈上的内存空间，free要与malloc、calloc、realloc等成对使用。  
注意：  
如果malloc、calloc、realloc比free多，会造成内存泄露；如果malloc、calloc、realloc比free少，会造成二次删除，破坏内存，导致程序崩溃。

使用libc库完全可以实现内存的分配与释放；但是直接使用上述提供的函数可能无法满足多线程高性能场景，在使用过程中也可能会产生很多内存碎片。因此，业界基于libc又衍生出了多种内存分配器，如glibc的ptmalloc，Facebook开源的JeMalloc和Google开源的TCMalloc等。

### JeMalloc
JeMalloc是Facebook开源的一款内存分配器，其在多线程场景下体现了很好的性能，也减少了内存碎片的产生。Redis默认使用JeMalloc作为内存分配器。

对于JeMalloc的原理，本文不做详细介绍，可参考[JeMalloc官网](http://jemalloc.net/)和[JeMalloc详解](https://zhuanlan.zhihu.com/p/48957114)。

### TCMalloc
TCMalloc是Google开源的一款内存分配器，与JeMalloc一样，也是基于libc库对内存分配进行了优化，提高了多线程场景的性能，同样也能减少内存碎片。Redis也同样支持使用TCMalloc进行内存分配。

对于TCMalloc的原理，本文也不再赘述，可参考[TCMalloc官网](https://google.github.io/tcmalloc/overview.html)和[图解TCMalloc](https://zhuanlan.zhihu.com/p/29216091)。

### Mac
Mac是Mac/iOS专用的内存分配器。

## Redis内存分配
Redis作为内存数据库，对内存的利用达到极致，因此内存分配对于Redis也显得尤为重要。Redis本身并未提供一套完整的内存分配器，其主要还是利用上述内存分配器，并根据Redis自身特点对其进行封装，从而实现Redis的内存分配。

以下是Redis内存分配的API，这些API底层都是基于libc、JeMalloc或TCMalloc实现，有兴趣可以参考上述文档详细了解。  
### malloc（分配一块大小为size的内存）
* void *zmalloc(size_t size); 如果未分配到内存抛出OOM异常
* void *ztrymalloc(size_t size); 如果未分配到内存直接返回NULL
* void *zmalloc_usable(size_t size, size_t *usable);  如果usable非空，则将分配的内存大小写入usable中
* void *ztrymalloc_usable(size_t size, size_t *usable);
* void *zmalloc_no_tcache(size_t size); 当前仅针对JeMalloc，绕过Thread Cache直接分配内存
### calloc（使用calloc分配一块大小为size的内存）
* void *zcalloc(size_t size);
* void *ztrycalloc(size_t size);
* void *zcalloc_usable(size_t size, size_t *usable);
* void *ztrycalloc_usable(size_t size, size_t *usable);
### realloc（为当前所指向的地址重新分配大小为size的内存）
* void *zrealloc(void *ptr, size_t size);
* void *ztryrealloc(void *ptr, size_t size);
* void *zrealloc_usable(void *ptr, size_t size, size_t *usable);
* void *ztryrealloc_usable(void *ptr, size_t size, size_t *usable);
### free（释放内存）
* void zfree(void *ptr); 释放内存
* void zlibc_free(void *ptr); 使用libc的free函数
* void zfree_no_tcache(void *ptr); 当前仅针对JeMalloc，绕过Thread Cache释放内存
### 其他
* char *zstrdup(const char *s);  拷贝字符串
* size_t zmalloc_used_memory(void); 返回已使用的内存大小
* void zmalloc_set_oom_handler(void (*oom_handler)(size_t));  设置OOM处理函数
* size_t zmalloc_get_rss(void);  以特定操作系统的方式获取RSS信息
* int zmalloc_get_allocator_info(size_t *allocated, size_t *active, size_t *resident); 获取分配器信息
* void set_jemalloc_bg_thread(int enable);  让JeMalloc异步清理内存碎片
* int jemalloc_purge(); 清理内存碎片
* size_t zmalloc_get_private_dirty(long pid);  返回私有脏页的大小
* size_t zmalloc_get_smap_bytes_by_field(char *field, long pid); 获取/proc/self/smaps中指定字段的总和（将KB转换为字节）
* size_t zmalloc_get_memory_size(void); 返回物理内存（RAM）的大小
