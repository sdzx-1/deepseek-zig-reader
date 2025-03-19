flowchart TD
    A[开始] --> B[检查ptr_ptr类型]
    B -->|指针类型| C[获取指针子类型ptr_ptr_child_ty]
    B -->|非指针类型| E[返回错误]
    C --> F{ptr_ptr_child_ty类型判断}
    F -->|数组类型| G[处理数组类型]
    F -->|指针类型| H{指针大小判断}
    H -->|单元素指针| I[处理单元素指针]
    H -->|many/c指针| J[处理多元素/C指针]
    H -->|切片指针| K[处理切片指针]
    F -->|其他类型| L[返回错误]
    
    I --> M[检查是否有结束索引]
    M -->|无结束索引| N[返回错误]
    M -->|有结束索引| O[验证start/end边界]
    O -->|边界无效| P[返回错误]
    O -->|边界有效| Q[构造数组类型]
    
    J --> R[加载指针值]
    J -->|C指针| S[检查是否为null]
    S -->|是null| T[返回错误]
    
    K --> U[加载切片值]
    
    G & Q & R & U --> V[获取元素类型elem_ty]
    V --> W[处理起始索引start]
    W --> X[处理结束索引end]
    X --> Y[处理哨兵值sentinel]
    Y --> Z[生成新指针new_ptr]
    Z --> AA[边界检查]
    AA --> AB{是否已知长度}
    AB -->|已知长度| AC[构造固定长度类型]
    AB -->|未知长度| AD[构造动态切片类型]
    AC & AD --> AE[添加运行时安全检查]
    AE --> AF[返回结果]
    
    classDef error fill:#f9c,stroke:#333;
    class E,N,P,T error;
