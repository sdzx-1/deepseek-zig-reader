graph TD
    Start[开始] --> CheckVector{src_ty是向量类型?}
    CheckVector -- 是 --> Fail[返回错误'TODO implement genByteSwap']
    CheckVector -- 否 --> LockSrcReg[根据src_mcv类型锁定寄存器]
    LockSrcReg --> AnalyzeAbiSize[根据abi_size进入不同分支]

    AnalyzeAbiSize --> Case1{abi_size == 1}
    Case1 -- 是 --> ReuseCheck1[检查是否重用操作数]
    ReuseCheck1 -- 是 --> ReturnSrcMCV1[直接返回src_mcv]
    ReuseCheck1 -- 否 --> CopyToReg1[复制到新寄存器并返回]

    AnalyzeAbiSize --> Case2{abi_size == 2}
    Case2 -- 是 --> ReuseCheck2[检查是否重用操作数]
    ReuseCheck2 -- 是 --> GenShift[生成ROL指令并返回src_mcv]
    ReuseCheck2 -- 否 --> CopyToReg2[复制到新寄存器并生成指令]

    AnalyzeAbiSize --> Case3_8{3 ≤ abi_size ≤ 8}
    Case3_8 -- 是 --> ReuseCheck3[检查是否寄存器操作数]
    ReuseCheck3 -- 是 --> GenBswap[生成BSWAP指令并返回]
    ReuseCheck3 -- 否 --> CopyToReg3[复制到新寄存器处理]

    AnalyzeAbiSize --> Case9_16{9 ≤ abi_size ≤ 16}
    Case9_16 -- 是 --> HandleComplex[处理多寄存器/内存情况]
    HandleComplex --> AllocRegs[分配两个目标寄存器]
    AllocRegs --> LoadSwap[加载并交换寄存器对]
    LoadSwap --> ReturnPair[返回交换后的寄存器对]

    AnalyzeAbiSize --> CaseLarge{abi_size >16}
    CaseLarge -- 是 --> BigTypeLoop[初始化循环计数器]
    BigTypeLoop --> LoopBody[生成内存交换指令]
    LoopBody --> UpdateCounters[更新索引计数器]
    UpdateCounters --> CheckLoop[检查循环条件]
    CheckLoop -- 继续循环 --> LoopBody
    CheckLoop -- 结束 --> ReturnDstMCV[返回最终结果]

    AnalyzeAbiSize --> DefaultCase[默认处理]
    DefaultCase --> AllocDst[分配目标寄存器/内存]
    AllocDst --> GenMovbe[根据特性生成MOVBE或常规指令]
    GenMovbe --> FinalReturn[返回结果]

    ReturnSrcMCV1 --> UnlockSrc[解锁源寄存器]
    GenShift --> UnlockSrc
    GenBswap --> UnlockSrc
    ReturnPair --> UnlockSrc
    ReturnDstMCV --> UnlockSrc
    FinalReturn --> UnlockSrc

    UnlockSrc --> End[结束]
