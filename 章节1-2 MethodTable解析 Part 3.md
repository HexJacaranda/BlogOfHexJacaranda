### Virtual/Interface调用解析
### VSD 机制介绍
VSD全称为Virtual Stub Dispatching，其使用Stub来用于虚函数的调用而不是使用传统的
虚函数表。在过去，接口的调度要求接口拥有相对于进程独一无二(process unique)的标识
符，并且每一个装载的接口都要被添加到全局的接口虚函数表Map中。这个要求意味着，所有的接口及其所有实现接口的类型于NGen情况下在运行时必须进行恢复，导致显著的启动工作集增大。使用Stub调度的动机是为了消除很大一部分的相关工作集，同时也将剩余的工作分布到进程的整个生命周期中。

尽管使用VSD来调度接口和虚函数调用是可以的，但目前仅仅被用于接口调度
### VSD依赖
#### 组件依赖
目前VSD代码相对于Runtime独立。其提供API供依赖组件使用，并且下方列出的依赖
保证只涉及一块相对小的方面
##### 代码管理器
VSD依赖于代码管理器来提供方法的状态信息，确切的说，是任意方法是否转移到了其
最终的状态以便于VSD可以根据细节决定Stub的生成与目标的缓存(指解析后的缓存)
##### 类型与方法
MethodTable持有调度Map的指针，其用于决定给定VSD调用的目标函数地址
##### 特殊类型
在COM交互类型上的调用必须使用自定义的调度，因为他们都有特殊化的目标解析方法
#### 依赖此组件的组件
##### 代码管理器
代码管理器依赖于VSD为JIT编译器提供接口调用的目标函数
##### 类构建器(Class Builder)
类构建器使用调度Map所暴露的API在类型构建时创建那些会被VSD代码在调度类型上
使用的调度Map

### 设计目标与副作用
#### 目的
##### 工作集精简
接口调度先前的实现是使用一个很大的,甚至有点稀疏的Virtual Table查找来应对
贯穿进程的接口标识符。此目标是为了通过按需产生调度Stub来减少
冷工作集(使用频率较低)的数据量，在理论上保持相关的调用以及他们的调度Stub
在一起并且增加工作集的数据紧凑度

需要注意且很重要的是，参与VSD的初始化工作集因为需要使用数据结构来在
在系统运行时跟踪不同需要构建和回收的Stub而实际上对于每个调用的地方是
多于一个的(工作集)；然而，当应用程序到达稳定状态时，这些数据结构将不再
被简单的调度所需要，因此会得到回收。不幸的是，对于客户端程序来讲，这也就
等于更慢的启动，这也是导致对虚函数禁用VSD的原因之一

##### 吞吐量的等价
保持接口和虚函数的调度与先前的Virtual Table调度机制在一个可平摊的吞吐量等价下是
相当重要的。

尽管很明显的是，使用接口调度可以做到这一点，但最后结果仍然是比传统虚函数调度慢了
一些，这也是一个对虚函数禁用VSD的原因

### Token的表示与调度Map的设计
调度Token是一个在运行时分配的本机字长大小的值，内部由一个表示接口和Slot的元组表示

此设计使用了分配的类型标识符值与Slot值结合。为了利用与Runtime的整合，实现也像传统
Virtual Table分布那样来分配Slot索引。这意味着Runtime仍然可以一样地应对
MethodTable，MethodDescriptor与Slot索引，除了Virtual Table必须使用帮助函数
而不是直接访问来处理此抽象。

词Slot总是用于经典的Virtual Table被创建且被映射机制翻译的Slot索引值的情况
也就是说，如果你要描述经典的虚函数Slot在MethodTable中的排布，Slot就意味着
这是一个索引。理解这个差异是非常重要的，因为在Runtime代码中，Slot表示经典
Virtual Table结构里的索引值和在Virtual Table里指针的地址。现有的变化是，Slot
仅仅是一个索引值，代码指针的地址被包含在实现表中。

### MethodTable
#### 实现表
实现表是一个数组，其中每一个方法都是被类型所引入的，拥有指向方法入口点的指针，其
成员按照如下顺序排布：

        * 引入(新Slot)的虚函数
        * 引入的非虚函数(实例函数和静态函数)
        * 重写的虚函数

使用此格式的原因是，其为传统V-Table的排布提供了一种自然的扩展。因此许多在SlotMap
的入口能被此顺序和其他细节，比如总的虚和非虚方法数量所推断出来。

