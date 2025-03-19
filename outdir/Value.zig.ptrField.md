flowchart TD
    Start[开始] --> A[获取parent_ptr类型和聚合类型]
    A --> B[检查parent_ptr大小为.one]
    B --> C{聚合类型判断}
    C -->|struct| D[struct处理分支]
    C -->|union| E[union处理分支]
    C -->|pointer| F[pointer处理分支]
    
    D --> G{struct布局类型}
    G -->|auto| H[计算字段类型和自然对齐]
    G -->|extern| I[计算字节偏移并构造指针]
    G -->|packed| J[处理packed布局的位/字节指针]
    
    E --> K{union布局类型}
    K -->|auto| L[计算字段类型和自然对齐]
    K -->|extern| M[直接类型转换]
    K -->|packed| N[处理大端对齐或位指针转换]
    
    F --> O{切片字段判断}
    O -->|ptr| P[构造指针类型和对齐]
    O -->|len| Q[构造长度类型和对齐]
    
    H & I & J & L & M & N & P & Q --> R[处理对齐和构造结果类型]
    R --> S{parent_ptr是否为undef?}
    S -->|是| T[返回undef结果]
    S -->|否| U[规范化基指针]
    U --> V[构造字段指针对象]
    V --> End[返回结果指针]
    
    classDef branch fill:#f9f,stroke:#333
    class C,G,K,O branch
