title: binder
---

### 1. 基础概述
android 中多进程的通信都会依赖Binder IPC机制
1.1 IPC原理
从进程的角度看：
{% asset_img binder_process.png binder_process %}
每个Android的进程，只能运行在自己进程所拥有的虚拟地址空间。
对应一个4GB的虚拟地址空间，其中3GB是用户空间，1GB是内核空间，当然内核空间的大小是可以通过参数配置调整的。
对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。
Client进程向Server进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，Client端与Server端进程往往采用ioctl等方法跟内核空间的驱动进行交互。
1.2 binder原理
![alt text](binder_principle.png)
可以看出无论是注册服务和获取服务的过程都需要ServiceManager，
需要注意的是此处的Service Manager是指Native层的ServiceManager（C++），并非指framework层的ServiceManager(Java)。
ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程，要掌握Binder机制，
首先需要了解系统是如何首次启动Service Manager。当Service Manager启动之后，Client端和Server端通信时都需要先获取Service Manager接口，才能开始通信服务。
注册服务(addService)：Server进程要先注册Service到ServiceManager。该过程：Server是客户端，ServiceManager是服务端。
获取服务(getService)：Client进程使用某个Service前，须先向ServiceManager中获取相应的Service。该过程：Client是客户端，ServiceManager是服务端。
使用服务：Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互。该过程：client是客户端，server是服务端。
1.3 C/S模式
BpBinder(客户端)和BBinder(服务端)都是Android中Binder通信相关的代表，它们都从IBinder类中派生而来，关系图如下：
![alt text](binder_cs.png)
client端：BpBinder.transact()来发送事务请求；
server端：BBinder.onTransact()会接收到相应事务。



### 2. 创建binder服务


数据结构
```c
typedef unsigned int __u32;
typedef unsigned long long __u64;
#ifdef BINDER_IPC_32BIT
typedef __u32 binder_size_t;
typedef __u32 binder_uintptr_t;
#else
typedef __u64 binder_size_t;
typedef __u64 binder_uintptr_t;
#endif

//native层binder通信的数据结构
struct binder_state
{
    int fd; // dev/binder的文件描述符
    void *mapped; //指向mmap的内存地址
    size_t mapsize; //分配的内存大小，默认为128KB
};


struct binder_write_read {
  binder_size_t write_size;
  binder_size_t write_consumed;
  binder_uintptr_t write_buffer;
  binder_size_t read_size;
  binder_size_t read_consumed;
  binder_uintptr_t read_buffer;
};



res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);


int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)

从read_buffer中读取binder事务数据，

//内核层binder通信的数据结构
BR_TRANSACTION

struct binder_transaction_data_secctx {
    struct binder_transaction_data transaction_data;
    binder_uintptr_t secctx; // secure context 扩展字段
};
/**
[cmd = 0x80000001]             ← 4 字节
[binder_transaction_data]     ← 56 or 64 字节（取决于架构）
 ├─ target.handle
 ├─ cookie
 ├─ code
 ├─ flags
 ├─ sender_pid / euid
 ├─ data_size / offsets_size
 └─ data.ptr.buffer / offsets
[optional: secctx]            ← 只有在 BR_TRANSACTION_SEC_CTX 才有

kernel → 用户空间
        |
        ↓
    binder_parse()
        |
        +-- 解析 ptr → txn
        |
        +-- bio_init_from_txn → msg
        |
        +-- func(bs, txn, msg, reply)
        |
        +-- 根据 flags 判断是否 reply / free buffer

**/


#read_buffer去掉binder事务数据，剩下的全是binder对象数据

struct binder_transaction_data {
  union {
    __u32 handle;
    binder_uintptr_t ptr;
  } target;
  binder_uintptr_t cookie;
  __u32 code;
  __u32 flags;
  pid_t sender_pid;
  uid_t sender_euid;
  binder_size_t data_size;
  binder_size_t offsets_size;
  union {
    struct {
      binder_uintptr_t buffer;
      binder_uintptr_t offsets;
    } ptr;
    __u8 buf[8];
  } data;
};

void bio_init_from_txn(struct binder_io *bio, struct binder_transaction_data *txn)
{
    bio->data = bio->data0 = (char *)(intptr_t)txn->data.ptr.buffer;
    bio->offs = bio->offs0 = (binder_size_t *)(intptr_t)txn->data.ptr.offsets;
    bio->data_avail = txn->data_size;
    bio->offs_avail = txn->offsets_size / sizeof(size_t);
    bio->flags = BIO_F_SHARED;
}

struct binder_io
{
    char *data;            /* pointer to read/write from */
    binder_size_t *offs;   /* array of offsets */
    size_t data_avail;     /* bytes available in data buffer */
    size_t offs_avail;     /* entries available in offsets array */

    char *data0;           /* start of data buffer */
    binder_size_t *offs0;  /* start of offsets buffer */
    uint32_t flags;
    uint32_t unused;
};


switch(txn->code)

do_find_service return si->handle

struct svcinfo
{
    struct svcinfo *next;
    uint32_t handle;
    struct binder_death death;
    int allow_isolated;
    uint32_t dumpsys_priority;
    size_t len;
    uint16_t name[0];
};



/**
内核层binder对象数据结构
**/
struct binder_object_header {
	__u32        type;
};

struct flat_binder_object {
	struct binder_object_header	hdr;
	__u32				flags;

	/* 8 bytes of data. */
	union {
		binder_uintptr_t	binder;	/* local object */
		__u32			handle;	/* remote object */
	};

	/* extra data associated with local object */
	binder_uintptr_t	cookie;
};

struct binder_node {
	int debug_id;
	spinlock_t lock;
	struct binder_work work;
	union {
		struct rb_node rb_node;
		struct hlist_node dead_node;
	};
	struct binder_proc *proc;
	struct hlist_head refs;
	int internal_strong_refs;
	int local_weak_refs;
	int local_strong_refs;
	int tmp_refs;
	binder_uintptr_t ptr;
	binder_uintptr_t cookie;
	struct {
		/*
		 * bitfield elements protected by
		 * proc inner_lock
		 */
		u8 has_strong_ref:1;
		u8 pending_strong_ref:1;
		u8 has_weak_ref:1;
		u8 pending_weak_ref:1;
	};
	struct {
		/*
		 * invariant after initialization
		 */
		u8 accept_fds:1;
		u8 txn_security_ctx:1;
		u8 min_priority;
	};
	bool has_async_transaction;
	struct list_head async_todo;
};
```


