## Malloc Lab

> Author: Jay Wei  GitHub@gitjaywei     2025/11/20

#### 0. 实现一个简单的分配器

> 源于CSAPP书中9.9.12. 一节内容；

- 基于隐式空闲链表，使用带边界标记的立即合并方式，构造一个分配器；

- 我们并不会调用底层的sbrk, brk等函数，不会直接操作系统的堆内存，相反，我们会采用一个内存模型系统，类似于一个沙盒，在此基础上我们能自由的写，调试，运行甚至破坏我们实现的分配器；

- 该内存模型系统由memlib.c包提供，具体如下：

  ~~~c
  static char *mem_heap;     // 指向模拟堆的首字节
  static char *mem_brk;      // 指向模拟堆尾字节的下一字节
  static char *mem_max_addr; // 指向模拟堆最大合法地址的下一地址
  
  // mem_init - 初始化模拟堆
  void mem_init(void)
  {
      // 分配一块大小为 MAX_HEAP 的内存作为模拟堆，并让 mem_heap 指向它的起始地址
      mem_heap = (char *)Malloc(MAX_HEAP);
      // 初始化时，堆的当前 breakpoint 与堆的起始地址相同
      mem_brk = (char *)mem_heap;
      // 计算并设置堆的最大地址界限
      mem_max_addr = (char *)(mem_heap + MAX_HEAP);
  }
  
  // mem_sbrk - 模拟系统 sbrk 函数，将堆扩展 incr 字节，并返回新区域的起始地址
  /*
   * 在这个模型中，堆不能缩小
   * 参数: incr - 堆需要扩展的字节数。如果为负，则表示尝试缩小堆(此模型不支持)
   * 返回值：成功时，返回扩展前堆的 breakpoint 地址（即新分配区域的起始地址）
   *        失败时（如内存不足或尝试缩小堆），返回 (void *)-1，并设置 errno 为 ENOMEM
   */
  void *mem_sbrk(int incr)
  {
      // 保存当前的 brk 地址，这将是新分配区域的起始地址
      char *old_brk = mem_brk;
  
      // 检查请求是否有效：
      // 1. 不能是缩小堆的请求（incr < 0）
      // 2. 扩展后的 brk 不能超过堆的最大地址界限
      if ((incr < 0) || ((mem_brk + incr) > mem_max_addr))
      {
          errno = ENOMEM; // 设置错误号为“内存不足”
          fprintf(stderr, "ERROR: mem_sbrk failed. Ran out of memory...\n");
          return (void *)-1; // 返回一个无效指针表示失败
      }
  
      // 调整 brk 指针，完成堆的扩展
      mem_brk += incr;
  
      // 返回新分配内存块的起始地址
      return (void *)old_brk;
  }
  ~~~

##### 通用分配器的设计

- 我们需要实现的分配器包含在源文件mm.c中，并且需要给出三个接口：

  ~~~c
  extern int mm_init(void);
  extern void *mm_malloc(size_t size);
  extern void mm_free(void *ptr);
  ~~~

  - mm_init用于初始化分配器，成功返回0，失败返回-1；
  - mm_malloc与mm_free分别用于模拟库函数malloc和free；

- 分配器将使用下图所示的块结构与组织方式：

  ![malloclab_0](./assets/malloclab_0.png)
  
  - 第一个字是不使用的填充字，用于双字边界对齐；
  - 填充字后面是一个特殊的序言块，只由头部和脚部组成，初始化时创建并且永不释放；
  - 末尾是结尾块，只有头部；
  - 分配器使用一个全局的heap_listp指向序言块；
  
  > 注1：为什么需要序言块和结尾快？用于消除边界条件；因为带边界的立即合并需要检查前一个块和后一个块，如果没有序言块和结尾块就还需要引入边界检查；
  >
  > 注2：为什么需要填充字？因为我们需要保证块的有效载荷是双字对齐的，而序言块是双字，块头部是单字，所以需要再加一个填充块，具体见图中虚线；

##### 空闲链表的基本常数和宏

> 块指针：块中有效载荷的指针；

