graph TD
    Start[开始] --> CheckSize{检查目标寄存器大小是否足够}
    CheckSize -- 不足 --> Fail[报错并返回]
    CheckSize -- 足够 --> SwitchSrcMCV{根据src_mcv类型分支}

    SwitchSrcMCV -- undef --> HandleUndef[生成特定模式值(如0xaa...)]
    HandleUndef --> GenTypedValue[生成完整类型值并递归调用genSetReg]

    SwitchSrcMCV -- eflags --> HandleEflags[生成setcc指令]
    SwitchSrcMCV -- immediate --> HandleImmediate[处理立即数]
    
    HandleImmediate -- 零值 --> UseXor[使用xor清零寄存器]
    HandleImmediate -- 32位可表示 --> Mov32[使用32位mov指令]
    HandleImmediate -- 负数且<=4字节 --> MovSignExtend[使用符号扩展mov]
    HandleImmediate -- 其他 --> MovNormal[直接mov指令]

    SwitchSrcMCV -- register --> HandleRegister[处理寄存器间移动]
    HandleRegister -- 同寄存器 --> Skip[跳过]
    HandleRegister -- 不同寄存器类型 --> RegTypeSwitch{目标寄存器类型}
    
    RegTypeSwitch -- 通用寄存器 --> GP2GP[生成mov指令]
    RegTypeSwitch -- SSE寄存器 --> SSE2SSE[生成向量mov指令]
    RegTypeSwitch -- x87寄存器 --> FPUStack[处理浮点栈操作]
    RegTypeSwitch -- 其他类型 --> Unreachable[不可达]

    SwitchSrcMCV -- memory/load_* --> HandleMemory[处理内存加载]
    HandleMemory --> GetAddrReg[获取地址寄存器]
    GetAddrReg --> MoveStrategy[选择移动策略并生成指令]

    SwitchSrcMCV -- register_pair/quadruple --> HandleMultiReg[处理多寄存器情况]
    HandleMultiReg -- SSE组合 --> CombineRegs[使用insr/movlh组合值]

    SwitchSrcMCV -- register_mask --> HandleMask[处理掩码寄存器]
    HandleMask --> ExtractBits[提取位掩码到通用寄存器]

    SwitchSrcMCV -- lea_symbol/lea_direct --> HandleLEA[生成lea指令]
    SwitchSrcMCV -- air_ref --> ResolveRef[解析引用并递归处理]

    AllPaths --> End[结束]
    
    Fail --> End
    GenTypedValue --> End
    HandleEflags --> End
    MovNormal --> End
    GP2GP --> End
    SSE2SSE --> End
    FPUStack --> End
    MoveStrategy --> End
    CombineRegs --> End
    ExtractBits --> End
    HandleLEA --> End
    ResolveRef --> End

    classDef logic fill:#f9f,stroke:#333;
    classDef data fill:#f96,stroke:#333;
    classDef io fill:#6f9,stroke:#333;
    class CheckSize,SwitchSrcMCV,RegTypeSwitch logic;
    class HandleUndef,HandleEflags,HandleImmediate,HandleRegister data;
    class UseXor,Mov32,MovSignExtend,MovNormal,GP2GP,SSE2SSE,FPUStack io;
