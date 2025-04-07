# Data dependency
## 1. Background
The original paper (Improving Data Cache Performance by Pre-executing Instructions Under a Cache Miss) on runahead execution discusses a specific form of data dependency hazard:  
store instructions do not write data into the cache or main memory. They do, however, mark the referenced L1 data cache item INV, if the base register used in
address calculation is not INV and a cache miss does not occur. However, it does not account for the case when stores cause a cache miss or cannot compute their target addresses because their base register has an INV value.

## 2. Assumption
1. The Denver processor may use a runahead cache to handle store instructions: a store writes its value into the runahead cache, and subsequent dependent load instructions can read the value from the runahead cache.
2. Despite the presence of a runahead cache, the hazard mentioned in the background can still occur: if a store instruction has an unresolved target address, it cannot mark the corresponding cache line as invalid or write the value into the runahead cache. Specifically, consider a store instruction like `store value, address`. If the `address` cannot be resolved — for example, if it's marked as invalid or a cache miss occurs when fetching the address from memory — then the store cannot execute and may be treated as an invalid instruction. Subsequently, a `load reg, address` instruction may execute and read a stale value, resembling the speculative store bypassing issue seen in store-to-load forwarding scenarios.

## 3. Experiment Design - make writing to array invalid by flush the index
The following code is a minimal example designed to investigate the behavior of `store` instruction in runahead. It sets up a scenario where `miss_index` is flushed from the cache, causing the subsequent `buffer[miss_index] = 10` to potentially be discarded during runahead execution due to unresolved data dependencies. The final `load` from `reloadbuffer[buffer[1]]` is used as flush-reload side channel to detect whether the write to `buffer[1]` actually took effect. 
```
miss_index = 1; // one valued index used to access the buffer
cache_miss_mem; // variable used to trigger cache-missed load (trigger runahead)
buffer[NUM] = {8}; // all elements in buffer is 8

cacheflush(&missed_index);
cacheflush(&cache_miss_mem);
instruction barrier;

load cache_miss_mem; // trigger runahead
buffer[miss_index] = 10;  // buffer[1] = 10, but miss_index is cache-missed, could be discarded
tmp = reloadbuffer[buffer[1]]; // load in reloadbuffer

```

## 4. Hypothesis
If we observe the value  `8` in `reloadbuffer`, it indicates that the write `buffer[miss_index] = 10` did not take effect during speculative or runahead execution.

## 5. Result and discussion
We indeed consistently observe a stable cache hit on the value 8 in the reloadbuffer. This strongly indicates that during speculative execution, the store operation `buffer[miss_index] = 10` was either skipped or discarded due to the unresolved `miss_index` (e.g., a cache miss or invalid address during runahead/speculative execution).

### 5.1 How to distinguish speculative execution and runahead in this case
To eliminate the possibility that the effect is caused purely by speculative execution due to cache miss, we modify the experiment by not flushing the `miss_index` variable. In fact, we can even promote it to a register-resident variable to ensure its value is readily available.

Instead, we introduce a data dependency between `miss_index` and a floating-point instruction `fcvtzx` (covert floating point to integer). Since floating-point instruction could be discarded in runahead execution, operation `buffer[miss_index] = 10` will also be marked as invalid and discarded in this case.
```
miss_index = 1; // one valued index used to access the buffer, we will keep it in the buffer rather than flushing it.
cache_miss_mem; // variable used to trigger cache-missed load (trigger runahead)
buffer[NUM] = {8}; // all elements in buffer is 8

cacheflush(&cache_miss_mem);
instruction barrier;

load cache_miss_mem // trigger runahead
fcvtzx d0 -> x2; // d0 = 0
miss_index = x2 + miss_index;
buffer[miss_index] = 10;  // buffer[1] = 10
tmp = reloadbuffer[buffer[1]]; // load in reloadbuffer

```

Finally, after introducing data dependency between `miss_index` and floating point instruction, we still observe value `8` is hit in `reloadbuffer`. 

