graph TD
    A[开始] --> B[调用 emit.lowerBranches]
    B --> C[遍历 mir_tags]
    C --> D{当前指令标签}
    
    D -->|add/sub/cmp 立即数类| E[调用 mirAddSubtractImmediate]
    D -->|asr/lsl/lsr/sdiv/udiv| F[调用 mirDataProcessing2Source]
    D -->|移位立即数指令| G[调用 mirShiftImmediate]
    D -->|条件分支 b_cond| H[调用 mirConditionalBranchImmediate]
    D -->|无条件分支 b/bl| I[调用 mirBranch]
    D -->|cbz/cbnz| J[调用 mirCompareAndBranch]
    D -->|blr/ret| K[调用 mirUnconditionalBranchRegister]
    D -->|异常指令 brk/svc| L[调用 mirExceptionGeneration]
    D -->|call_extern| M[调用 mirCallExtern]
    D -->|逻辑立即数指令| N[调用 mirLogicalImmediate]
    D -->|移位寄存器算术指令| O[调用 mirAddSubtractShiftedRegister]
    D -->|扩展寄存器算术指令| P[调用 mirAddSubtractExtendedRegister]
    D -->|条件选择指令| Q[调用 mirConditionalSelect]
    D -->|调试指令| R[调用对应调试方法]
    D -->|逻辑移位寄存器指令| S[调用 mirLogicalShiftedRegister]
    D -->|内存加载指令| T[调用 mirLoadMemoryPie]
    D -->|寄存器对操作| U[调用 mirLoadStoreRegisterPair]
    D -->|栈操作指令| V[调用 mirLoadStoreStack]
    D -->|栈参数加载| W[调用 mirLoadStackArgument]
    D -->|寄存器间加载存储| X[调用 mirLoadStoreRegisterRegister]
    D -->|立即数加载存储| Y[调用 mirLoadStoreRegisterImmediate]
    D -->|寄存器移动指令| Z[调用 mirMoveRegister]
    D -->|宽立即数移动| AA[调用 mirMoveWideImmediate]
    D -->|三源数据处理| AB[调用 mirDataProcessing3Source]
    D -->|nop| AC[调用 mirNop]
    D -->|压栈/弹栈| AD[调用 mirPushPopRegs]
    D -->|位域提取| AE[调用 mirBitfieldExtract]
    D -->|符号扩展指令| AF[调用 mirExtend]

    E --> CA[继续循环]
    F --> CA
    G --> CA
    H --> CA
    I --> CA
    J --> CA
    K --> CA
    L --> CA
    M --> CA
    N --> CA
    O --> CA
    P --> CA
    Q --> CA
    R --> CA
    S --> CA
    T --> CA
    U --> CA
    V --> CA
    W --> CA
    X --> CA
    Y --> CA
    Z --> CA
    AA --> CA
    AB --> CA
    AC --> CA
    AD --> CA
    AE --> CA
    AF --> CA

    CA -->|还有指令| C
    CA -->|遍历完成| ZA[结束]
