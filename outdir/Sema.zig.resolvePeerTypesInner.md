flowchart TD
    A[开始] --> B[初始化变量: pt, zcu, ip]
    B --> C[遍历peer_tys确定PeerResolveStrategy]
    C --> D{策略s是unknown?}
    D -->|是| E[设置为exact策略]
    D -->|否| F[过滤noreturn/undefined类型]
    F --> G[根据策略s分支处理]
    
    G -->|error_set| H[合并错误集类型]
    G -->|error_union| I[处理错误联合类型]
    G -->|nullable| J[检查所有类型是否为null]
    G -->|optional| K[处理可选类型]
    G -->|array| L[处理数组类型]
    G -->|vector| M[处理向量类型]
    G -->|c_ptr| N[处理C指针类型]
    G -->|ptr| O[处理通用指针类型]
    G -->|func| P[处理函数类型]
    G -->|enum_or_union| Q[处理枚举/联合类型]
    G -->|comptime_int| R[检查所有comptime_int]
    G -->|comptime_float| S[处理编译时浮点]
    G -->|fixed_int| T[处理固定整型]
    G -->|fixed_float| U[处理固定浮点]
    G -->|tuple| V[处理元组类型]
    G -->|exact| W[检查所有类型完全一致]
    
    H --> X[返回合并后的错误集]
    I --> Y[递归解析payload类型]
    J --> Z[返回null类型]
    K --> AA[递归解析子类型]
    L --> AB[处理数组长度和元素类型]
    M --> AC[处理向量长度和元素类型]
    N --> AD[处理指针对齐/地址空间]
    O --> AE[处理指针尺寸/sentinel]
    P --> AF[函数类型兼容检查]
    Q --> AG[枚举/联合类型兼容检查]
    R --> AH[返回comptime_int类型]
    S --> AI[返回comptime_float类型]
    T --> AJ[选择最大位宽整型]
    U --> AK[选择最大位宽浮点]
    V --> AL[递归处理元组字段]
    W --> AM[返回统一的具体类型]
    
    X --> AN[返回结果]
    Y --> AN
    Z --> AN
    AA --> AN
    AB --> AN
    AC --> AN
    AD --> AN
    AE --> AN
    AF --> AN
    AG --> AN
    AH --> AN
    AI --> AN
    AJ --> AN
    AK --> AN
    AL --> AN
    AM --> AN
    
    AN[返回success/conflict] --> AO[结束]
    
    style A fill:#90EE90,stroke:#006400
    style AN fill:#FFB6C1,stroke:#8B0000
    style AO fill:#D3D3D3,stroke:#808080
    classDef branch fill:#F0E68C,stroke:#A0522D
    class H,I,J,K,L,M,N,O,P,Q,R,S,T,U,V,W branch
