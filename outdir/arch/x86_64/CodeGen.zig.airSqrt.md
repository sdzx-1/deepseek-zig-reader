flowchart TD
    Start[开始] --> GetTypeInfo[获取类型信息: ty, abi_size]
    GetTypeInfo --> CheckType{检查类型}
    
    CheckType --> |Float| HandleFloat[处理浮点类型]
    CheckType --> |Vector| HandleVector[处理向量类型]
    CheckType --> |其他类型| Unreachable[无法处理]
    
    HandleFloat --> CheckFloatBits{检查浮点位数}
    CheckFloatBits --> |16/32/64位| CheckFeatures{检查硬件特性}
    CheckFloatBits --> |80/128位| CallLib[生成库函数调用]
    
    CheckFeatures --> |支持F16C/AVX| GenFloatInstr[生成浮点指令]
    CheckFeatures --> |不支持| CallLib
    
    HandleVector --> CheckElemType{检查元素类型}
    CheckElemType --> |Float| CheckElemBits{检查元素位数}
    CheckElemType --> |其他类型| Unreachable
    
    CheckElemBits --> |16位| CheckVectorLen{检查向量长度}
    CheckElemBits --> |32/64位| GenVecInstr[生成向量指令]
    
    CheckVectorLen --> |1| Convert16to32[16位转32位并计算]
    CheckVectorLen --> |2-8| WideRegConvert[使用宽寄存器转换并计算]
    
    GenFloatInstr --> |生成SS/SD指令| RegManage[寄存器管理]
    GenVecInstr --> |生成PS/PD指令| RegManage
    Convert16to32 --> RegManage
    WideRegConvert --> RegManage
    
    RegManage --> |复制到寄存器/锁定| GenMirTag[确定指令标签]
    GenMirTag --> GenAsm[生成汇编指令]
    
    CallLib --> GenLibCall[生成库函数调用路径]
    
    GenAsm --> Finish[完成指令生成]
    GenLibCall --> Finish
    Unreachable --> Fail[返回失败]
    
    Finish --> End[结束]
    Fail --> End
