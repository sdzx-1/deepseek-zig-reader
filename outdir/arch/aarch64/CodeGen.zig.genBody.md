graph TD
    Start[开始] --> Initialize[初始化变量: pt, zcu, ip, air_tags]
    Initialize --> Loop[循环遍历body中的每个指令inst]
    Loop --> CheckUnused{检查inst是否未被使用且不需处理?}
    CheckUnused -- 是 --> Loop
    CheckUnused -- 否 --> SaveOldState[保存旧的air_bookkeeping]
    SaveOldState --> EnsureCapacity[确保ProcessDeath容量]
    EnsureCapacity --> InitReusedOperands[初始化reused_operands]
    InitReusedOperands --> Switch[根据air_tags选择处理方式]
    Switch -->|调用对应处理方法| HandleInstruction[处理指令]
    HandleInstruction --> CheckRegisters{断言: 无锁定寄存器存在?}
    CheckRegisters -- 通过 --> RuntimeSafetyCheck{运行时安全检查开启?}
    RuntimeSafetyCheck -- 是 --> BookkeepingCheck{air_bookkeeping是否正确更新?}
    BookkeepingCheck -- 否 --> Panic[触发panic]
    BookkeepingCheck -- 是 --> Loop
    RuntimeSafetyCheck -- 否 --> Loop
    Loop --> LoopEnd[循环结束]
    LoopEnd --> End[函数结束]
    
    HandleInstruction -->|遇到未实现指令| TodoError[返回TODO错误]
    TodoError --> Loop
    Panic --> End
