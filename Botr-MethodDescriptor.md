# MethodDescriptor
# 介绍
MethodDescriptor(以下简称MD)是托管方法的内部表示，其有以下几个功能

* 提供唯一的方法句柄，其在整个runtime期间都可以使用。对于普通方法，MD是一个三元组<模块,元数据Token,实例>的句柄。
* 缓存那些经常需要访问的，但是其从元数据计算是相当昂贵的信息，比如一个方法是否为静态
* 捕获方法的运行时状态，比如方法的本机代码是否已经被生成了
* 持有方法的入口点

## 设计目标与非目标
### 目标
性能：MD的设计在大小上被特别优化，因为一个方法可能拥有一个或多个MD。比如在当下的设计，一个普通的非泛型方法的MD只有8字节。
### 非目标
丰富性：MD并没有缓存方法的所有信息，我们期待的是这些底层元数据访问频率必须低，比如方法签名

## MD的设计
### MD的种类
**IL**
用于普通IL方法

**Instantiated**
用于那些有泛型实例化的或者没有在MethodTable中提前分配槽位的方法

**FCall**
使用非托管代码实现的内部方法，这些方法拥有MethodImplAttribute(MethodImplOptions.InternalCall)标记，比如委托的构造方法与tlbimp的构造方法
注：tlbimp是Type Library Importer的缩写，其将COM的类型库中的定义转换为等效的托管代码定义。

**NDirect**
P/Invoke方法。这些方法有DllImport标记

**EEImpl**
Runtime提供实现的委托方法(Invoke,BeginInvoke,EndInvoke)

**Array**
Runtime提供实现的Array方法(Get,Set,Address)

**ComInterop**
COM接口方法。因为非泛型的接口默认能被用于COM交互，这种类型的MD被用于所有的接口方法

**Dynamic**
没有使用底层元数据动态构建的方法。其被Stub-as-IL或者LKG(Light-weight code generation，轻量级代码生成)产生。

## 可选实现
虚方法与继承可能是一个在C++中实现各种不同MD的自然方法。虚方法会增加VTable指针到每一个MD对象里，浪费掉许多宝贵的空间。在x86下，一个指针会浪费掉4个字节。相对于虚拟化取而代之的是使用基于MD种类的切换，只需要占用3个比特位。

    DWORD MethodDesc::GetAttrs()
    {
        if (IsArray())
            return ((ArrayMethodDesc*)this)->GetAttrs();
        if (IsDynamic())
            return ((DynamicMethodDesc*)this)->GetAttrs();
        return GetMDImport()->GetMethodDefProps(GetMemberDef());
    }

## 方法槽
每一个MD都有一个槽，包含有方法的入口点。对所有的方法，槽位和入口点都必须存在，即使是像那种永远不会被调用的抽象方法。在runtime中，有许多依赖入口点与MD一对一映射的地方，让此关系成为一个不变量。

方法槽要么在MT中，要么在MD中。槽的位置被flag：mdcHasNonVtableSlot 所决定。

有一些需要通过槽位索引快速查询的方法的槽位被存储在MT中，比如虚方法或者泛型类型的方法。在这种情况下，MD包含槽位索引来允许高效地查找入口点。

否则的话，槽位就是MD本身的一部分。这种安排提升了数据的本地性并且节省了工作集。同时，我们不总可能为动态创建的MD在MT中提前分配槽位，比如被EnC添加的方法，泛型方法的实例化或者动态方法。

## MD Chunks
多个MD在Chunk中被分配来节省空间。多个MD倾向于有相同的MT与元数据Token的高位。MDChunk就是通过提取共同的信息到拥有多个MD的数组前面形成的。MD只包含自己在数组中的索引。

## Precode
Precode是用于实现临时的入口点或者为stub实现一个高效的包装器的小段代码。Precode在以下两种情形中是一个小型代码生成器，其尽可能生成最高效的代码。在理想的情况下，所有Runtime动态生成的本机代码都是由JIT产出的。在这种情况下其不会满足，给定这两个情景的特定要求。在x86上基础的Precode可能长成这样：

    mov eax,pMethodDesc //将MD载入寄存器
    jmp target          //跳转到目标

