graph TD
    Start[开始: coerceExtra] --> CheckGenericPoison{目标类型是泛型毒药?}
    CheckGenericPoison -- 是 --> ReturnInst[直接返回inst]
    CheckGenericPoison -- 否 --> CheckSameType{源类型和目标类型相同?}
    CheckSameType -- 是 --> ReturnInst

    CheckSameType -- 否 --> ResolveValues[解析inst的值和类型]
    ResolveValues --> CheckInMemoryCoercion[检查内存强制转换是否允许]
    
    CheckInMemoryCoercion -- 允许 --> HandleMemoryCoercion[执行内存强制转换]
    HandleMemoryCoercion --> ReturnResult[返回转换结果]

    CheckInMemoryCoercion -- 不允许 --> SwitchDestType[根据目标类型分支处理]
    
    SwitchDestType -->|optional| OptionalHandling[处理可选类型]
    OptionalHandling --> CheckNullUndef{是null/undef?}
    CheckNullUndef -- 是 --> WrapOptional[包装为可选类型]
    CheckNullUndef -- 否 --> CheckPtrLike[指针类可选类型处理]
    CheckPtrLike --> RecurseChild[递归处理子类型]
    RecurseChild --> WrapOptional
    
    SwitchDestType -->|pointer| PointerHandling[处理指针类型]
    PointerHandling --> CheckFunctionPtr[函数体转函数指针]
    CheckFunctionPtr -- 是 --> CoerceFunctionPtr[强制转换函数指针]
    PointerHandling --> CheckArrayPtr[数组指针转切片]
    CheckArrayPtr -- 是 --> CoerceSlice[转换为切片]
    PointerHandling --> CheckAnyOpaque[转anyopaque指针]
    CheckAnyOpaque -- 是 --> CoerceOpaque[执行转换]
    PointerHandling --> CheckCPtr[C指针处理]
    CheckCPtr -- 成功 --> CoerceCPtr[转换C指针]

    SwitchDestType -->|int/float| NumberCoercion[数值类型转换]
    NumberCoercion --> CheckComptime[编译时数值处理]
    CheckComptime -- 成功 --> ReturnConst[返回常量]
    NumberCoercion --> RuntimeCast[运行时类型转换指令]

    SwitchDestType -->|error_union| ErrorUnionHandling[错误联合处理]
    ErrorUnionHandling --> WrapErrorSet[包装错误集]
    ErrorUnionHandling --> WrapPayload[包装有效载荷]

    SwitchDestType -->|array/vector| ArrayHandling[数组/向量处理]
    ArrayHandling --> CheckChildType[子类型兼容性检查]
    CheckChildType -- 通过 --> CoerceArray[数组强制转换]

    SwitchDestType -->|struct/tuple| StructHandling[结构体/元组处理]
    StructHandling --> CoerceTuple[元组到数组/结构体转换]

    SwitchDestType -->|enum| EnumHandling[枚举处理]
    EnumHandling --> CheckLiteral[枚举字面量匹配]
    CheckLiteral -- 匹配 --> ReturnEnumValue[返回枚举值]

    AllBranches -- 转换失败 --> GenerateError[生成错误信息]
    GenerateError --> ReturnError[返回错误]

    WrapOptional --> ReturnResult
    CoerceFunctionPtr --> ReturnResult
    CoerceSlice --> ReturnResult
    CoerceOpaque --> ReturnResult
    CoerceCPtr --> ReturnResult
    ReturnConst --> ReturnResult
    RuntimeCast --> ReturnResult
    WrapErrorSet --> ReturnResult
    WrapPayload --> ReturnResult
    CoerceArray --> ReturnResult
    CoerceTuple --> ReturnResult
    ReturnEnumValue --> ReturnResult

    Start -.->|所有路径| ReturnResult
    ReturnResult --> End[结束]
