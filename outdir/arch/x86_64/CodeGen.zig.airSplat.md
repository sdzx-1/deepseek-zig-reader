graph TD
    Start[开始] --> GetInfo[获取类型和操作数信息]
    GetInfo --> SwitchScalarType{根据标量类型判断}
    
    SwitchScalarType --> |bool| BoolBranch[布尔类型处理]
    BoolBranch --> AllocRegs[分配两个GP寄存器]
    AllocRegs --> GenSetZero[生成设置0到寄存器]
    GenSetZero --> GenSetMax[生成设置全1掩码到寄存器]
    GenSetMax --> ResolveOperand[解析操作数]
    ResolveOperarnd --> Cmovcc[生成条件移动指令]
    Cmovcc --> ResultReg[结果存入寄存器]
    
    SwitchScalarType --> |int| IntBranch[整数类型处理]
    IntBranch --> CheckAVX2{有AVX2?}
    CheckAVX2 --> |是| AVX2Path[使用VPBROADCAST指令]
    CheckAVX2 --> |否| SSEPath[使用SSE展开策略]
    AVX2Path --> AllocSSEReg[分配SSE寄存器]
    AVX2Path --> MemOrReg{操作数是内存?}
    MemOrReg --> |是| MemBroadcast[内存广播]
    MemOrReg --> |否| RegBroadcast[寄存器广播]
    SSEPath --> DupAndShuffle[展开和洗牌指令序列]
    
    SwitchScalarType --> |float| FloatBranch[浮点类型处理]
    FloatBranch --> CheckFloatBits{浮点位数判断}
    CheckFloatBits --> |32位| Float32[处理float32]
    CheckFloatBits --> |64位| Float64[处理float64]
    CheckFloatBits --> |128位| Float128[处理float128]
    
    Float32 --> CheckVectorLen32{向量长度判断}
    CheckVectorLen32 --> |1| DirectMove[直接移动]
    CheckVectorLen32 --> |2-4| ShuffleOrBroadcast[洗牌或广播]
    CheckVectorLen32 --> |5-8| AVXBroadcast[AVX广播]
    
    Float64 --> CheckVectorLen64{向量长度判断}
    CheckVectorLen64 --> |1| DirectMove64[直接移动]
    CheckVectorLen64 --> |2| Movddup[MOVDDUP指令]
    CheckVectorLen64 --> |3-4| AVXBroadcast64[AVX广播]
    
    Float128 --> CheckVectorLen128{向量长度判断}
    CheckVectorLen128 --> |1| DirectMove128[直接移动]
    CheckVectorLen128 --> |2| AVXInsert[AVX插入指令]
    
    ResultReg --> Finish[返回结果]
    MemBroadcast --> Finish
    RegBroadcast --> Finish
    DupAndShuffle --> Finish
    DirectMove --> Finish
    ShuffleOrBroadcast --> Finish
    AVXBroadcast --> Finish
    DirectMove64 --> Finish
    Movddup --> Finish
    AVXBroadcast64 --> Finish
    DirectMove128 --> Finish
    AVXInsert --> Finish
    
    SwitchScalarType --> |其他类型| TodoError[返回TODO错误]
    CheckFloatBits --> |其他| TodoError
    CheckVectorLen32 --> |其他| TodoError
    CheckVectorLen64 --> |其他| TodoError
    CheckVectorLen128 --> |其他| TodoError
    TodoError --> Finish
