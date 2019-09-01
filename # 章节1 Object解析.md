# 章节1 Object解析
## 1.什么是Object
摘自 *MSDN* 的定义:
>Supports all classes in the .NET class hierarchy and provides low-level services to derived classes. This is the ultimate base class of all .NET classes; it is the root of the type hierarchy.

**Object** 是整个 **.Net**框架中的基类，并且为许多功能提供基础性服务。看到这里我相信
大家应该明白 **Object** 在 **CLR** 中的地位了。因此我们第一章节就从 **Object** 的解析入手
。

## 2.代码来源
所有的代码都来自于 **Mircosoft CoreCLR** 在GitHub上开源项目以及 **CLI** 标准实现的雏形版本 **SSCLI** ，也称 “**Rotor**”。

## 3.Object成员一览
### 字段定义
    PTR_MethodTable* m_pMethTab
看到这里你是否会感到一些疑惑，不是说好 **Object** 还有 **ObjectHeader** 以及用于同步的 **SyncBlock** 呢？别着急，先来看一眼注释

    Object is the respesentation of an managed object on the GC heap.
    The only fields mandated by all objects are
     * a pointer to the code:MethodTable at offset 0
     * a poiner to a code:ObjHeader at a negative offset. This is often 
     zero.  It holds information that any addition information that we 
     might need to attach to arbitrary objects. 
 
 这里明确指出了，一个 **Object** 所拥有和管理的只有两个字段，一个 **MethodTable**
 指针在 **Object** 开头（即Offset为0），还有一个叫做 **ObjectHeader**的东西，这个对象存在于 **Object** 对象的开头之前，如果读者很熟悉 C\C++ 的话就可以很熟悉这个**Object**
 在内存中的排布情况(从左到右表示内存顺序)：

[ ObjectHeader ]{ several bytes } |这里才是Object开始的地方 [ MethodTablePtr ][ 第一个字段开始的地方 ]

那么ObjectHeader在 **CoreCLR** 中获取的代码是如何写的呢？   

    PTR_ObjHeader GetHeader()
    {
        LIMITED_METHOD_DAC_CONTRACT;
        return dac_cast &lt PTR_ObjHeader &gt(this) - 1;
    }

这里的 dac\_cast 我们暂且不管，可以将其视作static\_cast，简单地转换this指针类型
并且向前偏移 sizeof(**ObjectHeader**) 个字节，以便取到 **ObjectHeader** 对象的指针。

### 方法定义
这里我们事先将方法分类
#### 类型信息
##### TypeHandle获取
    Object::GetGCSafeTypeHandle
    Object::GetGCSafeTypeHandleIfPossible
    Object::GetTypeHandle
    Object::GetTrueTypeHandle
##### MethodTable的获取和设置
    Object::GetMethodTable
    Object::GetGCSafeMethodTable
    Object::GetMethodTablePtr
    Object::RawGetMethodTable
    Object::RawSetMethodTable
    Object::SetMethodTableForLargeObject
##### 其他信息
    Object::GetNumComponents
    Object::GetSize
    Object::SupportsInterface

#### 基础服务
##### Monitor
    Object::EnterObjMonitor
    Object::EnterObjMonitorHelper
    Object::EnterObjMonitorHelperSpin
    Object::LeaveObjMonitor
    Object::LeaveObjMonitorAtException
    Object::LeaveObjMonitorHelper
    Object::Pulse
    Object::PulseAll
    Object::Wait
    Object::GetThreadOwningMonitorLock

##### SyncBlock
    Object::PassiveGetSyncBlock
    Object::GetSyncBlock
    Object::GetSyncBlockIndex
    Object::HasEmptySyncBlockInfo

##### 拆箱
    Object::UnBox

##### HashCode
    Object::GetHashCodeEx
    Object::ComputeHashCode

##### 数据获取
    Object::GetAddress
    Object::GetData
    Object::GetOffsetOfFirstField


##### 辅助函数
    Object::AssertNotArray
    Object::GetOffset16
    Object::GetOffset32
    Object::GetOffset64
    Object::GetOffset8
    Object::SetOffset16
    Object::SetOffset32
    Object::SetOffset64
    Object::SetOffset8
    Object::SetOffsetObjectRef
    Object::SetOffsetPtr

##### GC
    Object::EnumMemoryRegions
    Object::GetSlotMap

