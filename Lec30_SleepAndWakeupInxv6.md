**Sleep and wakeup**
- Equivalent to condition variables in user space, except that this is in kernel mode
- A process P1 in kernel mode gives up CPU to block on an event
    * Example: process reads a block from disk, must block until disk read completes
- P1 invokes "sleep" function, which calls sched() and gives up CPU
- Another process P2 in kernel mode calls "wakeup" when event occurs, marks P1 as runnable, scheduler loop switches to P1 in future
    * Example: disk interrupt occurred when P2 is running, so P2 handles the interrupt, and marks P1 as runnable
- How does P2 know which process to wake up? When P1 sleeps, it sets a channel (void *chan) in its struct proc, and P2 calls wakeup on same chanel (channel = any known value known to both P1 and P2)
    * Example: channel value for disk read can be address of disk block
- Spinlock protects atomicity of sleep: P1 calls sleep with some spinlock L held, P2 calls wakeup with same spinlock L held
    * Eliminating missed wakeup problem that arises due to P2 issuing wakeup between P1 deciding to sleep and actually sleeping, which would mean P1 would end up sleeping indefinitely
    * Lock L released after sleeping, available for wakeup
    * Similar concept to condition variables studied before

**Sleep function**
- Sleep calls sched() to give up CPU
    * Needs to hold ptable.lock
- Acquire ptable.lock and change p->state to SLEEPING (remember ptable.lock will be held throughout the context switch even), release the lock given to sleep (make it available for wakeup)
    * Unless lock given is ptable.lock itself, in which case no need to acquire again
    * One of two locks held at all times, meaning wakeup can never occur in between
- Calls sched(), switched out of CPU, resumes again when woken up and ready to run
- Reacquires the lock given to sleep and retuns back
    * Code that invoked sleep with lock held returns with lock held again

**Wakeup function**
- Called by another process with lock held (same lock as when sleep was called)
- Since it changes ptable, ptable.lock will also be held
    * If sleep lock is ptable.lock itself, then no need to acquire ptable.lock again
- Sleep holds one of sleep's lock or ptable.lock at all times, so a wakeup cannot run in between sleep
- Wakes up all processes sleeping on the matching channels in ptable (more like signal broadcast of condition variables)
    * Good idea to check condition is still true upon waking up (use while loop while calling sleep)

**Example: pipes**
- Pipe is an example of the usage of sleep-wakeup functionality
- Two processes connected by a pipe (producer consumer)
    * <u>Common shared buffer, protected by a spinlock</u>
- One process writes into pipe, another reads from pipe
- Reader sleeps if pipe is empty, writer wakes it up after putting data
- Wrtier sleeps when pipe is full, reader wakes it up when data is consumed
- Address of pipe structure variables are channels (same channel known to both)

**Example: wait and exit**
- Another example of sleep-wakeup functionality in xv6 kernel
- If wait called in parent while children are still running, parent calls sleep and gives up CPU
    * Here, channel (used to determine which process to wake up) is parent struct proc pointer, lock is ptable.lock because parent and child both access ptable (sleep avoids double locking, doesn't acquire ptable.lock if it is already here before calling sleep)
```c
// Wait for children to exit 
sleep(curproc, &ptable.lock);
```
- In exit, child acquires ptable.lock and wakes up sleeping parent. 
```c
// Parent might be sleeping in wait()
wakeup1(curproc->parent);
```
- Why is terminated process memory cleaned up by parent? When a process calls exit, CPU is using its memory (kernel stack is in use, cr3 is pointing to page table) so <u>all this memory cannot be cleared until terminated process has been taken off by the CPU</u>
    * Parent code in wait is a good place to clean up child memory after child has stopped running

**Summary**
- Sleep and wakeup functionality in kernel for processes to wait for or signal each other
    * Similar to condition variables for synchronization of user threads
- Examples of sleep/wakeup
    * Pipe reader and pipe writer processes
    * Parent sleeps for child to die, zombie child wakes up parent
- Code calling sleep and wakeup need to hols same lock in order to avoid missed wakeup problem 
