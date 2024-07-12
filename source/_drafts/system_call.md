系统调用解析

trace:
```
───────────────────────────────────────────────────────────── code:arm64: ────
   0xffff800080023758 <__arm64_sys_ni_syscall+0010> ldp    x29,  x30,  [sp],  #16
   0xffff80008002375c <__arm64_sys_ni_syscall+0014> autiasp 
   0xffff800080023760 <__arm64_sys_ni_syscall+0018> ret    
●→ 0xffff800080023764 <__arm64_sys_getpuid+0000> paciasp 
   0xffff800080023768 <__arm64_sys_getpuid+0004> stp    x29,  x30,  [sp,  #-208]!
   0xffff80008002376c <__arm64_sys_getpuid+0008> mov    x4,  #0x800000000000        	// #140737488355328
   0xffff800080023770 <__arm64_sys_getpuid+000c> mov    x29,  sp
   0xffff800080023774 <__arm64_sys_getpuid+0010> stp    x19,  x20,  [sp,  #16]
   0xffff800080023778 <__arm64_sys_getpuid+0014> add    x19,  sp,  #0x50
──────────────────────────────────── source:../arch/arm64/kernel/sys.c+39 ────
     34	 		!system_supports_32bit_el0())
     35	 		return -EINVAL;
     36	 	return ksys_personality(personality);
     37	 }
     38	 
 →   39	 SYSCALL_DEFINE2(getpuid,pid_t __user *, pid,uid_t __user *,uid){
     40	 	pid_t kpid;
     41	 	uid_t kuid;
     42	 	printk("%s add syscall call",__func__);
     43	 	if(pid == NULL && uid == NULL){
     44	 		return -EINVAL;
───────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, stopped 0xffff800081854eb8 in cpu_do_idle (), reason: BREAKPOINT
[#1] Id 2, stopped 0xffff800080023764 in __arm64_sys_getpuid (), reason: BREAKPOINT
─────────────────────────────────────────────────────────────────── trace ────
[#0] 0xffff800080023764 → __arm64_sys_getpuid(regs=0xffff800086877eb0)
[#1] 0xffff80008002dabc → __invoke_syscall(syscall_fn=<optimized out>, regs=0xffff800086877eb0)
[#2] 0xffff80008002dabc → invoke_syscall(regs=0xffff800086877eb0, scno=0x1c9, sc_nr=0x1ca, syscall_table=0xffff8000818966c0 <sys_call_table>)
[#3] 0xffff80008002dc58 → el0_svc_common(regs=0xffff800086877eb0, scno=0x1c9, sc_nr=0x1ca, syscall_table=0xffff8000818966c0 <sys_call_table>)
[#4] 0xffff80008002dd68 → do_el0_svc(regs=0xffff800086877eb0)
[#5] 0xffff8000818536d4 → el0_svc(regs=0xffff800086877eb0)
[#6] 0xffff800081853a70 → el0t_64_sync_handler(regs=<optimized out>)
[#7] 0xffff800080011d4c → el0t_64_sync()

```
1. 用户空间调用 syscall，触发 svc 0。
2. CPU 进入内核模式，跳转到 el0t_64_sync。
3. 异常处理函数 el0t_64_sync_handler 被调用。
4. 进入 el0_svc 和 do_el0_svc 处理 SVC 异常。
5. 调用 el0_svc_common，它负责查找和调用具体的系统调用处理函数。
6. invoke_syscall 调用具体的系统调用处理函数。
7. __invoke_syscall 执行具体的系统调用逻辑，例如 __arm64_sys_getpuid。


1. 通过svc触发异常
在用户空间程序执行 svc 0 后，会触发一个异常，使得 CPU 从用户模式切换到特权模式（内核模式），并执行相应的异常处理程序。
```
mov x8, #0           // 设置系统调用号到 x8 寄存器
svc 0                // 触发系统调用
```

