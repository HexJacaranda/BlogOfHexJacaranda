# MethodTable解析 Part 2

#### MethodTable作为类型的描述符
这一部分有一些函数过于简单，就不做解释了。
    
    inline BOOL IsInterface();
    void SetIsInterface()

接下来两个函数是通过转发到EEClass对象获取数据

    inline BOOL IsSealed();
    inline BOOL IsAbstract();

    //是否对外可见
    BOOL IsExternallyVisible();

    //此函数获取已经实例化泛型类型的参数列表
    //比如Dictionary<string,int>会返回{string,int}
    //对于没有实例化的类型，返回null
    Instantiation GetInstantiation()
    {
        //如果有实例化
        if (HasInstantiation())
        {
            //获取泛型字典信息
            PTR_GenericsDictInfo  pDictInfo = GetGenericsDictInfo();
            TADDR base = dac_cast<TADDR>(&(GetPerInstInfo()[pDictInfo->m_wNumDicts-1]));
            return Instantiation(PerInstInfoElem_t::GetValueMaybeNullAtPtr(base)->GetInstantiation(), pDictInfo->m_wNumTyPars);
        }
        else
        {
            return Instantiation();
        }
    }

以下两个函数获取Array的实例化

    Instantiation GetClassOrArrayInstantiation()
    {
        //如果是Array，调用专有函数
        if (IsArray()) {
            return GetArrayInstantiation();
        }
        else {
            return GetInstantiation();
        }
    }
    Instantiation GetArrayInstantiation()
    {
        //断言是否为数组
        _ASSERTE(IsArray());
        //返回元素类型
        return Instantiation((TypeHandle *)&m_ElementTypeHnd, 1);
    }

嘿！你可能会想，为什么数组要特殊处理，不都是泛型么？这实际上也是我被C++\CLI语法所蒙骗，其中原生数组声明是这样的：

    array<ManagedType^>^

害得我以为数组也是泛型实现，其实不然，当我尝试在C#中对一个数组类型获取GenericArguments时，我就知道，我被骗了。

    //指示这个类型是否需要其余的Module被加载
    BOOL HasModuleDependencies()
    {
        return GetFlag(enum_flag_HasModuleDependencies);
    }

    //设置上面的标志
    void SetHasModuleDependencies();

    //关于内建类型的信息
    inline BOOL IsIntrinsicType();
    inline void SetIsIntrinsicType();

    //DoFullyLoad所需要的依赖关系
    inline BOOL DependsOnEquivalentOrForwardedStructs();
    inline void SetDependsOnEquivalentOrForwardedStructs();

    //是否有实例化
    inline BOOL HasInstantiation();

    //对于任意要么自己是泛型实现或者继承链中含有泛型实现的类型返回true
    //比如：class D:C<int>
    //或者:class E:D, class D:C<int>
    //需要注意的是，泛型接口不在此列
    BOOL HasGenericClassInstantiationInHierarchy()
    {
        return GetNumDicts() != 0;
    }

    //指示此类型是否为一个泛型定义类，比如List<T>
    inline BOOL IsGenericTypeDefinition();

    //是否含有泛型方法变量
    BOOL ContainsGenericMethodVariables()
    {
        //遍历泛型参数
        Instantiation inst = GetInstantiation();
        for (DWORD i = 0; i < inst.GetNumArgs(); i++)
        {
            CONSISTENCY_CHECK(!inst[i].IsEncodedFixup());
            if (inst[i].ContainsGenericVariables(TRUE))
            return TRUE;
        }
    }

    //是否有泛型变量
    inline void SetContainsGenericVariables()

    //泛型协变，逆变信息
    inline void SetHasVariance()
    inline BOOL HasVariance()

    //指示这是否为一个像List<T>或者List<Stack<T>>的类型
    //实际上List<Some<T>>只存在于反射和验证中
    inline DWORD ContainsGenericVariables(BOOL methodVarsOnly = FALSE)
    {
        if (methodVarsOnly)
            return ContainsGenericMethodVariables();
        else
            return GetFlag(enum_flag_ContainsGenericVariables);
    }

    //是否为一个ref struct
    BOOL IsByRefLike()
    void SetIsByRefLike()

    //此类型是一个Com对象类型
    //获取开放类型定义的Module
    Module* GetDefiningModuleForOpenType()
    {
        //如果有泛型参数
        if (ContainsGenericVariables())
        {
            //遍历参数
            Instantiation inst = GetInstantiation();
            for (DWORD i = 0; i < inst.GetNumArgs(); i++)
            {
                //已编码修复永远都不是开放类型
                if (!inst[i].IsEncodedFixup())
                {
                    Module *pModule = inst[i].GetDefiningModuleForOpenType();
                    if (pModule != NULL)
                    RETURN pModule;
                }
            }
        }
        RETURN NULL;
    }

    //是否为一个典型(代表性)类型
    inline BOOL IsTypicalTypeDefinition()       
    {
        return !HasInstantiation() || IsGenericTypeDefinition();
    }

接下来一点点就是关于WinRT类型映射和COM重定向的了

    //这是否为一个可以被用于COM交互的泛型接口或者委托?
    inline BOOL SupportsGenericInterop(TypeHandle::InteropKind interopKind, Mode = modeAll)
    {
    #ifdef FEATURE_COMINTEROP
        return ((IsInterface() || IsDelegate()) &&    // 接口或委托
                HasInstantiation() &&                 // 是泛型
                !IsSharedByGenericInstantiations() && // 没有共享部分
                !ContainsGenericVariables() &&        // 是一个封闭类型
                //在 .winmd 中定义或者是 mscrolib 中被重定向的接口
                ((((mode & modeProjected) != 0) && IsProjectedFromWinRT()) ||
                (((mode & modeRedirected) != 0) && (IsWinRTRedirectedInterface(interopKind) || IsWinRTRedirectedDelegate()))));
    #else
        return FALSE
    #endif
    }

    //检查是否与目标MethodTable拥有相同类型定义(包括泛型实例化后原本的泛型是否一致)
    BOOL HasSameTypeDefAs(MethodTable *pMT)
    {
        if (this == pMT)
            return TRUE;
        //优化Rid不匹配的情况
        if (GetTypeDefRid() != pMT->GetTypeDefRid())
            return FALSE;
        //比较代表性MethodTable
        if (GetCanonicalMethodTable() == pMT->GetCanonicalMethodTable())
            return TRUE;
        //比较模块
        return (GetModule() == pMT->GetModule());
    }

#### 泛型与代码共享

    //是否被泛型实例化共享
    BOOL IsSharedByGenericInstantiations()
    {
        return TestFlagWithMask(enum_flag_GenericsMask, enum_flag_GenericsMask_SharedInst);
    }

    //指示是否为一个代表性MethodTable，如果是非泛型，也返回true
    inline BOOL IsCanonicalMethodTable()
    {
        return (union_getLowBits(ReadPointer(this, &MethodTable::m_pCanonMT)) == UNION_EECLASS);
    }

    //获取一众泛型实例化类型中的代表性MethodTable
    PTR_MethodTable GetCanonicalMethodTable()
    {
        TADDR addr = ReadPointer(this, &MethodTable::m_pCanonMT);
        //如果低位是10，则本对象就是代表性MethodTable
        if ((addr & 2) == 0)
            return dac_cast<PTR_MethodTable>(this);
    #ifdef FEATURE_PREJIT
        //低位是01，则是PreJIT储存在addr前三个的位置
        if ((addr & 1) != 0)
            return PTR_MethodTable(*PTR_TADDR(addr - 3));
    #endif
        //否则在前两个位置处
        return PTR_MethodTable(addr - 2);
    }

    //获取代表性MethodTable的修复版本，如果其需要修复，否则返回null
    TADDR GetCanonicalMethodTableFixup()
    {
    #ifdef FEATURE_PREJIT
        TADDR addr = ReadPointer(this, &MethodTable::m_pCanonMT);
        LowBits lowBits = union_getLowBits(addr);
        if (lowBits == UNION_INDIRECTION)
        {
            //获取修复的代表性MethodTable
            return *PTR_TADDR(union_getPointer(addr));
        }
        else
    #endif
        {
            return NULL;
        }
    }

