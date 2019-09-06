#章节1-1 ObjectHeader解析
## 1. 前言
从章节1中，我们得知ObjectHeader承担了大部分Object基础服务的实现，
Object仅仅只是对服务进行了转发。在这节中我们就将解析ObjectHeader，
同时也一并连带SyncBlock，SyncBlockCache，AwareLock进行解析。

##2. 先导知识

### SyncBlock介绍
    1. 每一个对象都在负Offset处有一个ObjectHeader，ObjectHeader中含有一个
    指向SyncBlock的索引，对于所有未使用SyncBlock的对象，index都是0也就意味
    着大多数对象都共享一个SyncBlock。

    2. SyncBlock目前主要负责对象的同步，然而，其也是一个备用的数据储存空间。
    比如默认的Hash算法实现就是基于SyncTableEntry的。并且暴露给COM接口的对象
    或者穿过上下文边界的对象都可以在此储存数据。

    3. SyncTableEntries和SyncBlocks都没有使用GC内存分配，且使用SyncTableEntry
    指向对象实例的弱引用保证SyncBlock和SyncTableEntry在对象结束时被回收。

    4. 索引可以在SyncTableEntries类型实例g_pSyncTable中进行查找，当SyncBlock
    不够用时，SyncTable会以2的因子扩充，并且把原先所有Entry拷贝过来。前一个表
    将一直存在，直到GC将其回收。

    5. SyncTableEntry含有一个后向指针，指向对象，一个前向指针，指向真正的SyncBlock

    6. SyncBlockArray实际上是被SyncBlockCache所管理，其负责SyncBlock的分配
    与回收。因此每次的分配或者释放都要操控EntryTable的FreeList和SyncBlock的
    Table。

    7. 原本可以直接通过index查找SyncBlock，我们引入中间层Entry多用了一个指针
    需要调用对象HashCode()函数，而其刚好是基于SyncTableEntry的。

### AwareLock介绍
    1. AwareLock是一种GC-aware锁，被抢占任务式GC尝试执行任何阻塞操作时使用。
    一旦操作结束，其会恢复GC原有状态。

    2. AwareLock仅能在SyncBlock内被创建，因为他们依赖于SyncBlock内的功能进行
    协作。

## 3. ObjectHeader解析

## 成员一览
### 字段定义
    #ifdef _WIN64
    DWORD    m_alignpad;
    #endif // _WIN64
    Volatile<DWORD> m_SyncBlockValue;

这里Volatile<>是一个模板类，使用volatile关键字进行读写，在编译期防止存取优化。
这个DWORD值用于储存SyncBlock的索引和其他值。m_alignpad用于字节对齐。

#### m_SyncBlockValue储存规则
    1. m_SyncBlockValue被分割成几组bit set。

    2. 我们在Debug下使用最高位来确保我们不会忘记使用掩码来获取Index。

    3. 前三位仅仅被用于String对象，如果第一位是1，我们就知道这个String
    对象是否有高字节字符，并且第二位会告诉你是以什么形式。需要注意的是，
    这里实际上是在利用Finalizer的标记比特位，因为String没有Finalizer。

定义：
   
    #define BIT_SBLK_STRING_HAS_NO_HIGH_CHARS   0x80000000
    #define BIT_SBLK_STRING_HIGH_CHARS_KNOWN    0x40000000
    #define BIT_SBLK_STRING_HAS_SPECIAL_SORT    0xC0000000
    #define BIT_SBLK_STRING_HIGH_CHAR_MASK      0xC0000000
    #define BIT_SBLK_FINALIZER_RUN              0x40000000
    #define BIT_SBLK_GC_RESERVE                 0x20000000

    4. 自旋锁位仅仅在我们需要修改m_SyncBlockValue里的Index时使用。当对象已经
    持有真正的SyncBlock的Index时，不应该操作此位。

定义：

    #define BIT_SBLK_SPIN_LOCK                  0x10000000
    #define BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX    0x08000000

    5. 当BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX位是0时，剩下的位将被如此安排：
        1)低10位(0-9)被轻量锁用于储存线程ID
        2)接着的6位(10-15)被用作记录轻量锁的递归次数，如果没有上锁或者仅仅被
        同一个线程锁一次，那么其值是0

定义：
   
    #define SBLK_MASK_LOCK_THREADID             0x000003FF 
    #define SBLK_MASK_LOCK_RECLEVEL             0x0000FC00
    #define SBLK_LOCK_RECLEVEL_INC              0x00000400
    #define SBLK_RECLEVEL_SHIFT                 10        

    6. 当BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX和BIT_SBLK_IS_HASHCODE同时被设置
    时，剩下的比特位(0-25)将被用作储存hashcode。

看到这里，你是否突然明白Object解析章节中计算HashCode并储存的注释了?

    //需要注意的是，现在这个算法最多使用 HASHCODE_BITS 位来计算HashCode
    //在Object被冻结的情况下，HashCode必须被搬回ObjectHeader里面，这样就
    //刚刚好

定义：

    #define BIT_SBLK_IS_HASHCODE            0x04000000
    #define HASHCODE_BITS                   26
    #define MASK_HASHCODE                   ((1<<HASHCODE_BITS)-1)
    #define SYNCBLOCKINDEX_BITS             26
    #define MASK_SYNCBLOCKINDEX             ((1<<SYNCBLOCKINDEX_BITS)-1)

### 函数定义

#### Monitor

    ObjHeader::EnterObjMonitor
    ObjHeader::EnterObjMonitorHelper
    ObjHeader::EnterObjMonitorHelperSpin
    ObjHeader::EnterSpinLock
    ObjHeader::TryEnterObjMonitor
    ObjHeader::LeaveObjMonitor
    ObjHeader::LeaveObjMonitorAtException
    ObjHeader::LeaveObjMonitorHelper
    ObjHeader::Wait
    ObjHeader::Validate
    ObjHeader::GetThreadOwningMonitorLock
    ObjHeader::Pulse
    ObjHeader::PulseAll
    ObjHeader::ReleaseSpinLock

#### SyncBlock

    ObjHeader::GetSyncBlock
    ObjHeader::GetSyncBlockIndex
    ObjHeader::GCResetIndex
    ObjHeader::HasEmptySyncBlockInfo
    ObjHeader::HasSyncBlockIndex
    ObjHeader::PassiveGetSyncBlock

#### Value设置

    ObjHeader::SetBit
    ObjHeader::SetBits
    ObjHeader::SetGCBit
    ObjHeader::ClrBit
    ObjHeader::ClrGCBit
    ObjHeader::SetIndex
    ObjHeader::ResetIndex
    ObjHeader::GetBits

#### 其他函数

    ObjHeader::GetBaseObject

## ObjectHeader函数解析

### Monitor

    void ObjHeader::EnterObjMonitor()
    {
        GetSyncBlock()->EnterMonitor();
    }

可以看到这里ObjectHeader仍然只是进行了简单的转发。故这个函数我们在之后详解SyncBlock时再解析。

再看一个复杂一点的Helper

    FORCEINLINE AwareLock::EnterHelperResult ObjHeader::EnterObjMonitorHelper(Thread* pCurThread)
    {
        //直接不使用屏障读取Value
        LONG oldValue = m_SyncBlockValue.LoadWithoutBarrier();
        //如果这四项都没有占用的话
        //没有HashCode，没有SyncBlockIndex
        //没有线程设置ThreadID，没有递归次数（无多个线程持有）
        if ((oldValue & (BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX +
                        BIT_SBLK_SPIN_LOCK +
                        SBLK_MASK_LOCK_THREADID +
                        SBLK_MASK_LOCK_RECLEVEL)) == 0)
        {
            //获取线程ID
            DWORD tid = pCurThread->GetThreadId();
            //超过线程ID最大值，返回使用SlowPath标志
            if (tid > SBLK_MASK_LOCK_THREADID)
            {
                return AwareLock::EnterHelperResult_UseSlowPath;
            }
            设置线程ID
            LONG newValue = oldValue | tid;
            //尝试使用原子操作进行值的更新
            if (InterlockedCompareExchangeAcquire((LONG*)&m_SyncBlockValue, newValue, oldValue) == oldValue)
            {
                //成功，增加锁计数
                pCurThread->IncLockCount();
                //返回Entered成功值
                return AwareLock::EnterHelperResult_Entered;
            }
            //失败，返回竞争值
            return AwareLock::EnterHelperResult_Contention;
        }
        //如果值有SyncBlockIndex或者是HashCode
        if (oldValue & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX)
        {
            //如果我们有HashCode，就创建SyncBlock
            if (oldValue & BIT_SBLK_IS_HASHCODE)
            {
                return AwareLock::EnterHelperResult_UseSlowPath;
            }
            //如果没有，就从SyncBlockTable取下SyncBlock
            SyncBlock *syncBlock = g_pSyncTable[oldValue & MASK_SYNCBLOCKINDEX].m_SyncBlock;
            //断言有效性
            _ASSERTE(syncBlock != NULL);
            //尝试使用SyncBlock的Monitor进入
            if (syncBlock->m_Monitor.TryEnterHelper(pCurThread))
            {
                return AwareLock::EnterHelperResult_Entered;
            }
            //失败返回竞争代码
            return AwareLock::EnterHelperResult_Contention;
        }
        //如果有SpinLock位占用，返回竞争值
        if (oldValue & BIT_SBLK_SPIN_LOCK)
        {
            return AwareLock::EnterHelperResult_Contention;
        }
        //如果当选线程ID不是已有ID，返回竞争值
        if (pCurThread->GetThreadId() != (DWORD)(oldValue & SBLK_MASK_LOCK_THREADID))
        {
            return AwareLock::EnterHelperResult_Contention;
        }
        //否则就是同一个线程，增加递归值
        LONG newValue = oldValue + SBLK_LOCK_RECLEVEL_INC;
        //如果递归值为0，返回使用SlowPath值
        if ((newValue & SBLK_MASK_LOCK_RECLEVEL) == 0)
        {
            return AwareLock::EnterHelperResult_UseSlowPath;
        }
        //使用原子交换更新值
        if (InterlockedCompareExchangeAcquire((LONG*)&m_SyncBlockValue, newValue, oldValue) == oldValue)
        {
            return AwareLock::EnterHelperResult_Entered;
        }
        失败返回竞争值
        return AwareLock::EnterHelperResult_UseSlowPath;
    }

接下来看Helper的自旋版本

    AwareLock::EnterHelperResult ObjHeader::EnterObjMonitorHelperSpin(Thread* pCurThread)
    {
        //注意：此函数需要在EnterObjMonitorHelper被调用之后再调用
        //如果逻辑处理器只有一个，就直接返回竞争值
        if (g_SystemInfo.dwNumberOfProcessors == 1)
        {
            return AwareLock::EnterHelperResult_Contention;
        }
        YieldProcessorNormalizationInfo normalizationInfo;
        //获取最大自旋数
        const DWORD spinCount = g_SpinConstants.dwMonitorSpinCount;
        //进行自旋循环
        for (DWORD spinIteration = 0; spinIteration < spinCount; ++spinIteration)
        {
            //使用AwareLock进行自旋等待
            AwareLock::SpinWait(normalizationInfo, spinIteration);
            //不使用屏障读取Value
            LONG oldValue = m_SyncBlockValue.LoadWithoutBarrier();
            //因为自旋已经开始，所以有可能Monitor已经切换到了AwareLock模式，所以
            //首先检查下面这个情况
            if (oldValue & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX)
            {
                //如果我们有hash code，我们需要创建一个SyncBlock
                if (oldValue & BIT_SBLK_IS_HASHCODE)
                {
                    return AwareLock::EnterHelperResult_UseSlowPath;
                }
                //否则取出
                SyncBlock *syncBlock = g_pSyncTable[oldValue & MASK_SYNCBLOCKINDEX].m_SyncBlock;
                _ASSERTE(syncBlock != NULL);
                AwareLock *awareLock = &syncBlock->m_Monitor;
                //调用AwareLock的Helper进入
                AwareLock::EnterHelperResult result = awareLock->TryEnterBeforeSpinLoopHelper(pCurThread);
                //如果不是竞争条件，就直接返回
                if (result != AwareLock::EnterHelperResult_Contention)
                {
                    return result;
                }
                //自旋迭代数自增
                ++spinIteration;
                //如果仍小于最大数
                if (spinIteration < spinCount)
                {
                    //死循环调用AwareLock自旋等待，并尝试进入
                    while (true)
                    {
                        AwareLock::SpinWait(normalizationInfo, spinIteration);
                        ++spinIteration;
                        if (spinIteration >= spinCount)
                        {  
                            //最后一次尝试自旋进入会在这个循环结束以后    
                            break;
                        }
                        result = awareLock->TryEnterInsideSpinLoopHelper(pCurThread);
                        if (result == AwareLock::EnterHelperResult_Entered)
                        {
                            return AwareLock::EnterHelperResult_Entered;
                        }
                        if (result == AwareLock::EnterHelperResult_UseSlowPath)
                        {
                            break;
                        }
                    }
                }
                //最后尝试
                if (awareLock->TryEnterAfterSpinLoopHelper(pCurThread))
                {
                    return AwareLock::EnterHelperResult_Entered;
                }
                //失败跳出
                break;
            }
            //以下代码与上面的函数一致，不再赘述
            DWORD tid = pCurThread->GetThreadId();
            if ((oldValue & (BIT_SBLK_SPIN_LOCK +
                SBLK_MASK_LOCK_THREADID +
                SBLK_MASK_LOCK_RECLEVEL)) == 0)
            {
                if (tid > SBLK_MASK_LOCK_THREADID)
                {
                    return AwareLock::EnterHelperResult_UseSlowPath;
                }
                LONG newValue = oldValue | tid;
                if (InterlockedCompareExchangeAcquire((LONG*)&m_SyncBlockValue, newValue, oldValue) == oldValue)
                {
                    pCurThread->IncLockCount();
                    return AwareLock::EnterHelperResult_Entered;
                }
                continue;
            }
            //EnterObjMonitorHelper处理了轻量锁递归的情况。
            //如果没有递归情况，后面也不会出现这种情况
            //假若EnterObjMonitorHelper提升递归等级失败了，就会选择SlowPath
            //而不是到这里来，所以没必要检查递归的情形
            //检查SpinLock情况，检查线程ID
            _ASSERTE(oldValue & BIT_SBLK_SPIN_LOCK ||
                tid != (DWORD)(oldValue & SBLK_MASK_LOCK_THREADID));
        }
        return AwareLock::EnterHelperResult_Contention;
    }

