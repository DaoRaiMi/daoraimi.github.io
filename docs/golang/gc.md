# <center>垃圾回收器
## 简介
垃圾回收器的职责就是跟踪**堆内存**(Heap Memory)的使用情况，释放掉那些不再使用的内存，保留那些仍然还在使用的内存。

GO语言使用的是**非分代的**(non-generational), **三色**(tri-color)**并发**(concurrent)**标记清除**(mark and sweep)的垃圾回收器。

## 垃圾回收的对象
1. 堆中的分配的内存
2. goroutine栈中的指针、引用类型的变量。 

栈中的其他变量是不需要进行垃圾回收的。

## 垃圾回收过程
当垃圾回收器开始工作时，会有以下四个阶段：

* 结束-SWEEP
* 开始-MARK
* 结束-MARK
* 开始-SWEEP

### 结束-SWEEP
这个阶段其实也就是为本次GC做一些准备工作。
1. 开始STW，这将导致所有的P都进入到一个安全点(safe-point)，也可以说是所有的goroutine也就都暂停了。
> 在1.14版本之前，没有引入goroutine的抢占调度机制，那么会出现由于某些P上的goroutine并没有发起一个函数调用，进入不了安全点，导致这个暂停过程延长的情况。所以在1.14版本中引入了抢占式调度就解决了这个问题。

2. 如果本次GC是在预期执行之前被强制执行的，那么很可能此时还存在一些未被清理的span，所以此处会清理掉那些在上次GC中被标记为可清理的span.

### 开始-MARK
1. 把gcphase从_GCoff修改为_GCmark, 启用写屏障(write barrier)，启用辅助GC(mutator assists)，把root标记任务放入到队列中。直到所有的P都开启了写屏障后，才会开始扫描对象，这是通过STW来保证的。
> 这里的写屏障表示的是插入写屏障(insertion write barrier)和删除写屏障(deletion write barrier)，也叫混合写屏障(hybrid write barrier)

2. 停止STW。从此刻开始，Scheduler就会让标记worker开始标记工作，同时有必要的话，mutator 所在的P也会协助进行一部分标记工作。写屏障会对覆盖指针、任何指针写操作的新值，进行灰色处理。新创建的对象会立即被标记为黑色。

3. GC执行root标记任务。这包括扫描所有的stack，将所有的全局变量置灰，并且将任何在非堆内存中的运行时数据结构的堆指针置灰。在扫描栈时，需要暂停当前的goroutine，然后把栈上所有的找到的指针都置灰，最后再恢复当前goroutine.

4. GC从work queue中依次取出灰色对象，并把当前的灰色对象置黑，并把当前对象指向的对象置灰。

5. 由于GC的工作是分散在local cache中的，所以GC使用了一个分布式的termination 算法来检测什么时候没有root标记任务或没有灰色对象了。此时GC会转换到标记结束阶段。

### 结束-MARK
1. STW

2. 将gcphase设置为_GCmarktermination，禁用gc workers和mutator assist.

3. 执行housekeeping(比如flushing mcahces)

### 开始-SWEEP
1. 把gcphase设置为_GCoff，设置sweep状态，禁用写屏障。
2. 停止STW。从此刻开始，新创建的对象是白色的。
3. 开始执行并发SWEEP

## GC触发时机
1. 默认情况下每隔2min会触发一次。
   
2. GC百分比达到后也会触发GC，这个百分比默认值是100。表示：在下一次GC开始之前，**允许分配的堆内存**与**上次GC结束后仍然存活的内存**的比值。
> 比如上次GC结束后，堆中有存活对象2MB，那么下次GC就必须在堆中存活对象达到4MB或达到之前开始。

1. 应用开发者手动调用`runtime.GC()`也会立即触发GC，但是这个操作会阻塞直到GC完成。