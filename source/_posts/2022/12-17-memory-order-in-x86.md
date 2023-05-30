"title": "浅入 x86 内存序",
"date": "2022/12/17 17:10:42",
"categories": ["architecture"]
;;;

```log
[WARN] Using deprecated method `memory-order', it might be removed in the future.
```

## 内存序（Memory Order）

在 x86 中，内存屏障通常是不需要的，因为 x86 是比较严格的 TSO（不完全，因为始终没有“官宣”，但是按照定义来说比较接近，而且不同的厂商如 AMD/Intel 之间的实现也存在略微差别）。

百度找到了一份 [sc_tso.pdf](https://www.cis.upenn.edu/~devietti/classes/cis601-spring2016/sc_tso.pdf) 讲述了 TSO 的一些特点，其中包括程序顺序（program order）与内存序（memory order）之间的关系：

> 1. Whether they are to the same or different addresses (i.e., a=b or a≠b).If L(a) <p L(b) ⇒ L(a) <m L(b) /\* Load→Load \*/If L(a) <p S(b) ⇒ L(a) <m S(b) /\* Load→Store \*/If S(a) <p S(b) ⇒ S(a) <m S(b) /\* Store→Store \*/~~If S(a) <p L(b) ⇒ S(a) <m L(b)~~ /\* Store→Load \*/: Enable FIFO write buffer
> 2. When to the same address, every load gets its value from the last store before it.
>    If S(a) <p L(a) ⇒ S(a) <m L(a) /\* Store→Load \*/

其中 `<p` 表示的是程序顺序下的先后关系，而 `<m` 则是内存顺序下的先后关系。**注意这里不考虑编译器进行过的指令重排，这里的程序顺序也是编译完后的汇编顺序**。

内存序其实不一定要完全写回到内存才算，只要其他 CPU 能看到最终更改就行了（globally visible），例如最常见的 Store-Load，假如 Store(a, 100) 执行完后，那么其他任意的核心在执行 Load(a) 时结果都必须保证为 100；但是这不一定意味着内存中的 a 就一定是 100，有可能它还只是在 L3 Cache 中。

上面的引用处我还是稍微修改了一点，为了方便阅读；可以看到，对于所有的 Load/Store，三种 Load-Load、Load-Store、Store-Store 都是 SC 的，即内存序就是程序序。但是之所以我们说 TSO 要弱于 SC，是因为对于 Store-Load 这样的操作并没有这么严格，Store-Load 只有在当读写同一个地址（同一片内存时）才是成立的；读写不同内存时 Store-Load 下程序序是不能等价于内存序的。

能作证这一点的还有 [Intel SDM](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html) Volume 3 8.2.2 部分，这里也摘录了一下：

> 1. Reads are not reordered with other reads.
> 2. Writes are not reordered with older reads.
> 3. Writes to memory are not reordered with other writes, with the following exceptions:
>    * streaming stores (writes) executed with the non-temporal move instructions (MOVNTI, MOVNTQ, MOVNTDQ, MOVNTPS, and MOVNTPD); and
>    * string operations (see Section 8.2.4.1).
> 4. No write to memory may be reordered with an execution of the CLFLUSH instruction; a write may be reordered with an execution of the CLFLUSHOPT instruction that flushes a cache line other than the one being written.1 Executions of the CLFLUSH instruction are not reordered with each other. Executions of CLFLUSHOPT that access different cache lines may be reordered with each other. An execution of CLFLUSHOPT may be reordered with an execution of CLFLUSH that accesses a different cache line.
> 5. Reads may be reordered with older writes to different locations but not with older writes to the same location.
> 6. Reads or writes cannot be reordered with I/O instructions, locked instructions, or serializing instructions.
> 7. Reads cannot pass earlier LFENCE and MFENCE instructions.
> 8. Writes and executions of CLFLUSH and CLFLUSHOPT cannot pass earlier LFENCE, SFENCE, and MFENCE instructions.
> 9. LFENCE instructions cannot pass earlier reads.
> 10. SFENCE instructions cannot pass earlier writes or executions of CLFLUSH and CLFLUSHOPT.
> 11. MFENCE instructions cannot pass earlier reads, writes, or executions of CLFLUSH and CLFLUSHOPT.

其中比较重要的就是上面的第 1、2、3、5 点。翻译下来就是：

* 1\. Load-Load 下内存序等于程序序（不会被重排）
* 2\. Load-Store 下内存序等于程序序，注意这里写的是 older reads，也就是说当前 Store 指令（`MOV` 等）之前的 Load 指令都不会被重新排序
* 3\. Store-Store 下内存序等于程序序，例外情况是 non-temporal 和 rep mov 之类的指令（后面会提到，在写一般代码时不会太接触到）
* 5\. Store-Load 下只有当读写的都是同样的地址时内存序才等于程序序，否则对于不同的地址的读写依然会存在重排

到这里应该大概就了解了为何说 x86 架构普遍都是 TSO，以及为何 TSO 要弱于 SC。

## 多核心

**注意**上述例子都只在单核心中生效，包括 TSO 这些（写这篇文章的时候好像没考虑到单/多核心问题，可能要回炉重造了 FIXME）。对于多核心系统（multiple-processor），SDM Volume 3 8.2.2 同样也定义了：

> 1. Individual processors use the same ordering principles as in a single-processor system.
> 2. Writes by a single processor are observed in the same order by all processors.
> 3. Writes from an individual processor are NOT ordered with respect to the writes from other processors.
> 4. Memory ordering obeys causality (memory ordering respects transitive visibility).
> 5. Any two stores are seen in a consistent order by processors other than those performing the stores
> 6. Locked instructions have a total order.

重点就是，在多核心的情况下，单核心的写入顺序可以保持不变，但是很可能会穿插进其他核心的写操作。SDM 上附带了一个很好的例子来解释这一行为，考虑三个核心都在同时写入数据：

```
> Writes from individual processors:
 Processor#1 Processor#2 Processor#3
  Write A.1   Write A.2   Write A.3
  Write B.1   Write B.2   Write B.3
  Write C.1   Write C.2   Write C.3