#### 通过索引访问Slot

    有些函数现阶段基于他们是连续的假设来获取non-virtual方法
    实际上在泛型实例化中不总是成立，这些non-virtual方法存在于代表性MethodTable中

    enum
    {
        NO_SLOT = 0xffff //用于指示空Slot的数字
    };

    //获取Slot
    PCODE GetSlot(UINT32 slotNumber)
    {
        //检查Slot索引与最大数
        CONSISTENCY_CHECK(slotNumber < GetNumVtableSlots());
        //直接获取原始Slot
        TADDR pSlot = GetSlotPtrRaw(slotNumber);
        //如果Index处于VirtualTable Slot范围内，取出对应地址
        if (slotNumber < GetNumVirtuals())
        {
            return VTableIndir2_t::GetValueMaybeNullAtPtr(pSlot);
        }
        else if (IsZapped() && slotNumber >= GetNumVirtuals())
        {
            //在NGen模块中，是使用RelativePtr储存的
            return RelativePointer<PCODE>::GetValueAtPtr(pSlot);
        }
        //否则就是普通Slot，直接转换
        return *dac_cast<PTR_PCODE>(pSlot);
    }

    //针对我们已经知道这是一个对应Virtual Method的情况
    inline PCODE GetSlotForVirtual(UINT32 slotNum)
    {
        //检查是否处于Virtual Slot范围之内
        CONSISTENCY_CHECK(slotNum < GetNumVirtuals());
        //实际上Virtual Slot存在于Virtual Table Indirection所指向的一块内存中
        //获取Indirection索引
        DWORD index = GetIndexOfVtableIndirection(slotNum);
        //获取基址
        TADDR base = dac_cast<TADDR>(&(GetVtableIndirections()[index]));
        //跳过Indirection，获取真正的地址
        DPTR(VTableIndir2_t) baseAfterInd = VTableIndir_t::GetValueMaybeNullAtPtr(base) + GetIndexAfterVtableIndirection(slotNum);
        return VTableIndir2_t::GetValueMaybeNullAtPtr(dac_cast<TADDR>(baseAfterInd));
    }

    TADDR GetSlotPtrRaw(UINT32 slotNum)
    {
        //检查索引是否越界
        CONSISTENCY_CHECK(slotNum < GetNumVtableSlots());
        //虚函数，与上面的函数如出一辙
        if (slotNum < GetNumVirtuals())
        {          
            DWORD index = GetIndexOfVtableIndirection(slotNum);
            TADDR base = dac_cast<TADDR>(&(GetVtableIndirections()[index]));
            DPTR(VTableIndir2_t) baseAfterInd = VTableIndir_t::GetValueMaybeNullAtPtr(base) + GetIndexAfterVtableIndirection(slotNum);
            return dac_cast<TADDR>(baseAfterInd);
        }
        else if (HasSingleNonVirtualSlot())
        {
            //非虚函数Slot存在于Optional Member中所指向的一个内存块
            //但假如只有一个，那么这个Optional Member就储存此Slot
            _ASSERTE(slotNum == GetNumVirtuals());
            return GetNonVirtualSlotsPtr();
        }
        else
        {
            //否则就使用泛化版本，在Optional Member指向的内存块中获取
            _ASSERTE(HasNonVirtualSlotsArray());
            g_IBCLogger.LogMethodTableNonVirtualSlotsAccess(this);
            return dac_cast<TADDR>(GetNonVirtualSlotsArray() + (slotNum - GetNumVirtuals()));
        }
    }

    TADDR GetSlotPtr(UINT32 slotNum)
    {
        //不能是NGen过的函数，因为其必须返回RelativePointer
        CONSISTENCY_CHECK(!IsZapped());
        return GetSlotPtrRaw(slotNum);
    }        

    void SetSlot(UINT32 slotNum, PCODE slotVal)
    {
        //同样地，取出相应Slot，根据不同情况设置目标
        TADDR slot = GetSlotPtrRaw(slotNumber);
        if (slotNumber < GetNumVirtuals())
        {
            ((MethodTable::VTableIndir2_t *) slot)->SetValueMaybeNull(slotCode);
        }
        else
        {
            *((PCODE *)slot) = slotCode;
        }
    }

#### Virtual Table

    关于Virtual Table: 
    * 不同于传统上直接使用数组储存Slot，我们使用两级Virtual Table，其中虚函数存在
      于所指向的内存空间中。这样做的好处是，允许我们在不同MethodTable之间共享内存块
      (最常见的情形是，多个子类都没有重写父类的函数)，这样使得我们会省下虚函数所需要
      的固定开销，一个Indirection。
    * 需要注意的是，这些细节不应当对MethodTable外部可见，其余代码仍然使用传统的Slot
      索引来访问虚函数。这与我们使用Slot索引访问非虚函数是相似的，尽管他们的代码指针在
      很早以前就被移出了Virtual Table

      考虑一个类，其GetNumVirtuals为5，假定我们把VirtualTable分成大小为3的块，其
      排布就会长成这样：
        MethodTable              块1                       块2
       ------------------        ------------------        ------------------
       |                |        |      M1()      |        |      M4()      |
       |   MethodTable  |        ------------------        ------------------
       |    固定成员     |        |      M2()      |        |      M5()      |
       |                |        ------------------        ------------------
       |                |        |      M3()      |
       ------------------        ------------------
       | 指向块1的指针   |
       ------------------
       | 指向块2的指针   |
       ------------------
       在这里，指向块1和块2的指针就被我们称作Indirection

    * 现有的分块策略是相对于类独立的，每个大小都是8。也曾尝试过其他的策略，
      但唯一一个比这个快一点的是以大小为4开头(刚好匹配System.Object的虚函数)
      后面的块都是8。然而这样提升不是很大，并且使得下方列出的运行时助手跑
      得更慢了。
    * 如果你想改变此策略，你应该修改以下列出的Assembly助手的前四个函数，因为他们
      依赖于Slot的排布信息。目前有：
        JIT_IsInstanceOfInterface
        JIT_ChkCastInterface
        Transparent proxy stub
    * 这个排布仅仅是针对虚函数的，非虚函数Slot被安排在Optional Member所储存的指针
      指向的空间里(参见GetSlotPtrRaw的注释)

    #define VTABLE_SLOTS_PER_CHUNK 8
    #define VTABLE_SLOTS_PER_CHUNK_LOG2 3

    static DWORD GetIndexOfVtableIndirection(DWORD slotNum)
    {
        //使用Slot索引直接除以8
        return slotNum >> VTABLE_SLOTS_PER_CHUNK_LOG2;
    }

    static DWORD GetStartSlotForVtableIndirection(UINT32 indirectionIndex, DWORD wNumVirtuals)
    {
        //反过来也是一样，直接乘以8
        return indirectionIndex * VTABLE_SLOTS_PER_CHUNK;
    }

    static DWORD GetEndSlotForVtableIndirection(UINT32 indirectionIndex, DWORD wNumVirtuals)
    {
        DWORD end = (indirectionIndex + 1) * VTABLE_SLOTS_PER_CHUNK;
        if (end > wNumVirtuals)
        {
            end = wNumVirtuals;
        }
        return end;
    }

    //获取Indirection后的索引
    static UINT32 GetIndexAfterVtableIndirection(UINT32 slotNum)
    {
        //留下不超过VTABLE_SLOTS_PER_CHUNK的数字，即为对应Indirection内索引
        return (slotNum & (VTABLE_SLOTS_PER_CHUNK - 1));
    }

    //获取Indirection的数量
    static DWORD GetNumVtableIndirections(DWORD wNumVirtuals)
    {
        return (wNumVirtuals + (VTABLE_SLOTS_PER_CHUNK - 1)) >> VTABLE_SLOTS_PER_CHUNK_LOG2;
    }

     //获取Indirection数组
     DPTR(VTableIndir_t) GetVtableIndirections()
     {
            return dac_cast<DPTR(VTableIndir_t)>(dac_cast<TADDR>(this) + sizeof(MethodTable));
     }
