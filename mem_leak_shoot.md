**一次内存泄漏的定位过程**

**背景**

狮子王服务首次使用了并发cache(wait-free)的, 预计是要比其他使用线程cache使用的内存会少.

**现象**

上线后狮子王服务占用内存却明显高于别的服务, 且一直在递增.

![](media/image1.png)

中间的曲折是重新发布过, 可以看到最高到了7.5%, 而我们别的服务一般在1.5%左右, 所以猜测存在内存泄漏.

**首次解决**

因为是首次使用concurrent\_cache, 所以怀疑是concurrent\_cache导致的.

于是在测试机压测concurrent\_cache功能, 压测结果发现确实涨到了很高, 最高涨到了占用内存1.2G,
但是到1.2G就不再增长了. 反复阅读concurrent\_cache.h代码也没发现导致内存泄漏的地方,
于是怀疑是进程占用了过高的内存, 但是系统没有及时释放.

再次仔细阅读concurrent\_cache.h, 只有一处new的地方, 但是传递给了bthread\_timer\_add定时释放了.
联想到tcmallloc是thread cache的, bthread\_timer\_add的线程是一个专门处理定时任务的线程,
也就是在业务线程分配内存, 然后在定时任务线程释放内存. 于是定时任务线程中的内存越来越多, 而业务线程却一直需要新的内存new
参数. 内存分布极不均匀导致的伪内存泄漏, 这也符合测试环境压测到1.2G不再增长的原因.
因为tcmalloc一个线程的内存过大会释放内存到central\_cache,
于是业务线程再需要内存的时候直接从central\_cache获取, 不再需要向系统申请. 逻辑合情合理.

**解决**

于是重新组织了concurrent\_cache的代码, 使用thread\_local, 业务线程申请, 业务线程释放.

然后上线

**再次解决**

上线后观察监控, 发现依然继续增长. 也是一度达到了5%以上. 再次怀疑还是concurrent\_cache的问题. 再次在测试环境压测,
发现测试环境很稳定, 狮子王内存占用一直处于比较低的状态. 没法从测试环境发现问题

走读代码也没法发现问题. 只能上工具

**HeapProfiler**

brpc文档指出

|                              |
| ---------------------------- |
| 在实践中heap profiler对原程序的影响不明显。 |

于是线上开profiler, 由于狮子王使用了brpc server, 而brpc内置了heap profiler功能.  
如果没有使用brpc server, 而想使用brpc的heap profiler功能可以:

<table>
<tbody>
<tr class="odd">
<td>Go<br />
brpc::StartDummyServerAt(8888/*port*/);</td>
</tr>
</tbody>
</table>

使用brpc的heap profiler需要设置环境变量

<table>
<tbody>
<tr class="odd">
<td>Go<br />
TCMALLOC_SAMPLE_PARAMETER=524288</td>
</tr>
</tbody>
</table>

524288表示多少内存记录一次.

待线上内存涨到一定高度, 可以使用

<table>
<tbody>
<tr class="odd">
<td>Plaintext<br />
pprof --text localhost:9002/pprof/heap<br />
pprof --pdf localhost:9002/pprof/heap</td>
</tr>
</tbody>
</table>

导出pdf需要安装 ghostscript与graphviz工具 (可使用yum直接安装)

导出pdf可以在本地直接查看.

从pdf中可以清晰的看出调用链路的各内存分配

**\[lion\_heap.pdf\]**

可以明显看出以下链路分配了大量内存

![](media/image2.png)

阅读对应的代码发现,

![](media/image3.png)

DestroyLionKingGame会返回错误, 但是直接返回了M2E\_SUCCESS, 导致错误链表没有被处理,
链表越来越长导致了内存泄漏.

解决:

![](media/image4.png)

**systemtap定位内存泄漏**

**原理**

**监控 malloc**：

每次调用 malloc 时，记录返回的指针地址和分配的内存大小。

**监控 free**：

每次调用 free 时，检查释放的指针地址是否已经记录，如果记录存在，则删除。

**检测内存泄漏**：

程序结束时，检查记录中是否还有未释放的指针。如果存在，说明这些是泄漏的内存。

systemtap脚本

<table>
<tbody>
<tr class="odd">
<td>SQL<br />
// 用于存储 malloc 返回的指针地址及其分配的大小<br />
global allocations<br />
<br />
// 监控 malloc 调用<br />
probe process("/path/to/program").function("malloc") {<br />
ptr = returnval // malloc 返回的指针地址<br />
size = $size // malloc 分配的大小（传入参数）<br />
allocations[ptr] = size // 将指针地址和分配大小存储到全局哈希表<br />
printf("Allocated %d bytes at %p\n", size, ptr)<br />
}<br />
<br />
// 监控 free 调用<br />
probe process("/path/to/program").function("free") {<br />
ptr = $ptr // free 的参数是需要释放的指针<br />
if (ptr in allocations) {<br />
printf("Freed memory at %p (size: %d bytes)\n", ptr, allocations[ptr])<br />
delete allocations[ptr] // 删除记录，表示内存已释放<br />
} else {<br />
printf("Warning: Attempt to free untracked pointer at %p\n", ptr)<br />
}<br />
}<br />
<br />
// 程序结束时，检查未释放的指针<br />
probe end {<br />
printf("\n--- Memory Leak Report ---\n")<br />
foreach (ptr in allocations) {<br />
printf("Leaked memory at %p (size: %d bytes)\n", ptr, allocations[ptr])<br />
}<br />
printf("--------------------------\n")<br />
}</td>
</tr>
</tbody>
</table>

sudo stap malloc\_leak\_detect.stp -c "/path/to/program"

**加上调用栈**

<table>
<tbody>
<tr class="odd">
<td>SQL<br />
probe process("/path/to/program").function("malloc") {<br />
allocations[returnval] = $size<br />
printf("Allocated %d bytes at %p\n", $size, returnval)<br />
print_ubacktrace() // 打印调用堆栈<br />
}</td>
</tr>
</tbody>
</table>

## 更方便的方式
1. curl 127.0.0.1:19500/hotspots/heap
2. https://dreampuf.github.io/GraphvizOnline/  把输出粘到这里就可以渲染出来