> Actual writes (one of the example) in memory:
    Write A.1
    Write B.1
    Write A.2
    Write A.3
    Write C.1
    Write B.2
    Write C.2
    Write B.3
    Write C.3
```

P.S. 第四点的“因果律”可能一开始让人觉得有点摸不着头脑，可以参考 [Stack Overflow](https://stackoverflow.com/a/27374794) 的例子来了解，实际上就是保证如果单个变量写入且能被正确观测到，那么前面所有的变量写入都应该可以被观测到。个人理解 x86 在多核心的内存序上相对比较“难预测”，内存序依赖于观测到的值，也就是只有在“看”到了实际的值之后，才能知道哪些变量被正确地写入了。

## 一个栗子

一个我觉得很有“意思”的例子，来源一个知乎文章，介绍[内存屏障](https://zhuanlan.zhihu.com/p/454564295)所用的，原文代码有点乱，这里重新格式化并稍微精简了一下：

```cpp
#define _GNU_SOURCE
#include <sched.h>
#include <pthread.h>
#include <assert.h>

volatile int x, y, r1, r2;
static pthread_barrier_t barrier_start;
static pthread_barrier_t barrier_end;

static void* thread1(void* a) {
    while (1) {
        pthread_barrier_wait(&barrier_start);
        // run1
        x = 1;
        r1 = y;
        pthread_barrier_wait(&barrier_end);
    }
    return NULL;
}

static void* thread2(void* a) {
    while (1) {
        pthread_barrier_wait(&barrier_start);
        // run2
        y = 1;
        r2 = x;
        pthread_barrier_wait(&barrier_end);
    }
    return NULL;
}

