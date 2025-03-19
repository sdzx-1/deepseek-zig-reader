graph TD
    Start[开始] --> GetData[获取指令数据和类型]
    GetData --> CheckVector[检查是否是向量类型]
    CheckVector --> |是| FailVector[返回错误: 不支持向量]
    CheckVector --> |否| GetIntInfo[获取整数类型信息]
    
    GetIntInfo --> CheckBits{检查整数位数}
    
    CheckBits --> |32位| Handle32[处理32位整数]
    Handle32 --> Upcast32[提升到64位]
    Upcast32 --> Mul32[执行乘法]
    Mul32 --> Trunc32[截断回原类型]
    Trunc32 --> Compare32[比较结果是否溢出]
    Compare32 --> SetOverflow32[设置溢出位]
    
    CheckBits --> |64位| Handle64[处理64位整数]
    Handle64 --> Upcast64[提升到128位]
    Upcast64 --> Mul64[执行乘法]
    Mul64 --> Trunc64[截断回原类型]
    Trunc64 --> Compare64[比较结果是否溢出]
    Compare64 --> SetOverflow64[设置溢出位]
    
    CheckBits --> |128位| CheckSigned{检查符号}
    CheckSigned --> |无符号| Handle128u[处理128位无符号]
    Handle128u --> SplitOperands[分解高低64位]
    SplitOperands --> CallMulti3[调用__multi3计算交叉乘积]
    CallMulti3 --> CheckOverflow[检查高位非零或相加溢出]
    CheckOverflow --> SetOverflow128u[设置溢出位]
    
    CheckSigned --> |有符号| Handle128s[处理128位有符号]
    Handle128s --> CallMuloti4[调用__muloti4]
    CallMuloti4 --> GetOverflowBit[读取溢出标志]
    GetOverflowBit --> SetOverflow128s[设置溢出位]
    
    SetOverflow32 --> StoreResult
    SetOverflow64 --> StoreResult
    SetOverflow128u --> StoreResult
    SetOverflow128s --> StoreResult
    
    StoreResult --> AllocStack[分配结果栈空间]
    AllocStack --> StoreValue[存储计算结果]
    StoreValue --> StoreOverflow[存储溢出位]
    StoreOverflow --> Finish[结束并返回结果]
    
    FailVector --> Finish
