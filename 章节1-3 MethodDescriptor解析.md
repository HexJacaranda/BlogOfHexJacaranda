# MethodDescriptor解析
## MehtodDescriptor定义
**MethodDescriptor**(以下简称**MD**)，是一个类型中的某一个方法的表示，他们存在于**MethodDescChunk**中，而**MDChunk**存在于**EEClass**之中。

概念上讲，**MD**所代表的元数据是“冷”的(在正常程序执行中，我们不期望访问他们，但我们经常没有做到这点)

一个MD知道如何获取其元数据Token(通过**GetMemberDef**)，其所在的Chunk(通过**MethodDescChunk**)，而**MethodDescChunk**知道如何获取其类型(通过**MethodTable**)。其也知道如何获取对应对的IL代码(通过**IMAGE_COR_ILMETHOD**)

这里因为*botr*有相应的文档，所以建议先阅读翻译版本再继续: 






## MethodDescriptor成员
### 字段
一览(除去Debug字段)

| 类型 | 名称 |
| ---- | ---- |
| UINT16 | m_wFlags3AndTokenRemainder |
| BYTE | m_chunkIndex |
| BYTE | m_bFlags2 |
| WORD | m_wSlotNumber |
| WORD | m_wFlags |

接下来就是介绍字段用途了

**m_wFlags3AndTokenRemainder：**
其对应的enum组:

| 名称 | 值 |
| ---- | ---- |
| enum_flag3_TokenRemainderMask | 0x3FFF | 
| enum_flag3_HasForwardedValuetypeParameter | 0x4000 |
| enum_flag3_ValueTypeParametersWalked | 0x4000 |
| enum_flag3_DoesNotHaveEquivalentValuetypeParameters | 0x8000 |

目前本字段只有5个位可以被使用。然而，新的位相当难以访问，因此新的flag有相当的存在必要时才被添加。

* **TokenRemainderMask**
此值必须等于此前计算的**METHOD_TOKEN_REMAINDER_MASK**，这些值被分开以允许flag的可用空间存在，并且为了将此Token分裂的逻辑算法能够被利用定义的宏算法生成。
* **HasForwardedValuetypeParameter**
指示有TypeForward的ValueType被用于参数，此flag仅对NGen处理的项有效。
注：有关针对ValueType的TypeForward的处理在**MethodTable**解析中的**DoFullyLoad**函数有提及。
* **ValueTypeParametersWalked**
指示方法签名中的ValueType引用是否被解析到了具体定义(或者这个过程已经失败)，此flag只对非NGen方法有效。
注：在**DoFullyLoad**中，含有ValueType参数的方法被特殊处理，其特殊之处在于值类型必须被解析出来，得到具体的排布，否则JIT无法为GC正常生成引用信息。
* **DoesNotHaveEquivalentValuetypeParameters**
指示当前是否已经验证了当前方法不存在等价的ValueType参数

**m_chunkIndex:**
指示在MDChunk中的索引

**m_bFlags2**
其对应的enum组

| 名称 | 值 |
| ---- | ---- |
| enum_flag2_HasStableEntryPoint | 0x01 | 
| enum_flag2_HasPrecode | 0x02 |
| enum_flag2_HasNativeCodeSlot | 0x04 |
| enum_flag2_IsJitIntrinsic | 0x08 |
| enum_flag2_IsEligibleForTieredCompilation | 0x20 |

