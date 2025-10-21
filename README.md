# 4190.307 Operating Systems (Fall 2025)
# Project #3: The Road to Balance
### Due: 11:59 PM, November ~~2 (Sunday)~~ 6 (Thursday)

## Introduction

Modern OS schedulers support _CPU affinity_—binding a process (or its threads) to a subset of CPU cores—to preserve cache locality and reduce interference from other processes, while also load balancing to maximize throughput by keeping all cores busy. 
In this project, you will extend `xv6`'s scheduler with CPU affinity and design your own multicore load balancing strategy that achieves measurable improvements in completion time under imbalanced workloads. The goal is to understand `xv6`'s scheduling subsystem and modernize it toward an affinity-aware, scalable multicore scheduler. 

## Background

### CPU Affinity

Modern multicore operating systems give programmers a way to control where an individual thread may run. On Linux, this control is expressed as _CPU affinity_. Each thread carries an "allowed set" of CPUs on which it may execute. The scheduler is free to migrate the thread among CPUs within that set, but it must never run the thread elsewhere. Affinity exists for both performance and manageability. From a performance perspective, keeping a thread near the cache hierarchy that already contains its working set reduces cold misses and preserves locality; on NUMA machines, constraining execution to a node also reduces remote memory traffic. From a manageability perspective, affinity makes it possible to reserve cores for latency-sensitive tasks, isolate background work, and reduce scheduler noise during performance measurement.

Linux exposes affinity through the `sched_setaffinity()`/`sched_getaffinity()` system calls. Affinity is a per-thread attribute, so different threads within the same process may legally have different masks. When a thread's mask is narrowed and its current CPU is no longer permitted, the kernel ensures the thread is migrated to a valid CPU before it next runs. Conversely, when the mask is broadened, the kernel's load balancer may move the thread to take advantage of idle capacity — but only among CPUs that remain in the allowed set. "Pinning" is simply the special case in which the allowed set contains a single CPU.   

Linux provides a user-space tool called `taskset`, a thin wrapper around `sched_affinity()`. For example, if you are benchmarking a crypto routine and want to eliminate cross-core migrations, you can launch it pinned to core 2 as `$ taskset -c 2 ./bench-crypto`. You can also confine an already-running process `1234` to cores 0-3 with `$ taskset -p -c 0-3 1234`. 

### The `xv6` Scheduler

The `xv6` scheduler implements a simple round-robin policy driven by periodic timer interrupts. Each hart runs its own scheduler loop; it repeatedly searches for a process in the `RUNNABLE` state, marks it `RUNNING`, and context-switches to it. A per-hart hardware timer interrupt fires at a fixed interval; the trap handler records the tick and forces the currently running process to yield so another runnable process can make progress.

In the current `xv6` implementation, the "run queue" is a single global array (cf. `struct proc proc[NPROC]` in `kernel/proc.c`). This design does not scale as core counts grow; every hart's scheduler repeatedly scans the same array to find a runnable process and contends on the same per-process locks. As the number of harts increases, both the scheduling effort and lock contention grow, limiting throughput.

## Problem specification

### Part 1. CPU affinity with the `pin()` system call (10 points)

First, you are required to implement the `pin()` system call. The system call number for `pin()` is already assigned to 30 in  `kernel/syscall.h`. 

__NAME__

`pin` -- set or query CPU affinity of the calling process

__SYNOPSYS__
```C
    int pin(int mask);
```

__DESCRIPTION__

