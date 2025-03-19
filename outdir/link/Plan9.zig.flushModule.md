flowchart TD
    A[开始] --> B[初始检查与设置]
    B --> C{检查对象格式}
    C -- 不支持 --> D[抛出panic]
    C -- 支持 --> E[初始化追踪和资源]
    E --> F[获取comp, diags等变量]
    F --> G[启动子进度节点]
    G --> H[处理延迟符号(lazy_syms)]
    H --> I[准备GOT表和iovecs]
    I --> J[处理text段]
    J --> K[遍历函数导航表]
    K --> L[生成代码和行号信息]
    L --> M[更新offset和GOT表]
    M --> N[处理text段lazy符号]
    N --> O[更新etext符号]
    J --> P[处理data段]
    P --> Q[遍历数据导航表和匿名数据]
    Q --> R[生成数据代码]
    R --> S[更新offset和GOT表]
    S --> T[处理data段lazy符号]
    T --> U[更新edata/end符号]
    J --> V[生成符号表和行号信息]
    V --> W[构建头部信息]
    W --> X[处理重定位]
    X --> Y[遍历重定位条目]
    Y --> Z[计算目标地址并修正代码]
    Z --> AA[写入文件]
    AA --> AB[结束子进度节点]
    AB --> AC[返回结果]
    
    style A fill:#f9f,stroke:#333
    style D fill:#fbb,stroke:#f66
    style AC fill:#4f4,stroke:#0a0
    classDef logic fill:#eef,stroke:#00f;
    class C,J,P,X,Y,Z logic;
