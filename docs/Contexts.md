Contexts
========


Initial
-------

At startup, the kernel begins executing assembly in `startup.s`. This entire
file is designed to execute without using any stack, and interrupts should not
yet be triggered at this point. So there's not much to say about the context
this code executes within.

Just before executing `main()`, the stack pointer is set to the linker symbol
`stack_end`. This initial stack is 4KB, statically allocated. The kernel
initialization occurs in `main()` using this initial stack, up until the first
process is started.


Processes
---------

Each process has an associated kernel-mode stack, which is allocated by the page
allocator in `create_process()` or `create_kthread()`. This pointer is stored
within the `kstack` field of the process. Userspace processes manage their own
stack pointers without any kernel intervention.

At the end of initialization, the kernel uses `context_switch()` to enter the
first user-mode process. The kernel may be re-entered by a SWI or IRQ, at which
point the `kstack` field of `current` is stuck into the `sp` register, and used
for kernel stack frames.


Storing contexts
----------------

When a `SWI` occurs, the handler loads the current process's kernel stack into
`sp`, and dumps the current context into the top of the kernel stack. An SWI may
be triggered *only* by USR mode. A kernel thread in SYS triggering a SWI is a
bug. The user-space context stored by SWI can only be restored by returning from
the SWI.

When an `IRQ` occurs, the handler dumps the current context directly onto
the IRQ mode stack, and then calls into a handler. An IRQ may interrupt *any*
processor mode (except another interrupt): USR, SVC, or SYS. Interrupt handlers
MUST directly return (they cannot use setctx/resctx, see below). When interrupt
handlers return, they resume the context currently located on the IRQ mode
stack. In some cases (the timer interrupt being the only case as of now), the
context could have been switched during handling.

User-mode processes may only go to sleep by invoking a system call. The system
call, executing in SVC mode on behalf of the process, uses the `setctx()`
function to store its context into `current->context`. This function behaves
similar to `setjmp()`, in that it returns 0 at first, but could be resumed with
a different return value later.

Kernel threads work similarly, except they are in SYS mode, and there is no
system call which blocks on their behalf -- they simply call `setctx()`
directly.

Resuming contexts
-----------------

Thankfully, this is a fair bit easier than storing.  A context could be resumed
through the following:

1. The timer interrupt selects a new task and copies the `struct ctx` into its
   stack. When the interrupt handler returns, that context will be resumed.
2. The `resctx()` function is used to resume a context. Typically, this only
   happens in calls to `schedule()`, or sometimes `context_switch()` directly.

Since the setctx/resctx functions can only handle CPU contexts (not page tables
or anything else), they are not typically used directly for scheduling. However,
they can be used within kernel threads or system calls as a substitute for
setjmp/longjmp.

Race conditions
---------------

Interrupts may be triggered during any mode. If the interrupt switches the
currently active task, there could be some consequences.  If the currently
active task was in the process of storing contexts and switching to a new active
task, then this context could be clobbered. Since the kernel is fully
preemptive, we take care to disable preemption during these critical sections.

Preemption is always disabled when running a function in the `.nopreempt`
section. It is also controlled by a boolean flag.

At this point, I'm confident that the glaringly obvious race conditions with
respect to scheduling are taken care of.



上下文
========

初始
-------

启动时，内核开始在 `startup.s` 中执行汇编。整个文件设计为不使用任何堆栈执行，此时还不应触发中断。因此，关于此代码在其中执行的上下文，没有太多要说的。

在执行 `main()` 之前，堆栈指针设置为链接器符号 `stack_end`。此初始堆栈为 4KB，静态分配。内核初始化在 `main()` 中使用此初始堆栈进行，直到启动第一个进程。

进程
---------

每个进程都有一个关联的内核模式堆栈，由 `create_process()` 或 `create_kthread()` 中的页面分配器分配。此指针存储在进程的 `kstack` 字段内。用户空间进程管理自己的堆栈指针，无需任何内核干预。

在初始化结束时，内核使用 `context_switch()` 进入
第一个用户模式进程。内核可以通过 SWI 或 IRQ 重新进入，此时 `current` 的 `kstack` 字段被卡在 `sp` 寄存器中，并用于
内核堆栈帧。

存储上下文
----------------

当发生 `SWI` 时，处理程序将当前进程的内核堆栈加载到
`sp` 中，并将当前上下文转储到内核堆栈的顶部。SWI 可能
仅由 USR 模式触发。SYS 中的内核线程触发 SWI 是一个
错误。SWI 存储的用户空间上下文只能通过从
SWI 返回来恢复。

当发生 `IRQ` 时，处理程序将当前上下文直接转储到
IRQ 模式堆栈上，然后调用处理程序。IRQ 可以中断*任何*处理器模式（另一个中断除外）：USR、SVC 或 SYS。中断处理程序
必须直接返回（它们不能使用 setctx/resctx，见下文）。当中断
处理程序返回时，它们会恢复当前位于 IRQ 模式
堆栈上的上下文。在某些情况下（目前只有定时器中断的情况），上下文可能在处理过程中被切换。

用户模式进程只能通过调用系统调用才能进入睡眠状态。系统调用代表进程在 SVC 模式下执行，使用 `setctx()`
函数将其上下文存储到 `current->context` 中。此函数的行为类似于 `setjmp()`，因为它最初返回 0，但稍后可以使用不同的返回值恢复。

内核线程的工作方式类似，只是它们处于 SYS 模式，并且没有代表它们阻塞的系统调用——它们只是直接调用 `setctx()`。

恢复上下文
-----------------

幸运的是，这比存储要容易得多。可以通过以下方式恢复上下文：

1. 定时器中断选择一个新任务并将 `struct ctx` 复制到其堆栈中。当中断处理程序返回时，将恢复该上下文。

2. `resctx()` 函数用于恢复上下文。通常，这只发生在对 `schedule()` 的调用中，有时直接发生在 `context_switch()` 中。

由于 setctx/resctx 函数只能处理 CPU 上下文（而不是页表或其他任何东西），因此它们通常不直接用于调度。但是，它们可以在内核线程或系统调用中用作 setjmp/longjmp 的替代品。

竞争条件
---------------

中断可能在任何模式下触发。如果中断切换当前活动的任务，可能会产生一些后果。如果当前活动的任务正在存储上下文并切换到新的活动任务，那么这个上下文可能会被破坏。由于内核是完全抢占式的，因此我们注意在这些关键部分禁用抢占。

在 `.nopreempt`
部分中运行函数时，抢占始终处于禁用状态。它也由布尔标志控制。

此时，我相信与调度有关的明显竞争条件已得到解决。