**高效Stub包装器:** 特定方法的实现，比如P/Invoke，委托调用，多维数组的索引器，是由Runtime提供的，就好像手写的Assembly Stub一样。Precode提供了一种空间高效的包装器，对多个调用者多次复制。

Stub的代码被Precode代码段所包裹，其可以映射到MD上然后跳转到Stub的代码。这样的话，Stub的代码就可以被多个方法所共享。这是一个用于实现P/Invoke封送Stub的重要优化。其也在入口点与MD之间创建了一对一映射，形成了一个高效的低级系统。

**临时入口点:** 方法在没有被JIT之前必须提供入口点，这样已经JIT的代码就可以有一个调用的地址。这些临时的入口点就是由Precode提供的。他们是Stub包装器的特殊形式。

这个技巧就是JIT的延迟，为空间与时间提供了性能优化。否则的话，整个方法的传递闭包都需要在方法在被执行前被JIT。这将是一个浪费，因为只有被选择的代码分支依赖需要被JIT

每一个临时入口点都比一般的方法体小得多。他们需要变得很小因为有很多这样的对象，即使需要一些性能的损失。临时入口点在方法的真正代码被生成前仅触发一次。

临时入口点的目标是一个PreStub，其是一种触发方法被JIT的特殊Stub。其将使用原子操作把临时入口点替换为稳定入口点。稳定入口点将在整个方法生命周期中保持不变。此不变量需要保证线程安全，因为方法槽总是不上锁访问的。

**稳定入口点:** 其要么是本地代码，要么是Precode。本地代码要么是JIT的代码，要么是在NGen映像中的代码。

临时入口点永远不会被保存进NGen映像中。所有在NGen映像中的入口点都是不再改变的稳定入口点。这是一个减少私有工作集的重要优化。

## 单次Callable vs. 多次Callable
入口点在调用方法时被需要。MD暴露一些方法，这些方法可以对指定的情况下找到最高效的入口点。主要的不同之处在于入口点是否会被用于一次调用方法，或者其是否会被多次使用来调用方法。

举个例子，通过临时入口点调用方法多次可能是个相当坏的主意，因为其每次都会经过PreStub。在另一方面，使用临时入口点来调用一次方法会好很多。

这些方法是
* MethodDesc::GetSingleCallableAddrOfCode
* MethodDesc::GetMultiCallableAddrOfCode
* MethodDesc::GetSingleCallableAddrOfVirtualizedCode
* MethodDesc::GetMultiCallableAddrOfVirtualizedCode

## Precode的种类
有许多特殊的Precode类型

Precode的类型必须可以从指令序列中被简单地计算出来。在x86与x64下，Precode的类型是用过在固定偏移处取得一个byte来断定。当然，这也对实现不同种类的Precode指令序列施加了一些限制。

**StubPrecode**

StubPrecode是基础的Precode类型。其将MD载入寄存器中，然后跳转。其实现必须让Precode工作。当其他特殊的Precode类型不可用时，它将作为一个备选。

其他类型的Precode都是在平台特别的文件中的可选优化。

StubPrecode在x86上看起来像这样

    mov eax,pMethodDesc
    mov ebp,ebp //用于标识Precode类型的多余指令
    jmp target

"target"最初是指向Prestub的。其被打补丁指向最终的目标。最终的目标(Stub或者本地代码)可能使用或者不使用eax。Stub经常使用，本地代码不使用。

**FixupPrecode**

FixupPrecode在最终目标不需要在寄存器中的MD时被使用。FixupPrecode通过避免将MD装载进寄存器中而节省了几个周期。

其最常见的用处就是在NGen映像中的方法修复。

FixupPrecode在x86上的最初状态:

    call PrecoeFixupThunk //此调用从不返回。其将返回地址出栈，并且利用这个地址来获取下面的pMethodDesc来寻找方法中被需要且需要被JIT的东西。
    pop esi //用于标识Precode类型的多余指令
    dword pMethodDesc

