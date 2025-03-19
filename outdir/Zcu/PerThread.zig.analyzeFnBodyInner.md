graph TD
    A[开始 analyzeFnBodyInner] --> B[获取上下文: zcu, gpa, ip]
    B --> C[创建分析单元 AnalUnit]
    C --> D[将分析单元加入 analysis_in_progress]
    D --> E[设置函数为已分析状态]
    E --> F{是否有推断错误集?}
    F --> |是| G[设置 resolved_error_set 为 none]
    F --> |否| H[继续]
    G --> H
    H --> I[初始化 Sema 结构体]
    I --> J[声明依赖项]
    J --> K{是否是泛型函数实例?}
    K --> |是| L[预填充编译时参数]
    K --> |否| M[处理运行时参数]
    L --> M
    M --> N[生成参数对应的 arg 指令]
    N --> O[分析保存错误返回跟踪]
    O --> P[分析函数体 sema.analyzeFnBody]
    P --> Q{是否有未解决的推断分配?}
    Q --> |是| R[设置为默认 alloc 指令]
    Q --> |否| S[设置分支提示]
    R --> S
    S --> T{是否启用错误跟踪?}
    T --> |是| U[设置错误返回跟踪]
    T --> |否| V[继续]
    U --> V
    V --> W[生成主块 main_block]
    W --> X{是否有推断错误集需要解析?}
    X --> |是| Y[解析并设置 resolved_error_set]
    X --> |否| Z[继续]
    Y --> Z
    Z --> AA[解析函数类型]
    AA --> AB[导出处理 flushExports]
    AB --> AC[返回 Air 结果]
    AC --> AD[结束]
    
    style A fill:#f9f,stroke:#333
    style AD fill:#f9f,stroke:#333
