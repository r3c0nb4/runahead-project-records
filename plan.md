# Further plan
Here are some high-level ideas about the experiments to do in the future. I have not 
designed all of the experiments in detail.
## 1. Trigger event for runahead.
Does L2/3 cache miss will trigger runahead?

## 2. Prefetch mode: for linear memory, will runahead fetch more data?
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


## 3. When will runahead stop?
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

## 4. What about other instructions which are not required by memory fetching in runahead?
### 4.1 Arithmetic instructions
We already know: `x = x + 1` will be executed, because `pick = buffer[x]` depends on the value of `x`. However, we do not know whether `y = y + 1` will be executed or not, since `y` is not required by `pick` operation, it will not change the state of any microarchitecture components. How can we figure it out?
```
x = x + 1;
y = y + 1;
pick = buffer[x]
```
### 4.2 Special control instructions
There are many instructions which are able to control the some component of the processor, for 
example `flush` is used to flush the data from cache to memory, and `isb` instruction is 
used as a memory barrier (The next instruction will not be executed until all the previous `store`,
memory store instructions are retired).

What are the behaviors of those instructions in runahead?
They will also be executed: we should see the same behaviors without runahead. 
Example: if `flush` is executed in runahead, we should see the data is flushed to main memory.


## 5. Branch predictor
In previous experiments, we know that runahead is able to change the state of cache, to be more specific,
runahead prefetches the data which tends to cause cache miss to cache, in this case, 
the cache's state is changed.

Cache is stateful, since we are able to change the state of cache, at the same time,
we can monitor the cache's state is changing (Simply by checking if a specific data is in/not in the cache).

We may assume another component of processor is also stateful: branch predictor.
Branch predictor records the branch history and decide if the branch is taken or not. Definitely, branch 
predictor is stateful.

If we put a branch in runahead, will taken/no-taken history be recorded in branch predictor?
A simple example: in runahead mode, we call `func()`, in this case, will branch in `func` change 
its state: log new taken/no-taken information it gets in runahead mode, or it will be discarded.
```
func(){
    if(...)
        ...
    else
        ...
}

load data // data is cache missed to trigger runahead
func()    // we call func function
```
