flowchart TD
    A[Start comptimeOnlyInner] --> B{ty.toIntern()?}
    B --> |empty_tuple_type| C[Return false]
    B --> |else| D[indexToKey检查]
    D --> |int_type| E[Return false]
    D --> |ptr_type| F[检查child类型]
    F --> |fn类型| G[返回!fnHasRuntimeBits]
    F --> |opaque类型| C
    F --> |其他类型| H[递归检查child_ty]
    D --> |anyframe_type| I[检查child是否为none]
    I --> |是| C
    I --> |否| J[递归检查child类型]
    D --> |array_type/vector_type/opt_type| K[递归检查子类型]
    D --> |error_union_type| L[递归检查payload_type]
    D --> |error_set_type等| M[Return false]
    D --> |func_type| N[Return true]
    D --> |simple_type| O[简单类型判断]
    O --> |数值/基础类型| C
    O --> |comptime_int等| N
    D --> |struct_type| P[处理结构体]
    P --> Q{strat类型?}
    Q --> |normal| R[根据requiresComptime返回]
    Q --> |sema| S[设置状态并检查字段]
    S --> T[遍历所有字段类型]
    T --> U[递归检查字段类型]
    U --> |存在comptime字段| V[标记并返回true]
    U --> |全部非comptime| W[标记并返回false]
    D --> |tuple_type| X[遍历元组元素]
    X --> Y[存在无值且comptime的字段?]
    Y --> |是| N
    Y --> |否| C
    D --> |union_type| Z[处理联合体]
    Z --> AA{strat类型?}
    AA --> |normal| AB[根据requiresComptime返回]
    AA --> |sema| AC[遍历字段类型]
    AC --> AD[递归检查字段类型]
    AD --> |存在comptime字段| AE[标记并返回true]
    AD --> |全部非comptime| AF[标记并返回false]
    D --> |enum_type| AG[递归检查tag_ty]
    D --> |其他类型| AH[unreachable]
    style A stroke:#333,stroke-width:2px
    style C fill:#f9f,stroke:#333
    style N fill:#9f9,stroke:#333
    style V fill:#9f9,stroke:#333
    style W fill:#f9f,stroke:#333
