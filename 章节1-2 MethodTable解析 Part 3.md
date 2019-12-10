### Virtual/Interface调用解析
### VSD 机制介绍
VSD全称为Virtual Stub Dispatching，其使用Stub来用于虚函数的调用而不是使用传统的虚函数表。在过去，接口的调度要求接口拥有相对于进程独一无二(process unique)的标识符，并且每一个装载的接口都要被添加到全局的接口虚函数表Map中。这个要求意味着，所有的接口及其所有实现接口的类型于NGen情况下在运行时必须进行恢复，导致显著的启动工作集增大。使用Stub调度的动机是为了消除很大一部分的相关工作集，同时也将剩余的工作分布到进程的整个生命周期中。

尽管使用VSD来调度接口和虚函数调用是可以的，但目前仅仅被用于接口调度
### VSD依赖
#### 组件依赖
目前VSD代码相对于Runtime独立。其提供API供依赖组件使用，并且下方列出的依赖保证只涉及一块相对小的方面
##### 代码管理器
VSD依赖于代码管理器来提供方法的状态信息，确切的说，是任意方法是否转移到了其最终的状态以便于VSD可以根据细节决定Stub的生成与目标的缓存(指解析后的缓存)
##### 类型与方法
MethodTable持有调度Map的指针，其用于决定给定VSD调用的目标函数地址
##### 特殊类型
在COM交互类型上的调用必须使用自定义的调度，因为他们都有特殊化的目标解析方法
#### 依赖此组件的组件
##### 代码管理器
代码管理器依赖于VSD为JIT编译器提供接口调用的目标函数
##### 类构建器(Class Builder)
类构建器使用调度Map所暴露的API在类型构建时创建那些会被VSD代码在调度类型上使用的调度Map

### 设计目标与副作用
#### 目的
##### 工作集精简
接口调度先前的实现是使用一个很大的,甚至有点稀疏的Virtual Table查找来应对贯穿进程的接口标识符。此目标是为了通过按需产生调度Stub来减少工作集(使用频率较低)的数据量，在理论上保持相关的调用以及他们的调度Stub在一起并且增加工作集的数据紧凑度

需要注意且很重要的是，参与VSD的初始化工作集因为需要使用数据结构来在在系统运行时跟踪不同需要构建和回收的Stub而实际上对于每个调用的地方是多于一个的(工作集)；然而，当应用程序到达稳定状态时，这些数据结构将不再被简单的调度所需要，因此会得到回收。不幸的是，对于客户端程序来讲，这也就等于更慢的启动，这也是导致对虚函数禁用VSD的原因之一

##### 吞吐量的等价
保持接口和虚函数的调度与先前的Virtual Table调度机制在一个可平摊的吞吐量等价下是
相当重要的。

尽管很明显的是，使用接口调度可以做到这一点，但最后结果仍然是比传统虚函数调度慢了
一些，这也是一个对虚函数禁用VSD的原因

### Token的表示与调度Map的设计
调度Token是一个在运行时分配的本机字长大小的值，内部由一个表示接口和Slot的元组表示

此设计使用了分配的类型标识符值与Slot值结合。为了利用与Runtime的整合，实现也像传统Virtual Table分布那样来分配Slot索引。这意味着Runtime仍然可以一样地应对MethodTable，MethodDescriptor与Slot索引，除了Virtual Table必须使用帮助函数而不是直接访问来处理此抽象。

词Slot总是用于经典的Virtual Table被创建且被映射机制翻译的Slot索引值的情况也就是说，如果你要描述经典的虚函数Slot在MethodTable中的排布，Slot就意味着这是一个索引。理解这个差异是非常重要的，因为在Runtime代码中，Slot表示经典Virtual Table结构里的索引值和在Virtual Table里指针的地址。现有的变化是，Slot仅仅是一个索引值，代码指针的地址被包含在实现表中。

### MethodTable
#### 实现表
实现表是一个数组，其中每一个方法都是被类型所引入的，拥有指向方法入口点的指针，其成员按照如下顺序排布：

        * 引入(新Slot)的虚函数
        * 引入的非虚函数(实例函数和静态函数)
        * 重写的虚函数

使用此格式的原因是，其为传统V-Table的排布提供了一种自然的扩展。因此许多在SlotMap的入口能被此顺序和其他细节，比如总的虚和非虚方法数量所推断出来。

在当前Stub调度对虚方法禁用时，实现表实际上是不存在的，且被真正的V-Table所替代
 ，所有的映射结果都被表示成V-Table的Slot而非实现表的。

#### Slot Map
Slot Map是一个拥有多个形如<type,[<slot,scope,(index|slot)>]>>入口的表，其中类型使用了上面提到的动态分配的标识数来表示，其要么是表示当前类的守卫(sentinel)值(调用虚实例函数)，要么是被当前类型实现的接口标识符(或者是被其父类隐式地实现)子Slot Map，也就是被方括号所包围起来的，有一个或多个入口。其中第一个元素总是表示类型内的Slot，第二个元素Scope指定第三个元素是否为一个实现表的索引或是一个Slot索引Scope能够成为指示下一个元素用途的守卫值，其一就是下一个数字是否被解释为一个虚函数Slot并且应当被解析成虚函数调用this.slot。其二就是Scope也能标识在当前类型继承链上的特定类型。在这种情况下，第三个参数就是一个位于Scope所指示的类型的实现表的索引，并且这是一个type.slot最终的方法实现。

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

因此，看到如上的代码，Sub Map的第一列对应到经典的Virtual Table的索引(为了方便起见，我们省略掉了System.Object自带的4个函数)。寻找方法实现的顺序总是
自下而上。因此如果我们拥有一个B类型的对象，想要调用I.Foo方法，我们会从B的Slot Map中查找，发现没有以接口I的ID作为Key的，因此沿着继承链向上走，来到A的Slot Map，发现就在那里。其表示Foo的第一个Slot(值为0)，是被B中虚函数表索引为4的Slot所实现。接着返回到B的Slot Map并且查找Slot 4的实现，并且找到其位于实现表中的Slot 1。

#### 附加用途
很重要的一点是，此映射方法能够被用于实现虚方法Slot实现的重映射。因为Map的Scope能力，非虚函数也可以被引用。当Runtime想要支持用非虚函数实现接口时或许会非常有用。

#### 优化
Slot Map是按位来编码的，而且利用了经典接口使用Δ值来实现的优势，大大地减少了Map的大小。除此之外，新的Slot(无论虚或者非虚)都可以通过在实现表中的顺序被推测出来。如果表中含有新的虚函数Slot，其后跟随着非虚实例函数的Slot，其后又是重写函数，那么恰当的Slot Map 入口就可以通过其索引和各类Slot的数量被推测出来。所有这样可以被推测出来的入口都使用(*)号在上图被标注出来。现行的数据结构排布使用以下的方案，这里Dispatch Map仅当映射不能被使用顺序推断出来时才存在。

 MethodTable -> [DispatchMap ->] ImplementationTable

### 类型ID Map
此Map将类型映射到ID上，ID是单调增加的。目前来说，所有这样的类型都是接口。并且目前使用HashMap来实现，并且包含入口用于双向查找。

### 调度Token
调度Token都是<typeID,slot>这样的元组。对于接口，类型会是分配的接口ID对于虚方法，此值将会是一个表示此Slot会被类型内部所决议的目标函数Slot此值对在绝大情况下都和平台的本机字长一致，在x86上，有可能是每个值的低16位所连接起来的。这种情况能被泛化到处理溢出问题，就好像一个TypeHandle在运行时要么是MethodTable指针，要么是一个<TypeHandle,TypeHandle>对，使用守卫位来区分这两种情况。现在还没有决定这样做是否为必要的。
### VSD设计
#### 从调度Token到方法实现解析
给定一个Token和类型，方法的实现是通过映射Token到对应类型实现表的索引来查找。实现表必须从类型的MethodTable可达。此Map是被BuildMethodTable所创建，其遍历所有被此类型实现的接口来为类型构建MethodTable和决定每一个由此类型实现或者重写的接口方法。通过跟踪这些信息，在接口调度时期给定Token和对象来决定目标方法是可能的(对象中可获取MethodTable和Token映射)

#### Stub
接口调度调用通过Stub进行，这些Stub都是按需生成的，并且都有一个终极目标:匹配token和object到一个具体的方法实现，并且转发调用到那个方法实现。目前只有三种类型的Stub，他们分别是LookUp Stub，Dispatch Stub，Resolve Stub

##### Generic Resolver
这只是一个作为所有Stub失败的最终路径，其使用一个<token,type>的元组返回目标函数Generic Resolver也负责在需要时创建Dispatch和Resolve Stub，当更好的Stub可用时，为Indirection Cell替换Stub，缓存结果，还有其他的记录工作

##### Lookup Stub
这些stub首先被分配到接口调度的地方，并且在JIT编译接口调用的地方被创建。因为JIT在第一次调用执行之前完全不知道满足token的类型，这个stub就将token和类型作为参数传递给 Generic Resolver。如果有必要，Generic Resolver会创建Dispatch Stub和Resolve Stub，之后将调用地点进行打补丁，更新成Dispatch Stub因此Lookup Stub不会再被使用。

对每一个独一无二的token都会创建一个Lookup Stub(比如，对同一个接口Slot调用的调用地点会使用同一个Lookup Stub)

##### Dispatch Stub
这些Stub被用于认为行为上单态的调用地点，也就是用于某个调用地点的对象实际上都是一个类型，Dispatch Stub使用被调用的对象MethodTable并且与被缓存的类型比较，如果成功，就跳转到缓存的方法。在x86上这会导致一个"比较，条件失败跳转，跳转到目标"指令序列并且为任意Stub提供最佳的性能。如果Stub类型比较失败了，这就会跳转到对应的Resolve Stub

对每一个独一无二的<token,type>元组都生成一个Dispatch Stub，但会延迟创建，直到调用地点的LookUp Stub被调用时。

##### Resolve Stub
多态调用地点经由Resolve Stub处理。这些Stub用<token,type>来解析全局缓存中的目标方法，这里token在JIT时就可以确定，而类型在调用时可以确定。如果全局缓存不包含匹配项，那么Stub的最终步骤就是调用Generic Resolver并且调转到返回的目标方法。因为Generic Resolver会插入<token,type,target>元组到缓存中，紧随其后使用<token,type>进行调用会成功地在缓存中找到目标方法

当Dispatch Stub失败频率足够高时(也就是先前认为是单态)，这个调用地点会被认为是多态的并且Resolve Stub会修补调用地点来直接指向Resolve Stub来避免由DispatchStub 持续失败所造成的开销。在同步点(目前来说是一次GC的结尾)，多态调用地点会被随机提升回单态调用地点，这是基于一个调用地点的多态特性通常是暂时的假设。如果对任意调用地点的假设是错误的，那么它就会快速触发Backpatch将其再次降级为多态

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

这里expectedMT，failure标签地址，target是被编码进Stub的常量。典型的Stub序列有与前面接口调度机制有相同数量的指令，并且更少的内存间接寻址可以允许其在更小的工作集大小贡献下跑的更快。这也会导致JIT生成更少的代码，因为一部分工作是在Stub中进行，而不是在调用地点进行。这仅仅在调用地点很少被访问时有优势。需要注意的是，失败分支被安排在此以便于x86分支预测会跟随成功的情况。
### 目前状况
目前来说，VSD仅被用于接口方法调用而没有虚方法调用，原因参见上方介绍。对虚方法禁用VSD的结果就是，每一个类型为自己的虚方法都有VTable并且在上面描述的实现表是被禁用的。Dispatch Map仍然在担任接口方法调度的任务。

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
约束调用：词语来源于MSIL中的 constrained 指令，此指令被用于在应对值类型上调用引用类型的重写虚方法时应对更改自身的副作用。
比如:

    struct VT
    {
        public int i=0;
        public string ToString()
        {
            i++;
            return i.ToString();            
        }
    }
    代码如下:
    VT variable=new VT();
    Console.WriteLine(variable.ToString());
    Console.WriteLine(variable.i);
    结果:
    1
    1
