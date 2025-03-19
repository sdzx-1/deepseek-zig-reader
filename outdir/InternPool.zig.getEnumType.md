graph TD
    A[开始] --> B[构造Key]
    B --> C{replace_existing?}
    C -->|是| D[调用ip.putKeyReplace]
    C -->|否| E[调用ip.getOrPutKey]
    D --> F[处理gop]
    E --> F
    F --> G{gop == .existing?}
    G -->|是| H[返回.existing结果]
    G -->|否| I[获取local和items/extra]
    I --> J{ini.tag_mode?}
    J -->|.auto| K[处理auto模式]
    J -->|.explicit/.nonexhaustive| L[处理explicit/nonexhaustive模式]
    
    K --> K1[添加names_map]
    K1 --> K2[填充EnumAuto到extra]
    K2 --> K3[追加captures或type_hash]
    K3 --> K4[预留names空间]
    K4 --> M[构造WipEnumType.Result返回]
    
    L --> L1{ini.has_values?}
    L1 -->|是| L2[添加values_map]
    L1 -->|否| L3[跳过values_map]
    L2 --> L4[填充EnumExplicit到extra]
    L3 --> L4
    L4 --> L5[追加captures或type_hash]
    L5 --> L6[预留names和values空间]
    L6 --> M
    
    H --> Z[结束]
    M --> Z[结束]
    
    classDef decision fill:#f9f,stroke:#333,stroke-width:2px;
    classDef process fill:#bbf,stroke:#333,stroke-width:2px;
    classDef startend fill:#9f9,stroke:#333,stroke-width:2px;
    
    class C,J,L1 decision
    class B,D,E,F,G,I,K,K1,K2,K3,K4,L,L2,L3,L4,L5,L6,M process
    class A,H,Z startend
