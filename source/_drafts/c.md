### 宏定义


#### __init:
`#define __init		__section(".init.text") __cold  __latent_entropy __noinitretpoline __nocfi `

__section(".init.text")：

这个指令将函数放置在输出可执行文件的特定段中，本例中为“.init.text”段。这个段通常用于系统启动时需要的初始化程序，初始化完成后可以丢弃，以节省内存。
__cold：

这个属性告诉编译器，该函数不太可能经常执行。这可以帮助编译器优化代码以提高性能，可能通过安排函数的方式来减少其在不经常使用时对指令缓存性能的影响。
__latent_entropy：

这个属性用于增加系统的熵（随机性），这在安全方面特别有用，例如，使依赖于可预测执行模式的攻击更加困难。
__noinitretpoline：

这个属性表明该函数不需要retpoline（返回跳板），这是一种针对某些类型的投机执行侧通道攻击（如Spectre）的缓解技术。
__nocfi：

这个指令告诉编译器不对这个函数应用控制流完整性（CFI）检查。控制流完整性是一种安全特性，通过确保间接函数调用转到有效函数来帮助防止某些类型的攻击。对于某些低级函数，跳过CFI可能是必要的，因为CFI的开销或限制可能会干扰函数的操作。

#### 从device_initcall查看宏定义
device_initcall(binder_init)就是将指向binder_init的函数指针，注册在.initcall6.init段里。内核启动时，会调用它，对Binder驱动进行初始化。
```
typedef int (*initcall_t)(void);
typedef initcall_t initcall_entry_t;

/* Format: <modname>__<counter>_<line>_<fn> */
#define __initcall_id(fn)					\
	__PASTE(__KBUILD_MODNAME,				\
	__PASTE(__,						\
	__PASTE(__COUNTER__,					\
	__PASTE(_,						\
	__PASTE(__LINE__,					\
	__PASTE(_, fn))))))

/* Format: __<prefix>__<iid><id> */
#define __initcall_name(prefix, __iid, id)			\
	__PASTE(__,						\
	__PASTE(prefix,						\
	__PASTE(__,						\
	__PASTE(__iid, id))))
      
#define __initcall_section(__sec, __iid)			\
	#__sec ".init"

#define __initcall_stub(fn, __iid, id)	fn

#define ____define_initcall(fn, __unused, __name, __sec)	\
	static initcall_t __name __used 			\
		__attribute__((__section__(__sec))) = fn;
#endif

#define __unique_initcall(fn, id, __sec, __iid)			\
	____define_initcall(fn,					\
		__initcall_stub(fn, __iid, id),			\
		__initcall_name(initcall, __iid, id),		\
		__initcall_section(__sec, __iid))

#define ___define_initcall(fn, id, __sec)			\
	__unique_initcall(fn, id, __sec, __initcall_id(fn))

#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

#define device_initcall(fn)		__define_initcall(fn, 6)
```

device_initcall(binder_init)展开就是：
```
device_initcall(binder_init)
        ↓
__define_initcall(binder_init, 6)
        ↓
___define_initcall(binder_init, 6, .initcall6)
        ↓
__unique_initcall(binder_init, 6, .initcall6, <modname>__<counter>_<line>_binder_init)
        ↓
____define_initcall(binder_init, 
        binder_init,
        __initcall__<modname>__<counter>_<line>_binder_init6,
        .initcall6.init)
        ↓
static initcall_t __initcall__<modname>__<counter>_<line>_binder_init6 __used 
      __attribute__((__section__(.initcall6.init))) = binder_init;
```

##### \_\_initcall_id
这段代码定义了一个宏 __initcall_id(fn)，其目的是生成一个唯一的标识符，用于内核初始化函数（initcall）。这个宏通过多层嵌套的 __PASTE 宏，将几个预处理器变量和函数名结合成一个连续的字符串。下面是对这个宏的详细解释：