在当前Stub调度对虚方法禁用时，实现表实际上是不存在的，且被真正的V-Table所替代
 ，所有的映射结果都被表示成V-Table的Slot而非实现表的。

#### Slot Map
Slot Map是一个拥有多个形如<type,[<slot,scope,(index|slot)>]>>入口的表，其中
类型使用了上面提到的动态分配的标识数来表示，其要么是表示当前类的守卫(sentinel)值
(调用虚实例函数)，要么是被当前类型实现的接口标识符(或者是被其父类隐式地实现)
子Slot Map，也就是被方括号所包围起来的，有一个或多个入口。其中第一个元素总是表示
类型内的Slot，第二个元素Scope指定第三个元素是否为一个实现表的索引或是一个Slot索引
Scope能够成为指示下一个元素用途的守卫值，其一就是下一个数字是否被解释为一个虚函数
Slot并且应当被解析成虚函数调用this.slot。其二就是Scope也能标识在当前类型继承链上的
特定类型。在这种情况下，第三个参数就是一个位于Scope所指示的类型的实现表的索引，并且
这是一个type.slot最终的方法实现。

例子如下

    interface I {                           实现表
        void Foo();                         [I.Foo prestub]
    }                                       Slot Map
    经典Virtual Table(也就是现行的)           [I接口ID]
    [I.Foo prestub]                             [0,scope I,0*]

    class A : I {                           实现表
        virtual void Foo(){}                [A.Foo 代码入口]
        virtual void Bar(){}                [A.Bar 代码入口]
    }                                       Slot Map
    经典Virtual Table                       [A类型ID]
    [A.Foo 代码入口]                            [4,scope A,0*]
    [A.Bar 代码入口]                            [5,scope A,1*]
                                            [I接口ID]
                                                [0,virtual,4]

    class B : A {                           实现表
        virtual override void Foo(){}       [B.Baz 代码入口]
        virtual void Baz(){}                [B.Foo 代码入口]
    }                                       Slot Map
    经典Virtual Table                       [B类型ID]
    [B.Foo 代码入口]                            [6,scope B,0*]
    [A.Bar 代码入口]                            [4,scope B,1]
    [B.Baz 代码入口]

因此，看到如上的代码，Sub Map的第一列对应到经典的Virtual Table的索引(为了
方便起见，我们省略掉了System.Object自带的4个函数)。寻找方法实现的顺序总是
自下而上。因此如果我们拥有一个B类型的对象，想要调用I.Foo方法，我们会从B的
Slot Map中查找，发现没有以接口I的ID作为Key的，因此沿着继承链向上走，来到A
的Slot Map，发现就在那里。其表示Foo的第一个Slot(值为0)，是被B中虚函数表索引
为4的Slot所实现。接着返回到B的Slot Map并且查找Slot 4的实现，并且找到其位于
实现表中的Slot 1。

#### 附加用途
很重要的一点是，此映射方法能够被用于实现虚方法Slot实现的重映射。因为Map的Scope
能力，非虚函数也可以被引用。当Runtime想要支持用非虚函数实现接口时或许会非常有用。

#### 优化
Slot Map是按位来编码的，而且利用了经典接口使用Δ值来实现的优势，大大地减少了Map
的大小。除此之外，新的Slot(无论虚或者非虚)都可以通过在实现表中的顺序被推测出来。
如果表中含有新的虚函数Slot，其后跟随着非虚实例函数的Slot，其后又是重写函数，那么
恰当的Slot Map 入口就可以通过其索引和各类Slot的数量被推测出来。所有这样可以被推测
出来的入口都使用(*)号在上图被标注出来。
现行的数据结构排布使用以下的方案，这里Dispatch Map仅当映射不能被使用顺序推断出来
时才存在。

 MethodTable -> [DispatchMap ->] ImplementationTable

### 类型ID Map
此Map将类型映射到ID上，ID是单调增加的。目前来说，所有这样的类型都是接口。
并且目前使用HashMap来实现，并且包含入口用于双向查找。

