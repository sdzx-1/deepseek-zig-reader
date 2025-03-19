graph TD
    Start[开始] --> CheckModifier{modifier == .always_tail?}
    CheckModifier -->|是| FailTail[返回错误: 未实现尾部调用]
    CheckModifier -->|否| GetData[获取pl_op, callee, extra, args等数据]
    GetData --> ResolveType[解析函数类型fn_ty]
    ResolveType --> SpillCompareFlags[保存比较标志]
    SpillCompareFlags --> SaveRegisters[保存调用者保存的寄存器]
    SaveRegisters --> CheckReturnByRef{返回方式是否为引用?}
    CheckReturnByRef -->|是| AllocMem[分配内存并设置r0寄存器]
    AllocMem --> LockR0[锁定r0寄存器]
    CheckReturnByRef -->|否| SkipR0[无需处理r0]
    LockR0 --> AllocStackArgs[为栈参数分配空间]
    SkipR0 --> AllocStackArgs
    AllocStackArgs --> ProcessArgs[遍历参数并生成指令]
    ProcessArgs --> CheckCalleeType{callee是已知值?}
    CheckCalleeType -->|函数/外部函数| TodoCall[生成调用指令(TODO)]
    CheckCalleeType -->|指针| SetLR[将callee地址加载到lr寄存器]
    SetLR --> CheckV5T{目标支持v5t指令集?}
    CheckV5T -->|是| GenBLX[生成blx lr指令]
    CheckV5T -->|否| FailOldArch[返回错误: 不支持旧架构]
    GenBLX --> HandleReturnValue[处理返回值]
    FailOldArch --> HandleReturnValue
    TodoCall --> HandleReturnValue
    HandleReturnValue --> CopyReturnReg{返回值寄存器未跟踪?}
    CopyReturnReg -->|是| CopyToTmp[复制到临时寄存器]
    CopyReturnReg -->|否| Finalize[结束Air指令]
    CopyToTmp --> Finalize
    Finalize --> End[结束]
    FailTail --> End
