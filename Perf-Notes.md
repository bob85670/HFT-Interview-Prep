## Chapter 8 CPU Back-End Optimisations:
- Inefficiencies occur in the CPU back-end (BE) after the front-end (FE) fetches and decodes instructions. However, BE can’t execute said instruction due to two main reasons Memory Bound and Core Bound which causes it to overload and stall as it can’t get the required resources i.e. data-cache miss. 
- Only optimise code for BE when TMA points to a high “Back-End Bound” metric which is then broken down into mem or core bound.
- Memory Bound:
  - Occurs when an application executes many mem accesses and spends time waiting for them to finish, in this case, we would need to optimise how we can better access memory i.e. fewer accesses or better mem subsystem.
  - In TMA, memory bound metric estimates a fraction of some slots where the CPU is stalled due to demand for load or store instructions. Would need to identify memory access that contributes to this metric e.g. ```perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp ``` (records l3 misses) and ```perf report -n --stdio``` (outputs a report of funcs with highest misses) i.e. ```99.95%    33811   a.out     [.] foo ``` (revisit chapter 6.1.4 if unclear)
  - Once identified we can use a few perf tricks:
    - <ins> Cache-Friendly Data Structures: </ins>
      - A
  
 