Try版本直接将服务转发，并且是唯一的非阻塞进入版本

    BOOL ObjHeader::TryEnterObjMonitor(INT32 timeOut)
    {
        return GetSyncBlock()->TryEnterMonitor(timeOut);
    }


进入自旋锁

    void ObjHeader::EnterSpinLock()
    {
         DWORD dwSwitchCount = 0;
         while(TRUE)
         {
            //不使用屏障读取Value
            LONG curValue = m_SyncBlockValue.LoadWithoutBarrier();
            //如果没有线程持有SpinLock
            if (! (curValue & BIT_SBLK_SPIN_LOCK))
            {
                //设置SpinLock
                LONG newValue = curValue | BIT_SBLK_SPIN_LOCK;
                //原子操作更新值
                LONG result = FastInterlockCompareExchange((LONG*)&m_SyncBlockValue, newValue, curValue);
                //如果更新成功，就退出
                if (result == curValue)
                    break;
            }
            //如果逻辑处理器大于一个
            if  (g_SystemInfo.dwNumberOfProcessors > 1)
            {
                //进行自旋
                for (int spinCount = 0; spinCount < BIT_SBLK_SPIN_COUNT; spinCount++)
                {  
                    //如果没有被设置，就break
                    if  (! (m_SyncBlockValue & BIT_SBLK_SPIN_LOCK))
                        break;
                    //否则将剩余时间片让出
                    YieldProcessorNormalized();
             }
             //如果已经设置，就切换到其他线程
             if  (m_SyncBlockValue & BIT_SBLK_SPIN_LOCK)
                 __SwitchToThread(0, ++dwSwitchCount);
           }
           else
                __SwitchToThread(0, ++dwSwitchCount);
         }     
    }

下面我们再继续看相应的Leave版本

    BOOL ObjHeader::LeaveObjMonitor()
    {
        //GetBaseObject反向获取对象
        OBJECTREF thisObj = ObjectToOBJECTREF (GetBaseObject ());
        for (;;)
        {
            //调用AwareLock的Helper进行离开
            AwareLock::LeaveHelperAction action = thisObj->GetHeader ()->LeaveObjMonitorHelper(GetThread());
            //根据返回值判定
            switch(action)
            {
                //如果没有任何问题，直接返回
            case AwareLock::LeaveHelperAction_None:
                return TRUE;
                //如果需要向Monitor发送信号，就取出SyncBlock进行Singal
            case AwareLock::LeaveHelperAction_Signal:
                {
                    SyncBlock *psb = thisObj->GetHeader ()->PassiveGetSyncBlock();
                    if (psb != NULL)
                    psb->QuickGetMonitor()->Signal();
                }
                return TRUE;
                //如果要求让出时间片，就让出
            case AwareLock::LeaveHelperAction_Yield:
                YieldProcessorNormalized();
                continue;
                //如果正处于竞争状态，意味着有其他线程更新了SyncBlock的
                //Value，所以在切换模式时调用GCProtect
            case AwareLock::LeaveHelperAction_Contention:
            {
                GCPROTECT_BEGIN (thisObj);
                GCX_PREEMP();
                __SwitchToThread(0, ++dwSwitchCount);
                GCPROTECT_END ();
            }
            continue;
            default:
            //其余情况必须是Error
                _ASSERTE(action == AwareLock::LeaveHelperAction_Error);
                return FALSE;
            }
        }
    }

这里可以看到实际上大部分核心工作是LeaveObjMonitorHelper在进行，因此我们下面就接着
解析它。

    FORCEINLINE AwareLock::LeaveHelperAction ObjHeader::LeaveObjMonitorHelper(Thread* pCurThread)
    {
        //仍然是不使用屏障读取Value
        DWORD syncBlockValue = m_SyncBlockValue.LoadWithoutBarrier();、
        //如果当前不持有自旋锁并且没有Hash或者储存SyncBlockIndex
            if ((syncBlockValue & (BIT_SBLK_SPIN_LOCK + BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX)) == 0)
        {
            //如果线程ID不是当前线程ID，说明此线程不拥有这个锁
            //返回Error
            if ((syncBlockValue & SBLK_MASK_LOCK_THREADID) != pCurThread->GetThreadId())
            {
                return AwareLock::LeaveHelperAction_Error;                
            }
            //如果没有递归情况的话，就Leave
            if (!(syncBlockValue & SBLK_MASK_LOCK_RECLEVEL))
            {
                //清零线程ID
                DWORD newValue = (syncBlockValue & (~SBLK_MASK_LOCK_THREADID));
                //如果原子操作失败，即有其余线程也在进行更改，就通知调用函数让出时间片
                if (InterlockedCompareExchangeRelease((LONG*)&m_SyncBlockValue, newValue, syncBlockValue) != (LONG)syncBlockValue)
                {
                    return AwareLock::LeaveHelperAction_Yield;
                }
                //成功就减少线程的锁计数
                pCurThread->DecLockCount();
            }
            else
            {
                //否则就是递归情况
                //减少递归数
                DWORD newValue = syncBlockValue - SBLK_LOCK_RECLEVEL_INC;
                //如果原子操作失败，即有其余线程也在进行更改，就通知调用函数让出时间片
                if (InterlockedCompareExchangeRelease((LONG*)&m_SyncBlockValue, newValue, syncBlockValue) != (LONG)syncBlockValue)
                {
                    return AwareLock::LeaveHelperAction_Yield;
                }
            }
            //成功返回None
            return AwareLock::LeaveHelperAction_None;
        }
        //如果没有自旋锁计数且不是HashCode，意味着拥有一个SyncBlock
        if ((syncBlockValue & (BIT_SBLK_SPIN_LOCK + BIT_SBLK_IS_HASHCODE)) == 0)
        {
            //检查前提条件
            _ASSERTE((syncBlockValue & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX) != 0);
            //获取SyncBlock并且断言检查有效性，最后转发服务
            SyncBlock *syncBlock = g_pSyncTable[syncBlockValue &MASK_SYNCBLOCKINDEX].m_SyncBlock;
             _ASSERTE(syncBlock != NULL);
            return syncBlock->m_Monitor.LeaveHelper(pCurThread);
        }
        //如果有自旋锁计数，返回竞争值
        if (syncBlockValue & BIT_SBLK_SPIN_LOCK)
        {
            return AwareLock::LeaveHelperAction_Contention;        
        }   
        //这个线程不拥有这个锁
        return AwareLock::LeaveHelperAction_Error;
    }

再看一个有差异的版本，这个版本与ObjHeader::LeaveObjMonitor的差异在于后者在
__SwitchToThread时转换到了抢占模式，也就是启用了GCProtect
    
    BOOL ObjHeader::LeaveObjMonitorAtException(){
        OBJECTREF thisObj = ObjectToOBJECTREF (GetBaseObject ());
        for (;;)
        {
            AwareLock::LeaveHelperAction action = thisObj->GetHeader ()->LeaveObjMonitorHelper(GetThread());
            switch(action)
            {
            case AwareLock::LeaveHelperAction_None:
                return TRUE;
            case AwareLock::LeaveHelperAction_Signal:
                {
                    SyncBlock *psb = thisObj->GetHeader ()->PassiveGetSyncBlock();
                    if (psb != NULL)
                    psb->QuickGetMonitor()->Signal();
                }
                return TRUE;
            case AwareLock::LeaveHelperAction_Yield:
                YieldProcessorNormalized();
                continue;
                //这个函数这里没有启用GCProtect
                //在持有自旋锁时，我们永远不触发GC.
                //所以在自旋等待时，不需要切换到抢占式模式
            case AwareLock::LeaveHelperAction_Contention:
            {
                __SwitchToThread(0, ++dwSwitchCount);
            }
            continue;
            default:
                _ASSERTE(action == AwareLock::LeaveHelperAction_Error);
                return FALSE;
            }
        }
    }

释放自旋锁是相当简单的

    DEBUG_NOINLINE void ObjHeader::ReleaseSpinLock()
    {
        //使用原子操作与
        FastInterlockAnd(&m_SyncBlockValue, ~BIT_SBLK_SPIN_LOCK);
    }

接下来再来看其他的Monitor函数

    BOOL ObjHeader::Wait(INT32 timeOut, BOOL exitContext)
    {
        //接下来的代码可能会触发GC，所以我们必须从Object里获取
        //SyncBlock以防止SyncBlock被移动
        SyncBlock *pSB = GetBaseObject()->GetSyncBlock();
        _ASSERTE(pSB != NULL);
        //检查线程是否拥有Monitor
        if (!pSB->DoesCurrentThreadOwnMonitor())
            COMPlusThrow(kSynchronizationLockException);
        //转发服务并返回结果
        BOOL result = pSB->Wait(timeOut,exitContext);
        return result;
    }   

接下来的几个函数代码都大同小异，结构都是差不多的。

    void ObjHeader::Pulse()
    {   
        SyncBlock *pSB = GetBaseObject()->GetSyncBlock();
        _ASSERTE(pSB != NULL);
        if (!pSB->DoesCurrentThreadOwnMonitor())
        COMPlusThrow(kSynchronizationLockException);
        pSB->Pulse();
    }

    void ObjHeader::PulseAll()
    {   
        SyncBlock *pSB = GetBaseObject()->GetSyncBlock();
        _ASSERTE(pSB != NULL);
        if (!pSB->DoesCurrentThreadOwnMonitor())
        COMPlusThrow(kSynchronizationLockException);
        pSB->PulseAll();
    }



Value校验函数

    BOOL ObjHeader::Validate (BOOL bVerifySyncBlkIndex)
    {
        //获取Value
        DWORD bits = GetBits ();
        Object * obj = GetBaseObject ();
        BOOL bVerifyMore = g_pConfig->GetHeapVerifyLevel() & EEConfig::HEAPVERIFY_SYNCBLK;
        //如果有High Char位被设置(也有可能是Finalizer)
        if (bits & BIT_SBLK_STRING_HIGH_CHAR_MASK)
        {
            //检查是否为一个String类型
            if (obj->GetGCSafeMethodTable () == g_pStringClass)
            {
                //如果需要检查
                if (bVerifyMore)
                {
                    //进行HighChar校验
                    ASSERT_AND_CHECK (((StringObject *)obj)->ValidateHighChars());
                }
            }
            else
            {
                //否则就是持有Finalizer，进行校验
                if (bits & BIT_SBLK_FINALIZER_RUN)
                {
                    ASSERT_AND_CHECK (obj->GetGCSafeMethodTable ()->HasFinalizer ());
                }
            }
        }
        //BIT_SBLK_GC_RESERVE标记只在GC期间进行设置，对于已经冻结的对象
        //我们不会进行清零位
        if (bits & BIT_SBLK_GC_RESERVE)
        {   
            //检查GC是否正在进行
            if (!GCHeapUtilities::IsGCInProgress () && !GCHeapUtilities::GetGCHeap()->IsConcurrentGCInProgress ())
            {
    #ifdef FEATURE_BASICFREEZE
                //如果有Freeze功能，就进行检验，看Object是否在Frozen段中
                ASSERT_AND_CHECK (GCHeapUtilities::GetGCHeap()->IsInFrozenSegment(obj));
    #else
                _ASSERTE(!"Reserve bit not cleared");
                return FALSE;
    #endif
            }
        }
        //如果有HashCode或者SyncBlockIndex
        if (bits & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX)
        {
            //如果不是HashCode
            if (!(bits & BIT_SBLK_IS_HASHCODE))
            {
                //根据检查要求，检查SyncTableEntry所持有对象是否一致
                if (bVerifySyncBlkIndex  && GCHeapUtilities::GetGCHeap()->RuntimeStructuresValid ())
                {
                    DWORD sbIndex = bits & MASK_SYNCBLOCKINDEX;
                    ASSERT_AND_CHECK(SyncTableEntry::GetSyncTableEntry()[sbIndex].m_Object == obj);             
                }
            }
            else
            {
                //如果是HashCode，就无需检验
            }
        }
        else
        {
            //如果没有BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX
            //那么剩下的就是线程ID，和递归次数
            DWORD lockThreadId = bits & SBLK_MASK_LOCK_THREADID;
            DWORD recursionLevel = (bits & SBLK_MASK_LOCK_RECLEVEL) >> SBLK_RECLEVEL_SHIFT;
            //校验不为0
            ASSERT_AND_CHECK (lockThreadId != 0 || recursionLevel == 0 );     
        }
        //校验成功
        return TRUE;
    }

