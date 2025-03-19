graph TD
    A[开始] --> B[检查魔数是否正确]
    B --> |错误| C[返回错误BadObjectMagic]
    B --> |正确| D[读取版本号]
    D --> E[初始化各段起始索引]
    E --> F[进入段处理循环]
    
    F --> G{是否还有未处理段?}
    G --> |是| H[读取段类型和长度]
    H --> I{段类型判断}
    
    I --> |custom段| J[处理自定义段]
    J --> K{子段类型判断}
    K --> |linking| L[处理符号表/初始化函数/COMDAT]
    K --> |reloc| M[处理重定位信息]
    K --> |target_features| N[解析目标特性]
    K --> |.debug| O[处理调试信息]
    K --> |其他| P[跳过处理]
    
    I --> |type段| Q[处理函数类型]
    I --> |import段| R[处理导入项]
    I --> |function段| S[记录函数类型索引]
    I --> |table段| T[处理表定义]
    I --> |memory段| U[处理内存定义]
    I --> |global段| V[处理全局变量]
    I --> |export段| W[处理导出项]
    I --> |code段| X[处理代码段]
    I --> |data段| Y[处理数据段]
    I --> |其他段| Z[跳过处理]
    
    Z --> AA[检查段处理完整性]
    L --> AA
    M --> AA
    N --> AA
    O --> AA
    P --> AA
    Q --> AA
    R --> AA
    S --> AA
    T --> AA
    U --> AA
    V --> AA
    W --> AA
    X --> AA
    Y --> AA
    
    AA --> F
    
    G --> |否| AB[后续处理]
    AB --> AC[检查必须的linking段]
    AC --> |缺失| AD[返回错误MissingLinkingSection]
    AC --> |存在| AE[处理特性兼容性检查]
    AE --> AF[应用符号表信息]
    AF --> AG[处理初始化函数]
    AG --> AH[处理遗留间接函数表]
    AH --> AI[验证代码段存在性]
    AI --> AJ[构建返回对象]
    AJ --> AK[结束]
    
    style A fill:#90EE90,stroke:#006400
    style C fill:#FFB6C1,stroke:#8B0000
    style AD fill:#FFB6C1,stroke:#8B0000
    style AK fill:#90EE90,stroke:#006400
