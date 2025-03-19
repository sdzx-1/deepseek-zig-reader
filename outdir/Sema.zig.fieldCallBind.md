graph TD
    Start[开始] --> CheckPtrType{检查 raw_ptr 类型}
    CheckPtrType --> |是指针且大小合适| GetInnerType[获取 inner_ty]
    CheckPtrType --> |类型不符| FailType[返回类型错误]
    
    GetInnerType --> CheckDoublePtr{是否是双重指针?}
    CheckDoublePtr --> |是| GetConcreteType[获取 concrete_ty = inner_ty.childType]
    CheckDoublePtr --> |否| SetConcreteType[设置 concrete_ty = inner_ty]
    
    GetConcreteType --> FindField[开始查找字段]
    SetConcreteType --> FindField
    
    FindField --> SwitchType{根据 concrete_ty 类型分支}
    
    SwitchType --> |结构体/元组| StructCheck
    StructCheck --> |找到字段| ReturnField[返回字段信息]
    StructCheck --> |未找到字段| ContinueFind[继续查找]
    
    SwitchType --> |联合体| UnionCheck
    UnionCheck --> |找到标签字段| ReturnUnionField[返回联合体字段指针]
    UnionCheck --> |未找到标签| ContinueFind
    
    SwitchType --> |类型| TypeCheck
    TypeCheck --> LoadNamespace[加载命名空间并查找字段]
    LoadNamespace --> ReturnDirect[返回直接结果]
    
    ContinueFind --> CheckNamespace{检查命名空间声明}
    CheckNamespace --> |找到方法声明| MethodCheck[检查方法参数类型]
    CheckNamespace --> |未找到| GenerateError[生成错误信息]
    
    MethodCheck --> |参数类型匹配| ReturnMethod[返回方法调用结构]
    MethodCheck --> |参数不匹配| GenerateError
    
    GenerateError --> AddNotes[添加错误说明]
    AddNotes --> Fail[返回编译错误]
    
    ReturnField --> End[结束]
    ReturnUnionField --> End
    ReturnDirect --> End
    ReturnMethod --> End
    FailType --> End
    Fail --> End