获取拥有MonitorLock的线程信息

    //如果返回TRUE，就表示锁被拥有
    //ThreadID被设置为拥有锁的线程ID
    //AcquisitionCount被设置为在锁被释放前需要减少的计数
    BOOL ObjHeader::GetThreadOwningMonitorLock(DWORD *pThreadId, DWORD *pAcquisitionCount)
    {
        DWORD bits = GetBits();
        //检查HashCode或者SyncBlock位
        if (bits & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX)
        {
            //如果是HashCode，说明没有线程拥有
            if (bits & BIT_SBLK_IS_HASHCODE)
            {
                *pThreadId = 0;
                *pAcquisitionCount = 0;
                return FALSE;
            }
            else
            {
                //否则取出SyncBlock
                DWORD index = bits & MASK_SYNCBLOCKINDEX;
                SyncBlock* psb = g_pSyncTable[(int)index].m_SyncBlock;
                _ASSERTE(psb->GetMonitor() != NULL);
                //获取持有线程
                Thread* pThread = psb->GetMonitor()->GetHoldingThread();
                //如果有，就返回
                if(pThread == NULL)
                {
                    *pThreadId = 0;
                    *pAcquisitionCount = 0;
                    return FALSE;
                }
                else
                {
                    *pThreadId = pThread->GetThreadId();
                    *pAcquisitionCount = psb->GetMonitor()->GetRecursionLevel();
                    return TRUE;
                }
            }
        }
        else
        {
            //如果我们持有轻量锁
            //获取线程ID和递归数
            DWORD lockThreadId, recursionLevel;
            lockThreadId = bits & SBLK_MASK_LOCK_THREADID;
            recursionLevel = (bits & SBLK_MASK_LOCK_RECLEVEL) >> SBLK_RECLEVEL_SHIFT;
            //如果ID是0，那么递归数也必定是0
            //但是ID不必是有效的，因为锁可能是孤立的(未被线程持有)
            _ASSERTE (lockThreadId != 0 || recursionLevel == 0 );
            if(lockThreadId != 0)
            {
                //如果ID不为0且递归数为0
                //那么仅有一个线程拥有了一次，所以返回递归数+1
                *pAcquisitionCount = recursionLevel + 1;
                return TRUE;
            }
            else
            {
                //否则返回FALSE
                *pAcquisitionCount = 0;
                return FALSE;
            }
        }
    }

Monitor部分就结束了，趁着下一部分还没开始，喝杯咖啡吧。

### SyncBlock

第一个函数就是我们如何获取SyncBlock

    SyncBlock *ObjHeader::GetSyncBlock()
    {
        //先尝试惰性获取SyncBlock
        PTR_SyncBlock syncBlock = GetBaseObject()->PassiveGetSyncBlock();
        DWORD      indx = 0;
        BOOL indexHeld = FALSE;
        //如果SyncBlock不为空
        if (syncBlock)
        {
    #ifdef _DEBUG 
            //debug下检查GC是否在我们发出请求后将SyncTable相应Entry
            //的后向指针设置到本对象
            PTR_SyncTableEntry pEntries(SyncTableEntry::GetSyncTableEntry());
            _ASSERTE(pEntries[GetHeaderSyncBlockIndex()].m_Object == GetBaseObject());
    #endif
            //没有问题返回SyncBlock
            RETURN syncBlock;
        }
        {
            //否则我们就需要从Cache中取出一个SyncBlock
            //将Cache上锁
            SyncBlockCache::LockHolder lh(SyncBlockCache::GetSyncBlockCache());
            //真正执行前再尝试一次
            syncBlock = GetBaseObject()->PassiveGetSyncBlock();
            if (syncBlock)
                RETURN syncBlock;
            //获取SyncBlock缓存所持有的内存
            SyncBlockMemoryHolder syncBlockMemoryHolder(SyncBlockCache::GetSyncBlockCache()->GetNextFreeSyncBlock());
            syncBlock = syncBlockMemoryHolder;
            if ((indx = GetHeaderSyncBlockIndex()) == 0)
            {
                //如果获取index为0，表示还没有持有有效索引，取出一个
                indx = SyncBlockCache::GetSyncBlockCache()->NewSyncBlockSlot(GetBaseObject());
            }
            else
            {
                //已经有了，就进行标记
                indexHeld = TRUE;
            }
            {
                //需要注意的是，NewSyncBlockSlot会带来我们无法处理的副作用
                //因此其必须作为我们最后万不得已的手段
                //释放请求
                syncBlockMemoryHolder.SuppressRelease();
                //使用Placement new在内存上构造SyncBlock
                new (syncBlock) SyncBlock(indx);
                {
                    //对操作上锁
                    ENTER_SPIN_LOCK(this);
                    //如果ObjectHeader的轻量锁正在使用，将信息转到SyncBlock中
                    DWORD bits = GetBits();
                    //如果没有HashCode或者SyncBlockIndex
                    if ((bits & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX) == 0)
                    {
                        //取出线程ID和递归层级
                        DWORD lockThreadId = bits & SBLK_MASK_LOCK_THREADID;
                        DWORD recursionLevel = (bits & SBLK_MASK_LOCK_RECLEVEL) >> SBLK_RECLEVEL_SHIFT;
                        if (lockThreadId != 0 || recursionLevel != 0)
                        {
                            //若ID为0，那么递归层级不可能是0
                            _ASSERTE(lockThreadId != 0);
                            //获取线程对象
                            Thread *pThread = g_pThinLockThreadIdDispenser->IdToThreadWithValidation(lockThreadId);
                            //如果锁是孤立的，没有线程持有
                            if (pThread == NULL)
                            {
                                // The lock is orphaned.
                                pThread = (Thread*) -1;
                            }
                            //进行SyncBlock的初始化工作
                            syncBlock->InitState(recursionLevel + 1, pThread);                        
                        }                 
                    }
                    else if ((bits & BIT_SBLK_IS_HASHCODE) != 0)
                    {
                        //如果有HashCode
                        //设置HashCode
                        DWORD hashCode = bits & MASK_HASHCODE;
                        syncBlock->SetHashCode(hashCode);
                    }
                }
                //设置Entry的SyncBlock
                SyncTableEntry::GetSyncTableEntry() [indx].m_SyncBlock = syncBlock;
                //为了避免其他线程尝试在我们已经激活了AppDomain索引(SYNCBLKINDEX)后获取索引
                //而造成的竞争，我们需要保证SyncBlock和AppDomain Index一同在
                //替换Index之前完成设置
                if (GetHeaderSyncBlockIndex() == 0)
                {
                    //我们已经将Appdomain转移到以上的SyncBlock里面了
                    SetIndex(BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX | indx);
                }
                //如果我们已经有了Index，就将其设定与对象同时死亡
                if (indexHeld)
                    syncBlock->SetPrecious();
                //释放自旋锁
                LEAVE_SPIN_LOCK(this);
            }
        }
        RETURN syncBlock;
    }

获取SyncBlock索引

    DWORD ObjHeader::GetSyncBlockIndex()
    {
        DWORD   indx;
        //发现没有SyncBlockIndex，进行创建
        if ((indx = GetHeaderSyncBlockIndex()) == 0)
        {
            BOOL fMustCreateSyncBlock = FALSE;
            {
                 //将Cache上锁
                SyncBlockCache::LockHolder lh(SyncBlockCache::GetSyncBlockCache());
                //再试一次
                if (GetHeaderSyncBlockIndex() == 0)
                {
                    //上自旋锁
                    ENTER_SPIN_LOCK(this);
                    DWORD bits = GetBits();
                    //检查是否有HashCode AppDomain Index或者锁信息存在里面
                    if (((bits & (BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX | BIT_SBLK_IS_HASHCODE)) == (BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX | BIT_SBLK_IS_HASHCODE)) ||
                    ((bits & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX) == 0))
                    {
                        //需要创建一个SyncBlock储存这个Index
                        fMustCreateSyncBlock = TRUE;
                    }
                    else
                    {
                        //否则将其储存在Value里
                        SetIndex(BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX | SyncBlockCache::GetSyncBlockCache()->NewSyncBlockSlot(GetBaseObject()));
                    }
                    LEAVE_SPIN_LOCK(this);
                }
            } 
            //需要创建SyncBlock
            if (fMustCreateSyncBlock)
                GetSyncBlock();
            //如果仍然创建失败，抛出异常
            if ((indx = GetHeaderSyncBlockIndex()) == 0)
                COMPlusThrowOM();
        }
        return indx;
    }

    //本函数仅由GC调用
    void GCResetIndex()    
    {
        m_SyncBlockValue.RawValue() &=~(BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX | BIT_SBLK_IS_HASHCODE | MASK_SYNCBLOCKINDEX);
    }

上面这个函数没有使用任何Barrier或者原子交换正是因为仅由GC调用。
下面这个函数也相对很简单，仅用于DEBUG。

    BOOL HasEmptySyncBlockInfo()
    {
        return m_SyncBlockValue.LoadWithoutBarrier() == 0;
    }

下面这个函数需要注意的是它检查是否有一个真正的SyncBlockIndex。比如有些对象在SyncTable
里有一个Entry，却不见得在SyncBlockCache里面有Entry。

    BOOL HasSyncBlockIndex()
    {
        return (GetHeaderSyncBlockIndex() != 0);
    }

    PTR_SyncBlock PassiveGetSyncBlock()
    {
        return g_pSyncTable [(int)GetHeaderSyncBlockIndex()].m_SyncBlock;    
    }

这个函数是惰性获取SyncBlock，但不保证立即有真正的SyncBlock分配。

#### Value操作

Value操作不打算在这里讲解，因为它们都是利用原子操作FastInterlockedOperation来进行的。
实现也相当简单。

#### 其他函数

从ObjectHeader偏移获取Object地址

    PTR_Object GetBaseObject()
    {
        return dac_cast<PTR_Object>(dac_cast<TADDR>(this + 1));
    }

到这里基本上ObjectHeader的解析就基本结束了，下面我们接着进行SyncBlock的解析。

## 4. SyncBlock解析

## 成员一览

### 字段定义

    AwareLock  m_Monitor;
    PTR_InteropSyncBlockInfo    m_pInteropInfo;
    SLink       m_Link;
    DWORD m_dwHashCode;
    WCHAR m_BSTRTrailByte;

### 字段详解

#### m_Monitor
    真正的Monitor

#### m_pInteropInfo
    这个对象存留额外的信息，并且被设定为Public暴露给Native代码。

#### m_Link
    我们在这里使用一个链表穿插两个链表，当SyncBlock处于活动状态时，这个链表
    被用作储存等待线程。
    当SyncBlock被回收之后，SyncBlockCache在这里维护一个SyncBlock的空闲表。
    这里我们没有使用SLink<>是为了把内存开销降到最低。(这是一个非泛型版本)

#### m_dwHashCode
    这个字段被用作留存对象的HashCode。它可以从ObjectHeader中被转入，在Header
    中被限制只有26位，或者直接生成并以32位形式储存在这里。HashCode若为0则表示
    HashCode没有被设置，这省掉了一个用于表示此状态的Flag空间。

#### m_BSTRTrailByte
    在早期的VB.NET版本中，没有Array的概念，所以开发者使用BSTR来用作Array。
    这种表示的方法是在BSTR的结尾加上一个标记Byte。
    为了支持这种情况，我们需要使用SyncBlock来完成这个工作。
    这个字段就用作储存标记Byte。

### 函数定义

#### Monitor
    SyncBlock::DoesCurrentThreadOwnMonitor
    SyncBlock::EnterMonitor
    SyncBlock::GetMonitor
    SyncBlock::QuickGetMonitor 
    SyncBlock::GetPtrForLockContract
    SyncBlock::LeaveMonitor 
    SyncBlock::LeaveMonitorCompletely 
    SyncBlock::Pulse 
    SyncBlock::PulseAll 
    SyncBlock::TryEnterMonitor 
    SyncBlock::Wait
    SyncBlock::IsIDisposable 