## 4. 函数解析
相信大家在看到上述已经分类的成员函数后，已经对 *MSDN* 所对 **Object**下的定义有了更为
深刻的理解。那么就让我们从类型的获取部分开始，为了简明起见，某些不必要的语句会被删除。
### 类型信息
#### TypeHandle
    inline TypeHandle Object::GetGCSafeTypeHandle() const
    {
        //获取MethodTable
        MethodTable * pMT = GetGCSafeMethodTable();
        //判定MethodTable指针是否为空
        _ASSERTE(pMT != NULL);
        //如果是Array类型就使用ArrayBase提供的方法返回TypeHandle
        //否则就使用TypeHandle的构造函数，从MethodTable指针构造
        if (pMT->IsArray())
            return ArrayBase::GetTypeHandle(pMT);
        else 
            return TypeHandle(pMT);
    }

这部分代码较为简单，读者可直接参考作者所写注释阅读。在这里我们看到了一个奇特的现象，
按道理讲，类型信息应该是由Object本身直接提供，而这里体现出来的却是MethodTable担起了
获取类型信息的责任，这里我们会在后面关于MethodTable对象讲解里更深入探究。

还有一点就是这里似乎把普通类型和Array类型分开处理了，这一点我们会在后期系列Blog中讲到。

让我们再继续看另外一个较为复杂的版本
    
        //此函数用于正在进行GC时获取TypeHandle，如果能获取，就返回，否则返回null
        TypeHandle Object::GetGCSafeTypeHandleIfPossible() const
        {
            //熟悉的味道，获取MethodTable并且判null
            //有时候获取TypeHandle可能不安全，并且会引发递归查找
            //在某些情况下，直接获取MethodTable总是安全且直截了当的
            MethodTable * pMT = GetGCSafeMethodTable();
            _ASSERTE(pMT != NULL);
            //若MethodTable已经被释放，返回null
            if (pMT == g_pFreeObjectMethodTable)
            {
                return NULL;
            }
            //接下来的处理和检查是应对GC时可能出现的Access Violation
            //假设有这样的类型
            //ValueType1 &lt ValueType2 &gt[] array
            //这里的ValueType1和ValueType2定义在不同的Assembly里面，这样的话，如果要获取
            //TypeHandle,需要在Type1所在Module里的m_AssemblyRefByNameTable变量
            //里查找，但是！假如目标Appdomain正在卸载，就会导致Access Violation
            //类似也有相同情形
            //RefType1 &lt RefType2 &gt array
            //RefType2 所在Module在GC之前就被卸载
            //这样的话，GC会由AppDomain的卸载触发,以下是流程
            //AppDomain::Unload ->AppDomain::Exit -> GCInterface::AddMemoryPressure ->
            //WKS::GCHeapUtilities::GarbageCollect
            MethodTable * pMTToCheck = pMT;
            if (pMTToCheck->IsArray())
            {
                TypeHandle thElem = static_cast<const ArrayBase * const>(this)->GetArrayElementTypeHandle();
                //理想情况下，我们本应该直接调用 theElem.GetLoaderModule()
                //但现有的TypeDesc::GetLoaderModule()实现依赖于某个数据结构
                //而这个数据结构可能也已经随着AppDomain被卸载
                //所以这里的代码是在模拟TypeDesc::GetLoaderModule()对应Array的情况
                //一直迭代，找到Array嵌套最里层的类型，比如 MyType[][][]这种，最后我们要找到的就是
                //MyType
                while (thElem.HasTypeParam()) 
                {
                    thElem = thElem.GetTypeParam();
                }
                pMTToCheck = thElem.GetMethodTable();
            }
            //获取MethodTable所在Loader的模块
            //并且确定我们没有查找那些正在卸载的Assembly里的类型
            Module * pLoaderModule = pMTToCheck->GetLoaderModule();
            LoaderAllocator * pLoaderAllocator = pLoaderModule->GetLoaderAllocator();
            _ASSERTE(pLoaderAllocator != NULL);
            if ((pLoaderAllocator->IsCollectible()) &&(ObjectHandleIsNull(pLoaderAllocator->GetLoaderAllocatorObjectHandle())))
            {
                return NULL;
            }
            //做完所有检查了，现在就可以安全返回TypeHandle了
            return GetGCSafeTypeHandle();
        }