这里我们知道的是，如果一个值类型实现或重写了接口或者System.Object的虚方法，那么在值类型上调用虚方法时，需要首先将对象的拷贝装箱到对应引用类型再调用。这种情况下，如果重写的成员方法对自身成员有副作用，那么必须反映到栈上的variable，而不是更改装箱的拷贝值。

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
        MethodDesc * pInterfaceMD, 
        BOOL *       pfForceUseRuntimeLookup = NULL)
    {
        //当前类型不是ValueType，直接返回null
        if (!IsValueType())
        {
            LOG((LF_JIT, LL_INFO10000, "TryResolveConstraintmethodApprox: not a value type %s\n", GetDebugClassName()));
            return NULL;
        }
        //如果我们在封箱的ValueType上调用函数，首先找到实现了约束(可能是泛型)的方法
        MethodTable * pCanonMT = GetCanonicalMethodTable();
        //去掉方法实例化，替换为一般性的泛型参数( Type<int>.Method<String> -> Type<int>.Method<U> )
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
                //比较相容接口(泛型或普通意义上等同的接口)
                if (thPotentialInterfaceType.AsMethodTable()->GetCanonicalMethodTable() ==
                    thInterfaceType.AsMethodTable()->GetCanonicalMethodTable())
                {
                    //可能的匹配接口计数自增
                    cPotentialMatchingInterfaces++;
                    //获取目标MethodDescriptor
                    pMD = pCanonMT->GetMethodDescForInterfaceMethod(thPotentialInterfaceType, pGenInterfaceMD, FALSE);
                    //解析到的实现方法的声明类型并不是ValueType且pMD也不是一个接口声明方法，解析失败
                    if ((pMD != NULL) && !pMD->GetMethodTable()->IsValueType() && !pMD->IsInterface())
                    {
                        LOG((LF_JIT, LL_INFO10000, "TryResolveConstraintMethodApprox: %s::%s not a value type method\n",
                            pMD->m_pszDebugClassName, pMD->m_pszDebugMethodName));
                        return NULL;
                    }
                }
            }
            //检查可能的匹配接口数量
            _ASSERTE_MSG((cPotentialMatchingInterfaces != 0),
            "At least one interface has to implement the method, otherwise there's a bug in JIT/verification.");
            if (cPotentialMatchingInterfaces > 1)
            {
                //我们有很多个可能的接口匹配
                MethodTable * pInterfaceMT = thInterfaceType.GetMethodTable();
                _ASSERTE(pInterfaceMT->HasInstantiation());
                //是否有确切的方法被解析到了
                BOOL fIsExactMethodResolved = FALSE;
                //接口MethodTable没有被泛型共享
                //接口MethodTable不是泛型定义
                //此类型也同上
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
            //如果Interface的对应MD
            //ValueType且拥有有非VTable的Slot
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
                //获取
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
关于constrainted 前缀指令，Roslyn会对接口的调用直接生成call，对值类型重写的System.Object的虚方法会生成 constrainted + callvirt，因为struct不可继承，也就不存在重写。

#### 协议实现
    //是否有调度Map
    inline BOOL HasDispatchMap()
    {
        return GetDispatchMap() != NULL;
    }

    PTR_DispatchMap GetDispatchMap()
    {
        MethodTable * pMT = this;
        if (!pMT->HasDispatchMapSlot())
        {
            //如果没有调度Map Slot，检查代表性MethodTable
            pMT = pMT->GetCanonicalMethodTable();
            if (!pMT->HasDispatchMapSlot())
                return NULL;
        }
        g_IBCLogger.LogDispatchMapAccess(pMT);
        //获取Slot起始地址
        TADDR pSlot = pMT->GetMultipurposeSlotPtr(enum_flag_HasDispatchMapSlot, c_DispatchMapSlotOffsets);
        return RelativePointer<PTR_DispatchMap>::GetValueAtPtr(pSlot);     
    }

    inline BOOL HasDispatchMapSlot()
    {
        return GetFlag(enum_flag_HasDispatchMapSlot);
    }

    void SetDispatchMap(DispatchMap *pDispatchMap)
    {
        //存在的情况下进行设置
        _ASSERTE(HasDispatchMapSlot());
        TADDR pSlot = GetMultipurposeSlotPtr(enum_flag_HasDispatchMapSlot, c_DispatchMapSlotOffsets);
        RelativePointer<DispatchMap *> *pRelPtr = (RelativePointer<DispatchMap *> *)pSlot;
        pRelPtr->SetValue(pDispatchMap);
    }

     BOOL FindEncodedMapDispatchEntry(UINT32 typeID,
                                     UINT32 slotNumber,
                                     DispatchMapEntry *pEntry)
    {
        //LookupDispatchMapType可能会抛出异常
        //但在当下，延迟接口恢复是被禁用的，因此应当永远不抛出
        CONSISTENCY_CHECK(HasDispatchMap());
        //获得调度Token所对应的类型
        MethodTable * dispatchTokenType = GetThread()->GetDomain()->LookupType(typeID);
        //寻找确切的类型匹配
        {
            //遍历调度Map
            DispatchMap::EncodedMapIterator it(this);
            for (; it.IsValid(); it.Next())
            {
                DispatchMapEntry * pCurEntry = it.Entry();
                //Slot索引能够对应(先根据索引过滤掉不可能项)
                if (pCurEntry->GetSlotNumber() == slotNumber)
                {
                    //获取Entry所对应类型
                    MethodTable * pCurEntryType = LookupDispatchMapType(pCurEntry->GetTypeID());
                    if (pCurEntryType == dispatchTokenType)
                    {
                        //比较成功，拷贝对应Entry，返回
                        *pEntry = *pCurEntry;
                        return TRUE;
                    }
                }
            }
        }
        //如果没有找到确切的匹配，且涉及了协变/逆变，重复这个过程，直到
        //能够匹配上一个CanCastTo
        //我们把上下分开的理由是我们想尽可能避免检查类型是否有协变/逆变
        //注意:对于有协变逆变参与的接口，CER是不被保证的。
        if (dispatchTokenType->HasVariance() || dispatchTokenType->HasTypeEquivalence())
        {
            DispatchMap::EncodedMapIterator it(this);
            for (; it.IsValid(); it.Next())
            {
                DispatchMapEntry * pCurEntry = it.Entry();
                if (pCurEntry->GetSlotNumber() == slotNumber)
                {
                    //仍然是先过滤Slot索引
        #ifndef DACCESS_COMPILE
                    MethodTable * pCurEntryType = LookupDispatchMapType(pCurEntry->GetTypeID());
                    //比较协变逆变接口或者委托
                    if (dispatchTokenType->HasVariance() && 
                    pCurEntryType->CanCastByVarianceToInterfaceOrDelegate(dispatchTokenType, NULL))
                    {
                        *pEntry = *pCurEntry;
                        return TRUE;
                    }
                    //如果有实例化并且有类型等效的参与
                    if (dispatchTokenType->HasInstantiation() && dispatchTokenType->HasTypeEquivalence())
                    {
                        //检查类型等效
                        if (dispatchTokenType->IsEquivalentTo(pCurEntryType))
                        {
                            *pEntry = *pCurEntry;
                            return TRUE;
                        }
                    }
        #endif
                }
        #if !defined(DACCESS_COMPILE) && defined(FEATURE_TYPEEQUIVALENCE)
                if (this->HasTypeEquivalence() && //类型等效
                    !dispatchTokenType->HasInstantiation() &&//无实例化 
                    dispatchTokenType->HasTypeEquivalence() && //类型等效
                    dispatchTokenType->GetClass()->IsEquivalentType())//等效类型
                {
                    //必为接口
                    _ASSERTE(dispatchTokenType->IsInterface());
                    MethodTable * pCurEntryType = LookupDispatchMapType(pCurEntry->GetTypeID());
                    //检查等效
                    if (pCurEntryType->IsEquivalentTo(dispatchTokenType))
                    {
                        //找出目标MethodDescriptor
                         MethodDesc * pMD = dispatchTokenType->GetMethodDescForSlot(slotNumber);
                         //是否放得下
                         _ASSERTE(FitsIn<WORD>(slotNumber));
                        BOOL fNewSlotFound = FALSE;
                        //获取等效的方法Slot
                        DWORD newSlot = GetEquivalentMethodSlot(
                        dispatchTokenType, 
                        pCurEntryType, 
                        static_cast<WORD>(slotNumber), 
                        &fNewSlotFound);
                        if (fNewSlotFound && (newSlot == pCurEntry->GetSlotNumber()))
                        {
                            //如果找到了，且Slot相等
                            MethodDesc * pNewMD = pCurEntryType->GetMethodDescForSlot(newSlot);
                            //检查MethodDescriptor的签名
                            MetaSig msig(pMD);
                            MetaSig msignew(pNewMD);
                            if (MetaSig::CompareMethodSigs(msig, msignew, FALSE))
                            {
                                *pEntry = *pCurEntry;
                                return TRUE;
                            }
                        }
                    }
                }
            }
        }
    }

    BOOL FindIntroducedImplementationTableDispatchEntry(UINT32 slotNumber,
                                                        DispatchMapEntry *pEntry,
                                                        BOOL fVirtualMethodsOnly);

上面这个方法目前还没有实现，不过可以看得出，实现表的调度在未来将会替代当前分割的调度方式。                    

    BOOL FindDispatchEntryForCurrentType(UINT32 typeID, 
                                         UINT32 slotNumber,
                                         DispatchMapEntry *pEntry)
    {
        BOOL fRes = FALSE;
        if (HasDispatchMap())
        {
            fRes = FindEncodedMapDispatchEntry(
            typeID, slotNumber, pEntry);
        }
        return fRes;
    }

需要特别提出的是，这里的对当前类型是通过协议的前置条件保证的。

    PRECONDITION(typeID != TYPE_ID_THIS_CLASS);

    BOOL FindDispatchEntry(UINT32 typeID,
                           UINT32 slotNumber,
                           DispatchMapEntry *pEntry)
    {
        //从派生链的最底部，当前类型开始查找合适的Entry，
        //这与之前的VSD描述是一致的
        MethodTable *pCurMT = this;
        UINT32 iCurInheritanceChainDelta = 0;
        while (pCurMT != NULL)
        {
            g_IBCLogger.LogMethodTableAccess(pCurMT);
            if (pCurMT->FindDispatchEntryForCurrentType(
                typeID, slotNumber, pEntry))
            {
                RETURN (TRUE);
            }
            pCurMT = pCurMT->GetParentMethodTable();
            iCurInheritanceChainDelta++;
        }
        RETURN (FALSE);
    }

查找调度实现

    //只有一些这样的可能情况
    //      1. 指明接口的协议
    //          a. 对于非虚函数实现，把调度Slot当作实现返回
    //          b. 在'this'上映射到虚方法Slot，需要进一步解析新的虚方法Slot
    //      2. 'this'协议
    //          a. 同上
    //          b. 在'this'上映射到另一个虚方法Slot，需要进一步解析this的新Slot
    BOOL FindDispatchImpl(
        UINT32         typeID, 
        UINT32         slotNumber, 
        DispatchSlot * pImplSlot,
        BOOL           throwOnConflict)
    {
        LOG((LF_LOADER, LL_INFO10000, "SD: MT::FindDispatchImpl: searching %s.\n", GetClass()->GetDebugClassName()));
        //接口协议
        DispatchMapEntry declEntry;
        DispatchMapEntry implEntry;
        if (typeID != TYPE_ID_THIS_CLASS)
        {
            DispatchMapEntry e;
            if (!FindDispatchEntry(typeID, slotNumber, &e))
            {
                //找出被调用的目标接口
                MethodTable *pIfcMT = GetThread()->GetDomain()->LookupType(typeID);
                //找出被调用的接口函数
                MethodDesc * pIfcMD = pIfcMT->GetMethodDescForSlot(slotNumber);
                //通过IList<T>(或者IEnumerable<T>或者ICollection<T>)调用Array方法
                //必须被特殊处理，这些接口使用了魔法(主要是考虑到工作集的问题，这些
                //接口都是在内部按需生成，即使语义上像静态的接口）
                //注意：目前泛型数组接口不支持CER
                if (IsArray())
                {
                    //在这里，我们知道我们正在尝试将一个数组转换到常规静态查找会失败
                    //的接口上
                    //此函数假定转换是合法的，因此我们现在可以假定这是一个通过IList<T>
                    //对Array方法的调用
                    //获取IList<T>或者IReadOnlyList<T>的MethodTable
                    //快速检查纯净性
                    if (!(pIfcMT->HasInstantiation()))
                    {
                        _ASSERTE(!"Should not have gotten here. If you did, it's probably because multiple interface instantiation hasn't been checked in yet. This code only works on top of that.");
                        RETURN(FALSE);
                    }
                    //获取泛型类型(像IList<T>中的T一样)
                    TypeHandle theT = pIfcMT->GetInstantiation()[0];
                    //获取SZArrayHelper的对应方法。这才是真正要执行的方法
                    //此方法会变为一个泛型方法的实例化。如果一个调用者要求
                    //调用IList<T>.Meth()，实际上会被引导至SZArrayHelper.Meth<T>()
                    MethodDesc * pActualImplementor = GetActualImplementationForArrayGenericIListOrIReadOnlyListMethod(pIfcMD, theT);
                    //现在构造一个调度Slot
                    DispatchSlot ds(pActualImplementor->GetMethodEntryPoint());
                    if (pImplSlot != NULL)
                    {
                        *pImplSlot = ds;
                    }
                    RETURN(TRUE);
                }
                else
                {
                    //试一试我们能否从实现的接口之一找到默认函数
                    MethodDesc *pDefaultMethod = NULL;
                    //尝试精确匹配
                    BOOL foundDefaultInterfaceImplementation  = FindDefaultInterfaceImplementation(
                        pIfcMD,//解析的MethodDescriptor
                        pIfcMT,//解析的接口类型
                        &pDefaultMethod,
                        FALSE, //不允许协变逆变
                        throwOnConflict);
                    //如果没有精确的匹配，尝试协变逆变匹配
                    if (!foundDefaultInterfaceImplementation && pIfcMT->HasVariance())
                    {
                        foundDefaultInterfaceImplementation = FindDefaultInterfaceImplementation(
                            pIfcMD,
                            pIfcMT,
                            &pDefaultMethod,
                            TRUE, //允许协变逆变
                            throwOnConflict);
                    }
                    if (foundDefaultInterfaceImplementation)
                    {
                        //如果默认的实现是抽象的，我们就遇上了重抽象
                        //例子：
                        //interface IFoo { void Frob(){...} } 
                        //interface IBar : IFoo { abstract void IFoo.Frob() }
                        //class Foo : IBar { //IFoo.Frob在这里没有实现 }
                        if (pDefaultMethod->IsAbstract())
                        {
                              if (throwOnConflict)
                              {
                                    //抛出
                                    ThrowExceptionForAbstractOverride(this, pIfcMT, pIfcMD);
                              }
                        }
                        else
                        {
                            //为默认实现构造Slot
                            DispatchSlot ds(pDefaultMethod->GetMethodEntryPoint());
                            if (pImplSlot != NULL)
                            {
                                *pImplSlot = ds;
                            }
                            RETURN(TRUE);
                        }
                    }
                }
                //否则此协议未被此类型或者其余父类所实现
                RETURN(FALSE);
            }
            //更新类型ID和Slot数以便于下方的全盘搜索可以进行
            typeID = TYPE_ID_THIS_CLASS;
            slotNumber = e.GetTargetSlotNumber();
        }
        //'this'协议，直接从VirtualTable中取出Slot
        *pImplSlot = GetRestoredSlot(slotNumber);
        RETURN (TRUE);
    }

对接口调度找到接口方法的默认实现或者找到最深重写的方法进行调用

    BOOL FindDefaultInterfaceImplementation(
        MethodDesc *pInterfaceMD,
        MethodTable *pObjectMT,
        MethodDesc **ppDefaultMethod,
        BOOL allowVariance,
        BOOL throwOnConflict)
    {
    #ifdef FEATURE_DEFAULT_INTERFACES
        InterfaceMapIterator it = this->IterateInterfaceMap();
        CQuickArray<MatchCandidate> candidates;
        unsigned candidatesCount = 0;
        candidates.AllocThrows(this->GetNumInterfaces());
        //从派生类到父类遍历接口
        //我们使用一种最简单的实现，因为在大多数情况下接口数量都很小
        //且接口调度的结果早已被缓存。如果有在非常复杂的接口继承体系
        //下调用默认的接口方法，我们就会再次访问
        MethodTable *pMT = this;
        while (pMT != NULL)
        {
            MethodTable *pParentMT = pMT->GetParentMethodTable();
            unsigned dwParentInterfaces = 0;
            if (pParentMT)
                dwParentInterfaces = pParentMT->GetNumInterfaces();
            //判断当前类型的接口数是否多于父类，否则就没有引入新的接口
            if (pMT->GetNumInterfaces() > dwParentInterfaces)
            {
                MethodTable::InterfaceMapIterator it = pMT->IterateInterfaceMapFrom(dwParentInterfaces);
                {
                    while (!it.Finished())
                    {
                        MethodTable *pCurMT = it.GetInterface();
                        MethodDesc *pCurMD = NULL;
                        if (pCurMT == pInterfaceMT)
                        {
                            //找到匹配
                            if (!pInterfaceMD->IsAbstract())
                            {
                                pCurMD = pInterfaceMD;
                            }
                        }
                        else if (pCurMT->CanCastToInterface(pInterfaceMT))
                        {
                            //可以进行转换
                            if (pCurMT->HasSameTypeDefAs(pInterfaceMT))
                            {
                                if (allowVariance && !pInterfaceMD->IsAbstract())
                                {
                                    //泛型协变逆变匹配，之后我们会用正确的参数将pCurMD实例化
                                    pCurMD = pInterfaceMD;
                                }
                            }
                            else
                            {
                                //一个更特化的接口，查找显式的重写
                                //在默认接方法中隐式重写是不被允许的
                                MethodIterator methodIt(pCurMT);
                                for (; methodIt.IsValid() && pCurMD == NULL; methodIt.Next())
                                {
                                    MethodDesc *pMD = methodIt.GetMethodDesc();
                                    int targetSlot = pInterfaceMD->GetSlot();
                                    //如果此MethodDescriptor不是Impl,那么一定不是我们想要的
                                    if (!pMD->IsMethodImpl())
                                        continue;
                                    //如果有Impl，遍历所有其实现的声明
                                    //查找我们需要的接口方法
                                    for (; it.IsValid() && pCurMD == NULL; it.Next())
                                    {
                                        MethodDesc *pDeclMD = it.GetMethodDesc();
                                        //是否为正确的Slot
                                        if (pDeclMD->GetSlot() != targetSlot)
                                            continue;
                                        //是否为正确的接口
                                        if (!pDeclMD->HasSameMethodDefAs(pInterfaceMD))
                                            continue;
                                        if (pInterfaceMD->HasClassInstantiation())
                                        {
                                            //pInterfaceMD会处在代表性的形式，因此
                                            //我们需要检查特化版本与pInterfaceMD
                                            //pDeclMD的父类在这种情况下并不可靠因
                                            //为其可能没有被代表化
                                            SigTypeContext typeContext = SigTypeContext(pCurMT);
                                            mdTypeRef tkParent;
                                            IfFailThrow(pMD->GetModule()->GetMDImport()->GetParentToken(it.GetToken(), &tkParent));
                                            MethodTable* pDeclMT = ClassLoader::LoadTypeDefOrRefOrSpecThrowing(
                                                pMD->GetModule(),
                                                tkParent,
                                                &typeContext).AsMethodTable();
                                            //我们使用CanCastToInterface来保证覆盖到了协变逆变
                                            //我们已经知道这是一个在同一个类型定义的泛型接口上的函数
                                            //但是我们需要确保实例化是匹配的
                                            if ((allowVariance && pDeclMT->CanCastToInterface(pInterfaceMT))
                                                || pDeclMT == pInterfaceMT)
                                            {
                                                //我们找到匹配了
                                                pCurMD = pMD;
                                            }
                                        }
                                        else
                                        {
                                            //在没有泛型参与的情况下，如果方法定义匹配上了，那么就是一个匹配
                                            pCurMD = pMD;
                                        }
                                    }
                                }
                            }
                        }
                        if(pCurMD != NULL)
                        {
                            //找到一个匹配，但是这是一个更具体的匹配(我们需要最具体化的接口匹配)
                            if (pCurMD->HasClassOrMethodInstantiation())
                            {
                                //实例化MethodDescriptor
                                //我们不想从此指针中获取泛型字典就需要传递神秘的类型参数
                                //从实例化stub到解析的二义性
                                pCurMD = MethodDesc::FindOrCreateAssociatedMethodDesc(
                                    pCurMD,
                                    pCurMT,
                                    FALSE,                  // 强制封箱入口点
                                    pCurMD->HasMethodInstantiation() ?
                                        pCurMD->AsInstantiatedMethodDesc()->IMD_GetMethodInstantiation() :
                                        Instantiation(),    // 对于那些本身就是泛型的函数
                                    FALSE,                  
                                    TRUE                    // 强制可远程方法
                                );
                                bool needToInsert = true;
                                bool seenMoreSpecific = false;
                                //我们需要维护一路上遇到的所有候选者中总是最具体化的，使其成为不变量
                                //可能有多个不相容的候选者
                                for (unsigned i = 0; i < candidatesCount; ++i)
                                {
                                    MethodTable *pCandidateMT = candidates[i].pMT;
                                    if (pCandidateMT == NULL)
                                        continue;
                                    if (pCandidateMT == pCurMT)
                                    {
                                        //遇到重复的，结束了
                                        needToInsert = false;
                                        break;
                                    }
                                    if (allowVariance && pCandidateMT->HasSameTypeDefAs(pCurMT))
                                    {
                                        //在同一个类型上有协变逆变的匹配 - 平局
                                    }
                                    else if (pCurMT->CanCastToInterface(pCandidateMT))
                                    {
                                        //pCurMT是一个比重写IBlah的IFoo/IBar更为具体化的选择
                                        if (!seenMoreSpecific)
                                        {
                                            //没有找到更具体化的，就设置当前的为
                                            seenMoreSpecific = true;
                                            candidates[i].pMT = pCurMT;
                                            candidates[i].pMD = pCurMD;
                                        }
                                        else
                                        {
                                            candidates[i].pMT = NULL;
                                            candidates[i].pMD = NULL;
                                        }
                                        needToInsert = false;
                                    }
                                    else if (pCandidateMT->CanCastToInterface(pCurMT))
                                    {
                                        //pCurMT更不具体化，那么我们就不需要扫描其余的Entry因为此Entry
                                        //能够代表pCurMT（其余的都与pCurMT不相容）
                                        needToInsert = false;
                                        break;
                                    }
                                    else
                                    {
                                        //不相容，继续扫描
                                    }
                                }
                                if (needToInsert)
                                {
                                    ASSERT(candidatesCount < candidates.Size());
                                    candidates[candidatesCount].pMT = pCurMT;
                                    candidates[candidatesCount].pMD = pCurMD;
                                    candidatesCount++;
                                }
                            }
                            it.Next();
                        }
                    }
                    pMT = pParentMT;
                }
                //扫描并且查看是否有冲突
                //如果我们在做第二遍(允许协变逆变)，我们不再寻找冲突了，仅仅选取第一个匹配就行了
                MethodTable *pBestCandidateMT = NULL;
                MethodDesc *pBestCandidateMD = NULL;
                for (unsigned i = 0; i < candidatesCount; ++i)
                {
                    if (candidates[i].pMT == NULL)
                        continue;
                    if (pBestCandidateMT == NULL)
                    {
                        pBestCandidateMT = candidates[i].pMT;
                        pBestCandidateMD = candidates[i].pMD;
                        //第二遍
                        if (allowVariance)
                            break;
                    }
                    else if (pBestCandidateMT != candidates[i].pMT)
                    {
                        if (throwOnConflict)
                            ThrowExceptionForConflictingOverride(this, pInterfaceMT, pInterfaceMD);
                        *ppDefaultMethod = NULL;
                        RETURN(FALSE);
                    }
                }
                if (pBestCandidateMD != NULL)
                {
                    *ppDefaultMethod = pBestCandidateMD;
                    RETURN(TRUE);
                }
            }
        }
    #else
        *ppDefaultMethod = NULL;
    #endif
        RETURN(FALSE);
    }

找到对应的调度Slot

    DispatchSlot FindDispatchSlot(UINT32 typeID, UINT32 slotNumber, BOOL throwOnConflict)
    {
        DispatchSlot implSlot(NULL);
        FindDispatchImpl(typeID, slotNumber, &implSlot, throwOnConflict);
        return implSlot;
    }

如果pMD有可能是泛型接口的函数的话，你必须使用下面这两个函数的第二个。ownerType被用于在pMD是共享的MethodDescriptor时提供具体的资格

    DispatchSlot FindDispatchSlotForInterfaceMD(MethodDesc *pMD, BOOL throwOnConflict)
    {
        CONSISTENCY_CHECK(CheckPointer(pMD));
        CONSISTENCY_CHECK(pMD->IsInterface());
        //转发给第二个重载
        return FindDispatchSlotForInterfaceMD(TypeHandle(pMD->GetMethodTable()), pMD, throwOnConflict);
    }
    DispatchSlot FindDispatchSlotForInterfaceMD(TypeHandle ownerType, MethodDesc *pMD, BOOL throwOnConflict)
    {
        CONSISTENCY_CHECK(!ownerType.IsNull());
        CONSISTENCY_CHECK(CheckPointer(pMD));
        CONSISTENCY_CHECK(pMD->IsInterface());
        return FindDispatchSlot(ownerType.GetMethodTable()->GetTypeID(), pMD->GetSlot(), throwOnConflict);
    }

此函数被用于通过ComPlusMethodCall的Method Descriptor反向查找方法实现
此函数有以下假定:
         1.此函数实现是对 InterfaceToken->Slot索引 的实现
         2.对此Slot索引只有一个这样的映射
         3.此映射存在于此类型，而不是父类型中

    MethodDesc *ReverseInterfaceMDLookup(UINT32 slotNumber)
    {
        DispatchMap::Iterator it(this);
        //遍历调度图
        for (; it.IsValid(); it.Next())
        {
            if (it.Entry()->GetTargetSlotNumber() == slotNumber)
            {
                DispatchMapTypeID typeID = it.Entry()->GetTypeID();
                _ASSERTE(!typeID.IsThisClass());
                UINT32 slotNum = it.Entry()->GetSlotNumber();
                MethodTable * pMTItf = LookupDispatchMapType(typeID);
                CONSISTENCY_CHECK(CheckPointer(pMTItf));
                MethodDesc *pCanonMD = pMTItf->GetMethodDescForSlot((DWORD)slotNum);
                return MethodDesc::FindOrCreateAssociatedMethodDesc(
                        pCanonMD, 
                        pMTItf, 
                        FALSE,              // 强制封箱入口
                        Instantiation(),    
                        FALSE,              
                        TRUE);  
            }
        }
        return NULL;
    }

获取查找类型的ID，如果没有被赋值，则不会对其赋值

    UINT32 LookupTypeID()
    {
        PTR_MethodTable pMT = PTR_MethodTable(this);
        return GetDomain()->LookupTypeID(pMT);
    }

获取查找类型的ID，如果没有被赋值，则会对其赋值

    UINT32 GetTypeID()
    {
        PTR_MethodTable pMT = PTR_MethodTable(this);
        return GetDomain()->GetTypeID(pMT);
    }

在调度Map中根据Type ID查找类型

    MethodTable *LookupDispatchMapType(DispatchMapTypeID typeID)
    {
        _ASSERTE(!typeID.IsThisClass());
        InterfaceMapIterator intIt = IterateInterfaceMapFrom(typeID.GetInterfaceNum());
        return intIt.GetInterface();
    }

获取类型引入的方法Method Descriptor

    MethodDesc *GetIntroducingMethodDesc(DWORD slotNumber)
    {
        MethodDesc * pCurrentMD = GetMethodDescForSlot(slotNumber);
        DWORD        dwSlot = pCurrentMD->GetSlot();
        MethodDesc * pIntroducingMD = NULL;
        MethodTable * pParentType = GetParentMethodTable();
        MethodTable * pPrevParentType = NULL;
        //遍历继承链，如果Slot数在Virtual-Table数之外(也就是在其之后，为引入的方法，就停止)
        //如果不存在于父类之中，那么一定会在相同的V-Table Slot处
        while ((pParentType != NULL) &&
           (dwSlot < pParentType->GetNumVirtuals()))
        {
            pPrevParentType = pParentType;
            pParentType = pParentType->GetParentMethodTable();
        }
        if (pPrevParentType != NULL)
        {
            pIntroducingMD = pPrevParentType->GetMethodDescForSlot(dwSlot);
        }
        return pIntroducingMD;
    }

决定给定的接口里所有的方法是否在当前类型的父类中拥有最终实现。如果返回true的话，那么就可以直接在此类型
上调度函数而不是使用VSD

    BOOL ImplementsInterfaceWithSameSlotsAsParent(MethodTable *pItfMT, MethodTable *pParentMT)
    {
        MethodTable *pMT = this;
        do
        {
            DispatchMap::EncodedMapIterator it(pMT);
            for (; it.IsValid(); it.Next())
            {
                DispatchMapEntry *pCurEntry = it.Entry();
                if (LookupDispatchMapType(pCurEntry->GetTypeID()) == pItfMT)
                {
                    //此类型与其父类型一直到pParentMT一定没有对此接口的映射
                    return FALSE;
                }
            }
             pMT = pMT->GetParentMethodTable();
            _ASSERTE(pMT != NULL);
        }
        while(pMT != pParentMT);
        return TRUE;
    }

决定是否所有在给定接口中的方法都在其一个父类型中拥有最终实现。如果此函数返回true，那么此类型在接口函数调度时就与父类型的行为一致

    BOOL HasSameInterfaceImplementationAsParent(MethodTable *pItfMT, MethodTable *pParentMT)
    {
        if (!ImplementsInterfaceWithSameSlotsAsParent(pItfMT, pParentMT))
        {
            //如果Slot都不一样，那么此类型一定重新实现了接口
            return FALSE;
        }
        //即使目标的Slot都是一样的，他们仍然能被重写。我们会从继承链的pParentMT开始迭代整个调度Map
        //并且对于每一个接口的MethodTable的Entry进行内容检查(pParentMT和此类型)。一次匹配出错
        //意味着有一个重写。我们会跟踪我们所看到的所有Slot的源头(接口)以便于我们可以忽略在pParentMT
        //级别上已经没有效果的Entry(那些在继承链更高地方的Entry，他们已经被重写了)
        BitMask bitMask;
        WORD wSeenSlots = 0;
        WORD wTotalSlots = pItfMT->GetNumVtableSlots();
        MethodTable *pMT = pParentMT;
        do
        {
            DispatchMap::EncodedMapIterator it(pMT);
            for (; it.IsValid(); it.Next())
            {
                DispatchMapEntry *pCurEntry = it.Entry();
                if (LookupDispatchMapType(pCurEntry->GetTypeID()) == pItfMT)
                {
                    UINT32 ifaceSlot = pCurEntry->GetSlotNumber();
                    //检查是否访问过
                    if (!bitMask.TestBit(ifaceSlot))
                    {
                        bitMask.SetBit(ifaceSlot);
                        UINT32 targetSlot = pCurEntry->GetTargetSlotNumber();
                        if (GetRestoredSlot(targetSlot) != pParentMT->GetRestoredSlot(targetSlot))
                        {
                            //目标Slot已经被重写
                            return FALSE;
                        }
                        if (++wSeenSlots == wTotalSlots)
                        {
                            //解析过所有Slot后我们没有理由再继续下去
                            break;
                        }
                    }
                }
            }
            pMT = pMT->GetParentMethodTable();
        }
        while (pMT != NULL);
        return TRUE;
    }
将方法声明映射到实现

    static MethodDesc *MapMethodDeclToMethodImpl(MethodDesc *pMDDecl)
    {
        MethodTable * pMT = pMDDecl->GetMethodTable();
        //快速失败检查
        //如果这个方法不是虚方法，那么其不可能被映射到对应的方法实现上
        if (!pMDDecl->IsVirtual() ||
            //其是否为一个调用实例化Stub的非虚Stub
            (pMT->IsValueType() && !pMDDecl->IsUnboxingStub()))
        {
            return pMDDecl;
        }
        MethodDesc * pMDImpl = pMT->GetParallelMethodDesc(pMDDecl);
        //如果方法被实例化了，因此我们需要为新Slot索引解析其对应的实例化Method Descriptor
        if (pMDDecl->HasMethodInstantiation())
        {
            if (pMDDecl->GetSlot() != pMDImpl->GetSlot())
            {
                if (!pMDDecl->IsGenericMethodDefinition())
                {
        #ifndef DACCESS_COMPILE
                    pMDImpl = pMDDecl->FindOrCreateAssociatedMethodDesc(
                                        pMDImpl,
                                        pMT,
                                        pMDDecl->IsUnboxingStub(),
                                        pMDDecl->GetMethodInstantiation(),
                                        pMDDecl->IsInstantiatingStub());
        #else
                    DacNotImpl();
        #endif
                }
            }
        }
        else
        {
            //因为泛型方法定义总是在MethodTable的真实Slot上，并且因为对实现和定义的Slot
            //都是一样的，那么调用FindOrCreateAssociatedMethodDesc会导致返回相同的
            //pMDDecl，在这种情况下，我们可以跳过这个工作
            pMDImpl = pMDDecl;
        }
        CONSISTENCY_CHECK(CheckPointer(pMDImpl));
        CONSISTENCY_CHECK(!pMDImpl->IsGenericMethodDefinition());
        return pMDImpl;
    }
#### 终结(Finalization)语义
在没有Finalizer的情况下可以使用快速助手

    DWORD  CannotUseSuperFastHelper()
    {
        return HasFinalizer();
    }

    void SetHasFinalizer()
    {
        SetFlag(enum_flag_HasFinalizer);
    }
这个类型是否有非平凡的终结要求？

    DWORD HasFinalizer()
    {
        return GetFlag(enum_flag_HasFinalizer);
    }
在AppDomain被强制卸载时此类型是否需要被终结?
并且此类型的终结器必须使用与其他正常终结器不同的顺序么?

    DWORD HasCriticalFinalizer() const
    {
        return GetFlag(enum_flag_HasCriticalFinalizer);
    }

#### 静态字段
以下4个函数均为DAC编译且转发到Module的返回

    inline PTR_BYTE GetNonGCStaticsBasePointer();//从略
    inline PTR_BYTE GetGCStaticsBasePointer();//从略

    inline PTR_BYTE GetNonGCThreadStaticsBasePointer()
    {
        //获取当前线程
        PTR_Thread pThread = dac_cast<PTR_Thread>(GetThread());
        //获取当前模块的索引
        ModuleIndex index = GetModuleForStatics()->GetModuleIndex();
        //获取线程当前的本地Block
        PTR_ThreadLocalBlock pTLB = ThreadStatics::GetCurrentTLB(pThread);
        //获取线程本地Module
        PTR_ThreadLocalModule pTLM = pTLB->GetTLMIfExists(index);
        if (pTLM == NULL)
            return NULL;
        return pTLM->GetNonGCStaticsBasePointer(this);
    }
对于下面这个函数也是雷同的，通过获取线程本地(threadlocal 关键字)的Module来得到静态变量的基地址

    inline PTR_BYTE GetGCThreadStaticsBasePointer();

下面的四个函数都是通过Test或者SetFlag来转发实现的，从略

    inline DWORD IsDynamicStatics();
    inline void SetDynamicStatics(BOOL fGeneric);
    inline void SetHasBoxedRegularStatics();
    inline DWORD HasBoxedRegularStatics();
此函数转发到EEClass

    DWORD HasFixedAddressVTStatics();
好像混入了什么奇怪的东西(同转发到EEClass)

    BOOL HasOnlyAbstractMethods();

#### 实例化静态信息
    void SetupGenericsStaticsInfo(FieldDesc* pStaticFieldDescs)
    {
        //没有必要为开放类型生成ID，实际上我们没有把他们存放于NGen映像中
        //因为这样做是错误的
        //然而，我们在MethodTable的可选成员中为ID设置为-1
        GenericsStaticsInfo *pInfo = GetGenericsStaticsInfo();
        if (!ContainsGenericVariables() && !IsSharedByGenericInstantiations())
        {
            //不是泛型实例共享的部分
            //不含有泛型参数，即为普通类型
            Module * pModuleForStatics = GetLoaderModule();
            pInfo->m_DynamicTypeID = pModuleForStatics->AllocateDynamicEntry(this);
        }
        else
        {
            pInfo->m_DynamicTypeID = (SIZE_T)-1;
        }
        pInfo->m_pFieldDescs.SetValueMaybeNull(pStaticFieldDescs);
    }

以下函数均为Flag获取或者转发到EEClass

    BOOL HasGenericsStaticsInfo();
    PTR_FieldDesc GetGenericsStaticFieldDescs();
    BOOL HasCrossModuleGenericStaticsInfo();
    PTR_Module GetGenericsStaticsModuleAndID(DWORD * pID);
    WORD GetNumHandleRegularStatics();
    WORD GetNumBoxedRegularStatics ();
    WORD GetNumBoxedThreadStatics ();

#### 动态ID
此API被用作泛型和内存里的反射Emit

    DWORD GetModuleDynamicEntryID()
    {
        if (HasGenericsStaticsInfo())
        {
            DWORD dwDynamicClassDomainID;
            GetGenericsStaticsModuleAndID(&dwDynamicClassDomainID);
            return dwDynamicClassDomainID;
        }
        else
        {
            return GetClass()->GetModuleDynamicID();
        }
    }
    Module* GetModuleForStatics()
    {
        if (HasGenericsStaticsInfo())
        {
            DWORD dwDynamicClassDomainID;
            return GetGenericsStaticsModuleAndID(&dwDynamicClassDomainID);
        }
        else
        {
            return GetLoaderModule();
        }
    }
#### 泛型字典信息
    //获取泛型参数个数
    inline DWORD GetNumGenericArgs()
    {
        LIMITED_METHOD_DAC_CONTRACT;
        if (HasInstantiation())
            return (DWORD) (GetGenericsDictInfo()->m_wNumTyPars);
        else
            return 0;
    }
    //获取Dictionary的数量(关于Dictionary参见上面的注释)
    inline DWORD GetNumDicts()
    {
        LIMITED_METHOD_DAC_CONTRACT;
        if (HasPerInstInfo())
        {
            PTR_GenericsDictInfo  pDictInfo = GetGenericsDictInfo();
            return (DWORD) (pDictInfo->m_wNumDicts);
        }
        else
            return 0;
    }
#### 对象
    OBJECTREF Allocate()
    {
        //当被完全加载时才能进行内存分配
        CONSISTENCY_CHECK(IsFullyLoaded());
        //
        EnsureInstanceActive();
        if (HasPreciseInitCctors())
        {
            //类初始化构造函数
            CheckRunClassInitAsIfConstructingThrowing();
        }
        return AllocateObject(this);
    }
下面这个函数更为高效，但仅能用于IsRestored()，CheckInstanceActivated(), IsClassInited()都为true时。充分条件是在此AppDomain内已经有相同类型的实例存在。此函数目前只在Delegate.Combine内部通过COMDelegate::InternalAllocLike调用

    OBJECTREF AllocateNoChecks()
    {
        CONSISTENCY_CHECK(IsRestored_NoLogging());
        CONSISTENCY_CHECK(CheckInstanceActivated());
        return AllocateObject(this);
    }

接下来是几个有关拆装箱的函数。
有关函数需要处理Nullable的特殊情况

    OBJECTREF Box(void* data)
    {
        OBJECTREF ref;
        GCPROTECT_BEGININTERIOR (data);
        //是否为ref struct
        if (IsByRefLike())
        {
            //永远都不应该对一个含有栈指针的类型装箱
            COMPlusThrow(kInvalidOperationException, W("InvalidOperation_TypeCannotBeBoxed"));
        }
        ref = FastBox(&data);
        GCPROTECT_END ();
        return ref;
    }
    OBJECTREF FastBox(void** data)
    {
        //检查是否为Nullable，如果是，将其转发到Nullable::Box
        if (IsNullable())
            return Nullable::Box(*data, this);
        //分配对象
        OBJECTREF ref = Allocate();
        //拷贝值类型
        CopyValueClass(ref->UnBox(), *data, this);
    }
    BOOL UnBoxInto(void *dest, OBJECTREF src)
    {
        //转发
        if (Nullable::IsNullableType(TypeHandle(this)))
            return Nullable::UnBoxNoGC(dest, src, this);
        else  
        {
            if (src == NULL || src->GetMethodTable() != this)
                return FALSE;
            //直接拷贝
            CopyValueClass(dest, src->UnBox(), this);
        }
        return TRUE;
    }
拆箱到参数上，这个API很容易就可以被关系到实际应用场景:Delegate.DynamicInvoke

    BOOL UnBoxIntoArg(ArgDestination *argDest, OBJECTREF src)
    {
        if (Nullable::IsNullableType(TypeHandle(this)))
            return Nullable::UnBoxIntoArgNoGC(argDest, src, this);
        else  
        {   
            if (src == NULL || src->GetMethodTable() != this)
                return FALSE;
            CopyValueClassArg(argDest, src->UnBox(), this, 0);
        }
        return TRUE;
    }
无检查快速拆箱

    void UnBoxIntoUnchecked(void *dest, OBJECTREF src)
    {
        if (Nullable::IsNullableType(TypeHandle(this))) {
        BOOL ret;
        ret = Nullable::UnBoxNoGC(dest, src, this);
        _ASSERTE(ret);
        }
        else  
        {
            _ASSERTE(src->GetMethodTable()->GetNumInstanceFieldBytes() == GetNumInstanceFieldBytes());
            CopyValueClass(dest, src->UnBox(), this);
        }
    }
判断是否为敏捷并且可执行Finalize的类型

    inline BOOL IsAgileAndFinalizable()
    {
        //目前只有System.Thread满足这个条件
        //这个条件应当一直保持，请不要在没有和EE团队谈过之前更改
        return this == g_pThreadClass;
    }

#### 枚举，委托，ValueType，数组
元素类型的种类:
GetInternalCorElementType()获取类型的内部表示。使用此函数不一定总是恰当的。比如，我们将枚举看作其底层类型或者某些结构体被优化成了int的组合。为了得到签名类型或者验证者类型(与签名类型相同，除了枚举被归一化到其实现的底层基元类型)，使用TypeHandle.h里的这些API:

    TypeHandle.GetSignatureCorElementType()
    TypeHandle.GetVerifierCorElementType()
    TypeHandle.GetInternalCorElementType()
此函数会:
-返回枚举的底层类型
-为System.Int32等返回底层基元类型
-会返回在调用约定中被使用到的底层基元类型,比如对于
struct type{public int i;} 在x86下会返回ELEMENT_TYPE_I4而不是ELEMENT_TYPE_VALUETYPE, 我们将这种类型的值类型称为基元值类型。内部表示被用于调用约定之中(JIT从基元值类型中受益)或者优化封送。此函数不会转换E_T_ARRAY,E_T_SZARRAY等到E_T_CLASS(即使应当这么做)

    CorElementType GetInternalCorElementType()
    {
        //此函数不应当碰EEClass，至少在ELEMENT_TYPE_CLASS和ELEMENT_TYPE_VALUETYPE这种
        //通常情况下不应该
        g_IBCLogger.LogMethodTableAccess(this);
        CorElementType ret;
        switch (GetFlag(enum_flag_Category_ElementTypeMask))
        {
        case enum_flag_Category_Array:
            ret = ELEMENT_TYPE_ARRAY;
            break;
        case enum_flag_Category_Array | enum_flag_Category_IfArrayThenSzArray:
            ret = ELEMENT_TYPE_SZARRAY;
            break;
        case enum_flag_Category_ValueType:
            ret = ELEMENT_TYPE_VALUETYPE;
            break;
        case enum_flag_Category_PrimitiveValueType:
            //此路径应当只被mscorlib的内建类型和基元值类型所访问
            ret = GetClass()->GetInternalCorElementType();
            _ASSERTE((ret != ELEMENT_TYPE_CLASS) && 
                        (ret != ELEMENT_TYPE_VALUETYPE));
            break;
        default:
            ret = ELEMENT_TYPE_CLASS;
            break;
        }
        return ret;
    }

    void SetInternalCorElementType(CorElementType _NormType)
    {
        switch (_NormType)
        {
            case ELEMENT_TYPE_CLASS:
            _ASSERTE(!IsArray());
            //什么都不需要做
            break;
        case ELEMENT_TYPE_VALUETYPE:
            SetFlag(enum_flag_Category_ValueType);
            _ASSERTE(GetFlag(enum_flag_Category_Mask) == enum_flag_Category_ValueType);
            break;
        default:
            SetFlag(enum_flag_Category_PrimitiveValueType);
            _ASSERTE(GetFlag(enum_flag_Category_Mask) == enum_flag_Category_PrimitiveValueType);
            break;
        }
        GetClass_NoLogging()->SetInternalCorElementType(_NormType);
        _ASSERTE(GetInternalCorElementType() == _NormType);
    }
此帮助函数会返回GetInternalCorElementType一样的类型，除了枚举类型会返回底层类型

    CorElementType GetVerifierCorElementType()
    {
        g_IBCLogger.LogMethodTableAccess(this);
        CorElementType ret;
        switch (GetFlag(enum_flag_Category_ElementTypeMask))
        {
        case enum_flag_Category_Array:
            ret = ELEMENT_TYPE_ARRAY;
            break;
        case enum_flag_Category_Array | enum_flag_Category_IfArrayThenSzArray:
            ret = ELEMENT_TYPE_SZARRAY;
            break;
        case enum_flag_Category_ValueType:
            ret = ELEMENT_TYPE_VALUETYPE;
            break;
        case enum_flag_Category_PrimitiveValueType:
            //这是唯一与MethodTable::GetInternalCorElementType()所不同的地方
            if (IsTruePrimitive() || IsEnum())
                ret = GetClass()->GetInternalCorElementType();
            else
                ret = ELEMENT_TYPE_VALUETYPE;            
            break;
        default:
            ret = ELEMENT_TYPE_CLASS;
            break;
        }
        return ret;
    }
返回你在签名里需要使用的ELELEMT_TYPE_*。这其中唯一发生的归一化是针对实例化的类型(比如 List<String>或者值类型Pair<int,int>)此函数要么返回ELEMENT_TYPE_CLASS要么返回ELEMENT_TYPE_VALUE,而不是ELEMENT_TYPE_WITH

    CorElementType GetSignatureCorElementType()
    {
        g_IBCLogger.LogMethodTableAccess(this);
        CorElementType ret;
        switch (GetFlag(enum_flag_Category_ElementTypeMask))
        {
        case enum_flag_Category_Array:
            ret = ELEMENT_TYPE_ARRAY;
            break;
        case enum_flag_Category_Array | enum_flag_Category_IfArrayThenSzArray:
            ret = ELEMENT_TYPE_SZARRAY;
            break;
        case enum_flag_Category_ValueType:
            ret = ELEMENT_TYPE_VALUETYPE;
            break;
        case enum_flag_Category_PrimitiveValueType:
            //这是唯一与MethodTable::GetInternalCorElementType()所不同的地方
            if (IsTruePrimitive())
                ret = GetClass()->GetInternalCorElementType();
            else
                ret = ELEMENT_TYPE_VALUETYPE;            
            break;
        default:
            ret = ELEMENT_TYPE_CLASS;
            break;
        }
        return ret;
    }
真正的基元类型是:类型的GetVerifierCorElementType()是ELEMENT_TYPE_I,ELEMENT_TYPE_I4,ELEMENT_TYPE_TYPEDBYREF之类的。需要注意的是，GetIntenalCorElementType可能对一些附加类型如枚举和一些结构体返回相同的值。

    BOOL IsTruePrimitive()
    {
        return GetFlag(enum_flag_Category_Mask) == enum_flag_Category_TruePrimitive;
    }

     void SetIsTruePrimitive()
     {
        SetFlag(enum_flag_Category_TruePrimitive);
     }
是否为委托，对于System.Delegate和System.MulticastDelegate返回false

     inline BOOL IsDelegate()
     {
        //我们不再允许单播委托了，仅仅检查多播委托
         _ASSERTE(g_pMulticastDelegateClass);
        return ParentEquals(g_pMulticastDelegateClass);
     }
是否为System.Object

    inline BOOL IsObjectClass()
    {
        _ASSERTE(g_pObjectClass);
        return (this == g_pObjectClass);
    }
是否为System.ValueType

    inline DWORD IsValueTypeClass()
    {
        _ASSERTE(g_pValueTypeClass);
        return (this == g_pValueTypeClass);
    }
是否为值类型，对System.ValueType和System.Enum返回false

    inline BOOL IsValueType()
    {
        g_IBCLogger.LogMethodTableAccess(this);
        return GetFlag(enum_flag_Category_ValueType_Mask) == enum_flag_Category_ValueType;
    }

此函数返回true，当且仅当此类型的返回buffer必须是栈分配的。这个通常只会在struct包含有GC指针时成立，并且不超过一些大小的限制。把此当作不变量允许了一个优化动作:JIT可能假定对于此函数返回true的类型的buffer指针总是栈上分配的，因此，储存到GC指针的字段不需要GC写屏障。

    BOOL IsStructRequiringStackAllocRetBuf()
    {
        //禁用此优化，其有限制的值(仅仅在x86上有效，且仅针对那些不太常见的结构体)
        //且会导致bug并且引入与ReadyToRun不相容的奇怪ABI差异
        return FALSE;
    }
是否为Enum，对System.Enum返回false

    inline BOOL IsEnum()
    {
        //此函数不应当在父Method Table有效前被调用
        _ASSERTE_IMPL(IsParentMethodTablePointerValid());
        PTR_MethodTable pParentMT = GetParentMethodTable();
        //确保不是在启动时使用此函数
        _ASSERTE(g_pEnumClass != NULL);
        return (pParentMT == g_pEnumClass);
    }
是否为Array

    inline BOOL IsArray()
    {
        return GetFlag(enum_flag_Category_Array_Mask) == enum_flag_Category_Array; 
    }
如果此类型对某些类型是Nullable<T>则返回true

    inline BOOL IsNullable()
    {
        return GetFlag(enum_flag_Category_Mask) == enum_flag_Category_Nullable;
    }
结构体是否可以被封送

    inline BOOL IsStructMarshalable() 
    {
        PRECONDITION(!IsInterface());
        //不是接口即可以被封送
        return GetFlag(enum_flag_IfNotInterfaceThenMarshalable); 
    }

    inline void SetStructMarshalable()
    {
        PRECONDITION(!IsInterface());
        SetFlag(enum_flag_IfNotInterfaceThenMarshalable);
    }
接下来的函数仅对数组类型有效。这些Method Table可能被多个数组类型锁共享，因此GetArrayElementTypeHandle可能仅仅是一个大概的类型。如果你需要确切的元素类型那么你应当在TypeHandle对象，ArrayTypeDesc或者一个已知是Array的对象(比如BASEARRAYREF)的引用上调用GetArrayElementTypeHandle函数

    CorElementType GetArrayElementType()
    {
        _ASSERTE (IsArray());
        return dac_cast<PTR_ArrayClass>(GetClass())->GetArrayElementType();
    }
获得数组的维度

    DWORD GetRank()
    {
        if (GetFlag(enum_flag_Category_IfArrayThenSzArray))
            return 1;//单维零基数组
        else
            return dac_cast<PTR_ArrayClass>(GetClass())->GetRank();
    }
下面四个函数都是利用TypeHandle的函数实现的

    TypeHandle GetApproxArrayElementTypeHandle();
    void SetApproxArrayElementTypeHandle(TypeHandle th);
    TypeHandle * GetApproxArrayElementTypeHandlePtr();
    static inline DWORD GetOffsetOfArrayElementTypeHandle();

#### 底层元数据
获取元数据对应的类型声明的RID或Token

    unsigned GetTypeDefRid()
    {
        g_IBCLogger.LogMethodTableAccess(this);
        return GetTypeDefRid_NoLogging();
    }
    unsigned GetTypeDefRid_NoLogging()
    {
        WORD token = m_wToken;
        //如果Token溢出，就取溢出后放置的值
        if (token == METHODTABLE_TOKEN_OVERFLOW)
            return (unsigned)*GetTokenOverflowPtr();
        //否则返回原有值
        return token;
    }
类型元数据获取与设置，非常简单

    inline mdTypeDef GetCl()
    {
        return TokenFromRid(GetTypeDefRid(), mdtTypeDef);
    }
    inline mdTypeDef GetCl_NoLogging()
    {
        return TokenFromRid(GetTypeDefRid_NoLogging(), mdtTypeDef);
    }
    void SetCl(mdTypeDef token)
    {
        unsigned rid = RidFromToken(token);
        if (rid >= METHODTABLE_TOKEN_OVERFLOW)
        {
            m_wToken = METHODTABLE_TOKEN_OVERFLOW;
            *GetTokenOverflowPtr() = rid;
        }
        else
        {
            _ASSERTE(FitsIn<U2>(rid));
            m_wToken = (WORD)rid;        
        }
        _ASSERTE(GetCl() == token);
    }
Token是否溢出了

    BOOL HasTokenOverflow()
    {
        return m_wToken == METHODTABLE_TOKEN_OVERFLOW;
    }
获取从内部导出元数据的接口(对COM)

    IMDInternalImport* GetMDImport()
    {
        return GetModule()->GetMDImport();
    }
    HRESULT GetCustomAttribute(WellKnownAttribute attribute,
                               const void  **ppData,
                               ULONG *pcbData)
    {
        return GetModule()->GetCustomAttribute(GetCl(), attribute, ppData, pcbData);
    }
获取闭包类型的Token，也就是从一个嵌套类出发，获取外部的类型。当EEClass不是一个嵌套类时，返回mdTypeDefNil

    mdTypeDef GetEnclosingCl()
    {
        mdTypeDef tdEnclosing = mdTypeDefNil;
        if (GetClass()->IsNested())
        {
            HRESULT hr = GetMDImport()->GetNestedClassProps(GetCl(), &tdEnclosing);
            if (FAILED(hr))
            {
                ThrowHR(hr, BFA_UNABLE_TO_GET_NESTED_PROPS);
            }
        }
        return tdEnclosing;
    }
#### 远程函数信息
这些都是通过Flag设置或者获取实现的，相对简单，这里不再展开。不过值得一提的是，我们肯定知道RCW表示Runtime Callable Wrapper。那么这里CCW实际上的意思是COM Callable Wrapper。前者是通过RCW使得托管代码可以与非托管代码交互，后者就是非托管代码通过CCW与托管代码交互。

    void SetHasGuidInfo();
    BOOL HasGuidInfo();
    void SetHasCCWTemplate();
    BOOL HasCCWTemplate();
    void SetHasRCWPerTypeData();
    BOOL HasRCWPerTypeData();

#### 泛型实例化字典
MethodTable中PerInstInfo指针指向对于每一个泛型实例化都有的指针表，这个表里面的指针指向了一个实例化字典。对于一个已经实例化的泛型，最后一个指针指向对应当前Method Table的字典，前面的项都指向继承链上父类的字典。实例化的泛型接口与结构体仅仅只有一个字典(自己的)，因为对于他们而言没有继承。

GetNumDicts()给出字典的个数。

将指针放在VTable之中而不是另外一个单独的表中。这样做的好处有:
1.时间上:省掉了不必要的间接查找，因为不需要先访问PerInstInfo来获取表。
2.空间上:不需要单独的PerInstInfo成员了，省掉了一个字的空间。
但问题是，很多代码都假定VTable全是整齐的MethodDescriptor Stub，而没有料到会有其余指针。

对当前Method Table的字典仅仅是一个储存泛型参数类型句柄的数组罢了，这些泛型参数需要满足:要么是已经实例化的泛型接口，要么是没有被共享的实例化类型。否则这个字典就会以这些参数开头，紧随其后的是固定数量的句柄Slot(类型与方法)，他们都是在运行时才被惰性填充的。最后这里有一个"溢出桶"指针，当字典被填满后，这个指针将被使用。总结起来这个字典的的形式是这样的。

    类型句柄1   第一个满足要求的泛型参数的类型句柄
    ...
    类型句柄n   第n个满足要求泛型参数的类型句柄
    Slot 1      第一个运行时句柄
    ...
    Slot n      第n个运行时句柄
    "溢出桶"指针
溢出桶指针仅包含运行时的句柄，另外的一个选择是，把多个bucket链起来。这样做好处是当字典增长时，不需要回收内存。但坏处是在运行时需要更多的间接查找。

字典的排布是被GetClass()->GetDictionaryLayout()所决定的，因此在不相容实例化之间，排布可能发生变化。这在泛型参数中有个别类型被共享或没有被共享时很有用。比如考虑一个具有两个参数的泛型类 Dictionary<K,V> ，在实例化 Dictionary<double,string> 上，任何对K(这里也就是double类型)的引用都是在JIT编译时可知的。但是任意包含V的Token必须拥有一个字典项。从另一方面来说，对于与 Dictionary<double,string> 共享的实例反过来也成立。

    DPTR(PerInstInfoElem_t) GetPerInstInfo()
    {
        _ASSERTE(HasPerInstInfo());
        return ReadPointer(this, &MethodTable::m_pPerInstInfo);
    }

    PTR_GenericsDictInfo GetGenericsDictInfo()
    {
        //泛型字典信息被储存在字典的负偏移处
        return dac_cast<PTR_GenericsDictInfo>(GetPerInstInfo()) - 1;
    }

    //如果此类型已经被实例化，返回字典的指针
    //如果没有，返回null
    PTR_Dictionary GetDictionary()
    {
        if (HasInstantiation())
        {
            //获取最后一个Slot指针
            TADDR base = dac_cast<TADDR>(&(GetPerInstInfo()[GetNumDicts()-1]));
            return PerInstInfoElem_t::GetValueMaybeNullAtPtr(base);
        }
        else
        {
            return NULL;
        }
    }
返回父类中用于翻译元数据的合适替代。
打个比方，当前类的定义是:
    
    Class<T> : Parent<List<T>,T[]>
那么对于此类型的 Parent<!0,!1> 将是，0 --> List<T>, 1 --> T[]，以这样的顺序被添加到替代的链上。
假如Parent的定义是:

    Parent<T,U> : Super<Dictionary<T,U>>
那么下一个对Super<!0>的下一个合适替代将是：0 --> Dictionaruy<List<T>,T[]>

    Substitution GetSubstitutionForParent(const Substitution *pSubst)
    {
        mdToken crExtends;
        DWORD   dwAttrClass;
        if (IsArray())
        {
            return Substitution(GetModule(), SigPointer(), pSubst);
        }
        IfFailThrow(GetMDImport()->GetTypeDefProps(
            GetCl(), 
            &dwAttrClass, 
            &crExtends));
        return Substitution(crExtends, GetModule(), pSubst);
    }

    inline DWORD GetAttrClass()
    {
        return GetClass()->GetAttrClass();
    }

    inline BOOL HasFieldsWhichMustBeInited()
    {
        return GetClass()->HasFieldsWhichMustBeInited();
    }

    inline BOOL IsPreRestored() const
    {
    #ifdef FEATURE_PREJIT
        return GetFlag(enum_flag_IsPreRestored);
    #else
        return FALSE;
    #endif
    }
#### 托管对象
m_ExposedClassObject是对应此类型的运行时类型实例。但是不要对数组或者远程对象使用。所有对象数组都共享同一份MethodTable/EEClass。对泛型来讲，此数据为一个实例拥有一个。

    OBJECTREF GetManagedClassObject()
    {
        if (GetWriteableData()->m_hExposedClassObject == NULL)
        {
            //确保已经从本机映像中恢复了
            CheckRestore();
            REFLECTCLASSBASEREF  refClass = NULL;
            GCPROTECT_BEGIN(refClass);
            refClass = (REFLECTCLASSBASEREF) AllocateObject(g_pRuntimeTypeClass);
            LoaderAllocator *pLoaderAllocator = GetLoaderAllocator();
            ((ReflectClassBaseObject*)OBJECTREFToObject(refClass))->SetType(TypeHandle(this));
            ((ReflectClassBaseObject*)OBJECTREFToObject(refClass))->SetKeepAlive(pLoaderAllocator->GetExposedObject());
            LOADERHANDLE exposedClassObjectHandle = pLoaderAllocator->AllocateHandle(refClass);
            //让所有的线程在此进行竞争，只有获胜者能将m_ExposedClassObject赋值
            if (FastInterlockCompareExchangePointer(&(EnsureWritablePages(GetWriteableDataForWrite())->m_hExposedClassObject), exposedClassObjectHandle, static_cast<LOADERHANDLE>(NULL)))
            {
                pLoaderAllocator->FreeHandle(exposedClassObjectHandle);
            }
            GCPROTECT_END();
        }
        RETURN(GetManagedClassObjectIfExists());
    }
    OBJECTREF GetManagedClassObjectIfExists()
    {
        LOADERHANDLE handle = GetWriteableData_NoLogging()->GetExposedClassObjectHandle();
        //GET_LOADERHANDLE_VALUE_FAST宏在此处被内联好让我们在返回值为非null时给予编译器提示
        if (!LoaderAllocator::GetHandleValueFast(handle, &retVal) &&
        !GetLoaderAllocator()->GetHandleValueFastPhase2(handle, &retVal))
        {
            return NULL;
        }
        COMPILER_ASSUME(retVal != NULL);
        return retVal;
    }
#### GUID信息
在进行COM交互时获取GUID(IID与CLSID)使用

    PTR_GuidInfo GetGuidInfo();
    void SetGuidInfo(GuidInfo* pGuidInfo);
获取并且缓存此接口/类型的GUID

    HRESULT GetGuidNoThrow(GUID *pGuid, BOOL bGenerateIfNotFound, BOOL bClassic = TRUE);
    void    GetGuid(GUID *pGuid, BOOL bGenerateIfNotFound, BOOL bClassic = TRUE);

#ifdef FEATURE_COMINTEROP
获取用于WinRT交互的GUID，对于投影的泛型接口返回等效的WinRT类型的GUID(例如List<T> -> IVector<T>)

    BOOL    GetGuidForWinRT(GUID *pGuid)
    {
        BOOL bRes = FALSE;
        if ((IsProjectedFromWinRT() && !HasInstantiation()) || 
            (SupportsGenericInterop(TypeHandle::Interop_NativeToManaged) && IsLegalNonArrayWinRTType()))
        {
            bRes = SUCCEEDED(GetGuidNoThrow(pGuid, TRUE, FALSE));
        }
        return bRes;
    }

构造对每一个类型的RCW数据

    RCWPerTypeData *CreateRCWPerTypeData(bool bThrowOnOOM)
    {
        AllocMemTracker amTracker;
        RCWPerTypeData *pData;
        if (bThrowOnOOM)
        {
            //在低频堆分配后进行追踪
            TaggedMemAllocPtr ptr = GetLoaderAllocator()->GetLowFrequencyHeap()->AllocMem(S_SIZE_T(sizeof(RCWPerTypeData)));
            pData = (RCWPerTypeData *)amTracker.Track(ptr);
        }
        else
        {
            TaggedMemAllocPtr ptr = GetLoaderAllocator()->GetLowFrequencyHeap()->AllocMem_NoThrow(S_SIZE_T(sizeof(RCWPerTypeData)));
            pData = (RCWPerTypeData *)amTracker.Track_NoThrow(ptr);
            if (pData == NULL)
            {
                return NULL;
            }
        }
        //分配的内存都被0初始化了，所有的都还没有被计算
        _ASSERTE(pData->m_dwFlags == 0);
        RCWPerTypeData **pDataPtr = GetRCWPerTypeDataPtr();
        if (bThrowOnOOM)
        {
            EnsureWritablePages(pDataPtr);
        }
        else
        {
            if (!EnsureWritablePagesNoThrow(pDataPtr, sizeof(*pDataPtr)))
            {
                return NULL;
            }
        }
        if (InterlockedCompareExchangeT(pDataPtr, pData, NULL) == NULL)
        {
            amTracker.SuppressRelease();
        }
        else
        {
            pData = *pDataPtr;
        }
        return pData;
    }
获取与此类型关联的RCW数据，如果此类型不需要这样的数据或者分配失败了，此函数返回null(当且仅当ThrowOnOOM为false)

    RCWPerTypeData *GetRCWPerTypeData(bool bThrowOnOOM = true)
    {
        if (!HasRCWPerTypeData())
        return NULL;
        RCWPerTypeData *pData = *GetRCWPerTypeDataPtr();
        if (pData == NULL)
        {
            pData = CreateRCWPerTypeData(bThrowOnOOM);
        } 
        return pData;
    }

#### 嵌套类-MethodData
MethodData是用于储存方法具体数据的类型，其实为数据 + 纯虚函数，供继承者实现。这里我们只列出成员以及函数签名
    
    用于储存方法被引用的次数
    ULONG m_cRef;
    方法实现所在的表
    MethodTable *const m_pImplMT;
    方法定义所在的表
    MethodTable *const m_pDeclMT;
    用于表示Slot无效的数目
    static const UINT32 INVALID_SLOT_NUMBER = UINT32_MAX;
纯虚方法签名一览

        virtual MethodData  *GetDeclMethodData() = 0;
        virtual MethodDesc  *GetDeclMethodDesc(UINT32 slotNumber) = 0; 
        virtual MethodData  *GetImplMethodData() = 0;
        virtual DispatchSlot GetImplSlot(UINT32 slotNumber) = 0;
        virtual UINT32       GetImplSlotNumber(UINT32 slotNumber) = 0;
        virtual MethodDesc  *GetImplMethodDesc(UINT32 slotNumber) = 0;
        virtual void InvalidateCachedVirtualSlot(UINT32 slotNumber) = 0;
        virtual UINT32 GetNumVirtuals() = 0;
        virtual UINT32 GetNumMethods() = 0;
非虚方法。
对MethodData对象进行引用计数

        inline ULONG AddRef()
        { 
            return (ULONG) InterlockedIncrement((LONG*)&m_cRef); 
        }
        ULONG Release()
        {
            //必须调整此函数使用其他备选的分配器使得在Debug线程上不会造成潜在的死锁
            SUPPRESS_ALLOCATION_ASSERTS_IN_THIS_SCOPE;
            ULONG cRef = (ULONG) InterlockedDecrement((LONG*)&m_cRef);
            if (cRef == 0) {
                delete this;
            }
            return (cRef);
        }

        static void HolderAcquire(MethodData *pEntry)
        { 
            return; 
        }
        static void HolderRelease(MethodData *pEntry)
        {
            if (pEntry != NULL) 
                pEntry->Release(); 
        }

        MethodTable *GetDeclMethodTable() { return m_pDeclMT; }
        MethodTable *GetImplMethodTable() { return m_pImplMT; }

#### 嵌套类-MethodData->MethodDataEntry
此类型在构建MethodData时被使用
字段:

    无效Chain与Index数
    static const UINT32 INVALID_CHAIN_AND_INDEX = (UINT32)(-1);
    无效Slot数
    static const UINT16 INVALID_IMPL_SLOT_NUM = (UINT16)(-1);
    此字段同时包含链偏移和表索引。把他们放到一起的原因是:
    我们需要同时对这两个变量进行原子更新，因此如果他们都UINT16大小，就可以凑成一个UINT32
    UINT32           m_chainDeltaAndTableIndex;
    此字段针对被虚拟化(virtual)重映射的方法Slot
    UINT16           m_implSlotNum;
    在调度实现表中的入口
    DispatchSlot     m_slot;   
    此Slot的MethodDescriptor         
    MethodDesc      *m_pMD;
因为其余的方法都是对应的字段访问包装，这里不再列出，唯一一个值得关注的是:

    static void ProcessMap(
            const DispatchMapTypeID * rgTypeIDs, 
            UINT32                    cTypeIDs, 
            MethodTable *             pMT, 
            UINT32                    cCurrentChainDepth, 
            MethodDataEntry *         rgWorkingData)
    {
        for (DispatchMap::EncodedMapIterator it(pMT); it.IsValid(); it.Next())
        {
            for (UINT32 nTypeIDIndex = 0; nTypeIDIndex < cTypeIDs; nTypeIDIndex++)
            {
                if (it.Entry()->GetTypeID() == rgTypeIDs[nTypeIDIndex])
                {
                    UINT32 curSlot = it.Entry()->GetSlotNumber();
                    if ((curSlot < pMT->GetNumVirtuals()) || (iCurrentChainDepth == 0))
                    {
                        MethodDataEntry * pCurEntry = &rgWorkingData[curSlot];
                        if (!pCurEntry->IsDeclInit() && !pCurEntry->IsImplInit())
                        {
                            pCurEntry->SetImplData(it.Entry()->GetTargetSlotNumber());
                        }
                    }
                }
            }
        }
    }
#### 嵌套类-MethodDataObject
此类型紧接其后，继承自MethodData并实现了所有签名的虚方法。其字段为:

    //此字段被用于阶段性图解码，其表示了下一个我们需要解码的类型
    UINT32       m_iNextChainDepth;
    static const UINT32 MAX_CHAIN_DEPTH = UINT32_MAX;    
    BOOL m_containsMethodImpl;
下面是虚函数的实现:

    virtual MethodData  *GetDeclMethodData()
    { 
        return this;
    }
    virtual MethodData  *GetImplMethodData()
    { 
        return this; 
    }

    virtual MethodDesc *GetDeclMethodDesc(UINT32 slotNumber)
    {
        //获取对应Slot的入口点
        MethodDataObjectEntry * pEntry = GetEntry(slotNumber);
        //一次填充继承链上的一级入口，当我们遇到需要的Method Descriptpr被填充时，就停下来
        while (!pEntry->GetDeclMethodDesc() && PopulateNextLevel());
        //尝试获取Method Descriptor
        MethodDesc * pMDRet = pEntry->GetDeclMethodDesc();
        if (pMDRet == NULL)
        {
            //如果是空，就尝试从方法实现Method Descriptor获取声明
            pMDRet = GetImplMethodDesc(slotNumber)->GetDeclMethodDesc(slotNumber);
            _ASSERTE(CheckPointer(pMDRet));
            //设置对应入口的声明Method Descriptor
            pEntry->SetDeclMethodDesc(pMDRet);
        }
        else
        {
            //否则校验是否与已有数据一致
            _ASSERTE(pMDRet == GetImplMethodDesc(slotNumber)->GetDeclMethodDesc(slotNumber));
        }
        return pMDRet;
    }

    virtual DispatchSlot GetImplSlot(UINT32 slotNumber)
    {
        //断言Slot数
        _ASSERTE(slotNumber < GetNumMethods());
        //直接构造调度Slot
        return DispatchSlot(m_pDeclMT->GetRestoredSlot(slotNumber));
    }
此函数获取实现方法的Slot数，因为实现表还未被引入，我们使用的仍然是传统VTable，因此不存在额外映射关系。

    UINT32 MethodTable::MethodDataObject::GetImplSlotNumber(UINT32 slotNumber)
    { 
        _ASSERTE(slotNumber < GetNumMethods());
        return slotNumber;
    }
获取Slot对应的方法实现Method Descriptor

    virtual MethodDesc  *GetImplMethodDesc(UINT32 slotNumber)
    {
        _ASSERTE(slotNumber < GetNumMethods());
        MethodDataObjectEntry *pEntry = GetEntry(slotNumber);
        while (!pEntry->GetImplMethodDesc() && PopulateNextLevel());
        //获取实现MethodDescriptor
        MethodDesc *pMDRet = pEntry->GetImplMethodDesc();
        if (pMDRet == NULL)
        {
            //如果没有，就表示此Slot对应的必为虚方法
            _ASSERTE(slotNumber < GetNumVirtuals());
            //尝试在声明中查找此方法的Method Descriptor
            pMDRet = m_pDeclMT->GetMethodDescForSlot(slotNumber);
            _ASSERTE(CheckPointer(pMDRet));
            //将实现方法Method Descriptor设置为声明方法的Method Descriptor
            pEntry->SetImplMethodDesc(pMDRet);
        }
        else
        {
            //否则的话，要么此方法非虚，要么此方法实现Method Descriptor被设置为声明方法Method Descriptor
            _ASSERTE(slotNumber >= GetNumVirtuals() || pMDRet == m_pDeclMT->GetMethodDescForSlot(slotNumber));
        }
        return pMDRet;
    }
此函数的潜在逻辑就是，如果没有找到实现Method Descriptor，那么一定是虚方法，然后将虚方法的Entry的实现与声明设置为一样的。找到的，要么是非虚方法，要么是已经被设置的虚方法Entry。

无效化缓存的Virtual Slot

    virtual void InvalidateCachedVirtualSlot(UINT32 slotNumber)
    {
        _ASSERTE(slotNumber < GetNumVirtuals());
        MethodDataObjectEntry *pEntry = GetEntry(slotNumber);
        pEntry->SetImplMethodDesc(NULL);
    }

初始化当前MethodDataObject

    void Init(MethodData *pParentData)
    {
        m_iNextChainDepth = 0;
        m_containsMethodImpl = FALSE;
        ZeroMemory(GetEntryData(), sizeof(MethodDataObjectEntry) * GetNumMethods());
    }
激活下一个级别的MethodDataObject

    BOOL PopulateNextLevel()
    {
        UINT32 iChainDepth = GetNextChainDepth();
        //如果链深度是此值，那么我们已经处理了所有的父类
        if (iChainDepth == MAX_CHAIN_DEPTH) {
            return FALSE;
        }
        //现在随着链移动到目标上去
        MethodTable *pMTCur = m_pDeclMT;
        for (UINT32 i = 0; pMTCur != NULL && i < iChainDepth; i++) {
            pMTCur = pMTCur->GetParentMethodTable();
        }
        //如果已经到了尽头，我们就结束了
        if (pMTCur == NULL) {
            //设置标记
            SetNextChainDepth(MAX_CHAIN_DEPTH);
            return FALSE;
        }
        //为祖先节点设置入口数据
        FillEntryDataForAncestor(pMTCur);
        //链深度加一
        SetNextChainDepth(iChainDepth + 1);
        return TRUE;
    }

这里面的MethodDataObjectEntry长相简单，只是一个键值对的形式，其余访问函数被忽略掉了。

    struct MethodDataObjectEntry
    {
        MethodDesc *m_pMDDecl;
        MethodDesc *m_pMDImpl;
    }
获取Entry数据入口，这里可以看得出，此函数结尾就是一个数组，因此不能被继承。

    inline MethodDataObjectEntry *GetEntryData()
    { 
        return (MethodDataObjectEntry *)(this + 1); 
    }

    inline MethodDataObjectEntry *GetEntry(UINT32 i)
    {
        CONSISTENCY_CHECK(i < GetNumMethods()); 
        return GetEntryData() + i;
    }
接下来就是相对重要的函数，透过此函数的实现与注释我们可以了解到整个MethodDataObject的机制。

因为我们从继承链的最底端遍历到最顶端，第一个我们我们遇到的Slot对应的方法通常是一个声明和实现兼有的Method Descriptor。
然而如果这个Slot是一个MethodImpl的目标，那么pMD也就是不必要的了。我们在继承链上寻找到虚方法的实现时保守地避免填充而不是在每Slot一个的基础上追踪。
需要注意的是，可能在继承链更高的位置上有我们还没看见的方法实现，并且我们将会在到达那个级别之后才会填充虚方法。这样子做是安全的，因为我们填充过的Slot已经被子类引入或者重写了，并且这样会比任意继承的方法实现的优先级高。
在我们填充入口数据之前，先判断当前的祖先是否有任何的方法实现。

    void FillEntryDataForAncestor(MethodTable *pMT)
    {
        if (pMT->GetClass()->ContainsMethodImpls())
            m_containsMethodImpl = TRUE;
        if (m_containsMethodImpl && pMT != m_pDeclMT)
            return;
        MethodTable::IntroducedMethodIterator it(pMT, FALSE);
        for (; it.IsValid(); it.Next())
        {
            MethodDesc * pMD = it.GetMethodDesc();
            g_IBCLogger.LogMethodDescAccess(pMD);
            unsigned slot = pMD->GetSlot();
            if (slot == MethodTable::NO_SLOT)
                continue;
            //我们想要填充所有被我们想要收集数据的类型所引入的方法，还有那些继承链以上的虚方法
            if (pMT == m_pDeclMT)
            {
                if (m_containsMethodImpl && slot < nVirtuals)
                    continue;
            }
            else
            {
                if (slot >= nVirtuals)
                    continue;
            }
            MethodDataObjectEntry * pEntry = GetEntry(slot);
            if (pEntry->GetDeclMethodDesc() == NULL)
            {
                pEntry->SetDeclMethodDesc(pMD);
            }
            if (pEntry->GetImplMethodDesc() == NULL)
            {
                pEntry->SetImplMethodDesc(pMD);
            }
        }
    }
#### 嵌套类-MethodDataInterface
此类型与MethodDataObject相似，只不过是针对接口的，其GetImplMethodDesc转发到GetDeclMethodDesc，而也对应地没有Virtual Slot缓存。
#### 嵌套类-MethodDataInterfaceImpl
此类型对应接口的实现数据，不过仍然有一些细节需要被提出来。在处理Impl数据获取时，相关API:

        virtual DispatchSlot GetImplSlot(UINT32 slotNumber);
        virtual UINT32       GetImplSlotNumber(UINT32 slotNumber);
        virtual MethodDesc  *GetImplMethodDesc(UINT32 slotNumber);
都使用了另外一个函数:MapToImplSlotNumber

    UINT32 implSlotNumber = MapToImplSlotNumber(slotNumber);
说明在做接口的实现获取时，需要先从原有的Slot映射到当前VTable可用的索引上。

    UINT32 MapToImplSlotNumber(UINT32 slotNumber)
    {
        _ASSERTE(slotNumber < GetNumMethods());
        //获取Entry
        MethodDataEntry *pEntry = GetEntry(slotNumber);
        //惰性激活MethodDescriptor
        while (!pEntry->IsImplInit() && PopulateNextLevel()) {}
        if (pEntry->IsImplInit()) {
            //从Entry获取实现方法的索引
            return pEntry->GetImplSlotNum();
        }
        else {
            return INVALID_SLOT_NUMBER;
        }
    }
这里也就解释了之前Entry中的Slot成员被用于虚拟化重映射，指的就是类在实现了接口方法后又将其标记为virtual使得子类可以重写此方法所需要的重映射。
#### MehtodData相关字段及API
    static MethodDataCache *s_pMethodDataCache;
    static BOOL             s_fUseParentMethodData;
    static BOOL             s_fUseMethodDataCache;
允许方法数据缓存

    static void AllowMethodDataCaching()
    {
        CheckInitMethodDataCache(); 
        s_fUseMethodDataCache = TRUE;
    }
清除方法数据缓存

    static void ClearMethodDataCache()
    {
        if (s_pMethodDataCache != NULL) {
            s_pMethodDataCache->Clear();
        }
    }
允许拷贝父类方法数据

    static void AllowParentMethodDataCopy()
    {
        s_fUseParentMethodData = TRUE; 
    }
fCanCache参数决定了返回的方法数据能否被添加进全局的缓存中。此标志在请求一个正在被构建的类型的方法数据时被使用。

    static MethodData *GetMethodData(MethodTable *pMT, BOOL fCanCache = TRUE)
    {
        return GetMethodData(pMT, pMT, fCanCache);
    }

    static MethodData *GetMethodData(MethodTable *pMTDecl, MethodTable *pMTImpl, BOOL fCanCache = TRUE)
    {
        MethodDataWrapper hData(GetMethodDataHelper(pMTDecl, pMTImpl, fCanCache));
        hData.SuppressRelease();
        return hData;
    }
此函数被BuildMethodTable类所使用，因为具体的接口还没有被加载进来。此函数也不会储存缓存。

    static MethodData * GetMethodData(
        const DispatchMapTypeID * rgDeclTypeIDs, 
        UINT32                    cDeclTypeIDs, 
        MethodTable *             pMTDecl, 
        MethodTable *             pMTImpl)
    {
        MethodDataWrapper hData(GetMethodDataHelper(rgDeclTypeIDs, cDeclTypeIDs, pMTDecl, pMTImpl));
        hData.SuppressRelease();
        return hData;
    }
拷贝Slot

    void CopySlotFrom(UINT32 slotNumber, MethodDataWrapper &hSourceMTData, MethodTable *pSourceMT)
    {
        //获取实现Method Descriptor
        MethodDesc *pMD = hSourceMTData->GetImplMethodDesc(slotNumber);
        _ASSERTE(CheckPointer(pMD));
        _ASSERTE(pMD == pSourceMT->GetMethodDescForSlot(slotNumber));
        SetSlot(slotNumber, pMD->GetInitialEntryPointForCopiedSlot());
    }
检查并初始化方法数据缓存

    static void CheckInitMethodDataCache()
    {
        if (s_pMethodDataCache == NULL)
        {
            UINT32 cb = MethodDataCache::GetObjectSize(8);
            NewArrayHolder<BYTE> hb(new BYTE[cb]);
            MethodDataCache *pCache = new (hb.GetValue()) MethodDataCache(8);
            if (InterlockedCompareExchangeT(
                    &s_pMethodDataCache, pCache, NULL) == NULL)
            {
                hb.SuppressRelease();
            }
            else
            {
                //如果其他的线程成功了，那么就直接返回并且让Holder负责清理工作
                return;
            }
        }
    }

    static MethodData *FindParentMethodDataHelper(MethodTable *pMT)
    {
        MethodData *pData = NULL;
        if (s_fUseMethodDataCache && s_fUseParentMethodData) {
            if (!pMT->IsInterface()) {
                //对于非共享的代码，此操作是不正确的(待修复)
                MethodTable *pMTParent = pMT->GetParentMethodTable();
                if (pMTParent != NULL) {
                    pData = FindMethodDataHelper(pMTParent, pMTParent);
                }
            }
        }
        return pData;
    }

    static MethodData *FindMethodDataHelper(MethodTable *pMTDecl, MethodTable *pMTImpl)
    {
        return s_pMethodDataCache->Find(pMTDecl, pMTImpl);
    }

    static MethodData *GetMethodDataHelper(MethodTable *pMTDecl, MethodTable *pMTImpl, BOOL fCanCache)
    {
        SUPPRESS_ALLOCATION_ASSERTS_IN_THIS_SCOPE;
        //先在缓存中寻找
        if (s_fUseMethodDataCache) {
            MethodData *pData = FindMethodDataHelper(pMTDecl, pMTImpl);
            if (pData != NULL) {
                return pData;
            }
        }
        //如果我们到了这，意味着在缓存中没有任何入口
        MethodData *pData = NULL;
        if (pMTDecl == pMTImpl)
        {
            if (pMTDecl->IsInterface())}
            {
                //接口直接分配方法数据
                pData = new MethodDataInterface(pMTDecl);
            }
            else
            {
                //否则分配Array并在第一个位置构造MethodDataObject
                UINT32 cb = MethodDataObject::GetObjectSize(pMTDecl);
                NewArrayHolder<BYTE> pb(new BYTE[cb]);
                MethodDataHolder h(FindParentMethodDataHelper(pMTDecl));
                pData = new (pb.GetValue()) MethodDataObject(pMTDecl, h.GetValue());
                pb.SuppressRelease();
            }
        }
        else
        {
            pData = GetMethodDataHelper(
                NULL, 
                0, 
                pMTDecl, 
                pMTImpl);
        }
        if (fCanCache && s_fUseMethodDataCache) {
            s_pMethodDataCache->Insert(pData);
        }
        return pData;
    }

    static MethodData * GetMethodDataHelper(
        const DispatchMapTypeID * rgDeclTypeIDs, 
        UINT32                    cDeclTypeIDs, 
        MethodTable *             pMTDecl, 
        MethodTable *             pMTImpl)
    {
        SUPPRESS_ALLOCATION_ASSERTS_IN_THIS_SCOPE;
        CONSISTENCY_CHECK(pMTDecl->IsInterface() && !pMTImpl->IsInterface());
        //因为这是在BuildMethodTable中使用的自定义方法，所以不能被缓存 
        MethodDataWrapper hDecl(GetMethodData(pMTDecl, FALSE));
        MethodDataWrapper hImpl(GetMethodData(pMTImpl, FALSE));
        UINT32 cb = MethodDataInterfaceImpl::GetObjectSize(pMTDecl);
        NewArrayHolder<BYTE> pb(new BYTE[cb]);
        MethodDataInterfaceImpl * pData = new (pb.GetValue()) MethodDataInterfaceImpl(rgDeclTypeIDs, cDeclTypeIDs, hDecl, hImpl);
        pb.SuppressRelease();
        return pData;
    }
#### 嵌套类-MethodIterator
此类用于遍历所有方法，较为简单
#### 嵌套类-IntroducedMethodIterator
此类用于遍历当前类型引入的方法，新的静态方法，非虚方法，重写的父类方法。

### 结语
虽然对MethodTable的探索到这里就正式结束了，考虑到整个Method Table承担的责任很多，要点细节也很多，因此决定再追加一篇精要的总结来捋清楚Method Table内部外部的联系。实际上我们仍然有一些残余没有解析完成，例如VSD具体实现，Method Descriptor等，将会放在紧随其后的章节中进行剖析。