## Chapter 8 CPU Back-End Optimisations:
- Inefficiencies occur in the CPU back-end (BE) after the front-end (FE) fetches and decodes instructions. However, BE can’t execute said instruction due to two main reasons Memory Bound and Core Bound which causes it to overload and stall as it can’t get the required resources i.e. data-cache miss. 
- Only optimise code for BE when TMA points to a high “Back-End Bound” metric which is then broken down into mem or core bound.
- Memory Bound:
  - Occurs when an application executes many mem accesses and spends time waiting for them to finish, in this case, we would need to optimise how we can better access memory i.e. fewer accesses or better mem subsystem.
  - In TMA, the memory bound metric estimates a fraction of some slots where the CPU is stalled due to demand for load or store instructions. Would need to identify memory access that contributes to this metric e.g. ```perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp ``` (records l3 misses) and ```perf report -n --stdio``` (outputs a report of funcs with highest misses) i.e. ```99.95%    33811   a.out     [.] foo ``` (revisit chapter 6.1.4 if unclear)
  - Once identified we can use a few perf tricks:
    - <ins> Cache-Friendly Data Structures: </ins>
      - Fetching a variable from main memory takes > 100 clock cycles compared to the few it takes from the cache, to prevent this write code that is cache-friendly with spatial and temporal locality in mind, to allow for efficient fetching of data from caches. think in terms of cache lines and not individual variables in memory.
      - <ins> Access data sequentially: </ins> To exploit the spatial locality of the caches is to make sequential memory accesses. This makes HW prefetcher, recognise the mem access pattern and bring the next chunk ahead of time. For example, a standard bin search is not sequential as it accesses elements all over the memory that do not share the same cache line. To avoid this, one could store elements of the array using the Eytzinger layout. This maintains an implicit bin search tree packed into an array (using bfs-like-layout like in heaps).
      - <ins> Use appropriate containers: </ins> There are various containers in C++, one should know the performance trade-offs of each and why i.e. map vs unordered_map. Also, choose the data storage with what the code will do with it in mind e.g. storing objects in an array vs storing pointers to memory in an array (when size is big). The latter takes less amount of memory and will benefit operations that modify the array (less mem transferred). However, a linear scan through an array the former would be better as it's more cache-friendly (no indirect mem access)
      - <ins> Packing the data: </ins> Mem hierarchy utilization will improve by making data more compact. To pack data, one can use bitfields e.g. ```
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
      - <ins> Tune the code for memory hierarchy: </ins>
      - <ins> Explicit Memory Prefetching: </ins>
    - <ins> Optimizing For DTLB: </ins>
  
 

