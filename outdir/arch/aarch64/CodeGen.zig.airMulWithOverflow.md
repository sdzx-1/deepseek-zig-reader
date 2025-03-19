flowchart TD
    Start[开始] --> CheckUnused{检查指令是否未使用?}
    CheckUnused -->|是| MarkDead[标记为dead并返回]
    CheckUnused -->|否| GetTypeInfo[获取类型信息]
    
    GetTypeInfo --> CheckIntType{是整数类型?}
    CheckIntType -->|否| FailVector[报错: 不支持向量类型]
    CheckIntType -->|是| CheckBitSize{检查整数位数}
    
    CheckBitSize -->|<=32位| Handle32Bit[处理32位及以下整数]
    CheckBitSize -->|<=64位| Handle64Bit[处理64位整数]
    CheckBitSize -->|>64位| FailLargeInt[报错: 不支持大整数]
    
    Handle32Bit --> AllocMem32[分配栈内存]
    AllocMem32 --> SpillFlags32[溢出比较标志寄存器]
    SpillFlags32 --> SelectMulOp[选择smull/umull指令]
    SelectMulOp --> TruncateResult[截断结果寄存器]
    TruncateResult --> CompareResult[比较高位结果]
    CompareResult --> SetStack32[设置栈值]
    SetStack32 --> ReturnResult32[返回栈偏移结果]
    
    Handle64Bit --> AllocMem64[分配栈内存]
    AllocMem64 --> SpillFlags64[溢出比较标志寄存器]
    SpillFlags64 --> AllocRegs64[分配寄存器组]
    AllocRegs64 -->|有符号| SignedOps[生成smulh/mul指令]
    AllocRegs64 -->|无符号| UnsignedOps[生成umulh/mul指令]
    SignedOps --> ShiftCompare[移位比较高位]
    UnsignedOps --> ZeroCheck[检查高位是否为零]
    ShiftCompare --> TruncateResult64[截断结果寄存器]
    ZeroCheck --> TruncateResult64
    TruncateResult64 --> SetStack64[设置栈值]
    SetStack64 --> ReturnResult64[返回栈偏移结果]
    
    MarkDead --> Finish[结束]
    FailVector --> Finish
    FailLargeInt --> Finish
    ReturnResult32 --> Finish
    ReturnResult64 --> Finish
