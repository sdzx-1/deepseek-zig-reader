flowchart TD
    Start[开始] --> GetInfo[获取指令信息、操作类型、左右操作数类型]
    GetInfo --> CheckType{检查左操作数类型}
    CheckType --> |整数类型| IntType[处理整数类型]
    CheckType --> |向量类型| VectorType[处理向量类型]
    CheckType --> |其他类型| TodoError[返回TODO错误]
    
    IntType --> SpillReg[保存寄存器状态(spillRegisters), 获取RCX寄存器]
    SpillReg --> ResolveOperands[解析左右操作数值]
    ResolveOperands --> GenShiftOp[生成位移操作指令]
    GenShiftOp --> CheckShiftType{检查位移类型}
    
    CheckShiftType --> |shl/shl_exact| TruncateCheck[检查目标存储类型]
    TruncateCheck --> |寄存器| TruncReg[截断寄存器]
    TruncateCheck --> |寄存器对| TruncRegPair[截断第二个寄存器]
    TruncateCheck --> |内存地址| HandleFrame[生成临时寄存器并截断]
    CheckShiftType --> |shr/shr_exact| NoTrunc[无需截断]
    
    VectorType --> CheckElemType{检查向量元素类型}
    CheckElemType --> |整数元素| ProcessVector[处理向量位移]
    ProcessVector --> CheckBitsLength{根据元素位数选择指令集}
    CheckBitsLength --> |16位| CheckVectorLen16[检查向量长度]
    CheckBitsLength --> |32位| CheckVectorLen32[检查向量长度]
    CheckBitsLength --> |64位| CheckVectorLen64[检查向量长度]
    
    CheckVectorLen16 --> |1-8元素| SelectSSEorAVX[选择SSE/AVX指令]
    CheckVectorLen16 --> |9-16元素| CheckAVX2[检查AVX2支持]
    CheckVectorLen32 --> |1-4元素| SelectSSEorAVX
    CheckVectorLen32 --> |5-8元素| CheckAVX2
    CheckVectorLen64 --> |1-2元素| SelectSSEorAVX
    CheckVectorLen64 --> |3-4元素| CheckAVX2
    
    SelectSSEorAVX --> |有AVX支持| UseAVX[使用VP指令]
    SelectSSEorAVX --> |无AVX支持| UseSSE[使用P指令]
    CheckAVX2 --> |有AVX2支持| UseAVX2[使用VP指令]
    CheckAVX2 --> |无AVX2支持| Skip[不支持则跳过]
    
    ProcessVector --> HandleRHS{处理右操作数}
    HandleRHS --> |立即数| GenImmShift[生成立即数位移]
    HandleRHS --> |splat指令| GenSplatShift[生成广播位移]
    GenSplatShift --> MaskShift[生成掩码并应用]
    
    GenImmShift --> |AVX模式| AVXImmShift[使用三操作数指令]
    GenImmShift --> |非AVX模式| SSEImmShift[使用双操作数指令]
    
    MaskShift --> LoadMask[加载掩码地址]
    LoadMask --> ApplyMask[应用掩码到位移量]
    ApplyMask --> GenShiftInst[生成最终位移指令]
    
    GenShiftInst --> VectorResult[返回向量寄存器结果]
    
    IntType --> IntResult[返回整数结果]
    TodoError --> Fail[抛出TODO错误]
    
    IntResult --> End[结束]
    VectorResult --> End
    Fail --> End
