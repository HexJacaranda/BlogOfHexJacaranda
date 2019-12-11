# Method Table细节一览
## 前导概念
Canonical MT译作代表性 MT，其定义是，泛型全开放类型或者普通无泛型类型的MT。比如List&lt;A&gt; List&lt;B&gt;就有一个共同的Canonical MT -> List&lt;T&gt;

## 排布
废话不多说，直接上图:

![](https://github.com/HexJacaranda/BlogOfHexJacaranda/blob/master/images/Method%20Table%20Layout.jpg)

这里读者便可以一览整个MT的内存排布了，再让我们深入一点，看看背后还有哪一些细节。

## 排布细节
方便起见，我们从上到下逐个讲解。

1.关于打头的一个Flag集合，实际上类型是DWORD，其低位的WORD被用于记录数组和字符串的Component Size，所以看起来应当是这样的，(dwFlag) -> (各种Flag|Component Size)

2.类Token实际上小于16位就会被储存于指定的命名区域，而如果大于16位，将会被移动到溢出Token的可选成员中，这也意味着整个程序集里面拥有大概64K个类型，很有趣的是，在溢出Token可选成员上方有一句这样的注释:"TypeDef token for assemblies with more than 64k types. Never happens in real world."。

3.在泛型类型被实例化后，实际上图中第一个匿名联合所包含的组中，m_pCannonMT是有效的，也就是实例化的泛型MT会在此字段指向代表性MT。而代表性MT，也就要么是完全开放泛型，或者普通类型，其m_pEEClass是有效的。也从侧面揭示了，泛型实例化之间是相互共享EEClass的。在PreJIT的特性引入下，m_pCannonMT也可能是一个Indirection指向真正的代表性MT。

4.m_pParentMethodTable 可能有两种形式，第一种直接指向父类MT，第二种则是指向Indirection，Indirection指向父类MT。而这两种情况是取决于恢复的MT是否被编码修复了，此变动的引起与特性PreJIT有关。

5.虚方法通过Indirection指向大小为8的Chunk的二级结构来实现，表面上我们仍然使用逻辑上的线性排布的索引来访问对应的虚方法，经由相关API完成映射。

6.多用途槽位可以被以下槽位占用：DispatchMapSlot，NonVirtualSlots，ModuleOverride，虽然m_pPerInstInfo和m_pInterfaceMap因为JIT代码和Helper的性能敏感性必须被放在在固定的偏移处，但实际上这两个字段不会经常储存在这两个匿名联合组中，我们遵循先来先到原则，当固定的字段没有被设置时，多用途槽位将被启用。
 
7.溢出的多用途槽位实际在逻辑上与在MethodTable体内的多用途槽位 -1，-2是一体的，当多用途槽位的联合没有被相应成员使用时，溢出多用途槽内的对应槽会按顺序占用空闲的多用途槽位。

8.可选成员数据实际是与溢出多用途槽位紧密相连的，都是属于可选范畴，实际上溢出的多用途槽位最后有一个槽是用于标记可选成员开始的偏移量。

9.接口图附加的数据存储：为了更好的数据密度，我们在一些独立可选的位置放置这些信息，如果我们把这些信息放置于接口图中，那么一个bool值在内存对齐的要求下需要32位甚至64位来储存。目前我们只有一个Flag被储存IsDeclaredOnClass，也就是这个接口是否被此类型显式声明。因此我们采用了内联化存储形式，在sizeof(TADDR)*8之下直接将一个指针可选成员(额外接口图信息)当作位图储存，当超过这个范围时，就为相应数据分配内存块，可选的指针成员指向内存块，在内存块中储存信息。

10.非虚方法块槽位实际上也采用了内联化储存的技巧，当只有一个非虚方法时，其直接被存储在非虚方法块槽中。当有多个方法时则分配内存块。

11.泛型实例化字典实际排布，先直接看图：

![](https://github.com/HexJacaranda/BlogOfHexJacaranda/blob/master/images/Generic%20Dictionary%20Layout.jpg)

这里的句柄用于运行时存放类型，或者方法。溢出桶指针指向下一个桶，里面仅存放运行时句柄。

## 选读节 - 多用途槽排布计算 - 元编程的应用
在上面一小节我们提到了多用途槽与溢出多用途槽实际在逻辑上是连续一体的，但其在内存上又是不相邻的，因此就让我们看一看其映射逻辑是如何的。
首先我们看到GetMultipurposeSlotPtr函数

    TADDR GetMultipurposeSlotPtr(WFLAGS2_ENUM flag, const BYTE * offsets)
    {
        _ASSERTE(GetFlag(flag));
        DWORD offset = offsets[GetFlag((WFLAGS2_ENUM)(flag - 1))];
        if (offset >= sizeof(MethodTable)){
            offset += GetNumVtableIndirections() * sizeof(VTableIndir_t);
        }
        return dac_cast<TADDR>(this) + offset;
    }

我们暂且不管offset是如何计算的，只需要看到后面的代码。如果获取的offset超出了MethodTable本体之外(也就是开头连续的蓝色块之外)，就加上虚方法块槽的大小，偏移到其之后。进一步推断，假若多用途槽没有被占用且目标槽位放置在那里，得到的offset就会在MethodTable之内，直接通过this + offset访问。这正好对应了两部分槽位分离的情况。

那么继续，看到最上面的语句，从offsets数组中获取offset。首先是通过GetFlag函数对flag - 1取值，映射到对应的offset数组的索引上。这样做原因是什么呢，我们再看到对应多用途槽位的Enum值。
    
        enum_flag_HasPerInstInfo            = 0x0001,
        enum_flag_HasInterfaceMap           = 0x0002,
        enum_flag_HasDispatchMapSlot        = 0x0004,
        enum_flag_HasNonVirtualSlots        = 0x0008,
        enum_flag_HasModuleOverride         = 0x0010,
再看看GetFlag是怎么样的

    __forceinline DWORD GetFlag(WFLAGS2_ENUM flag) const
    {
        return m_wFlags2 & flag;
    }
直接通过一个按位与返回。

如果看不太明白，没有关系，我们取一个实例进行计算，以enum_flag_HasNonVirtualSlots为例，其高位不展开，只看最后低位是1000，如果其-1，就会成为0111。这是因为函数开头保证当前的Flag是一定存在的，那么其最低位二进制值一定是这样的：1???，做按位与，则会得到0???。类似的，我们取其他Flag计算，会得到这样的结果：

    enum_flag_HasPerInstInfo        0000
    enum_flag_HasInterfaceMap       000?
    enum_flag_HasDispatchMapSlot    00??
    enum_flag_HasNonVirtualSlots    0???
    enum_flag_HasModuleOverride     ????
发现规律没有，计算结果对应的索引，是由?决定的。这低四位中每一位都对应着一个Flag，也就是比当前Flag数值小的Flag设置情况决定了我们当前Flag通过这样的计算能够获取到的索引。再更近一点，各个Flag能够获取到的索引值范围是这样的:

    enum_flag_HasPerInstInfo        0
    enum_flag_HasInterfaceMap       0 - 1
    enum_flag_HasDispatchMapSlot    0 - 3
    enum_flag_HasNonVirtualSlots    0 - 7
    enum_flag_HasModuleOverride     0 - 15
所以这些索引对应的数组又是如何计算出来的呢?让我们看到一组宏定义

    #define MULTIPURPOSE_SLOT_OFFSET_1(mask) MULTIPURPOSE_SLOT_OFFSET  (mask) MULTIPURPOSE_SLOT_OFFSET  (mask | 0x01)
    #define MULTIPURPOSE_SLOT_OFFSET_2(mask) MULTIPURPOSE_SLOT_OFFSET_1(mask) MULTIPURPOSE_SLOT_OFFSET_1(mask | 0x02)
    #define MULTIPURPOSE_SLOT_OFFSET_3(mask) MULTIPURPOSE_SLOT_OFFSET_2(mask) MULTIPURPOSE_SLOT_OFFSET_2(mask | 0x04)
    #define MULTIPURPOSE_SLOT_OFFSET_4(mask) MULTIPURPOSE_SLOT_OFFSET_3(mask) MULTIPURPOSE_SLOT_OFFSET_3(mask | 0x08)
    #define MULTIPURPOSE_SLOT_OFFSET_5(mask) MULTIPURPOSE_SLOT_OFFSET_4(mask) MULTIPURPOSE_SLOT_OFFSET_4(mask | 0x10)
    #define MULTIPURPOSE_SLOT_OFFSET(mask) MultipurposeSlotOffset<mask>::slotOffset,
其中DispatchMap的生成Offset，这里没有前两个是因为他们可以直接通过已经命名的联合成员访问。

    const BYTE MethodTable::c_DispatchMapSlotOffsets[] = {
        MULTIPURPOSE_SLOT_OFFSET_2(0)
    };
这组宏，递归展开并通过MultipurposeSlotOffset这个模板计算每一个情况需要的Slot偏移，那么可以暂且先不管宏怎么写，现在只需要关注MultipurposeSlotOffset这个模板是如何计算的。

    template<int mask>
    struct MethodTable::MultipurposeSlotOffset
    {
        //此raw值是根据先来先到原则计算的槽位索引
        enum { raw = CountBitsAtCompileTime<mask>::value };
        //这才是真正的槽位索引，除开第一个多用途槽位没有被使用但第二个多用途槽位被使用的情况，其余情况都与raw值保持一致
        //在上述提到的特殊情况中，第一个多用途槽位仍然要被我们使用起来。
        enum { index = (((mask & 3) == 2) && (raw == 1)) ? 0 : raw };
        //槽位的偏移量
        enum { slotOffset = (index == 0) ? offsetof(MethodTable, m_pMultipurposeSlot1) :
                        (index == 1) ? offsetof(MethodTable, m_pMultipurposeSlot2) :
                        (sizeof(MethodTable) + index * sizeof(TADDR) - 2 * sizeof(TADDR)) };
        //Method Table连带溢出槽位的大小，被用于计算可选成员的起始索引
        enum { totalSize = (slotOffset >= sizeof(MethodTable)) ? slotOffset : sizeof(MethodTable) };            
    }
这里我们拿出CountBitsAtCompileTime的模板代码

    template<int N>
    struct CountBitsAtCompileTime
    {
        enum { value = (N & 1) + CountBitsAtCompileTime<(N >> 1)>::value };
    };
    template<>
    struct CountBitsAtCompileTime<0>
    {
        enum { value = 0 };
    };
这段代码相对简单，通过递归计算把N的每一位值加起来。比如N是1001 ，那么得到的结果就是2。这也解释了为什么此计算值是先来先到原则，如果低位没有被设置，那么我的值也会相对小。举个例子，状态若是1011，得到的索引就是3，但假如状态是1001，我们得到的索引就是2。

我们再看回index计算表达式，首先是(mask & 3) == 2，表示mask必为??10，而随后raw值为1，说明整个mask中只有一个1，这两者相结合就只能是0010这种情况了，与注释提到的第一个没有被用而第二个被用的情况一致，这种情况下，根据三目表达式，将索引设置为0。否则就设置为raw值。

slotOffset也相对好读得多，如果索引为0或1，那么就直接取多用途槽的偏移，否则就先加上Method Table大小，再加上index个槽位的大小，最后减去前两个固定槽位的大小。看到这里是否恍然大悟，slotOffset在溢出部分索引计算时首先加上了Method Table的大小，这与之前GetMultipurposeSlotPtr的逻辑保持了一致。

在明白了整个核心计算逻辑之后，我们回到宏上面，看看到底是如何为槽位生成索引数组的。以最简单的为例子，一步一步展开这个宏

    const BYTE MethodTable::c_DispatchMapSlotOffsets[] = {
        MULTIPURPOSE_SLOT_OFFSET_2(0)
    };
    第一步
    MULTIPURPOSE_SLOT_OFFSET_1(0) MULTIPURPOSE_SLOT_OFFSET_1(0|0x2)
    第二步
    MULTIPURPOSE_SLOT_OFFSET(0) MULTIPURPOSE_SLOT_OFFSET(0|0x1) MULTIPURPOSE_SLOT_OFFSET(0|0x2) MULTIPURPOSE_SLOT_OFFSET(0|0x2|0x1)

然后我们就可以开始愉快地计算这四项值

    表达式                                          raw                 index
    MULTIPURPOSE_SLOT_OFFSET(0)                     0                    0
    MULTIPURPOSE_SLOT_OFFSET(0|0x1)                 1                    1
    MULTIPURPOSE_SLOT_OFFSET(0|0x2)                 1                    0
    MULTIPURPOSE_SLOT_OFFSET(0|0x2|0x1)             2                    2

看到这个表之后，是否感觉到了一点什么?这里的|运算符意图太明显了，不知道各位明白没有，这里0x1 , 0x2对应的正是上面那些槽位占用的Flag，整个展开过程相当于在穷举低位Flag被占用的各种情况。然后根据这几种情况通过模板分别计算出对应的槽位该放在哪个偏移处。

所以这整个一套的过程我们再捋一捋。
1.宏用于穷举低位Flag的所有占用情况，调用模板进行计算对应情况下的偏移，形成数组
2.GetFlag(flag - 1)利用了枚举值特殊的数学关系，获取到低位Flag占用情况，作为索引映射到固有数组上获取偏移。
3.偏移的计算是根据各种规则计算的index来决定取哪个槽位的地址。小于2就取固定的多用途槽位，大于等于2就是数组偏移 + MethodTable的大小来做区分。

##  完全加载
类型的完全加载需要其依赖全部被加载，这里依赖项指:父类，接口，代表性类型，经典类型

## 对象的大小限制
虽然Conponent Size是16位的，但是所有计算对象大小的地方都使用了SIZE_T类型，因为支持大于2GB的对象是非常重要的。

## System.Void 数组
CoreCLR其实并没有阻止System.Void类型的数组存在，即使其Compinent Size为0。

## 三种CorElementType
签名类型:表面上的
验证者类型:与签名类型一致，但枚举被转换为其底层的类型
内部类型:枚举被转换为底层类型，System.Int32等将是底层基元类型，而一些可以被优化成int组合结构体，如struct type{public int i;} 在x86下会成为ELEMENT_TYPE_I4而不是ELEMENT_TYPE_VALUETYPE，这种类型的值类型被称为基元值类型。其被用于调用约定之中，JIT且会从这种类型中受益(优化)。
