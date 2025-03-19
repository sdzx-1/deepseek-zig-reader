graph TD
    A[开始: transBinaryOperator] --> B[获取op, qt, isPointerDiffExpr]
    B --> C{switch(op)}
    C -- Assign --> D[调用transCreateNodeAssign]
    C -- Comma --> E[创建block_scope]
    E --> F[处理LHS并添加到statements]
    F --> G[处理RHS并创建break节点]
    G --> H[完成block并返回]
    C -- Div --> I{是否带符号整数?}
    I -- 是 --> J[生成@divTrunc节点]
    I -- 否 --> K[继续后续处理]
    C -- Rem --> L{是否带符号整数?}
    L -- 是 --> M[生成signed_remainder节点]
    L -- 否 --> K
    C -- Shl/Shr --> N[调用transCreateNodeShiftOp]
    C -- LAnd/LOr --> O[调用transCreateNodeBoolInfixOp]
    C -- Add/Sub --> P{是否为指针运算?}
    P -- 是 --> Q[处理指针算术]
    P -- 否 --> K
    C -- 其他操作符 --> K
    K --> R[确定op_id映射关系]
    R --> S[处理LHS/RHS表达式]
    S --> T{是否需要类型转换?}
    T -- 是 --> U[生成int_from_bool/int_from_ptr]
    T -- 否 --> V[保留原始表达式]
    V --> W[创建中缀操作节点]
    W --> X{是否指针差异表达式?}
    X -- 是 --> Y[生成位转换和除法操作]
    X -- 否 --> Z[直接返回中缀节点]
    Y --> Z
    Z --> AA[返回结果节点]