#### HashCode
    SyncBlock::GetHashCode
    SyncBlock::SetHashCode 

#### Interop
    SyncBlock::GetInteropInfo
    SyncBlock::GetInteropInfoNoCreate
    SyncBlock::SetInteropInfo

#### SyncBlock
    SyncBlock::InitState
    SyncBlock::GetSyncBlockIndex
    SyncBlock::IsPrecious 
    SyncBlock::SetPrecious 

#### TrailByte
    SyncBlock::GetCOMBstrTrailByte
    SyncBlock::HasCOMBstrTrailByte
    SyncBlock::SetCOMBstrTrailByte 

#### Enc(EditAndContinue)
    SyncBlock::SetEnCInfo 
    SyncBlock::GetEnCInfo


### 函数解析

#### Monitor
还是从我们的老朋友Monitor开始。这次的逻辑就相当简单了，仅仅是做了转发。

    void EnterMonitor()
    {
        m_Monitor.Enter();
    }

    BOOL TryEnterMonitor(INT32 timeOut = 0)
    {
        return m_Monitor.TryEnter(timeOut);
    }

    AwareLock* GetMonitor()
    {
        //设置持有SyncBlock
        SetPrecious();
        //需要注意的是DAC不会返回PTR_这种类型.这个指针属于内部指针
        //并且SyncBlock已经被封送了，所以GetMonitor可以被调用。
        return &m_Monitor;
    }

    AwareLock* QuickGetMonitor()
    {
        //这个方法没有调用 SetPrecious()来保证SyncBlock的生命周期
        //因此需要格外小心
        return &m_Monitor;
    }

对应Leave版本也很简单

    BOOL LeaveMonitor()
    {
        return m_Monitor.Leave();
    }

    LONG LeaveMonitorCompletely()
    {
        return m_Monitor.LeaveCompletely();
    }

再来看看Pulse是如何实现的

    void SyncBlock::Pulse()
    {
         WaitEventLink  *pWaitEventLink;
         //在线程队列里(m_link)出队列，并且唤醒一个线程
         if ((pWaitEventLink = ThreadQueue::DequeueThread(this)) != NULL)
            pWaitEventLink->m_EventWait->Set();
    }

PulseAll同上，只不过是一直Dequeue

    void SyncBlock::PulseAll()
    {
        WaitEventLink  *pWaitEventLink;
        while ((pWaitEventLink = ThreadQueue::DequeueThread(this)) != NULL)
            pWaitEventLink->m_EventWait->Set();
    }

在这里我们不再深入讨论CLREvent，其实现与EE(执行引擎)有关，我们有机会放在后面再解析。

接下来看一个实现略微复杂的Wait

    //我们为SyncBlock::Wait维护两个队列
    //一个是SyncBlock内部的link，当我们执行Pulse时，我们从队列里拿出一个线程
    //另一个是Thread::m_WaitEventLink。当我们Pulse时，我们首先找到队列里的
    //Event并进行设置，同时也在SyncBlock的值里设置一个从队列得到的比特位
    //以便我们可以在已经Pulse SyncBlock后再次调用时立即返回
    BOOL SyncBlock::Wait(INT32 timeOut, BOOL exitContext)
    {
        Thread  *pCurThread = GetThread();
        BOOL     isTimedOut = FALSE;
        BOOL     isEnqueued = FALSE;
        WaitEventLink waitEventLink;
        WaitEventLink *pWaitEventLink;
        //一旦我们打开了这个开关，我们就与GC有竞争了，GC可能在
        //这个过程中更改SyncBlock除非我们上报对象
        _ASSERTE(pCurThread->PreemptiveGCDisabled());
        WaitEventLink *walk = pCurThread->WaitEventLinkForSyncBlock(this);
        //检查这个线程是否已经在等待SyncBlock了
        //如果是
        if (walk->m_Next)
        {
            //在同一个锁上再次等待
            if (walk->m_Next->m_WaitSB == this) {
                walk->m_Next->m_RefCount ++;
                pWaitEventLink = walk->m_Next;
            }
            else if ((SyncBlock*)(((DWORD_PTR)walk->m_Next->m_WaitSB) & ~1)== this) {
                //检查bit位得知已经Pulse过，直接返回
                return TRUE;
            }
        }
        else
        {
            //否则这就是此线程第一次等待这个SyncBlock
            CLREvent* hEvent;
            //取一个Event
            if (pCurThread->m_WaitEventLink.m_Next == NULL) {
                hEvent = &(pCurThread->m_EventWait);
            }
            else {
                hEvent = GetEventFromEventStore();
            }
            //设置WaitEventLink节点
            waitEventLink.m_WaitSB = this;
            waitEventLink.m_EventWait = hEvent;
            waitEventLink.m_Thread = pCurThread;
            waitEventLink.m_Next = NULL;
            waitEventLink.m_LinkSB.m_pNext = NULL;
            waitEventLink.m_RefCount = 1;
            pWaitEventLink = &waitEventLink;
            walk->m_Next = pWaitEventLink;
            //在Enqueue之前，重设会将我们唤醒的Event
            hEvent->Reset();
            //现在这个线程就在此SyncBlock上等待了
            ThreadQueue::EnqueueThread(pWaitEventLink, this);
            isEnqueued = TRUE;
        }
        //断言第二种情况的设置
        _ASSERTE ((SyncBlock*)((DWORD_PTR)walk->m_Next->m_WaitSB & ~1)== this);
        //PendingSync被用作线程抛出InterruptedException时储存正确的线程状态。
        PendingSync   syncState(walk);
        OBJECTREF     obj = m_Monitor.GetOwningObject();
        //进行同步控制，短暂上升持有度
        m_Monitor.IncrementTransientPrecious();
        //开启GCProtect
        GCPROTECT_BEGIN(obj);
        {
            GCX_PREEMP();
            syncState.m_EnterCount = LeaveMonitorCompletely();
            _ASSERTE(syncState.m_EnterCount > 0);
            //进行阻塞
            isTimedOut = pCurThread->Block(timeOut, &syncState);
        }
        GCPROTECT_END();
        //下降
        m_Monitor.DecrementTransientPrecious();
        //返回等待是否成功
        return !isTimedOut;
    }

下面是判断SyncBlock是否可以回收，逻辑较为简单

    BOOL IsIDisposable()
    {
        return (!IsPrecious() &&
                m_Monitor.IsUnlockedWithNoWaiters() &&
                m_Monitor.m_TransientPrecious == 0);
    }

是否被当前线程持有

    BOOL DoesCurrentThreadOwnMonitor()
    {
        return m_Monitor.OwnedByCurrentThread();
    }

LOCK_TAKEN/RELEASED 宏需要一个指向对象的指针来判断到底是已经取得还是释放。此函数为protected

    void * GetPtrForLockContract()
    {
        return m_Monitor.GetPtrForLockContract();
    }

#### HashCode

GetHashCode简单返回，不再讲解。
我们来看SetHashCode
    
    DWORD SetHashCode(DWORD hashCode)
    {
        //使用原子交换，当HashCode为0时才进行设置
        DWORD result = FastInterlockCompareExchange((LONG*)&m_dwHashCode, hashCode, 0);
        if (result == 0)
        {
            //如果SyncBlock已经有HashCode，我们就直接返回
             SetPrecious();
             return hashCode;
        }
        else//否则就返回先前的HashCode
            return result;
    }

#### Interop

下面就看一下交互对象部分

    InteropSyncBlockInfo* GetInteropInfo()
    {
        //检查是否已经存在
        if (!m_pInteropInfo)
        {
            NewHolder<InteropSyncBlockInfo> pInteropInfo;
            //从缓存中取出
            pInteropInfo = (InteropSyncBlockInfo *)InterlockedPopEntrySList(&InteropSyncBlockInfo::s_InteropInfoStandbyList);
            //如果有缓存，在内存上构造对象，否则新构建一个
            if (pInteropInfo != NULL)
            {
                new (pInteropInfo) InteropSyncBlockInfo();
            }
            else
            {
                pInteropInfo = new InteropSyncBlockInfo();
            }
            if (SetInteropInfo(pInteropInfo))
                pInteropInfo.SuppressRelease();
        }
        //返回交互对象
        RETURN m_pInteropInfo;
    }

接下来两个函数就更加简单了

    PTR_InteropSyncBlockInfo GetInteropInfoNoCreate()
    {
        RETURN m_pInteropInfo;
    }

使用原子交换设定指针

    bool SyncBlock::SetInteropInfo(InteropSyncBlockInfo* pInteropInfo)
    {
        SetPrecious();
        return (FastInterlockCompareExchangePointer(&m_pInteropInfo,
                                                pInteropInfo,
                                                NULL) == NULL);
    }

#### SyncBlock

再看SyncBlock自身服务部分

    void InitState(ULONG recursionLevel, PTR_Thread holdingThread)
    {
        m_Monitor.InitializeToLockedWithNoWaiters(recursionLevel, holdingThread);
    }

这里也是直接调用了AwareLock的Initialize函数。

    DWORD GetSyncBlockIndex()
    {
        return m_Monitor.GetSyncBlockIndex();
    }

获取SyncBlockIndex也是借由AwareLock完成

    BOOL IsPrecious()
    {
        return (m_Monitor.m_dwSyncIndex & SyncBlockPrecious) != 0;
    }

这里Precious(意味着依附于对象的生命周期的标志)，也是通过AwareLock的SyncIndex中一位进行标记。

    void SetPrecious()
    {
        m_Monitor.SetPrecious();
    }

标记Precious也是通过AwareLock进行。

#### TrailByte

TrailByte相关是早期VB.NET的遗留产物，实现也比较简单，这里就不再讲解。

#### Enc

Enc是CLR的一项技术，全称叫Edit and Continue，这两个函数实现都相当简单，但是对Enc的解析暂时不在本Blog计划内。

## 5. SyncBlockCache解析

## 成员一览

### 字段定义
    PTR_SLink   m_pCleanupBlockList
    SLink*      m_FreeBlockList
    Crst        m_CacheLock
    DWORD       m_FreeCount
    DWORD       m_ActiveCount
    SyncBlockArray *m_SyncBlocks
    DWORD       m_FreeSyncBlock
    DWORD       m_FreeSyncTableIndex
    size_t      m_FreeSyncTableList
    DWORD       m_SyncTableSize
    SyncTableEntry *m_OldSyncTables
    BOOL        m_bSyncBlockCleanupInProgress
    DWORD*      m_EphemeralBitmap

### 字段详解

#### m_pCleanupBlockList
    需要清理的SyncBlock列表
#### m_FreeBlockList
    可用SyncBlock列表
#### m_CacheLock
    缓存锁
#### m_FreeCount
    可用SyncBlock计数
#### m_ActiveCount
    活跃SyncBlock计数
#### m_SyncBlocks
    SyncBlock数组列表
#### m_FreeSyncBlock
    在数组里下一个可用的SyncBlock

以下变量处理SyncTableEntries

#### m_FreeSyncTableIndex
    我们分配大量SyncTableEntry并存于数组中，这个索引指示从未用过和已经用过的SyncTableEntry的边界
#### m_FreeSyncTableList
    第一个在FreeList可用SyncTableEntry的索引
#### m_SyncTableSize
    SyncTable大小
#### m_OldSyncTables
    指向下一个已经不用的SyncTable
#### m_bSyncBlockCleanupInProgress
    表明SyncBlock的清理是否正在进行
#### m_EphemeralBitmap
    用于暂时性扫描的位图

### 函数定义

#### 分配-回收
    SyncBlockCache::Start 
    SyncBlockCache::Stop 
    SyncBlockCache::Grow 
    SyncBlockCache::GetNextFreeSyncBlock 
    SyncBlockCache::GetNextCleanupSyncBlock
    SyncBlockCache::InsertCleanupSyncBlock 
    SyncBlockCache::NewSyncBlockSlot
    SyncBlockCache::DeleteSyncBlock 
    SyncBlockCache::DeleteSyncBlockMemory 
    SyncBlockCache::CleanupSyncBlocks 
#### GC相关
    SyncBlockCache::GCDeleteSyncBlock 
    SyncBlockCache::GCDone 
    SyncBlockCache::GCWeakPtrScan 
    SyncBlockCache::GCWeakPtrScanElement
#### 信息获取
    SyncBlockCache::GetActiveCount  
    SyncBlockCache::GetTableEntryCount 
    SyncBlockCache::IsSyncBlockCleanupInProgress 
    SyncBlockCache::GetSyncBlockCache 
#### 其他
    SyncBlockCache::CardSetP 
    SyncBlockCache::CardTableSetBit  
    SyncBlockCache::ClearCard  
    SyncBlockCache::SetCard 
    SyncBlockCache::VerifySyncTableEntry

### 函数解析

直接让我们从Start开始

