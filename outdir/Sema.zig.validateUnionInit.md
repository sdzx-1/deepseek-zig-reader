flowchart TD
    Start[开始] --> CheckInstrsLen{检查instrs长度是否为1?}
    
    CheckInstrsLen -- 否 --> GenerateError[生成错误信息]
    GenerateError --> AddNotes[为额外初始化器添加错误注释]
    AddNotes --> Fail[返回错误消息]
    
    CheckInstrsLen -- 是 --> CheckComptime{块在编译时且指针已定义?}
    
    CheckComptime -- 是 --> Return[直接返回]
    
    CheckComptime -- 否 --> ParseField[解析字段指针和名称]
    ParseField --> GetFieldIndex[获取字段索引]
    GetFieldIndex --> AnalyzeAIR[分析AIR指令]
    
    AnalyzeAIR --> LoopBlock[循环检查block中的指令]
    LoopBlock --> FindStore{找到store指令?}
    
    FindStore -- 是 --> CheckComptimeValue{初始化值是编译时已知?}
    FindStore -- 否 --> ContinueLoop[继续循环]
    
    CheckComptimeValue -- 是 --> ShrinkBlock[删除冗余指令]
    ShrinkBlock --> StoreComptime[插入编译时union存储]
    StoreComptime --> End[结束]
    
    CheckComptimeValue -- 否 --> CheckComptimeOnly{联合体需编译时初始化?}
    CheckComptimeOnly -- 是 --> FailComptime[返回编译时错误]
    CheckComptimeOnly -- 否 --> ValidateRuntime[验证运行时值]
    
    ValidateRuntime --> SetTag[设置联合体标签指令]
    SetTag --> End
