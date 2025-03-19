flowchart TD
    A[开始] --> B[初始化变量和动态数组]
    B --> C{遍历参数args}
    C --> D{参数以'-'开头?}
    D -- 是 --> E[处理选项]
    D -- 否 --> F[加入input_files]
    E --> G[选项类型判断]
    G -- --help --> H[输出帮助并退出]
    G -- --color --> I[读取颜色参数]
    G -- --stdin --> J[设置stdin_flag]
    G -- --check --> K[设置check_flag]
    G -- --ast-check --> L[设置check_ast_flag]
    G -- --exclude --> M[读取排除文件]
    G -- --zon --> N[设置force_zon]
    G -- 未知选项 --> O[报错退出]
    C --> P{还有参数?}
    P -- 是 --> C
    P -- 否 --> Q{有--stdin?}
    Q -- 是 --> R[检查input_files是否为空]
    R -- 非空 --> S[报错退出]
    R -- 空 --> T[读取stdin内容]
    T --> U[解析AST]
    U --> V{check_ast_flag?}
    V -- 是 --> W[生成ZIR/Zon并检查错误]
    V -- 否 --> X[检查普通错误]
    W --> Y{有错误?}
    X --> Y
    Y -- 是 --> Z[输出错误并退出]
    Y -- 否 --> AA[格式化代码]
    AA --> AB{--check?}
    AB -- 是 --> AC[比较输出与原内容]
    AC --> AD[根据结果退出]
    AB -- 否 --> AE[输出格式化结果]
    Q -- 否 --> AF{input_files为空?}
    AF -- 是 --> AG[报错退出]
    AF -- 否 --> AH[初始化Fmt结构]
    AH --> AI[处理排除文件]
    AI --> AJ[遍历输入文件]
    AJ --> AK[调用fmtPath处理]
    AK --> AL{有错误?}
    AL -- 是 --> AM[设置any_error]
    AL -- 否 --> AJ
    AJ --> AN{还有文件?}
    AN -- 是 --> AJ
    AN -- 否 --> AO{any_error?}
    AO -- 是 --> AP[退出状态1]
    AO -- 否 --> AQ[结束]
    H --> AQ
    S --> AQ
    O --> AQ
    Z --> AQ
    AD --> AQ
    AE --> AQ
    AG --> AQ
    AP --> AQ
