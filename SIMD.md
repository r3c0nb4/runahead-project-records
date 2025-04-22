# Experiment check whether vector recources are available during runahead
## 1. Purpose and assumption
Due to the observation that floating-point operations are discarded during runahead execution, we hypothesize that ARM SIMD operations might also be discarded. To verify this, we conduct the following experiment to test whether SIMD operations are similarly dropped.

## 2. Assumption
Assuming that regular arithmetic instructions execute normally during runahead (which they do), if an `add` arithmetic operation depends on a SIMD operation, the add instruction will be marked as `INV` and the corresponding register will also become `INV`.

## 3. Experiment design
In a Spectre v1 gadget triggered by runahead, we introduce a dependency between the index used in the reload operation and a SIMD instruction. If we are unable to observe a leakage signal in the flushed reload buffer, it indicates that the SIMD instruction is marked as INV. At the same time, we design a control group that does not use a cache miss to open the Spectre v1 window, but still introduces the same SIMD dependency as before.

```
/*group 1*/
v0 = 0; //make NEON register v0 = 0;
if(index < array_length){ //cache-missed array_length
    umov v0 -> x1;
    index += x1;
    tmp = reloadbuffer[array[index] << 12];
}
```

```
/*control group*/
v0 = 0; //make NEON register v0 = 0;
if(index < ***array_length_ptr){ //cache-hit array_length_ptr
    umov v0 -> x1;
    index += x1;
    tmp = reloadbuffer[array[index] << 12];
}
```
### 4. Hypothesis
1. If we see signal in both of two groups: SIMD operations are valid in runahead and speculation
2. If we only see signal in control group: SIMD operations are invalid in runahead, but valid in speculation.
3. If we do not see signal in both of cases: SIMD operations are invalid in both speculation and runahead.

### 5. Result
For gadget with cache miss(runahead), we are not able to see leaked signal.

In the control group, the signal is stable. 

SIMD operations are unavailable in runahead.






