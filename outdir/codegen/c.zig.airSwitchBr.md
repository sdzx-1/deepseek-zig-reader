graph TD
    Start[开始] --> InitVar[初始化变量: pt, zcu, gpa, switch_br]
    InitVar --> ResolveCond[解析初始条件 init_condition]
    ResolveCond --> Reap[回收操作数]
    Reap --> CheckDispatchLoop{是 is_dispatch_loop?}
    
    CheckDispatchLoop -- 是 --> CreateLocal[创建局部变量 new_local]
    CreateLocal --> CopyCondition[复制条件到 new_local]
    CopyCondition --> WriteLoopLabel[写入 zig_switch_loop 标签]
    WriteLoopLabel --> StoreLocal[存储 new_local 到 loop_switch_conds]
    
    CheckDispatchLoop -- 否 --> UseInitCond[直接使用 init_condition]
    
    StoreLocal --> Defer[注册 defer 清理 loop_switch_conds]
    UseInitCond --> Defer
    
    Defer --> StartSwitch[生成 switch 语句]
    StartSwitch --> LowerType{需要类型转换?}
    
    LowerType -- 是 --> AddCast[添加类型转换到条件]
    LowerType -- 否 --> WriteCondition[直接写入条件]
    
    AddCast --> WriteCondition
    WriteCondition --> OpenSwitchBlock[写入 switch { ... ]
    
    OpenSwitchBlock --> ProcessCases[遍历普通 case]
    ProcessCases --> GenCaseLabels[生成 case 标签]
    GenCaseLabels --> WriteCaseBody[生成 case 代码块]
    WriteCaseBody --> CheckNakedFunc{存在 expected_block?}
    
    CheckNakedFunc -- 是 --> FailNaked[返回错误]
    CheckNakedFunc -- 否 --> ContinueCases[继续处理下一个 case]
    
    ProcessCases --> AnyRangeCases{存在 range case?}
    AnyRangeCases -- 是 --> PrepareDefault[准备 default 分支]
    AnyRangeCases -- 否 --> ProcessElse[处理 else 分支]
    
    PrepareDefault --> ProcessRangeCases[遍历 range case]
    ProcessRangeCases --> GenIfConditions[生成 if 条件判断]
    GenIfConditions --> WriteRangeBody[生成范围 case 代码块]
    WriteRangeBody --> CheckNakedFuncRange{存在 expected_block?}
    
    CheckNakedFuncRange -- 是 --> FailNaked
    CheckNakedFuncRange -- 否 --> ContinueRanges[继续处理下一个 range case]
    
    ProcessRangeCases --> ProcessElse
    ProcessElse --> HasElseBody{else_body 存在?}
    
    HasElseBody -- 是 --> GenElseBody[生成 else 代码块]
    HasElseBody -- 否 --> WriteUnreachable[写入 zig_unreachable]
    
    GenElseBody --> CheckNakedFuncElse{存在 expected_block?}
    CheckNakedFuncElse -- 是 --> FailNaked
    CheckNakedFuncElse -- 否 --> CloseSwitch[结束 switch 块]
    
    WriteUnreachable --> CloseSwitch
    CloseSwitch --> End[结束]
    
    FailNaked --> End
