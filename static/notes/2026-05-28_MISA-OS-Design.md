# MISA OS Design

MISA doesn't support asynchronous interrupts, but it does support synchronous exceptions and
distinct privilege modes.

The goal is create a cooperative realtime OS that:

- Provides privilege separation and a syscall API for user code to access the hardware
- Provides other quality-of-life kernel services (semaphore?, timer?, etc.)
- Has a task scheduling system (tasks will need to "preempt" themselves by yielding to the kernel
  via a syscall) to manage basic concurency
  
The disadvantages of this architecture are:

- Lots of busy waiting and polling of MMIO for any time-based/asynchronous peripheral task
- Requires cooperation of all the tasks, and also fundamentally limits the maximum number of tasks
  and the complexity of those tasks

But without interrupts in hardware there isn't much of an alternative in software. This design will
still make it much easier to write useful programs for the MISA architecture.

## Layer 0: HAL

The HAL is primarily responsible for the vector table at `0xFFF0` - `0xFFFF` and all the handler
functions in the table. This will be served through `vectors.S`, which will:

- Populate `.section vectors` with vector function pointers
- Define the reset handler: initialize registers, jump to `_start`, trap or loop if `_start` returns
- Define the fault handler: dump processor state to memory, kill task if user-mode-caused, and loop
  infinitely if machine-mode-caused
- Define the syscall handler: switch on the syscall code and jump to Layer 2 services

The HAL will also create manage a system for dumping the entire state of the processor to a region
in memory. This region will be at a constant address and only one dump at a time is supported (not
an array of debug structs). This will be in `dump.S`. The dump must save:

- All general purpose registers, except `RSCRATCH`, but including `R0`
- All CSRs

The convention will be to dump this data into the `direct` section in the first memory page. That
allows writes to use `ST R* R0 RSCRATCH1`, allowing `RSCRATCH0` to be set to the constant `1` and
used to add/subtract from the `RSCRATCH1` write address.

Finally, the HAL will manage a file defining macro constants for the structures above.

## Layer 1: Context Switching and TCB

TCBs are Task Control Blocks, which are the task structures that hold information used for
context save-and-restore (registers, stack, sleep/block timers, etc). A macro file will define
offsets/sizes for TCB fields.

The main feature of this layer will be a `switch_to` function in `context.S`, which takes pointers
to two TCBs and switches from the first to the second. `switch_to` is called in the context of the
first TCB (i.e. when it yields to the kernel/scheduler) and will return in the context of the
second TCB.

No scheduling is done at this layer. Since context switching is the most dangerous/corruptable part
of the kernel, it will be as isolated/abstracted as possible from the perspective of the scheduler.

## Layer 2: Blocking Kernel Services

Kernel services are the implementation-level details of mechanisms that tasks can request which
have the ability to block a task. Tasks actually interact with these services via Layer 4, but the
implementation is in Layer 2. The minimum provided services should include:

- A semaphore structure (other libraries/extensions might use this to implement things like
  mutexes and condition variables, but that won't be a kernel service, and will probably be user-
  code that performs syscalls into the semaphore service)
- A concept of time. here won't be an easy way to track wall-clock time without interrupts, so
  the timer implementation will operate on atoms of "timer ticks," which are not guaranteed to have
  a specific wall-clock meaning. The timer can still be useful for guaranteeing a total ordering of
  tasks (e.g. if task A sleeps for 5 ticks and task B sleeps for 10 ticks, it is guaranteed task A
  will resume first, regardless of how much wall-clok time is in any given tick). Ticks will be
  defined as the time between each task yielding to the kernel.

## Layer 3: Task Scheduler

The scheduler implements a scheduling algorithm that decides which task to run next every time
control is returned to the kernel. When "control is returned to the kernel," that means control
is returned to the scheduler, which will pick a new task and use `switch_to` in Layer 1 to switch
contexts.

The scheduler will know that the current task is, and have a ready queue of tasks that can be run
at the current moment (added to when, e.g., a task wakes up after sleeping for N ticks). The
scheduler will also increment the global tick time by 1 every time it runs.

The scheduling function should never return. The control flow should be:

- A running task yields to the kernel (which is a syscall)
- The syscall handler sees the syscall code is "yield" and invokes the scheduler
- The scheduler chooses a new task and switches to it using the Layer 1 API
- Repeat (the scheduler doesn't return, and instead is indirectly entered via the new task yielding)

## Layer 4: System Call API

This API is what's exposed to user-mode tasks to access the hardware and kernel primitives. It must
define a set of syscall codes for the following calls:

- Yielding to the kernel --> goes to Layer 3
- Task management (creating, exiting, etc.) --> goes to Layer 1
- Semaphore usage (wait, post, etc. which use the kernel primitive) --> goes to Layer 2

## Layer 5: Tasks

Tasks are largely defined by programmers/users of the kernel. They interact only with the Layer 4
syscall API by requesting to:

- Create a task
- Yield to the kernel
- Interact with blocking kernel services

The one task that is guaranteed to exist, and which is managed by the kernel, is the idle task. It
is defined as follows:

```asm
idle_task:
    yield  // A macro for syscall with the yield code
    goto idle_task
```

The purpose of the idle task is to guarantee the scheduler always has something to run if every
user-defined task is sleeping/blocked on a given tick.

## Startup

The startup routine must do the following (e.g. in the reset handler)

- Set `SADDR` to the kernel stack
- Zero-initialize BSS
- Initialize the kernel (TCB array, ready list, semaphore pool, create an idle task)
- Initialize the first user task with `main` as the entry function
- Enter the scheduler (which will never return)