在寄存器中传递MD有时候被称为**MD Calling Convention(MD调用约定)**

**FixupPrecode Chunks**

FixupPrecode chunk是多个FixupPrecode的空间高效表示。其借鉴了MD Chunk的思想，通过提取相同的MD指针到一个公共区域。

FixupPrecode chunk节省空间并且提升了Precode的代码密度。代码密度在x64大型服务器上有1%-2%的提升。

FixupPrecode chunk在x86上看起来像这样

    jmp Target2
    pop edi //用于标识Precode类型的多余指令
    db MethodDescChunkIndex
    db 2 (PrecodeChunkIndex)

    jmp Target1
    pop edi
    db MethodDescChunkIndex
    db 1 (PrecodeChunkIndex)

    jmp Target0
    pop edi
    db MethodDescChunkIndex
    db 0 (PrecodeChunkIndex)

    dw pMethodDescBase

一个FixupPrecode chunk对应一个MDChunk。但这里没有一一映射。每一个FixupPrecode都有一个其属于的方法的索引。其允许在Chunk中为需要的方法分配FixupPrecode

**压缩入口点**
压缩入口点是多个临时入口点的空间高效实现。

使用StubPrecode或者FixupPrecode实现的临时入口点能够被打补丁指向真正的代码。JIT过的代码可以直接调用临时入口点。在这种情况下，临时入口点必须可以被多次调用。

压缩入口点不能被替换为真正的代码。JIT过的代码不能直接调用他们。他们使用速度来换空间。对这些入口点的调用是通过 FuncPtrStubs 表中的槽位间接实现的，这些槽位最终将指向真正的入口点。对一个多次可调用的入口点进行请求会导致一个StubPrecode或者FixupPrecode被分配出来。

原始的速度差距在于，压缩入口点的一次间接寻址的消耗与在给定平台上的一次直接调用和一次跳转的消耗。后者在大型服务器场景下曾经比前者快上几个百分点，因为其可以被硬件更好地预测(2005年)。在当下的硬件可能就不总是这样了(2015年)

压缩入口点从开始就一直只是在x86平台上实现的。他们的附加复杂性，空间与速度的交换和硬件的先进性使得他们在其他平台上相较于x86不公平。

在x86下的压缩入口点看起来像这样：

    entrypoint0:
    mov al,0
    jmp short Dispatch

    entrypoint1:
    mov al,1
    jmp short Dispatch

    entrypoint2:
    mov al,2
    jmp short Dispatch

    Dispatch:
    movzx eax,al
    shl eax, 3
    add eax, pBaseMD
    jmp PreStub

临时入口点的分配总是尝试在可能的方案中选择体积小的。比如，在x86上一个单压缩入口点是比一个StubPrecode更大的。这种情况下，StubPrecode将比压缩入口点更受青睐。对稳定入口点的Precode分配会尝试重用一个类型匹配的临时入口点。

**ThisPtrRetBufPrecode**
ThisPtrRetBufPrecode被用于为返回ValueType的开放实例委托切换返回值Buffer和this指针。其用于转换MyValueTaype Bar(Foo)的调用约定到MyValueType Foo::Bar()

注：开放委托是指将第一个参数object作为this参加真正的方法调用。也就是对实例方法不指定Target，在调用时再指定。

此类型的Precode总是按需分配，作为真正方法的包装器，并且被存储在 FuncPtrStubs 表里

ThisPtrRetBufPrecode看起来像这样：

    mov eax,ecx
    mov ecx,edx
    mov edx,eax
    nop
    jmp entrypoint
    dw pMethodDesc

**NDirectImportPrecode**
NDirectImportPrecode用于非托管P/Invoke目标的延迟绑定。其出现是为了方便，并且减少了一定量的特定平台探测

每一个NDirectMethodDesc在常规Precode之外还有NDirectImportPrecode。

NDirectImportPrecode 在x86上看起来像这样：

    mov eax,pMethodDesc
    mov eax,eax 
    jmp NDirectImportThunk
 
