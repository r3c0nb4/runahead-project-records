# Runahead gadgets
In the Denver processor, the primary cause of following microarchitectural behaviors is runahead rather than speculation, and we have demonstrated this through the fact of floating-point instructions not being executed.

## 1. Bounds check bypass (spectre v1)
When accessing `array_length` results in a cache miss, the processor enters runahead mode. However, since the value of `array_length` is temporarily unavailable, the branch instruction `if (index < array_length)` relies on the branch predictor’s history to make a prediction. If a larger index value is provided and the branch prediction is incorrect, this may lead to accessing memory regions beyond the bounds of the array.
```
/*length_array is flushed*/
if(index < length_array){
    tmp = array[index];
    tmp = reloadbuffer[tmp << 12];
}
```

## 2. Data dependency(similar to Spectre v4)
During runahead execution, data dependency hazards may also occur. Consider the following code snippet: suppose all values in a `buffer` are initially set to `1`. If we modify `buffer[index]` to `2` using an `index` that causes a cache miss, this triggers runahead mode. However, because the `index` is not cached, the store instruction (`buffer[index] = 2`) cannot resolve its target address and thus cannot execute. Additionally, since the target address remains undetermined, the processor cannot mark it as INV (invalid). Consequently, subsequent accesses to `buffer[index]` will still return the old value (1) instead of the updated value (2).
```
buffer[] = {1};
/*index is flushed*/
buffer[index] = 2;
tmp = buffer[index];
tmp = reloadbuffer[tmp << 12]; // we should see 1 hitted in reloadbuffer
```

## 3. Variant
In runahead mode, to improve throughput, floating-point instructions may be squashed and even the entire FPU (Floating-Point Unit) might be disabled. This is because address calculations (e.g., for memory operations) typically do not rely on floating-point instructions, allowing the processor to prioritize critical integer-based operations and reduce speculative overhead during runahead execution.

### 3.1 Bounds check bypass variant
To construct a bounds check bypass gadget, we first trigger runahead mode by loading a cache-missed value, which opens the runahead execution window. Next, we execute the `fcvtzs` instruction to convert a floating-point value (should always be `0.0`) into an integer and write it to `register1`. However, since floating-point operations are discarded in runahead, this conversion is invalidated, leaving `register1` marked as INV (invalid). When adding `register0` (valid) and `register1` (INV), the result stored back into `register0` propagates the INV state, poisoning its validity. During subsequent branch evaluation (e.g., `if (index < register0)`), the processor, unable to resolve the invalid `register0`, defaults to branch predictor history to guess the outcome. If the prediction is incorrect, runahead execution proceeds with the erroneous path, enabling unauthorized access to memory beyond the array’s bounds.

```
register0 = array_length;
load cachemiss; //trigger runahead
fcvtzs floating_point_register -> register1;
register0 = register0 + register1;
if(index < register0){
    tmp = array[index];
    tmp = reloadbuffer[tmp << 12];
}
```

### 3.2 Data dependency variant
This gadget is similiar to last one, apply runahead execution in the Denver processor to bypass bounds checks by leveraging cache-miss-triggered speculation. When a cache-miss load activates runahead, the `fcvtzs` instruction converts a floating-point value to an integer, but the result in `register1` is marked INV (invalid) due to FPU suppression in runahead. Propagating this INV state to `register0` via arithmetic operations invalidates the address calculation for `buffer[register0] = 2`, and since the target address is not resolved, target memory is not able to be marked as INV. At the end, we shall get the old value (1) if we load `buffer[index]`.
```
register0 = index;
buffer[] = {1};
load cachemiss //trigger runahead
fcvtzs floating_point_regster -> register1;
register0 = register0 + register1;
buffer[register0] = 2;
tmp = buffer[index];
tmp = reloadbuffer[tmp << 12]; // we should see 1 hitted in reloadbuffer
```