### 调度Token
调度Token都是<typeID,slot>这样的元组。对于接口，类型会是分配的接口ID
对于虚方法，此值将会是一个表示此Slot会被类型内部所决议的目标函数Slot
此值对在绝大情况下都和平台的本机字长一致，在x86上，有可能是每个值的低
16位所连接起来的。这种情况能被泛化到处理溢出问题，就好像一个TypeHandle
在运行时要么是MethodTable指针，要么是一个<TypeHandle,TypeHandle>对
，使用守卫位来区分这两种情况。现在还没有决定这样做是否为必要的。
### VSD设计
#### 从调度Token到方法实现解析
给定一个Token和类型，方法的实现是通过映射Token到对应类型实现表的索引来查找。
实现表必须从类型的MethodTable可达。此Map是被BuildMethodTable所创建，其遍历
所有被此类型实现的接口来为类型构建MethodTable和决定每一个由此类型实现或者重写
的接口方法。通过跟踪这些信息，在接口调度时期给定Token和对象来决定目标方法是可能
的(对象中可获取MethodTable和Token映射)

#### Stub
接口调度调用通过Stub进行，这些Stub都是按需生成的，并且都有一个终极目标:匹配token
和object到一个具体的方法实现，并且转发调用到那个方法实现。
目前只有三种类型的Stub，他们分别是LookUp Stub，Dispatch Stub，Resolve Stub

##### Generic Resolver
这只是一个作为所有Stub失败的最终路径，其使用一个<token,type>的元组返回目标函数
Generic Resolver也负责在需要时创建Dispatch和Resolve Stub，当更好的Stub可用
时，为Indirection Cell替换Stub，缓存结果，还有其他的记录工作
##### Lookup Stub
这些stub首先被分配到接口调度的地方，并且在JIT编译接口调用的地方被创建。因为JIT
在第一次调用执行之前完全不知道满足token的类型，这个stub就将token和类型作为参数
传递给 Generic Resolver。如果有必要，Generic Resolver会创建Dispatch Stub和
Resolve Stub，之后将调用地点进行打补丁，更新成Dispatch Stub因此Lookup Stub
不会再被使用。

对每一个独一无二的token都会创建一个Lookup Stub(比如，对同一个接口Slot调用的
调用地点会使用同一个Lookup Stub)

##### Dispatch Stub
这些Stub被用于认为行为上单态的调用地点，也就是用于某个调用地点的对象实际上都是
一个类型，Dispatch Stub使用被调用的对象MethodTable并且与被缓存的类型比较，
如果成功，就跳转到缓存的方法。在x86上这会导致一个"比较，条件失败跳转，跳转到目标"
指令序列并且为任意Stub提供最佳的性能。如果Stub类型比较失败了，这就会跳转到对应的
Resolve Stub

对每一个独一无二的<token,type>元组都生成一个Dispatch Stub，但会延迟创建，直到
调用地点的LookUp Stub被调用时。
##### Resolve Stub
多态调用地点经由Resolve Stub处理。这些Stub用<token,type>来解析全局缓存中的目标
方法，这里token在JIT时就可以确定，而类型在调用时可以确定。如果全局缓存不包含匹配
项，那么Stub的最终步骤就是调用Generic Resolver并且调转到返回的目标方法。因为
Generic Resolver会插入<token,type,target>元组到缓存中，紧随其后使用<token,type
>进行调用会成功地在缓存中找到目标方法

当Dispatch Stub失败频率足够高时(也就是先前认为是单态)，这个调用地点会被认为是
多态的并且Resolve Stub会修补调用地点来直接指向Resolve Stub来避免由Dispatch
Stub 持续失败所造成的开销。在同步点(目前来说是一次GC的结尾)，多态调用地点会被随机
提升回单态调用地点，这是基于一个调用地点的多态特性通常是暂时的假设。如果对任意调用
地点的假设是错误的，那么它就会快速触发Backpatch将其再次降级为多态

对于每一个Token都会创建一个Resolve Stub，但是他们都使用全局缓存。每一个Token对
一个Stub允许快速，高效的哈希算法，其使用从<token,type>元组中不变的部分所派生
的预计算Hash。
##### 代码序列
先前提到的接口Virtual Table调度机制会导致像这样的代码序列

    调用者:                                   
    mov     eax,[ecx];ecx为this               
    mov     eax,[eax+Δ];Δ为Map指针的offset     
    mov     eax,[eax+Δ];Δ为ID                  
    call    [eax+Δ];Δ为Slot  ----------------->目标方法:

典型的Stub调度序列长成这样:

    调用者:                    Dispatch Stub                   
    mov     eax,[ecx] ------>  cmp      eax,expectedMT
    call    [addr]             jne      failure
                               jmp      target    ------------>目标方法:

这里expectedMT，failure标签地址，target是被编码进Stub的常量。
典型的Stub序列有与前面接口调度机制有相同数量的指令，并且更少的内存间接寻址可以
允许其在更小的工作集大小贡献下跑的更快。这也会导致JIT生成更少的代码，因为一部分
工作是在Stub中进行，而不是在调用地点进行。这仅仅在调用地点很少被访问时有优势。
需要注意的是，失败分支被安排在此以便于x86分支预测会跟随成功的情况。
### 目前状况
目前来说，VSD仅被用于接口方法调用而没有虚方法调用，原因参见上方介绍。对虚方法禁用
VSD的结果就是，每一个类型为自己的虚方法都有VTable并且在上面描述的实现表是被禁用的。
Dispatch Map仍然在担任接口方法调度的任务。

#### 代码解析续

    实际上这些内容应当在metod.hpp里面，因为他们与MethodTable没什么太大
    的关系

    //给定Interface及其MethodDescriptor，找到对应类型MethodDescriptor
    static MethodDesc *GetMethodDescForInterfaceMethodAndServer(TypeHandle ownerType, MethodDesc *pItfMD, OBJECTREF *pServer)
    {
        VALIDATEOBJECTREF(*pServer);
        MethodTable *pServerMT = (*pServer)->GetMethodTable();
        PREFIX_ASSUME(pServerMT != NULL);
    #ifdef FEATURE_ICASTABLE
        //在ICastable情况下，我们不是在当前的对象上查找接口函数实现
        //而是通过调用GetValueInternal()并且在其上再次调用GetMethodDescForInterfaceMethod()
        //这使得实现了ICastable的对象可以模仿其余类型的行为
        if (pServerMT->IsICastable() && 
            !pItfMD->HasMethodInstantiation() && 
            !TypeHandle(pServerMT).CanCastTo(ownerType))
        {
            GCStress<cfg_any>::MaybeTrigger();
            //进行ICastableHelpers.GetImplType托管函数的调用
            //进行准备
            PREPARE_NONVIRTUAL_CALLSITE(METHOD__ICASTABLEHELPERS__GETIMPLTYPE);
            //可能触发GC
            OBJECTREF ownerManagedType = ownerType.GetManagedClassObject(); 
            DECLARE_ARGHOLDER_ARRAY(args, 2);
            args[ARGNUM_0] = OBJECTREF_TO_ARGHOLDER(*pServer);
            args[ARGNUM_1] = OBJECTREF_TO_ARGHOLDER(ownerManagedType);
            OBJECTREF impTypeObj = NULL;
            CALL_MANAGED_METHOD_RETREF(impTypeObj, OBJECTREF, args);
            //ownerManagedType在调用期间不受保护
            INDEBUG(ownerManagedType = NULL);
            //返回默认值RuntimeTypeHandle
            if (impTypeObj == NULL)
            {
                COMPlusThrow(kEntryPointNotFoundException);
            }
            ReflectClassBaseObject* resultTypeObj = ((ReflectClassBaseObject*)OBJECTREFToObject(impTypeObj));
            TypeHandle resulTypeHnd = resultTypeObj->GetType();
            MethodTable *pResultMT = resulTypeHnd.GetMethodTable();
            RETURN(pResultMT->GetMethodDescForInterfaceMethod(ownerType, pItfMD, TRUE//冲突时抛出异常));
        }
    #endif
    #ifdef FEATURE_COMINTEROP
        if (pServerMT->IsComObjectType() && !pItfMD->HasMethodInstantiation())
        {
            pItfMD = MethodDesc::FindOrCreateAssociatedMethodDesc(
            pItfMD, 
            ownerType.GetMethodTable(), 
            FALSE,              // 强制封箱入口
            Instantiation(),    // 方法实体
            FALSE,              // 不允许实体类型参数
            TRUE);              // 强制远程方法
            RETURN(pServerMT->GetMethodDescForComInterfaceMethod(pItfMD, false));
        }
    #endif
        //处理纯CLR类型
        RETURN (pServerMT->GetMethodDescForInterfaceMethod(ownerType, pItfMD, TRUE));
    }

COM交互

    #ifdef FEATURE_COMINTEROP
    //给定Interface及其在COM上实现的MethodDescriptor，找到对应类型MethodDescriptor
    //如果fNullOk被设置，则null可以被允许作为返回值
    MethodDesc *GetMethodDescForComInterfaceMethod(MethodDesc *pItfMD, bool fNullOk)
    {
        MethodTable * pItfMT =  pItfMD->GetMethodTable();
        PREFIX_ASSUME(pItfMT != NULL);
        //处理没有动态Interface Map的__ComObject类
        if (!HasDynamicInterfaceMap())
        {
            RETURN(pItfMD);
        }
        else
        {
            //现在我们处理更复杂的可扩展RCW，第一个要检查的便是其
            //定义是否指定了一些实现了此Interface的类
            DWORD slot = (DWORD) -1;
            //使用GetTarget而不是FindDispatchImpl以利用缓存来提高速度
            PCODE tgt = VirtualCallStubManager::GetTarget(
            pItfMT->GetLoaderAllocator()->GetDispatchToken(pItfMT->GetTypeID(), pItfMD->GetSlot()), this, TRUE);
            if (tgt != NULL)
            {
                RETURN(MethodTable::GetMethodDescForSlotAddress(tgt));
            }
            //Interface没有在静态声明中，查找动态Interface
            else if (FindDynamicallyAddedInterface(pItfMT))
            {
                //此Interface被动态添加到类上，因此这是被COM
                //对象所实现的，我们像对待COM对象一样对待这些
                //动态添加的接口，也就是使用Interface VTable
                RETURN(pItfMD);
            }
        }
    }
    #endif

    //* 尝试部分解析约束调用，取决于泛型代码共享
    //* 需要注意的是，这实际上不会必要地解析到具体的调用上
    //  因为我们有可能正在进行共享的泛型代码编译
    //  这只会解析其到一个合适进行JIT的候选者
    //* 如果不能解析，返回null，因为其是在一个继承了System.Object
    //  或者System.ValueType的方法实现上进行调用
    //* 总是返回使用统一调用约定的未封箱入口点
    //==========================================================
    //* 找到实现了Interface对应MethodDescriptor的MethodDescriptor
    //* 注意：我们解析约束方法的能力是被泛型代码共享程度所影响的
    //* 返回值：MethodDescriptor可以被直接用作非虚函数调用
    //  如果必须使用VSD，那么就返回null
    MethodDesc * TryResolveConstraintMethodApprox(
        TypeHandle   ownerType, 
        MethodDesc * pMD, 
        BOOL *       pfForceUseRuntimeLookup = NULL)
    {
        //不是ValueType，直接返回null
        if (!IsValueType())
        {
            LOG((LF_JIT, LL_INFO10000, "TryResolveConstraintmethodApprox: not a value type %s\n", GetDebugClassName()));
            return NULL;
        }
        //如果我们在封箱的ValueType上调用函数，首先找到实现了约束(可能的泛型)方法
        MethodTable * pCanonMT = GetCanonicalMethodTable();
        MethodDesc * pGenInterfaceMD = pInterfaceMD->StripMethodInstantiation();
        MethodDesc * pMD = NULL;
        if (pGenInterfaceMD->IsInterface())
        {
            //有时候(当正在编译共享泛型代码时)
            //我们在JIT时没有足够确切的类型信息
            //甚至要决定我们能否解析到一个未封箱的入口点
            //一旦有任何的机会通过检查所有可能与此调用相兼容的接口
            //而让此情况发生时，我们总是通过帮助方法来应对这个情况
            MethodTable::InterfaceMapIterator it = pCanonMT->IterateInterfaceMap();
            DWORD cPotentialMatchingInterfaces = 0;
            //遍历所有可能的接口实例
            while (it.Next())
            {
                TypeHandle thPotentialInterfaceType(it.GetInterface());
                //比较相容接口
                if (thPotentialInterfaceType.AsMethodTable()->GetCanonicalMethodTable() ==
                    thInterfaceType.AsMethodTable()->GetCanonicalMethodTable())
                {
                    //计数自增
                    cPotentialMatchingInterfaces++;
                    //获取目标MethodDescriptor
                    pMD = pCanonMT->GetMethodDescForInterfaceMethod(thPotentialInterfaceType, pGenInterfaceMD, FALSE);
                    //非ValueType
                    if ((pMD != NULL) && !pMD->GetMethodTable()->IsValueType() && !pMD->IsInterface())
                    {
                        LOG((LF_JIT, LL_INFO10000, "TryResolveConstraintMethodApprox: %s::%s not a value type method\n",
                            pMD->m_pszDebugClassName, pMD->m_pszDebugMethodName));
                        return NULL;
                    }
                }
            }
            //检查数量
            _ASSERTE_MSG((cPotentialMatchingInterfaces != 0),
            "At least one interface has to implement the method, otherwise there's a bug in JIT/verification.");
            if (cPotentialMatchingInterfaces > 1)
            {
                //我们有很多个可能的接口匹配
                MethodTable * pInterfaceMT = thInterfaceType.GetMethodTable();
                _ASSERTE(pInterfaceMT->HasInstantiation());
                BOOL fIsExactMethodResolved = FALSE;
                if (!pInterfaceMT->IsSharedByGenericInstantiations() &&
                    !pInterfaceMT->IsGenericTypeDefinition() &&
                    !this->IsSharedByGenericInstantiations() &&
                    !this->IsGenericTypeDefinition())
                {
                    //我们有确切的接口与类型实例化(无泛型参数与代表性对象)
                    if (this->CanCastToInterface(pInterfaceMT))
                    {
                        //我们能够解析到对应的方法上
                        pMD = this->GetMethodDescForInterfaceMethod(pInterfaceMT, pInterfaceMD, FALSE);
                        fIsExactMethodResolved = pMD != NULL;
                    }
                }
                if (!fIsExactMethodResolved)
                {
                    //我们不能静态解析接口
                    _ASSERTE(pfForceUseRuntimeLookup != NULL);
                    //通知调用者应当使用运行时的查找
                    //需要注意的是，我们可以让pMD不正确，因为我们在运行时查找时会使用
                    *pfForceUseRuntimeLookup = TRUE;
                }
            }
            else
            {
                //如果我们能够解析到具体的方法上，那么就这样干
                //比如在运行时进行查找或者在没有共享的泛型代码时
                if (pCanonMT->CanCastToInterface(thInterfaceType.GetMethodTable()))
                {
                    pMD = pCanonMT->GetMethodDescForInterfaceMethod(thInterfaceType, pGenInterfaceMD, FALSE);
                    if (pMD == NULL)
                    {
                        LOG((LF_JIT, LL_INFO10000, "TryResolveConstraintMethodApprox: failed to find method desc for interface method\n"));
                    }
                }
            }
        }
        else if (pGenInterfaceMD->IsVirtual())
        {
            //ValueType且无VTableSlot
            if (pGenInterfaceMD->HasNonVtableSlot() && pGenInterfaceMD->GetMethodTable()->IsValueType())
            {
                //GetMethodDescForSlot对此Slot可能会引发AV
                //我们可以通过这些无效的并且未验证的IL到达这里
                //  constrained. int32
                //  callvirt System.Int32::GetHashCode()
                pMD = pGenInterfaceMD;
            }
            else
            {
                pMD = GetMethodDescForSlot(pGenInterfaceMD->GetSlot());
            }
        }
        else
        {
            //如果在System.Object上调用非虚的实例函数被用作约束时
            //pMD会变成null
            pMD = NULL;
        }
        if (pMD == NULL)
        {   
            //退回到VSD
            return NULL;
        }
        if (!pMD->GetMethodTable()->IsInterface())
        {
            //如果值类型自己声明了这个方法就返回方法
            //否则我们可能会从System.Object或者System.ValueType
            //得到一个方法
            if (!pMD->GetMethodTable()->IsValueType())
            {
                //退回到VSD
                return NULL;
            }
            pMD = MethodDesc::FindOrCreateAssociatedMethodDesc(
                pMD,
                this,
                FALSE//不要装箱的入口点Stub,
                pInterfaceMD->GetMethodInstantiation(),
                FALSE//不允许实例参数);
            _ASSERTE(pMD != NULL);
            _ASSERTE(!pMD->IsUnboxingStub());
        }
        return pMD;
    }

#### 协议实现



#### 终结(Finalization)语义

#### 静态字段

#### 实例化静态信息

#### 动态ID

#### 泛型字典信息

#### 对象

#### 枚举，委托，ValueType，数组

#### 底层元数据

#### 远程函数信息

#### 托管对象

#### GUID信息

#### 嵌套类-MethodData
#### 嵌套类-MethodDataObject
#### 嵌套类-MethodDataInterface
#### 嵌套类-MethodDataInterfaceImpl
#### 嵌套类-MethodIterator
#### 嵌套类-IntroducedMethodIterator

#### 实例字段