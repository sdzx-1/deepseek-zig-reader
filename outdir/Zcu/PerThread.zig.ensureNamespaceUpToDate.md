flowchart TD
    A[开始] --> B{namespace.generation == zcu.generation?}
    B -- 是 --> C[返回]
    B -- 否 --> D[确定Container类型和full_key]
    D --> E{full_key类型?}
    E -- reified/generated_tag --> F[更新generation并返回]
    E -- declared --> G[解析key.declared]
    G --> H[获取inst_info]
    H --> I{解析成功?}
    I -- 否 --> J[返回AnalysisFail错误]
    I -- 是 --> K[获取文件及ZIR数据]
    K --> L{Container类型}
    L -- struct --> M[解析struct_decl的ZIR数据]
    L -- union --> N[解析union_decl的ZIR数据]
    L -- enum --> O[解析enum_decl的ZIR数据]
    L -- opaque --> P[解析opaque_decl的ZIR数据]
    M & N & O & P --> Q[计算decls的位置和长度]
    Q --> R[调用scanNamespace处理decls]
    R --> S[更新namespace.generation]
    S --> C
