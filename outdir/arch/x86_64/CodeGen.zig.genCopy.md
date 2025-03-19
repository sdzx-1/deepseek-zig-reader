graph TD
    Start[开始genCopy] --> CheckDstMCV{检查dst_mcv类型}
    
    CheckDstMCV -->|不可修改类型| Unreachable[触发unreachable]
    
    CheckDstMCV -->|.register| GenSetReg[调用genSetReg直接设置寄存器]
    
    CheckDstMCV -->|.register_offset| AdjustOffset[调整源值偏移量]
    AdjustOffset -->|生成新MCV| GenSetRegOffset[调用genSetReg设置寄存器]
    
    CheckDstMCV -->|寄存器组类型| HandleRegGroup[处理多寄存器复制]
    HandleRegGroup --> CheckSrcType{检查源值类型}
    
    CheckSrcType -->|SSE寄存器| GenerateSSEMov[生成SSE移动指令]
    CheckSrcType -->|寄存器组| ResolveHazard[解析寄存器冲突]
    ResolveHazard -->|交换循环| HazardLoop[循环处理冲突寄存器]
    HazardLoop -->|交换/xch指令| FinishRegGroup[完成寄存器组复制]
    
    CheckSrcType -->|符号加载类型| LoadSymbol[分配地址寄存器]
    LoadSymbol --> GenerateMemAccess[生成内存访问指令]
    
    HandleRegGroup --> EmitHazardLoop[发射危险处理循环]
    EmitHazardLoop -->|分两次发射| GenPartMov[生成部分移动指令]
    
    CheckDstMCV -->|.indirect| GenSetMemIndirect[调用genSetMem写入内存]
    
    CheckDstMCV -->|内存/符号类型| HandleMemDst[处理内存目标]
    HandleMemDst -->|需要临时寄存器| AllocTmpReg[分配临时寄存器]
    AllocTmpReg --> GenSetMemTmp[用临时寄存器调用genSetMem]
    
    CheckDstMCV -->|.load_frame| GenSetFrameMem[直接调用genSetMem写帧内存]
    
    GenSetReg --> End
    GenSetRegOffset --> End
    FinishRegGroup --> End
    GenerateMemAccess --> End
    GenPartMov --> End
    GenSetMemIndirect --> End
    GenSetMemTmp --> End
    GenSetFrameMem --> End
    
    End[结束genCopy]