int main() {  
    assert(pthread_barrier_init(&barrier_start, NULL, 3) == 0);
    assert(pthread_barrier_init(&barrier_end, NULL, 3) == 0);

    pthread_t t1;
    pthread_t t2;
    assert(pthread_create(&t1, NULL, thread1, NULL) == 0);
    assert(pthread_create(&t2, NULL, thread2, NULL) == 0);

    cpu_set_t cs;
    CPU_ZERO(&cs);
    CPU_SET(0, &cs);
    assert(pthread_setaffinity_np(t1, sizeof(cs), &cs) == 0);
    CPU_ZERO(&cs);
    CPU_SET(1, &cs);
    assert(pthread_setaffinity_np(t2, sizeof(cs), &cs) == 0);
  
    while (1) {
        // start()
        x = y = r1 = r2 = 0;
        pthread_barrier_wait(&barrier_start);
        pthread_barrier_wait(&barrier_end);
        // end()
        assert(!(r1 == 0 && r2 == 0));
    }
   
    return 0;
}

// gcc xxx.c -O2 -pthread && ./a.out
```

根据原作者阐述，这段程序必定会产生 `end()` 处引发的断言错误。那么我们来对着上面的多核心下的内存模型稍微解释一下吧，根据上面的定义，线程 t1 和线程 t2 同时写入变量 `x` 与 `y`，那么真实的写入顺序有可能是先写 `x` 或是先写 `y`；不过重点还是“读”的顺序，按照前文关于 Store-Load 序的解释，`r1` 和 `r2` **不一定**会读到写入的值（因为不是同一个核心)，x86 内存模型只保证假设真实内存写入顺序为 `x = 1; y = 1`，那么如果 `r1` 读到 `y == 1`，则 `r2` 必定也能读到 `x == 1`。

另外按照 SDM 官方的说法（位于 Volume 3 8.2.3.4），

> At each processor, the load and the store are to different locations and hence may be reordered. Any interleaving of the operations is thus allowed.One such interleaving has the two loads occurring before the two stores. This would result in each load returning value 0.

也就是说对于一个核心而言，只有在**该核心**上读写同一地址才是顺序的；否则就算多核心都读写同一地址，中间都有可能发生乱序。这两种解释都是行得通的。

解决方法可以是下面所述的内存屏障，即在写入 `x` 或 `y` 后引入内存屏障，保证下一条读指令执行前完成写入，即：

```c
// t1/run1
x = 1;
__asm__ __volatile__("mfence" ::: "memory");
r1 = y;