2. 触发异常后，会调用el0t_64_sync
el0代表此刻还在用户态
其定义在：
arch/arm64/kernel/entry.S
```
	entry_handler	0, t, 64, sync
```
enter_handler 是个宏定义，定义如下
```
	.macro entry_handler el:req, ht:req, regsize:req, label:req
SYM_CODE_START_LOCAL(el\el\ht\()_\regsize\()_\label)
	kernel_entry \el, \regsize
	mov	x0, sp
	bl	el\el\ht\()_\regsize\()_\label\()_handler
	.if \el == 0
	b	ret_to_user
	.else
	b	ret_to_kernel
	.endif
SYM_CODE_END(el\el\ht\()_\regsize\()_\label)
	.endm

```
SYM_CODE_START_LOCAL 宏：通常用于定义局部代码符号的开始，确保生成的汇编标签符合特定的格式，

* el\el\ht\()_\regsize\()_\label：
* * el\el\ht\()_\regsize\()_\label 这一部分展开成 el0t_64_sync，其中：
* * * el\el 是 el0。
* * * ht 是 t。
* * * regsize\() 是 64_。
* * * label 是 sync。
* * 最终生成的符号名是 el0t_64_sync。
* \()_：

* * * \( )_ 表示在宏参数后插入一个下划线 _，以便将参数值与下一个部分连接起来。
* * * 例如，\regsize\() 是 64_，即寄存器大小后跟一个下划线。

kernel_entry 宏，它通常用于设置进入内核时的上下文环境。这包括禁用中断、保存上下文和切换到内核模式等操作。
```
	.macro	kernel_entry, el, regsize = 64
	.if	\el == 0
	alternative_insn nop, SET_PSTATE_DIT(1), ARM64_HAS_DIT
	.endif
	.if	\regsize == 32
	mov	w0, w0				// zero upper 32 bits of x0
	.endif
	stp	x0, x1, [sp, #16 * 0]
	stp	x2, x3, [sp, #16 * 1]
	stp	x4, x5, [sp, #16 * 2]
	stp	x6, x7, [sp, #16 * 3]
	stp	x8, x9, [sp, #16 * 4]
	stp	x10, x11, [sp, #16 * 5]
	stp	x12, x13, [sp, #16 * 6]
	stp	x14, x15, [sp, #16 * 7]
	stp	x16, x17, [sp, #16 * 8]
	stp	x18, x19, [sp, #16 * 9]
	stp	x20, x21, [sp, #16 * 10]
	stp	x22, x23, [sp, #16 * 11]
	stp	x24, x25, [sp, #16 * 12]
	stp	x26, x27, [sp, #16 * 13]
	stp	x28, x29, [sp, #16 * 14]

    ...
    .endm
```
3. 调用 el0t_64_sync_handler
主要处理从 EL0（用户态）到 EL1（内核态）的64位同步异常

函数定义和初始化
```
asmlinkage void noinstr el0t_64_sync_handler(struct pt_regs *regs)
{
	unsigned long esr = read_sysreg(esr_el1);


```
asmlinkage：这是一个宏，通常用于指定函数的调用约定，确保参数通过堆栈传递而不是通过寄存器传递。这对于内核和用户态之间的调用非常重要。
noinstr：这是一种属性，指示编译器不要在这个函数中插入调试或分析相关的指令。
el0t_64_sync_handler：函数名，处理 EL0 到 EL1 的64位同步异常。
struct pt_regs *regs：一个指向保存异常发生时 CPU 寄存器状态的结构体的指针。
unsigned long esr = read_sysreg(esr_el1)：读取异常综合寄存器（ESR），这个寄存器包含了异常的详细信息。
ESR_ELx_EC_SVC64：处理系统调用异常

```
switch (ESR_ELx_EC(esr)) {
	case ESR_ELx_EC_SVC64:
		el0_svc(regs);
		break;
}
```
4. el0_svc
```
static void noinstr el0_svc(struct pt_regs *regs)
{
	enter_from_user_mode(regs);
	cortex_a76_erratum_1463225_svc_handler();
	fp_user_discard();
	local_daif_restore(DAIF_PROCCTX);
	do_el0_svc(regs);
	exit_to_user_mode(regs);
}

```
这段代码定义了一个静态函数 el0_svc，用于处理从用户态（EL0）到内核态（EL1）的系统调用（SVC，Supervisor Call）异常。让我们按照之前解释的规则对这段代码进行详细解析。

函数定义