*组件解释*
* \_\_KBUILD_MODNAME：
这是一个由内核构建系统定义的宏，它表示正在编译的模块的名称。如果代码不是为模块编译的，它通常是内核的名称或为空。
* \_\_COUNTER\_\_：
这是一个特殊的预处理器变量，每次出现在源代码中时都会递增。\_\_COUNTER__ 用于在编译过程中生成唯一的序列号，确保每次宏展开时都得到一个不同的值。是GNU 编译器的非标准编译器扩展，可以认为它是一个计数器，代表一个整数，它的值一般被初始化为0，在每次编译器编译到它时，会自动 +1。
* \_\_LINE\_\_：
这个宏展开为当前代码行的行号。它在源代码中是唯一的，除非多个宏展开发生在同一行。
* fn：
这是宏的参数，表示将要注册的初始化函数的名字。
* __PASTE 宏
__PASTE 宏的目的是将两个参数连接在一起（没有空格）。__initcall_id 宏通过多次使用 __PASTE，将上述元素拼接成一个单一的、唯一的标识符。这种连续使用 __PASTE 的方式确保了不同部分之间没有空格，并且生成的字符串是平滑连续的。

*完整的宏展开过程*
    如果我们假设：
    \_\_KBUILD_MODNAME 是 "mykernelmodule"
    \_\_COUNTER\_\_ 当前是 42
    \_\_LINE\_\_ 是 123
    fn 是 init_my_device
宏展开的结果将是：`mykernelmodule__42_123_init_my_device`
这个结果是由以下步骤生成的：
1. 首先连接 mykernelmodule 和 _，结果是 mykernelmodule__。
2. 然后连接前面的结果和 42，得到 mykernelmodule__42。
3. 接着连接 _ 和 123，再和前面的结果连接，得到 mykernelmodule__42_123。
4. 最后将 init_my_device 通过 _ 连接上去，得到最终的字符串 mykernelmodule__42_123_init_my_device。

```
#define __initcall_section(__sec, __iid)			\
	#__sec ".init"
这个宏 __initcall_section(__sec, __iid) 定义了一个用于生成内核初始化代码段名的格式。宏的工作是把传入的 __sec 和字符串 .init 拼接起来，构成一个用于特定初始化函数的ELF段（section）名称。这里，__iid 参数虽然在宏定义中出现，但在实际的字符串连接操作中没有被使用，可能是为了保持与其他相关宏的参数格式一致性，或为未来的扩展预留。
```
详细解释：
* #__sec：
这部分使用了字符串化操作符 #，它会将宏参数转化为一个字符串。这意味着传入的宏参数 __sec 会被转换成一个字面量字符串。
* ".init"：
这是一个直接的字符串常量，表示初始化段的后缀。在Linux内核中，用于初始化代码的段通常以 .init 结尾，比如 .init.text、.init.data 等。
拼接：

宏中没有明显的连接操作（如 ## 运算符），但在宏的用法中，这两部分（#__sec 和 ".init"）通过宏替换自然连接成一个完整的字符串。
使用例子：
假设有如下调用：

__initcall_section(my_section, 123)
根据宏的定义，这将展开为：

"my_section.init"
这里，my_section 被字符串化并与 .init 后缀连接，生成一个新的字符串，表示一个内核ELF段的名称。

#### initcall机制
当我们试图将一个驱动程序加载进内核时，我们需要提供一个`xxx_init()`函数。这样内核才会定位到该函数，加载驱动，初始化驱动。
`binder_init()`就是这样一个初始化驱动的函数。但是怎么向内核注册这样一个函数呢？直观的做法是维护一个初始化驱动的函数指针的数组，将`binder_init()`添加进该数组中。不过这样在多人开发时，容易造成编码冲突。
linux采用了更优雅的方法——initcall机制：
在内核镜像文件中，自定义一个段，这个段里面专门用来存放这些初始化函数的地址，内核启动时，只需要在这个段地址处取出函数指针，一个个执行即可。
##### .initcallXX.init段
`.initcallXX.init`段就是专门用来存放各个内核模块的初始化函数的地址。

