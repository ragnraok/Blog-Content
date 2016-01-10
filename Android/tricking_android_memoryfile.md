Title: Tricking Android MemoryFile
Date: 2016-1-10
Slug: tricking-android-memoryfile

之前在做一个内存优化的时候，使用到了MemoryFile，由此发现了MemoryFile的一些特性以及一个非常trickly的使用方法，因此在这里继续

###What is it
MemoryFile是android在最开始就引入的一套框架，其内部实际上是封装了android特有的内存共享机制[Ashmem](http://elinux.org/Android_Kernel_Features#ashmem)匿名共享内存，简单来说，Ashmem在Android内核中是被注册成一个特殊的字符设备，Ashmem驱动通过在内核的一个自定义[slab](https://en.wikipedia.org/wiki/Slab_allocation)缓冲区中初始化一段内存区域，然后通过mmap把申请的内存映射到用户的进程空间中（通过[tmpfs](https://en.wikipedia.org/wiki/Tmpfs)），这样子就可以在用户进程中使用这里申请的内存了，另外，Ashmem的一个特性就是可以在系统内存不足的时候，回收掉被标记为"unpin"的内存，这个后面会讲到，另外，MemoryFile也可以通过Binder跨进程调用来让两个进程共享一段内存区域。由于整个申请内存的过程并不再Java层上，可以很明显的看出使用MemoryFile申请的内存实际上是**并不会**占用Java堆内存的。

MemoryFile暴露出来的用户接口可以说跟他的名字一样，基本上跟我们平时的文件的读写基本一致，也可以使用InputStream和OutputStream来对其进行读写等操作：

```Java
MemoryFile memoryFile = new MemoryFile(null, inputStream.available());
memoryFile.allowPurging(false);
OutputStream outputStream = memoryFile.getOutputStream();
outputStream.write(1024);
```

上面可以看到``allowPurging``这个调用，这个就是之前说的"pin"和"unpin"，在设置了allowPurging为false之后，这个MemoryFile对应的Ashmem就会被标记成"pin"，那么即使在android系统内存不足的时候，也不会对这段内存进行回收。另外，由于Ashmem默认都是"unpin"的，因此申请的内存在某个时间点内都可能会被回收掉，这个时候是不可以再读写了

###Tricks
MemoryFile是一个非常trickly的东西，由于并不占用Java堆内存，我们可以将一些对象用MemoryFile来保存起来避免GC，另外，这里可能android上有个BUG：

在4.4及其以上的系统中，如果在应用中使用了MemoryFile，那么在dumpsys meminfo的时候，可以看到多了一项Ashmem的值：

![](static/images/memoryfile_1.jpg)

可以看出来虽然MemoryFile申请的内存不计入Java堆也不计入Native堆中，但是占用了Ashmem的内存，这个实际上是算入了app当前占用的内存当中

但是在4.4以下的机器中时，使用MemoryFile申请的内存居然是**不算入**app的内存中的：

![](static/images/memoryfile_2.jpg)

这个应该是android的一个BUG，但是我搜了一下并没有搜到对应的issue，搞不好这里也可能是一个feature

而在大名鼎鼎的Fresco当中，他们也有用到这个bug来避免在decode bitmap的时候，将文件的字节读到Java堆中，使用了MemoryFile，并利用了这个BUG然这部分内存不算入app中，这里分别对应了Fresco中的[GingerbreadPurgeableDecoder](https://github.com/facebook/fresco/blob/master/imagepipeline/src/main/java/com/facebook/imagepipeline/platform/GingerbreadPurgeableDecoder.java)和[KitKatPurgeableDecoder](https://github.com/facebook/fresco/blob/master/imagepipeline/src/main/java/com/facebook/imagepipeline/platform/KitKatPurgeableDecoder.java)，Fresco在decode图片的时候会在4.4和4.4以下的系统中分别使用这两个不同的decoder