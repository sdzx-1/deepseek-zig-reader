graph TD
    Start([开始]) --> CheckComptimeKnown{操作数是编译时已知?}
    CheckComptimeKnown -- 是 --> Coerce[强制转换类型并返回]
    CheckComptimeKnown -- 否 --> CheckDestComptimeInt{目标类型是comptime_int?}
    CheckDestComptimeInt -- 是 --> Fail[报错: 无法转换运行时值]
    CheckDestComptimeInt -- 否 --> CheckVectorizable[检查向量化操作数]
    CheckVectorizable --> CheckSingleValue{目标类型只有一个可能值?}
    CheckSingleValue -- 是 --> HandleSingleValue[处理单值类型]
    HandleSingleValue --> CheckRuntimeSafety{运行时安全检查开启?}
    CheckRuntimeSafety -- 是 --> CheckZero[添加零值检查和安全措施]
    CheckRuntimeSafety -- 否 --> ReturnOPV[返回唯一可能值]
    CheckZero --> AddSafetyCheck[添加安全性检查]
    AddSafetyCheck --> ReturnOPV
    CheckSingleValue -- 否 --> RequireRuntime[进入运行时处理]
    RequireRuntime --> CheckBackendSupport{后端支持安全指令?}
    CheckBackendSupport -- 是 --> IntCastSafe[生成intcast_safe指令]
    CheckBackendSupport -- 否 --> AnalyzeIntRanges[分析整型范围]
    AnalyzeIntRanges --> CheckRangeShrinkage{目标范围是否更小?}
    CheckRangeShrinkage -- 是 --> HandleSignedRange[处理有符号范围]
    CheckRangeShrinkage -- 否 --> CheckSignLoss{是否符号丢失?}
    HandleSignedRange --> AddTruncationCheck[添加截断检查]
    AddTruncationCheck --> AddSafetySignCheck[添加符号安全性检查]
    CheckSignLoss -- 是 --> HandleSignLoss[处理符号丢失]
    HandleSignLoss --> AddSignCheck[添加非负检查]
    AddSignCheck --> AddSafetyNegativeCheck[添加负值安全性检查]
    AnalyzeIntRanges --> IntCast[生成intcast指令]
    AddSafetySignCheck --> IntCast
    AddSafetyNegativeCheck --> IntCast
    Coerce --> End([结束])
    Fail --> End
    ReturnOPV --> End
    IntCastSafe --> End
    IntCast --> End
