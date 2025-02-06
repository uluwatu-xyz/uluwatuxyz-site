+++
title = 'Debugging Python memory leaks with Heapy (Guppy)'
date = 2025-02-06T12:13:03Z
draft = false
+++

# Introduction 

Guppy [^1] is a python library containing among other things the heap debugging tool Heapy. This came in useful recently when attempting to debug a memory leak in a long running Python process. Large enterprises are rumoured to restart server processes every weekend to work around such issues, but we prefer a more scientific solution.

Unfortunately in the Python 3 port of Guppy the remote shell functionality for interactive debugging was disabled, although this would "stop the world" for other processing - not great for debugging production systems. To enable production profiling (as a last resort) we created a small utility class to write specific Guppy output to file on a schedule.

The Guppy output really requires interactive debugging, dumping pre-specified api interactions was slow and painful. we added a `pdb.set_trace()` option as a homage to the remote shell functionality, ultimately the ncurses nature of the application in question rendered this almost unusable also. We also investigated serialising the heapy snapshot with `pickle` but it was not trivial.

Alas, with some perseverance and some hardcoded Gappy api calls the memory leak was eventually found.

# The Code

The utlity class is designed to be instantiated in a long running process and called frequently from a control loop (via `tick()`), if the required time has elapsed the utility will write a text representation of the snapshot to disk. It requires the Python library `gappy3`. The specific Gappy calls it makes are specific, hardcoded and ugly - it's an exercise for the reader to tune these as required.

{{< highlight Python >}}
import datetime
from guppy import hpy
import pdb, gc

hp = hpy()

class HeapySnapshot:
    def __init__(self, snapshot_interval: datetime.timedelta, 
            drop_shell: bool = False, refresh_snapshot: bool = False):
        self.snapshot_interval = snapshot_interval
        self.heap = None
        self.last_snapshot = None
        self.drop_shell = drop_shell
        self.refresh_snapshot = refresh_snapshot

    def _generate_output_fn(self):
        return "/tmp/heapy.log"
    
    def _take_snapshot(self):
        self.heap = hp.heap()
        self.last_snapshot = datetime.datetime.utcnow()

    def tick(self):
        if self.heap is None:
            return self._take_snapshot()
        
        now = datetime.datetime.utcnow()
        if now - self.last_snapshot > self.snapshot_interval:

            gc.collect()
            delta = hp.heap() - self.heap

            if self.drop_shell:
                pdb.set_trace()

            with open(self._generate_output_fn(), "a") as fd:
                fd.write(str(delta.byrcs) + "\n")
                fd.write(str(delta[0].byid[0].shpaths) + "\n")
                fd.write("==\n\n")

            if self.refresh_snapshot:
                self._take_snapshot()
            else:
                self.last_snapshot = datetime.datetime.utcnow()

{{< /highlight >}}


# Consecutive vs Cumulative snapshot deltas

The initial investigation focused on analysing the delta between rolling snapshots (*refresh_snapshots*=True), ultimately this was
too noisey as it included expected allocations such as logger output and other objects which we expect to be garbage collected 
shortly after. In this small sample size it was hard to pin-point the underlying issue.

Switching to taking cumulative snapshot deltas (ie. now vs. initial snapshot) proved much more valuable, the 
expected short term allocations remained small over time and were utlimately dwarfed by the genuinely leaked object(s) as the sample size increased.

In this example, the top four items are all traced back to the single object which is leaked. In the output below more than 50% of the allocated memory can be traced to an `asyncio.Event` inside the object ultimately being leaked (confirmed via the omitted `shpaths` output). Specifically, the class variable `_waiters` (a queue) inside the `Event` is flagged as being the largest in size.

```
Partition of a set of 274737 objects. Total size = 46022859 bytes.
 Index  Count   %     Size   % Cumulative  % Referrers by Kind (class / dict of class)
     0  39457  14 24621168  53  24621168  53 dict of asyncio.locks.Event
     1 114693  42  6752028  15  31373196  68 dict of <LEAKED OBJECT>
     2  39457  14  4103528   9  35476724  77 asyncio.locks.Event
     3  13152   5  3051264   7  38527988  84 <LEAKED OBJECT>
     4  26034   9  1512763   3  40040751  87 dict (no owner)
    ...
```

# Why is the size of asyncio.Event objects so large?

It is curious that the `asyncio.Event` object dominated the allocated memory in terms of raw object size, especially when the containing (leaked) object contains strings and other objects - items I assumed would be more memory intensive than simple synchronisation logic.

We dive into the details, of `asyncio.Event` - a truncated representation is shown below. The high level process flow is that:
* Consumers invoke `wait()` which creates a future, adds it to `self._waiters` queue and blocks waiting for the future to finish.
* When the event is "set", we loop through each of the futures in `self._waiters` and mark them as complete, allowing the consumer to return and continue execution.

{{< highlight Python >}}
class Event:
    def __init__(self, *, loop=mixins._marker):
        super().__init__(loop=loop)
        self._waiters = collections.deque()
        self._value = False

    def set(self):
        """Set the internal flag to true. All coroutines waiting for it to
        become true are awakened. Coroutine that call wait() once the flag is
        true will not block at all.
        """
        if not self._value:
            self._value = True

            for fut in self._waiters:
                if not fut.done():
                    fut.set_result(True)

    async def wait(self):
        """Block until the internal flag is true.

        If the internal flag is true on entry, return True
        immediately.  Otherwise, block until another coroutine calls
        set() to set the flag to true, then return True.
        """
        if self._value:
            return True

        fut = self._get_loop().create_future()
        self._waiters.append(fut)
        try:
            await fut
            return True
        finally:
            self._waiters.remove(fut)
{{< /highlight >}}

At a high level, this is relatively simple. In our specific scenario we must have a huge amount of consumers waiting on a single `Event` to finish? This doesn't immediately register as a valid use case in the application under investigation.

Next we examine the consumer of this specific object, which resides in a framework the application is relying on and understand the underlying issue:

{{< highlight Python >}}
async def get_id(self):
    if self.id is None:
        async with timeout(TIMEOUT):
            await self.id_update_event.wait()
    return self.id
{{< /highlight >}}

This `timeout()` wrapper cancels the `wait()` task after 10 seconds if it has not completed already, but nothing is removing the corresponding entry from `_waiters`. In an error state, where the `id` field in question is never set - we will continuously append orphaned futures to `_waiters`. If this object is important and polled continuously, we well generate a large enough number of orphaned waiters for it to stand out in terms of memory consumption.  

This pattern is a pretty sub-optimal way to query a "possibly yet to be updated" value asynchronously, it fails to cater for the error case of the value never appearing and also blocks exection for up to 10 seconds waiting on every invocation.

Ultimately the `asyncio.Event` memory usage issue is a secondary one. Had the parent object not have leaked, it would have been removed by a "Time To Live" type logic, removing it's child Event from the scope of callers.


[^1]: Guppy 3, https://zhuyifei1999.github.io/guppy3/