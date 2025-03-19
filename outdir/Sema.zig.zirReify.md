graph TD
    A[开始: zirReify] --> B[初始化pt, zcu, gpa等]
    B --> C[解析操作数和跟踪指令]
    C --> D[获取类型信息类型type_info_ty]
    D --> E[解析并强制转换操作数]
    E --> F[检查未定义值]
    F --> G{switch(type_info_ty的tag_index)}
    
    G -->|.type| H[返回.type_type]
    G -->|.void| I[返回.void_type]
    G -->|.int| J[解析符号/位数-->创建整数类型]
    G -->|.vector| K[解析长度和子类型-->创建向量类型]
    G -->|.pointer| L[解析大小/对齐/地址空间等属性-->创建指针类型]
    G -->|.array| M[解析长度/子类型/sentinel-->创建数组类型]
    G -->|.struct| N[检查布局/字段-->创建结构体类型]
    G -->|.enum| O[解析标签类型/字段-->创建枚举类型]
    G -->|.opaque| P[创建不透明类型]
    G -->|.union| Q[解析布局/标签类型-->创建联合类型]
    G -->|.fn| R[解析调用约定/参数/返回类型-->创建函数类型]
    G -->|其他类型标签| S[对应处理分支...]
    
    J --> T[返回整数类型引用]
    K --> T
    L --> T
    M --> T
    N --> T
    O --> T
    P --> T
    Q --> T
    R --> T
    S --> T
    
    T --> U[结束返回Air.Ref]
    
    style A fill:#f9f,stroke:#333
    style G fill:#f96,stroke:#333
    style T fill:#bbf,stroke:#333