`device_initcall(fn)`就是表示将指向fn的函数指针，存放在`.initcall6.init`段。类似的宏定义有：
```
#define pure_initcall(fn)		__define_initcall(fn, 0)              →  .initcall0.init
#define core_initcall(fn)		__define_initcall(fn, 1)              →  .initcall1.init
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)             →  .initcall1s.init
#define postcore_initcall(fn)		__define_initcall(fn, 2)              →  .initcall2.init
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)             →  .initcall2s.init
#define arch_initcall(fn)		__define_initcall(fn, 3)              →  .initcall3.init
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)             →  .initcall3s.init
#define subsys_initcall(fn)		__define_initcall(fn, 4)              →  .initcall4.init
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)             →  .initcall4s.init
#define fs_initcall(fn)			__define_initcall(fn, 5)              →  .initcall5.init
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)             →  .initcall5s.init
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)         →  .initcallrootfs.init
#define device_initcall(fn)		__define_initcall(fn, 6)              →  .initcall6.init
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)             →  .initcall6s.init
#define late_initcall(fn)		__define_initcall(fn, 7)              →  .initcall7.init
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)             →  .initcall7s.init
```

#### .initcallXX.init段的定义
.initcallXX.init段的定义是在common/include/asm-generic/vmlinux.lds.h和common/arch/arm64/kernel/vmlinux.lds.S中。
```
//common/include/asm-generic/vmlinux.lds.h
#define INIT_CALLS_LEVEL(level)						\
		__initcall##level##_start = .;				\
		KEEP(*(.initcall##level##.init))			\
		KEEP(*(.initcall##level##s.init))			\

#define INIT_CALLS							\
		__initcall_start = .;					\
		KEEP(*(.initcallearly.init))				\
		INIT_CALLS_LEVEL(0)					\
		INIT_CALLS_LEVEL(1)					\
		INIT_CALLS_LEVEL(2)					\
		INIT_CALLS_LEVEL(3)					\
		INIT_CALLS_LEVEL(4)					\
		INIT_CALLS_LEVEL(5)					\
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					\
		INIT_CALLS_LEVEL(7)					\
		__initcall_end = .;
                
//common/arch/arm64/kernel/vmlinux.lds.S
SECTIONS
{
        ...
        .init.data : {
		INIT_DATA
		INIT_SETUP(16)
		INIT_CALLS
		CON_INITCALL
		INIT_RAM_FS
		*(.init.altinstructions .init.bss)	/* from the EFI stub */
	}
        ...
}
```
参考：https://www.cnblogs.com/jianhua1992/p/16852793.html
解释：
* *() 
在链接器脚本中确实用作通配符。在链接器脚本的语法中，*() 用于匹配符合特定模式的所有符号或者段(section)。这种用法允许链接器指令应用于多个相关的符号或段，而不需要单独列举它们。这是组织和控制链接输出文件的一个强大特性，特别是在操作系统内核或其他复杂软件系统的构建中。

