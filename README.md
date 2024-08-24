# mini-lsm-starter

Starter code for Mini-LSM.

## Day 1
- **Why doesn't the memtable provide a delete API?**
The deletion of keys in the LSM is done via setting the given key with an empty value. Thus, there's no need for a delete API for the memtable.

- **Is it possible to use other data structures as the memtable in LSM? What are the pros/cons of using the skiplist?**
One of the main requisite of the memtable is to store the data in a sorted manner, so that upon flushing into disk, there's no need to sort all records (which would slow down the merging step into disk).
Another structure would be to use a `BTreeMap` or any other sorted maps (e.g. Red-Black trees).

Skip lists vs B-Tree maps
- Skip lists don't need re-structuring after insert and deletion.
- Same search complexity (*O(log n)*).
- Skip lists supports easily concurrent read and write operations. Concurrent operations in trees are more tricky (locking needed).

- **Why do we need a combination of state and state_lock? Can we only use state.read() and state.write()?**
In case two threads detect that the current memtable has exceeded its capacity, they both would decide to freeze that memtable. In that case, we might create an empty memtable which is immediately frozen. To prevent such situation, we would add a `state_lock` to detect if another thread has already detected a memtable being at maximum capacity.

- **Why does the order to store and to probe the memtables matter? If a key appears in multiple memtables, which version should you return to the user?**
The order defines the latest value of a certain key, in case a key is in a previous frozen `memtable` entry. Thus, we need to get the value of the key that is in the latest memtable (in the state order is the one with smaller index - latest to earliest sorting).

- **Is the memory layout of the memtable efficient / does it have good data locality? (Think of how Byte is implemented and stored in the skiplist...) What are the possible optimizations to make the memtable more efficient?**
An aspect that could be improved would be the fact that certain keys are stored multiple times across different memtables, which is not the best efficient storage-wise. However, this allows the LSM to don't have to search in all the frozen `memtables` whenever updating a value.

- **So we are using parking_lot locks in this tutorial. Is its read-write lock a fair lock? What might happen to the readers trying to acquire the lock if there is one writer waiting for existing readers to stop?**
The read-write lock from `parking_lot` ([See docs](https://docs.rs/parking_lot/latest/parking_lot/type.RwLock.html)) uses eventual fairness to ensure that the lock will be fair on average without sacrificing throughput. This is done by forcing a fair unlock on average every 0.5ms, which will force the lock to go to the next thread waiting for the rwlock.
Those readers would need to wait for the writer to free up the lock that it's waiting to get.

- **After freezing the memtable, is it possible that some threads still hold the old LSM state and wrote into these immutable memtables? How does your solution prevent it from happening?**
No, it's not possible, since the `memtable` state is under an `RwLock`, meaning that only a single thread can write into the memtables at a time. Thus, only a single thread is able to trigger a freeze and no writing open locks should be available in other threads.

- **There are several places that you might first acquire a read lock on state, then drop it and acquire a write lock (these two operations might be in different functions but they happened sequentially due to one function calls the other). How does it differ from directly upgrading the read lock to a write lock? Is it necessary to upgrade instead of acquiring and dropping and what is the cost of doing the upgrade?**
It's not possible to upgrade a read lock into a write lock, as you need to free up always the read lock so that a write lock can be acquired (single writer, multiple readers).