从这里的注释可以看出来，为什么我们使用类型信息的时候是使用MethodTable而不是直接的TypeHandle，因为"有时候获取TypeHandle可能不安全，并且会引发递归查找"。

     TypeHandle Object::GetTrueTypeHandle()
     {
        if (m_pMethTab->IsArray())
            return ((ArrayBase*) this)->GetTypeHandle();
        else
            return TypeHandle(GetMethodTable());
    }



    inline TypeHandle Object::GetTypeHandle()
    { 
        if (m_pMethTab->IsArray())
            return (dac_cast<PTR_ArrayBase>(this))->GetTypeHandle();
        else 
            return TypeHandle(m_pMethTab);
    }

这部分代码同上

#### MethodTable

    #define MARKED_BIT 0x1
    PTR_MethodTable GetMethodTable() const {
        //检查掩码，如果正在进行GC，使用GetGCSafeMethodTable()获取MethodTable
        _ASSERTE((dac_cast<ULONG_PTR>(m_pMethTab) & MARKED_BIT) == 0);
        return m_pMethTab;
    }

这里已经可以看出端倪了，含有GCSafe等字样的函数是作用于GC期间安全获取某些数据使用的。
他们的逻辑往往更为复杂，需要考虑很多Access Violation问题。

    //为简便起见，作者已经直接翻译了一些无关紧要宏
    PTR_MethodTable* GetMethodTablePtr() const{
         return dac_cast &lt PTR_MethodTable* &gt((ULONG_PTR)&(this)->m_pMethTab);
    }

GCSafe版本

    PTR_MethodTable GetGCSafeMethodTable() const{
        //GC期间，LSB位被设置用于已经标记的对象
        //第二个临着LSB位的是保留位，所以如果我们想要在GC期间
        //安全获取MethodTable，我们必须清零最后两位
        //3的二进制是11取反即00
        return dac_cast &lt PTR_MethodTable &gt((dac_cast<ULONG_PTR>(m_pMethTab)) & ~((UINT_PTR)3));
    }

接下来两个版本就很原始粗暴了，如同他们的名字一样“Raw”

    MethodTable *RawGetMethodTable() const
    {
        return m_pMethTab;
    }

    void RawSetMethodTable(MethodTable *pMT)
    {
        m_pMethTab = pMT;
    }

接下来这个是针对大对象的MethodTable设置，不过有常识的开发人员应当知道，大对象堆一般都是数组对象。

    void SetMethodTableForLargeObject(MethodTable *pMT)
    {
        //如果大对象堆上正在发生内存分配，就必须使用写屏障
        //因为MethodTable可能会成为可回收对象
        ErectWriteBarrierForMT(&m_pMethTab, pMT);
    }

#### 其他信息
    //Component个数获取
    inline DWORD Object::GetNumComponents()
    {
        //我们甚至可以不是Array来调用这个方法，这个方法只是在读取Object内存而已
        //与Array无关，但是ComponentSize会乘出这个值，因此m_NumComponents必须是
        //ArrayBase的第一个字段
        return dac_cast &lt PTR_ArrayBase &gt (this)->m_NumComponents;
    }

