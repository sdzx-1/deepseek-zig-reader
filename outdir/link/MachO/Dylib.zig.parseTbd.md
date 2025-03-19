graph TD
    A[开始] --> B[记录日志]
    B --> C[加载LibStub]
    C --> D{加载成功?}
    D -- 失败 --> E[报告错误并返回]
    D -- 成功 --> F[处理umbrella库]
    F --> G[设置ID和版本]
    G --> H[初始化umbrella_libs哈希表]
    H --> I[初始化目标匹配器]
    I --> J[遍历lib_stub.inner元素]
    J --> K{元素索引>0?}
    K -- 是 --> L[添加到umbrella_libs]
    K -- 否 --> M{检查元素类型}
    L --> M
    M --> N[v3类型处理]
    N --> O[处理exports]
    O --> P[遍历符号并添加到导出]
    O --> Q[遍历弱符号并添加]
    O --> R[处理ObjC相关]
    N --> S[处理re-exports]
    S --> T{检查是否在umbrella_libs}
    T -- 否 --> U[添加依赖库]
    M --> V[v4类型处理]
    V --> W[处理exports]
    W --> X[遍历符号并添加]
    W --> Y[处理ObjC相关]
    V --> Z[处理reexports]
    Z --> AA[遍历符号并添加]
    V --> AB[处理ObjC全局条目]
    J --> AC[继续下一个元素]
    AC --> J
    J --> AD[遍历完成]
    AD --> AE[单独处理v4的reexported_libraries]
    AE --> AF[遍历每个reexport]
    AF --> AG{匹配目标?}
    AG -- 是 --> AH[遍历库]
    AH --> AI{在umbrella_libs中?}
    AI -- 否 --> AJ[添加依赖库]
    AE --> AK[结束]
