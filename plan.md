# Further plan

## Trigger event for runahead.
Does L2/3 cache miss will trigger runahead?


## Prefetcher
### Prefetch mode: for linear memory, will runahead fetch more data?
"Run-ahead uses the idle time that a CPU
spends waiting on a long latency operation to
discover cache and DTLB misses further
down the instruction stream and generates
prefetch requests for these misses." 

Because prefetcher tends to fetch more data than required when we are fetching data from array(linear memory), maybe in runahead mode, the prefetcher will also prefetch more data then required.

Consider about a scenario: If the processor finds `array[0-2]` is not in the cache in runahead mode, will it merely prefetches `array[0-2]`? Is that possible it will prefetch part of `array[3 - N]`.
```
// in runahead mode, assume array[0-2] is not in the cache.
pick = array[0];
pick = array[1];
pick = array[2];
```


## When will runahead stop?
1. Runahead stops when the data which triggers runahead is available in the cache.

For the following example, in runhead mode, assume data1, data2, data3 generate three prefetch requests.
```
pick = data // triggers runahead
data1
data2       //Assume data is available before prefetch request for data3 is completed,
data3       // then data3 will not be prefetched.
....
```
2. Runahead use somehow fixed cycles to prefetch data, and even the very first data (it is the very first cache miss and triggers runahead) is available in the cache, it does not stop immediately but continue to complete the prefetching for all of the prefetch requests.

For the following example, in runhead mode, assume data1, data2, data3 generate three prefetch requests.
```
pick = data // triggers runahead
data1
data2       //Assume data is available before prefetch request for data3 is completed,
data3       //runahead mode stops when data3 prefetching is completed.
```

## What about other instructions which are not required by memory fetching in runahead?
We already know: `x = x + 1` will be executed, because `pick = buffer[x]` depends on the value of `x`. However, we do not know whether `y = y + 1` will be executed or not, since `y` is not required by `pick` operation, it will not change the state of any microarchitecture components. How can we figure it out?
```
x = x + 1;
y = y + 1;
pick = buffer[x]
```