#### 分配-释放
    void SyncBlockCache::Start()
    {
        //初始化位图并置为0
        DWORD* bm = new DWORD [BitMapSize(SYNC_TABLE_INITIAL_SIZE+1)];
        memset (bm, 0, BitMapSize (SYNC_TABLE_INITIAL_SIZE+1)*sizeof(DWORD));
        //设置SyncTableEntry表
        SyncTableEntry::GetSyncTableEntryByRef() = new SyncTableEntry[SYNC_TABLE_INITIAL_SIZE+1];
        //把第一个Entry的SyncBlock指向变为null
        SyncTableEntry::GetSyncTableEntry()[0].m_SyncBlock = 0;
        //在全局对象上进行构造
        SyncBlockCache::GetSyncBlockCache() = new (&g_SyncBlockCacheInstance) SyncBlockCache;
        //设置位图
        SyncBlockCache::GetSyncBlockCache()->m_EphemeralBitmap = bm;
        //初始化链表头结点
        InitializeSListHead(&InteropSyncBlockInfo::s_InteropInfoStandbyList);
    }

    void SyncBlockCache::Stop()
    {
        //Cache必须被首先回收，因为其可以遍历整个表
        //找到所有存活的SyncBlock并且因此必须把他们的Critical Section回收掉
        if (SyncBlockCache::GetSyncBlockCache())
        {
            delete SyncBlockCache::GetSyncBlockCache();
            SyncBlockCache::GetSyncBlockCache() = 0;
        }
        if (SyncTableEntry::GetSyncTableEntry())
        {
            delete SyncTableEntry::GetSyncTableEntry();
            SyncTableEntry::GetSyncTableEntryByRef() = 0;
        }
    }

下面看看增长过程中发生了什么

    void SyncBlockCache::Grow()
    {
        NewArrayHolder<SyncTableEntry> newSyncTable (NULL);
        NewArrayHolder<DWORD>          newBitMap    (NULL);
        DWORD *                        oldBitMap;
        //计算新SyncTable的大小，通常情况下我们直接加倍
        //除非这样做会导致Slot的索引值过大而无法被放入掩码内
        //如果是这样的话，调用者可就遭殃了
        DWORD newSyncTableSize;
        if (m_SyncTableSize <= (MASK_SYNCBLOCKINDEX >> 1))
        {
            newSyncTableSize = m_SyncTableSize * 2;
        }
        else
        {
            newSyncTableSize = MASK_SYNCBLOCKINDEX;
        }
        //保证有增长空间
        if (!(newSyncTableSize > m_SyncTableSize))
        {
            COMPlusThrowOM();
        }
        //分配新内存
        newSyncTable = new SyncTableEntry[newSyncTableSize];
        newBitMap = new DWORD[BitMapSize (newSyncTableSize)];
        {
            //从这里开始，我们假定我们会成功并且造成全局的影响
            //任意可失败的操作必须在这之前出现
            CANNOTTHROWCOMPLUSEXCEPTION();
            FAULT_FORBID();
            newSyncTable.SuppressRelease();
            newBitMap.SuppressRelease();
            //我们把老的表串起来，因为我们不能在所有线程停止(GC时)前删除掉他们
            SyncTableEntry::GetSyncTableEntry() [0].m_Object = (Object *)m_OldSyncTables;
            m_OldSyncTables = SyncTableEntry::GetSyncTableEntry();
            //清空内存
            memset (newSyncTable, 0, newSyncTableSize*sizeof (SyncTableEntry));
            memset (newBitMap, 0, BitMapSize (newSyncTableSize)*sizeof (DWORD));
            //拷贝内存
            CopyMemory (newSyncTable, SyncTableEntry::GetSyncTableEntry(),
                    m_SyncTableSize*sizeof (SyncTableEntry));
            CopyMemory (newBitMap, m_EphemeralBitmap,
                    BitMapSize (m_SyncTableSize)*sizeof (DWORD));
            oldBitMap = m_EphemeralBitmap;
            m_EphemeralBitmap = newBitMap;
            //删除老位图
            delete[] oldBitMap;
            _ASSERTE((m_SyncTableSize & MASK_SYNCBLOCKINDEX) == m_SyncTableSize);
            //我们不关心其他线程是否看得见新大小
            //但是我们真的不想另一个在看见新数组前就看见大小
            //TODO: 如果两个线程同时来到这里，我们仍然会导致内存泄漏么?
            FastInterlockExchangePointer(&SyncTableEntry::GetSyncTableEntryByRef(), newSyncTable.GetValue());
            //自增可用SyncTable索引
            m_FreeSyncTableIndex++;
            m_SyncTableSize = newSyncTableSize;
        }
    }

    //调用此函数前必须上Cache锁
    SyncBlock *SyncBlockCache::GetNextFreeSyncBlock()
    {
        SyncBlock       *psb;
        SLink           *plst = m_FreeBlockList;
        m_ActiveCount++;
        if (plst)
        {
            //如果plst不为null，指向下一个块
            m_FreeBlockList = m_FreeBlockList->m_pNext;
            m_FreeCount--;
            //获取实际的SyncBlock指针
            psb = (SyncBlock *) (((BYTE *) plst) - offsetof(SyncBlock, m_Link));
            return psb;
        }
        else
        {
            //如果没有了
            //如果为空或者超出了最大SyncBlock索引
            if ((m_SyncBlocks == NULL) || (m_FreeSyncBlock >= MAXSYNCBLOCK))
            {
                //分配一个新Array
                SyncBlockArray* newsyncblocks = new(SyncBlockArray);
                //若分配失败，抛出异常
                if (!newsyncblocks)
                    COMPlusThrowOM ();
                newsyncblocks->m_Next = m_SyncBlocks;
                m_SyncBlocks = newsyncblocks;
                m_FreeSyncBlock = 0;
            }
            //否则根据索引获取下一个可用SyncBlock
            return &(((SyncBlock*)m_SyncBlocks->m_Blocks)[m_FreeSyncBlock++]);
        }
    }

所以整体策略是，先从FreeList里面拿，拿不到再从原始的数组中获取

    SyncBlock* SyncBlockCache::GetNextCleanupSyncBlock()
    {
        //这里不需要锁，因为目前只由Finalizer线程调用
        SyncBlock       *psb = NULL;
        if (m_pCleanupBlockList)
        {
            psb = (SyncBlock *) (((BYTE *) m_pCleanupBlockList) - offsetof(SyncBlock, m_Link));
            m_pCleanupBlockList = m_pCleanupBlockList->m_pNext;
        }
        return psb;
    }

    void SyncBlockCache::InsertCleanupSyncBlock(SyncBlock* psb)
    {
        //在我们下手前清空所有等待的线程
        if (psb->m_Link.m_pNext != NULL)
        {
            while (ThreadQueue::DequeueThread(psb) != NULL)
                continue;
        }
        //我们不需要锁，因为这个函数仅由GC调用
        psb->m_Link.m_pNext = m_pCleanupBlockList;
        m_pCleanupBlockList = &psb->m_Link;
    }

    DWORD SyncBlockCache::NewSyncBlockSlot(Object *obj){
        //GetSyncBlock持有锁，确保不是其他线程干的
        _ASSERTE(m_CacheLock.OwnedByCurrentThread());
        DWORD indexNewEntry;
        if (m_FreeSyncTableList)
        {
            //如果索引不为0
            //新索引就是原有的一半
            indexNewEntry = (DWORD)(m_FreeSyncTableList >> 1);
            //确保第一个Entry已经死亡
            _ASSERTE ((size_t)SyncTableEntry::GetSyncTableEntry()[indexNewEntry].m_Object.Load() & 1);
            //设置新SyncTableList
            m_FreeSyncTableList = (size_t)SyncTableEntry::GetSyncTableEntry()[indexNewEntry].m_Object.Load() & ~1;
        }
        else if ((indexNewEntry = (DWORD)(m_FreeSyncTableIndex)) >= m_SyncTableSize)
        {
            //如果新索引大于m_SyncTableSize
            //就进行增长
            Grow();
        }
        CardTableSetBit (indexNewEntry);
        //清空并设置相应Entry
        _ASSERTE(SyncTableEntry::GetSyncTableEntry() [indexNewEntry].m_SyncBlock == NULL);
        SyncTableEntry::GetSyncTableEntry() [indexNewEntry].m_SyncBlock = NULL;
        SyncTableEntry::GetSyncTableEntry() [indexNewEntry].m_Object = obj;
        _ASSERTE(indexNewEntry != 0);
        return indexNewEntry;
    }

再来看看删除逻辑

    void SyncBlockCache::DeleteSyncBlock(SyncBlock *psb)
    {
        //需要清空COM数据
        if (psb->m_pInteropInfo)
        {
            CleanupSyncBlockComData(psb->m_pInteropInfo);
            //如果EE已经关闭
            if (g_fEEShutDown)
            {
                //回收内存
                delete psb->m_pInteropInfo;
            }
            else
            {
                //否则调用析构函数
                psb->m_pInteropInfo->~InteropSyncBlockInfo();
                //把对象放回StandByList里面
                InterlockedPushEntrySList(&InteropSyncBlockInfo::s_InteropInfoStandbyList, (PSLIST_ENTRY)psb->m_pInteropInfo);
            }
        }
        //删除对象，但不回收内存(这是由SyncBlock的重载delete决定)
        delete psb;
        {
            SyncBlockCache::LockHolder lh(this);
            DeleteSyncBlockMemory(psb);
        }
    }

DeleteMemory也较为简单

    void SyncBlockCache::DeleteSyncBlockMemory(SyncBlock *psb)
    {
        //此函数调用前必须持有缓存锁
        m_ActiveCount--;
        m_FreeCount++;
        psb->m_Link.m_pNext = m_FreeBlockList;
        m_FreeBlockList = &psb->m_Link;
    }

    //当GC决定一个对象已经死亡，并且SyncTableEntry的m_Object字段低位已经被设置为1时
    //只因为我们不能在GC期间做COM交互数据的清理而导致没有被回收
    //因此我们会把对象放进CleanUpList里，在后面一些时间进行回收
    void SyncBlockCache::CleanupSyncBlocks()
    {
        //断言是Finalizer线程
        _ASSERTE(GetThread() == FinalizerThread::GetFinalizerThread());
        //设置清理标识符
        m_bSyncBlockCleanupInProgress = TRUE;
        //EE异常处理准备
        struct Param
        {
            SyncBlockCache *pThis;
            SyncBlock* psb;
        } param;
        param.pThis = this;
        param.psb = NULL;
        EE_TRY_FOR_FINALLY(Param *, pParam, &param)
        {
            //重置Flag
            FinalizerThread::GetFinalizerThread()->ResetSyncBlockCleanup();
            //遍历列表进行清理
            while ((pParam->psb = pParam->pThis->GetNextCleanupSyncBlock()) != NULL)
            {
                //删除SyncBlock
                pParam->pThis->DeleteSyncBlock(pParam->psb);
                pParam->psb = NULL;
                //通知GC模式允许GC开始执行工作
                if (FinalizerThread::GetFinalizerThread()->CatchAtSafePointOpportunistic())
                {
                    FinalizerThread::GetFinalizerThread()->PulseGCMode();
                }
            }
        }
        EE_FINALLY
        {
            //结束
            m_bSyncBlockCleanupInProgress = FALSE;
            //如果捕获到，就进行删除
            if (param.psb)
                DeleteSyncBlock(param.psb);
        }EE_END_FINALLY;
    }

#### GC相关
#### 其他
同样的，这部分逻辑牵扯了GC，我们暂时进行搁置

#### 信息获取
信息获取依然非常简单，不再进行讲解

## 6. AwareLock解析

## 成员一览

### 字段定义
    LockState m_lockState
    ULONG           m_Recursion
    PTR_Thread      m_HoldingThread
    LONG            m_TransientPrecious
    DWORD           m_dwSyncIndex
    CLREvent        m_SemEvent
    DWORD m_waiterStarvationStartTimeMs
    DWORD WaiterStarvationDurationMsBeforeStoppingPreemptingWaiters = 100;

### 字段详解

#### m_lockState
    表示AwareLock的状态

#### m_Recursion
    递归次数

#### m_HoldingThread
    持有锁的线程

#### m_TransientPrecious
    暂时性的生命周期设置

#### m_dwSyncIndex
    提供SyncIndex以定位SyncTableEntry来查找持有的对象,其中最高位用作了Precious开启的设定

#### m_SemEvent
    提供最基础的通知服务(Pulse等等)

#### m_waiterStarvationStartTimeMs
    饥饿等待判定的开始时间，用于作差计算时间

#### WaiterStarvationDurationMsBeforeStoppingPreemptingWaiters
    顾名思义，在停止其余抢占等待者的饥饿持续时间间隔(ms)

### 函数定义

#### Enter-Leave对
    AwareLock::Enter 
    AwareLock::TryEnter 
    AwareLock::EnterEpilog 
    AwareLock::EnterEpilogHelper
    AwareLock::TryEnterBeforeSpinLoopHelper AwareLock::TryEnterAfterSpinLoopHelper 
    AwareLock::TryEnterHelper 
    AwareLock::TryEnterInsideSpinLoopHelper
    AwareLock::Leave 
    AwareLock::LeaveCompletely 
    AwareLock::LeaveHelper 

