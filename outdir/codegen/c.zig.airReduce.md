graph TD
    A[开始] --> B[获取pt、zcu、reduce]
    B --> C[解析操作数operand和类型operand_ty]
    C --> D[判断use_operator: scalar_ty位宽<=64?]
    D -->|是| E[根据reduce.operation选择中缀/三元操作符]
    D -->|否| F[根据reduce.operation选择内置函数]
    E --> G[初始化accum变量]
    F --> G
    G --> H[根据操作类型设置初始值]
    H --> I[进入向量循环处理]
    I --> J{操作类型判断}
    J -->|builtin| K[生成内置函数调用代码]
    J -->|infix| L[生成中缀运算符代码]
    J -->|ternary| M[生成三元运算符代码]
    K --> N[结束循环处理]
    L --> N
    M --> N
    N --> O[返回accum结果]
    O --> P[结束]
    
    style A fill:#9f9,stroke:#333
    style P fill:#f99,stroke:#333
    classDef logic fill:#e6f3ff,stroke:#3388ff
    class D,J logic
