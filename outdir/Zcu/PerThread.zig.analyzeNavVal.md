graph TD
    A[开始] --> B[声明变量: zcu, gpa, ip, old_nav]
    B --> C[记录日志 analyzeNavVal]
    C --> D[解析指令 inst_resolved]
    D --> E{解析成功？}
    E -- 否 --> F[返回错误 AnalysisFail]
    E -- 是 --> G[获取文件信息和ZIR数据]
    G --> H[将分析单元加入进行中列表]
    H --> I[初始化分析内存池]
    I --> J[创建语义分析器 Sema]
    J --> K[声明依赖关系 declareDependency]
    K --> L[初始化代码块 Block]
    L --> M[处理ZIR声明类型 zir_decl]
    M --> N{是否有类型体？}
    N -- 是 --> O[处理类型体并获取类型]
    N -- 否 --> P[处理值体并推导类型]
    O --> Q[验证变量类型 validateVarType]
    P --> Q
    Q --> R[解析指针修饰符 alignment/linksection/addrspace]
    R --> S[确定最终nav_val]
    S --> T{是否是usingnamespace？}
    T -- 是 --> U[检查类型有效性并更新Nav]
    T -- 否 --> V[处理extern声明/变量/函数]
    U --> W[移除分析单元标记]
    V --> W
    W --> X[处理导出声明 analyzeExport]
    X --> Y[决定是否加入代码生成队列]
    Y --> Z[比较新旧Nav状态]
    Z --> AA{val是否变化？}
    AA -- 是 --> AB[返回 val_changed=true]
    AA -- 否 --> AC[返回 val_changed=false]
    
    subgraph 错误处理
        H --> H1[errdefer移除分析单元]
        I --> I1[defer释放内存池]
        J --> J1[defer释放sema资源]
        L --> L1[defer释放block指令]
        style H1 stroke:#ff6666
        style I1 stroke:#ff6666
        style J1 stroke:#ff6666
        style L1 stroke:#ff6666
    end

    subgraph 条件分支
        E
        N
        T
        AA
    end

    style A fill:#90EE90,stroke:#333
    style F fill:#FFB6C1,stroke:#333
    style AB fill:#90EE90,stroke:#333
    style AC fill:#90EE90,stroke:#333
