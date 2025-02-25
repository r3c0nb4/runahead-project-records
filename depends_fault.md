## Incorrect data load (maybe) caused by runahead
## How does runahead handle store/load instructions(Background)

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
Instruction Windows for Out-of-order Processors, the experiment introduced a runahead cache to handle load and store operations during runahead mode.

<u>In traditional Runahead execution, store results are not stored, causing dependent load instructions to be invalid and leading to performance loss.</u> To address this, a Runahead Cache is introduced to store pseudo-retired store (all instructions in runahead will not be retired anyway) results and enable Store-to-Load Forwarding. During execution, stores first attempt to forward their results from the Store Buffer. If a store has already pseudo-retired, dependent loads fetch the data from the Runahead Cache instead. To prevent data pollution in normal execution, stores never write their results to the data cache. <u> However, a challenge arises when a store has an invalid address (The target address of the store instruction has not been determined yet, or other reasons), as dependent loads may incorrectly read stale data. </u>

## Runahead store-load-forwarding in denver 2
In the Denver processor, runahead may also implement a store-load forwarding mechanism. If there is a cache similar to the runahead cache, load instructions with data dependencies can also execute during runahead. Therefore, the challenge mentioned above may also exist.

### How to distinguish runahead store-load-forwarding and speculative store-load-forwarding in denver 2
In an in-order processor, Store-to-Load Forwarding can only occur between instructions that have entered the pipeline and reached the execution unit. Since instructions execute strictly in program order, a Load must wait for preceding Stores to execute before accessing data. Forwarding happens when a Store has executed, its address and data are resolved, and a subsequent Load accesses the same address. The processor can then forward the value from the Store Buffer without waiting for memory writes. Forwarding cannot apply to instructions not yet in the pipeline, as they haven’t been decoded or reached execution, making dependency checking and forwarding impossible.


## Uncover incorrect data load in denver runahead mode
### Experiment hypothesis and assumption
Assumption: we assume the runahead implementation in denver 2 should not discard dependent load instructions.

Hypothesis: if a store instruction writes to an invalid address—specifically, an address that has not yet been resolved—then a load instruction that depends on this store may read incorrect data.

### Experiment design

We initialize two buffers, `fake_buffer` and `secret`, where `secret` is the data we want to leak during runahead execution. At the same time, we initialize two (two level) pointers `**a` and `**b`, as well as a pointer `*c`. First, we set pointer `a` to point to the address of `c`, and pointer `c` to point to the address of `secret`. Then, we flush (clear) pointer `a` to ensure it is not in the cache.

Next, we make pointer `b` point to the address that pointer a is pointing to. Since pointer `a` has been flushed and is not in the cache, this triggers runahead execution, and for a brief period, pointer `b` cannot resolve the address that `a` is pointing to.

Afterwards, we insert a certain number of dummy instructions (nops, movs etc.) to ensure that subsequent operations do not immediately appear in the pipeline, which helps eliminate any potential interference from store-load forwarding in the pipeline.

Finally, we access the address pointed to by `b` and point it to fake_buffer. Then, we use pointer `c` to access `reloadbuffer` (`reloadbuffer[c[idx] << 12]`), in order to reload the cache and leverage the data temporarily leaked from secret through runahead execution.

![fault-depend](./imgs/depend_error.png)
### Result and discussion
We found that we were able to extract the contents of secret from `reloadbuffer`, which indicates that when we set `b` to point to the address that `a` is pointing to, but `a` is not in the cache. This triggers runahead execution, and `b` cannot resolve the address stored in `a`.

After executing the dummy instructions, when we attempt to make the address pointed to by `b` point to `fake_buffer`, this instruction fails to execute and is discarded because the address that b points to has not been resolved.

In architectural execution, this operation would actually cause pointer `c` to point to `fake_buffer`. However, since `b` does not know that the address it is pointing to is actually `c`, it cannot correctly update it.

Finally, when we access memory using pointer `c`, due to the previous instruction being discarded, `c` still accesses secret instead of `fake_buffer`, successfully leaking the data.


### Detailed Algorithm
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

### Detailed Explaination for algorithm
1. At first we initialize and flushes the `reloadbuffer` to clear any previous cache state. 
2. Then, `memory_slot_ptr` is set to point to `memory_slot`, which initially points to the `secret_buffer` containing the secret data. 
3. The `memory_slot_ptr` is flushed to ensure it is not cached before further execution. 
4. Next, `memory_slot_ptr_tmp` is set to point to `memory_slot`, triggering runahead execution due to the `flush memory_slot_ptr` operation. 
5. Dummy instructions (`nops; mov reg, reg`) are inserted to ensure that the following instructions are not in the CPU pipeline. 
6. Then, `memory_slot_ptr_tmp` is redirected to point to a `fake_buffer`. 
7. Afterward, we access `reloadbuffer[memory_slot[index] << 12]` to leverage the cache side-channel for monitoring speculative memory accesses.