#### Precious
    AwareLock::DecrementTransientPrecious
    AwareLock::IncrementTransientPrecious
    AwareLock::SetPrecious

#### 信息获取设置类
    AwareLock::GetHoldingThread 
    AwareLock::GetLockState 
    AwareLock::GetMonitorHeldStateVolatile 
    AwareLock::GetOwningObject 
    AwareLock::GetOwningThread 
    AwareLock::GetPtrForLockContract 
    AwareLock::GetRecursionLevel 
    AwareLock::GetSyncBlockIndex 
    AwareLock::IsUnlockedWithNoWaiters 
    AwareLock::OwnedByCurrentThread 

#### 其余
    AwareLock::RecordWaiterStarvationStartTime AwareLock::ResetWaiterStarvationStartTime  
    AwareLock::ShouldStopPreemptingWaiters 
    AwareLock::Signal 
    AwareLock::SpinWait 
    AwareLock::AllocLockSemEvent
    AwareLock::InitializeToLockedWithNoWaiters 


### 函数解析

#### Enter-Leave
AwareLock的函数就难以进行分类了，因此我们依照一些作者所偏好的顺序进行解析。
首先看到Enter一类函数

    void AwareLock::Enter()
    {
        Thread *pCurThread = GetThread();
        LockState state = m_lockState.VolatileLoadWithoutBarrier();
        //保证没有上锁，或者线程不是持有线程
        if (!state.IsLocked() || m_HoldingThread != pCurThread)
        {
            //尝试获取锁进行登记
            if (m_lockState.InterlockedTryLock_Or_RegisterWaiter(this, state))
            {
                //成功后我们就获取了互斥锁
                m_HoldingThread = pCurThread;
                m_Recursion = 1;
                pCurThread->IncLockCount();
                return;
            }
            //失败就必须进行等待
            EnterEpilog(pCurThread);
            return;
        }
        //否则是持有线程再次上锁，进行递归自加
        _ASSERTE(m_Recursion >= 1);
        m_Recursion++;
    }

    BOOL AwareLock::TryEnter(INT32 timeOut)
    {
        Thread  *pCurThread = GetThread();
        //如果线程要求进行终止，进行终止处理
        if (pCurThread->IsAbortRequested())
        {
            pCurThread->HandleThreadAbort();
        }
        LockState state = m_lockState.VolatileLoadWithoutBarrier();
        //与上面情况是相同的
        if (!state.IsLocked() || m_HoldingThread != pCurThread)
        {
            //根据TimeOut选择是否进行登记等待
            if (timeOut == 0
                ? m_lockState.InterlockedTryLock(state)
                : m_lockState.InterlockedTryLock_Or_RegisterWaiter(this, state))
            {
                m_HoldingThread = pCurThread;
                m_Recursion = 1;
                pCurThread->IncLockCount();
                return true;
            }
            //未抢占成功，且TimeOut为0，因此直接返回
            if(timeOut == 0)
            {
                return false;
            }
            //否则进行阻塞等待
            return EnterEpilog(pCurThread, timeOut);
        }
        //否则同一线程，进行递归数自增
        _ASSERTE(m_Recursion >= 1);
        m_Recursion++;
        return true;
    }

    FORCEINLINE bool AwareLock::TryEnterHelper(Thread* pCurThread)
    {
        //尝试获取锁
        if (m_lockState.InterlockedTryLock())
        {
            m_HoldingThread = pCurThread;
            m_Recursion = 1;
            pCurThread->IncLockCount();
            return true;
        }
        //处理递归情况
        if (GetOwningThread() == pCurThread)
        {
            m_Recursion++;
            return true;
        }
        //失败
        return false;
    }

再看一下如何实现阻塞

    BOOL AwareLock::EnterEpilog(Thread* pCurThread, INT32 timeOut)
    {
        DebugBlockingItem blockingMonitorInfo;
        blockingMonitorInfo.dwTimeout = timeOut;
        blockingMonitorInfo.pMonitor = this;
        blockingMonitorInfo.pAppDomain = SystemDomain::GetCurrentDomain();
        blockingMonitorInfo.type = DebugBlock_MonitorCriticalSection;
        DebugBlockingItemHolder holder(pCurThread, &blockingMonitorInfo);
        //这里分开Helper的原因是，Helper使用了SEH(结构化异常处理)并且持有对应资源
        //释放所需的析构函数
        return EnterEpilogHelper(pCurThread, timeOut);
    }

那么就让我们看看核心逻辑Helper是如何实现的

    BOOL AwareLock::EnterEpilogHelper(Thread* pCurThread, INT32 timeOut)
    {
        //以下内容非常重要
        //调用者已经登记了Waiter，这个函数需要注销所有路径上的Waiter(包括异常处理)
        //在Runtime期间，如果哪里需要支持Thread终止，那么仍然也需要其注销Waiter
        //抢占式GC有可能切换模式来处理Thread终止，因此这个情况需要在把代码搬回
        //.Net Framework时被考虑
        _ASSERTE(pCurThread->PreemptiveGCDisabled());
        BOOLEAN IsContentionKeywordEnabled = ETW_TRACING_CATEGORY_ENABLED(MICROSOFT_WINDOWS_DOTNETRUNTIME_PROVIDER_DOTNET_Context, TRACE_LEVEL_INFORMATION, CLR_CONTENTION_KEYWORD);
        LARGE_INTEGER startTicks = { {0} };
        //如果竞争关键字被启用
        if (IsContentionKeywordEnabled)
        {
            QueryPerformanceCounter(&startTicks);
            //为托管竞争运行竞争开始的事件
            FireEtwContentionStart_V1(ETW::ContentionLog::ContentionStructs::ManagedContention, GetClrInstanceId());
        }
        LogContention();
        Thread::IncrementMonitorLockContentionCount(pCurThread);
        OBJECTREF obj = GetOwningObject();
        //我们不允许GC在我们运行时回收AwareLock，因此提高临时Precious
        IncrementTransientPrecious();
        DWORD ret;
        //启用GCProtect
        GCPROTECT_BEGIN(obj);
        {
            //检查Event是否分配了内存，如果没有，构造
            if (!m_SemEvent.IsMonitorEventAllocated())
            {
                AllocLockSemEvent();
            }
            _ASSERTE(m_SemEvent.IsMonitorEventAllocated());
            pCurThread->EnablePreemptiveGC();
            for (;;)
            {
                //我们可能在等待过程中被打断，所以我们需要异常处理
                struct Param
                {
                    AwareLock *pThis;
                    INT32 timeOut;
                    DWORD ret;
                } param;
                param.pThis = this;
                param.timeOut = timeOut;
                //统计我们等待的时间，以便我们醒来并且获取互斥锁失败时好调整剩余的TimeOut
                ULONGLONG start = CLRGetTickCount64();
                //请求EE异常处理
                EE_TRY_FOR_FINALLY(Param *, pParam, &param)
                {
                    //调用Event.Wait
                    pParam->ret = pParam->pThis->m_SemEvent.Wait(pParam->timeOut, TRUE);
                    _ASSERTE((pParam->ret == WAIT_OBJECT_0) || (pParam->ret == WAIT_TIMEOUT));
                }
                EE_FINALLY
                {
                    //如果捕获到了异常
                    if (GOT_EXCEPTION())
                    {
                        //到这里很有可能是APC(asynchronous procedure call,异步调用)抛出了异常
                        //负责等待的子系统保证线程的Wait会返回WAIT_OBJECT_0如果被已经唤醒的线程观测到发给Event的Signal
                        //因此在任意情形下出现Event被Signal和Wait抛异常之间的竞争时//被异常唤醒的线程不会观测到Signal
                        //如果有必要，另外一个线程会被Singal唤醒
                        m_lockState.InterlockedUnregisterWaiter();
                    }
                } EE_END_FINALLY;
                ret = param.ret;
                if (ret != WAIT_OBJECT_0)
                {
                    //如果超时也进行Waiter的注销
                    m_lockState.InterlockedUnregisterWaiter();
                    break;
                }
                //以下是在获取锁前自旋
                //这样做有几个好处
                //第一个，自旋帮助减少Waiter的饥饿状态，因为其他非Waiter的线程可以在有Waiter的情况下获取锁
                //一旦其中一个Waiter被唤醒，那么它就会在与其余自旋Waiter中更有可能获取到锁
                //第二个，如果有另一个线程在反复获取和释放锁，在等待前自旋可以防止Waiter反复切换上下文(切换上下文是相当耗时的操作)
                //第三个，在上面一种情况下，唤醒Waiter并且短暂性等待会导致Waiter的优先级下降，因为Event释放Waiter的顺序是FIFO.
                //短暂自旋会至少保留一个自旋阶段的优先级
                if (g_SystemInfo.dwNumberOfProcessors > 1)
                {
                    bool acquiredLock = false;
                    YieldProcessorNormalizationInfo normalizationInfo;
                    const DWORD spinCount = g_SpinConstants.dwMonitorSpinCount;
                    for (DWORD spinIteration = 0; spinIteration < spinCount; ++spinIteration)
                    {
                        if (m_lockState.InterlockedTry_LockAndUnregisterWaiterAndObserveWakeSignal(this))
                        {
                            acquiredLock = true;
                            break;
                        }
                        SpinWait(normalizationInfo, spinIteration);
                    }
                }
                //如果观测到信号就退出
                if (m_lockState.InterlockedObserveWakeSignal_Try_LockAndUnregisterWaiter(this))
                {
                    break;
                }
                //在计算timeout时间时，我们考虑几个特殊的情形
                //如果结束时Tick与开始时Tick相同，我们将其设置为1ms
                //来保证当有很多对互斥锁的竞争时，我们会有一些进展
                //其次，我们需要应对在我们等待时TickCount有溢出中断的情形
                //(我们最多能应付一次溢出中断，所以不要期待三个月的TimeOut会很准确)
                //很幸运的是，后者这种情形已经直接被32位算数运算巧妙地解决了。
                if (timeOut != (INT32)INFINITE)
                {
                    ULONGLONG end = CLRGetTickCount64();
                    ULONGLONG duration;
                    if (end == start)
                    {
                        duration = 1;
                    }
                    else
                    {
                        duration = end - start;
                    }
                    duration = min(duration, (DWORD)timeOut);
                    timeOut -= (INT32)duration;
                }
            }
            pCurThread->DisablePreemptiveGC();
        }
        GCPROTECT_END();
        DecrementTransientPrecious();
        //同样的，如果启用了竞争关键字，现在为竞争结束运行事件
        if (IsContentionKeywordEnabled)
        {
            LARGE_INTEGER endTicks;
            QueryPerformanceCounter(&endTicks);
            double elapsedTimeInNanosecond = ComputeElapsedTimeInNanosecond(startTicks, endTicks);
            FireEtwContentionStop_V1(ETW::ContentionLog::ContentionStructs::ManagedContention, GetClrInstanceId(), elapsedTimeInNanosecond);
        }
        //如果已经超时，返回false
        if (ret == WAIT_TIMEOUT)
        {
            return false;
        }
        //设置参量
        m_HoldingThread = pCurThread;
        m_Recursion = 1;
        pCurThread->IncLockCount();
        return true;
    }

再来看看TryEnter的Before,Inside,After

    FORCEINLINE AwareLock::EnterHelperResult AwareLock::TryEnterBeforeSpinLoopHelper(Thread *pCurThread)
    {
        LockState state = m_lockState.VolatileLoadWithoutBarrier();
        //在进行自旋循环前需要检查递归锁情况，如果一开始就不是递归情况
        //以后也不会是这种情况，因此自旋循环可以避免多次检查递归情况
        if (!state.IsLocked() || GetOwningThread() != pCurThread)
        {
            if (m_lockState.InterlockedTrySetShouldNotPreemptWaitersIfNecessary(this, state))
            {
                //这个线程不应该抢占Waiter，所以进行等待
                return EnterHelperResult_UseSlowPath;
            }
            //如果不是递归，就尝试获取锁或者登记自旋
            EnterHelperResult result = m_lockState.InterlockedTry_LockOrRegisterSpinner(state);
            if (result != EnterHelperResult_Entered)
            {
                //如果不是进入的情况
                //那么就是竞争，锁没有被获取，但是自旋被登记了
                //要么就是等待，当前线程不应该抢占Waiter
                return result;
            }
            //锁被获取，自旋没有被登记
             m_HoldingThread = pCurThread;
            m_Recursion = 1;
            pCurThread->IncLockCount();
            return EnterHelperResult_Entered;
        }
        //那么就是递归进入
        m_Recursion++;
        return EnterHelperResult_Entered;
    }

