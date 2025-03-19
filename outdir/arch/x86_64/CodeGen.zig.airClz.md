flowchart TD
    Start[开始] --> CheckVector{是否为向量类型?}
    CheckVector -- 是 --> Error[返回错误]
    CheckVector -- 否 --> ResolveSrc[处理源操作数]
    
    ResolveSrc --> AllocDstReg[分配目标寄存器]
    AllocDstReg --> CheckBits{src_bits > 128/64?}
    
    CheckBits -- 是 --> LargeBits[处理大位数]
    LargeBits --> AllocFrame[分配帧地址]
    AllocFrame --> InitLoop[初始化循环计数器和结果]
    InitLoop --> Loop[循环检查内存块]
    Loop --> TestZero{是否为0?}
    TestZero -- 是 --> DecrementCounter[递减计数器并继续循环]
    TestZero -- 否 --> FindMSB[找到最高有效位]
    FindMSB --> CalcTotal[计算总前导零]
    
    CheckBits -- 否 --> CheckLZCNT{支持LZCNT?}
    CheckLZCNT -- 是 --> HandleLZCNT[用LZCNT指令处理]
    HandleLZCNT --> AdjustResult[调整额外位数]
    
    CheckLZCNT -- 否 --> HandleBSR[用BSR+CMOV模拟]
    HandleBSR --> CheckPowerOfTwo{src_bits是2的幂?}
    CheckPowerOfTwo -- 是 --> PowerOfTwoPath[直接异或处理]
    CheckPowerOfTwo -- 否 --> GeneralPath[通用BSR处理]
    
    AdjustResult --> Finish
    PowerOfTwoPath --> Finish
    GeneralPath --> Finish
    CalcTotal --> Finish
    
    Finish[返回结果] --> End[结束]
    
    subgraph 大位数处理
        AllocFrame
        InitLoop
        Loop
        TestZero
        DecrementCounter
        FindMSB
        CalcTotal
    end
    
    subgraph LZCNT处理
        HandleLZCNT
        AdjustResult
    end
    
    subgraph BSR模拟处理
        HandleBSR
        CheckPowerOfTwo
        PowerOfTwoPath
        GeneralPath
    end
