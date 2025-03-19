flowchart TD
    Start[开始] --> Init[初始化变量]
    Init --> CheckReturnType[检查返回类型comptime要求\n设置is_generic标志]
    CheckReturnType --> LoopParams[遍历所有参数]
    
    subgraph 参数处理循环
        LoopParams --> ParamCheck[检查参数]
        ParamCheck --> |检查comptime/泛型| UpdateFlags[更新comptime_bits和is_generic]
        UpdateFlags --> CheckCC[调用约定兼容性检查]
        CheckCC --> |不兼容| ErrorCC[返回错误]
        CheckCC --> |兼容| ValidateType[验证参数类型有效性]
        ValidateType --> |无效类型| ErrorType[返回错误]
        ValidateType --> |有效类型| CheckSpecialCC[特殊调用约定检查]
        CheckSpecialCC --> |参数不符合要求| ErrorSpecial[返回错误]
        CheckSpecialCC --> |参数合规| ContinueLoop[继续下一个参数]
        ContinueLoop --> LoopParams
    end
    
    LoopParams --> |参数遍历完成| HandleVarArgs[处理变长参数]
    HandleVarArgs --> |var_args为true| CheckVarArgsValid[检查generic状态和调用约定]
    CheckVarArgsValid --> |不合法| ErrorVarArgs[返回错误]
    
    HandleVarArgs --> |var_args处理完成| ProcessReturn[处理返回类型]
    ProcessReturn --> |推断错误集合| BuildIesFunc[构建推断错误函数]
    ProcessReturn --> |普通返回类型| BuildNormalFunc[构建标准函数类型]
    
    BuildIesFunc --> FinishFunc[调用finishFunc]
    BuildNormalFunc --> |有函数体| WithBody[生成带体函数声明]
    BuildNormalFunc --> |无函数体| WithoutBody[生成类型签名]
    WithBody --> FinishFunc
    WithoutBody --> FinishFunc
    
    ErrorCC --> End[结束]
    ErrorType --> End
    ErrorSpecial --> End
    ErrorVarArgs --> End
    FinishFunc --> End[结束]
