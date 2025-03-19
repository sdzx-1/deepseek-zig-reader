flowchart TD
    Start[开始] --> CheckUnused{检查指令是否未使用?}
    CheckUnused -- 是 --> FinishDead[返回.dead结果]
    CheckUnused -- 否 --> GetTypeInfo[获取类型信息]
    GetTypeInfo --> CheckType{检查类型是否为整数?}
    CheckType -- 向量 --> FailVector[返回未实现错误]
    CheckType -- 整数 --> CheckBits{检查整数位数}
    CheckBits -- <=16位 --> Handle16Bit[处理16位整数乘法]
    CheckBits -- <=32位 --> Handle32Bit[处理32位整数乘法]
    CheckBits -- >32位 --> FailLargeInt[返回未实现错误]

    Handle16Bit --> AllocStack16[分配栈空间]
    AllocStack16 --> SpillFlags16[溢出比较标志]
    SpillFlags16 --> GenMul16[生成乘法指令]
    GenMul16 --> Truncate16[截断寄存器]
    Truncate16 --> CompareTrunc16[比较结果与截断值]
    CompareTrunc16 --> SetStack16[设置栈值]
    SetStack16 --> SetOverflow16[设置溢出位]
    SetOverflow16 --> ResultStack16[返回栈偏移结果]

    Handle32Bit --> AllocStack32[分配栈空间]
    AllocStack32 --> SpillFlags32[溢出比较标志]
    SpillFlags32 --> AllocRegs32[分配多个寄存器]
    AllocRegs32 --> GenMul32[生成smull/umull指令]
    GenMul32 --> Truncate32[截断低位寄存器]
    Truncate32 --> StoreTrunc32[存储截断值到栈]
    StoreTrunc32 --> CompareTrunc32[比较截断值与低位]
    CompareTrunc32 --> SetCondition32[设置条件标志]
    SetCondition32 --> CheckHigh32[检查高位寄存器]
    CheckHigh32 --> SetOverflow32[设置溢出位]
    SetOverflow32 --> ResultStack32[返回栈偏移结果]

    FinishDead --> End[结束]
    FailVector --> End
    FailLargeInt --> End
    ResultStack16 --> End
    ResultStack32 --> End

    style Start fill:#9f9,stroke:#333
    style End fill:#f99,stroke:#333
    style CheckUnused,CheckType,CheckBits fill:#ccf,stroke:#333
    style FailVector,FailLargeInt fill:#fbb,stroke:#333