The `pin()` system call constrains where the calling process may run by setting a CPU affinity mask. The mask is a bitmap written as _b<sub>n-1...</sub>b<sub>3</sub>b<sub>2</sub>b<sub>1</sub>b<sub>0</sub>_, where _b<sub>i</sub>_ corresponds to RISC-V hart (core) _i_. The argument is interpreted as follows:

 * `mask > 0` (set mode)

   The process may run only on RISC-V harts whose bits are set in `mask`. For example, a mask of 1 (0b1) allows execution only on hart 0, whereas a mask of 9 (0b1001) allows execution on harts 0 and 3. If the call restricts the mask such that the current hart is no longer permitted, the process should yield the CPU so that it can be scheduled next on one of the allowed harts. The scheduler remains free to move the process _within_ the allowed set.
   At runtime, the actual number of online harts is exposed as `nharts`. When enforcing an affinity mask, bits corresponding to non-existent harts (indices `i` &ge; `nharts`) are ignored. On `fork()`, the child inherits the parent's CPU affinity mask. On `exec()`, the mask is preserved, i.e., executing a new program does not change the mask.
   
 * `mask == 0` (unpin mode)
   
   Clears any prior restriction; the process may run on any online harts.
   
 * `mask == -1` (query mode)
   
   Returns the current CPU affinity mask for the calling process without modifying it.

__RETURN VALUE__

* On success, `pin(mask)` returns 0 in set/unpin mode (`mask >= 0`) and returns the current mask of the calling process in query mode (`mask == -1`).
* On error, `pin()` returns -1.

### Part 2. Per-core run queues with process migration (30 points)

Your next task is to reorganize the scheduler from a single system-wide run queue to per-core run queues. Each RISC-V hart maintains its own run queue with a capacity of `QPROC` (cf. `kernel/param.h`) processes and selects the next task from that local queue using the same round-robin policy as before.

CPU affinity must be honored across all per-core queues: a process may be enqueued and executed only on harts permitted by its affinity mask. On `fork()`, the child process inherits its parent's affinity mask and may be placed on the same queue as its parent, or on any other queue allowed by the affinity mask.

When a process calls `pin()` to change its CPU affinity, such that the current hart is no longer permitted, the process must be migrated from its current run queue to a permitted hart's queue. Implement this by removing the process from the source queue and enqueuing it on a destination queue that satisfies the new mask.

Each run queue has its own lock. Local enqueue/dequeue operations require only the local queue's lock, but moving a process across queues requires locking both the source and destination queues. To avoid deadlock, enforce a global lock-ordering rule whenever holding multiple locks; for example, always acquire locks in ascending hart ID order and release them in reverse.

### Part 3. Load Balancing Policy (50 points)

The final task is to design and implement your own load balancing policy for the scheduler with per-core run queues. Your goal is to minimize the completion time of a given batch of CPU-intensive processes while preserving the correctness and safety properties established in the earlier requirements. In this project, completion time is defined as wall-clock time from the moment the workload becomes runnable until the last process exits; any time a process spends sleeping is naturally included in this measurement, although the tested workloads are primarily CPU-bound.

Load balancing among per-core queues is difficult because it forces the scheduler to trade off immediate queue-length gains against longer-term costs. Moving work to an idle or lightly loaded hart can shorten waiting time and improve overall throughput, but migrations are not free: they disrupt cache/TLB locality and incur synchronization overhead. CPU affinity constraints further reduce feasible placement choices, and local decisions made independently by multiple cores can interact in surprising ways, leading to oscillations or "thrashing" if the policy is too reactive. To quantify these costs in this project, a process experiences a 10ms penalty (one-tenth of a tick) whenever it runs on a different hart than in its previous run (a hart-migration event). Your policy should account for this cost and avoid needless migrations that would lengthen completion time once the penalty is paid.

You are free to choose any load balancing policy so long as it is implementable with per-core run queues in the current `xv6`. Typical patterns include periodic rebalancing (each hart occasionally scans its neighbors and pushes/pulls work), idle-time stealing (an idle hart attempts to steal a runnable process from an overloaded neighbor), or a hybrid. All migrations must respect each process's affinity mask, avoid deadlock, and not starve runnable tasks. Also, respect the per-queue capacity when placing or moving tasks. Whatever you choose, document your policy and mechanisms, along with the expected benefit and cost. 

To receive a full credit, your implementation should pass `usertests` available in `xv6`.

### BONUS: (Up to 20 points)

We will award a 20% bonus to the top 10 submissions and a 10% bonus to the next 10 based on the effectiveness of their load balancing policies. Only submissions that pass all functional tests are eligible. Efficiency is measured by completion time: for a set of different workloads, we will run each five times, discard the two outliers, compute the average of the remaining runs, and rank the results in ascending order of execution time.