Inside

    FORCEINLINE AwareLock::EnterHelperResult AwareLock::TryEnterInsideSpinLoopHelper(Thread *pCurThread)
    {
        //尝试获取锁并且注销自旋.Before函数已经替我们检查了递归情况
        //所以这里就不必继续了
        EnterHelperResult result = m_lockState.InterlockedTry_LockAndUnregisterSpinner();
        if (result != EnterHelperResult_Entered)
        {
            //注释同上个函数
            return result;
        }
        m_HoldingThread = pCurThread;
        m_Recursion = 1;
        pCurThread->IncLockCount();
        return EnterHelperResult_Entered;
    }

After

    FORCEINLINE bool AwareLock::TryEnterAfterSpinLoopHelper(Thread *pCurThread)
    {
        //注销自旋并尝试获取锁
        //自旋者不能在没有尝试获取锁的情况下注销其自己
        //因为锁的释放者在自旋者能获取锁时不会唤醒Waiter
        if (!m_lockState.InterlockedUnregisterSpinner_TryLock())
        {
            return false;
        }
        m_HoldingThread = pCurThread;
        m_Recursion = 1;
        pCurThread->IncLockCount();
    }

再看看与之对应的Leave系列

    BOOL AwareLock::Leave()
    {
        Thread* pThread = GetThread();
        AwareLock::LeaveHelperAction action = LeaveHelper(pThread);
        switch(action)
        {
        case AwareLock::LeaveHelperAction_None:
            //成功，什么都不需要做
            return TRUE;
        case AwareLock::LeaveHelperAction_Signal:
            //通知事件
            Signal();
            return TRUE;
        default:
            _ASSERTE(action == AwareLock::LeaveHelperAction_Error);
            return FALSE;
        }
    }

此函数核心仍然是LeaveHelper

    FORCEINLINE AwareLock::LeaveHelperAction AwareLock::LeaveHelper(Thread* pCurThread)
    {
        //检查线程持有性
        if (m_HoldingThread != pCurThread)
            return AwareLock::LeaveHelperAction_Error;
        //检查Lock是否上锁
        _ASSERTE(m_lockState.VolatileLoadWithoutBarrier().IsLocked());
        //检查递归次数
        _ASSERTE(m_Recursion >= 1);
        //如果最后一次，进行释放
        if (--m_Recursion == 0)
        {
            m_HoldingThread->DecLockCount();
            m_HoldingThread = NULL;
            //清空锁所占用的Bit位，并且决定我们是否要让Waiter唤醒
            if (!m_lockState.InterlockedUnlock())
            {
                return AwareLock::LeaveHelperAction_None;
            }
            return AwareLock::LeaveHelperAction_Signal;
        }
        return AwareLock::LeaveHelperAction_None;
    }

Leave Compeletely就是简单地全部退出

    LONG AwareLock::LeaveCompletely()
    {
        LONG count = 0;
        while (Leave()) {
            count++;
        }
        //断言，否则我们连锁没拥有，Leave个锤子
        _ASSERTE(count > 0);
    }

#### Precious
     inline void AwareLock::SetPrecious()
     {
         m_dwSyncIndex |= SyncBlock::SyncBlockPrecious;
     }

Inc和Dec版本非常简单，通过Interlocked进行原子操作加减

#### 信息获取设置

此部分函数也非常简单，这里不再讲解。

#### 其余

    FORCEINLINE void AwareLock::RecordWaiterStarvationStartTime()
    {
        DWORD currentTimeMs = GetTickCount();
        if (currentTimeMs == 0)
        {
            //这里我们不记录0，0代表没有记录
            --currentTimeMs;
        }
        m_waiterStarvationStartTimeMs = currentTimeMs;
    }

下面这个函数通过饥饿时间判定是否应停止抢占性Waiter，让其进行Wait

    FORCEINLINE bool AwareLock::ShouldStopPreemptingWaiters() const
    {
        DWORD waiterStarvationStartTimeMs = m_waiterStarvationStartTimeMs;
        return
        waiterStarvationStartTimeMs != 0 &&
        GetTickCount() - waiterStarvationStartTimeMs >= WaiterStarvationDurationMsBeforeStoppingPreemptingWaiters;
    }

Reset也相当简单

    FORCEINLINE void AwareLock::ResetWaiterStarvationStartTime()
    {
        m_waiterStarvationStartTimeMs = 0;
    }

    void Signal()
    {
        //通知事件
        //CLREvent在没有初始化时也可以工作
        m_SemEvent.SetMonitorEvent();
        m_lockState.InterlockedTrySetShouldNotPreemptWaitersIfNecessary(this);
    }

    FORCEINLINE void AwareLock::SpinWait(const YieldProcessorNormalizationInfo &normalizationInfo, DWORD spinIteration)
    {
        //需要逻辑处理器个数大于一
        _ASSERTE(g_SystemInfo.dwNumberOfProcessors != 1);
        _ASSERTE(spinIteration < g_SpinConstants.dwMonitorSpinCount);
        //调用函数进行自旋等待
        YieldProcessorWithBackOffNormalized(normalizationInfo, spinIteration);
    }

    void AwareLock::AllocLockSemEvent()
    {
        //在我们从合作模式切换之前，保证SyncBlock不会在我们眼皮子下消失
        //(被GC回收)，像Event一样"昂贵"的对象，永久设置而不是使用临时性措施
        //"transiently"
        SetPrecious();
        GCX_PREEMP();
        //这里无需加锁，因为这个函数是线程安全的
        m_SemEvent.CreateMonitorEvent((SIZE_T)this);
    }

最后一个函数直接调用LockState进行转发初始化

    void InitializeToLockedWithNoWaiters(ULONG recursionLevel, PTR_Thread holdingThread)
    {
        m_lockState.InitializeToLockedWithNoWaiters();
        m_Recursion = recursionLevel;
        m_HoldingThread = holdingThread;
    } 

看到这里，是不是长舒一口气，我们有关于ObjectHeader的部分基本上已经全解析了。但是，还有一个AwareLock的内部类，LockState，我保证这是我们在进行下一节解析前所进行最后的"演出"了。

## 7. AwareLock::LockState解析

## 成员一览

### 字段定义
#### 成员
    UINT32 m_state
#### 静态常量
    UINT32 IsLockedMask
    UINT32 ShouldNotPreemptWaitersMask
    UINT32 SpinnerCountIncrement
    UINT32 SpinnerCountMask
    UINT32 IsWaiterSignaledToWakeMask
    UINT8 WaiterCountShift
    UINT32 WaiterCountIncrement
    UINT32 WaiterCountMask

### 字段解析
#### m_state
    用于储存State的变量

#### IsLockedMask
    位0，表示是否上锁掩码

#### ShouldNotPreemptWaitersMask
    位1，表示是否抢占Waiter掩码

#### SpinnerCountIncrement
    自旋计数的增量

#### SpinnerCountMask
    位2-4，自旋计数掩码

#### IsWaiterSignaledToWakeMask
    位5，表示Waiter是否被Signal唤醒

#### WaiterCountShift
    Waiter计数位移量

#### WaiterCountIncrement
    Waiter计数增量

#### WaiterCountMask
    Waiter计数掩码

### 函数定义

    AwareLock::LockState::CompareExchangeAcquire AwareLock::LockState::CompareExchange AwareLock::LockState::DecrementSpinnerCount AwareLock::LockState::DecrementWaiterCount AwareLock::LockState::GetMonitorHeldState 
    AwareLock::LockState::GetState 
    AwareLock::LockState::HasAnySpinners 
    AwareLock::LockState::HasAnyWaiters AwareLock::LockState::InitializeToLockedWithNoWaiters AwareLock::LockState::IncrementWaiterCount 
    AwareLock::LockState::InterlockedObserveWakeSignal_Try_LockAndUnregisterWaiter 
    AwareLock::LockState::InterlockedTry_LockAndUnregisterSpinner 
    AwareLock::LockState::InterlockedTry_LockAndUnregisterWaiterAndObserveWakeSignal 
    AwareLock::LockState::InterlockedTry_LockOrRegisterSpinner AwareLock::LockState::InterlockedTryLock AwareLock::LockState::InterlockedTryLock AwareLock::LockState::InterlockedTryLock_Or_RegisterWaiter AwareLock::LockState::InterlockedTrySetShouldNotPreemptWaitersIfNecessary AwareLock::LockState::InterlockedTrySetShouldNotPreemptWaitersIfNecessary AwareLock::LockState::InterlockedUnlock AwareLock::LockState::InterlockedUnregisterSpinner_TryLock AwareLock::LockState::InvertIsLocked AwareLock::LockState::InterlockedUnregisterWaiter AwareLock::LockState::InvertIsWaiterSignaledToWake AwareLock::LockState::InvertShouldNotPreemptWaiters AwareLock::LockState::IsLocked AwareLock::LockState::IsUnlockedWithNoWaiters AwareLock::LockState::IsWaiterSignaledToWake AwareLock::LockState::LockState AwareLock::LockState::NeedToSignalWaiter AwareLock::LockState::ShouldNonWaiterAttemptToAcquireLock AwareLock::LockState::ShouldNotPreemptWaiters AwareLock::LockState::TryIncrementSpinnerCount AwareLock::LockState::VolatileLoadWithoutBarrier AwareLock::LockState::VolatileLoad

### 函数解析

因为其中Private短函数很多且实现简单，我们着重解析逻辑较为复杂的Public函数

     FORCEINLINE bool AwareLock::LockState::InterlockedTryLock()
     {
         return InterlockedTryLock(VolatileLoadWithoutBarrier());
     }

这里进行了转发，因此我们直接看到核心函数

    FORCEINLINE bool AwareLock::LockState::InterlockedTryLock(LockState state)
    {
        //Monitor公平地以FIFO顺序释放Waiter，但是同时允许非Waiter在可以避开
        //Lock Convoys时候获取锁。
        //Lock Convoys对性能有着决定性影响，尤其是在多个线程周期性获取锁一小段时间
        //来访问数据时。
        //在Lock Convoys的情况下，一旦有Waiter在等待Lock，工作线程在抛弃时间片之前尝试获取锁
        //时被强制切换上下文
        //这个过程会一直重复，直到锁不再有Waiter，否则就会一直让所有工作线程切换上下文
        //导致的结果就是性能下降，并且进入一种很有可能累积更多Waiter的负反馈循环
        //为了避免Lock Convoys,每一个工作线程可以多次尝试获取锁，即使有一个Waiter
        //在没有锁竞争时让工作线程继续在其时间片内高效运作
        //这个策略可能导致Waiter承受饥饿，但减轻方法是由其余部分进行
        //参见InterlockedTrySetShouldNotPreemptWaitersIfNecessary().
        if (state.ShouldNonWaiterAttemptToAcquireLock())
        {
            LockState newState = state;
            newState.InvertIsLocked();
            return CompareExchangeAcquire(newState, state) == state;
        }
        return false;
    }

    FORCEINLINE bool AwareLock::LockState::InterlockedUnlock()
    {
        //断言确定是否已经上锁
        static_assert_no_msg(IsLockedMask == 1);
        _ASSERTE(IsLocked());
        //原子操作释放
        LockState state = InterlockedDecrementRelease((LONG *)&m_state);
        while (true)
        {
            //记录线程是否已经被Signal激活但还没有从Wait中唤醒
            //IsWaiterSignaledToWakeMask在被Signal的线程观测唤醒时被清空
            //这允许一个线程在有其余很多Waiter时多次获取并且释放锁
            //在这种情况下，我们不希望线程在每次释放锁时都唤醒Waiter
            //因为会导致更多被唤醒的线程在发现锁不能获取造成的不必要上下文切换
            //后又进入Wait状态，因此，一次只唤醒一个线程
            if (!state.NeedToSignalWaiter())
            {
                return false;
            }
            //进行原子操作交换
            LockState newState = state;
            newState.InvertIsWaiterSignaledToWake();
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            //如果成功就返回true
            if (stateBeforeUpdate == state)
            {
                return true;
            }
            //失败就继续
            state = stateBeforeUpdate;
        }
    }


    FORCEINLINE bool AwareLock::LockState::InterlockedTrySetShouldNotPreemptWaitersIfNecessary(AwareLock *awareLock)
    {
        return InterlockedTrySetShouldNotPreemptWaitersIfNecessary(awareLock, VolatileLoadWithoutBarrier());
    }

