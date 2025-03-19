graph TD
    Start([开始]) --> ExtractData[提取指令数据]
    ExtractData --> ParseOperands[解析左右操作数(lhs, rhs)]
    ParseOperands --> TypeCheck[类型检查与类型解析]
    TypeCheck --> ScalarTypeCheck{是标量类型?}
    
    ScalarTypeCheck --> |整数/编译时整数| IntPath[整数处理路径]
    ScalarTypeCheck --> |浮点数/编译时浮点| FloatPath[浮点数处理路径]
    
    IntPath --> CheckLHS{检查左操作数}
    CheckLHS --> |LHS是undef| FailUndefLHS[抛出undef错误]
    CheckLHS --> |LHS为零| ReturnZero[返回零值]
    CheckLHS --> |LHS有效| CheckRHS{检查右操作数}
    
    CheckRHS --> |RHS是undef| FailUndefRHS[抛出undef错误]
    CheckRHS --> |RHS为零| FailDivZero[抛出除零错误]
    CheckRHS --> |RHS为负| FailNegativeRHS[抛出负数错误]
    CheckRHS --> |RHS有效| CheckConstant{是否常量?}
    
    CheckConstant --> |是常量| ComputeRem[计算余数并返回]
    CheckConstant --> |非常量| AddRuntimeCheck[添加运行时检查]
    
    FloatPath --> CheckRHSFloat{检查右操作数}
    CheckRHSFloat --> |RHS是undef| FailUndefRHSFloat[抛出undef错误]
    CheckRHSFloat --> |RHS为零| FailDivZeroFloat[抛出除零错误]
    CheckRHSFloat --> |RHS为负| FailNegativeRHSFloat[抛出负数错误]
    CheckRHSFloat --> |RHS有效| CheckLHSFloat{检查左操作数}
    
    CheckLHSFloat --> |LHS是undef或负| FailNegativeLHS[抛出负数错误]
    CheckLHSFloat --> |LHS有效| ComputeFloatRem[计算浮点余数并返回]
    
    AddRuntimeCheck --> GenAIR[生成AIR二元操作指令]
    
    FailUndefLHS --> End
    ReturnZero --> End
    FailUndefRHS --> End
    FailDivZero --> End
    FailNegativeRHS --> End
    ComputeRem --> End
    FailUndefRHSFloat --> End
    FailDivZeroFloat --> End
    FailNegativeRHSFloat --> End
    FailNegativeLHS --> End
    ComputeFloatRem --> End
    GenAIR --> End([结束])
    
    style Start fill:#9f9,stroke:#333
    style End fill:#f9f,stroke:#333
    style FailUndefLHS fill:#f96,stroke:#333
    style FailDivZero fill:#f96,stroke:#333
    style FailNegativeRHS fill:#f96,stroke:#333
    style ComputeRem fill:#6f9,stroke:#333
    style GenAIR fill:#6f9,stroke:#333
