graph TD
    Start[开始] --> ValidateParams[验证输入参数]
    ValidateParams --> GetLocal[获取本地local对象]
    GetLocal --> EnsureCapacity[确保items和extra容量足够]
    EnsureCapacity --> CreateIndices[创建临时索引(func_index, error_union_type等)]
    CreateIndices --> AddFuncDeclExtra[向extra添加FuncDecl数据]
    AddFuncDeclExtra --> AddTypeFunctionExtra[向extra添加TypeFunction数据]
    AddTypeFunctionExtra --> AppendItems[向items追加4个新条目]
    AppendItems --> SetupErrDefer[设置errdefer回滚机制]
    SetupErrDefer --> TryGetOrCreateFunc[尝试获取或创建函数声明]
    
    TryGetOrCreateFunc -->|存在| ExistingFunc[回滚items/extra长度]
    ExistingFunc --> UpdateZirBodyInst[更新zir_body_inst]
    UpdateZirBodyInst --> ReturnExisting[返回现有索引]
    
    TryGetOrCreateFunc -->|不存在| PutTentative[标记临时索引]
    PutTentative --> ProcessErrorUnion[处理错误联合类型]
    ProcessErrorUnion --> ProcessErrorSet[处理错误集合类型]
    ProcessErrorSet --> ProcessFuncType[处理函数类型]
    ProcessFuncType --> FinalizeAll[确认所有索引]
    FinalizeAll --> ReturnNew[返回新func_index]
    
    ReturnExisting --> End[结束]
    ReturnNew --> End
