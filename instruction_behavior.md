# Test instructions in Runahead

## Main purpose and experiment design
We observe the behavior of ARMv8 instructions during runahead execution. By leveraging a Spectre v1 gadget to open a speculative window, we monitor the execution phenomena of individual ARM instructions within this window: whether they execute normally, get discarded, or exhibit anomalous behaviors.

```
if(index < array_size){
    /*
    * ex. arm_ops(index, add)
    */
    arm_ops(index, ops);
    tmp = reloadbuffer[buffer[index] << 12];
}
```

## 