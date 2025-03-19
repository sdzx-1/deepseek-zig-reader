graph TD
    A[开始 airAtomicRmw] --> B{是否支持原子操作?}
    B -->|是| C[处理原子操作]
    B -->|否| D[处理非原子操作]
    
    C --> E{操作类型?}
    E -->|Max/Min/Nand| F[加载当前值到value]
    F --> G[创建循环块]
    G --> H[生成cmpxchg指令]
    H --> I[比较新旧值]
    I --> J{是否相等?}
    J -->|否| G
    J -->|是| K[结束循环块]
    K --> L[返回结果]
    
    E -->|其他操作| M[生成对应原子指令]
    M --> L
    
    D --> N{操作类型?}
    N -->|Xchg| O[存储操作数到指针]
    N -->|Add/Sub/And/Or/Xor| P[计算新值并存储]
    N -->|Max/Min| Q[比较并选择值后存储]
    N -->|Nand| R[与操作后取反并存储]
    O --> S[返回结果]
    P --> S
    Q --> S
    R --> S
    
    L --> S
    S[结束并返回]
    
    style A fill:#f9f,stroke:#333
    style B fill:#bbf,stroke:#333
    style C fill:#fbb,stroke:#333
    style D fill:#fbb,stroke:#333
    style S fill:#4f4,stroke:#333
