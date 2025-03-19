graph TD
    Start[开始genSetStackArgument] --> CheckMCV{检查mcv类型}
    CheckMCV -->|mcv是.dead| Unreachable[触发unreachable]
    CheckMCV -->|mcv是.none/.unreach| Return1[直接返回]
    CheckMCV -->|mcv是.undef| WantSafety{检查wantSafety}
    WantSafety -->|否| Return2[直接返回]
    WantSafety -->|是| AbiSizeUndef[根据abi_size选择立即数]
    AbiSizeUndef -->|1| Set0xAA[调用genSetStack(0xaa)]
    AbiSizeUndef -->|2| Set0xAAAA[调用genSetStack(0xaaaa)]
    AbiSizeUndef -->|4| Set0xAAAAAAAA[调用genSetStack(0xaaaaaaaa)]
    AbiSizeUndef -->|8| Set0xAAAAAAAAAAAAAAAA[调用genSetStack(0xaaaaaaaaaaaaaaaa)]
    AbiSizeUndef -->|其他| FailUndef[返回TODO错误]

    CheckMCV -->|mcv是.register| CheckAbiSizeReg{检查abi_size}
    CheckAbiSizeReg -->|1/2/4/8| SelectTag[选择存储指令tag]
    SelectTag --> CheckOffsetAlignment[检查栈偏移对齐]
    CheckOffsetAlignment -->|对齐成功| GenerateStore[生成存储指令]
    CheckOffsetAlignment -->|未对齐| FailOffset[返回TODO错误]
    CheckAbiSizeReg -->|其他| FailAbiSize[返回TODO错误]

    CheckMCV -->|mcv是.register_with_overflow| FailRWO[返回TODO错误]

    CheckMCV -->|mcv是内存相关类型| CheckAbiSizeMem{abi_size≤4?}
    CheckAbiSizeMem -->|是| CopyToTmpReg[复制到临时寄存器并递归调用]
    CheckAbiSizeMem -->|否| MemcpySetup[分配寄存器/准备memcpy]
    MemcpySetup --> SetupSrc[设置源地址]
    SetupSrc --> SetupDst[设置目标地址(sp+stack_offset)]
    SetupDst --> SetLenReg[设置长度寄存器]
    SetLenReg --> GenMemcpy[生成inline memcpy]

    CheckMCV -->|mcv是立即数/指针等| CopyToTmp[复制到临时寄存器并递归调用]
    
    Set0xAA --> Return
    Set0xAAAA --> Return
    Set0xAAAAAAAA --> Return
    Set0xAAAAAAAAAAAAAAAA --> Return
    GenerateStore --> Return
    CopyToTmpReg --> Return
    GenMemcpy --> Return
    CopyToTmp --> Return
    
    Return1[Return] --> End[结束]
    Return2[Return] --> End
    FailUndef --> End
    FailOffset --> End
    FailAbiSize --> End
    FailRWO --> End
