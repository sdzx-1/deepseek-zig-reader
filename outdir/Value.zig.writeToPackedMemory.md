graph TD
    A[开始] --> B[获取zcu, ip, target, endian]
    B --> C{val是未定义?}
    C -- 是 --> D[写入0到buffer并返回]
    C -- 否 --> E[switch ty的类型标签]
    E -->|void| F[无操作]
    E -->|bool| G[计算字节索引]
    G --> H{val为真?}
    H -- 是 --> I[设置对应位为1]
    H -- 否 --> J[设置对应位为0]
    E -->|int/enum| K[处理整数存储]
    K --> L{整数存储类型}
    L -->|u64/i64| M[写入普通整数]
    L -->|big_int| N[写入大整数补码]
    L -->|lazy_align| O[写入对齐值]
    L -->|lazy_size| P[写入类型大小]
    E -->|float| Q[根据位数写入浮点位模式]
    E -->|vector| R[遍历向量元素递归写入]
    R --> S[计算元素位置]
    S --> T[递归调用writeToPackedMemory]
    E -->|struct| U[遍历结构体字段递归写入]
    U --> V[获取字段值和类型]
    V --> W[递归调用writeToPackedMemory]
    E -->|union| X[处理联合体布局]
    X --> Y{存在tag?}
    Y -- 是 --> Z[写入对应字段值]
    Y -- 否 --> AA[写入backing type值]
    E -->|pointer| AB[检查指针类型并写入地址]
    AB --> AC{是整数地址?}
    AC -- 是 --> AD[写入usize值]
    AC -- 否 --> AE[返回ReinterpretDeclRef错误]
    E -->|optional| AF[解包可选类型递归写入]
    AF --> AG{有值?}
    AG -- 是 --> AH[递归写入some值]
    AG -- 否 --> AI[写入0值]
    E -->|其他类型| AJ[抛出未实现panic]
    
    style A fill:#f9f,stroke:#333
    style D fill:#f96,stroke:#333
    style F fill:#bbf,stroke:#333
    style I fill:#9f9,stroke:#333
    style J fill:#f99,stroke:#333
    style M fill:#9f9,stroke:#333
    style N fill:#9f9,stroke:#333
    style O fill:#9f9,stroke:#333
    style P fill:#9f9,stroke:#333
    style Q fill:#9f9,stroke:#333
    style T fill:#9f9,stroke:#333
    style W fill:#9f9,stroke:#333
    style Z fill:#9f9,stroke:#333
    style AA fill:#9f9,stroke:#333
    style AD fill:#9f9,stroke:#333
    style AH fill:#9f9,stroke:#333
    style AI fill:#9f9,stroke:#333
    style AJ fill:#f00,stroke:#333