正如我们先前所看见的定义与排布，Indirection开始于MethodTable固定成员部分

    DWORD GetNumVtableIndirections()
    {
        return GetNumVtableIndirections(GetNumVirtuals_NoLogging());
    }

#### Virtual Table嵌套辅助类-VtableIndirectionSlotIterator

##### 字段
    DPTR(VTableIndir_t) m_pSlot
    DWORD m_i;
    DWORD m_count;
    PTR_MethodTable m_pMT;

##### 成员函数
    VtableIndirectionSlotIterator::VtableIndirectionSlotIterator
    VtableIndirectionSlotIterator::Next
    VtableIndirectionSlotIterator::Finished
    VtableIndirectionSlotIterator::GetIndex
    VtableIndirectionSlotIterator::GetOffsetFromMethodTable
    VtableIndirectionSlotIterator::GetIndirectionSlot
    VtableIndirectionSlotIterator::GetStartSlot
    VtableIndirectionSlotIterator::GetEndSlot
    VtableIndirectionSlotIterator::GetNumSlots
    VtableIndirectionSlotIterator::GetSize

嗯，乍一看相当有.Net本身设计迭代器的味道，而不是偏向于C++的指针式。让我们重点看一下迭代过程是怎么样的。

    BOOL Next()
    {
        //如果没有到最大值，Slot直接++
        if (m_i != (DWORD) -1)
            m_pSlot++;
        return (++m_i < m_count);
    }

    BOOL Finished()
    {
        return (m_i == m_count);
    }

所以整个迭代器都是按线性排布来访问Indirection的，与实际排布一致。

#### Virtual Table - 续
获取用于迭代的迭代器

    VtableIndirectionSlotIterator IterateVtableIndirectionSlots();
    VtableIndirectionSlotIterator IterateVtableIndirectionSlotsFrom(DWORD index);

下面就是关于分配策略中提到的Virtual Table块共享，分为了有PreJIT和没有的情况

    #ifdef FEATURE_PREJIT
        static BOOL CanShareVtableChunksFrom(MethodTable *pTargetMT, Module *pCurrentLoaderModule, Module *pCurrentPreferredZapModule);
        BOOL CanInternVtableChunk(DataImage *image, VtableIndirectionSlotIterator it);
    #else
        static BOOL CanShareVtableChunksFrom(MethodTable *pTargetMT, Module *pCurrentLoaderModule);
    #endif

    //PreJIT
    static BOOL CanShareVtableChunksFrom(MethodTable *pTargetMT, Module *pCurrentLoaderModule, Module *pCurrentPreferredZapModule)
    {
        //实际上共享的约束来自于两个方面
        //* 一个未激活的MethodTable不能与一个激活的MethodTable共享
        //  不然可能导致SetSlot函数在一个只读的Slot上被调用
        //* 在MethodTable::Save中激活此MethodTable不能取消共享某些我们
        //  现在决定要共享的数据
        //我们可以通过做以下的修补(fix)来使得未激活MethodTable完全共享分块
        //* 修复那些SetSlot调用函数，使得其一开始先检查分块所在MethodTable
        //  是否被激活(参见MethodTableBuilder::CopyExactParentSlot，或者
        //  我们可以使用ExecutionManager::FindZapModule)
        //* 如果进程是编译进程且依赖于MethodTable::Save来在NGen的情况下共享
        //  那么这个函数就返回false
        return !pTargetMT->IsZapped() &&
            pTargetMT->GetLoaderModule() == pCurrentLoaderModule &&
            pCurrentLoaderModule == pCurrentPreferredZapModule &&
            pCurrentPreferredZapModule == Module::GetPreferredZapModuleForMethodTable(pTargetMT);
    }

    //PreJIT
    BOOL MethodTable::CanInternVtableChunk(DataImage *image, VtableIndirectionSlotIterator it)
    {
        //断言编译进程
        _ASSERTE(IsCompilationProcess());
        BOOL canBeSharedWith = TRUE;
        //我们允许完全的共享，除了可能打破MethodTable::Fixup的情况
        //也就是，当Slot被修补后，我们需要确保无论是谁在做这个修补
        //工作，都必须是同一个目标。
        //如果这个条件未被达成，ZapStoredStructure::Save中的断言
        //将会被触发
        if (GetFlag(enum_flag_NotInPZM))
        {
            canBeSharedWith = FALSE;
        }
        //能够被共享，检查Slot具体情况
        if (canBeSharedWith)
        {
            for (DWORD slotNumber = it.GetStartSlot(); slotNumber < it.GetEndSlot(); slotNumber++)
            {
                MethodDesc *pMD = GetMethodDescForSlot(slotNumber);
                _ASSERTE(pMD != NULL);
                pMD->CheckRestore();
                //任何一个不能Bind，设置为false
                if (!image->CanEagerBindToMethodDesc(pMD))
                {
                    canBeSharedWith = FALSE;
                    break;
                }
            }
        }
        return canBeSharedWith;
    }

    //非PreJIT
    static BOOL CanShareVtableChunksFrom(MethodTable *pTargetMT, Module *pCurrentLoaderModule)
    {
        return pTargetMT->GetLoaderModule() == pCurrentLoaderModule;
    }

#### Non-Virtual Slots

下面这个几个就比较简单了，通过Flag取出

    inline BOOL HasNonVirtualSlots();
    inline BOOL HasSingleNonVirtualSlot();

不过这一个有点意思

    inline BOOL HasNonVirtualSlotsArray()
    {
        return HasNonVirtualSlots() && !HasSingleNonVirtualSlot();
    }

可以看得出，当有多于一个Non-Virtual Slot后，才会有数组的构建。

    TADDR GetNonVirtualSlotsPtr()
    {
        return GetMultipurposeSlotPtr(enum_flag_HasNonVirtualSlots, c_NonVirtualSlotsOffsets);
    }
    inline PTR_PCODE GetNonVirtualSlotsArray()
    {
        _ASSERTE(HasNonVirtualSlotsArray());        
        return RelativePointer<PTR_PCODE>::GetValueAtPtr(GetNonVirtualSlotsPtr());
    }

取出的操作也是对应其排布规则。
DAC下对应的设置函数

    #ifndef DACCESS_COMPILE
    inline void SetNonVirtualSlotsArray(PCODE *slots)
    {
        _ASSERTE(HasNonVirtualSlotsArray());
        RelativePointer<PCODE *> *pRelPtr = (RelativePointer<PCODE *> *)GetNonVirtualSlotsPtr();
        pRelPtr->SetValue(slots);
    }
    inline void SetHasSingleNonVirtualSlot()
    {
        SetFlag(enum_flag_HasSingleNonVirtualSlot);
    }
    #endif

    //获取数组Size
    inline unsigned GetNonVirtualSlotsArraySize()
    {
        return GetNumNonVirtualSlots() * sizeof(PCODE);
    }

    //获取普通函数Slot个数
    inline WORD GetNumNonVirtualSlots()
    {
        return HasNonVirtualSlots() ? GetClass()->GetNumNonVirtualSlots() : 0;
    }

    //虚函数个数
    inline WORD GetNumVirtuals()
    {
        g_IBCLogger.LogMethodTableAccess(this);
        return GetNumVirtuals_NoLogging();
    }
    inline WORD GetNumVirtuals_NoLogging()
    {
        return m_wNumVirtuals;
    }
    inline void SetNumVirtuals (WORD wNumVtableSlots)
    {
        m_wNumVirtuals = wNumVtableSlots;
    }

    //获取父类虚函数个数
    unsigned GetNumParentVirtuals()
    {
        //如果是接口，返回0
        if (IsInterface()) {
            return 0;
        }
        //获取父类MethodTable
        MethodTable *pMTParent = GetParentMethodTable();
        g_IBCLogger.LogMethodTableAccess(this);
        return pMTParent == NULL ? 0 : pMTParent->GetNumVirtuals();
    }

     #define SIZEOF__MethodTable_ (0x10 + (6 INDEBUG(+1)) * TARGET_POINTER_SIZE)
     一个MethodTable标准大小为，16+6*sizeof(void*)
    static inline DWORD GetVtableOffset()
    {
        return SIZEOF__MethodTable_;
    }

