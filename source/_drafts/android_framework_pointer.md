title: android强弱引用指针
author: nuoen
tags: []
categories:
  - cpp 
  - android
  - framework
date: 2025-03-29 21:39:00

# android强弱引用指针
```
                    ┌───────────────────────────────┐
                    │ new MyObject();               │
                    │ → RefBase::init()             │
                    │ → mStrong = 0, mWeak = 0      │
                    └──────────────┬────────────────┘
                                   │
                     ┌────────────▼────────────┐
                     │ sp<MyObject> strongRef  │  ← 构造时调用 incStrong()
                     └────────────┬────────────┘
                                   │
                     +1           ▼           +1
              mStrong = 1     MyObject()     mWeak = 1  ← RefBase 中自动创建弱引用计数
                                   │
                                   ▼
                          ┌──────────────┐
                          │ wp<MyObject> │  ← 构造时调用 incWeak()
                          └──────┬───────┘
                                 │
                     +1         ▼         +0
              mStrong = 1   still alive   mWeak = 2

   ┌────────────┐
   │ clear sp<> │ → decStrong()，mStrong -1 = 0
   └────┬───────┘
        │
        ▼
  ┌──────────────┐
  │ 析构 MyObject │  ← 对象析构（~MyObject）
  └────┬─────────┘
       │
       ▼
  ┌──────────────┐
  │ promote()    │ ← wp.promote()
  └────┬─────────┘
       │
       ▼
  mStrong == 0 → 返回 nullptr
  对象已被销毁，无法再访问

最终：
  mStrong = 0
  mWeak = 1（直到 wp 也析构）

```
## RefBase
```cpp
class RefBase
{
public:
            void            incStrong(const void* id) const;
            void            decStrong(const void* id) const;
            void            forceIncStrong(const void* id) const;
            //! DEBUGGING ONLY: Get current strong ref count.
            int32_t         getStrongCount() const;

    class weakref_type
    {
    public:
        RefBase*            refBase() const;

        void                incWeak(const void* id);
        void                decWeak(const void* id);

        // acquires a strong reference if there is already one.
        bool                attemptIncStrong(const void* id);

        // acquires a weak reference if there is already one.
        // This is not always safe. see ProcessState.cpp and BpBinder.cpp
        // for proper use.
        bool                attemptIncWeak(const void* id);

        //! DEBUGGING ONLY: Get current weak ref count.
        int32_t             getWeakCount() const;

        //! DEBUGGING ONLY: Print references held on object.
        void                printRefs() const;

        //! DEBUGGING ONLY: Enable tracking for this object.
        // enable -- enable/disable tracking
        // retain -- when tracking is enable, if true, then we save a stack trace
        //           for each reference and dereference; when retain == false, we
        //           match up references and dereferences and keep only the
        //           outstanding ones.

        void                trackMe(bool enable, bool retain);
    };

            weakref_type*   createWeak(const void* id) const;
            
            weakref_type*   getWeakRefs() const;

            //! DEBUGGING ONLY: Print references held on object.
    inline  void            printRefs() const { getWeakRefs()->printRefs(); }

            //! DEBUGGING ONLY: Enable tracking of object.
    inline  void            trackMe(bool enable, bool retain)
    { 
        getWeakRefs()->trackMe(enable, retain); 
    }

    typedef RefBase basetype;

protected:
                            RefBase();
    virtual                 ~RefBase();
    
    //! Flags for extendObjectLifetime()
    enum {
        OBJECT_LIFETIME_STRONG  = 0x0000,
        OBJECT_LIFETIME_WEAK    = 0x0001,
        OBJECT_LIFETIME_MASK    = 0x0001
    };
    
            void            extendObjectLifetime(int32_t mode);
            
    //! Flags for onIncStrongAttempted()
    enum {
        FIRST_INC_STRONG = 0x0001
    };
    
    // Invoked after creation of initial strong pointer/reference.
    virtual void            onFirstRef();
    // Invoked when either the last strong reference goes away, or we need to undo
    // the effect of an unnecessary onIncStrongAttempted.
    virtual void            onLastStrongRef(const void* id);
    // Only called in OBJECT_LIFETIME_WEAK case.  Returns true if OK to promote to
    // strong reference. May have side effects if it returns true.
    // The first flags argument is always FIRST_INC_STRONG.
    // TODO: Remove initial flag argument.
    virtual bool            onIncStrongAttempted(uint32_t flags, const void* id);
    // Invoked in the OBJECT_LIFETIME_WEAK case when the last reference of either
    // kind goes away.  Unused.
    // TODO: Remove.
    virtual void            onLastWeakRef(const void* id);

private:
    friend class weakref_type;
    class weakref_impl;
    
                            RefBase(const RefBase& o);
            RefBase&        operator=(const RefBase& o);

private:
    friend class ReferenceMover;

    static void renameRefs(size_t n, const ReferenceRenamer& renamer);

    static void renameRefId(weakref_type* ref,
            const void* old_id, const void* new_id);

    static void renameRefId(RefBase* ref,
            const void* old_id, const void* new_id);

        weakref_impl* const mRefs;
};



```

## SP
sp<T> 是 Android Binder IPC 系统中的一个智能指针模板，定义在 android::sp<T> 中，类似于 C++ 标准库的 std::shared_ptr<T>，但它是 Android 自己实现的强引用指针（Smart Pointer，sp = Smart Pointer）。


举例说明 ：
frameworks/native/libs/binder/include/private/binder/Static.h 
```cpp
extern Mutex gDefaultServiceManagerLock;
extern sp<IServiceManager> gDefaultServiceManager;//附录1
#ifndef __ANDROID_VNDK__
extern sp<IPermissionController> gPermissionController;
#endif
extern bool gSystemBootCompleted;
```
```cpp
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != nullptr) return gDefaultServiceManager;

    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == nullptr) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(nullptr));
            if (gDefaultServiceManager == nullptr)
                sleep(1);
        }
    }

    return gDefaultServiceManager;
}
```

WP


## 附录
### 1.为什么将gDefaultServiceManager放在Static.h中
	•	这样你就可以在多个 .cpp 文件中 #include "Static.h"，就能用 gDefaultServiceManager，不会引起多重定义错误（Multiple Definition Error）。
	•	项目中如果有多个类似 gSomething 的共享变量，都可以统一声明在 Static.h 中，维护方便。
	•	避免在各个头文件中乱声明，形成“头文件污染”。
	•	让别人一看 Static.h 就知道项目里有哪些共享变量。