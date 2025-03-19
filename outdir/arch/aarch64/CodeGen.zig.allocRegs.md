flowchart TD
    Start[开始] --> Assert1{检查断言1}
    Assert1 -->|成立| Assert2{检查断言2}
    Assert2 -->|成立| AllocLocks[分配锁数组locks]
    AllocLocks --> InitLocks[初始化锁为null]
    InitLocks --> DeferUnlock[注册解锁延迟操作]
    
    DeferUnlock --> LockRead[锁定已分配的读参数寄存器]
    LockRead --> LockWrite[锁定已分配的写参数寄存器]
    
    LockWrite --> AllocRead[为未分配的读参数分配寄存器]
    AllocRead --> ReuseCheck{复用元数据存在?}
    
    ReuseCheck -->|是| TryReuse[尝试复用读参数作为目标寄存器]
    TryReuse --> Reused{找到可复用的参数?}
    Reused -->|是| MarkReused[标记复用参数, 跳过锁定]
    Reused -->|否| AllocNewWrite[为写参数分配新寄存器]
    
    ReuseCheck -->|否| AllocWrite[为写参数分配寄存器]
    
    MarkReused --> MoveNonReg
    AllocNewWrite --> MoveNonReg
    AllocWrite --> MoveNonReg
    
    MoveNonReg[移动非寄存器值到寄存器] --> End[结束并释放锁]
    
    subgraph 锁定读参数
        LockRead --> |遍历read_args| CheckReadMCV{MCV是寄存器?}
        CheckReadMCV -->|是| LockReadReg[锁定寄存器]
        CheckReadMCV -->|否| ContinueRead[继续下一个]
    end
    
    subgraph 锁定写参数
        LockWrite --> |遍历write_args| CheckWriteBind{绑定类型是reg?}
        CheckWriteBind -->|是| LockWriteReg[锁定寄存器]
        CheckWriteBind -->|否| ContinueWrite[继续下一个]
    end
    
    subgraph 分配读寄存器
        AllocRead --> |遍历read_args| ReadAllocCheck{需要分配?}
        ReadAllocCheck -->|是| AllocReadReg[分配新寄存器并锁定]
        ReadAllocCheck -->|否| NextRead[继续下一个]
    end
    
    subgraph 处理复用逻辑
        TryReuse --> |遍历read_args| CheckReuseCond[检查复用条件]
        CheckReuseCond --> |条件满足| UpdateWriteReg[更新写参数寄存器]
        CheckReuseCond --> |不满足| ContinueReuseCheck[继续检查]
    end
    
    style Start fill:#90EE90
    style End fill:#FFA07A
    style Assert1,Assert2 fill:#FFD700
    style AllocLocks,InitLocks fill:#87CEEB
    style LockReadReg,LockWriteReg fill:#FFB6C1
    style AllocReadReg,AllocNewWrite fill:#98FB98
