graph TD
    Start[开始 genSetReg] --> CheckMCV{检查 mcv 类型}
    
    CheckMCV -->|.dead| Unreachable[触发 unreachable]
    CheckMCV -->|.unreach, .none| ReturnNothing[直接返回]
    
    CheckMCV -->|.undef| CheckSafety{检查 wantSafety}
    CheckSafety -->|是| WriteUndef[生成立即数 0xaaaaaaaa]
    CheckSafety -->|否| ReturnNothing
    
    CheckMCV -->|.ptr_stack_offset| CheckOffsetValid{检查偏移量是否合法}
    CheckOffsetValid -->|有效| AddSubInst[生成 sub 指令]
    CheckOffsetValid -->|无效| Fail[报错 TODO]
    
    CheckMCV -->|.cpsr_flags| GenerateFlags[生成条件移动指令]
    GenerateFlags --> MovZero[生成 mov reg, 0]
    MovZero --> MovCondOne[生成条件 mov reg, 1]
    
    CheckMCV -->|.immediate| CheckImmediateValid{检查立即数是否可用}
    CheckImmediateValid -->|直接 mov| AddMov[生成 mov 指令]
    CheckImmediateValid -->|取反 mvn| AddMvn[生成 mvn 指令]
    CheckImmediateValid -->|u16 范围| CheckV7Feature{检查 v7 特性}
    CheckV7Feature -->|支持| AddMovw[生成 movw 指令]
    CheckV7Feature -->|不支持| AddMultiOrr[分步生成 orr 指令]
    CheckImmediateValid -->|大立即数| SplitImmediate[拆分立即数为 movw/movt 或多步 orr]
    
    CheckMCV -->|.register| CheckSameReg{检查寄存器相同}
    CheckSameReg -->|不同| AddMovReg[生成 mov reg, src_reg]
    CheckSameReg -->|相同| ReturnNothing
    
    CheckMCV -->|.memory| LoadMemory[生成地址加载 + ldr 指令]
    LoadMemory --> GenImmediate[递归调用 genSetReg]
    GenImmediate --> GenLdr[调用 genLdrRegister]
    
    CheckMCV -->|.stack_offset| HandleStackOffset[根据类型生成加载指令]
    HandleStackOffset --> CheckSize{检查 abi_size}
    CheckSize -->|1| LdrByte[生成 ldrb/ldrsb]
    CheckSize -->|2| LdrHalf[生成 ldrh/ldrsh]
    CheckSize -->|3-4| LdrWord[生成 ldr]
    CheckSize -->|其他| Unreachable
    
    CheckMCV -->|.stack_argument_offset| LoadStackArg[生成栈参数加载指令]
    
    AddSubInst --> End
    Fail --> End
    WriteUndef --> End
    MovCondOne --> End
    AddMov --> End
    AddMvn --> End
    AddMovw --> End
    AddMultiOrr --> End
    SplitImmediate --> End
    AddMovReg --> End
    GenLdr --> End
    LdrByte --> End
    LdrHalf --> End
    LdrWord --> End
    LoadStackArg --> End
    
    End[结束]
    
    style Start fill:#90EE90,stroke:#333
    style CheckMCV fill:#FFD700,stroke:#333
    style Unreachable fill:#FF6347,stroke:#333
    style ReturnNothing fill:#87CEEB,stroke:#333
    style End fill:#FF6347,stroke:#333