static void noinstr el0_svc(struct pt_regs *regs)
static：表示这个函数在定义它的文件中是私有的，不能被其他文件引用。
noinstr：这个属性指示编译器不要在这个函数中插入调试或分析相关的指令。
void：表示函数不返回任何值。
el0_svc：函数名，处理从EL0到EL1的系统调用异常。
struct pt_regs *regs：一个指向保存异常发生时CPU寄存器状态的结构体的指针。
函数实现
```
{
	enter_from_user_mode(regs);
	cortex_a76_erratum_1463225_svc_handler();
	fp_user_discard();
	local_daif_restore(DAIF_PROCCTX);
	do_el0_svc(regs);
	exit_to_user_mode(regs);
}
```
1. enter_from_user_mode(regs)

enter_from_user_mode(regs);
这个函数将上下文从用户模式切换到内核模式。它通常包括保存用户态寄存器、设置内核态堆栈等操作。
2. cortex_a76_erratum_1463225_svc_handler()

cortex_a76_erratum_1463225_svc_handler();
这是一个特定于 Cortex-A76 处理器的错误处理程序。这个函数处理 Cortex-A76 处理器中的特定硬件错误（erratum 1463225）。
3. fp_user_discard()

fp_user_discard();
这个函数用于丢弃用户态的浮点上下文，以确保在内核态不会使用用户态的浮点寄存器。这通常是为了安全性和上下文切换的正确性。
4. local_daif_restore(DAIF_PROCCTX)

local_daif_restore(DAIF_PROCCTX);
这个函数用于恢复中断状态。DAIF_PROCCTX 是一个标志，用于指示当前的处理器上下文。这个函数通常会重新使能中断或其他处理器状态。
5. do_el0_svc(regs)

do_el0_svc(regs);
这个函数处理实际的系统调用逻辑。regs 包含了系统调用号和参数，通过这个函数调用具体的系统调用处理函数。
6. exit_to_user_mode(regs)

exit_to_user_mode(regs);
这个函数将上下文从内核模式切换回用户模式。它通常包括恢复用户态寄存器、设置用户态堆栈等操作。

总结
el0_svc 函数处理从用户态到内核态的系统调用异常。具体步骤如下：

* 进入内核模式：通过 enter_from_user_mode 保存用户态上下文并切换到内核模式。
* 处理特定处理器错误：通过 cortex_a76_erratum_1463225_svc_handler 处理 Cortex-A76 处理器的特定错误。
* 丢弃用户态浮点上下文：通过 fp_user_discard 确保内核态不会使用用户态浮点寄存器。
* 恢复中断状态：通过 local_daif_restore 恢复处理器的中断状态。
* 处理系统调用：通过 do_el0_svc 调用具体的系统调用处理函数。
返回用户模式：通过 exit_to_user_mode 恢复用户态上下文并返回用户模式。

5. do_el0_svc
```
void do_el0_svc(struct pt_regs *regs)
{
	el0_svc_common(regs, regs->regs[8], __NR_syscalls, sys_call_table);
}
```

5. el0_sev_common

函数定义

static void el0_svc_common(struct pt_regs *regs, int scno, int sc_nr,
                           const syscall_fn_t syscall_table[])
static：表示这个函数在定义它的文件中是私有的，不能被其他文件引用。
void：表示函数不返回任何值。
el0_svc_common：函数名，处理从 EL0（用户态）到 EL1（内核态）的系统调用。
struct pt_regs *regs：一个指向保存异常发生时 CPU 寄存器状态的结构体的指针。
int scno：系统调用号。
int sc_nr：系统调用的总数量。
const syscall_fn_t syscall_table[]：系统调用表，包含所有系统调用的函数指针。
函数实现
初始化和上下文保存

unsigned long flags = read_thread_flags();

regs->orig_x0 = regs->regs[0];
regs->syscallno = scno;
read_thread_flags()：读取当前线程的标志。
regs->orig_x0 = regs->regs[0]：保存原始的 x0 寄存器值。
regs->syscallno = scno：保存系统调用号。
注释部分：BTI 处理说明

/*
 * BTI note:
 * The architecture does not guarantee that SPSR.BTYPE is zero
 * on taking an SVC, so we could return to userspace with a
 * non-zero BTYPE after the syscall.
 *
 * This shouldn't matter except when userspace is explicitly
 * doing something stupid, such as setting PROT_BTI on a page
 * that lacks conforming BTI/PACIxSP instructions, falling
 * through from one executable page to another with differing
 * PROT_BTI, or messing with BTYPE via ptrace: in such cases,
 * userspace should not be surprised if a SIGILL occurs on
 * syscall return.
 *
 * So, don't touch regs->pstate & PSR_BTYPE_MASK here.
 * (Similarly for HVC and SMC elsewhere.)
 */