按照排布，Virtual Table在MethodTable固定成员之后，因此直接返回sizeof

    //获取所有函数的总和，虚函数，静态函数，普通成员函数
    WORD GetNumMethods()
    {
        //转发到EEClass
        return GetClass()->GetNumMethods();
    }

    //返回总Slot个数，仅用于功能性检验
    WORD GetNumVtableSlots()
    {
        return GetNumVirtuals() + GetNumNonVirtualSlots();
    }

#### Slot与MethodDescriptor

    MethodDesc* GetMethodDescForSlot(DWORD slot)
    {
        //取得起始地址
        PCODE pCode = GetRestoredSlot(slot);
        //这实际上是一个优化，对于Interface和虚函数，其Slot通常指向Stub
        if (IsInterface() && slot < GetNumVirtuals())
        {
            return MethodDesc::GetMethodDescFromStubAddr(pCode);
        }
        //普通函数
        return MethodTable::GetMethodDescForSlotAddress(pCode);
    }

##### 补充:
下方函数的注释可能让人摸不着头脑，真实意思是：
这个情况目前只针对委托的构造函数，因为这是唯一一个 fcall 里对一个实现有多个
MethodDescriptor的类型。断言是为了防止BackPatch，实际上这个Assert应当放在
SetSlot里面。

    static MethodDesc*  GetMethodDescForSlotAddress(PCODE addr, BOOL fSpeculative = FALSE)
    {
        //如果我们看到一个 fcall 实现被传入此函数，那么意味着 fcall 的
        //Virtual Table Slot在其不应该被后修补(BackPatch)时被后修补了
        //我们不能对此方法进行后修补的原因是，一个 fcall 实现有多个
        //MethodDescriptor。如果我们后修补委托的构造函数，本函数将
        //无法发现此方法的MethodDescriptor
        _ASSERTE_IMPL(!ECall::IsSharedFCallImpl(addr) &&
                  "someone backpatched shared fcall implementation -- "
                  "see comment in code");
        MethodDesc* pMethodDesc = ExecutionManager::GetCodeMethodDesc(addr);
        if (NULL != pMethodDesc)
        {
            goto lExit;
        }
    #ifdef FEATURE_INTERPRETER
        //我真的不知道为什么这个有用。把原因找出来
    #ifndef DACCESS_COMPILE
        //如果在上面都没找到，就尝试作为一个Interpretation stub
        pMethodDesc = Interpreter::InterpretationStubToMethodInfo(addr);
        if (NULL != pMethodDesc)
        {
            goto lExit;
        }
    #endif
    #endif
        //是否为一个 fcall 函数
        pMethodDesc = ECall::MapTargetBackToMethod(addr);
        if (pMethodDesc != 0)
        {
            goto lExit;
        }
        //普通函数
        pMethodDesc = MethodDesc::GetMethodDescFromStubAddr(addr, fSpeculative);
    lExit:
        RETURN(pMethodDesc);
    }

    PCODE GetRestoredSlot(DWORD slot)
    {
        //与MethodTable::GetRestoredSlotMT保持同步
        MethodTable * pMT = this;
        while (true)
        {
            g_IBCLogger.LogMethodTableAccess(pMT);
            //获取代表性MethodTable并且检查
            pMT = pMT->GetCanonicalMethodTable();
            _ASSERTE(pMT != NULL);
            PCODE slot = pMT->GetSlot(slotNumber);
            //Slot正常，且PreJIT下不能是从Virtual导入
            if ((slot != NULL)
    #ifdef FEATURE_PREJIT
            && !pMT->GetLoaderModule()->IsVirtualImportThunk(slot)
    #endif
            )
            {
                return slot;
            }
            //推断出是一个继承且未被修复的Slot，尝试在继承链上查找
            pMT = pMT->GetParentMethodTable();
        }
    }

     //与上面函数相似，只不过是返回对应MethodTable
     MethodTable * GetRestoredSlotMT(DWORD slot)
     {
        MethodTable * pMT = this;
        while (true)
        {
            g_IBCLogger.LogMethodTableAccess(pMT);
            pMT = pMT->GetCanonicalMethodTable();
            _ASSERTE(pMT != NULL);
            PCODE slot = pMT->GetSlot(slotNumber);
            if ((slot != NULL)
    #ifdef FEATURE_PREJIT
                && !pMT->GetLoaderModule()->IsVirtualImportThunk(slot)
    #endif
                )
            {
                return pMT;
            }
            pMT = pMT->GetParentMethodTable();
        }
     }

     //用于泛型实例化间映射函数到相同Slot上
     MethodDesc * GetParallelMethodDesc(MethodDesc * pDefMD)
     {
        return GetMethodDescForSlot(pDefMD->GetSlot());
     }

