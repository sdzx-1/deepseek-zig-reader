graph TD
    A[开始lowerConstant] --> B{val是Undef?}
    B -- 是 --> C[返回emitUndefined]
    B -- 否 --> D[进入switch val.ip_index类型]

    D --> E[类型如int_type等]
    E --> F[unreachable]

    D --> G[simple_value]
    G --> H{具体值类型}
    H -- false/true --> I[返回imm32 0或1]
    H -- 其他非运行时值 --> J[unreachable]

    D --> K[int]
    K --> L{有符号?}
    L -- 是 --> M{bits范围}
    M -- 0-32 --> N[返回imm32]
    M -- 33-64 --> O[返回imm64]
    L -- 否 --> P{bits范围}
    P -- 0-32 --> Q[返回imm32]
    P -- 33-64 --> R[返回imm64]

    D --> S[err]
    S --> T[获取错误值]
    T --> U[返回imm32]

    D --> V[error_union]
    V --> W{是否有有效负载?}
    W -- 无 --> X[递归处理错误类型]
    W -- 有 --> Y[抛出TODO错误]

    D --> Z[float]
    Z --> AA{浮点类型}
    AA -- f16 --> AB[返回imm32]
    AA -- f32 --> AC[返回float32]
    AA -- f64 --> AD[返回float64]

    D --> AE[ptr]
    AE --> AF[lowerPtr处理]

    D --> AG[opt]
    AG --> AH{optional类型表示}
    AH -- 是有效载荷 --> AI[递归处理payload]
    AH -- 空值 --> AJ[返回imm32 0]
    AH -- 其他 --> AK[返回imm32 bool值]

    D --> AL[aggregate]
    AL --> AM{具体聚合类型}
    AM -- array --> AN[抛出TODO错误]
    AM -- vector --> AO[写入内存并返回SIMD值]
    AM -- struct --> AP[处理packed结构体]
    AP --> AQ[读取内存并处理backing_int]

    D --> AR[un]
    AR --> AS{是否有tag?}
    AS -- 无 --> AT[使用unionBackingType]
    AS -- 有 --> AU[获取对应字段类型]
    AU --> AV[递归处理union值]

    D --> AW[memoized_call等]
    AW --> AX[unreachable]

    style A stroke:#333,stroke-width:2px
    style C stroke:#f66,stroke-dasharray:5
    style F stroke:#f66,stroke-dasharray:5
    style J stroke:#f66,stroke-dasharray:5
    style Y stroke:#f66
    style AN stroke:#f66