这里注释可能会让人有些疑惑，不过我们可以看到 object.h 开头的注释:

    因为有GC，堆对象大小必须被非常快的计算出来。
    对象内存排布的限制保证了这一点是可以做到的。

    任意从Object派生的对象都必须使用对象跟在Object后的头四个字节
    以及从MethodTable出发可到达的常量来计算自身所需的全部字节数

    其计算公式是:
    MT->GetBaseSize() + ((OBJECTTYPEREF->GetSizeField() * MT->GetComponentSize())
    因此对于Object，这个值是固定的，因为ComponentSize是0，所以大小只有BaseSize这么多

看到这里大家应该很清楚上面注释的意义了，因为我们只能使用头四个字节
而Array有固定数量的元素，所以理所应当地，头四个字节应当储存Array的Length，
而一个int的大小刚好是4 bytes，和我们在库里看到的定义如出一辙！

这里我们再次搬出内存排布做对比
    
    真实内存: [Header][     Object    ][其余内存]
    Object排布:       [MethodTablePtr][字段的开始的地方]
    Array排布:                        [长度信息(int 4 bytes)][元素储存开始的地方]

接下来看GetSize

    遵循公式计算大小
    inline SIZE_T Object::GetSize()
    {
        //获取MethodTable
        MethodTable *mT = GetGCSafeMethodTable();
        //检查ComponentSize要么小于等于2，要么是Array
        //其中String的就刚好为2
        _ASSERTE(( mT->GetComponentSize() <= 2) || mT->IsArray());
        size_t s = mT->GetBaseSize();
        //如果有ComponentSize
        if (mT->HasComponentSize())
            s += (size_t)GetNumComponents() * mT->RawGetComponentSize();
        return s;
    }

这里其实已经清楚了，ComponentSize是内建类型的特权，他们可以辅助GC进行优化。
我们再次回到开头查看注释，已然明了
    
    Object 基类
    StringObject 针对String操作特殊化的对象类型(UCS-2/UTF-16数据)，用于获取更好的性能
    Utf8StringObject 同上，只不过编码为UTF-8
    BaseObjectWithCachedData 加一个字段进行缓存
        ReflectClassBaseObject 反射对象基类
        ReflectMethodObject 反射Method对象
        ReflectFieldObject  反射Field对象
    ArrayBase
        I1Array 一维数组
        I2Array 二维数组
        ...
        INArray N维数组
        PtrArray 对象引用数组
    AssemblyBaseObject Assembly对象

接下来看一个有点意思的

    BOOL Object::SupportsInterface(OBJECTREF pObj, MethodTable* pInterfaceMT)
    {
        UNREACHABLE();
    }

这里直接被标记为Unreachable，没有实现。 根据名字知道这是一个检测一个对象是否支持某个
Interface的接口，不过目前没有看见。在C#里面，我们仍然使用这样的语句来判断是否实现Interface

    Object.IsAssignableFrom()
	
### 基础服务
#### Monitor
这里我们只列举一些函数
    
    void EnterObjMonitor()
    {
        GetHeader()->EnterObjMonitor();
    }

    BOOL TryEnterObjMonitor(INT32 timeOut = 0)
    {
        return GetHeader()->TryEnterObjMonitor(timeOut);
    }

    FORCEINLINE AwareLock::EnterHelperResult EnterObjMonitorHelper(Thread* pCurThread)
    {
        return GetHeader()->EnterObjMonitorHelper(pCurThread);
    }

    FORCEINLINE AwareLock::EnterHelperResult EnterObjMonitorHelperSpin(Thread* pCurThread)
    {
        WRAPPER_NO_CONTRACT;
        return GetHeader()->EnterObjMonitorHelperSpin(pCurThread);
    }

    BOOL LeaveObjMonitorAtException()
    {
        WRAPPER_NO_CONTRACT;
        return GetHeader()->LeaveObjMonitorAtException();
    }

观察以上函数，可以发现几乎都是一个代理，将很多操作转发给ObjectHeader。那么我们就暂且不深究。ObjectHeader的内容会放到下一章进行讲解。

#### SyncBlock

    SyncBlock *GetSyncBlock()
    {
        WRAPPER_NO_CONTRACT;
        return GetHeader()->GetSyncBlock();
    }

    DWORD GetSyncBlockIndex()
    {
        WRAPPER_NO_CONTRACT;
        return GetHeader()->GetSyncBlockIndex();
    }

同样的，SyncBlock内容也是转发到了ObjectHeader，这里就不再赘述。

##### 拆箱
    //如果是ValueType，获取内存中数据的首地址
    inline PTR_VOID Object::UnBox()
    {
        //先判断是否为值类型
        _ASSERTE(GetMethodTable()->IsValueType());
        //判断nullable
        _ASSERTE(!Nullable::IsNullableType(TypeHandle(GetMethodTable())));
        //跳过自身，获取内存地址，与之前的内存排布图一致
        return dac_cast<PTR_BYTE>(this) + sizeof(*this);
    }

##### HashCode
    INT32 Object::GetHashCodeEx()
    {
        //这里有一个循环的原因是
        //我们在检索ObjectHeader，而对象可能在多个线程竞争时被更改
        //这种情况下，自旋锁bit位会被设置，我们不应该去修改
        //因此，我们失败后需要重试
        DWORD iter = 0;
        DWORD dwSwitchCount = 0;
        while (true)
        {
            DWORD bits = GetHeader()->GetBits();
            if (bits & BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX)
            {
                //如果已经有了HashCode，直接返回
                if (bits & BIT_SBLK_IS_HASHCODE)
                {            
                    return  bits & MASK_HASHCODE;
                }
                else
                {
                    //如果我们有SyncBlock的索引，那么SyncBlock里一定有HashCode
                    //否则我们就创建一个，并且储存在里面
                    SyncBlock *psb = GetSyncBlock();
                    DWORD hashCode = psb->GetHashCode();
                    if (hashCode != 0)
                        return  hashCode;
                    hashCode = ComputeHashCode();
                    return psb->SetHashCode(hashCode);
                }
            }
            else
            {
                //如果已经有一个线程持有锁，那么我们需要一个SyncBlock
                if ((bits & (SBLK_MASK_LOCK_THREADID)) != 0)
                {
                    GetSyncBlock();
                }
                else
                {
                    //我们这时想要修改Header，就必须检查BIT_SBLK_SPIN_LOCK
                    if (bits & BIT_SBLK_SPIN_LOCK)
                    {
                        iter++;
                        if ((iter % 1024) != 0 && g_SystemInfo.dwNumberOfProcessors > 1)
                        {
                            //向处理器报告我们正在自旋
                            YieldProcessorNormalized(); 
                        }
                        else
                        {
                            __SwitchToThread(0, ++dwSwitchCount);
                        }
                        continue;
                    }
                    DWORD hashCode = ComputeHashCode();
                    DWORD newBits = bits | BIT_SBLK_IS_HASH_OR_SYNCBLKINDEX | BIT_SBLK_IS_HASHCODE | hashCode;
                    //如果修改成功，直接返回
                    if (GetHeader()->SetBits(newBits, bits) == bits)
                        return hashCode;
                    //否则，Header在我们操作过程中已经被更改，再来一次
                }
            }
        }    
    }

上面的函数是基于竞争的流程，下面让我们看看如何真正计算HashCode

    #define HASHCODE_BITS 26
    DWORD Object::ComputeHashCode()
    {
        DWORD hashCode;
        //需要注意的是，现在这个算法最多使用 HASHCODE_BITS 位来计算HashCode
        //在Object被冻结的情况下，HashCode必须被搬回ObjectHeader里面，这样就
        //刚刚好
        do
        {
            hashCode = GetThread()->GetNewHashCode() >> (32-HASHCODE_BITS);
        }
        while (hashCode == 0);//确保hashcode不为0
        _ASSERTE((hashCode & ((1<<HASHCODE_BITS)-1)) == hashCode);//检查Hashcode是否与 HASHCODE_BITS 相容
        return hashCode;
    }

##### 数据获取

    PTR_BYTE GetAddress()
    {
        return dac_cast<PTR_BYTE>(this);
    }

    PTR_BYTE GetData()
    {
        return dac_cast<PTR_BYTE>(this) + sizeof(Object);
    }

    static UINT GetOffsetOfFirstField()
    {
        LIMITED_METHOD_CONTRACT;
        return sizeof(Object);
    }

这些函数都比较简单，也间接告诉了我们对象的内存排布。

##### 辅助函数

辅助函数中，GetOffset,SetOffset都是基于GetData获取或者设置不同大小的指针，我们不再单独开篇幅讲。我们只看下面几个函数。

    //这里OBJECTREF在编译时是Object*
    typedef OBJECTREF Object*
    #define SetObjectReference SetObjectReferenceUnchecked
    void Object::SetOffsetObjectRef(DWORD dwOffset, size_t dwValue)
    {
        OBJECTREF*  location;
        OBJECTREF   o;
        //获取Offset处的OBJECTREF数据指针
        location = (OBJECTREF *) &GetData()[dwOffset];
        //从dwValue转换为目标引用
        o        = ObjectToOBJECTREF(*(Object **)  &dwValue);
        //设置引用
        SetObjectReference( location, o );
    }

紧接着是SetObjectReferenceUnchecked函数体

    void SetObjectReferenceUnchecked(OBJECTREF *dst,OBJECTREF ref)
    {
         //这里的转换实际上跟OBJECTREF有关
         //OBJECTREF在debug时是一个代理类，release编译时是Object*
         //如果不转换，OJBECTREF的 operator= 重载会导致一个写屏障的断言失败
         VolatileStore((Object**)dst, OBJECTREFToObject(ref));
         //屏障
         ErectWriteBarrier(dst, ref);
    }

##### GC相关

    CGCDesc* GetSlotMap()                        
    { 
        return (CGCDesc::GetCGCDescFromMT(GetMethodTable())); 
    }

还有一个函数也是类似，不过逻辑会复杂一些，最后通过GC的函数转发。我们不打算在这里讲，因为这涉及到GC的知识，会专门在GC篇章讲解这些函数。

## 结语
解析到这里，我们的工作似乎还远远不够，但大体上我们已经了解了Object的基本构成，和一些函数流程。下一期我们会继续讲解 **ObjectHeader**。

