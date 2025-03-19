graph TD
    A[开始] --> B[初始化变量]
    B --> C[填充type_queue]
    C --> D{主循环}
    D --> E{type_queue不为空?}
    E -- 是 --> F[取出类型ty]
    F --> G[记录已检查类型]
    G --> H{需要类型解析?}
    H -- 是 --> I[将对应AnalUnit加入unit_queue]
    H -- 否 --> J[处理联合类型的标签类型]
    J --> K[遍历命名空间中的声明]
    K --> L[将符合条件的单元加入unit_queue]
    E -- 否 --> M{unit_queue不为空?}
    M -- 是 --> N[取出单元unit]
    N --> O[记录到result]
    O --> P[处理关联单元]
    P --> Q[处理引用表和类型引用表]
    Q --> R[将新引用加入队列]
    M -- 否 --> S[结束循环]
    S --> T[返回result]
    
    style A fill:#9f9,stroke:#333,stroke-width:2px
    style T fill:#f9f,stroke:#333,stroke-width:2px
