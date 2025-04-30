# Store memory permission check bypass
## 1. Background
Memory read and write operations in a processor typically require permission checks, such as read, write, and execute permissions.
In some processor designs, the write permission check is deferred until the commit stage.
If a store instruction attempts to write data to an invalid address, the data may still be temporarily written into the store buffer during out-of-order execution.
Eventually, when the instruction reaches the commit stage, the processor will perform the permission check, and due to the invalid access, an exception will be triggered.

## 2. Illegal store in runahead
We perform illegal write operations during the runahead phase to observe whether these writes enter the store buffer or other bypass structures before the permission check is performed.

## 3. Experiment Design
We base our experiments on data dependency errors during the runahead phase.
We initialize a `buffer` with all elements set to `1`, and set up a `read_only memory` region with all values initialized to `0`.
We then use a cache-missing `index` to access the `buffer` and attempt to write the value `2` to `buffer[index]`.
Because the index causes a cache miss, the processor is unable to resolve the target address during runahead execution. As a result, `buffer[index]` is neither updated with the value `2` nor marked as invalid.
Finally, we use a conditional structure `if(buffer[0] == 1)`to suppress any architectural execution errors: if `buffer[index] == 1`, we write the value `1` to the `read_only_memory` and then use this value as an index to access  `reloadbuffer`.
```
buffer[] = {1};
read_only_memory[] = {0}

index = 0;
flush(index);
buffer[index] = 2;
if(buffer[0] == 1){
    read_only_memory[0] = 1;
    tmp = reloadbuffer[read_only_memory[0] << 12]
}
```

## 4. Hypothesis
If we see value 1 hitted in reloadbuffer, we can reach a conclusion: memory permission check for store instruction in the read only memory is bypassed.

## 5. Result
We observed that value 1 is fetched in the reloadbuffer. (The hit rate is not high, but the behavior is obvious)