- ~~~c
  #define WSIZE 4             // 单字大小，也是头部/脚部大小
  #define DSIZE 8             // 双字大小
  #define CHUNKSIZE (1 << 12) // 扩展堆的默认大小，1<<12 = 4096，也就是4KB
  
  // 工具宏，返回二者中的较大者
  #define MAX(x, y) ((x) > (y) ? (x) : (y)) 
  
  // 打包块的头部/脚部 (将块大小与标记位按位或)
  #define PACK(size, alloc) ((size) | (alloc)) 
  
  // 读取地址p处的一个字
  #define GET(p) (*(unsigned int *)(p))
  // 在地址p处存储一个字
  #define PUT(p, val) (*(unsigned int *)(p) = (val))
  
  // 提取地址p处字(头部/尾部)中的块大小信息
  #define GET_SIZE(p) (GET(p) & ~0x7)
  // 提取地址p处的字中的块分配标记
  #define GET_ALLOC(p) (GET(p) & 0x1)
  
  // 计算块指针bp指向的有效载荷所处块的头部的地址
  #define HDRP(bp) ((char *)(bp) - WSIZE)    
  // 计算块指针bp指向的有效载荷所处块的尾部的地址
  #define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
  
  // 计算块指针bp的下一个块指针
  #define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE((char *)(bp) - WSIZE))
  // 计算块指针bp的上一个块指针
  #define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE((char *)(bp) - DSIZE))
  ~~~

  - eg. size_t size = GET_SIZE(HDRP(NEXT_BLKP(bp))); 拥用于计算块指针bp的下一个块大小；

##### 初始化隐式空闲链表

- 在能够调用mm_malloc或mm_free之前，必须先用mm_init初始化堆，得到一个空的隐式空闲链表；

  ~~~c
  int mm_init(void)
  {
      // 申请4个字用于存放填充字，序言块和结尾块
      // 先让heap_listp指向填充字便于后面的数据填充
      if ((heap_listp = mem_sbrk(4 * WSIZE)) == (void *)-1)
          return -1;
  
      PUT(heap_listp, 0);                            // 填充填充字
      PUT(heap_listp + (1 * WSIZE), PACK(DSIZE, 1)); // 填充序言块头部
      PUT(heap_listp + (2 * WSIZE), PACK(DSIZE, 1)); // 填充序言块尾部
      PUT(heap_listp + (3 * WSIZE), PACK(0, 1));     // 填充结尾块头部
      heap_listp += (2 * WSIZE);                     // 让heap_listp指向序言块正中
  
      // 先拓展一次堆(实参需要除一个单字大小是因为extend_heap函数接受字数量而不是字节数量)
      if (extend_heap(CHUNKSIZE / WSIZE) == NULL)
          return -1;
      return 0;
  }
  ~~~

- extend_heap函数：

  ~~~c
  static void *extend_heap(size_t words)
  {
      char *bp;
      size_t size;
  
      // 保证双字对齐，words必须是偶数
      size = (words % 2) ? (words + 1) * WSIZE : words * WSIZE;
  
      // 调用mem_sbrk扩展堆
      if ((long)(bp = mem_sbrk(size)) == -1)
          return NULL;
      // 如果拓展成功，bp现在是新分配区域的起始地址，也就是结尾块的下一个地址，也就是新块的有效载荷的地址
  
      PUT(HDRP(bp), PACK(size, 0));         // 设置新块的头部(原来的结尾块变成了新块的头部)
      PUT(FTRP(bp), PACK(size, 0));         // 设置新块的尾部
      PUT(HDRP(NEXT_BLKP(bp)), PACK(0, 1)); // 紧接着放置新的结尾块(结尾块是新分配区域的最后一个字)
  
      return coalesce(bp); // 尝试与前一个块合并
  }
  ~~~

  - extend_heap函数不止在mm_init中被调用，后面实现mm_malloc时，如果mm_malloc找不到合适的空闲块也会调用extend_heap；
  - 最后的coalesce函数我们还未实现，主要是为了避免旧的空闲链表以空闲块结束，我们直接追加空闲块会造成假碎片；

##### 释放与合并空闲块

- mm_free函数用于释放一个已经分配的块，并尝试与相邻的两个块合并；

  ~~~c
  void mm_free(void *bp)
  {
      size_t size = GET_SIZE(HDRP(bp)); // 从当前块的头部读取块大小
      PUT(HDRP(bp), PACK(size, 0));     // 改写头部：设置为「size，未分配」
      PUT(FTRP(bp), PACK(size, 0));     // 改写脚部：设置为「size，未分配」
      coalesce(bp);                     // 尝试与前后块进行合并
  }
  ~~~

