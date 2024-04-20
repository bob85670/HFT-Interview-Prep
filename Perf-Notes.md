## Chapter 8 CPU Back-End Optimisations:
- Inefficiencies occur in the CPU back-end (BE) after the front-end (FE) fetches and decodes instructions. However, BE can’t execute said instruction due to two main reasons Memory Bound and Core Bound which causes it to overload and stall as it can’t get the required resources i.e. data-cache miss. 
- Only optimise code for BE when TMA points to a high “Back-End Bound” metric which is then broken down into mem or core bound.
- <ins> Memory Bound Optimisations </ins>:
  - Occurs when an application executes many mem accesses and spends time waiting for them to finish, in this case, we would need to optimise how we can better access memory i.e. fewer accesses or better mem subsystem.
  - In TMA, the memory bound metric estimates a fraction of some slots where the CPU is stalled due to demand for load or store instructions. Would need to identify memory access that contributes to this metric e.g. ```perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp ``` (records l3 misses) and ```perf report -n --stdio``` (outputs a report of funcs with highest misses) i.e. ```99.95%    33811   a.out     [.] foo ``` (revisit chapter 6.1.4 if unclear)
  - Once identified we can use a few perf tricks:
    - <ins> Cache-Friendly Data Structures: </ins>
      - Fetching a variable from main memory takes > 100 clock cycles compared to the few it takes from the cache, to prevent this write code that is cache-friendly with spatial and temporal locality in mind, to allow for efficient fetching of data from caches. think in terms of cache lines and not individual variables in memory.
      - <ins> Access data sequentially: </ins>
        - To exploit the spatial locality of the caches is to make sequential memory accesses. This makes HW prefetcher, recognise the mem access pattern and bring the next chunk ahead of time. For example, a standard bin search is not sequential as it accesses elements all over the memory that do not share the same cache line. To avoid this, one could store elements of the array using the Eytzinger layout. This maintains an implicit bin search tree packed into an array (using bfs-like-layout like in heaps).
      - <ins> Use appropriate containers: </ins>
        - There are various containers in C++, one should know the performance trade-offs of each and why i.e. map vs unordered_map. Also, choose the data storage with what the code will do with it in mind e.g. storing objects in an array vs storing pointers to memory in an array (when size is big). The latter takes less amount of memory and will benefit operations that modify the array (less mem transferred). However, a linear scan through an array the former would be better as it's more cache-friendly (no indirect mem access)
      - <ins> Packing the data: </ins>
        - Mem hierarchy utilization will improve by making data more compact. To pack data, one can use bitfields e.g. ```
  struct S {
unsigned a:4; unsigned b:2; unsigned c:2;
}; // S is only 1 byte
``` VS ``` struct S {
unsigned a; unsigned b; unsigned c;
}; // S is sizeof(unsigned int) * 3 ``` This works if you know the number of bits you need ahead of time. By doing this you greatly reduce the amount of mem transferred back and forth and save cache space. Comes at the cost of accessing every packed element since a, b, and c share the same machine word so the compiler would need to perform some bitwise ops such as >> and & to load and << and | to store. If the additional computation is cheaper than the transfer delay do this. Also, arrange members in order of decreasing size to avoid access padding.
      - <ins> Aligning and padding: </ins> Another trick for improving usage of the memory subsystem is aligning the data. In the situation where an object of the size 16 bytes occupies two cache lines, you need to have two cache lines ready to fetch but can be avoided using the alignas keyword in C++. A variable is better accessed if it is stored at a mem address divisible by the size of the variable i.e. a double is 8 bytes and should be stored at an address divisible by 8.
        - ```
           Make an aligned array
          alignas(16) int16_t a[N];
          // Objects of struct S are aligned at cache line boundaries
          #define CACHELINE_ALIGN alignas(64)
          struct CACHELINE_ALIGN S {
          //...
          };
          ```
        This can cause holes of unused bytes that can increase memory bandwidth, i.e. S above uses 40 bytes, so we have 24 bytes unused (64 - 40) for every line that has S. Padding would be good in cases where we have cache contentions and false sharing (threads accessing variable of a struct using the same cache line).
          - ```
            #define CACHELINE_ALIGN alignas(64) struct S {
            int a; // written by thread A
            CACHELINE_ALIGN int b; // written by thread B
            };
            ```
          With dynamic allocation via malloc, the returned memory is guaranteed to meet alignment requirements but you can benefit from stricter alignment. Also with SMID code addresses need to be divisible by 16, 32 or 64, vec types given by SMID intrinsic headers are annotated to ensure alignment.  
      - <ins> Dynamic memory allocation: </ins>
        - Using dynamic memory allocation i.e. malloc, causes problems such as speed, scalability and fragmentation. Also, problems with start-up threads racing to allocate memory to the same region. Using custom allocators such as arena allocator is faster and has low overhead as allocators don't execute sys calls for every mem allocation. Also, very flexible as you can have your own mem allocation strats based on mem regions provided by the OS.
        - A simple strat would be to maintain two allocators with diff arenas (one for hot and cold) allowing hot data to share a cache line (improves bandwidth, utilization and spatial locality). Also better TLB utilisation, as hot data would occupy fewer pages. Custom allocators can also use thread-local storage per thread allocation so no sync between (good if app is based on thread pool and doesn't spawn a lot of threads).
      - <ins> Tune the code for memory hierarchy: </ins>
        - Perf depends on size of cache at a level, for example, use loop tilting to break size of data to chunks so they can fit in L2.
      - <ins> Explicit Memory Prefetching: </ins>
        -  Can be very bad when used wrong, consumes CPU resources, is not portable (won't work for multiple platforms) and if you use the wrong size of mem block or prefetch to often can evict needed data from cache.
        -  When to use? When the hardware can't effectively pre-fetch as there is no clear access pattern which in turn causes lots of cache misses and you can estimate a good prefetch window add ``` __builtin_prefetch() ``` well before the load so it's ready ( but if you do it too early you may pollute the cache with data not needed rn and evict other data required) (hard to get right ik lmao).  
    - <ins> Optimizing For DTLB: </ins>
      - As we know, the TLB is a fast per-core cache that allows us to translate virtual addresses into physical addresses. Without we would have to page walk up multiple layers of page tables (BAD!). The hierarchy goes as follows L1 ITLB (instructions), L1 DTLB (data), and L2 STLB (shared between instruction and data). A L1 ITLB miss has a small penalty due to out-of-order execution (executes next instruction instead of waiting) but a miss in the STLB invokes a noticeable page walk as the CPU is stalled. (random numbers to remember page size is normally 4KB l1 can take few 100 ~1MB but l2 thousands). Using larger page sizes, the TLB entry can cover a larger memory range thus better utilisation of the TLB slots!
      - <ins> Explicit Hugepages: </ins>
        - Huge pages are part of the system's memory presented as a huge page file system (hugelbfs) that can be accessed using system calls e.g. mmap, can allocate dynamically using the lib "libhugetlbfs" that overrides malloc calls.
      - <ins> Transparent Hugepages: </ins>
        - Linux offers Transparent Hugepage's (THP) which auto manages large pages and is transparent for applications, in Linux you can enable THP which dynamically switches to huge pages when large blocks of memory are needed. Has two modes of operation (system-wide: The kernel assigns huge pages to any process when it is possible so don't need to reserve manually && per-process: kernel only assigns huge pages individually per-process memory areas)
      - <ins> Explicit vs Transparent Hugepages: </ins>
        - EHP are reserved in virtual memory upfront, THPs are not. With THPs kernel attempts to assign a THP and if it fails, a standard 4KB page is given which happens transparent to the user. The allocation process involves several kernel processes to make space in virtual memory (latency overhead and some fragmentation/swapping to make space). EHP are not subject to these things and is available to use on all segments (like text which is good for DTLB and ITLB) while THP are only available for dynamically allocated memory regions.
        - Less config needed for THP though which is better for faster experiments.
- <ins> Core Bound Optimisations: </ins>
  - Core bound bottlenecks can be defined in two ways, 1: Shortage in hardware compute resources (limited throughput) and 2: Dependencies between software instructions (increased latency). The idea with core bound optimisations is decreasing the number of instructions so we can free up hardware compute resources and decrease the amount of dependencies causing it to stall less (removing stall is a memory-bound issue).
  - <ins> Inlining Functions: </ins>
    - As we know, inlining a function brings the code into the caller scope which mitigates the overhead invoked by calling the function (storing variables on the stack etc). Modern compilers are very smart and normally inline functions as they see fit, but as a programmers, we can force inlines with new information.
    - How does the compiler choose when to inline? They rely on a sort of cost model, for example, the LLVM compiler bases it on computing cost and a threshold for each function call (inlining happens if cost < threshold) (cost == # and the type of instructions in that function). Thresholds are normally fixed but can be varied under circumstances. Tiny funcs, funcs with single call site are always linlined and large funcs aren't (too much code bloat takes longer to compile). Recursive funcs cant inline themselves and fucns that are referred to by a pointer are not fully inlined (have to stay in binary).
    - How do we as programmers know when to force inlines? Profiling Data! (How hot the prologue and epilogue are i.e. moving data to stack and restoring) if both are super hot, could benefit from forced inline to avoid such overhead. ``` [[gnu::always_inline]] ``` attribute can be used. (Make sure to profile after if perf increases, always measure as inlining can cause other stuff to happen so it is hard to predict without data).
  - <ins> Loop Optimisations: </ins>
    - <ins> Low-level optimizations: </ins>
      - Most transformations happen by the compiler, but sometimes the programmer needs to step in. Below are a few low-level optimizations.
      - Loop Invariant Code Motion (LICM): Move invariants (expressions that don't change) outside of the loop and store them in a temp variable to be used inside.
      - Loop Unrolling:
  
 

