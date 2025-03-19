graph TD
    Start[开始] --> Init[初始化: 获取switch_br, target_ty]
    Init --> Assert[断言: target_ty有运行时表示]
    Assert --> CheckDispatchLoop{is_dispatch_loop?}
    
    CheckDispatchLoop -->|是| HandleDispatchLoop[处理dispatch循环]
    HandleDispatchLoop --> AllocLocal[分配本地变量target]
    AllocLocal --> LowerToStack[将initial_target下放到栈]
    LowerToStack --> StartLoop[启动loop块]
    StartLoop --> StoreBlock[存储块信息到cg.blocks]
    
    CheckDispatchLoop -->|否| ResolveTarget[直接解析target]
    
    HandleDispatchLoop --> MergeTarget
    ResolveTarget --> MergeTarget
    MergeTarget --> Liveness[处理liveness分析]
    
    Liveness --> CheckEmptyCase{case数量为0?}
    CheckEmptyCase -->|是| HandleElseOnly[直接处理else分支]
    HandleElseOnly --> GenElseBody[生成else body代码]
    GenElseBody --> EndDispatchLoop{is_dispatch_loop?}
    EndDispatchLoop -->|是| EndLoopBlock[结束loop块]
    EndLoopBlock --> FinishAir[完成Air指令]
    
    CheckEmptyCase -->|否| CalcMinMax[计算min/max和branching_size]
    CalcMinMax --> CheckBrTable{满足br_table条件?}
    
    CheckBrTable -->|是| GenBrTable[生成br_table指令]
    GenBrTable --> FillJumpTable[填充跳转表]
    
    CheckBrTable -->|否| GenBrIf[生成br_if指令链]
    GenBrIf --> AddFinalBr[添加最终br指令]
    
    GenBrTable --> ProcessCases[处理所有case分支]
    GenBrIf --> ProcessCases
    
    ProcessCases --> IterateCases[遍历case生成代码块]
    IterateCases --> EndCaseBlock[结束当前case块]
    EndCaseBlock --> GenCaseBody[生成case body代码]
    GenCaseBody --> AddBrExit[添加退出分支]
    
    ProcessCases --> CheckElse{存在else分支?}
    CheckElse -->|是| GenElse[生成else分支]
    GenElse --> GenElseBody
    CheckElse -->|否| AddUnreachable[插入unreachable]
    
    GenElse --> FinishSwitchBlock[结束整个switch块]
    AddUnreachable --> FinishSwitchBlock
    
    FinishSwitchBlock --> FinalCheckDispatchLoop{is_dispatch_loop?}
    FinalCheckDispatchLoop -->|是| EndDispatchLoopBlock[结束dispatch循环块]
    EndDispatchLoopBlock --> FinishAir
    
    FinalCheckDispatchLoop -->|否| FinishAir
    
    FinishAir --> End[结束]
    
    style Start fill:#90EE90,stroke:#333
    style End fill:#FFA07A,stroke:#333
    style CheckDispatchLoop fill:#FFD700,stroke:#333
    style CheckBrTable fill:#FFD700,stroke:#333
    style CheckEmptyCase fill:#FFD700,stroke:#333
    style CheckElse fill:#FFD700,stroke:#333
    style FinalCheckDispatchLoop fill:#FFD700,stroke:#333
