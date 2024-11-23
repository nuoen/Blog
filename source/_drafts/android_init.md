# Android 启动流程 

idle 进程 优先级
生成  0 进程init 
生成 2 进程 system_server


基于android 10来分析

## Boot ROM
当按下电源键，硬件上电之后，会有一个固定的内存区域读取程序，这个程序是烧写到硬件上的ROM,用于将bootloader加载到RAM中，并开始执行它。

## bootloader
bootloader用于告诉设备如何找到系统内核，和启动内核。
手机厂商一般会在bootloader中加上密钥锁和一些限制.
bootloader执行一般分为两个阶段:
* 1. 检测外部RAM内存,并加载一段bootloader代码用于第二阶段的执行
* 2. 设置运行内核所需要的网络和内存等
高通芯片提供的LK,就可以作为一个Android的bootloader.
常见的bootloader有:
* U-boot
* LK

## kernel
Android使用的linux kernel,当kernel启动时,会执行一系列的初始化操作,比如设置缓存,内存,加载驱动程序,挂载根文件系统,初始化输入输出等.
当内核启动完成之后,第一件要做的事就是在系统文件中找一个“init”,作为根进程或者第一个系统进程.
看一下linux kernel的源码,找一个arm64架构开始分析:

入口在```arch/arm64/kernel/head.S```
```S

 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
	__HEAD
	/*
	 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
	 */
	efi_signature_nop			// special NOP to identity as PE/COFF executable 
	b	primary_entry			// branch to kernel start, magic 开始跳转到内核入口
	.quad	0				// Image load offset from start of RAM, little-endian
	le64sym	_kernel_size_le			// Effective size of kernel image, little-endian
	le64sym	_kernel_flags_le		// Informative flags, little-endian
	.quad	0				// reserved
	.quad	0				// reserved
	.quad	0				// reserved
	.ascii	ARM64_IMAGE_MAGIC		// Magic number
	.long	.Lpe_header_offset		// Offset to the PE header.

	__EFI_PE_HEADER

	.section ".idmap.text","a"

	/*
	 * The following callee saved general purpose registers are used on the
	 * primary lowlevel boot path:
	 *
	 *  Register   Scope                      Purpose
	 *  x19        primary_entry() .. start_kernel()        whether we entered with the MMU on
	 *  x20        primary_entry() .. __primary_switch()    CPU boot mode
	 *  x21        primary_entry() .. start_kernel()        FDT pointer passed at boot in x0
	 *  x22        create_idmap() .. start_kernel()         ID map VA of the DT blob
	 *  x23        primary_entry() .. start_kernel()        physical misalignment/KASLR offset
	 *  x24        __primary_switch()                       linear map KASLR seed
	 *  x25        primary_entry() .. start_kernel()        supported VA size
	 *  x28        create_idmap()                           callee preserved temp register
	 */

SYM_CODE_START(primary_entry) // 是代码的启动入口，这个函数负责在内核启动之前的各种配置
	bl	record_mmu_state  //记录当前 MMU 状态。它读取寄存器 SCTLR_EL1（在 EL1 模式下）或 SCTLR_EL2（在 EL2 模式下）的值，以确定 MMU、缓存等设置是否已开启。
	bl	preserve_boot_args //保存启动参数，以便后续阶段使用。
	bl	create_idmap //创建恒等映射（identity mapping），即物理地址和虚拟地址一一对应的映射，以确保在 MMU 关闭的情况下代码可以执行。

	/*
	 * If we entered with the MMU and caches on, clean the ID mapped part
	 * of the primary boot code to the PoC so we can safely execute it with
	 * the MMU off.
	 */
	cbz	x19, 0f
	adrp	x0, __idmap_text_start
	adr_l	x1, __idmap_text_end
	adr_l	x2, dcache_clean_poc
	blr	x2
0:	mov	x0, x19
	bl	init_kernel_el			// w0=cpu_boot_mode
	mov	x20, x0

	/*
	 * The following calls CPU setup code, see arch/arm64/mm/proc.S for
	 * details.
	 * On return, the CPU will be ready for the MMU to be turned on and
	 * the TCR will have been set.
	 */
#if VA_BITS > 48
	mrs_s	x0, SYS_ID_AA64MMFR2_EL1
	tst	x0, ID_AA64MMFR2_EL1_VARange_MASK
	mov	x0, #VA_BITS
	mov	x25, #VA_BITS_MIN
	csel	x25, x25, x0, eq
	mov	x0, x25
#endif
	bl	__cpu_setup			// initialise processor
	b	__primary_switch
SYM_CODE_END(primary_entry)

```
SYM_CODE_START(primary_entry) 宏解析
```s
#define ASM_NL ;
#define SYM_L_GLOBAL(name) .globl name

#define SYM_CODE_START(name) \
	SYM_START(name, SYM_L_GLOBAL , SYM_A_ALIGN)

#define SYM_START(name, linkage, align...) \
	SYM_ENTRY(name,linkage,align)

#define SYM_ENTRY(name,linkage,align...) \
	linkage(name) ASM_NL \
	align ASM_NL \
	name:
#解析为:
.globl primary_entry;
.balign 4; 
primary_entry:
```
```S
SYM_FUNC_START_LOCAL(__primary_switch)
	...
	ldr	x8, =__primary_switched
	adrp	x0, KERNEL_START		// __pa(KERNEL_START)
	br	x8
	...
SYM_FUNC_END(__primary_switch)

/*
 * The following fragment of code is executed with the MMU enabled.
 *
 *   x0 = __pa(KERNEL_START)
 */
SYM_FUNC_START_LOCAL(__primary_switched)
	adr_l	x4, init_task
	...
	mov	x0, x20
	bl	finalise_el2			// Prefer VHE if possible
	ldp	x29, x30, [sp], #16
	bl	start_kernel
	ASM_BUG()
SYM_FUNC_END(__primary_switched)
```
初始化会执行到__primary_switched,再执行start_kernel,start_kernel是内核C代码的入口
这个时候“0号”进程“swapper”(一个Idle进程)已经启动了(init_task.c中调用).
* 启动内核
```c
// 声明：inlude/linux/start_kernel.h
extern asmlinkage void __init __noreturn start_kernel(void);
/*
asmlinkage 是一个宏，用于指定函数的参数传递方式。它告诉编译器，函数的参数应从 堆栈 而不是 寄存器 中传递。
	•	作用：asmlinkage 在 Linux 内核中常用于定义一些系统调用函数，使得它们可以按照固定的方式接收参数，从而符合调用约定。通过 asmlinkage，内核可以在不同的体系架构上保持一致的参数传递方式，特别是在 x86 和其他架构中。

在 ARM64 中，asmlinkage 会影响函数调用的 ABI（应用程序二进制接口），通常用于系统调用和某些特殊的内核函数，以确保调用者在调用函数时符合规定的参数传递方式。*/

//实现：init/main.c

void start_kernel(void){
	...
	arch_call_rest_init();
	...
}


void __init __weak __noreturn arch_call_rest_init(void){
	rest_init();
}
/**
rest_init 函数的职责包括：

	1.	启动 init 进程，使其获得 PID 1。
	2.	启动 kthreadd 内核线程管理进程，负责管理所有的内核线程。
	3.	将系统状态设置为 SYSTEM_SCHEDULING，表明系统已经完成启动并准备好调度。
	4.	最终将 CPU 置于空闲状态。
*/

noinline void _ref __noreturn rest_init(void){
	int pid;

	pid = user_mode_thread(kernel_init,NULL,CLONE_FS);
}
/**
user_mode_thread 通过创建一个新的线程并切换到用户模式，完成了从内核空间向用户空间的过渡。这个过程需要配置和设置特定的寄存器、栈以及状态，以确保新线程可以在用户模式下正常运行。
1.	init 进程必须是用户态进程：
	•	在 Linux 系统中，init 进程（PID 1）是第一个用户态进程。所有用户态进程都是直接或间接从 init 派生的。
	•	init 进程通常负责启动用户空间的初始化脚本和守护进程，建立用户空间环境。因此，init 必须以用户态运行。
2.	kernel_init 的任务是启动用户空间应用：
	•	kernel_init 函数的作用是完成一些内核启动后的收尾工作，然后启动用户态的 /sbin/init 或者其他指定的初始化程序，以此开始加载用户态进程和服务。
	•	通过 user_mode_thread 进入用户模式执行 kernel_init，使得它可以顺利地过渡到用户空间，并启动用户态程序。	
3.	安全性和隔离性：
	•	将 kernel_init 放在用户模式运行可以将其与内核空间隔离。这种设计符合 Linux 内核的安全模型，使用户空间和内核空间隔离，避免了对内核资源的意外或恶意操作。
	•	用户态进程的错误（如非法访问内存）不会影响到内核本身的稳定性，而是被限制在用户空间。
4.	切换到用户态是 Linux 启动流程的必要步骤：
	•	Linux 的启动流程分为内核初始化和用户空间初始化两个主要阶段。rest_init 调用 user_mode_thread 生成一个用户态进程，这标志着内核的初始化阶段已经完成，系统从内核态过渡到用户态。
	•	这一步完成后，内核将主要作为用户空间进程的支持系统，不会主动执行其他任务，除非通过系统调用、异常或中断进入内核。
*/

/**kernel_init 函数的主要目标是执行用户空间的 init 程序（通常是 /sbin/init），从而让系统进入多用户操作模式，并为用户空间进程提供运行环境。*/
static int __ref kernel_init(void *unused){
	int ret;

	/*
	 * Wait until kthreadd is all set-up.
	•	wait_for_completion 等待 kthreadd（内核线程管理进程）完成初始化，确保在内核线程管理机制准备好之前不会继续执行。
	•	这是启动用户空间进程的前提条件，因为用户态的进程可能需要创建内核线程，而 kthreadd 管理所有内核线程的创建。
	 */
	wait_for_completion(&kthreadd_done);

	/**
	•	kernel_init_freeable 负责完成一些可以在内核空间中释放的初始化任务。这些任务包括加载设备驱动程序、设置文件系统等。
	•	此时系统仍处于内核态，确保内核态初始化完整后再释放内存。
	*/
	kernel_init_freeable();
	/* need to finish all async __init code before freeing the memory 
	•	async_synchronize_full() 确保所有的异步初始化操作都已完成。
	•	设置 system_state 为 SYSTEM_FREEING_INITMEM，表明系统已经准备好释放初始化用的内存。
	•	调用一系列函数（如 kprobe_free_init_mem、ftrace_free_init_mem 等）释放内核初始化过程中分配的内存。
	•	free_initmem() 释放 .init 段的内存，这部分内存仅用于启动时的内核态初始化，初始化完成后便不再需要，因此可以释放。
	•	mark_readonly() 将某些内核代码段设置为只读，以增强系统的安全性
	*/
	async_synchronize_full();

	system_state = SYSTEM_FREEING_INITMEM;
	kprobe_free_init_mem();
	ftrace_free_init_mem();
	kgdb_free_init_mem();
	exit_boot_config();
	free_initmem();
	mark_readonly();

	/*
	 * Kernel mappings are now finalized - update the userspace page-table
	 * to finalize PTI.
	 */
	pti_finalize();

	/**
	•	将系统状态设置为 SYSTEM_RUNNING，表示系统已经完成内核态的初始化，可以正常运行。
	•	设置默认的 NUMA（非一致性内存访问）策略。
	•	调用 rcu_end_inkernel_boot() 结束 RCU 的启动状态，进入正常的 RCU 调度状态。
	*/
	system_state = SYSTEM_RUNNING;
	numa_default_policy();

	rcu_end_inkernel_boot();
	/**
	•	do_sysctl_args() 解析启动时传入的系统控制参数（sysctl arguments），并应用这些参数。
	*/
	do_sysctl_args();
	/**
	内核接下来会尝试启动 init 进程，进入用户空间，并完成从内核态到用户态的过渡。它会按顺序检查不同的命令，直到成功启动一个 init 程序。
	*/
	if (ramdisk_execute_command) {
		ret = run_init_process(ramdisk_execute_command);
		if (!ret)
			return 0;
		pr_err("Failed to execute %s (error %d)\n",
		       ramdisk_execute_command, ret);
	}

	/*
	 * We try each of these until one succeeds.
	 *
	 * The Bourne shell can be used instead of init if we are
	 * trying to recover a really broken machine.
	 */
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}

	if (CONFIG_DEFAULT_INIT[0] != '\0') {
		ret = run_init_process(CONFIG_DEFAULT_INIT);
		if (ret)
			pr_err("Default init %s failed (error %d)\n",
			       CONFIG_DEFAULT_INIT, ret);
		else
			return 0;
	}

	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;

	panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/admin-guide/init.rst for guidance.");
}
内核态到用户态的过渡

	1.	启动 init 进程：内核首先在内核态完成各项初始化工作，创建并执行 init 进程。
	2.	执行用户空间的 init 程序：init 程序成功启动后，系统将从内核态过渡到用户态。此时，init 成为用户空间中第一个进程，接管系统控制权。
	3.	进入用户空间：执行 execve（或类似的系统调用）将用户空间的 init 程序加载到内存中，内核清除旧的内存映射并建立用户空间的内存映射，完成从内核态到用户态的过渡。
	4.	系统进入正常运行：一旦 init 进程运行，系统便进入多用户模式，可以运行用户应用程序和守护进程。

kernel_init:
	•	kernel_init 函数完成内核态的初始化任务并最终启动 init 进程。
	•	内核通过逐步释放内存、更新系统状态等操作，为进入用户态做好准备。
	•	当 init 程序启动成功，系统完成从内核态到用户态的过渡，内核初始化阶段结束，用户空间的系统管理程序正式接管控制权。

```
## init
源码：```system/core/init/main.cpp```
进程位置：```/system/core/init```

