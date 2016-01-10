Title: Tricking Android MemoryFile
Date: 2016-1-10
Slug: tricking-android-memoryfile

之前在做一个内存优化的时候，使用到了MemoryFile，由此发现了MemoryFile的一些特性以及一个非常trickly的使用方法，因此在这里继续

###What is it
MemoryFile是android在最开始就引入的一套框架，其内部实际上是封装了android特有的内存共享机制[Ashmem](http://elinux.org/Android_Kernel_Features#ashmem)匿名共享内存，简单来说，Ashmem驱动通过在内核的一个自定义[slab](https://en.wikipedia.org/wiki/Slab_allocation)缓冲区中初始化一段内存区域，然后通过mmap把申请的内存映射到用户的进程空间中（通过[tmpfs](https://en.wikipedia.org/wiki/Tmpfs)），这样子就可以在用户进程中使用这里申请的内存了。由于整个申请内存的过程并不再Java层上，可以很明显的看出使用MemoryFile申请的内存实际上是并不会占用