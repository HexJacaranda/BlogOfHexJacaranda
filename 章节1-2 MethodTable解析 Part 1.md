# 一点写在开头的话
上一章节貌似太过于长了，后面有时间会考虑进行分割。对于本次解析对象MethodTable也是如此，我将尽可能将内容分割开来。

#勘误
    1. 在DoFullyLoad中 ForwardedStruct 实际上应当被翻译为转发结构体，
       这与运行时特性转发类型(Forward Type)有关，简单地来讲，这个特性
       允许你将类型移动到另一个Assembly里面而不必重新编译。
       比如：我们在开发时叫utility.dll的assembly中有一个example类，现在
       我们需要重构这个example，并且放在新的assembly里，那么新版本的就是
       utility.dll+新assembly，这会导致库中原有example无法被解析，所以
       需要使用TypeForward特性


# 章节1-2 MethodTable解析
## 前导知识
    1. 关于泛型:
    每一个泛型类都有一个对应的泛型MethodTable，其有以下的作用:
        * MethodTable在反射中被用于表示泛型类。
        * 在VirtualTable中的MethodDescription被用于反射，但其不应该被调用。
        其他信息，比如BaseSzie，与泛型毫不相干，但是最终还是被放进了MethodTable。

    2. 每一个不同的泛型实例化有自己对的MethodTable.
    然而EEClass可以在一些可兼容的泛型实例化中被共享，例如List<string>和List<object>
    在这种情况下MethodDescription也在可兼容实例化中被共享。因此属于同一个EEClass的VirtualTableEntry也是相同的。

    3. Non-VitrualTable部分仅仅表示给众多泛型实例其中一个(实际上是要求第一个)。
    Non-VitrualTable永远不会从对象的VitrualTable指针进行访问，因此总可以确保
    其是通过包含他们的代表MethodTable访问。

    4. MehtodTable是运行时类型的基础表示。它拥有对象的大小，GC内存排布，和针对
    虚函数调度的VirtualTable(不包含Interface的调度，这个我们后面会讲到，Interface
    使用了一种叫VSD的调度方法)

## 成员一览

### 字段一览
    DWORD           m_dwFlags
    DWORD           m_BaseSize
    WORD            m_wFlags2
    WORD            m_wToken
    WORD            m_wNumVirtuals
    WORD            m_wNumInterfaces
    ParentMT_t m_pParentMethodTable
    RelativePointer<PTR_Module> m_pLoaderModule
    PlainPointer<PTR_MethodTableWriteableData> m_pWriteableData
    union (1)
    {
        PlainPointer<DPTR(EEClass)> m_pEEClass
        PlainPointer<TADDR> m_pCanonMT
    }
    union (2)
    {
        PerInstInfo_t m_pPerInstInfo
        TADDR         m_ElementTypeHnd
        TADDR         m_pMultipurposeSlot1
    }
    union (3)
    {
        PlainPointer<PTR_InterfaceInfo>   m_pInterfaceMap
        TADDR               m_pMultipurposeSlot2
    }

事情开始变得有趣了，字段越来越多而且出现了union。其实这里没有列出所有的成员，MethodTable仍有叫Optional Member(可选成员)的东西。

### 字段解析
字段解析前的一些提示:这些成员必须放在结构体的开头并且与缓存线相容，不要乱动，这些内容会被GC使用(这里指前两个成员)。
#### m_dwFlags
    低位一个WORD大小被用于记录Array和String的Component Size
    高位WORD被用于其他Flag记录
#### m_BaseSize
    在堆上分配时的对象基本大小
#### m_wFlags2
    用于Flag记录
#### m_wToken
    如果ClassToken小于16位，就储存于此。若大于16位，此字段为(DWORD)-1
    Token被储存于TokenOverflow Optional Member中
#### m_wNumVirtuals
#### m_wNumInterfaces
    分别记录了Virtual函数个数和接口个数
    * 通常情况下我们没法用完整个WORD
#### m_pParentMethodTable
    父类的MethodTable
    * 在Linux ARM上此指针为RelativeFixupPointer，其余情况就是PTR_MethodTable
    * 此指针在enum_flag_enum_flag_HasIndirectParent被设置时指向一个
      Indirection Cell
    * 此成员允许转换Helper沿着派生链向上走。其不需要检查enum_flag_HasIndirectParentMethodTable
#### m_pLoaderModule
    装载器所在模块。这与NGEN映像中的激活的Module一样
#### union(1)
    union
    {
        PlainPointer<DPTR(EEClass)> m_pEEClass
        PlainPointer<TADDR> m_pCanonMT
    }
    先来看一组指示union储存什么的枚举，此枚举使用低2位进行储存
    enum LowBits {
        UNION_EECLASS      = 0, 指向EEClass，且这个MethodTable是代表性(Canonical)的
        UNION_INVALID      = 1, 不使用
        UNION_METHODTABLE  = 2, 指向代表性MethodTable
        UNION_INDIRECTION  = 3  指向Indirection Cell，Cell指向代表性MethodTable(仅当PreJIT特性开启时使用)
    };
    "Canonical"在这里表示是否为泛型的代表MethodTable或者一个普通的MethodTable

#### union(2)
    union (2)
    {
        PerInstInfo_t m_pPerInstInfo
        TADDR         m_ElementTypeHnd
        TADDR         m_pMultipurposeSlot1
    }
    m_pPerInstInfo和m_pInterfaceMap因为JIT代码和Helper的性能敏感性
    必须在固定的偏移处。
    然而，这两个字段经常不储存在这里。这段空间实际上遵循先来先到的准则，如果
    固定的字段没有被设置，这个字段就会被多用途槽使用(Multipurpose Slot)
    这里多用途槽包括: DispatchMapSlot，NonVirtualSlots，ModuleOverride
    (详细参见enum_flag_MultipurposeSlotsMask)
    如果这里无法放下多用途槽，那么其就会储存在VirtualTableSlot之后。
#### union(3)
    union
    {
        PlainPointer<PTR_InterfaceInfo>   m_pInterfaceMap
        TADDR               m_pMultipurposeSlot2
    }
    此union相关信息参考union(2)

### 字段排布
    在此之后，会依次出现以下未直接写出的字段，他们按顺序是:
    1. VirtualTable Slot 和 Non-VirtualTable Slot
    2. Overflow Multipurpose Slot
    3. Optional Members
    4. Generic Dictionary Pointers
    5. Interface Map
    6. Generic Instantiation 与 Dictionary

### 函数解析
咦？函数一览和分类哪去了？实际上是函数太多，太杂，作者已经放弃掉列举了。结合代码本身的分块，我们就将逐个讲解MethodTable涉及到的方方面面。

#### 纯正性校验
    BOOL MethodDesc::SanityCheck()
    {
        //检查是否储存了其余内存
        //检查是否已经恢复
        if (IsRestored())
        {
            //进一步校验
            //我们不关心结果如何，实际上只要不触发AV(Access Violation)即可
            return GetMethodTable() == m_pDebugMethodTable.GetValue() && this->GetModule() != NULL;
        }
    }