```cpp
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#endif

    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap function_map;

            return SubcontextMain(argc, argv, &function_map);
        }

        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }

        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }

    return FirstStageMain(argc, argv);
}
```
FirstStageMain
```cpp
int FirstStageMain(int argc, char** argv) {
	std::vector<std::pair<std::string, int>> errors;
	#define CHECKCALL(x) \
    if (x != 0) errors.emplace_back(#x " failed", errno);
	/**
	•	#x：# 是预处理操作符，将参数 x 转换为字符串字面量。例如，如果 x 是 open("/some/file")，那么 #x 会被替换为字符串 "open(\"/some/file\")"。
	•	#x " failed"：会生成类似 "open(\"/some/file\") failed" 的字符串，表示 x 的调用失败。
	•	errno：一个全局变量，通常用来存储最近一次系统调用或库函数的错误代码。errno 的值在发生错误时会被相应函数设置成错误代码，以便诊断错误原因。
	*/
    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));

    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    execv(path, const_cast<char**>(args));
}
```
执行```/system/bin/init selinux_setup```
SetupSelinux
```cpp
int SetupSelinux(char** argv) {
    InitKernelLogging(argv);

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    // Set up SELinux, loading the SELinux policy.
    SelinuxSetupKernelLogging();
    SelinuxInitialize();

    // We're in the kernel domain and want to transition to the init domain.  File systems that
    // store SELabels in their xattrs, such as ext4 do not need an explicit restorecon here,
    // but other file systems do.  In particular, this is needed for ramdisks such as the
    // recovery image for A/B devices.
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }

    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never return from this function.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```
执行```/system/bin/init second_stage```
SecondStageMain
```cpp

int SecondStageMain(int argc, char** argv) {

    Epoll epoll;
    if (auto result = epoll.Open(); !result) {
        PLOG(FATAL) << result.error();
    }
	InstallSignalFdHandler(&epoll);

	while (true) {}
}
```
### 一、概述
init进程是Linux系统中用户空间的第一个进程，进程号固定为1。Kernel启动后，在用户空间启动init进程，并调用init中的main()方法执行init进程的职责。对于init进程的功能为：
 * 挂载(mount) /sys, /dev 或者/proc等文件
 * 解析并运行所有的init.rc相关文件
 * 根据rc文件，生成相应的设备驱动节点
 * 处理子进程的终止(signal方式)
 * 提供属性服务的功能


### 解析启动脚本

### 启动解析的服务

### 守护解析的服务

### 启动java虚拟机

### 预加载资源

### 循环等待孵化进程

### 启动SystemServer 进程

### 创建SystemServer

### 管理SystemServer