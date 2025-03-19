graph TD
    A[开始: readFromMemory] --> B{类型检查 ty.zigTypeTag}
    B -->|void| C[返回 Value.void]
    B -->|bool| D{buffer[0] == 0?}
    D -->|是| E[返回 Value.false]
    D -->|否| F[返回 Value.true]
    
    B -->|int/enum| G[准备整数类型]
    G --> H{bits <= 64?}
    H -->|是| I[快速路径: 直接读取并符号扩展]
    I --> J[调用 zcu.getCoerced]
    H -->|否| K[慢速路径: 大整数处理]
    K --> L[分配limb内存]
    L --> M[构建BigInt]
    M --> J
    
    B -->|float| N[读取浮点字节]
    N --> O[根据位数选择格式]
    O --> P[构造float值]
    P --> Q[返回Value.fromInterned]
    
    B -->|array| R[计算元素类型和大小]
    R --> S[分配elems数组]
    S --> T[循环读取每个元素]
    T --> U[递归调用readFromMemory]
    U --> V[填充elems]
    V --> W[构造array aggregate]
    W --> Q
    
    B -->|struct| X{布局类型?}
    X -->|extern| Y[遍历字段计算偏移]
    Y --> Z[递归读取每个字段]
    Z --> AA[构造struct aggregate]
    X -->|packed| AB[调用readFromPackedMemory]
    
    B -->|error_set| AC[读取错误索引]
    AC --> AD[查找错误名称]
    AD --> AE[构造error值]
    
    B -->|union| AF{容器布局?}
    AF -->|extern| AG[按字节数组读取]
    AG --> AH[构造union值]
    AF -->|packed| AB
    
    B -->|pointer| AI[读取usize值]
    AI --> AJ[构造pointer值]
    
    B -->|optional| AK[读取子类型值]
    AK --> AL{是否为0?}
    AL -->|是| AM[返回.none]
    AL -->|否| AN[构造optional值]
    
    B -->|其他类型| AO[返回Unimplemented错误]
    
    J --> AP[返回结果]
    Q --> AP
    AA --> AP
    AB --> AP
    AE --> AP
    AH --> AP
    AJ --> AP
    AM --> AP
    AN --> AP
    AO --> AP
    
    style A stroke:#333,stroke-width:2px
    style B stroke:#666,stroke-dasharray:5
    classDef decision fill:#f9f,stroke:#333;
    classDef process fill:#9f9,stroke:#333;
    classDef endpoint fill:#f96,stroke:#333;
    class B,H,X,AF,AL decision
    class C,E,F,J,K,N,R,S,Y,AC,AG,AI,AK process
    class AP endpoint
