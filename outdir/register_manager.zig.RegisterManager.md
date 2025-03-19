flowchart TD
    A[开始] --> B[初始化RegisterManager]
    B --> C[检查tracked_registers非空]
    C --> D[创建结构体实例]
    D --> E[公共方法调用]
    
    subgraph 分配寄存器流程
        E --> F[调用allocRegs/tryAllocRegs]
        F --> G{是否有足够空闲且未锁定的寄存器?}
        G -- 是 --> H[分配寄存器并标记为已用]
        G -- 否 --> I[尝试释放其他寄存器]
        I --> J{是否找到可释放的寄存器?}
        J -- 是 --> K[溢出到堆栈并重新分配]
        J -- 否 --> L[返回OutOfRegisters错误]
        H --> M[记录分配的寄存器]
        K --> M
    end
    
    subgraph 锁定寄存器流程
        E --> N[调用lockReg/lockRegAssumeUnused]
        N --> O{寄存器是否已锁定?}
        O -- 否 --> P[设置锁定标记]
        O -- 是 --> Q[返回null/断言失败]
        P --> R[返回RegisterLock令牌]
    end
    
    subgraph 释放寄存器流程
        E --> S[调用unlockReg/freeReg]
        S --> T{是否持有有效锁?}
        T -- 是 --> U[清除锁定/释放标记]
        T -- 否 --> V[忽略操作]
        U --> W[更新寄存器状态]
    end
    
    subgraph 状态查询流程
        E --> X[调用isRegFree/isRegLocked]
        X --> Y[查找寄存器索引]
        Y --> Z{寄存器存在且符合条件?}
        Z -- 是 --> AA[返回true]
        Z -- 否 --> AB[返回false]
    end
    
    M --> AC[结束]
    R --> AC
    W --> AC
    AA --> AC
    AB --> AC
    L --> AC
    Q --> AC
    V --> AC
