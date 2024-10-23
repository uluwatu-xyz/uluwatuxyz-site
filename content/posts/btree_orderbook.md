---
title: "Is the BTree an efficient structure to model an order book?"
date: 2023-12-20T13:29:35Z
draft: false
toc: false
images:
tags:
  - rust
  - orderbook
  - btree
---

# Introduction

The order book is an interesting object to model, likely to be dense with orders near the top of the book and sparser as we walk outwards towards the less likely to get filled "stink bids". While reading *Database Internals: A Deep-Dive into How Distributed Data Systems Work* [^1], its low level discussion of BTree internals peaked my curiosity into the efficiency of using a BTree to model an orderbook. My main observations are:

1. It was presented as a disadvantage that file based databases have to marshall their data structures explicitly on disk while in-memory databases only have to `malloc()` memory chunks from an allocator. However, if the memory returned from the allocator is fragmented then the cache efficiency of traversing the BTree structure is going to be impacted negatively.

2. The search path from the root node to the leafs involves a lot of searching sorted lists which equates to a lot of work for the CPUs branch predictor. Incorrectly guessed branches equates to wasted computation.

3. Mutations / admin tasks where nodes are expanded and collapsed dynamically suggests unpredictable latency spikes. This cost will be "amortized" in the CPU complexity of the structure - It would be unfortunate for us if this appeared during a period of high market intensity.

One parameter we can tune to attempt to optimise the points above is `B`, controlling the number of entries in each BTree node. If we increase this we can leverage bigger nodes for memory cache efficiency due to data locality and incur some additional costs when searching / mutating. Unfortunately the Rust std implementation does not let you specify this parameter, it is hardcoded at 16.

# An alternative - `StableVec`

Since we have a predictable range of keys (price ticks), can we throw memory at the problem and pre-allocate a cache friendly contiguous array of the size of every possible price?

The [StableVec](https://docs.rs/stable-vec/latest/stable_vec/) library allows us to allocate a contiguous array with "stable" keys, stable meaning the memory they point to will never change (Unlike a Vec, where data indexes change as items are removed). This means when an entry is removed it is simply flagged as deleted and no cost of shifting entries backwards as you get with a standard Vec is incurred. This is conceptually the same as allocating a large Vec of Options and setting values to `None` instead of removing, for example: `Vec<Option<T>>::with_capacity(N)`.

The orderbook structure is useless if we can't get timely snapshots of the data to feed to our trading logic. After some research it becomes clear that the generic StableVec `Iterator` is to inefficient to retrieve our existing data. Due to the sparsity of the data, it implements a naive scan from 0 to the capacity of the structure: 

{{< highlight Rust >}}
unsafe fn first_filled_slot_from(&self, idx: usize) -> Option<usize> {
    debug_assert!(idx <= self.cap());

    (idx..self.len()).find(|&idx| self.has_element_at(idx))
}
{{< /highlight >}}

In this proof of concept we have a significant capacity size of 1_000_000_000. To work around this we implement some tracking pointers which continually identify the best bid and ask indexes. When we generate a snapshot we skip the library Iterator and do a walk outward from these indexes manually. 

# Profiling 

## Profiling the orderbook message processing logic 

To benchmark we process a days worth of Binance L2 price updates for BTCUSDT into each orderbook implementation (141327104 bid and ask updates). We capture the elapsed time and look at the detailed ouput of the `perf` command.

| Implementation | Elapsed      | 
|----------------|--------------|
| BTree          | 12.9s        |
| StableVec      | 5.9s         |

We see a significant end to end speed up and as expected cache-misses and branches are notably reduced.

### BTreeMap perf output
```
 Performance counter stats for 'target/release/btree-ob-eval b':

     3,681,556,457      cache-references
       202,381,615      cache-misses              #    5.497 % of all cache refs
   224,968,394,162      cycles
   595,727,002,465      instructions              #    2.65  insn per cycle
   140,040,270,447      branches
         2,761,188      faults
                38      migrations
```

### StableVec perf output
```
 Performance counter stats for 'target/release/btree-ob-eval s':

     2,781,531,040      cache-references
       132,323,303      cache-misses              #    4.757 % of all cache refs
   196,283,494,716      cycles
   544,535,582,371      instructions              #    2.77  insn per cycle
   129,142,369,121      branches
         2,785,045      faults
                33      migrations
```


## Profiling the orderbook snapshotting logic 

When benchmarking the two implementations we take the mean of 5 `get_snapshot()` elapsed time measurements (with an prior call intended to warm the cache). Snapshot sizes are selected as multiples of the BTreeMaps `B` parameter (16), as it is likely that the memory fragmentation of the BTreeMap implementation scales somewhat relatively in multiples of this. The results are below:
  
  

| Implementation | Snapshot(32) | snapshot(64) | snapshot(128) | snapshot(256) |
|----------------|--------------|--------------|---------------|---------------|
| BTree          | 698.866us    | 711.125us    | 669.833us     | 745.166us     |
| StableVec      | 524ns        | 825ns        | 1.641us       | 3.075us       |


# Conclusion

The StableVec implementation has proven itself a superior alternative, notably in the bulk load benchmark but more significantly in the snapshot benchmark where it is consistently order of magnatudes faster than the BTree alternative. I believe this is due to the cache efficiency of the pre-allocated structure.

In terms of tuning the BTreeMap there are some interesting avenues we've not explored. One is a study of tuning the `B` parameter on an implementation which allows it. Another is to provide the BTreeMap with a custom allocator (such as a bump allocator) which allows it to allocate nodes from the same block of contigous memory even over a long lifetime, exploiting memory locality.

Profiling code is provided [here](https://github.com/uluwatu-xyz/rust-scratch-public/tree/main/btree-ob-eval)


[^1]: Database Internals: A Deep-Dive into How Distributed Data Systems Work - https://www.databass.dev/