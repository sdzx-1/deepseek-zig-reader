graph TD
    Start[开始 airArrayElemVal] --> ResolveInst[解析 bin_op.lhs 和 bin_op.rhs]
    ResolveInst --> CheckVector{检查是否为向量数组且元素大小1位?}
    
    CheckVector -->|是| VectorHandle[处理向量数组逻辑]
    CheckVector -->|否| NonVectorHandle[处理普通数组逻辑]
    
    VectorHandle --> SpillEflags[溢出 EFLAGS 寄存器]
    SpillEflags --> VectorSwitch[根据数组存储类型选择处理方式]
    
    VectorSwitch -->|寄存器+立即数索引| BtImm[生成 bt 指令(寄存器+立即数)]
    VectorSwitch -->|寄存器+寄存器索引| BtReg[生成 bt 指令(寄存器+寄存器)]
    VectorSwitch -->|SSE 寄存器| SSECase[将向量存储到栈帧后处理]
    VectorSwitch -->|内存地址| MemCase[通过基地址+偏移处理]
    
    VectorHandle --> SetCC[生成 setc 指令存入目标寄存器]
    SetCC --> VectorEnd[返回寄存器结果]
    
    NonVectorHandle --> CalcAddr[计算数组基地址到 addr_reg]
    CalcAddr --> ElemOffset[计算元素偏移量到 offset_reg]
    ElemOffset --> AddOffset[addr_reg += offset_reg]
    AddOffset --> MemCopy[从地址拷贝元素到目标位置]
    MemCopy --> NonVectorEnd[返回内存/寄存器结果]
    
    VectorEnd --> Finish[结束并返回结果]
    NonVectorEnd --> Finish
