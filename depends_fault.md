## Incorrect data load (maybe) caused by runahead
## How does runahead handle store/load instructions

### Load store forwarding
In an out-of-order processor, reordering load and store instructions is challenging. Consider the following scenario:

```
/*Example 1*/
store A
store B
store X // we dont know if X == A
load A
```

If the address of a load instruction has already been resolved, but there are previous store instructions in the reorder buffer (ROB) whose addresses have not yet been resolved, the processor does not know the most recent value written to that address.

To optimize this, processors implement load-store forwarding. When a store instruction writes its result into the store buffer, a subsequent load instruction checks the buffer for a matching address. If a matching store is found, the processor assumes that the most recent store to the same address contains the correct value. The load then forwards this value from the store buffer to the register.

However, this optimization can introduce Spectre Variant 4 (Speculative Store Bypass, SSB) vulnerabilities.

As shown in the previous example 1, the processor does not immediately know whether the final address `X` is equal to `A`. If `X` and `A` turn out to be the same, the load instruction might still speculatively assume that the required value is the one written by the first store instruction. This results in the load using an outdated value instead of the correct one, potentially leading to security vulnerabilities.

### Load store implementations in runahead
In runahead execution, the implementation of load and store operations is likely similar to load-store forwarding in out-of-order processors.

In the paper Runahead Execution: An Alternative to Very Large
Instruction Windows for Out-of-order Processors, the experiment introduced a runahead cache to handle load and store operations during runahead mode. The mechanism of this cache closely resembles the way out-of-order execution handles memory dependencies, ensuring that speculative loads can obtain values from stores before they are committed to memory.

### Runahead in denver 2
However, the Denver processor is an in-order processor, which means the handling of store-load forwarding may differ from that of an out-of-order processor.

We cannot determine whether Denver implements store-load forwarding when runahead is not triggered. However, without runahead, an in-order processor has a very small execution window (assuming DCO is not triggered).

For example, if a load instruction has not yet been decoded and is not present in the pipeline, then store-load forwarding cannot occur. This is because there is no speculative execution window where a later load could observe an earlier store before it reaches pipeline.

## Uncover incorrect data load in denver runahead mode
Based on the discussion above, we assume that the Denver processor generally does not exhibit store-load forwarding during normal execution. This is because, as an in-order processor, its execution window remains small under typical operation.

However, runahead execution significantly expands the execution window, making it highly likely that store-load forwarding occurs during runahead mode. The increased instruction window allows loads to access stores earlier in the store buffer, mimicking the behavior of out-of-order execution in this specific scenario.

### Algorithm
```
variable:
**memory_slot_ptr           // point to memory_slot
**memory_slot_ptr_tmp       // use this pointer to point to *memory_slot_ptr, and trigger runahead
*memory_slot                //point to secret_buffer or fake_buffer
secret_buffer               //what we want to leak in reloadbuffer
fake_buffer                 //final architectural execution commit
reloadbuffer

1. init and flush reload buffer
2. *memory_slot_ptr = &memory_slot //point to the address of pointer: memory_slot
3. memory_slot = secret_buffer  //point to secret at first
4. flush memory_slot_ptr
5. ...
6. memory_slot_pr_tmp = *memory_slot_ptr //point to memory_slot(memory_slot_ptr points to memory_slot), at the same time, activate runahead, because of flush in code 4
7. Dummy instructions, make sure the following instructions are not in the pipeline
8. *memory_slot_ptr_tmp = fake_buffer    //point to fake buffer
9. reloadbuffer[memory_slot[index] << 12]
10. reload reloadbuffer and check cache hit (make sure to exclude the cache hit of fake_buffer, in architectural execution, fake_buffer will be accessed for whatever).
```

### Explaination for algorithm
1. At first we initialize and flushes the `reloadbuffer` to clear any previous cache state. 
2. Then, `memory_slot_ptr` is set to point to `memory_slot`, which initially points to the `secret_buffer` containing the secret data. 
3. The `memory_slot_ptr` is flushed to ensure it is not cached before further execution. 
4. Next, `memory_slot_ptr_tmp` is set to point to `memory_slot`, triggering runahead execution due to the `flush memory_slot_ptr` operation. 
5. Dummy instructions (`nops; mov reg, reg`) are inserted to ensure that the following instructions are not in the CPU pipeline. 
6. Then, `memory_slot_ptr_tmp` is redirected to point to a `fake_buffer`. 
7. Afterward, we access `reloadbuffer[memory_slot[index] << 12]` to leverage the cache side-channel for monitoring speculative memory accesses.


### Result and discussion
We can see the signal in reloadbuffer from `secret_buffer`
I considered a possibility to explain the result:

At the beginning, when `memory_slot_pr_tmp = *memory_slot_ptr` is executed, since memory_slot_ptr was flushed, this might trigger runahead execution and expand the instruction window. Even though we later perform the operation `*memory_slot_ptr_tmp = fake_buffer`, at this point, `memory_slot_ptr` has not been resolved yet. Therefore, this instruction `*memory_slot_ptr_tmp = fake_buffer` may be discarded or bypassed due to the dependency between it and `memory_slot_pr_tmp = *memory_slot_ptr` (cache-miss-dependent instructions will be discarded in runahead mode). Subsequently, the following instruction that extracts a value from `memory_slot` still retrieves the contents of `secret_buffer`. This happens because, although the value pointed to by `memory_slot` was modified in the previous instruction to point to `fake_buffer`, the instruction was discarded or bypassed. As a result, `memory_slot` still points to `secret_buffer`, causing the secret buffer to be leaked.




