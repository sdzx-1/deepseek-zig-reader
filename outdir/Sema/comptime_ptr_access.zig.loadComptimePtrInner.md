graph TD
    Start([开始]) --> A[处理ptr_val]
    A --> B{检查ptr.base_addr类型}
    B -->|nav| C[解析nav并获取值]
    C --> D{检查值的类型}
    D -->|variable| E[返回runtime_load]
    D -->|extern函数| F[继续处理]
    D -->|其他| F
    B -->|comptime_alloc| G[获取comptime_alloc值]
    B -->|uav/comptime_field| H[直接获取值]
    B -->|int| I[返回runtime_load]
    B -->|eu_payload| J[递归加载基指针]
    J --> K{检查错误联合结果}
    K -->|undef| L[返回undef]
    K -->|err| M[返回err_payload]
    K -->|payload| N[继续处理]
    B -->|opt_payload| O[递归加载基指针]
    O --> P{检查可选结果}
    P -->|undef| Q[返回undef]
    P -->|null| R[返回null_payload]
    P -->|payload| S[继续处理]
    B -->|arr_elem| T[处理数组元素]
    T --> U[计算加载类型和数量]
    U --> V[递归加载数组基指针]
    V --> W[返回结果或错误]
    B -->|field| X[处理结构体/联合体字段]
    X --> Y[递归加载基指针]
    Y --> Z{检查聚合类型}
    Z -->|struct/pointer| AA[获取字段值]
    Z -->|union| AB[检查标签匹配]
    AB -->|标签不匹配| AC[返回inactive_union_field]
    AB -->|标签匹配| AD[获取payload]
    AA & AD --> AE[继续处理]
    
    F & G & H & N & S & W & AE --> AF{检查偏移和类型}
    AF -->|无偏移且类型匹配| AG[返回base_val]
    AF -->|需要重构数组| AH[处理数组重构]
    AH --> AI[检查元素类型和数量]
    AI --> AJ[展开数组元素]
    AJ --> AK[重组数组]
    AK --> AL[返回结果或错误]
    
    AF -->|需要内存重解释| AM[检查布局合法性]
    AM -->|布局非法| AN[返回needed_well_defined]
    AM -->|布局合法| AO[处理内存偏移]
    AO --> AP[遍历子元素调整偏移]
    AP -->|找到合适子元素| AQ[更新当前值和偏移]
    AP -->|无法分解| AR[执行位转换]
    AQ --> AP
    AR --> AS[生成结果值]
    AS --> AT[返回success]
    
    style Start fill:#90EE90,stroke:#006400
    style E fill:#FFB6C1,stroke:#8B0000
    style L fill:#FFB6C1,stroke:#8B0000
    style M fill:#FFB6C1,stroke:#8B0000
    style Q fill:#FFB6C1,stroke:#8B0000
    style R fill:#FFB6C1,stroke:#8B0000
    style AC fill:#FFB6C1,stroke:#8B0000
    style AN fill:#FFB6C1,stroke:#8B0000
    style AT fill:#87CEFA,stroke:#00008B
