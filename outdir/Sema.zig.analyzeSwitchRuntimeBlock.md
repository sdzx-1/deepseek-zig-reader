graph TD
    A[开始: analyzeSwitchRuntimeBlock] --> B[初始化变量和数据结构]
    B --> C[处理标量cases]
    C --> D{是否还有标量case?}
    D -- 是 --> E[解析case信息]
    E --> F[处理错误解包或分析prong]
    F --> G[收集分支提示和case数据]
    G --> D
    D -- 否 --> H[处理多cases]
    H --> I{是否还有多case?}
    I -- 是 --> J[解析items和ranges]
    J --> K[生成范围case或离散case]
    K --> L[分析prong并收集数据]
    L --> I
    I -- 否 --> M[处理特殊prong(else分支)]
    M --> N[根据类型生成补充case]
    N --> O{是否枚举类型?}
    O -- 是 --> P[生成未覆盖的枚举case]
    O -- 否 --> Q{是否错误集?}
    Q -- 是 --> R[生成未覆盖的错误case]
    Q -- 否 --> S{是否整型?}
    S -- 是 --> T[生成未覆盖的整型范围]
    S -- 否 --> U{是否布尔?}
    U -- 是 --> V[生成未覆盖的布尔值]
    U -- 否 --> W[报错不支持类型]
    N --> X[处理else主体]
    X --> Y[收集安全检查和分支提示]
    Y --> Z[组装最终SwitchBr指令]
    Z --> AA[返回生成的指令]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style Z fill:#ccf,stroke:#333,stroke-width:2px
    style AA fill:#cfc,stroke:#333,stroke-width:2px
