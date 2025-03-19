graph TD
    Start[开始] --> CheckUnionType[检查union_ty是否为联合类型]
    CheckUnionType --> GetUnionPtrType[获取union_ptr的类型信息]
    GetUnionPtrType --> ResolveFields[解析联合的字段]
    ResolveFields --> GetFieldIndex[通过字段名获取字段索引]
    GetFieldIndex --> DetermineFieldType[确定字段类型field_ty]
    DetermineFieldType --> HandlePtrType[处理指针类型和对齐]

    HandlePtrType --> CheckInitializingNoreturn{initializing且field_ty是noreturn?}
    CheckInitializingNoreturn -->|是| CreateError[生成错误信息并返回]
    CheckInitializingNoreturn -->|否| CheckCompileTimeValue{union_ptr是编译时已知值?}

    CheckCompileTimeValue -->|是| SwitchLayout[根据布局类型处理]
    SwitchLayout -->|auto且initializing| StoreTag[存储标签并初始化联合值]
    SwitchLayout -->|auto且非initializing| CheckTagMatch[检查标签是否匹配]
    CheckTagMatch -->|不匹配| GenerateTagError[生成标签错误并返回]
    SwitchLayout -->|packed/extern| Continue

    CheckCompileTimeValue -->|否| CheckSafety{需要安全检查且自动布局?}
    CheckSafety -->|是| InsertRuntimeCheck[插入运行时标签检查]
    CheckSafety -->|否| Continue

    Continue --> CheckNoreturnField{field_ty是noreturn?}
    CheckNoreturnField -->|是| AddUnreachable[添加unreachable指令]
    CheckNoreturnField -->|否| ReturnFieldPtr[返回字段指针]

    StoreTag --> Continue
    CheckTagMatch -->|匹配| Continue
    GenerateTagError --> ReturnError[返回错误]
    CreateError --> ReturnError
    InsertRuntimeCheck --> CheckNoreturnField
    AddUnreachable --> ReturnUnreachable[返回unreachable_value]
    ReturnError --> End[结束]
    ReturnUnreachable --> End
    ReturnFieldPtr --> End

    style Start fill:#9f9,stroke:#333
    style End fill:#f9f,stroke:#333
    style CheckInitializingNoreturn fill:#f96,stroke:#333
    style CheckCompileTimeValue fill:#f96,stroke:#333
    style CheckSafety fill:#f96,stroke:#333
    style CheckNoreturnField fill:#f96,stroke:#333
    style CreateError fill:#f99,stroke:#333
    style GenerateTagError fill:#f99,stroke:#333
    style ReturnError fill:#f99,stroke:#333