这段注释解释了 BTI（Branch Target Identification）的处理和可能出现的问题。主要说明不要修改 regs->pstate 中的 PSR_BTYPE_MASK 位。
处理异步标记检查故障

if (flags & _TIF_MTE_ASYNC_FAULT) {
    syscall_set_return_value(current, regs, -ERESTARTNOINTR, 0);
    return;
}
检查 _TIF_MTE_ASYNC_FAULT 标志，如果设置了这个标志，表示有异步的标记检查故障需要处理。
syscall_set_return_value(current, regs, -ERESTARTNOINTR, 0)：设置系统调用返回值为 -ERESTARTNOINTR，表示系统调用需要重启。
return：返回，不继续执行后续的系统调用处理。
处理系统调用的额外工作

if (has_syscall_work(flags)) {
    if (scno == NO_SYSCALL)
        syscall_set_return_value(current, regs, -ENOSYS, 0);
    scno = syscall_trace_enter(regs);
    if (scno == NO_SYSCALL)
        goto trace_exit;
}
has_syscall_work(flags)：检查是否有系统调用需要处理的额外工作，例如 ptrace 相关的操作。
如果系统调用号为 NO_SYSCALL，设置返回值为 -ENOSYS（系统调用不存在）。
syscall_trace_enter(regs)：处理系统调用进入的跟踪逻辑，并返回可能修改过的系统调用号。
如果系统调用号仍然是 NO_SYSCALL，跳转到 trace_exit 标签，进行系统调用退出的跟踪逻辑。
调用具体的系统调用

invoke_syscall(regs, scno, sc_nr, syscall_table);
invoke_syscall(regs, scno, sc_nr, syscall_table)：调用系统调用表中的具体系统调用函数。
检查和处理系统调用退出的跟踪逻辑

if (!has_syscall_work(flags) && !IS_ENABLED(CONFIG_DEBUG_RSEQ)) {
    flags = read_thread_flags();
    if (!has_syscall_work(flags) && !(flags & _TIF_SINGLESTEP))
        return;
}

trace_exit:
syscall_trace_exit(regs);
再次检查是否有系统调用需要处理的额外工作以及是否启用了 RSEQ（重启动序列）调试。
如果没有额外工作需要处理，并且没有启用单步调试，则直接返回。
如果有工作需要处理，或者启用了单步调试，调用 syscall_trace_exit(regs) 进行系统调用退出的跟踪逻辑。
总结
el0_svc_common 函数处理系统调用的主要逻辑，包括：

保存上下文：保存原始寄存器值和系统调用号。
处理异步标记检查故障：在系统调用实际执行前，处理异步标记检查故障。
处理系统调用的额外工作：包括 ptrace 相关的操作。
调用具体的系统调用：根据系统调用号，从系统调用表中调用具体的系统调用函数。
处理系统调用退出的跟踪逻辑：包括检查系统调用退出时是否需要进行额外的处理。
通过这些步骤，确保系统调用的执行是安全且符合预期的，同时处理了系统调用前后的各种特殊情况。

6.invoke_syscall()
```
static void invoke_syscall(struct pt_regs *regs, unsigned int scno,
			   unsigned int sc_nr,
			   const syscall_fn_t syscall_table[])
{
	long ret;

	add_random_kstack_offset();

	if (scno < sc_nr) {
		syscall_fn_t syscall_fn;
		syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
		ret = __invoke_syscall(regs, syscall_fn);
	} else {
		ret = do_ni_syscall(regs, scno);
	}

	syscall_set_return_value(current, regs, 0, ret);

	/*
	 * Ultimately, this value will get limited by KSTACK_OFFSET_MAX(),
	 * but not enough for arm64 stack utilization comfort. To keep
	 * reasonable stack head room, reduce the maximum offset to 9 bits.
	 *
	 * The actual entropy will be further reduced by the compiler when
	 * applying stack alignment constraints: the AAPCS mandates a
	 * 16-byte (i.e. 4-bit) aligned SP at function boundaries.
	 *
	 * The resulting 5 bits of entropy is seen in SP[8:4].
	 */
	choose_random_kstack_offset(get_random_u16() & 0x1FF);
}
```