// t2/run2
y = 1;
__asm__ __volatile__("mfence" ::: "memory");
r2 = x;
```

这样一来，所有可能的路径都不再会产生 `r1` 和 `r2` 同时为零的情况了：

1. t1 执行到 `x = 1`，t2 执行到 `mfence`，那么 t1 必然能保证 `r1 = y = 1`，因为 t2 使用了 `mfence` 强制 CPU 按照 `x = 1; r2 = x` 的顺序执行（如果没有 `mfence` 就可能会出现 `r2 = x; r1 = y; x = 1; y = 1` 的内存读写顺序出现）
2. t1 执行到 `mfence`，t2 执行到 `y = 1`，同样，t2 必然能保证 `r2 = x = 1`

注意 `x` 与 `y` 是否能够被观测到与指令乱序没有太大关系，内存屏障只保证读写的顺序，但是不保证读写的值是否有效，后者是由 atomic 负责保证的，只是恰巧 x86 下绝大多数读写操作都是原子的，一定要区分好两者的关系 orz。

另一个粗暴的方法就是所有 mov 操作都带上 lock prefix，保证 total order。

总而言之，多核心下 x86 内存序能保证的只有写入的顺序，以及“因果律”读到的结果；但是无法保证多核心下读和写的顺序，要保证这种顺序需要依靠内存屏障或是总线锁。

## 内存屏障（Memory Fence）

前文列出的 SDM 卷 3 8.2.2 节里，用了好几个点叙述 `xFENCE` 指令，包括 `LFENCE`、`SFENCE` 和 `MFENCE`。

注意到 SDM 卷 3 8.2.5 节里的一句话：

> The SFENCE instruction (introduced to the IA-32 architecture in the Pentium III processor) and the LFENCE and MFENCE instructions (introduced in the Pentium 4 processor) provide memory-ordering and serialization capabilities **for specific types of memory operations.**

也就是说屏障在 x86 下基本只有“特殊”的内存操作才需要。所以先有一个概念：x86“常规”的指令下不是很需要内存屏障。那么我们继续往下就能看到这三条指令的定义：

> The SFENCE, LFENCE, and MFENCE instructions provide a performance-efficient way of ensuring load and store memory ordering between routines that **produce weakly-ordered results** and routines that consume that data. The functions of these instructions are as follows:
>
> * SFENCE — Serializes all store (write) operations that occurred prior to the SFENCE instruction in the program instruction stream, but does not affect load operations.
> * LFENCE — Serializes all load (read) operations that occurred prior to the LFENCE instruction in the program instruction stream, but does not affect store operations.2
> * MFENCE — Serializes all store and load operations that occurred prior to the MFENCE instruction in the program instruction stream.
>
> _2\. Specifically, LFENCE does not execute until all prior instructions have completed locally, and no later instruction begins execution until LFENCE completes. As a result, an instruction that loads from memory and that precedes an LFENCE receives data from memory prior to completion of the LFENCE. An LFENCE that follows an instruction that stores to memory might complete before the data being stored have become globally visible. Instructions following an LFENCE may be fetched from memory before the LFENCE, but they will not execute until the LFENCE completes._

上面加粗的那句“produce weakly-ordered results”，其实也说明了 x86 不仅仅只有 TSO，为了性能它也有更弱的内存模型（如 ARM、RISC-V 般的）。

回归正题，稍微地阐述一下这三条内存屏障指令：

* `SFENCE` 会保证该指令之前的所有 Store 操作不会“跨过”该指令（可能也是“Serializes”的意思？），但是不对 Load 做任何保证
* `LFENCE` 则保证该指令之前的所有 Load 操作不会“跨过”该指令，也同样不对 Store 做任何保证；注意这里带了一条尾注，指明 `LFENCE` 不仅仅能同步数据，也能同步指令，在其他指令没有执行完前不会执行 `LFENCE`，所以在一个[知乎回答](https://www.zhihu.com/question/29465982/answer/44465936)说明了这个特性被用在了内核 `RDTSC` 之前，以获取更加精确的时钟。
* `MFENCE` 则会保证之前所有的 Store/Load 操作都不会“跨过”该指令；注意在汇编的世界中（我认为）单条指令永远不等价于多条指令的“和”，例如这里的 `MFENCE != SFENCE + LFENCE`，原因就是“原子性”，如果用 `SFENCE + LFENCE` 替代 `MFENCE` 的话，有个比较大的问题就是两条指令中间有可能存在一定的“空窗期”：在两条指令中间，虽然 `SFENCE` 保证了当前核心之前的所有 Store 指令都完成了，但是现在空窗期间（执行 `LFENCE` 前）我们已经不能再保证 Store-Store 序了，因为其他 CPU 可能会执行任何的读写操作。这一点在 [Preshing 的文章](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/#storeload)中也提及到了：

  > However, those two barrier types are insufficient. Remember, the push  operation may be delayed for an arbitrary number of instructions, and  the pull operation might not pull from the head revision.
  >

  所以在汇编上我们必须要有一个 single instrument 来保证 atomicity，这跟 `inc`、`xchg` 等“混合”指令的作用是类似的，虽然它们都能拆分成多个指令，但是为了保证原子性必须要提供（就算非常精简以及 weakly-ordered 的 RISC-V 架构依然存在着 `xchg` 这样的指令，来保证原子性）。

总而言之，通常操作下我们并不需要关心 x86 的内存屏障，因为它已经“足够的强”了，但是面对其他情况下依然需要屏障。

P.S. 比较有意思的一段话：

> Intel does not guarantee that future processors will support this model.

看来 Intel 也非常想废弃掉 TSO 啊（笑）。

## Sequentially Consistent

内存屏障的其中一个作用，就是将 TSO 内存序变为更强的 SC 内存序，具体做法参考 [StackOverflow 的答案](https://stackoverflow.com/a/27635527)，里面列出了四种做法：

> 1. `LOAD` (without fence) and `STORE` + `MFENCE`
> 2. `LOAD` (without fence) and `LOCK XCHG`
> 3. `MFENCE` + `LOAD` and `STORE` (without fence)
> 4. `LOCK XADD` ( 0 ) and `STORE` (without fence)

并且指出 GCC 使用的就是方法 1，也就是 `STORE + MFENCE` 就能将 TSO 转变为 SC。这个比较好理解，因为 `MFENCE` 保证了指令之前的**所有读写**操作都对其他核心可见，那么如果我们每次写的时候都执行一次 `MFENCE`，就意味着每次写完就去强制读写可见，那么后面的读就必定只会读到最新的指。另外因为 SC 要保证 StoreLoad，所以就必须要 `MFENCE`（？）。

## Non-Temporal

上面提到了对于 NT（Non-Temporal）操作，我们是不能保证 Store-Store 序的，也就是说一个 NT 指令后立刻接上另外的 Store 指令，那么最终的结果是无法确定的，这也是 x86 中 relaxed-order 的体现。所以对于所有的 NT 指令，都建议带上 `SFENCE` 内存屏障。

> Non-Temporal SSE instructions (MOVNTI, MOVNTQ, etc.), don't follow  the normal cache-coherency rules. Therefore non-temporal stores must be  followed by an SFENCE instruction in order for their results to be seen  by other processors in a timely fashion.
>
> When data is produced and not (immediately) consumed again, the fact  that memory store operations read a full cache line first and then  modify the cached data is detrimental to performance. This operation  pushes data out of the caches which might be needed again in favor of  data which will not be used soon. This is especially true for large data structures, like matrices, which are filled and then used later. Before the last element of the matrix is filled the sheer size evicts the  first elements, making caching of the writes ineffective.
>
> For this and similar situations, processors provide support for  non-temporal write operations. Non-temporal in this context means the  data will not be reused soon, so there is no reason to cache it. These  non-temporal write operations do not read a cache line and then modify  it; instead, the new content is directly written to memory.
>
> Source: http://lwn.net/Articles/255364/ https://stackoverflow.com/a/37092

NT 与否其实是跟 cache 有很大关系，这里的解释说是 Non-Temporal 的所有操作都不走 cache（所以才叫“非临时”？）。还是翻一下 SDM 吧，首先直接找到 `MOVNTDQA` 的指令介绍吧，在卷 2 4.3 中，摘录关于 Non-Temporal 的部分：

> The non-temporal hint is implemented by using a write combining (WC) memory type protocol when reading the data from memory. Using this protocol, the processor does not read the data into the cache hierarchy, nor does it fetch the corresponding cache line from memory into the cache hierarchy. The memory type of the region being read can override the non-temporal hint, if the memory address specified for the non-temporal read is not a WC memory region. Information on non-temporal reads and writes can be found in “Caching of Temporal vs. Non-Temporal Data” in Chapter 10 in the Intel® 64 and IA-32 Architecture Software Developer’s Manual, Volume 3A.
>
> Because the WC protocol uses a weakly-ordered memory consistency model, a fencing operation implemented with a MFENCE instruction should be used in conjunction with MOVNTDQA instructions if multiple processors might use different memory types for the referenced memory locations or to synchronize reads of a processor with writes by other agents in the system. A processor’s implementation of the streaming load hint does not override the effective memory type, but the implementation of the hint is processor dependent. For example, a processor implementation may choose to ignore the hint and process the instruction as a normal MOVDQA for any memory type. Alternatively, another implementation may optimize cache reads generated by MOVNTDQA on WB memory type to reduce cache evictions.

这里简要地说明了这个指令的一些细节，例如它的作用就是将内存中的数据读到寄存器中，但是不经过缓存层（也不触发任何的 fetch），第二段则指出了该指令工作在 weakly-ordered 的内存序下，因此需要借助前文说的内存屏障来保证可见性（以及顺序性）。顺便这里也指路到了 SDM 卷 1 10.4.6.2（跟原文的卷号对不上，标错了？）：

> Data referenced by a program can be temporal (data will be used again) or non-temporal (data will be referenced once and not reused in the immediate future). For example, program code is generally temporal, whereas, multimedia data, such as the display list in a 3-D graphics application, is often non-temporal. **To make efficient use of the processor’s caches, it is generally desirable to cache temporal data and not cache non-temporal data.** Overloading the processor’s caches with non-temporal data is sometimes referred to as “polluting the caches.” The SSE and SSE2 cacheability control instructions enable a program to write non-temporal data to memory in a manner that minimizes pollution of caches.

也就是说 NT 宽泛来说（概念性）指的就是不会立刻被使用到的数据，也因此出于性能考虑也就不再进行缓存，将缓存让给 temporal data。因此从实现角度来说 NT 就是不经过缓存的数据，考据终了。

P.S. 在 SDM 卷 3 11.3 中能找到所有的 Memory Types 和相关解释。不贴原文了，整理（粗略地理解）了一下：

* Write Combining (WC)：对于一些无法被缓存（cache）的项目，会存在一个专门的 buffer（write combining buffer）来收集并“聚合”（combine）这些读写操作并推迟写入，以此来减少内存（或者 I/O）的读写次数；一些 serializing 指令（如上面提到的 `SFENCE`、`MFENCE` 以及 `CPUID` 等）会强行落地并清空 WC Cache。更多关于 WC Buffer 可以参考 [StackOverflow 上的答案](https://stackoverflow.com/a/49961612)，比较复杂，我也没仔细地去看了 orz。
* Write-through (WT)：所有的 Load/Store 都走 cache，但是每次在往 cache 中写入（Store）时都会直接同步到内存中；读取（Load）则依然会从 cache 中读取。因此 WT 比较适合写不敏感，但是要求与内存保持同步的一些场景（I/O？）。
* Write-back (WB)：所有的 Load/Store 都走 cache，数据会保持在 cache 中直到缓存策略决定该何时写回到内存。这就是我们最常用（也是性能最好的）一种 cache 模式。但是对于 I/O 等需要访问内存的外设（device）肯定就需要考虑一致性的问题了，同时也需要考虑缓存一致性的问题（cache coherency）。
* Write protected (WP)：只对 Load 操作走 cache，所有的 Store 的操作都会让缓存失效。

P.P.S. 有一个 `PREFETCH` 指令可以显式地将部分内存载入到缓存中，这也是在内核代码里看到很多 prefetch 的原理。具体可以看指令说明。

## folly/memcpy.S

这里我的兴趣点在于 `.L_NON_TEMPORAL_LOOP` 这个 label 上，首先可以找到哪里会跳转到这里来：

```nasm
// This threshold is half of L1 cache on a Skylake machine, which means that
// potentially all of L1 will be populated by this copy once it is executed
// (dst and src are cached for temporal copies).
#define NON_TEMPORAL_STORE_THRESHOLD $32768
...
cmp         NON_TEMPORAL_STORE_THRESHOLD, %rdx
jae         .L_NON_TEMPORAL_LOOP
```

也就是说当我们 `memcpy` 的大小超过了 L1 缓存（folly 取了 Skylake 作为基准）后，就不再借助缓存去拷贝了，而是直接利用上述的 `MOVNTx` 系列的 NT 指令绕过缓存，实现比较高性能的拷贝：

```nasm
.L_NON_TEMPORAL_LOOP:
        testb       $31, %sil
        jne         .L_ALIGNED_DST_LOOP
        // This is prefetching the source data unlike ALIGNED_DST_LOOP which
        // prefetches the destination data. This choice is again informed by
        // benchmarks. With a non-temporal store the entirety of the cache line
        // is being written so the previous data can be discarded without being
        // fetched.
        prefetchnta 128(%rsi)
        prefetchnta 196(%rsi)

        vmovntdqa   (%rsi), %ymm0
        vmovntdqa   32(%rsi), %ymm1
        vmovntdqa   64(%rsi), %ymm2
        vmovntdqa   96(%rsi), %ymm3
        add         $128, %rsi

        vmovntdq    %ymm0, (%rdi)
        vmovntdq    %ymm1, 32(%rdi)
        vmovntdq    %ymm2, 64(%rdi)
        vmovntdq    %ymm3, 96(%rdi)
        add         $128, %rdi

        cmp         %r8, %rsi
        jb          .L_NON_TEMPORAL_LOOP

        sfence
        jmp         .L_ALIGNED_DST_LOOP_END
