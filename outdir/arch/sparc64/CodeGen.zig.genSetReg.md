graph TD
    Start[("genSetReg开始")] --> CheckMCV{检查mcv类型}
    
    CheckMCV --> |.dead| Unreachable[("触发unreachable")]
    CheckMCV --> |.unreach/.none| Return1[("直接返回")]
    CheckMCV --> |.condition_flags| ConditionFlags
    CheckMCV --> |.condition_register| ConditionRegister
    CheckMCV --> |.undef| Undef
    CheckMCV --> |.ptr_stack_offset| PtrStackOffset
    CheckMCV --> |.immediate| Immediate
    CheckMCV --> |.register| Register
    CheckMCV --> |.memory| Memory
    CheckMCV --> |.stack_offset| StackOffset
    
    ConditionFlags --> AddMov[添加mov指令]
    AddMov --> AddMovCC[添加movcc指令]
    AddMovCC --> End
    
    ConditionRegister --> AddMov2[添加mov指令]
    AddMov2 --> AddMovR[添加movr指令]
    AddMovR --> End
    
    Undef --> CheckSafety{检查安全模式}
    CheckSafety --> |否| Return2[("直接返回")]
    CheckSafety --> |是| GenUndef[生成0xaaaaaaaaaaaaaaaa]
    GenUndef --> End
    
    PtrStackOffset --> CalcOffset[计算栈偏移]
    CalcOffset --> AddAdd[添加add指令]
    AddAdd --> End
    
    Immediate --> CheckImmediateSize{检查立即数大小}
    CheckImmediateSize --> |≤u12| AddMovImm[添加mov指令]
    CheckImmediateSize --> |≤u32| AddSethi[添加sethi指令]
    AddSethi --> AddOr1[添加or指令]
    CheckImmediateSize --> |≤u44| GenShift[生成高位立即数]
    GenShift --> AddSllx[添加sllx指令]
    AddSllx --> AddOr2[添加or指令]
    CheckImmediateSize --> |>u44| AllocTmp[分配临时寄存器]
    AllocTmp --> GenLow[生成低32位]
    GenLow --> GenHigh[生成高32位]
    GenHigh --> AddSllx32[添加sllx 32位]
    AddSllx32 --> AddOr3[添加or指令]
    
    Register --> CheckSameReg{是否相同寄存器?}
    CheckSameReg --> |是| Return3[("直接返回")]
    CheckSameReg --> |否| AddMovReg[添加mov指令]
    AddMovReg --> End
    
    Memory --> GenAddr[生成地址到寄存器]
    GenAddr --> GenLoad[添加加载指令]
    GenLoad --> End
    
    StackOffset --> CalcStackOffset[计算栈偏移]
    CalcStackOffset --> GenLoadStack[添加加载指令]
    GenLoadStack --> End
    
    AddMovImm --> End
    AddOr1 --> End
    AddOr2 --> End
    AddOr3 --> End
    
    End[("流程结束")]
    
    subgraph 立即数处理
        CheckImmediateSize
        AddMovImm
        AddSethi
        AddOr1
        GenShift
        AddSllx
        AddOr2
        AllocTmp
        GenLow
        GenHigh
        AddSllx32
        AddOr3
    end