#### 获取类型相关AppDomain，NGen Module

    PTR_Module GetModule()
    {
        g_IBCLogger.LogMethodTableAccess(this);
        //为非泛型，非数组类型使用快速通道返回
        if ((m_dwFlags & (enum_flag_HasComponentSize | enum_flag_GenericsMask)) == 0)
            //返回Module
            return GetLoaderModule();
        //如果是Array，就是自己的Module，否则就是代表性MethodTable
        MethodTable * pMTForModule = IsArray() ? this : GetCanonicalMethodTable();
        //如果Module内没有对MethodTable的重载
        if (!pMTForModule->HasModuleOverride())
            //返回Module
            return pMTForModule->GetLoaderModule();
        //否则从附加字段中查找储存Module的Slot
        TADDR pSlot = pMTForModule->GetMultipurposeSlotPtr(enum_flag_HasModuleOverride, c_ModuleOverrideOffsets);
        //转换为Slot
        return RelativeFixupPointer<PTR_Module>::GetValueAtPtr(pSlot);
    }

    //本函数只是没有开头一句Log而已
    PTR_Module GetModule_NoLogging();

    Assembly *GetAssembly()
    {
        //简单转发，获取Assembly
        return GetModule()->GetAssembly();
    }

    PTR_Module GetModuleIfLoaded()
    {
    #ifdef FEATURE_PREJIT // PreJIT特性
        g_IBCLogger.LogMethodTableAccess(this);
        MethodTable * pMTForModule = IsArray() ? this : GetCanonicalMethodTable();
        if (!pMTForModule->HasModuleOverride())
            return pMTForModule->GetLoaderModule();
        //思路同上，如果已经储存就返回，没有就要求ModuleLoad
        return Module::RestoreModulePointerIfLoaded(pMTForModule->GetModuleOverridePtr(), pMTForModule->GetLoaderModule());
    #else
        return GetModule();
    #endif
    }

    //此函数通过已经实例化的类型获取Domain，比如C<L,R>会返回共同的Domain(Domain中立)
    //如果其中任何部分来自于一个Domain，就会返回那个Domain
    //需要注意的是，如果所有部分都是Domain-Bound，那么他们一定来自一个Domain
    PTR_BaseDomain GetDomain()
    {
        return dac_cast<PTR_BaseDomain>(AppDomain::GetCurrentDomain());
    }

    //指示在NGen Module里是否被激活
    BOOL IsZapped()
    {
    #ifdef FEATURE_PREJIT
        return GetFlag(enum_flag_IsZapped);
    #else
        //默认返回FALSE
        return FALSE;
    #endif
    }

    //对于在已经过NGen的Assembly里的类型，此函数返回含有这个MethodTable的Module
    PTR_Module GetZapModule()
    {
        //默认为null
        PTR_Module zapModule = NULL;
        //检测是否活跃或者PreJIT根本没有开启
        if (IsZapped())
        {
            zapModule = ReadPointer(this, &MethodTable::m_pLoaderModule);
        }
        return zapModule;
    }

    //* 通常情况下，对于未构造类型如List<T>，GetLoaderModule() == GetModule()
    //  对于已经构造的类型，如List<string>，在LoaderModule里持有一个用于类型访问
    //  的HashTable。
    //* 决定LoaderModule的规则必须保证一个类型在考虑到AppDomain卸载下不会超出
    //  LoaderModule的生命周期
    //* GetModuleForStatics() 是第三类的Module，其担任了承载静态信息的职责
    PTR_Module GetLoaderModule()
    {
        return ReadPointer(this, &MethodTable::m_pLoaderModule);
    }

    //获取Loader的Allocator
    PTR_LoaderAllocator MethodTable::GetLoaderAllocator() 
    {
        return GetLoaderModule()->GetLoaderAllocator();
    }

    //设置LoaderModule
    void MethodTable::SetLoaderModule(Module* pModule) 
    {
        m_pLoaderModule.SetValue(pModule);
    }

    void MethodTable::SetLoaderAllocator(LoaderAllocator* pAllocator) 
    {
        //实际上并不会设置Allocator，我们要求Allocator是一致的
        _ASSERTE(pAllocator == GetLoaderAllocator());
        if (pAllocator->IsCollectible())
        {
            SetFlag(enum_flag_Collectible);
        }
    }

    //获取Domain本地Module，对静态初始化检查非常有用
    PTR_DomainLocalModule MethodTable::GetDomainLocalModule()
    {
        return GetModuleForStatics()->GetDomainLocalModule();
    }

    //* 装载封闭此类型的MethodTable(即此类型是一个嵌套类型)
    //* 此函数能被DoFullyLoad()间接调用以进行一部分的访问性检查
    //  因此只会装载已经处于CLASS_DEPENDENCIES_LOADED级别的类型
    MethodTable *MethodTable::LoadEnclosingMethodTable(ClassLoadLevel targetLevel)
    {
        //尝试获取嵌套EEClass外层类型Token，如果不是嵌套，则返回mdTypeDefNil
        mdTypeDef tdEnclosing = GetEnclosingCl();
        if (tdEnclosing == mdTypeDefNil)
        {
            return NULL;
        }
        //装载类型定义并返回MethodTable
        return ClassLoader::LoadTypeDefThrowing(GetModule(),
                                            tdEnclosing,
                                            ClassLoader::ThrowIfNotFound,
                                            ClassLoader::PermitUninstDefOrRef,
                                            tdNoTypes,
                                            targetLevel
                                            ).GetMethodTable();
    }