KEEP(*(.initcall##level##.init)) 和 KEEP(*(.initcall##level##s.init)) 使用了通配符 * 来包含所有在特定命名模式的段中的符号。这里的通配符 * 表示匹配所有符号，而括号内的表达式定义了要匹配的段名。例如：

.initcall0.init
.initcall1.init
.initcall0s.init
.initcall1s.init

*(.initcall##level##.init)：

这表示匹配所有在 .initcall<level>.init 段中的符号。例如，如果 level 是 0，则匹配 .initcall0.init 段中的所有符号。
*(.initcall##level##s.init)：

类似地，这表示匹配所有在 .initcall<level>s.init 段中的符号。这个 s 后缀可能代表某种特定类型的初始化调用，如同步初始化调用。
为什么使用 KEEP
KEEP 指令在链接器脚本中用来确保即使没有被程序的其他部分直接引用，也不会被链接器优化掉（即不被删除）。这对于初始化代码特别重要，因为这些代码在执行完毕后通常不会再被其他运行时代码直接引用，但它们在系统启动阶段是必需的。通过使用 KEEP，可以保证这些重要的初始化段不会因为“看似”未被使用而被剔除，确保内核能够按预期顺序执行这些初始化步骤。

这种机制是内核和其他底层系统软件能够精确控制初始化过程的关键，特别是在涉及到内存和资源优化的上下文中。

* ##在宏定义中的作用是将多个符号连接成一个符号，并不将其字符串化

### 标号元素
```
static const struct file_operations __maybe_unused kmem_fops = {
    .llseek     = memory_lseek,
    .read       = read_kmem,
    .write      = write_kmem,
    .mmap       = mmap_kmem,
    .open       = open_kmem,
#ifndef CONFIG_MMU
    .get_unmapped_area = get_unmapped_area_mem,
    .mmap_capabilities = memory_mmap_capabilities,
#endif
};
```
#### 解释
1. 结构体声明与类型
struct file_operations：这是一个预定义的结构体类型，通常在Linux内核源代码中定义，用于描述文件操作相关的函数指针。
2. 修饰符和存储类
static：表示这个结构体变量 kmem_fops 的作用域限制在定义它的文件内，外部文件不能访问它。
const：表示这个结构体变量是常量，一旦初始化后，其内容不应被修改。
__maybe_unused：这是一个编译器指示，用于告诉编译器这个变量可能在代码中未被使用，这可以避免编译器发出未使用变量的警告。这通常用在可能因为不同的编译配置而使用或不使用的变量上。
3. 结构体成员初始化
.llseek = memory_lseek，.read = read_kmem，等：这种使用点（.）后跟成员名的初始化方式是C99标准中引入的指定初始化器（designated initializer）。它允许在初始化结构体时直接指定每个成员的值，而不需要依赖于成员在结构体中的顺序。
4. 条件编译
#ifndef CONFIG_MMU ... #endif：这是预处理指令，用于条件编译。它检查 CONFIG_MMU 是否未被定义。如果未定义 CONFIG_MMU，则编译器会编译这两行代码：
.get_unmapped_area = get_unmapped_area_mem
.mmap_capabilities = memory_mmap_capabilities
这通常在不同的硬件配置或内核配置选项下使用，以适应不同的系统需求。

当在内核模块中将 memory_lseek 函数指定给 file_operations 结构体的 .llseek 成员时，任何针对该设备文件的 lseek 系统调用最终都会调用你的 memory_lseek 函数。
#### 使用
这里是详细的调用流程：

用户空间调用：当用户空间的程序调用 lseek() 函数时，这个调用最初由标准C库（例如glibc）封装，该库将系统调用参数封装为适当的系统调用请求。
系统调用转换：系统调用接口在内核中接收到这个请求后，会找到与文件描述符关联的 file 结构体实例。
文件操作结构体：每个打开的文件在内核中都有一个与之关联的 file 结构体，这个结构体包含了指向 file_operations 结构体的指针。file_operations 结构体为各种文件操作定义了具体的函数指针，如打开、读取、写入、定位等。
调用 .llseek：如果对应的 file_operations 结构体中的 .llseek 成员被设置为 memory_lseek，那么内核就会调用 memory_lseek 函数来处理这个定位请求。
执行定位操作：memory_lseek 函数将根据提供的参数（文件指针、偏移量和起始位置标志）计算新的文件位置，并将这个新位置返回给调用者。如果操作有效，返回的就是新的文件偏移量；如果无效（如偏移量无效或操作不被支持），则返回错误码。


c语言结构体大小计算规则：
1、结构体变量的首地址，必须是结构体变量中的“最大基本数据类型成员所占字节数”的整数倍。（对齐）
2、结构体变量中的每个成员相对于结构体首地址的偏移量，都是该成员基本数据类型所占字节的整数倍。（对齐）
3、结构体变量的总大小，为结构体变量中“最大基本数据类型成员所占字节数”的整数倍（补齐）

--
typedef struct Work_{
    int num;

}Work_;

如果不对自己起别名，直接用
Work_ work=new Work_(); 会报错，必须用 struct Work_ work=new Work_();
--

c:内存申请

栈：占最大内存中的2M,开辟内存的方式是静态内存开辟，比如：函数内声明变量，函数方法结束后会自动回收
堆：占最大内存中的80%，动态内存开辟，不会自动回收,必须手动回收。malloc()开辟，free()释放

利用realloc扩展内存，如果原来的内存不足以向下扩展，会重新找一块足够的内存，因此，指针指向的地址会变动。
动态申请内存时有可能会失败，所以会返回null,如果是扩展内存返回null,那么需要把malloc的内存释放掉
释放时，也务必进行空指针判断，释放完最好把指针置空
'''
//动态开辟内存
void dyMemoryTest(int num,int newNum){
    int* arrPoint= (int*)malloc(10*1024*1024*sizeof(int)); //40M

    //必须手动释放
    if(arrPoint){
        free(arrPoint);  
    }
    int* arrPoint2=(int*)malloc(num*sizeof(int));
    if(arrPoint2){
        for(int i=0;i<num;i++){
            *(arrPoint2+i)=i;
        }
    }
    int* arrPoint3 =(int *)realloc(arrPoint2,(num+newNum)*sizeof(int));
    if(arrPoint3){
        for(int i=num;i<num+newNum;i++){
            *(arrPoint3+i)=i;
        }
        for(int i=0;i<num+newNum;i++){
            LOGD("the number is %d ,the value is %d",i,*(arrPoint3+i));
        }
        //成功了，只释放旧指针就行
        free(arrPoint3);
        arrPoint3= nullptr;
    }else{
        //没有成功，释放旧指针
        if(arrPoint2){
            free(arrPoint2);
        }
    }
}
'''

linux config example:

CONFIG_FTRACE_SYSCALLS 是一个内核配置选项，用于控制是否启用系统调用的函数追踪（ftrace）功能.

linux/syscalls.h
```
#ifdef CONFIG_FTRACE_SYSCALLS
#define SYSCALL_METADATA(sname, nb, ...)			\
	static const char *types_##sname[] = {			\
		__MAP(nb,__SC_STR_TDECL,__VA_ARGS__)		\
	};							\
	static const char *args_##sname[] = {			\
		__MAP(nb,__SC_STR_ADECL,__VA_ARGS__)		\
	};							\
	SYSCALL_TRACE_ENTER_EVENT(sname);			\
	SYSCALL_TRACE_EXIT_EVENT(sname);			\
	static struct syscall_metadata __used			\
	  __syscall_meta_##sname = {				\
		.name 		= "sys"#sname,			\
		.syscall_nr	= -1,	/* Filled in at boot */	\
		.nb_args 	= nb,				\
		.types		= nb ? types_##sname : NULL,	\
		.args		= nb ? args_##sname : NULL,	\
		.enter_event	= &event_enter_##sname,		\
		.exit_event	= &event_exit_##sname,		\
		.enter_fields	= LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), \
	};							\
	static struct syscall_metadata __used			\
	  __section("__syscalls_metadata")			\
	 *__p_syscall_meta_##sname = &__syscall_meta_##sname;

static inline int is_syscall_trace_event(struct trace_event_call *tp_event)
{
	return tp_event->class == &event_class_syscall_enter ||
	       tp_event->class == &event_class_syscall_exit;
}
#else
#define SYSCALL_METADATA(sname, nb, ...)

static inline int is_syscall_trace_event(struct trace_event_call *tp_event)
{
	return 0;
}
#endif

#ifend

```
if you define CONFIG_FTRACE_SYSCALLS in kconfig file,it will exe "ifdefine" block else it will exe "else" block. 

在C语言的宏定义中，## 操作符被称为“粘合”或“连接”操作符（token-pasting operator）。它的作用是将两个令牌（token）连接成一个新的令牌。这在宏定义和宏展开时非常有用，可以动态生成新的标识符或变量名。

让我们详细解释一下：

使用 ## 的例子
考虑以下宏定义：
```#define CONCAT(a, b) a##b```
如果我们使用这个宏：
```CONCAT(foo, bar)```
这将被预处理器展开为：
```foobar```

在C语言的宏定义中，# 操作符被称为字符串化操作符（stringizing operator）。它的作用是将宏参数转换为字符串字面量。这在宏定义和预处理过程中非常有用，尤其是在需要生成字符串的情况下。

解释 # 操作符
让我们详细解释一下 # 操作符的使用：

字符串化操作符：

```#define TO_STRING(x) #x```
这个宏 TO_STRING 将参数 x 转换为字符串字面量。例如：

```TO_STRING(hello)```
这将被预处理器展开为：


```"hello"```
在 SYSCALL_METADATA 宏中的使用：

```.name = "sys"#sname,``
这里的 #sname 将 sname 宏参数转换为字符串字面量，并与 "sys" 连接在一起。
字符串强化

```c
extern asmlinkage void __init __noreturn start_kernel(void);
```
这行代码声明了一个函数 start_kernel，其中包含了多个关键字和修饰符。每个关键字都有特定的含义和作用。下面我们逐个详细解释每个关键词的含义及其在这段代码中的用途：

extern asmlinkage void __init __noreturn start_kernel(void);

1. extern

	•	含义：extern 关键字表示 start_kernel 是一个外部函数，即它在其他文件中定义，而非在当前文件中实现。
	•	作用：在多个源文件之间共享函数或变量时，extern 用于告诉编译器该函数的定义在别的文件中。这避免了重复定义的问题，并允许链接器在其他文件中找到函数的实际定义。

在这段代码中，extern 表示 start_kernel 函数的定义位于其他源文件中。

2. asmlinkage

	•	含义：asmlinkage 是一个宏，用于指定函数的参数传递方式。它告诉编译器，函数的参数应从 堆栈 而不是 寄存器 中传递。
	•	作用：asmlinkage 在 Linux 内核中常用于定义一些系统调用函数，使得它们可以按照固定的方式接收参数，从而符合调用约定。通过 asmlinkage，内核可以在不同的体系架构上保持一致的参数传递方式，特别是在 x86 和其他架构中。

在 ARM64 中，asmlinkage 会影响函数调用的 ABI（应用程序二进制接口），通常用于系统调用和某些特殊的内核函数，以确保调用者在调用函数时符合规定的参数传递方式。

3. void

	•	含义：void 表示 start_kernel 函数没有返回值。
	•	作用：这表明 start_kernel 函数执行后不会向调用者返回任何值。

这是 C 语言中用于声明函数返回值类型的标准关键字。void 返回类型常见于执行一系列操作后无需返回结果的函数。

4. __init

	•	含义：__init 是一个内核特定的宏，用于标识只在初始化阶段使用的函数或数据。
	•	作用：在 Linux 内核中，__init 修饰的函数或数据会放入专门的 .init 段（section）。在系统启动完成后，这段内存会被释放，以节省内存资源。

标记为 __init 的函数或数据会在初始化完成后被内核自动丢弃，因此仅在初始化期间有效。在 start_kernel 中使用 __init 表示它是一个只在系统启动时调用的初始化函数。

5. __noreturn

	•	含义：__noreturn 是一个函数属性，用来告诉编译器该函数不会返回到调用者。
	•	作用：编译器使用 __noreturn 来优化代码和发出警告。如果一个标记了 __noreturn 的函数尝试返回，编译器会给出警告。

在内核中，__noreturn 表示函数会在执行过程中进入无限循环或在某种情况下导致系统重启、关机等操作，因此永远不会返回。

6. start_kernel(void)

	•	start_kernel：这是函数的名称，表示该函数是内核启动过程中的主入口点。start_kernel 是 Linux 内核的核心初始化函数，负责从系统启动到内核完全初始化的过渡。
	•	(void)：表示该函数不接受任何参数。

综合解释

这行代码声明了一个外部函数 start_kernel，它具有以下特性：

	•	定义在其他源文件中；
	•	参数从堆栈传递；
	•	不返回任何值；
	•	仅在内核初始化阶段有效，之后会被释放；
	•	不会返回给调用者（即它是一个无限循环或最终会导致系统不再执行任何其他代码的函数）；

整体含义

start_kernel 是内核初始化的入口函数之一，负责从引导加载程序过渡到完全初始化的操作系统内核。它在初始化完成后不再需要，因此标记为 __init。因为这个函数在启动后不会再返回，标记为 __noreturn。