此函数仍然是进行转发，看到核心函数

    FORCEINLINE bool AwareLock::LockState::InterlockedTrySetShouldNotPreemptWaitersIfNecessary(
    AwareLock *awareLock,
    LockState state)
    {
        _ASSERTE(awareLock != nullptr);
        _ASSERTE(&awareLock->m_lockState == this);
        //通常的，线程被允许抢占Waiter来获取锁来避免Lock Convoys
        //有一种情形下Waiter能很容易地进入饥饿状态
        //比如，一个线程持有锁很长时间(比上下文切换时间长的多)
        //然后马上释放，又在很短时间内成功获取锁，并且一直重复. 虽然Waiter会在锁被释放
        //时被唤醒，但通常其没有足够的时间做上下文切换并且得到锁
        //因此会承受超长时间的饥饿
        //---------------------------------------------------------------------
        //为了防止这种饥饿情况并且将进程推进，有时候改变常规的策略禁止线程抢占Waiter是非常必要的
        //ShouldNotPreemptWaiters()表明当前的状态并且决定是否需要禁止非Waiter抢占Waiter
        //-当第一个Waiter开始等待时，我们记录一个开始时间，这段时间往后Waiter实际上没有任何进展
        // 当Waiter获取锁时，这个时间就被更新到现有时间
        //-这个函数检查是否饥饿时间超过了阈值，如果是的话，设置ShouldNotPreemptWaiters()
        //---------------------------------------------------------------------
        //当不合理的饥饿发生时，锁会被时不时释放。如果这个是被自旋者引起的，自旋者将开始自旋
        //-在自旋开始之前，如果ShouldNotPreemptWaiters()被设置，自旋者会跳过自旋部分
        // 并且进行等待.
        // 子选择如果早在ShouldNotPreemptWaiters()被设置前登记了，那么其会出于必要停止自旋
        // 最终，所有自旋者都会消失并且没有新的会被登记。
        //-在释放锁的基础上，如果没有自旋者，Waiter将被唤醒.
        // 在那种情况下，此函数被调用
        //-最后，当所有自旋者都消失后，仅有一个Waiter能够获取锁，当Waiter获取锁或者
        // 当最后一个Waiter注销自己后，ShouldNotPreemptWaiters()将被清空来储存正常
        // 的策略
        while (true)
        {
            //如果没有Waiter，返回false
            if (!state.HasAnyWaiters())
            {
                //检查是否为正常策略
                _ASSERTE(!state.ShouldNotPreemptWaiters());
                return false;
            }
            //如果不应抢占，返回true
            if (state.ShouldNotPreemptWaiters())
            {
                return true;
            }
            //如果没有发生长时间饥饿，返回false
            if (!awareLock->ShouldStopPreemptingWaiters())
            {
                return false;
            }
            //交换状态
            LockState newState = state;
            newState.InvertShouldNotPreemptWaiters();
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            if (stateBeforeUpdate == state)
            {
                return true;
            }
            state = stateBeforeUpdate;
        }
    }

让我们再来看一下自旋者登记相关函数

    FORCEINLINE AwareLock::EnterHelperResult AwareLock::LockState::InterlockedTry_LockOrRegisterSpinner(LockState state)
    {
        while (true)
        {
            LockState newState = state;
            //如果应该进行锁的获取
            if (state.ShouldNonWaiterAttemptToAcquireLock())
            {
                //反转上锁位
                newState.InvertIsLocked();
            }
            //如果不应该抢占，或者不能增加自旋数，就进行等待
            else if (state.ShouldNotPreemptWaiters() || !newState.TryIncrementSpinnerCount())
            {
                return EnterHelperResult_UseSlowPath;
            }
            //标准原子交换步骤
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            if (stateBeforeUpdate == state)
            {
                //如果新状态仍可以进获取，那么就表示已经进入
                //否则就是正在竞争
                return state.ShouldNonWaiterAttemptToAcquireLock() ? EnterHelperResult_Entered : EnterHelperResult_Contention;
            }
            //再试
            state = stateBeforeUpdate;
        }
    }

    FORCEINLINE AwareLock::EnterHelperResult AwareLock::LockState::InterlockedTry_LockAndUnregisterSpinner()
    {
        LockState state = VolatileLoadWithoutBarrier();
        while (true)
        {
            //此函数是在自旋循环中被调用，其必须注销Spinner当且仅当锁被获取时
            _ASSERTE(state.HasAnySpinners());
            //如果不能获取锁
            if (!state.ShouldNonWaiterAttemptToAcquireLock())
            {
                //判断是否不应该抢占
                //如果是，就进行Wait
                //否则就是竞争(有其余Spinner)
                return state.ShouldNotPreemptWaiters() ? EnterHelperResult_UseSlowPath : EnterHelperResult_Contention;
            }
            //原子交换
            LockState newState = state;
            newState.InvertIsLocked();
            newState.DecrementSpinnerCount();
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            if (stateBeforeUpdate == state)
            {
                //成功就返回
                return EnterHelperResult_Entered;
            }
            //再试
            state = stateBeforeUpdate;
        }
    }

    FORCEINLINE bool AwareLock::LockState::InterlockedUnregisterSpinner_TryLock()
    {
        //原子操作减少计数
        LockState stateBeforeUpdate = InterlockedExchangeAdd((LONG *)&m_state, -(LONG)SpinnerCountIncrement);
        //断言之前的自旋者数量
        _ASSERTE(stateBeforeUpdate.HasAnySpinners());
        //如果已经上锁，就返回false
        if (stateBeforeUpdate.IsLocked())
        {
            return false;
        }
        //一直使用原子操作，直到锁被自己获取
        do
        {
            LockState newState = state;
            newState.InvertIsLocked();
            LockState stateBeforeUpdate = CompareExchangeAcquire(newState, state);
            if (stateBeforeUpdate == state)
            {
                return true;
            }
            state = stateBeforeUpdate;
        } while (!state.IsLocked());
        return false;
    }

    FORCEINLINE bool AwareLock::LockState::InterlockedTryLock_Or_RegisterWaiter(AwareLock *awareLock, LockState state)
    {
        //前置断言
        _ASSERTE(awareLock != nullptr);
        _ASSERTE(&awareLock->m_lockState == this);
        bool waiterStarvationStartTimeWasReset = false;
        while (true)
        {
            LockState newState = state;
            //如果可以获取，反转
            if (state.ShouldNonWaiterAttemptToAcquireLock())
            {
                newState.InvertIsLocked();
            }
            else
            {
                //否则增加Waiter数量
                newState.IncrementWaiterCount();
                //如果原有State不含有任何Waiter并且时间没有重设
                if (!state.HasAnyWaiters() && !waiterStarvationStartTimeWasReset)
                {
                    //这个分支应该是第一个Waiter.
                    //一旦Waiter被登记，另一个线程也许会检查Waiter的饥饿状态
                    //起始时间这时已经无效，造成ShouldNotPreemptWaiters()被不必要地设置
                    //在登记Waiter之前进行重置
                    waiterStarvationStartTimeWasReset = true;
                    awareLock->ResetWaiterStarvationStartTime();
                }
            }
            //原子操作
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            if (stateBeforeUpdate == state)
            {
                //更新成功
                //如果仍可以获取锁，返回true
                if (state.ShouldNonWaiterAttemptToAcquireLock())
                {
                    return true;
                }
                //如果没有任何Waiter，断言已经重设
                if (!state.HasAnyWaiters())
                {
                    _ASSERTE(waiterStarvationStartTimeWasReset);
                    awareLock->RecordWaiterStarvationStartTime();
                }
                //返回false(有其余竞争者)
                return false;
            }
            //交换失败，继续
            state = stateBeforeUpdate;
        }
    }

    FORCEINLINE void AwareLock::LockState::InterlockedUnregisterWaiter()
    {
        LockState state = VolatileLoadWithoutBarrier();
        while (true)
        {
            //断言Waiter存在
            _ASSERTE(state.HasAnyWaiters());
            //原子操作，减少Waiter数量，设置抢占策略
            LockState newState = state;
            newState.DecrementWaiterCount();
            if (newState.ShouldNotPreemptWaiters() && !newState.HasAnyWaiters())
            {
                newState.InvertShouldNotPreemptWaiters();
            }
            //交换
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            //如果成功，直接返回
            if (stateBeforeUpdate == state)
            {
                return;
            }
            //再试
            state = stateBeforeUpdate;
        }
    }

    FORCEINLINE bool AwareLock::LockState::InterlockedTry_LockAndUnregisterWaiterAndObserveWakeSignal(AwareLock *awareLock)
    {
        //断言
        _ASSERTE(awareLock != nullptr);
        _ASSERTE(&awareLock->m_lockState == this);
        //此函数是从Waiter自旋循环的内部被调用且仅当锁被获取时观测唤醒信号
        //以防止锁的释放者唤醒另一个Waiter并且这过程中有另一个已经在自旋获取
        //锁了
        bool waiterStarvationStartTimeWasRecorded = false;
        LockState state = VolatileLoadWithoutBarrier();
        while (true)
        {
            //断言是否有Waiter和Waiter是否已经唤醒
            _ASSERTE(state.HasAnyWaiters());
            _ASSERTE(state.IsWaiterSignaledToWake());
            //如果已经上锁，返回false
            if (state.IsLocked())
            {
                return false;
            }
            //设置新状态
            LockState newState = state;
            //设置上锁状态
            newState.InvertIsLocked();
            //唤醒
            newState.InvertIsWaiterSignaledToWake();
            //减少Waiter数量
            newState.DecrementWaiterCount();
            //如果不应该抢占就进行设置
            if (newState.ShouldNotPreemptWaiters())
            {
                newState.InvertShouldNotPreemptWaiters();
                if (newState.HasAnyWaiters() && !waiterStarvationStartTimeWasRecorded)
                {
                    //更新时间
                    waiterStarvationStartTimeWasRecorded = true;
                    awareLock->RecordWaiterStarvationStartTime();
                }
            }
            //原子操作交换
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            //交换成功
            if (stateBeforeUpdate == state)
            {
                //如果有Waiter
                if (newState.HasAnyWaiters())
                {
                    _ASSERTE(!state.ShouldNotPreemptWaiters() || waiterStarvationStartTimeWasRecorded);
                    //且没有进行计时，开始计时
                    if (!waiterStarvationStartTimeWasRecorded)
                    {
                        awareLock->RecordWaiterStarvationStartTime();
                    }
                }
                //返回true
                return true;
            }
            //失败重来
            state = stateBeforeUpdate;
        }
    }

    FORCEINLINE bool AwareLock::LockState::InterlockedObserveWakeSignal_Try_LockAndUnregisterWaiter(AwareLock *awareLock)
    {
        //断言检查
        _ASSERTE(awareLock != nullptr);
        _ASSERTE(&awareLock->m_lockState == this);
        //使用原子操作来注销IsWaiterSignaledToWakeMask
        //这个函数在自旋循环的结尾被调用，其必须总是观察唤醒Signal
        //如果锁可以被获取，其必须获取锁并且注销Waiter
        //同样地，Waiter必须获取锁且一道观测唤醒Signal
        //因为锁的释放者不会唤醒一个已经被Signal但是还没有醒来的Waiter
        LockState stateBeforeUpdate = InterlockedExchangeAdd((LONG *)&m_state, -(LONG)IsWaiterSignaledToWakeMask);
        //检查之前是否有唤醒
        _ASSERTE(stateBeforeUpdate.IsWaiterSignaledToWake());
        //如果之前已经上锁，就返回false
        if (stateBeforeUpdate.IsLocked())
        {
             return false;
        }
        bool waiterStarvationStartTimeWasRecorded = false;
        LockState state = stateBeforeUpdate;
        //设置IsWaiterSignaledToWakeMask
        state.InvertIsWaiterSignaledToWake();
        //再次检查
        _ASSERTE(!state.IsLocked());
        do
        {
            //检查是否有Waiter
            _ASSERTE(state.HasAnyWaiters());
            LockState newState = state;
            newState.InvertIsLocked();
            newState.DecrementWaiterCount();
            //如果不应该抢占Waiter
            if (newState.ShouldNotPreemptWaiters())
            {
                //反转
                newState.InvertShouldNotPreemptWaiters();
                //如果有Waiter且没有记录
                if (newState.HasAnyWaiters() && !waiterStarvationStartTimeWasRecorded)
                {
                    //更新饥饿起始时间，这个时间必须在ShouldNotPreemptWaiters()
                    //被清空前进行更新，一旦其被清空，其余现场可能会检查Waiter的
                    //饥饿时间，并且之前的值已经过期，导致ShouldNotPreemptWaiters()
                    //被不必要地设置
                    waiterStarvationStartTimeWasRecorded = true;
                    awareLock->RecordWaiterStarvationStartTime();
                }
            }
            //原子交换
            LockState stateBeforeUpdate = CompareExchange(newState, state);
            if (stateBeforeUpdate == state)
            {
                //锁被成功获取
                if (newState.HasAnyWaiters())
                {
                    _ASSERTE(!state.ShouldNotPreemptWaiters() || waiterStarvationStartTimeWasRecorded);
                    if (!waiterStarvationStartTimeWasRecorded)
                    {
                        //更新饥饿起始时间
                        awareLock->RecordWaiterStarvationStartTime();
                    }
                }
                return true;
            }
            //否则就继续
            state = stateBeforeUpdate;
        }while (!state.IsLocked());
        return false;
    }

## 结语

解析到这里，我们已经熟知了ObjectHeader所牵扯的内容，AwareLock的实现，SyncBlock的机制。相信你对整个ObjectHeader的功能有了更为深刻的理解，下一章我们就将进入类型的世界，首先下手的仍是与Object有关的MethodTable。