让我们按照之前的规则对 invoke_syscall 函数进行详细解析。这段代码用于调用系统调用处理函数，根据系统调用号选择合适的系统调用处理程序，并设置返回值。

函数定义

static void invoke_syscall(struct pt_regs *regs, unsigned int scno,
			   unsigned int sc_nr,
			   const syscall_fn_t syscall_table[])
static：表示这个函数在定义它的文件中是私有的，不能被其他文件引用。
void：表示函数不返回任何值。
invoke_syscall：函数名，处理系统调用的实际执行。
struct pt_regs *regs：一个指向保存异常发生时 CPU 寄存器状态的结构体的指针。
unsigned int scno：系统调用号。
unsigned int sc_nr：系统调用的总数量。
const syscall_fn_t syscall_table[]：系统调用表，包含所有系统调用的函数指针。
函数实现
添加随机堆栈偏移

add_random_kstack_offset();
add_random_kstack_offset()：这个函数用于添加一个随机的内核堆栈偏移，增加堆栈布局的随机性以提高安全性。
调用具体的系统调用

if (scno < sc_nr) {
	syscall_fn_t syscall_fn;
	syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
	ret = __invoke_syscall(regs, syscall_fn);
} else {
	ret = do_ni_syscall(regs, scno);
}
if (scno < sc_nr)：检查系统调用号是否在有效范围内（即是否小于总的系统调用数量）。
有效系统调用：
syscall_fn_t syscall_fn;：声明一个系统调用函数指针。
syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];：从系统调用表中获取对应的系统调用函数，使用 array_index_nospec 来防止投机执行漏洞。
ret = __invoke_syscall(regs, syscall_fn);：调用具体的系统调用处理函数，并将返回值保存到 ret 中。
无效系统调用：
ret = do_ni_syscall(regs, scno);：调用未实现的系统调用处理函数 do_ni_syscall，并将返回值保存到 ret 中。
设置系统调用返回值

syscall_set_return_value(current, regs, 0, ret);
syscall_set_return_value(current, regs, 0, ret)：设置系统调用的返回值。
current：当前进程的任务结构指针。
regs：保存寄存器状态的结构指针。
0：系统调用成功的标志（通常为 0）。
ret：系统调用的返回值。
选择新的随机堆栈偏移

/*
 * Ultimately, this value will get limited by KSTACK_OFFSET_MAX(),
 * but not enough for arm64 stack utilization comfort. To keep
 * reasonable stack head room, reduce the maximum offset to 9 bits.
 *
 * The actual entropy will be further reduced by the compiler when
 * applying stack alignment constraints: the AAPCS mandates a
 * 16-byte (i.e. 4-bit) aligned SP at function boundaries.
 *
 * The resulting 5 bits of entropy is seen in SP[8:4].
 */
choose_random_kstack_offset(get_random_u16() & 0x1FF);
注释部分解释了为什么要限制堆栈偏移值，并确保其合理性和安全性。
choose_random_kstack_offset(get_random_u16() & 0x1FF)：选择一个新的随机堆栈偏移值。
get_random_u16() & 0x1FF：生成一个 16 位的随机数，并将其与 0x1FF 进行与操作，限制偏移值在 9 位以内。
总结
invoke_syscall 函数主要负责调用系统调用处理函数，具体步骤如下：

添加随机堆栈偏移：通过 add_random_kstack_offset() 增加堆栈布局的随机性。
检查系统调用号的有效性：如果系统调用号在有效范围内，从系统调用表中获取对应的处理函数并调用它；否则调用未实现的系统调用处理函数。
设置系统调用返回值：通过 syscall_set_return_value(current, regs, 0, ret) 设置系统调用的返回值。
选择新的随机堆栈偏移：通过 choose_random_kstack_offset(get_random_u16() & 0x1FF) 选择一个新的随机堆栈偏移值。
通过这些步骤，invoke_syscall 确保系统调用的执行过程是安全的，并增加了堆栈布局的随机性以提高安全性。


7. static long __invoke_syscall(struct pt_regs *regs, syscall_fn_t syscall_fn)
{
	return syscall_fn(regs);
}
这里便开始执行真正的系统调用函数。