#### 封箱的入口点的MethodDescriptor

    结构体(ValueType)的虚函数在他们的Virtual Table里有封箱入口点MethodDescriptor
    参见MethodDesc::FindOrCreateAssociatedMethodDesc
    如果你需要在拆箱Stub和非拆箱Stub之间映射的话，你应当会使用那个函数

    //给定一个ValueType方法(泛型共享或者泛型实例化的)
    //找到给定方法的泛型共享拆箱Stub，我们会搜索整个Virtual Table
    //在创建非静态成员方法的委托时需要此函数
    MethodDesc* GetBoxedEntryPointMD(MethodDesc *pMD)
    {
            RETURN MethodDesc::FindOrCreateAssociatedMethodDesc(pMD,
                                                    pMD->GetMethodTable(),
                                                    TRUE //获取拆箱入口点,
                                                    pMD->GetMethodInstantiation(),
                                                    FALSE //不允许实例参数);
    }

    //跟定一个ValueType方法，找到非装箱方法
    //这个函数被用在为一个封箱入口点Stub生成代码时使用
    MethodDesc* GetUnboxedEntryPointMD(MethodDesc *pMD)
    {
        BOOL allowInstParam = (pMD->GetNumGenericMethodArgs() == 0);
        RETURN MethodDesc::FindOrCreateAssociatedMethodDesc(pMD,
                                                        this,
                                                        FALSE,//不要获取拆箱入口点
                                                        pMD->GetMethodInstantiation(),
                                                        allowInstParam);
    }

    //此函数与上方函数几乎一致，只是禁用掉了创建
    MethodDesc* GetExistingUnboxedEntryPointMD(MethodDesc *pMD);

#### 字段排布，对象大小

    //是否具有排布
    inline BOOL HasLayout()
    {
        //转发到EEClass
        return GetClass()->HasLayout();
    }

    //获取排布信息
    inline EEClassLayoutInfo *GetLayoutInfo()
    {
        //转发到EEClass
        return GetClass()->GetLayoutInfo();
    }

接下来一些都是直接转发EEClass或者及其简单，不再讲解

    inline BOOL IsBlittable()
    inline BOOL IsManagedSequential()
    inline BOOL HasExplicitSize()
    DWORD GetBaseSize()
    void SetBaseSize(DWORD baseSize) 

    BOOL IsStringOrArray()
    {
        return HasComponentSize();
    }

    BOOL IsString()
    {
        return HasComponentSize() && !IsArray() && RawGetComponentSize() == 2;
    }

    BOOL HasComponentSize() const
    {
        return GetFlag(enum_flag_HasComponentSize);
    }

    WORD RawGetComponentSize()
    {
        return *(WORD*)&m_dwFlags;
    } 

    //如果没有就返回0
    //Component Size实际上只是16位 WORD类型的，但此方法返回SIZE_T来确保
    //SIZE_T被用在每一个对象大小计算的地方。支持大于2GB的对象是很有必要的
    SIZE_T GetComponentSize()  
    {
        return HasComponentSize() ? RawGetComponentSize() : 0;
    }

    void SetComponentSize(WORD wComponentSize)
    {
        //如果这里有一个断言确保是String或者Array，但是我们怎么会知道？
        //断言ComonentSize大于0也是极好的，但是对于System.Void的数组
        //我们就遭殃了
        SetFlag(enum_flag_HasComponentSize);
        m_dwFlags = (m_dwFlags & ~0xFFFF) | wComponentSize;
    }

下面几个都是转发到EEClass

    inline WORD GetNumInstanceFields();
    inline WORD GetNumStaticFields();
    inline WORD GetNumThreadStaticFields();

    需要注意的是，对于ValueType，GetBaseSize对已装箱的值返回实例字段的大小
    GetNumInstanceFieldsBytes对应未装箱值
    我们在MethodTable上主要这样放置方法以便于与我们可以选择在MethodTable
    内部缓存信息，因此更少的代码会直接操控EEClass对象，因为这样会导致泛型
    Bug

    inline DWORD GetNumInstanceFieldBytes()
    {
        return (GetBaseSize() - GetClass()->GetBaseSizePadding());
    }

    inline WORD GetNumIntroducedInstanceFields()
    {
        //直接减去父类的字段
        WORD wNumFields = GetNumInstanceFields();
        MethodTable * pParentMT = GetParentMethodTable();
        if (pParentMT != NULL)
        {
            WORD wNumParentFields = pParentMT->GetNumInstanceFields();
            //如果此断言被除非，那么我们记账就做得很差。或许我们自增了基类
            //的字段数量但是忘了自增继承类的字段数(EnC情景下)
            _ASSERTE(wNumFields >= wNumParentFields);
            wNumFields -= wNumParentFields;
        }
        return(wNumFields);
    }

    //这个总是返回与GetBaseSize相关，或者相同的大小么？
    inline DWORD GetAlignedNumInstanceFieldBytes()
    {
        return((GetNumInstanceFieldBytes() + 3) & (~3));
    }

    //注意：这个Flag即使在未恢复的MethodTable中也必须可用
    //参见siginfo.cpp中的GcScanRoots
    DWORD ContainsPointers()
    {
        return GetFlag(enum_flag_ContainsPointers);
    }

    //当前类型是否可卸载
    BOOL Collectible()
    {
    #ifdef FEATURE_COLLECTIBLE_TYPES
        return GetFlag(enum_flag_Collectible);
    #else
        return FALSE;
    #endif
    }

    //含有指针或者可卸载类型对象
    BOOL ContainsPointersOrCollectible()
    {
        return GetFlag(enum_flag_ContainsPointers) || GetFlag(enum_flag_Collectible);
    }

    //获取加载器的分配器对象句柄
    OBJECTHANDLE GetLoaderAllocatorObjectHandle()
    {
        return GetLoaderAllocator()->GetLoaderAllocatorObjectHandle();
    }

    NOINLINE BYTE *GetLoaderAllocatorObjectForGC()
    {
        //要求可卸载对象
        if (!Collectible())
        {
            return NULL;
        }
        BYTE * retVal = *(BYTE**)GetLoaderAllocatorObjectHandle();
        return retVal;
    }

    //转发到EEClass
    //此标志指示ValueType是否为宽松打包的并且可以用于标记是否
    //能进行逐位比较来实现Equals方法，此函数对Class无效
    inline BOOL MethodTable::IsNotTightlyPacked()

    void SetContainsPointers()
    {
        SetFlag(enum_flag_ContainsPointers);
    }

#### 64位对齐

    inline bool RequiresAlign8()
    {
        return !!GetFlag(enum_flag_RequiresAlign8);
    }

    inline void SetRequiresAlign8()
    {
        SetFlag(enum_flag_RequiresAlign8);
    }

#### 字段描述符

    本段多数API仍然直接依靠EEClass
    警告：实例字段的FieldDescriptor是代表性的，会被相容的泛型实例化共享

    //本方法直接访问EEClass
    //使用本方法需要小心，有可能一些字段是通过Enc加入进来的
    //因此需要使用FieldDescIterator，因为Enc添加的字段不在RawList上
    PTR_FieldDesc GetApproxFieldDescListRaw()
    {
        return GetClass()->GetFieldDescList();
    }

    //对于静态字段，此函数返回确定类型的FieldDescriptor
    //对于非静态字段，可能仍然会返回代表性的
    PTR_FieldDesc GetFieldDescByIndex(DWORD fieldIndex)
    {
        if (HasGenericsStaticsInfo() &&
            fieldIndex >= GetNumIntroducedInstanceFields())
        {
            //静态泛型字段
            return GetGenericsStaticFieldDescs() + (fieldIndex - GetNumIntroducedInstanceFields());
        }
        else
        {
            //其余字段
            return GetClass()->GetFieldDescList() + fieldIndex;
        }
    }

    DWORD GetIndexForFieldDesc(FieldDesc *pField)
    {
        if (pField->IsStatic() && HasGenericsStaticsInfo())
        {
            //静态泛型
            FieldDesc *pStaticFields = GetGenericsStaticFieldDescs();
            return GetNumIntroducedInstanceFields() + DWORD(pField - pStaticFields);
        }
        else
        {
            //非静态或非泛型
            FieldDesc *pFields = GetClass()->GetFieldDescList();
            return DWORD(pField - pFields);
        }
    }

    //是否通过Ref封送
    BOOL IsMarshaledByRef()
    {
        return FALSE;
    }

    //是否上下文有关
    BOOL IsContextful()
    {
        return FALSE;
    }

    //是否要求调度使用胖Token
    inline bool RequiresFatDispatchTokens()
    {
        return !!GetFlag(enum_flag_RequiresDispatchTokenFat);
    }
    inline void SetRequiresFatDispatchTokens()
    {
        SetFlag(enum_flag_RequiresDispatchTokenFat);
    }

    //是否需要在分配时调用构造函数
    inline bool HasPreciseInitCctors()
    {
        return !!GetFlag(enum_flag_HasPreciseInitCctors);
    }
    inline void SetHasPreciseInitCctors()
    {
        SetFlag(enum_flag_HasPreciseInitCctors);
    }

#### HFA(Homogeneous Floating-point Aggregate)

#### 父接口

    unsigned GetNumInterfaces()
    {
        return m_wNumInterfaces;
    }

#### 转换

    对于以下函数有两个变种
    CanCastTo[]
        -需要时恢复已编码的指针
        -可能抛出异常，可能触发GC
        -返回Boolean
    CanCastTo[]NoGC
        -不会恢复已编码指针
        -不会抛出，不会触发GC
        -返回值为三值(能转换，不能转换，可能转换)
        -可能转换意味着在编码指针上测试失败了
         因此如果调用者很在意是否能转换，应当调用CanCastTo[]Restoring

    BOOL CanCastToInterface(MethodTable *pTargetMT, TypeHandlePairList *pVisited = NULL)
    {
        if (!pTargetMT->HasVariance())
        {
            //如果没有协变或者逆变
            //其中之一参与进了等效类型中
            if (HasTypeEquivalence() || pTargetMT->HasTypeEquivalence())
            {
                //当前是接口，且与对应接口等效
                if (IsInterface() && IsEquivalentTo(pTargetMT))
                    return TRUE;
                //是否实现了等效接口
                return ImplementsEquivalentInterface(pTargetMT);
            }
            //直接返回非协变或逆变转换结果
            return CanCastToNonVariantInterface(pTargetMT);
        }
        else
        {
            if (CanCastByVarianceToInterfaceOrDelegate(pTargetMT, pVisited))
                return TRUE;
        }
    }

    //能否通过协变或逆变转换到接口或者委托
    BOOL CanCastByVarianceToInterfaceOrDelegate(MethodTable *pTargetMT, TypeHandlePairList *pVisited)
    {
        BOOL returnValue = FALSE;
        EEClass *pClass = NULL;
        //是否已经存在类型对
        if (TypeHandlePairList::Exists(pVisited, this, pTargetMT))
            goto Exit;
        //校验ID和模块
        if (GetTypeDefRid() != pTargetMT->GetTypeDefRid() || GetModule() != pTargetMT->GetModule())
        {
            goto Exit;
        }
        {
            //获取EEClass
            pClass = pTargetMT->GetClass();
            //获取实例化列表，目标实例化列表
            Instantiation inst = GetInstantiation();
            Instantiation targetInst = pTargetMT->GetInstantiation();
            for (DWORD i = 0; i < inst.GetNumArgs(); i++)
            {
                TypeHandle thArg = inst[i];
                TypeHandle thTargetArg = targetInst[i];
                //不等价
                if (!thArg.IsEquivalentTo(thTargetArg))
                {
                    //获取参数协变/逆变性
                    switch (pClass->GetVarianceOfTypeParameter(i))
                    {
                    case gpCovariant :
                        //协变，如果此参数不能转换到目标参数，返回
                        if (!thArg.IsBoxedAndCanCastTo(thTargetArg, &pairList))
                            goto Exit;
                        break;
                    case gpContravariant :
                        //逆变，如果此参数不能转换到目标参数，返回
                        if (!thTargetArg.IsBoxedAndCanCastTo(thArg, &pairList))
                            goto Exit;
                        break;
                    case gpNonVariant :
                        //没有任何协变逆变，且不相容，返回
                        goto Exit;
                    default :
                        _ASSERTE(!"Illegal variance annotation");
                        goto Exit;
                    }
                }
            }
        }
        //检查完全
        returnValue = TRUE;
    Exit:    
        //出口
        return returnValue;
    }

    BOOL CanCastToNonVariantInterface(MethodTable *pTargetMT)
    {
        //是否就为目标接口
        if (this == pTargetMT)
            return TRUE;
        //是否实现了接口
        return ImplementsInterfaceInline(pTargetMT);
    }

以下是NoGC版本的三值函数

    TypeHandle::CastResult CanCastToInterfaceNoGC(MethodTable *pTargetMT)
    {
        //不是数组，没有参与类型等效，没有协变逆变
        if (!pTargetMT->HasVariance() && !IsArray() && !HasTypeEquivalence() && !pTargetMT->HasTypeEquivalence())
        {
            return CanCastToNonVariantInterface(pTargetMT) ? TypeHandle::CanCast : TypeHandle::CannotCast;
        }
        else
        {
            //对协变逆变与等效类型持有保守态度
            return TypeHandle::MaybeCast;
        }
    }

    //转换类型
    TypeHandle::CastResult CanCastToClassNoGC(MethodTable *pTargetMT)
    {
        //协变逆变类型保守估计
        if (pTargetMT->HasVariance() || g_IBCLogger.InstrEnabled())
        {
            return TypeHandle::MaybeCast;
        }
        //等效类型需要走SlowPath
        if (HasTypeEquivalence() || pTargetMT->HasTypeEquivalence())
        {
            return TypeHandle::MaybeCast;
        }
        else
        {
            //最后沿着继承链查找
            PTR_VOID pMT = this;
            do {
                if (pMT == pTargetMT)
                    return TypeHandle::CanCast;
                pMT = MethodTable::GetParentMethodTableOrIndirection(pMT);
            } while (pMT);
        }
        return TypeHandle::CannotCast;
    }
#### 等效类型
    BOOL IsEquivalentTo_Worker(MethodTable *pOtherMT COMMA_INDEBUG(TypeHandlePairList *pVisited))
    {
        //泛型与非泛型一致性
        if (HasInstantiation() != pOtherMT->HasInstantiation())
            return FALSE;
        //如果是Array
        if (IsArray())
        {
            //另一个不是Array或者维度不相同
            if (!pOtherMT->IsArray() || GetRank() != pOtherMT->GetRank())
                return FALSE;
            //struct的Array有其自己的隐藏MethodTable并且会走这条路径
            return (GetApproxArrayElementTypeHandle().IsEquivalentTo(pOtherMT->GetApproxArrayElementTypeHandle() COMMA_INDEBUG(&newVisited)));
        }
        //走Inner比较
        return IsEquivalentTo_WorkerInner(pOtherMT COMMA_INDEBUG(&newVisited));
    }

    BOOL IsEquivalentTo_WorkerInner(MethodTable *pOtherMT COMMA_INDEBUG(TypeHandlePairList *pVisited))
    {
        TypeEquivalenceHashTable *typeHashTable = NULL;
        AppDomain *pDomain = GetAppDomain();
        if (pDomain != NULL)
        {
            //获取等效Hash表
            typeHashTable = pDomain->GetTypeEquivalenceCache();
            TypeEquivalenceHashTable::EquivalenceMatch match = typeHashTable->CheckEquivalence(TypeHandle(this), TypeHandle(pOtherMT));
            //检查结果
            switch (match)
            {
            case TypeEquivalenceHashTable::Match:
                return TRUE;
            case TypeEquivalenceHashTable::NoMatch:
                return FALSE;
            case TypeEquivalenceHashTable::MatchUnknown:
                break;
            default:
                _ASSERTE(FALSE);
                break;
            }
        }
        BOOL fEquivalent = FALSE;
        //检查是否为泛型
        if (HasInstantiation())
        {
            //将泛型的协变逆变限制到Inteface上
            if (!IsInterface() || !pOtherMT->IsInterface())
            {
                fEquivalent = FALSE;
                goto EquivalenceCalculated;
            }
            Instantiation inst1 = GetInstantiation();
            Instantiation inst2 = pOtherMT->GetInstantiation();
            //检查参数个数
            if (inst1.GetNumArgs() != inst2.GetNumArgs())
            {
                fEquivalent = FALSE;
                goto EquivalenceCalculated;
            }
            //逐个比较参数
            for (DWORD i = 0; i < inst1.GetNumArgs(); i++)
            {
                if (!inst1[i].IsEquivalentTo(inst2[i] COMMA_INDEBUG(pVisited)))
                {
                    fEquivalent = FALSE;
                    goto EquivalenceCalculated;
                }
            }
            if (GetTypeDefRid() == pOtherMT->GetTypeDefRid() && GetModule() == pOtherMT->GetModule())
            {
                //现在就可以确定MethodTable等效，我们只关心的是IList<IFoo>
                //和IList<IBar>之间IFoo与IBar是等效的
                fEquivalent = TRUE;
            }
            else
            {
                fEquivalent = FALSE;
            }
            goto EquivalenceCalculated;
        }
        //对Array检查
        if (IsArray())
        {
            if (!pOtherMT->IsArray() || GetRank() != pOtherMT->GetRank())
            {
                fEquivalent = FALSE;
                goto EquivalenceCalculated;
            }
            TypeHandle elementType1 = GetApproxArrayElementTypeHandle();
            TypeHandle elementType2 = pOtherMT->GetApproxArrayElementTypeHandle();
            fEquivalent = elementType1.IsEquivalentTo(elementType2 COMMA_INDEBUG(pVisited));
            goto EquivalenceCalculated;
        }
        fEquivalent = CompareTypeDefsForEquivalence(GetCl(), pOtherMT->GetCl(), GetModule(), pOtherMT->GetModule(), NULL);
    EquivalenceCalculated:
        if (typeHashTable != NULL)
        {
            //均不是可卸载类型，可卸载类型结果不会被缓存
            if ((!Collectible() && !pOtherMT->Collectible()))
            {
                auto match = fEquivalent ? TypeEquivalenceHashTable::Match : TypeEquivalenceHashTable::NoMatch;
                typeHashTable->RecordEquivalence(TypeHandle(this), TypeHandle(pOtherMT), match);
            }
        }
        //返回结果
        return fEquivalent;
    }

#### 父类
    //父类加载级别
    BOOL HasApproxParent()
    {
         return (GetWriteableData()->m_dwFlags & MethodTableWriteableData::enum_flag_HasApproxParent) != 0;      
    }
    inline void SetHasExactParent()
    {
        FastInterlockAnd(&(GetWriteableDataForWrite()->m_dwFlags), ~MethodTableWriteableData::enum_flag_HasApproxParent);
    }

    //调用者必须知道父MethodTable不是已编码修复的
    inline PTR_MethodTable GetParentMethodTable()
    {
        return ReadPointerMaybeNull(this, &MethodTable::m_pParentMethodTable, GetFlagHasIndirectParent());
    }

    //获取父MethodTable或者其Indirection
    inline static PTR_VOID GetParentMethodTableOrIndirection(PTR_VOID pMT)
    {
        return PTR_VOID(*PTR_TADDR(dac_cast<TADDR>(pMT) + offsetof(MethodTable, m_pParentMethodTable)));
    }

    //父MethodTable是否为Indirection
    inline bool IsParentMethodTableIndirectPointer()
    {
         return m_pParentMethodTable.IsIndirectPtrIndirect(GetFlagHasIndirectParent(), PARENT_MT_FIXUP_OFFSET);
    }

    //获取Indirection所指向的MethodTable指针
    inline MethodTable ** GetParentMethodTableValuePtr()
    {
        return m_pParentMethodTable.GetValuePtrIndirect(GetFlagHasIndirectParent(), PARENT_MT_FIXUP_OFFSET);
    }

    //父类是否与给定MethodTable一致
    BOOL ParentEquals(PTR_MethodTable pMT)
    {
        //有效性先验
        PRECONDITION(IsParentMethodTablePointerValid());
        g_IBCLogger.LogMethodTableAccess(this);
        return GetParentMethodTable() == pMT;
    }

    //设置父MethodTable
    void SetParentMethodTable (MethodTable *pParentMethodTable)
    {     
        PRECONDITION(!IsParentMethodTableIndirectPointerMaybeNull());
        m_pParentMethodTable.SetValueMaybeNull(pParentMethodTable);
    }

    //在继承链上寻找，TypeDef相匹配的父类。
    //这样有用的原因参见
    //MethodTable::GetInstantiationOfParentClass
    //Generics::GetExactInstantiationsOfMethodAndItsClassFromCallInformation
    MethodTable * GetMethodTableMatchingParentClass(MethodTable * pWhichParent)
    {
        MethodTable *pMethodTableSearch = this;
        //遍历继承链
        while (pMethodTableSearch != NULL) 
        {
            if (pMethodTableSearch->HasSameTypeDefAs(pWhichParent))
            {
                return pMethodTableSearch;
            }
            pMethodTableSearch = pMethodTableSearch->GetParentMethodTable();
        }
    }
##### 父类小小结
m_pParentMethodTable可能有两种形式，一种为直接的MethodTable，另一种为指向MethodTable
的Indirection
#### EEClass

    EEClass在实例化之间被共享
    需要注意的是，普通情况下GetClass().GetMethodTable()==this并不成立

    //此函数进行了转发到Nologging版本
    PTR_EEClass GetClass();
    inline PTR_EEClass GetClass_NoLogging()
    {
        //获取地址
        TADDR addr = ReadPointer(this, &MethodTable::m_pCanonMT);
        //低两位为10，则为EEClass指针
        if ((addr & 2) == 0)
        {
            return PTR_EEClass(addr);
        }
    #ifdef FEATURE_PREJIT
        //低位为01
        if ((addr & 1) != 0)
        {
            //获取代表性指针地址，向前偏移三个指针长度
            //这里实际上是指向Indirection的指针，其中指向了代表性MethodTable
            TADDR canonicalMethodTable = *PTR_TADDR(addr - 3);
            //在代表性MethodTable中获取EEClass
            return PTR_EEClass(ReadPointer((MethodTable *) PTR_MethodTable(canonicalMethodTable), &MethodTable::m_pCanonMT));
        }
    #endif     
        //否则向前偏移两个指针长度读取代表性指针
        return PTR_EEClass(ReadPointer((MethodTable *) PTR_MethodTable(addr - 2), &MethodTable::m_pCanonMT));
    }

    //验证MethodTable(有AV风险)
    BOOL ValidateWithPossibleAV()
    {
        //MethodTable有以下代表化的性质
        //先代表化，再次代表化，最后检查结果是否为一致的
        //这个性质对系统里的每一个有效对象都是成立的，但是需要持有一些
        //其余的地址信息
        //对于非泛型类，我们依赖于比较
        //  object->methodtable->class->methodtable 与
        //  object->methodtable
        //对于泛型，上面的不起作用，我们必须比较
        //  object->methodtable->class->methodtable->class 与
        //  object->methodtable->class
        //当然，这些验证MethodTable是否有效的方法还不够，我们需要更多的
        //检验来确保这里的指针是有效的
        PTR_EEClass pEEClass = this->GetClassWithPossibleAV();
        return ((this == pEEClass->GetMethodTableWithPossibleAV()) ||
        ((HasInstantiation() || IsArray()) &&
        (pEEClass->GetMethodTableWithPossibleAV()->GetClassWithPossibleAV() == pEEClass)));
    }

    //检查EEClass指针是否有效
    BOOL IsClassPointerValid()
    {
        TADDR addr = ReadPointer(this, &MethodTable::m_pCanonMT);
        //获取低位比特位
        LowBits lowBits = union_getLowBits(addr);
        //是EEClass，检验是否为null
        if (lowBits == UNION_EECLASS)
        {
            return !m_pEEClass.IsNull();
        }
        else if (lowBits == UNION_METHODTABLE)
        {
            //如果是MethodTable，那么就获取代表性MethodTable地址
            //再检查共享的EEClass是否有效
            TADDR canonicalMethodTable = union_getPointer(addr);
            return !PTR_MethodTable(canonicalMethodTable)->m_pEEClass.IsNull();
        }
    #ifdef FEATURE_PREJIT   
        else if (lowBits == UNION_INDIRECTION)
        {
            //PreJIT下储存在Indirection里，取出代表性MethodTable
            TADDR canonicalMethodTable = *PTR_TADDR(union_getPointer(addr));
            if (CORCOMPILE_IS_POINTER_TAGGED(canonicalMethodTable))
                return FALSE;
            return !PTR_MethodTable(canonicalMethodTable)->m_pEEClass.IsNull();
        }
    #endif
        //不正常的EEClass指针
        _ASSERTE(!"Malformed m_pEEClass in MethodTable");
        return FALSE;
    }

##### EEClass小小结
从上面两个函数可以看得出来，m_pCanonMT的储存规则是
    
    1. 普通非泛型或者代表性的MethodTable，EEClass指针直接存于m_pCanonMT中
    2. 泛型实例共享的EEClass通过m_pCanonMT指向的代表性MethodTable获取
    3. PreJIT的则是通过m_pCanonMT储存Indirection，在Indirection中获取代表性
       MethodTable

#### MethodTable构造
这几个函数较为简单，不再讲解

    inline void SetClass(EEClass *pClass);
    inline void SetCanonicalMethodTable(MethodTable * pMT);
    inline void SetHasInstantiation(BOOL fTypicalInstantiation, BOOL fSharedByGenericInstantiations);
#### 接口实现
    //检测接口实现的强制内联版本
    BOOL ImplementsInterfaceInline(MethodTable *pInterface)
    {
        DWORD numInterfaces = GetNumInterfaces();
        if (numInterfaces == 0)
            return FALSE;
        InterfaceInfo_t *pInfo = GetInterfaceMap();
        do
        {
            if (pInfo->GetMethodTable() == pInterface)
            {
                //可扩展RCW需要特殊处理，因为他们的Map拥有运行时添加上去的接口
                //这些接口会拥有一个起始偏移为-1的偏移量来表示这种情况
                //我们不能把每一个此COM对象的实例都有这个接口因此FindInterface
                //在这些接口上注定会失败的情况视作理所应当
                //然而这里我们只考虑静态的Slot(m_wNumInterface不包含动态Slot)
                //因此我们可以安全地忽略这个细节
                return TRUE;
            }
            pInfo++;
        }
        while (--numInterfaces);
        return FALSE;
    }

    ImplementsInterface简单转发了强制Inline版本

    //实现类型等效接口
    BOOL ImplementsEquivalentInterface(MethodTable *pInterface)
    {
        //首先检查具体情况，为成功Case优化
        if (ImplementsInterfaceInline(pInterface))
            return TRUE;
        //接口无类型等效
        if (!pInterface->HasTypeEquivalence())
            return FALSE;
        DWORD numInterfaces = GetNumInterfaces();
        if (numInterfaces == 0)
            return FALSE;
        //遍历Map查找等效接口
        InterfaceInfo_t *pInfo = GetInterfaceMap();
        do
        {
            if (pInfo->GetMethodTable()->IsEquivalentTo(pInterface))
                return TRUE;
            pInfo++;
        }
        while (--numInterfaces);
        return FALSE;
    }

    //获取Interface函数的MethodDescriptor
    MethodDesc *GetMethodDescForInterfaceMethod(TypeHandle ownerType, MethodDesc *pInterfaceMD, BOOL throwOnConflict)
    {
        MethodDesc *pMD = NULL;
        MethodTable *pInterfaceMT = ownerType.AsMethodTable();
    #ifdef CROSSGEN_COMPILE
        //CrossGen编译下使用DispatchSlot查找目标MethodDesc
        DispatchSlot implSlot(FindDispatchSlot(pInterfaceMT->GetTypeID(), pInterfaceMD->GetSlot(), throwOnConflict));
        if (implSlot.IsNull())
        {
            _ASSERTE(!throwOnConflict);
            return NULL;
        }
        PCODE pTgt = implSlot.GetTarget(); 
    #else
        //否则使用VirtualCall
        PCODE pTgt = VirtualCallStubManager::GetTarget(
        pInterfaceMT->GetLoaderAllocator()->GetDispatchToken(pInterfaceMT->GetTypeID(), pInterfaceMD->GetSlot()),
        this, throwOnConflict);
        if (pTgt == NULL)
        {
            _ASSERTE(!throwOnConflict);
            return NULL;
        }
    #endif
        pMD = MethodTable::GetMethodDescForSlotAddress(pTgt);
        //检查并恢复(可能是在本机映像中)
        pMD->CheckRestore();
        return pMD;
    }

而第二个重载版本仅能适用于非泛型接口函数，其做了简单转发，并直接从MethodTable构造TypeHandle(这就是为什么)

    MethodDesc *GetMethodDescForInterfaceMethod(MethodDesc *pInterfaceMD, BOOL throwOnConflict);
#### 接口图(Interface Map)
    //获取接口图
    inline PTR_InterfaceInfo GetInterfaceMap()
    {
        //直接读取指针
        return ReadPointer(this, &MethodTable::m_pInterfaceMap);
    }
#### 嵌套类-InterfaceMapIterator
此类被用于接口图的迭代，而不直接接触接口图，这样就可以轻松更改图的实现。
这个类目前比较简单，仍然是C#风格迭代器，遍历数组。这里不再讲解。
#### 附加Interface Map数据
    //* 我们为了更好的数据密度(其实就是省空间)，选择在独立的，可选的位置储存
    //  此MethodTable实现的Interfaces的额外信息(一些Flag)，如果我们直接把
    //  他们放在Interface Map里，内存对齐会要求我们使用32位甚至64位来储存一个
    //  bool值。目前我们储存的唯一Flag是IsDeclaredOnClass(这个接口是否被此类型
    //  显式声明)

    //目前来讲，当我们有Interface Map时，我们总是储存额外的信息
    //你可以想象在未来这个被限制于那些至少有一个接口拥有一个Flag的非默认值
    //的情况
    inline BOOL HasExtraInterfaceInfo()
    {
        return HasInterfaceMap();
    }

    //接口可以拥有其自己的额外信息并储存在可选数据结构里的数量
    //一旦接口数量超过此值，可选数据Slot会被替换为指向Buffer的指针
    //Buffer里储存有额外的信息
    enum { kInlinedInterfaceInfoThreshhold = sizeof(TADDR) * 8 };

    //本函数计算接口需要多少bytes来储存其额外的信息
    //当没有接口时，这个值是0，但在接口数量小的时候也可能是0
    //调用者必须准备处理这个情况
    static SIZE_T GetExtraInterfaceInfoSize(DWORD cInterfaces)
    {
        //对于小接口数量，我们可以将可选成员当作BitMap储存
        if (cInterfaces <= kInlinedInterfaceInfoThreshhold)
            return 0;
        //否则我们会导致TADDR数组被分配
        //使用TADDR是因为几乎所有堆上分配的空间需要按TADDR的方式对齐
        return ALIGN_UP(cInterfaces, sizeof(TADDR) * 8) / 8;
    }

    //此函数在GetExtraInterfaceInfoSize之后被调用设立附加的内存
    //用于追踪额外的接口信息
    void InitializeExtraInterfaceInfo(PVOID pInfo)
    {
        //检查内存是否在正确的情景下被分配
        _ASSERTE(((pInfo == NULL) && (GetExtraInterfaceInfoSize(GetNumInterfaces()) == 0)) ||
            ((pInfo != NULL) && (GetExtraInterfaceInfoSize(GetNumInterfaces()) != 0)));
        //如果我们不要求额外的接口信息，此调用就是没有作用的(这种情况下
        //buffer永远不应该被分配)
        if (!HasExtraInterfaceInfo())
        {
            _ASSERTE(pInfo == NULL);
            return;
        }
        //获取内联储存或者指向数据块的可选Slot
        PTR_TADDR pInfoSlot = GetExtraInterfaceInfoPtr();
        //无论是哪种情况，将pInfo写入总是正确的，在内联直接储存的情况下
        //我们希望设置所有flag为0并且写入null就可以做到这一点。
        //否则我们只是简单地想把buffer指针写入slot(这里不需要分辨位，我们总是
        //可以通过接口数量来分辨使用了哪种格式)
        *pInfoSlot = (TADDR)pInfo;
    }

下面又是熟悉的NGen环节

    //储存额外信息
    void SaveExtraInterfaceInfo(DataImage *pImage)
    {
        //要么没有额外数据，要么是内联的
        if (GetNumInterfaces() <= kInlinedInterfaceInfoThreshhold)
            return;
        //储存到本机映像
        pImage->StoreStructure((LPVOID)*GetExtraInterfaceInfoPtr(),
                           GetExtraInterfaceInfoSize(GetNumInterfaces()),
                           DataImage::ITEM_INTERFACE_MAP);
    }

    //修复额外信息
    void FixupExtraInterfaceInfo(DataImage *pImage)
    {
        if (GetNumInterfaces() <= kInlinedInterfaceInfoThreshhold)
            return;
        pImage->FixupPointerField(this, (BYTE*)GetExtraInterfaceInfoPtr() - (BYTE*)this);
    }

    //标记通过特殊Index在指定Map中的Interface为在类上显式声明
    //对于RCW动态添加的Interface不合法
    void SetInterfaceDeclaredOnClass(DWORD index)
    {
        _ASSERTE(HasExtraInterfaceInfo());
        _ASSERTE(index < GetNumInterfaces());
        //获取可选信息Slot
        PTR_TADDR pInfoSlot = GetExtraInterfaceInfoPtr();
        //如果是内联储存，直接设置
        if (GetNumInterfaces() <= kInlinedInterfaceInfoThreshhold)
        {
            *pInfoSlot |= SELECT_TADDR_BIT(index);
        }
        else
        {
            //获取Buffer
            TADDR *pBitmap = (PTR_TADDR)*pInfoSlot;
            //对应数组Index
            DWORD idxTaddr = index / (sizeof(TADDR) * 8); 
            //对应bitset内的Index
            DWORD idxInTaddr = index % (sizeof(TADDR) * 8);
            TADDR bitmask = SELECT_TADDR_BIT(idxInTaddr);
            //设置比特位
            pBitmap[idxTaddr] |= bitmask;
            _ASSERTE((pBitmap[idxTaddr] & bitmask) == bitmask);
        }
    }

    bool IsInterfaceDeclaredOnClass(DWORD index)
    {
        _ASSERTE(HasExtraInterfaceInfo());
        //对于动态添加的Interface，其总不是显式声明的(这是规定)
        if (index >= GetNumInterfaces())
        {
    #ifdef FEATURE_COMINTEROP
            _ASSERTE(HasDynamicInterfaceMap());
    #endif // FEATURE_COMINTEROP
            return false;
        }
        //以下代码同上
        TADDR taddrInfo = *GetExtraInterfaceInfoPtr();
        if (GetNumInterfaces() <= kInlinedInterfaceInfoThreshhold)
        {
            return (taddrInfo & SELECT_TADDR_BIT(index)) != 0;
        }
        else
        {
            TADDR *pBitmap = (PTR_TADDR)taddrInfo;
            DWORD idxTaddr = index / (sizeof(TADDR) * 8);
            DWORD idxInTaddr = index % (sizeof(TADDR) * 8);
            TADDR bitmask = SELECT_TADDR_BIT(idxInTaddr);
            return (pBitmap[idxTaddr] & bitmask) != 0;
        }
    }

    