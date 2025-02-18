## Experiment: runahead vs architectural execution with fault
We use program exception errors to capture the microarchitectural 
execution behavior of the processor. For now, we have not adopted 
the Spectre-v1 approach, as we are currently unable 
to distinguish whether the behavior of Spectre-v1 is caused by 
runahead execution or branch prediction execution.

### Algorithm
```
signal_handler(){
    1. reload the reloadbuffer 
    2. check which index is fetched into cache
}

main(){
    /*Operation 5, 6 is considered as microarchitecture execution, the results will never be committed*/

    operation0. Register the signal handler function

    operation1. Flush the reloadbuffer

    operation2. Flush a variable `data`
    ...
    operation3. Load data, this operation may make processor enter runahead mode

    operation4. Illegal memory access

    operation5. Dummy instructions such as `mov reg, reg`, `nop`, avoid operation 6   enter the pipeline together with operation 3, because there are at least two load
    units in denver, have to avoid superscaler parallel execution.

    operation6. access reloadbuffer: `load reloadbuffer[10 << 12]`
}
```

### Implementation
I mainly considered two methods to crash the program.
1. Access `NULL` pointer: `*((volatile char*)0)`. But this approach may have an issue: when accessing a NULL pointer, the process generally goes through TLB miss, page walk, and then page fault. On the Denver processor, this situation may also trigger the processor to enter runahead mode. According to the Denver 1 documentation:
"Run-ahead uses the idle time that a CPU spends waiting on a long-latency operation to discover cache and DTLB misses further down the instruction stream and generates prefetch requests for these misses." However, another report states: "implements a ‘run-ahead’ feature that continues to execute after a data-cache miss." This means that both operation 3 and operation 4 could potentially trigger runahead execution. To avoid ambiguity, we refine our approach based on this idea.
2. Use `mmap` to allocate a read-only memory region `buffer` and access this memory (`load buffer[0]`) before operation 3 to ensure that the physical page is allocated and that it is cached in the TLB cache and the data cache. In operation 4, we write data to this memory region. Since it is set as read-only, this will trigger a segfault, causing execution to jump to our designed signal handler function. The signal handler then performs a reload operation to check whether reloadbuffer[10 << 12] has been accessed.
   
### Results
The experiment is divided into two groups. In the first group, we access the data variable (which was flushed) in operation 3. We run the experiment multiple times, and almost every time, we observe that the block at index 10 in the `reloadbuffer` has been cached. This strongly suggests that runahead was triggered, and it successfully prefetched the block in the `reloadbuffer`.

In the second group of experiments, we did not access the `data` variable in operation 3. As a result, the initial cache miss load does not occur. Instead, we directly trigger the write error, causing a jump to the signal handler function. In this case, there are no cache hits in the reloadbuffer.

However, after repeatedly running the experiment, we found that in the second group, where we removed operation 3 with the intention of not triggering runahead execution, every 1000 executions, we observed that 2-5 times the block at index 10 in the reloadbuffer was cached. This suggests that the program somehow still executed some subsequent instructions before the page fault and access error, possibly through some other mechanism.

### Discussion
For this issue, I have conducted the following analysis and made some guesses:
1. When writing to a read-only page, runahead execution could potentially be triggered. The MMU needs to check the page table's read/write permissions, and this process might take some time. The processor might classify this as a long latency operation. The Denver 2 processor has made optimizations over Denver 1, although very few documents describe these optimizations. However, it is possible that the conditions for triggering runahead execution have become more lenient, not just limited to cache misses. The published paper about Denver 1 also states that processor will enter runahead mode with there is a long latency operation, not merely cache miss.
As for why reloadbuffer cache hits are not commonly observed, the reason might be that the time interval between the write instruction entering the pipeline and the page fault being triggered is relatively short. Once the page fault occurs, the signal handler intervenes. This quick sequence of events may not allow the runahead mechanism to prefetch the block into the cache before the fault is handled.

2. The impact of DCO (Dynamic Code Optimization) could also be a potential reason. During repeated execution of the program, the processor likely analyzes and identifies optimizable portions of the instruction blocks being executed repeatedly, and it may perform load/store reordering. However, due to the presence of illegal memory access, the DCO's optimization might be seen as erroneous, leading to the invalidation or clearing of the optimized microcode in the DCO cache. As a result, we don't frequently observe cache hits in the reloadbuffer, but only occasionally in some runs.

3. Another possible factor is speculative execution. If Denver is an out-of-order processor, this phenomenon would be entirely plausible. However, the documentation does not explicitly mention whether Denver 2 is an out-of-order processor. My speculation is that Denver 2 is still a in-order processor. The design intention behind the Denver series processors was to replace the complex and expensive out-of-order execution engines with DCO (Dynamic Code Optimization), and it has been confirmed that DCO is integrated into Denver 2, making it unlikely that an out-of-order execution unit was implemented.Since Denver 2 is likely still an in-order processor, I believe the probability of speculative execution being the cause of this phenomenon is low.





