#### 默认构造函数
    //是否有默认构造函数，使用GetFlag做转发
    BOOL HasDefaultConstructor()
    {
        GetFlag(enum_flag_HasDefaultCtor);
    }

    //此函数同理，使用SetFlag转发
    void SetHasDefaultConstructor();

    //获取默认构造函数的Slot索引
    WORD GetDefaultConstructorSlot()
    {
        //断言检查
        _ASSERTE(HasDefaultConstructor());
        //默认构造函数跟随在Class构造器后(如果有的话)
        //而Non-VirtualTableSlot又是放在VirtualTableSlot后,所以简单相加
        return GetNumVirtuals() + (HasClassConstructor() ? 1 : 0);
    }

    //获取默认构造函数的描述符
    MethodDesc *GetDefaultConstructor()
    {
        //断言检查
        _ASSERTE(HasDefaultConstructor());
        //获取Method
        MethodDesc *pCanonMD = GetMethodDescForSlot(GetDefaultConstructorSlot());
        //* ValueType的默认构造函数是一个实例化的Stub
        //  找到对应Stub最简单的方法就是使用以下函数
        //* 在最简单的情形下，寻找一个Class的默认构造函数立即返回传入的pCanonMD
        return MethodDesc::FindOrCreateAssociatedMethodDesc(pCanonMD,
                                                    this,
                                                    FALSE//去掉已装箱的Stub,
                                                    Instantiation(), 
                                                    FALSE//没有参数的Stub);
    }

    //是否有显式或隐式公开的默认构造函数
    BOOL HasExplicitOrImplicitPublicDefaultConstructor()
    {
        //如果是ValueType，则默认隐式有一个构造函数
        if (IsValueType())
        {
            return TRUE;
        }
        //根本没有，返回False
        if (!HasDefaultConstructor())
        {
            return FALSE;
        }
        MethodDesc * pCanonMD = GetMethodDescForSlot(GetDefaultConstructorSlot());
        //获取Method，检查访问性
        return pCanonMD != NULL && pCanonMD->IsPublic();
    }


#### 类初始化条件检查
    * 如果有需要，激活DomainLocalModule
    * 调用.cctor(类初始化器)

    //检查类初始化器是否应该对此类型调用，如果有需要就调用
    void CheckRunClassInitThrowing()
    {
        //如果已经提前初始化了
        if (IsClassPreInited())
            return;
        //不要初始化被泛型实例共享的类如 MyClass<T>
        if (IsSharedByGenericInstantiations())
            return;
        //获取LocalModule
        DomainLocalModule *pLocalModule = GetDomainLocalModule();
        _ASSERTE(pLocalModule);
        //获取类型索引
        DWORD iClassIndex = GetClassIndex();
        //检查我们是否已经为此类型调用了.cctor
        if (!pLocalModule->IsClassAllocated(this, iClassIndex))
            pLocalModule->PopulateClass(this);
        //如果没有初始化就调用初始化
        if (!pLocalModule->IsClassInitialized(this, iClassIndex))
            DoRunClassInitThrowing();
    }

    //* 检查没有beforefieldinit特性的类的初始化器是否对此类型的继承链向上的
    //  所有类型已经调用了。如果有必要，进行调用。  
    //* 此函数模拟了对象构造时调用构造函数的行为
    void CheckRunClassInitAsIfConstructingThrowing()
    {
        //是否有.cctor
        if (HasPreciseInitCctors())
        {
            MethodTable *pMTCur = this;
            //沿着继承链向上走
            while (pMTCur != NULL)
            {
                if (!pMTCur->GetClass()->IsBeforeFieldInit())
                    pMTCur->CheckRunClassInitThrowing();
                pMTCur = pMTCur->GetParentMethodTable();
            }
        }
    }

以下四个函数比较简单，就不做详解了

    void SetClassInited();
    BOOL  IsClassInited();
    BOOL IsInitError();
    void SetClassInitError();
    BOOL IsClassPreInited();


#### 设置/获取信息函数

    //从另一个MethodTable拷贝Flag
    void CopyFlags(MethodTable * pOldMT)
    {
        m_dwFlags = pOldMT->m_dwFlags;
        m_wFlags2 = pOldMT->m_wFlags2;
    }

    //为Array初始化m_dwFlags字段
    void SetIsArray(CorElementType arrayType, CorElementType elementType)
    {
        //设置分类
        DWORD category = enum_flag_Category_Array;
        if (arrayType == ELEMENT_TYPE_SZARRAY)
            category |= enum_flag_Category_IfArrayThenSzArray;
        _ASSERTE((m_dwFlags & enum_flag_Category_Mask) == 0);
        m_dwFlags |= category;
        //最后检查类型
        _ASSERTE(GetInternalCorElementType() == arrayType);
    }

    //是否为全局类
    inline BOOL IsGlobalClass()
    {
        return (GetTypeDefRid() == RidFromToken(COR_GLOBAL_PARENT_TOKEN));
    }

    //获取Domain里唯一表示该类的Index
    DWORD GetClassIndex()
    {
        return GetClassIndexFromToken(GetCl());
    }
其中GetClassIndexFromToken非常简单

    DWORD   GetClassIndexFromToken(mdTypeDef typeToken)
    {
        return RidFromToken(typeToken) - 1;
    }

#### 类构造函数(.cctor)

    //类构造函数描述符获取
    MethodDesc * GetClassConstructor()
    {
        return GetMethodDescForSlot(GetClassConstructorSlot());
    }

    //是否定义了.cctor
    BOOL HasClassConstructor()
    {
        return GetFlag(enum_flag_HasCctor);
    }

下面的代码同理

    void SetHasClassConstructor();

    inline WORD MethodTable::GetClassConstructorSlot()
    {
        //断言是否有.cctor
        _ASSERTE(HasClassConstructor());
        //正如前面所说, .cctor是在VirtualTableSlot后第一个
        return GetNumVirtuals();
    }
很遗憾，Set函数在源代码中只有一个签名，其余什么都没有，也没有任何地方引用。

    void SetClassConstructorSlot (WORD wCCtorSlot);
接下来是一个与PreJIT有关的功能

    //获取.cctor的信息
    ClassCtorInfoEntry* MethodTable::GetClassCtorInfoIfExists()
    {
    #ifdef FEATURE_PREJIT
        //如果在Module中不是活跃状态，返回null
        if (!IsZapped())
            return NULL;
        g_IBCLogger.LogCCtorInfoReadAccess(this);
        //如果有已经封箱的静态数据
        if (HasBoxedRegularStatics())
        {
            //Module的构造函数信息
            ModuleCtorInfo *pModuleCtorInfo = GetZapModule()->GetZapModuleCtorInfo();
            //MethodTable_Ptr指针
            DPTR(RelativePointer<PTR_MethodTable>) ppMT = pModuleCtorInfo->ppMT;
            //热数据偏移量HashTableEntry数组
            PTR_DWORD hotHashOffsets = pModuleCtorInfo->hotHashOffsets;
            //冷数据偏移量HashTableEntry数组
            PTR_DWORD coldHashOffsets = pModuleCtorInfo->coldHashOffsets;
            //如果热数据数量不为0
            if (pModuleCtorInfo->numHotHashes)
            {
                //为此MethodTable生成HashCode
                DWORD hash = pModuleCtorInfo->GenerateHash(PTR_MethodTable(this), ModuleCtorInfo::HOT);
                //索引小于数量
                _ASSERTE(hash < pModuleCtorInfo->numHotHashes);
                //开放寻址HashTable查找InfoEntry的偏移量
                for (DWORD i = hotHashOffsets[hash]; i != hotHashOffsets[hash + 1]; i++)
                {
                    _ASSERTE(!ppMT[i].IsNull());
                    //找到，进行返回
                    if (dac_cast<TADDR>(pModuleCtorInfo->GetMT(i)) == dac_cast<TADDR>(this))
                    {
                        return pModuleCtorInfo->cctorInfoHot + i;
                    }
                }
            }
            //与上面一致，这里只是在冷数据HashTable中查找
            if (pModuleCtorInfo->numColdHashes)
            {
                DWORD hash = pModuleCtorInfo->GenerateHash(PTR_MethodTable(this), ModuleCtorInfo::COLD);
                _ASSERTE(hash < pModuleCtorInfo->numColdHashes);
                for (DWORD i = coldHashOffsets[hash]; i != coldHashOffsets[hash + 1]; i++)
                {
                    _ASSERTE(!ppMT[i].IsNull());
                    if (dac_cast<TADDR>(pModuleCtorInfo->GetMT(i)) == dac_cast<TADDR>(this))
                    {
                        return pModuleCtorInfo->cctorInfoCold + (i - pModuleCtorInfo->numElementsHot);
                    }
                }
            }
        }
    #endif
        //没有PreJIT就直接返回null
        return NULL;
    }

#### 从MethodTable储存与恢复

    * 如果此MethodTable没有被恢复，那么就做恢复工作
    * 此操作通过强制类型装载来达到目的(类型装载会调用从MethodTable恢复的函数)
    * 恢复在父类，或者Interface中引用自己的类型
      (比如Int32:IComparable<Int32>)需要使用挂起列表

    //获取保存的扩展，传出开始和结尾指针
    void MethodTable::GetSavedExtent(TADDR *pStart, TADDR *pEnd)
    {
        TADDR start;
        //如果含有指针，就减掉Desc的大小，从Desc算起
        if (ContainsPointersOrCollectible())
            start = dac_cast<TADDR>(this) - CGCDesc::GetCGCDescFromMT(this)->GetSize();
        else
            //否则就从自身开始
            start = dac_cast<TADDR>(this);
        //结尾自然就在Optional Member结尾处
        TADDR end = dac_cast<TADDR>(this) + GetEndOffsetOfOptionalMembers();
        _ASSERTE(start && end && (start < end));
        *pStart = start;
        *pEnd = end;
    }

    //分配用于常规静态信息储存的箱子
    void MethodTable::AllocateRegularStaticBoxes()
    {
        //进入GC协作模式
        GCX_COOP();
        //获取静态信息基址
        PTR_BYTE pStaticBase = GetGCStaticsBasePointer();
        //开启GC内部保护
        GCPROTECT_BEGININTERIOR(pStaticBase);
    #ifdef FEATURE_PREJIT
        //在过NGen的情况下，我们有储存已经装箱的静态MethodTable缓存数组.
        //在过JIT的情况下，我们只有字段描述符
        //尝试获取.cctor信息
        ClassCtorInfoEntry *pClassCtorInfoEntry = GetClassCtorInfoIfExists();
        //如果是过NGen
        if(pClassCtorInfoEntry != NULL)
        {
            //获取第一个Slot
            OBJECTREF* pStaticSlots = (OBJECTREF*)(pStaticBase + pClassCtorInfoEntry->firstBoxedStaticOffset);
            //GC保护
            GCPROTECT_BEGININTERIOR(pStaticSlots);
            //获取MethodTable数组指针
            ArrayDPTR(RelativeFixupPointer<PTR_MethodTable>) ppMTs = GetLoaderModule()->GetZapModuleCtorInfo()->
            GetGCStaticMTs(pClassCtorInfoEntry->firstBoxedStaticMTIndex);
            //获取已经装箱的个数
            DWORD numBoxedStatics = pClassCtorInfoEntry->numBoxedStatics;
            for (DWORD i = 0; i < numBoxedStatics; i++)
            {
                //逐个恢复MethodTable
                Module::RestoreMethodTablePointer(&(ppMTs[i]), GetLoaderModule());
                //读取MethodTable进行断言
                MethodTable *pFieldMT = ppMTs[i].GetValue();
                _ASSERTE(pFieldMT);
                //向日志写入
                LOG((LF_CLASSLOADER, LL_INFO10000, "\tInstantiating static of type %s\n", pFieldMT->GetDebugClassName()));
                //从MethodTable构建对象
                OBJECTREF obj = AllocateStaticBox(pFieldMT, pClassCtorInfoEntry->hasFixedAddressVTStatics);
                //在Slot中储存对应对象
                SetObjectReference( &(pStaticSlots[i]), obj);
            }
            GCPROTECT_END();
        }else
    #endif
        {
            //否则就是没有JIT的情况
            //检查是否激活
            _ASSERTE(!IsZapped());
            //获取Field指针
            //如果有泛型静态信息，就取泛型静态字段描述符，否则就取实例Field末尾
            FieldDesc *pField = HasGenericsStaticsInfo() ? 
            GetGenericsStaticFieldDescs() : (GetApproxFieldDescListRaw() + GetNumIntroducedInstanceFields());
            //静态字段结尾
            FieldDesc *pFieldEnd = pField + GetNumStaticFields();
            //进行遍历
            while (pField < pFieldEnd)
            {
                //断言是否为静态字段
                _ASSERTE(pField->IsStatic());
                //如果不是特殊静态字段，且是值
                if (!pField->IsSpecialStatic() && pField->IsByValue())
                {
                    //获取类型Handle
                    TypeHandle  th = pField->GetFieldTypeHandleThrowing();
                    //获取类型MethodTable
                    MethodTable* pFieldMT = th.GetMethodTable();
                    //记录
                    LOG((LF_CLASSLOADER, LL_INFO10000, "\tInstantiating static of type %s\n", pFieldMT->GetDebugClassName()));
                    //从MethodTable构建对象
                    OBJECTREF obj = AllocateStaticBox(pFieldMT, HasFixedAddressVTStatics());
                    //设置对象
                    SetObjectReference( (OBJECTREF*)(pStaticBase + pField->GetOffset()), obj);
                }
                pField++;
            }
        }
        GCPROTECT_END();
    }

可以看见以上函数在进行分配Box时，为了获取对应MethodTable，在NGen和JIT情况下采取了不同的路径。再来看看AllocateStaticBox是如何实现的

    static OBJECTREF AllocateStaticBox(MethodTable* pFieldMT, BOOL fPinned, OBJECTHANDLE* pHandle = 0)
    {
        //需要断定为ValueType
        _ASSERTE(pFieldMT->IsValueType());
        //如果有必要，激活一切MethodTable所代表类型所需要的Module
        pFieldMT->EnsureInstanceActive();
        //从MethodTable构造对象
        OBJECTREF obj = AllocateObject(pFieldMT);
        //如果有必要，固定对象
        if (fPinned)
        {
            LOG((LF_CLASSLOADER, LL_INFO10000, "\tSTATICS:Pinning static (VT fixed address attribute) of type %s\n", pFieldMT->GetDebugClassName()));
            //创建固定句柄
            OBJECTHANDLE oh = GetAppDomain()->CreatePinningHandle(obj);
            //设置固定句柄
            if (pHandle)
            {
                *pHandle = oh;
            }
        }
        else
        {
            if (pHandle)
            {
                *pHandle = NULL;
            }
        }
        //返回对象
        return obj;
    }

    void MethodTable::CheckRestore()
    {
        //是否完全被装载
        if (!IsFullyLoaded())
        {
            //进行装载
            ClassLoader::EnsureLoaded(this);
            //断言
            _ASSERTE(IsFullyLoaded());
        }
        //写入日志
        g_IBCLogger.LogMethodTableAccess(this);
    }

    //* 对于在Native Image(本机映像)中的MethodTable，对已经编码的，且其TypeKey
    //  对此类型是可恢复的指针进行解码
    //* 对于已经实例化的泛型，我们需要泛型的参数类型，EEClass指针，还有其所在
    //  Module的指针(对于非泛型类，EEClass和Module是紧紧被绑定在一起的)
    //* 此过程是递归进行的，比如对于类型C<D<string>[]>，且保证在遇到环形
    //  解析时进行终止
    //* 应当注意的是，这个函数不需要任何锁，恢复信息的过程是幂等的
    //  (任意多次操作和一次操作的效果是一样的)
    void DoRestoreTypeKey()
    {
        //如果我们有Indirection Cell,就直接恢复m_pCanonMT和其module指针
        if (union_getLowBits(m_pCanonMT.GetValue()) == UNION_INDIRECTION)
        {
            Module::RestoreMethodTablePointerRaw((MethodTable **)(union_getPointer(m_pCanonMT.GetValue())),
                GetLoaderModule(), CLASS_LOAD_UNRESTORED);
        }
        //获取正确的MethodTable
        MethodTable * pMTForModule = IsArray() ? this : GetCanonicalMethodTable();
        //如果在Module内有重载，恢复重载MethodTable信息
        if (pMTForModule->HasModuleOverride())
        {
            Module::RestoreModulePointer(pMTForModule->GetModuleOverridePtr(), pMTForModule->GetLoaderModule());
        }
        //如果是Array，则需要恢复其元素的TypeHandle
        if (IsArray())
        {
            Module::RestoreTypeHandlePointerRaw(GetApproxArrayElementTypeHandlePtr(), 
                                                GetLoaderModule(), CLASS_LOAD_UNRESTORED);
        }
        //获取实例化列表，逐个恢复并且进行递归
        Instantiation inst = GetInstantiation();
        for (DWORD j = 0; j < inst.GetNumArgs(); j++)
        {
            Module::RestoreTypeHandlePointer(&inst.GetRawArgs()[j], GetLoaderModule(), CLASS_LOAD_UNRESTORED);
        }
        //设置标志位
        FastInterlockAnd(&(EnsureWritablePages(GetWriteableDataForWrite())->m_dwFlags), ~MethodTableWriteableData::enum_flag_UnrestoredTypeKey);
    }

    //获取是否仍然有未恢复的Type
    inline BOOL HasUnrestoredTypeKey() const
    {
        return !IsPreRestored() && 
            (GetWriteableData()->m_dwFlags & MethodTableWriteableData::enum_flag_UnrestoredTypeKey) != 0;
    }

接下来这个才是真正恢复类型数据的函数

    void MethodTable::Restore()
    {
        //检查Class指针是否已经被恢复了(在DoRestoreTypeKey中)
        CONSISTENCY_CHECK(IsClassPointerValid());
        //如果这个MethodTable本身不是代表性MethodTable，就恢复代表性MethodTable
        //我们会把代表性MethodTable在LoadExactParents函数中加载到EXACTPARENTS级别
        if (!IsCanonicalMethodTable())
        {
            ClassLoader::EnsureLoaded(GetCanonicalMethodTable(), CLASS_LOAD_APPROXPARENTS);
        }
        //恢复父类的MethodTable
        if (IsParentMethodTableIndirectPointerMaybeNull())
        {
            Module::RestoreMethodTablePointerRaw(GetParentMethodTableValuePtr(), GetLoaderModule(), CLASS_LOAD_APPROXPARENTS);
        }
        else
        {
            ClassLoader::EnsureLoaded(ReadPointer(this, &MethodTable::m_pParentMethodTable, GetFlagHasIndirectParent()),
                                    CLASS_LOAD_APPROXPARENTS);
        }
        //然后恢复Inteface
        InterfaceMapIterator it = IterateInterfaceMap();
        while (it.Next())
        {
            //只需要保证Interface处在Approximate级别即可
            //LoadExactParents稍后会把实际的Interface类型填入
            MethodTable * pIftMT;
            pIftMT = it.GetInterfaceInfo()->GetApproxMethodTable(GetLoaderModule());
            _ASSERTE(pIftMT != NULL);
        }
        //如果有跨Module的泛型信息，恢复之
        if (HasCrossModuleGenericStaticsInfo())
        {
            MethodTableWriteableData * pWriteableData = GetWriteableDataForWrite();
            CrossModuleGenericsStaticsInfo * pInfo = pWriteableData->GetCrossModuleGenericsStaticsInfo();
            EnsureWritablePages(pWriteableData, sizeof(MethodTableWriteableData) + sizeof(CrossModuleGenericsStaticsInfo));
            pInfo->m_pModuleForStatics = GetLoaderModule();
        }
        //最后设置恢复标志
        SetIsRestored();
    }

    inline BOOL IsRestored_NoLogging()
    {
        //如果我们做了提前的恢复，就返回true
        //IsPreRestored在已经过JIT的代码里总是为false
        if (IsPreRestored())
            return TRUE;
        return !(GetWriteableData_NoLogging()->m_dwFlags & MethodTableWriteableData::enum_flag_Unrestored);
    }

下面这个版本只是少了Log的读写

    inline BOOL IsRestored()
    {
        if (IsPreRestored())
            return TRUE;
        return !(GetWriteableData()->m_dwFlags & MethodTableWriteableData::enum_flag_Unrestored);
    }

#### 加载级别
    * MethodTable加载级别是从各种各样的Flag比特位派生来的
      详细参见classloadlevel.h
    * CLASS_LOADED级别(即完全加载)非常特殊: 当且仅当这个类型
      和所依赖的类型(泛型参数，父类，接口)都处于这个级别时才能
      被称为完全加载
    * 完全加载一个类型到这个级别是完全在锁外进行的，因此需要一个
      原子操作来设置这个级别

    inline void SetIsFullyLoaded()
    {
        //Parent不应当还在Approximate级别
        PRECONDITION(!HasApproxParent());
        //已经恢复
        PRECONDITION(IsRestored_NoLogging());
        //原子操作设置
        FastInterlockAnd(EnsureWritablePages(&GetWriteableDataForWrite()->m_dwFlags), ~MethodTableWriteableData::enum_flag_IsNotFullyLoaded);
    }


    //与GetLoadLevel() == CLASS_LOADED等效
    inline BOOL IsFullyLoaded() 
    {
        return (IsPreRestored())
            || (GetWriteableData()->m_dwFlags & MethodTableWriteableData::enum_flag_IsNotFullyLoaded) == 0;
    }

以下两个函数设置或者指示是否跳过WinRT重载

    inline BOOL IsSkipWinRTOverride()
    {
        return (GetWriteableData_NoLogging()->m_dwFlags & MethodTableWriteableData::enum_flag_SkipWinRTOverride);
    }
    
    inline void SetSkipWinRTOverride()
    {
        FastInterlockOr(EnsureWritablePages(&GetWriteableDataForWrite_NoLogging()->m_dwFlags), MethodTableWriteableData::enum_flag_SkipWinRTOverride);
    }

接着四个函数是有关于Equals比较和GetHashCode的Flag设置

    inline BOOL CanCompareBitsOrUseFastGetHashCode()
    {
        return (GetWriteableData_NoLogging()->m_dwFlags & MethodTableWriteableData::enum_flag_CanCompareBitsOrUseFastGetHashCode);
    }
    //如果上面的函数返回true，这个函数将会保证使用原子操作设置
    //enum_flag_HasCheckedCanCompareBitsOrUseFastGetHashCode与
    //enum_flag_CanCompareBitsOrUseFastGetHashCode这两个flag
    inline void SetCanCompareBitsOrUseFastGetHashCode(BOOL canCompare)
    {
        if (canCompare)
        {
            FastInterlockOr(EnsureWritablePages(&GetWriteableDataForWrite_NoLogging()->m_dwFlags),
                MethodTableWriteableData::enum_flag_HasCheckedCanCompareBitsOrUseFastGetHashCode | MethodTableWriteableData::enum_flag_CanCompareBitsOrUseFastGetHashCode);
        }
        else
        {
            SetHasCheckedCanCompareBitsOrUseFastGetHashCode();
        }
    }

    inline BOOL HasCheckedCanCompareBitsOrUseFastGetHashCode()
    {
        return (GetWriteableData_NoLogging()->m_dwFlags & MethodTableWriteableData::enum_flag_HasCheckedCanCompareBitsOrUseFastGetHashCode);
    }

    inline void SetHasCheckedCanCompareBitsOrUseFastGetHashCode()
    {
        FastInterlockOr(EnsureWritablePages(&GetWriteableDataForWrite_NoLogging()->m_dwFlags), MethodTableWriteableData::enum_flag_HasCheckedCanCompareBitsOrUseFastGetHashCode);
    }

    //设置所有依赖项是否被加载
    inline void SetIsDependenciesLoaded()
    {
        PRECONDITION(!HasApproxParent());
        PRECONDITION(IsRestored_NoLogging());
        FastInterlockOr(EnsureWritablePages(&GetWriteableDataForWrite()->m_dwFlags), MethodTableWriteableData::enum_flag_DependenciesLoaded);
    }

    //获取MethodTable的加载级别
    inline ClassLoadLevel GetLoadLevel()
    {
        //对于已经激活的Iamge，直接返回完全加载
        if (IsPreRestored())
            return CLASS_LOADED;
        //否则逐个检查级别
        DWORD dwFlags = GetWriteableData()->m_dwFlags;
        if (dwFlags & MethodTableWriteableData::enum_flag_IsNotFullyLoaded)
        {
            if (dwFlags & MethodTableWriteableData::enum_flag_UnrestoredTypeKey)
                return CLASS_LOAD_UNRESTOREDTYPEKEY;
            if (dwFlags & MethodTableWriteableData::enum_flag_Unrestored)
                return CLASS_LOAD_UNRESTORED;
            if (dwFlags & MethodTableWriteableData::enum_flag_HasApproxParent)
                return CLASS_LOAD_APPROXPARENTS;
            if (!(dwFlags & MethodTableWriteableData::enum_flag_DependenciesLoaded))
                return CLASS_LOAD_EXACTPARENTS;
            return CLASS_DEPENDENCIES_LOADED;
        }
        return CLASS_LOADED;
    }

本模块最后一个函数，也是最为重要的函数，对MethodTable做完全加载。其篇幅会比较长，且涉及方面众多。
    
    * 此函数将递归解析的类型提升到目标的级别，其中类型的依赖包括
      1.父类
      2.接口
      3.代表性类型
      4.经典类型

    * 参数详解
      1.pVisited 用于检测循环依赖
      2.level 用于指示提升到的目标加载等级，其取值只能是
        CLASS_DEPENDENCIES_LOADED 或者 CLASS_LOADED
        - 如果是CLASS_DEPENDENCIES_LOADED，所有的依赖将会被解析到
          具体的类型
        - 如果是CLASS_LOADED，将会对此类型和所有依赖类型进行类型安全检查
          需要注意的是，对于CLASS_LOADED的情况，一些类型可能在循环依赖下
          会被留在Pending List里面而不是被直接推上CLASS_LOADED类型加载级别。
          因此起始调用者必须处理这个情况
      3.pfBailed 如果其中一个依赖因为循环依赖而提早进入保释列表，我们必须设置
        其为true. 否则，我们必须保持其不变(因此此bool变量就像累计或一样)
      4.pPending 如果其中一个依赖被保释，此类型还不能被提升到CLASS_LOADED级别
        因为之后我们会进行类型安全检查，并且可能会失败.
        因此，DoFullyLoad会把类型加入Pending List.起始的调用者在完整的类型闭包
        被遍历后将提升滞留在Pending List的类型。
        需要注意的是，总是把类型推迟到Pending List上是正确的，但是性能会变得很糟糕。
      5.pInstContext 在SigPointer.GetTypeHandleThrowing中创建的实例化上下文并且
        最后将流向TypeVarTypeDesc.SatisfiesConstraints

    void MethodTable::DoFullyLoad(Generics::RecursionGraph * const pVisited,  const ClassLoadLevel level, DFLPendingList * const pPending,
                              BOOL * const pfBailed, const InstantiationContext * const pInstContext)
    {
        //需要Class已经完全加载或者其依赖项已经完全加载
        _ASSERTE(level == CLASS_LOADED || level == CLASS_DEPENDENCIES_LOADED);
        //返回是否已经保释状态的指针不能为空
        _ASSERTE(pfBailed != NULL);
        _ASSERTE(!(level == CLASS_LOADED && pPending == NULL));
        //如果循环图里已经存在此类型，就直接返回
        if (Generics::RecursionGraph::HasSeenType(pVisited, TypeHandle(this)))
        {
            *pfBailed = TRUE;
            return;
        }
        //如果当前加载级别大于等于期望的级别，直接返回
        if (GetLoadLevel() >= level)
        {
            return;
        }
        //如果已经完全加载
        if (level == CLASS_LOADED)
        {
            //遍历PendingList，看自己是否已经被"保释"
            UINT numTH = pPending->Count();
            TypeHandle *pTypeHndPending = pPending->Table();
            for (UINT idxPending = 0; idxPending < numTH; idxPending++)
            {
                if (pTypeHndPending[idxPending] == this)
                {
                    //已经被保释了直接返回
                    *pfBailed = TRUE;
                    return;
                }
            }
        }
        //首先确保我们的加载级别在CLASS_DEPENDENCIES_LOADED之下
        ClassLoader::EnsureLoaded(this, (ClassLoadLevel) (level-1));
        //前提条件一致性检查
        CONSISTENCY_CHECK(IsRestored_NoLogging());
        CONSISTENCY_CHECK(!HasApproxParent());
        //声明CheckForEquivalenceAndFullyLoadType所需要的数据结构，以及需要使用的变量
        //这样整个算法就只使用这一个被用于函数参数类型等效性遍历的结构对象
        DoFullyLoadLocals locals(pPending, level, this, pVisited);
        //是否需要纯净性检查，已经被激活的类早已做过检查
        bool fNeedsSanityChecks = !IsZapped();
    #ifdef FEATURE_READYTORUN
        if (fNeedsSanityChecks)
        {
            Module * pModule = GetModule();
            //对于ReadyToRun编译的映像，如果没有必要，就不进行检查
            if (pModule->IsSystem() || (pModule->IsReadyToRun() && pModule->GetReadyToRunInfo()->SkipTypeValidation()))
                fNeedsSanityChecks = false;
        }
    #endif
        //是否需要访问检查
        bool fNeedAccessChecks = (level == CLASS_LOADED) &&
                             fNeedsSanityChecks &&
                             IsTypicalTypeDefinition();
        TypeHandle typicalTypeHnd;
        //如果不是过NGen的类，就需要做有效性检查
        if (!IsZapped())
        {
            //完全加载代表性(Typical，Canonical)类型的实例，确保在加载其余依赖之前完成此工作
            //因为递归泛型探测算法需要检验在闭包中所有类型的代表性实例
            if (!IsTypicalTypeDefinition())
            {
                //进行类型加载
                typicalTypeHnd = ClassLoader::LoadTypeDefThrowing(GetModule(), GetCl(),
                ClassLoader::ThrowIfNotFound, ClassLoader::PermitUninstDefOrRef, tdNoTypes,
                (ClassLoadLevel) (level - 1));
                //检查是否加载成功
                CONSISTENCY_CHECK(!typicalTypeHnd.IsNull());
                //进行完全加载
                typicalTypeHnd.DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, pInstContext);
            }
            else if (level == CLASS_DEPENDENCIES_LOADED && HasInstantiation())
            {
                //* 走到这里表示这是一个泛型类型的代表性实例
                //  在尝试到达CLASS_DEPENDENCIES_LOADED级别之前，递归继承图
                //  (参见 ECMA CLI标准中 Part.II Section 9.2)会被构造用于检查
                //  “扩展环”以探测无限递归，比如 A<T>:B<A<A<T>>>
                //* 被此函数加载的依赖确保我们会生成ECMA中定义的有限的实例闭包
                //  此加载级别在加锁情况下无法到达，因此使用TypeVarTypeDesc来表示
                //  图的节点是不可能的，因为多个线程同时想要从闭包中将类型提升到
                //  完全加载级别会干扰到其余线程。除此之外，此图对象只用于类型加载
                //  在完全加载之后，对象便可以被抛弃掉
                //* 图对象是由Generics::RecursionGraph实例表示的，其使用多个链表
                //  来表示图，每个链表拥有图的一部分
                //  并且其存在于栈上，在DoFullyLoad返回之前会被自动地清理掉
                if (locals.newVisited.CheckForIllegalRecursion())
                {
                    //如果已经探测到扩展循环，那么这个类型会成为闭包的一部分
                    IMDInternalImport* pInternalImport = GetModule()->GetMDImport();
                    GetModule()->GetAssembly()->ThrowTypeLoadException(pInternalImport, GetCl(), IDS_CLASSLOAD_GENERICTYPE_RECURSIVE);
                }
            }
        }
        //对父类型进行完全加载
        MethodTable *pParentMT = GetParentMethodTable();
        if (pParentMT)
        {
            //完全加载
            pParentMT->DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, pInstContext);
            //需要访问检查
            if (fNeedAccessChecks)
            {
                //* RCW是需要特殊处理的，其是被Runtime生成的
                //  并且继承自非公开类型 System.__ComObject
                //* RCW: Runtime Callable Wrapper，是CLR运行时
                //  用于包装COM对象的一个包裹器
                if (!IsComObjectType())
                {
                    //透明(Transparent)类型不应当从关键(Critical)类型继承
                    //然而在有很多类违反此条例之前，此规则未被强制化
                    //如果现在开始强制此规则的话，将会是一个重大的变化
                    //PS:这里提到的Transparent和Critical都是.NET 4引进的安全机制
                    //所用词汇，这个话题会考虑在后面为大家介绍
                    DoAccessibilityCheck(this, pParentMT, E_ACCESSDENIED);
                }
            }
        }
        //对Interface做完全加载
        MethodTable::InterfaceMapIterator it = IterateInterfaceMap();
        //遍历实现的Interface
        while (it.Next())
        {
            //逐个进行完全加载
            it.GetInterface()->DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, pInstContext);
            if (fNeedAccessChecks)
            {
                //仅仅测试直接实现的接口
                //对一个继承的接口访问性为Private是合法的
                if (IsInterfaceDeclaredOnClass(it.GetIndex()))
                {      
                    //这里的注释同对父类的加载
                    DoAccessibilityCheck(this, it.GetInterface(), IDS_CLASSLOAD_INTERFACE_NO_ACCESS);
                }
            }
        }
        //完全加载泛型参数
        Instantiation inst = GetInstantiation();
        for (DWORD i = 0; i < inst.GetNumArgs(); i++)
        {
            inst[i].DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, pInstContext);
        }
        //完全加载代表性MethodTable
        if (!IsCanonicalMethodTable())
        {
            GetCanonicalMethodTable()->DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, NULL);
        }
        //纯净性检查
        if (fNeedsSanityChecks)
        {
            //完全加载ValueType字段的类型
            //需要注意的是，MethodTableBuilder::InitializeFieldDescs()仅仅将
            //类型提升到CLASS_LOAD_APPROXPARENTS级别
            FieldDesc *pField = GetApproxFieldDescListRaw();
            FieldDesc *pFieldEnd = pField + GetNumStaticFields() + GetNumIntroducedInstanceFields();
            while (pField < pFieldEnd)
            {
                g_IBCLogger.LogFieldDescsAccess(pField);
                //如果是VauleType，完全加载
                if (pField->GetFieldType() == ELEMENT_TYPE_VALUETYPE)
                {
                    TypeHandle th = pField->GetFieldTypeHandleThrowing((ClassLoadLevel) (level - 1));
                    CONSISTENCY_CHECK(!th.IsNull());
                    th.DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, pInstContext);
                    if (fNeedAccessChecks)
                    {
                        DoAccessibilityCheck(this, th.GetMethodTable(), E_ACCESSDENIED);
                    }
                }
                pField++;
            }
            //完全加载泛型ValueType字段类型
            if (HasGenericsStaticsInfo())
            {
                FieldDesc *pGenStaticField = GetGenericsStaticFieldDescs();
                FieldDesc *pGenStaticFieldEnd = pGenStaticField + GetNumStaticFields();
                while (pGenStaticField < pGenStaticFieldEnd)
                {
                    if (pGenStaticField->GetFieldType() == ELEMENT_TYPE_VALUETYPE)
                    {
                        TypeHandle th = pGenStaticField->GetFieldTypeHandleThrowing((ClassLoadLevel) (level - 1));
                        CONSISTENCY_CHECK(!th.IsNull());
                        th.DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, pInstContext);
                        //对泛型字段的检查是不必要的，因为泛型字段只是常规字段
                        //的一份拷贝而已，唯一不同的是后者有具体的类型
                    }
                    pGenStaticField++;
                }
            }
        }
        //如果启用了本机映像生成
    #ifdef FEATURE_NATIVE_IMAGE_GENERATION
        //此类型已经有明确排布，并且处于编译Domain中，且未被激活
        if (HasLayout() && GetAppDomain()->IsCompilationDomain() && !IsZapped())
        {
            //获取字段封送对象
            FieldMarshaler* pFM                   = this->GetLayoutInfo()->GetFieldMarshalers();
            //获取调用时封送具有变化的字段个数
            UINT  numReferenceFields              = this->GetLayoutInfo()->GetNumCTMFields();
            //遍历
            while (numReferenceFields--)
            {
                FieldDesc *pMarshalerField = pFM->GetFieldDesc();
                //如果FieldDescription是Token-Tagged指针，那么我们正在
                //处理的字段封送对象将不需要被保存到此NGen映像中
                //那正就是我们需要加载此对象的原因，因此，我们不需要完全加载
                //与此字段有关的类型
                if (!CORCOMPILE_IS_POINTER_TAGGED(pMarshalerField))
                {
                    TypeHandle th = pMarshalerField->GetFieldTypeHandleThrowing((ClassLoadLevel) (level-1));
                    CONSISTENCY_CHECK(!th.IsNull());         
                    th.DoFullyLoad(&locals.newVisited, level, pPending, &locals.fBailed, pInstContext);
                }
                //访问性检查在此处不会进行，因为某些情况下NGen与非NGen有功能性差异
                ((BYTE*&)pFM) += MAXFIELDMARSHALERSIZE;
            }
        }
    #endif
        //* 在PreStub期间，假如GC被触发，则会完全加载参与类型等效性的ValueType参数类
        //  型。GC需要知道栈上的引用在哪，从函数签名获得的参数是否为一个结构体，      
        //  这依赖于已经加载的类型的排布来获取信息。对于一般性的结构体，我们保证
        //  在进入PreStub前就完全加载类型-调用者一定已经加载了它
        //  然而，因为类型等效性，调用者可能使用了与函数签名所不同的类型。
        //* 我们通过主动加载可能导致此问题的类型(比如由此类型引入的在函数签名
        //  中的ValueType)
        //  为了避免没有类型等效情况下的性能下降，我们只提前加载被标记为类型等效
        //  的结构体
        //  在没有PIA的世界里面，这些结构体被叫做本地类型并且通常用编译器自动生成
        //  需要注意的是，这里实际上有相关的逻辑性存在于
        //  CompareTypeDefsForEquivalence函数中，其声明两个Token来对应基于扩展
        //  的等效性检查的结构体
        //  PS: PIA:Primary Interop Assemblies
        //  Interop Assemblies是COM类型库的对应托管版本Assembly
        //  而Primary Interop Assemblies总是由发行方在原有非托管库中进行签名
        //* 为了解决这种情况下已经NGen的类型和函数，我们阻止其被提前储存-参见
        //  ComputeNeedsRestoreWorker。这会强制它们在运行时加载走到最后一步
        //  并且遇见下面的代码
        if ((level == CLASS_LOADED) 
        && (GetCl() != mdTypeDefNil) 
        && !ContainsGenericVariables() 
        && (!IsZapped() 
        || DependsOnEquivalentOrForwardedStructs()
    #ifdef DEBUG
            || TRUE //在Debug时总是加载类型以便于我们随时可以计算
            //fDependsOnEquivalentOrForwardedStructs
    #endif
            )
        )
        {
            //遍历引入的函数
            MethodTable::IntroducedMethodIterator itMethods(this, FALSE);
            for (; itMethods.IsValid(); itMethods.Next())
            {
                MethodDesc * pMD = itMethods.GetMethodDesc();
                //如果是编译进程
                if (IsCompilationProcess())
                {
                    locals.fHasTypeForwarderDependentStructParameter = FALSE;
                    EX_TRY
                    {
                        //逐个检查
                        pMD->WalkValueTypeParameters(this, CheckForTypeForwardedTypeRefParameter, &locals);
                    }
                    EX_CATCH
                    {
                    }
                    EX_END_CATCH(RethrowTerminalExceptions);
                }
                else if (pMD->IsZapped() && pMD->HasForwardedValuetypeParameter())
                {
                    //如果此模块被激活且参数中含有转发的ValueType
                    //遍历并且加载引用程序集或者类型
                    pMD->WalkValueTypeParameters(this, LoadTypeDefOrRefAssembly, NULL);
                    locals.fDependsOnEquivalentOrForwardedStructs = TRUE;
                }
            #ifdef FEATURE_TYPEEQUIVALENCE
                //启用了类型等效
                if (!pMD->DoesNotHaveEquivalentValuetypeParameters() && pMD->IsVirtual())
                {
                    //如果有等效ValueType类型的参数并且是虚函数
                    locals.fHasEquivalentStructParameter = FALSE;
                    //遍历并且完全加载类型
                    pMD->WalkValueTypeParameters(this, CheckForEquivalenceAndFullyLoadType, &locals);
                    //处理后设置没有被激活的映像
                    if (!locals.fHasEquivalentStructParameter && !IsZapped())
                        pMD->SetDoesNotHaveEquivalentValuetypeParameters();
                }
            #else
            #ifdef FEATURE_PREJIT
                //没有启用类型等效且启用PreJIT功能
                if (!IsZapped() && pMD->IsVirtual() && !IsCompilationProcess() )
                {
                    //未被激活，且是虚函数，当前不是编译器进程
                    //准备作为本机映像的依赖
                    pMD->PrepareForUseAsADependencyOfANativeImage();
                }
            #endif
            #endif
            }
        }
        //断言检查
        _ASSERTE(!IsZapped() || !IsCanonicalMethodTable() || (level != CLASS_LOADED) || ((!!locals.fDependsOnEquivalentOrForwardedStructs) == (!!DependsOnEquivalentOrForwardedStructs())));
        if (locals.fDependsOnEquivalentOrForwardedStructs)
        {
            if (!IsZapped())
            {     
                //如果此类型声明了一个有等效类型或者转发的结构体参数
                //确保我们到这里并且在NGen的情况下也要提前加载这些结构体类型
                SetDependsOnEquivalentOrForwardedStructs();
            }
        }
        //泛型的约束环检查与访问性检查是一致的
        if (fNeedAccessChecks)
        {
            //检查环形类约束
            {
                //遍历所有参数
                Instantiation formalParams = GetInstantiation();
                for (DWORD i = 0; i < formalParams.GetNumArgs(); i++)
                {
                    BOOL Bounded(TypeVarTypeDesc *tyvar, DWORD depth);
                    TypeVarTypeDesc *pTyVar = formalParams[i].AsGenericVariable();
                pTyVar->LoadConstraints(CLASS_DEPENDENCIES_LOADED);
                    if (!Bounded(pTyVar, formalParams.GetNumArgs()))
                    {
                        COMPlusThrow(kTypeLoadException, VER_E_CIRCULAR_VAR_CONSTRAINTS);
                    }
                    DoAccessibilityCheckForConstraints(this, pTyVar, E_ACCESSDENIED);
                }
            }
            //检查方法环形约束
            {
                //确保是元数据类型
                if (GetCl() != mdTypeDefNil) 
                {
                    //遍历MethodTable逐个检查
                    MethodTable::IntroducedMethodIterator itMethods(this, FALSE);
                    for (; itMethods.IsValid(); itMethods.Next())
                    {
                        MethodDesc * pMD = itMethods.GetMethodDesc();
                    
                        if (pMD->IsGenericMethodDefinition() && pMD->IsTypicalMethodDefinition())
                        {
                            BOOL fHasCircularClassConstraints = TRUE;
                            BOOL fHasCircularMethodConstraints = TRUE;
                        
                            pMD->LoadConstraintsForTypicalMethodDefinition(&fHasCircularClassConstraints, &fHasCircularMethodConstraints, CLASS_DEPENDENCIES_LOADED);
                        
                            if (fHasCircularClassConstraints)
                            {
                                COMPlusThrow(kTypeLoadException, VER_E_CIRCULAR_VAR_CONSTRAINTS);
                            }
                            if (fHasCircularMethodConstraints)
                            {
                                COMPlusThrow(kTypeLoadException, VER_E_CIRCULAR_MVAR_CONSTRAINTS);
                            }
                        }
                    }
                }
            }
        }
        switch (level)
        {
            case CLASS_DEPENDENCIES_LOADED:
                //设置加载级别
                SetIsDependenciesLoaded();
        #if defined(FEATURE_COMINTEROP) && !defined(DACCESS_COMPILE)
            //如果支持WinRT，就基于GUID做类型的缓存
            if (WinRTSupported() && g_fEEStarted)
            {
                _ASSERTE(GetAppDomain() != NULL);
                AppDomain* pAppDomain = GetAppDomain();
                if (pAppDomain->CanCacheWinRTTypeByGuid(this))
                {
                    pAppDomain->CacheWinRTTypeByGuid(this);
                }
            }
        #endif
            break;
            case CLASS_LOADED:
                //检查NGen是否已经做过这部分工作了
                if (!IsZapped() && 
                !IsTypicalTypeDefinition() &&
                !IsSharedByGenericInstantiations())
                {
                    TypeHandle thThis = TypeHandle(this);
                    //如果我们到达了这里，我们即将标记一个泛型的实例为完全加载
                    //在我们做之前，检查是否有未满足的约束
                    SatisfiesClassConstraints(thThis, typicalTypeHnd, pInstContext);
                }
                //如果已经保释
                if (locals.fBailed)
                {
                    //我们不能对某些依赖完成安全检查因为它已经在被我们其余的调用者
                    //之一处理。
                    //不要将其标记为完全加载，把他放到PendingList上，当一切准备就绪
                    //时，其会被标记为完全加载
                    *pfBailed = TRUE;
                    TypeHandle *pTHPending = pPending->AppendThrowing();
                    *pTHPending = TypeHandle(this);
                }
                else
                {
                    //最终，标记为完全加载
                    SetIsFullyLoaded();
                }
                break;
            default:
                //此处无法到达
                _ASSERTE(!"Can't get here.");
                break;
        }
        if (level >= CLASS_DEPENDENCIES_LOADED && IsArray())
        {
            //如果模板MethodTable被加载了，那么数组类型也应当被加载
            //参见ArrayBase::SetArrayMethodTable, 
            //ArrayBase::SetArrayMethodTableForLargeObject
            TypeHandle th = ClassLoader::LoadArrayTypeThrowing(GetApproxArrayElementTypeHandle(),
            GetInternalCorElementType(),
            GetRank(),
            ClassLoader::LoadTypes,
            level);
            _ASSERTE(th.IsTypeDesc() && th.IsArray());
            _ASSERTE(!(level == CLASS_LOADED && !th.IsFullyLoaded()));
        }
    }

# 未完待续...

啊，最近因为学校的实验，国庆（我在假期绝不做工作，除非无聊至极）等事拖延了那么久，Part 1的解析就到此结束了。

# Part 2预告

    1. MethodTable作为类型的描述符
    2. 泛型与代码共享
    3. 通过Slot索引访问函数
    4. Virtual Table
    5. MethodDescription与Slot
    6. 封箱的EntryPoint MethodDescription




