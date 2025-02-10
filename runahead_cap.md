## Experiment Background
Runahead will checkpoint the processor state after encountering a high-latency cache miss. While waiting for the data to be fetched into the cache, it continues executing subsequent instructions to identify potential new cache misses for prefetching. However, the results of instructions executed during the runahead phase will be discarded. After runahead execution ends, the processor will roll back to the previous checkpoint and resume normal execution.

## Experiment Design
For the execution of a piece of code (initially causing a cache miss), we divide it into two parts. The first part is the runahead execution triggered by the cache miss. The results of this execution will not be committed and are considered microarchitectural execution. The second part occurs after the runahead execution ends, where the processor rolls back to the checkpoint and resumes normal execution with commit.

We aim to distinguish these two parts through certain methods. To achieve this, we assume the existence of a continuously changing variable `var`. Suppose `var` holds the value 1 during runahead execution, but its value changes to 4 after the rollback following runahead. This allows us to differentiate runahead execution from subsequent normal execution based on the value of `var`.

We construct a reloadbuffer (256 × 4096):

1. First, we introduce a cache miss.
2. Then, we access the reloadbuffer using the value of var as an index `reloadbuffer[var << 12]`.
3. Finally, we perform a reload operation to check which part of the reloadbuffer has been accessed.

### Expected experiment result
1. During the runahead execution phase, since the value of `var` is 1, the corresponding part of the `reloadbuffer` indexed by 1 will be accessed (`reloadbuffer[1 << 12]`).
2. After the runahead execution ends and the processor rolls back to the checkpoint to resume normal execution, the value of `var` changes to 4, resulting in the access of a different part of the `reloadbuffer` indexed by 4.
3. We will see two cache hit in `reloadbuffer` in reload operation: `realoadbuffer[1 << 12]` is fetched during runahead process, `reloadbuffer[4 << 12]` is fetched during architectural execution phrase.

## Experiment setup
In the experiment, we choose to use the performance counter (or another processor clock) as the continuously changing value.

1. First, we flush both the `reloadbuffer[256 * 4096]` and a variable `data` from the cache.
2. Next, we access the variable `data`, which is not in the cache. This triggers the processor to enter runahead execution.
3. We insert some padding instructions to prevent subsequent instructions from being executed in the pipeline simultaneously with the load data instruction (avoid parallel execution in pipeline).
4. We then take the lower 8 bits of the performance counter (`index = performance_counter & 0xFF`) as an `index`.
5. Using this `index`, we access the reloadbuffer at `reloadbuffer[index << 12]`.
6. Finally, we perform a reload operation to check which parts of the `reloadbuffer` have been accessed.
   
## Result
In the end, we did not observe two cache hits. If the value of the lower 8 bit of performance counter is 100, there was only a single cache hit at index 100. Currently, we are unable to confirm the value of the performance counter during runahead execution because the results of these computations are not committed.:w