### 4. Documentation (10 points)

Along with your code, submit a design document in a single PDF file. Your document should include the following sections.

1. New data structures
   * Provide details about any newly introduced data structures or modifications made to existing ones.
   * Explain why these data structures/modifications were necessary and how they contribute to the implementation.
2. Algorithm design
   * Explain your placement policy, which selects a target core from the process's allowed set when a new process is created, or the process's CPU affinity mask changes.
   * Explain your load balancing policy, its trigger condition, and how it accounts for the 10ms migration penalty.
   * Summarize any additional metrics you collect and briefly argue why your results match the policy's intent
   * Describe all corner cases you considered and the strategies you used to address them.
   * Discuss any optimizations you applied to improve code efficiency, both in terms of time and space.
3. Testing and validation
   * Outline the test cases you created to validate your implementation.
   * Provide a simple experiment that runs a batch of CPU-intensive processes with heterogeneous runtimes and report the total completion time under the per-core scheduler (a) without load balancing, and (b) with load balancing. 
   * Describe how you verified the correct handling of the corner cases mentioned in Section 2.
   * Explain which part of the project consumed most of your time and why.


## Skeleton code

The skeleton code for this project assignment (PA3) is available as a branch named `pa3`. Therefore, you should work on the `pa3` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ git checkout pa3
```
After downloading, you must first set your `STUDENTID` in the `Makefile` again.

We mimic compute-bound work with a simple busy loop. The function below executes a tight `nop` loop compiled with `-Og` and marked `noinline` so the compiler won't optimize it away; calling `busyloop(1)` takes roughly half a tick (~ 0.5x tick) on our setup, and the runtime scales approximately linearly with `n`. This gives us predictable, visible runtimes without doing any I/O.

```C
__attribute__((optimize("Og"), noinline))
void
busyloop(int n)
{
  for (int i = 0; i < 100000000 * n; i++)
    asm volatile("nop");
}
```

To help you observe scheduling behavior, the skeleton code already includes a logging facility for CPU bursts. A CPU burst is defined as the interval from the moment a process starts running on a CPU until it is descheduled because it (a) exhausts its time slice, (b) blocks/sleeps (e.g., for I/O), or (c) exits.

The `pa3` branch includes the required macros `PRINTLOG_START` and `PRINTLOG_END` (in `kernel/proc.h`) and they are already placed in the relevant scheduler paths. These macros are compiled as no-ops unless logging is enabled. To enable logging, add `-DLOG` in `CFLAGS` in the `Makefile`. When logging is enabled, the kernel prints a line at every CPU burst start and end, as follows:

```
xv6 kernel is booting

hart 1 starting
hart 3 starting
hart 2 starting
1306714 1 starts on 0
1311999 1 starts on 2
1314987 1 ends on 2
1318221 1 starts on 3
1324603 1 ends on 3
1327009 1 starts on 2
1332268 1 ends on 2
1335041 1 starts on 2
1339681 1 ends on 2
1341984 1 starts on 2
1353253 1 ends on 2
...
```

The log format is as follows: `[timestamp] [pid] ["starts"|"ends"] on [hart id]`.
* The `timestamp` is obtained using the `r_time()` function, which returns the current cycle count, where 1,000,000 cycles correspond to 1 timer tick or 100 milliseconds (see `clockintr()` in `kernel/trap.c`). 
* The `pid` indicates the process ID that just began or ended a CPU burst.
* The actions `starts` or `ends` signify whether this event marks the beginning or end of a CPU burst.
* The `hart id` represents the RISC-V hart (core) ID on which the event occurred (i.e., the CPU currently executing the process).

The skeleton code includes a user-level program `task` (source in `./user/task.c`) that is used to demonstrate the effect of CPU affinity and scheduling behavior. On startup, the parent first sets its CPU mask with `pin(3)`, meaning it may run on hart 0 or hart 1 (`0b0011`). It then forks eight children and each child immediately calls `pin(9)`, allowing execution on hart 0 or hart 3 (`0b1001`). Any child that initially runs on hart 1 must therefore be migrated to a permitted hart (0 or 3). Each child performs a CPU-bound `busyloop(n)` whose runtime scales with the sequence 1:2:1:4:5:3:3:2 (from the first to the last child), producing visually distinct CPU bursts. The program finishes after the parent waits for all children to exit.

To make cross-hart migrations visible and non-free, the skeleton code imposes a 10ms penalty whenever a process resumes on a different hart than it previously ran on. This is implemented in `kernel/trap.c`: on each return to user mode, the kernel checks the current hart (`cpuid()`) against `p->lasthart`. If the process is neither `init` nor `sh` and the hart changed, we (1) log the overhead window with `PRINTLOG_STARTov` and `PRINTLOG_ENDov`, (2) increment the process's migration counter (`p->migcount`), and (3) call `msdelay(HART_MIG_DELAY)` with `HART_MIG_DELAY=10`, which busy-waits for ~10 ms using the cycle counter. Afterward, `p->lastcpu` is updated to the current hart.

The skeleton also maintains per-process metrics that you may use but must not change:
 * `p->lasthart`: the hart ID on which the process last ran
 * `p->migcount`: the number of core migrations, incremented by one each time the process runs on a hart different from `p->lastcpu`
 * `p->schedmask`: a bitmask of harts on which the process has actually executed. Each time the process runs, the bit corresponding to the current hart is set

```C
// @ kernel/trap.c
#define HART_MIG_DELAY  10

