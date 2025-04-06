# Fingerprint runahead
## 1 Denver 2 cpu
The Denver 2 CPU is a superscalar processor. While documentation for Denver 1 explicitly states it operates in a strictly in-order manner, we currently lack definitive confirmation about Denver 2's execution behavior. Although preliminary experiments (e.g., instruction latency analysis) provide circumstantial evidence, such methods cannot conclusively determine whether Denver 2 retains in-order execution or adopts out-of-order capabilities. Further microarchitectural profiling or direct vendor clarification would be required for authoritative verification.

## 2. Fingerprint Runahead execution
### 2.1 What is unique in runahead?
Based on later research on runahead execution and IBM Power9's runahead implementation, runahead mechanisms are likely to discard floating-point instructions because they are rarely used for address calculation (e.g., pointer arithmetic). However, in general speculative execution (e.g., branch prediction or prefetching), there is no inherent reason to discard floating-point operations, as they may still contribute to program logic and data flow. The distinction arises because runahead focuses specifically on prefetching memory addresses, while broader speculation aims to optimize overall execution efficiency.

### 2.2 Experiment assumption
We hypothesize that floating-point operations are discarded during runahead execution in the Denver 2 processor.

### 2.3 Experiment design
We employ the Spectre V1 gadget to test this hypothesis. <u>Please noted that, `index` is larger than `size`, we are leaking another buffer rather than `fake_buffer`, and variable `size` is not in the cache. </u> 
```
/*Spectre V1*/
if (index < size){	
	pick = reloadbuffer[fake_buffer[index] << 12];
}
```

We use the `fcvtzs` instruction to convert the value of a floating-point register (`reg_floating`, set to `0`) into an integer register (`reg_integer`), then add the value of this integer register to the `index`. 

If the floating-point instruction (`fcvtzs reg_floating -> reg_integer`) is discarded, the integer register (`reg_integer`) will not receive the converted value, causing the subsequent load operation (`pick = reloadbuffer[fake_buffer[index] << 12]`) to use an invalid index. This would prevent the array access from executing properly, confirming that floating-point operations are ignored during runahead.
```
/*fcvtzs instruction is discarded*/
if (index < size){	
    reg_floating = 0;
    fcvtzs reg_floating -> reg_integer;
    index = index + reg_integer;
	pick = reloadbuffer[fake_buffer[index] << 12];
}
```

We replace the floating-point `fcvtzs` instruction with a latency-equivalent instruction (1-2 `mov` operations) as a control group. Since the control group contains no floating-point operations, the subsequent load instruction may execute correctly. If so, we will observe cache hits in the `reloadbuffer`, confirming that non-floating-point instructions are preserved during runahead.
```
/*Control group*/
if (index < size){	
    mov #0 -> reg_interger0
    mov reg_integer0 -> reg_integer1
    index = index + reg_integer1;
	pick = reloadbuffer[fake_buffer[index] << 12];
}
```

### 2.4 Hypothesis
If the control group (using `mov` instructions) shows cache hits in the reloadbuffer, while the experimental group (with the floating-point `fcvtzs` instruction) does not, this demonstrates two conclusions:

1. Runahead Execution Was Very Possibly Triggered: The observed behavior aligns with runahead mode (speculative prefetching during long-latency stalls).

2. Floating-Point Instructions Are Discarded in Runahead: The `fcvtzs` instruction was ignored during runahead.

## 3. Result
In the experimental group containing floating-point instructions, we observed no cache hits. However, in the control group (without floating-point instructions), stable cache hits were detected.

## 4. Compare with speculaion execution
We need to compare whether the `fcvtzs` instruction gets discarded during speculative execution.

### 4.1 Assumption
Assuming Spectre v1 can still be triggered without causing a cache miss, i.e., when the branch resolution time is relatively short. For example, in a condition like `if (a < ***b_ptr)`, multiple levels of pointer dereferencing introduce several `ldr` instructions, which may exhaust backend resources and cause pipeline stalls. This could make speculative execution more pronounced.

### 4.2 Experiment Design
For the Spectre v1 gadget, we make the following modification: we use a three-level chained pointer (`*size_ptr1 = &size; **size_ptr2 = &size_ptr1; ***size_ptr3 = &size_ptr2`)to reference the variable `size`, and we preload each pointer and the size variable in advance to ensure that no cache miss occurs during execution, thus avoiding runahead execution. At the same time, since the Denver 2 processor may only have two units capable of handling load instructions simultaneously, performing three consecutive dereferences (`if(index < ***size_ptr3)`) of `size_ptr3` within the branch instruction could lead to backend resource contention.

In addition, within the branch condition, we introduce a dependency between `index` and a floating-point instruction `fcvtzs`, in order to observe whether a cache hit signal still occurs in the end.

```
/*fcvtzs instruction is discarded or not in speculation*/
*size_ptr1 = &size;
**size_ptr2 = &size_ptr1;
***size_ptr3 = &size_ptr2; //make sure everything is loaded in the cache
if (index < ***size_ptr3){	
    reg_floating = 0;
    fcvtzs reg_floating -> reg_integer;
    index = index + reg_integer;
	pick = reloadbuffer[fake_buffer[index] << 12];
}
```
### 4.3 Hypothesis
1. If we observe cache hit signal finally: dereference the three level pointer could cause more pronouced speculation execution, and floating point instructions are not skipped in speculative execution.
2. If not: speculative execution window is too small, or fcvtzs is also discarded in speculation execution.
   
### 4.4 Result
In such setup, we are still able to see the cache miss signal, which indicates that: in speculative execution, floating pointing instructions are not skipped.




