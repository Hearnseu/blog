---
layout:     post                   
title:      `@synchronized`实现分析             
subtitle:    
date:       2018-03-07             
author:     Hearnseu                      
header-img: img/post-bg-2015.jpg    
catalog: true                       
tags:                              
    - iOS
    - 多线程
     
---


源代码：

```objective-c
- (void)sumer{
    
    @synchronized(_arr){
     
        [_arr addObject:@"1"];
    }
}

```
用clang 重写后输出：

```c
static void _I_Testdd_sumer(Testdd * self, SEL _cmd) {

    { id _rethrow = 0; id _sync_obj = (id)(*(NSMutableArray **)((char *)self + OBJC_IVAR_$_Testdd$_arr)); 
    
    //enter
objc_sync_enter(_sync_obj);

try {
	struct _SYNC_EXIT { _SYNC_EXIT(id arg) : sync_exit(arg) {}
	~_SYNC_EXIT() {objc_sync_exit(sync_exit);}
	id sync_exit;
	} _sync_exit(_sync_obj);


        ((void (*)(id, SEL, ObjectType))(void *)objc_msgSend)((id)(*(NSMutableArray **)((char *)self + OBJC_IVAR_$_Testdd$_arr)), sel_registerName("addObject:"), (id)(NSString *)&__NSConstantStringImpl__var_folders_5m_jf1yfr0s54x2_6rf51qlcjp00000gp_T_Testdd_043a95_mi_0);
    } catch (id e) {_rethrow = e;}
{ struct _FIN { _FIN(id reth) : rethrow(reth) {}
	~_FIN() { if (rethrow) objc_exception_throw(rethrow); }
	id rethrow;
	} _fin_force_rethow(_rethrow);}
}

}
```

简化代码为：

```c
id _rethrow = 0; 
id _sync_obj = (id)(*(NSMutableArray **)((char *)self + OBJC_IVAR_$_Testdd$_arr)); 
objc_sync_enter(_sync_obj);
try{
     objc_sync_exit(sync_exit);
}catch (id e) { _rethrow = e;}
```
`@synchronized`不仅可以实现锁，还可以输出意外。

再查看`objc_sync_enter`的文档

```

/** 
 * Begin synchronizing on 'obj'.  
 * Allocates recursive pthread_mutex associated with 'obj' if needed.
 * 
 * @param obj The object to begin synchronizing on.
 * 
 * @return OBJC_SYNC_SUCCESS once lock is acquired.  
 */
OBJC_EXPORT int
objc_sync_enter(id _Nonnull obj)
   
```

得知其使用了递归的互斥锁。
再看[这里](http://www.opensource.apple.com/source/objc4/objc4-646/runtime/objc-sync.mm)找到 objc-sync 的全部源码，部分如下：


```c

typedef struct SyncData {
    struct SyncData* nextData;
    id               object;
    int              threadCount;  // number of THREADS using this block
    recursive_mutex_t        mutex;
} SyncData;

typedef struct {
    SyncData *data;
    spinlock_t lock;

    char align[64 - sizeof (spinlock_t) - sizeof (SyncData *)];
} SyncList 

// Begin synchronizing on 'obj'. 
// Allocates recursive mutex associated with 'obj' if needed.
// Returns OBJC_SYNC_SUCCESS once lock is acquired.  
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        require_action_string(data != NULL, done, result = OBJC_SYNC_NOT_INITIALIZED, "id2data failed");
	
        result = recursive_mutex_lock(&data->mutex);
        require_noerr_string(result, done, "mutex_lock failed");
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

done: 
    return result;
}


// End synchronizing on 'obj'. 
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        require_action_string(data != NULL, done, result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR, "id2data failed");
        
        result = recursive_mutex_unlock(&data->mutex);
        require_noerr_string(result, done, "mutex_unlock failed");
    } else {
        // @synchronized(nil) does nothing
    }
	
done:
    if ( result == RECURSIVE_MUTEX_NOT_LOCKED )
         result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;

    return result;
}


```

以上可以看到加锁，解锁操作。同时Spinlock prevents multiple threads from creating multiple locks for the same new object.