void
msdelay(int ms)
{
  uint64 cur = r_time();
  uint64 target = cur + 10000*ms;

  while (r_time() < target)
    asm volatile("nop");
}

uint64
usertrap(void)
{
  ...
  // the user page table to switch to, for trampoline.S
  uint64 satp = MAKE_SATP(p->pagetable);

#ifdef SNU
  // We give the penalty when a process is migrated to another hart
  int h = cpuid();
  if (p->pid > 2 && p->lasthart != -1 && p->lasthart != h) {
    PRINTLOG_STARTov;
    p->migcount++;
    msdelay(HART_MIG_DELAY);
    PRINTLOG_ENDov;
  }
  p->lasthart = h;
  p->schedmask |= (1 << h);
#endif

  // return to trampoline.S; satp value in a0.
  return satp;
}
``` 

We also provide a Python script `graph.py` in the `./xv6-riscv-snu` directory. You can use this Python script to convert the CPU-burst log into a graph image. This graph will visually represent the order and timing of process executions, allowing you to see how processes are scheduled across harts under different scheduling policies. Note that the `graph.py` script requires the Python matplotlib package, which can be installed using the following command in Ubuntu:

```sh
$ sudo apt install python3-matplotlib
```

To generate a graph, you should run `xv6` using the `make qemu-log` command that saves all the output into the file named `xv6.log`. And then, run the `make png` command to generate the `graph.png` file using the Python script `graph.py` as shown below.

```
qemu-system-riscv64 -machine virt -bios none -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -kernel kernel/kernel-log

xv6 kernel is booting

1215159 1 starts on 0 userinit
hart 3 starting
hart 2 starting
hart 1 starting
1222153 1 starts on 0 sched
1225057 1 ends on 0 sleep
1227917 1 starts on 0 sched
1233749 1 ends on 0 sleep
1236792 1 starts on 0 sched
1242028 1 ends on 0 sleep
1244248 1 starts on 0 sched
...
$ init: starting sh
...
$ task                            <--- Run the task program
29305042 2 starts on 1 sched
29335404 2 ends on 1 sleep
29344951 3 starts on 2 sched
29413177 3 ends on 2 sleep
29417031 3 starts on 2 sched
29422818 3 ends on 2 sleep
29425882 3 starts on 2 sched
29430481 3 ends on 2 sleep
29433467 3 starts on 2 sched
29438558 3 ends on 2 sleep
...
$ QEMU: Terminated                <--- Quit qemu using ^a-x
*** The output of xv6 is logged in the `xv6.log` file.