- coalesce函数：块合并的核心

  ~~~c
  static void *coalesce(void *bp)
  {
  
      size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp))); // 获取前块是否已分配
      size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp))); // 获取后块是否已分配
      size_t size = GET_SIZE(HDRP(bp));                   // 当前块大小
  
      // 情况 1：前后块都已分配 → 无需合并
      if (prev_alloc && next_alloc) return bp;
  
      // 情况 2：前已分配，后未分配 → 向后合并
      else if (prev_alloc && !next_alloc)
      {
  
          size += GET_SIZE(HDRP(NEXT_BLKP(bp))); // 合并后大小 = 当前块 + 后块
          PUT(HDRP(bp), PACK(size, 0));          // 更新当前块新的头部
          PUT(FTRP(bp), PACK(size, 0));          // 更新当前块新的脚部（覆盖后块原本脚部）
      }
  
      // 情况 3：前未分配，后已分配 → 向前合并
      else if (!prev_alloc && next_alloc)
      {
          size += GET_SIZE(HDRP(PREV_BLKP(bp)));   // 合并后大小 = 当前块 + 前块
          PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0)); // 更新前块的头部（新的大块头部）
          PUT(FTRP(bp), PACK(size, 0));            // 更新前块的脚部（新的大块脚部）
          bp = PREV_BLKP(bp);                      // 更新 bp，使其指向合并后的块起始处（前块）
      }
  
      // 情况 4：前后块均未分配 → 三块全部合并
      else
      {
          // 合并后大小 = 前块 + 当前块 + 后块
          size += GET_SIZE(HDRP(PREV_BLKP(bp))) + GET_SIZE(FTRP(NEXT_BLKP(bp)));
          PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0)); // 头部更新在前块位置
          PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0)); // 脚部更新在后块位置
          bp = PREV_BLKP(bp);                      // 合并后的大块起点是前块
      }
  
      return bp; // 返回合并后的块指针
  }
  ~~~

  - 这里就能看出为什么序言块也结尾块能避免边界情况了，我们无需关心当前块是第一个块还是最后一个块，反正序言块和结尾块永远被标记为已分配；

##### 分配块

- ~~~c
  void *mm_malloc(size_t size)
  {
      size_t asize; // 经过调整后的块大小
      size_t extendsize;
      char *bp;
  
      // 如果要求分配0字节，直接返回NULL
      if (size == 0)
          return NULL;
  
      /* 调整size大小，使其满足双字对齐要求 */
      // 因为双字对齐要求，所以块大小最小为4个字(头部+有效载荷+尾部)
      // 如果size小于两个字，则块大小直接调整为4个字
      if (size <= DSIZE)
          asize = 2 * DSIZE;
      // 首先头部+尾部占据一个DSIZE
      // 然后size需要调整为(size + (DSIZE - 1)) / DSIZE，式中DSIZE-1是向上取整的“凑整因子”
      // 最后相加再乘以DSIZE，得到调整后的块大小
      // size能不能调整为 size / DSIZE + 1 或者 (size + DSIZE) / DSIZE 呢？显然不行，当size为DSIZE倍数时会多一个DSIZE
      else
          asize = DSIZE * (((DSIZE) + size + (DSIZE - 1)) / DSIZE);
  
      // 在空闲链表中寻找合适的空闲块(三种策略，记得么？)
      if ((bp = find_fit(asize)) != NULL)
      {
          place(bp, asize); // 找到 → 在该空闲块中放置新分配的块，必要时分割
          return bp;        // 返回有效载荷的起始地址
      }
  
      // 没有找到合适的空闲块 → 扩展堆空间
      extendsize = MAX(asize, CHUNKSIZE); // 扩展大小至少是CHUNKSIZE
      if ((bp = extend_heap(extendsize / WSIZE)) == NULL)
          return NULL;
      place(bp, asize); // 在刚拓展的新空闲块中放置，必要时分割
      return bp;        // 返回有效载荷的起始地址
  }
  ~~~

  - 搜索函数find_fit和放置函数place并未实现，这两个函数也是接下来的malloclab的核心；



#### 正式开始Malloc Lab

