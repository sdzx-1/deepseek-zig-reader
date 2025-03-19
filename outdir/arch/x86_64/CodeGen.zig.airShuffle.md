flowchart TD
    Start[开始处理airShuffle指令] --> GetInfo[获取类型信息/操作数/mask]
    GetInfo --> ProcessMask[处理mask生成mask_elems数组]
    ProcessMask --> CheckAllUndef{所有mask元素都是undef?}
    CheckAllUndef -- 是 --> AllocDst[分配目标寄存器/内存并返回]
    CheckAllUndef -- 否 --> CheckIdentityLHS{是否mask是LHS恒等排列?}
    CheckIdentityLHS -- 是 --> CopyLHS[复用或拷贝LHS到目标]
    CheckIdentityLHS -- 否 --> CheckIdentityRHS{是否mask是RHS恒等排列?}
    CheckIdentityRHS -- 是 --> CopyRHS[复用或拷贝RHS到目标]
    CheckIdentityRHS -- 否 --> TryUnpck[尝试UNPCKL/UNPCKH指令]
    
    TryUnpck --> CheckUnpckCond{满足UNPCK条件?}
    CheckUnpckCond -- 是 --> GenUnpck[生成UNPCK指令]
    CheckUnpckCond -- 否 --> TryPshufd[尝试PSHUFD指令]
    
    TryPshufd --> CheckPshufdCond{满足PSHUFD条件?}
    CheckPshufdCond -- 是 --> GenPshufd[生成PSHUFD指令]
    CheckPshufdCond -- 否 --> TryShufps[尝试SHUFPS指令]
    
    TryShufps --> CheckShufpsCond{满足SHUFPS条件?}
    CheckShufpsCond -- 是 --> GenShufps[生成SHUFPS指令]
    CheckShufpsCond -- 否 --> TryShufpd[尝试SHUFPD指令]
    
    TryShufpd --> CheckShufpdCond{满足SHUFPD条件?}
    CheckShufpdCond -- 是 --> GenShufpd[生成SHUFPD指令]
    CheckShufpdCond -- 否 --> TryBlend[尝试BLEND指令]
    
    TryBlend --> CheckBlendCond{满足BLEND条件?}
    CheckBlendCond -- 是 --> GenBlend[生成BLEND指令]
    CheckBlendCond -- 否 --> TryBlendv[尝试BLENDV指令]
    
    TryBlendv --> CheckBlendvCond{满足BLENDV条件?}
    CheckBlendvCond -- 是 --> GenBlendv[生成BLENDV指令]
    CheckBlendvCond -- 否 --> TryPshufb[尝试PSHUFB指令]
    
    TryPshufb --> CheckPshufbCond{满足PSHUFB条件?}
    CheckPshufbCond -- 是 --> GenPshufb[生成PSHUFB指令]
    CheckPshufbCond -- 否 --> Fallback[所有方案失败，触发TODO错误]
    
    GenUnpck --> End[返回结果]
    GenPshufd --> End
    GenShufps --> End
    GenShufpd --> End
    GenBlend --> End
    GenBlendv --> End
    GenPshufb --> End
    AllocDst --> End
    CopyLHS --> End
    CopyRHS --> End
    Fallback --> EndError[报错结束]