$ make png                        <--- Generate the graph (this should be done on the host machine, not on xv6)
graph saved in the `graph.png` file
```

If you prefer a one-shot workflow, just run:

```sh
./test-xv6.py pa3log
```

This script boots the logging kernel, runs `task` automatically inside QEMU, saves the full console output to `xv6.log`, and then calls `graph.py` to produce `graph.png` — no manual QEMU interaction needed!


## Examples

The following examples are obtained by running the `task` program under various scheduling policies. In the graphs, the dotted region marks the hart migration penalty window.

### Default `xv6` scheduler (no affinity)
`pin()` is a no-op, so every hart's scheduler can run any runnable process; the timeline shows processes freely appearing on all harts with no placement constraints.

![PNG image](https://github.com/snu-csl/os-pa3/blob/main/xv6-default.png)

### Default `xv6` scheduler + `pin()`
Schedulers now respect the CPU affinity mask; a process runs only on harts allowed by its mask, so harts 1 and 2 stay idle while each process executes on permitted harts.

![PNG image](https://github.com/snu-csl/os-pa3/blob/main/xv6-pin.png)

### Per-core run queues (no load balancing)
Each hart maintains its own run queue, and processes are enqueued according to their affinity masks. If the mask changes from 3 to 9 after `fork()`, any tasks that were placed on core 1 are migrated to a permitted hart (0 or 3), while all others remain on their original harts.

![PNG image](https://github.com/snu-csl/os-pa3/blob/main/xv6-percoreq.png)

### Per-core run queues + load balancing
In addition to honoring CPU affinity masks, idle (or lightly loaded) harts also pull runnable work from busy harts; in the figure, two processes (PID 7 and 8) are rebalanced from hart 3 to hart 0 — reducing idle time while still respecting CPU affinity.

![PNG image](https://github.com/snu-csl/os-pa3/blob/main/xv6-loadbalance.png)

## Restrictions

* You will get zero points for Part 2 and 3 if you attempt to simulate the behavior of per-core run queues using the current `xv6`'s system-wide run queue. You must implement true per-core run queues, where each hart has its own queue and the associated lock. The common-case scheduling path—enqueue, pick next, dequeue—must operate locally on that hart's queue. TAs and AI-based tools will inspect your code to confirm that per-core queues are implemented as intended.
* You may not add artificial delays/sleeps, tamper with logging, or modify the migration-penalty logic. The provided penalty, counters, and logging hooks must remain intact.
* You only need to change the following files in the `./kernel` directory: `proc.h`, `proc.c`, and `sysproc.c`. Any other changes will be ignored during grading.
* For this project assignment, you should use the `qemu` version 8.2.0 or higher. To determine the `qemu` version, use the command: `$ qemu-system-riscv64 --version`

## Tips

* Read Chap. 8 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2025-2/book-riscv-rev5.pdf) to understand the scheduling subsystem of `xv6`.

* Read Chap. 7.1 ~ 7.4 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2025-2/book-riscv-rev5.pdf) to understand locks and lock ordering.
  
* For your reference, the following roughly shows the required code changes; each `+` denotes about 1~10 lines to add, remove, or modify.
   ```
   kernel/proc.h        |  ++
   kernel/proc.c        |  ++++++++++++++++++++
   kernel/sysproc.c     |  ++
   ```
  
## Hand in instructions

* First, make sure you are on the `pa3` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. In addition, you must also upload your design document as a PDF file for this project assignment.

* The total number of submissions for this project assignment will be limited to 30. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Now, adding your new IP address is automated by a script that periodically checks the Google Form at minutes 0, 20, and 40 during the hours between 09:00 and 00:40 the following day, and at minute 0 every hour between 01:00 and 09:00.
     + If you cannot reach the server a minute after the update time, check your IP address, as you might have sent the wrong IP address.
     + If you still cannot access the server after some time, it is likely due to an error in the automated process. The TAs will verify whether the script is running correctly, but since this check must be performed __manually__, please understand that it may not be completed immediately.

       
## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delayed.
* __You can use up to _3 slip days_ during this semester__. If your submission is delayed by one day and you decide to use one slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use on the QnA board of the submission server before the next project assignment is announced. Once slip days have been used, they cannot be canceled later, so saving them for later projects is highly recommended!
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
