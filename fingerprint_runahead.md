# Fingerprint runahead
## 1.1 Denver 2 cpu
The Denver 2 CPU is a superscalar processor. While documentation for Denver 1 explicitly states it operates in a strictly in-order manner, we currently lack definitive confirmation about Denver 2's execution behavior. Although preliminary experiments (e.g., instruction latency analysis) provide circumstantial evidence, such methods cannot conclusively determine whether Denver 2 retains in-order execution or adopts out-of-order capabilities. Further microarchitectural profiling or direct vendor clarification would be required for authoritative verification.

## Fingerprint Runahead execution
### 1. What is unique in runahead?
Based on later research on runahead execution and IBM Power9's runahead implementation, runahead mechanisms are likely to discard floating-point instructions because they are rarely used for address calculation (e.g., pointer arithmetic). However, in general speculative execution (e.g., branch prediction or prefetching), there is no inherent reason to discard floating-point operations, as they may still contribute to program logic and data flow. The distinction arises because runahead focuses specifically on prefetching memory addresses, while broader speculation aims to optimize overall execution efficiency.

### 1.1 Experiment assumption
We hypothesize that floating-point operations are discarded during runahead execution in the Denver 2 processor.

### 1.2 Experiment design
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

### 1.3 Hypothesis
If the control group (using `mov` instructions) shows cache hits in the reloadbuffer, while the experimental group (with the floating-point `fcvtzs` instruction) does not, this demonstrates two conclusions:

1. Runahead Execution Was Very Possibly Triggered: The observed behavior aligns with runahead mode (speculative prefetching during long-latency stalls).

2. Floating-Point Instructions Are Discarded in Runahead: The `fcvtzs` instruction was ignored during runahead.

## Result
In the experimental group containing floating-point instructions, we observed no cache hits. However, in the control group (without floating-point instructions), stable cache hits were detected.