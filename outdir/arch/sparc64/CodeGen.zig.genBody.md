flowchart TD
    Start[开始] --> Loop[循环遍历body中的每个指令inst]
    Loop --> CheckUnused{检查isUnused且不需要生成？}
    CheckUnused -- 是 --> Loop
    CheckUnused -- 否 --> SaveState[保存旧的air_bookkeeping状态]
    SaveState --> EnsureCapacity[确保ProcessDeath容量]
    EnsureCapacity --> ResetReusedOperands[重置reused_operands]
    ResetReusedOperands --> Switch[根据air_tags分发指令类型]
    Switch --> HandleCase[调用对应的处理方法]
    HandleCase --> AssertionCheck{断言检查寄存器管理器和簿记}
    AssertionCheck -- 通过 --> Loop
    AssertionCheck -- 失败 --> Panic[触发panic]
    Loop --> End[结束]