```

可以看到 folly 借助 `movntdqa` 指令将数据读入到 AVX 寄存器 `ymmX` 中，再使用 `movntdq` 将寄存器的值写回到内存，并循环。最后，注意到这里还有个 `sfence` 指令，保证在拷贝完后数据对其他核心可见（但是拷贝途中无法保证，所以可能需要调用着确保不会出现 data race），符合 SDM 上的定义。

到此，基本上我们也理解了 x86 的 TSO 内存序（比 SC 弱的地方在于只能保证同一个地址的 Store-Load，但是可以通过 `MFENCE` 指令做到所有地址的 Store-Load 以实现 SC）、x86 的内存屏障的作用（通常程序中因为 TSO 的存在没有什么作用，但是对于如 NT 系列的特殊指令需要 weakly-ordered memory，就必须使用内存屏障了；不过为了程序的 portable 性，还是尽量地加上内存屏障操作，也能加深理解）。现在，我们再来尝试理解（困扰我多年的.jpg）C++ `std::memory_order` 了。

## Cache

关于 Memory Order，偶尔还是会提到缓存，因为像 Intel x86 等主流实现来说，基本都会带有缓存。那么我们对内存进行操作的时候就会存在一个中间的 cache 状态，这可能会导致内存序出现一些小小的差异（例如如果有一个简单的 cache，我们往同一个内存地址多次写入数据，那么实际就只是一直在往 cache 中写入数据，没有*及时地*写回到内存中去，其他 CPU 就无法观测，也就无法保证 Store-Load 等内存序了）。

这个问题就交由缓存一致性算法来解决，最终实现的效果就是在 x86 下无论我们怎么操作 cache 和内存，只要地址相同，就能保证其他 CPU 始终能观测到最新值，从而保证 Store-Load 序正确（上面的例子也就不会出现无法观测的问题）。而对于其他架构而言，应该也需要实现类似的效果，起码是需要有处理 atomic 的能力，即能让别的 CPU 观测到对内存操作的变化。

对于 x86 而言，常用的缓存一致性协议是 MESI，这个协议比较重要的就是“嗅探机制”（snoop），也是通过 snoop 通过总线来得知其他 CPU 的操作；具体可以查看[维基百科 MESI](https://en.wikipedia.org/wiki/MESI_protocol) 的页面，这部分我暂时还没深入地去了解，大概只知道这些了（逃。

## 编译器的指令重排

有时候在大量存在变量的读写时，编译器会选择性地进行指令重排（例如你能在 GDB 中发现很多“诡异”的跳转，常常执行到一半的时候跳回到开头执行变量的初始化等），这就是区别于 CPU 指令重排的编译器指令重排。编译器重排基本都是在编译期做的，有两种方式可以抑制：

1. 变量使用 `volatile` 关键字，针对的是该变量；
2. 使用 `asm volatile("" ::: "memory")`，针对的是该语句上下的重排，跟 CPU 的内存屏障很像，就是避免语句之前的读写操作重新排序到该语句之后。另外语句起作用的地方是 `"memory"`，所以第一个参数无论是任何汇编指令都会有抑制编译器重排的效果。

大概也就是这些了，文章重点依然在 x86 上，暂时不讨论编译器了（其实是不会 QAQ）。

后续会写一篇 C/C++ 的 Memory Order，来对“真实世界”中的内存模型（希望能）有一个更稍微透彻的理解~~以及咕咕咕~~。
