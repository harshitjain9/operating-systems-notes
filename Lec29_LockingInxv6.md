**Why locking in xv6?**

- No threads in xv6, so no two user programs can access same userspace memory image
    * No need for userspace locks like pthreads mutex
- However, scope for concurrency in xv6 kernel
    * Two processes in kernel mode in different CPU cores can access same kernel data strcutures
    * When a process is running in kernel mode, another trap occurs, and the trap handler can access data that was being accessed by previous kernel code
- Solution: spinlocks used to protect critical sections
    * Limit concurrent access to kernel data structures that can result in race conditions
- xv6 also has a sleeping lock (built on spinlock, not discussed)

**Spinlocks in xv6**
- Acquiring lock: uses xchg x86 atomic instruction (test and set)
    * Atomically set lock variable to 1 and returns previous value
    * If previous value is 0, it means free lock has been acquired, success!
    * If previous value is 1, it means lock is held by someone, continue to spin in a busy while loop till success
- Releasing lock: set lock variable to 0
- Must disable interrupts on CPU core before spinning for lock
    * Interrupts disabled only on this CPU core to prevent another trap handler running and requesting same lock, leading to deadlock
    * OK for process on another core to spin for same lock, as the process on this core will release it
    * Disable interrupts before starting to spin (otherwise, vulnerable window after lock acquired and before interrupts disabled)

**Disabling interrupts**
- Must disable interrupts on CPU core before beginning to spin for spinlock
- Interrupts stay disabled until lock is released
- What if multiple spinlocks are acquired?
    * Interrupts must stay disabled until all locks are released
- Disabling/enabling interrupts:
    * pushcli disables interrupts on first lock acquire, increments count for future locks
    * popcli decrements count, <u>reenables interrupts only when all locks released and count is zero</u>

**ptable.lock**
```c
struct {
    struct spinlock lock;
    struct proc proc[NPROC];
} ptable;
```
- <u>The process table protected by a lock, any access to ptable must be done with ptable.lock held</u>
- Normally, a process in kernel mode acquires ptable.lock, changes ptable, releases lock
    * Example: when allocproc allocates new struct proc
- But during context switch from process P1 to P2, ptable structure is being changed all throughout context switch, so when to release lock?
    * P1 acquires lock, switches to scheduler, switches to P2, P2 releases lock
    * Normally, the same process acquires and releases the lock, this is a special case
- Every function that calls sched() to give up CPU will do so with ptable.lock held. Which functions invoke sched()?
    * Yield, when a process gives up CPU due to timer interrupt
    * Sleep, when process wishes to block
    * Exit, when process terminates
- Every function that swtch switches to will release ptable.lock. What functions does swtch return to?
    * Yield, when switching in a process that is resuming after yielding is done
    * Sleep, when switching in a process that is waking up after sleep
    * Forkret for newly created process
- <u>Purpose of forkret: release ptable.lock after context switch, before returning from trap to userspace</u>
- In summary, anytime a prodcess is going to context switch will hold the ptable.lock and the process which runs immediately after the context switch will release the ptable.lock
  
 **ptable.lock (continued)**
 - Scheduler goes into loop to search for next process to run with lock held
 - Switch to P1, P1 switches back to scheduler with lock held, scheduler switches to P2, P2 releases lock
 - Periodically, end of looping over all processes, releases lock temporarily
    * What if no runnable process due to interrupts being disabled? Release lock, enable interrupts, allow processes to become runnable

**Summary**
- Spinlocks in xv6 based on xchg (equivalent to test-and-set) atomic instruction
- Processes in kernel mode hold spinlock when accessing shared data structures, disabling interrupts on that core while lock is held
- Special ptable.lock held across context switch