1. 
2. ProcessState::self()
```
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != nullptr) {
        return gProcess;
    }
    gProcess = new ProcessState(kDefaultDriver);
    return gProcess;
}
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(nullptr)
    , mBinderContextUserData(nullptr)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
    , mCallRestriction(CallRestriction::NONE)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        //私有的，可读不可写
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}

ProcessState::~ProcessState()
{
    if (mDriverFD >= 0) {
        if (mVMStart != MAP_FAILED) {
            munmap(mVMStart, BINDER_VM_SIZE);
        }
        close(mDriverFD);
    }
    mDriverFD = -1;
}
```

service_manager
```c
binder_write

for(::)
    binder_read();? //什么进行了阻塞？

    binder_parse();
```
一句话总结：servicemanager 是通过 ioctl(fd, BINDER_SET_CONTEXT_MGR) 向内核注册了自己的 BBinder 对象，成为了 handle 0，
### 获取binder服务


```cpp 懒汉式单例模式
extern sp<ProcessState> gProcess;

sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != nullptr) {
        return gProcess;
    }
    gProcess = new ProcessState(kDefaultDriver);
    return gProcess;
}
```
```cpp
ProcessState::ProcessState()
    : mDriverFD(open_driver(driver))
    , mVMStart(MAP_FAILED)

    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        ...
    }

```
mmap 的作用是 将 binder 驱动程序提供的一块内存映射到用户空间的虚拟地址空间中，以便用户空间进程可以通过这块内存与 binder 驱动进行通信。
```
┌────────────────────────────┐
│        App 进程            │
│  - open /dev/binder        │
│  - mmap 内存区域 A         │◄──┐
└────────────────────────────┘   │
                                 │
┌────────────────────────────┐   │
│     system_server 进程      │   │
│  - open /dev/binder         │   │
│  - mmap 内存区域 B          │◄──┤
└────────────────────────────┘   │
                                 │
┌────────────────────────────┐   │  通过 ioctl + mmap 进行通信
│     servicemanager 进程     │   │
│  - open /dev/binder         │   │
│  - mmap 内存区域 C          │◄──┘
└────────────────────────────┘

         ▲
         │
   ┌─────────────┐
   │  binder 驱动 │（内核空间）
   └─────────────┘
```


返回BpBinder

```cpp
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{

        handle_entry* e = lookupHandleLocked(handle);
            b = BpBinder::create(handle);
                        e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;

return result;
}
```
BpBinder 转为 IServiceManager 即BpBinderServiceManager
```cpp
template<typename INTERFACE>
inline sp<INTERFACE> interface_cast(const sp<IBinder>& obj)
{
    return INTERFACE::asInterface(obj);
}

//函数名展开为：
IServiceManager::asInterface(sp<IBinder>(new BpBinder(0)))
//而此函数
// 由宏DECLARE_META_INTERFACE 在 IServiceManager.h 实现

// 由宏IMPLEMENT_META_INTERFACE 在 IServiceManager.cpp实现

//在 IServiceManager.h
DECLARE_META_INTERFACE(ServiceManager)

//在 IServiceManager.cpp
IMPLEMENT_META_INTERFACE(ServiceManager, "android.os.IServiceManager");

//函数体最终展开为：
sp<IServiceManager> IServiceManager::asInterface(const sp<IBinder>& obj)
{
    return new BpServiceManager(obj);
}
```
defaultServiceManager() 返回的是 BpServiceManager，因为它通过 interface_cast<IServiceManager>() 把一个 BpBinder(0)（远端的 servicemanager 代理）转换成了一个 BpServiceManager 类型的对象。

这个 BpServiceManager 是 servicemanager 的客户端代理，它包装了所有 Binder 通信细节（比如 transact(CHECK_SERVICE)），供 App 或 System Server 调用。

每个进程如果要注册服务，都会创建自己的 BpServiceManager 实例，用来通过 Binder 向 servicemanager 发起 addService() 等调用。
注意：
	•	虽然每个进程都会 构造一个 BpServiceManager 对象，但这个对象内部都指向 同一个远程 Binder 节点（handle=0），也就是 servicemanager；
	•	所以每个进程只是创建了一个“代理对象”，而不是实际的服务管理器本体；
	•	真正提供服务注册逻辑的，是 servicemanager 进程中的 BnServiceManager::onTransact()。

## 附录