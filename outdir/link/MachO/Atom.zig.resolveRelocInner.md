graph TD
    A[开始] --> B[初始化变量: P, A, S, G, TLS, SUB]
    B --> C[记录日志: local或extern]
    C --> D{判断rel.type}
    
    D -->|subtractor| E[空操作]
    D -->|unsigned| F[处理unsigned类型]
    D -->|got| G[处理got类型]
    D -->|branch| H[处理branch类型]
    D -->|got_load| I[处理got_load类型]
    D -->|tlv| J[处理tlv类型]
    D -->|signed*| K[处理signed相关类型]
    D -->|page*| L[处理page相关类型]
    D -->|pageoff| M[处理pageoff类型]
    D -->|got_load_pageoff| N[处理got_load_pageoff]
    D -->|tlvp_pageoff| O[处理tlvp_pageoff]
    
    F --> F1[检查meta.length]
    F1 -->|3| F2[处理64位写入]
    F1 -->|2| F3[处理32位写入]
    
    G --> G1[写入G + A - P到writer]
    
    H --> H1{判断CPU架构}
    H1 -->|x86_64| H2[写入S + A - P]
    H1 -->|aarch64| H3[处理分支指令和thunk]
    
    I --> I1{检查符号的got标志}
    I1 -->|有got| I2[写入G + A - P]
    I1 -->|无got| I3[调用x86_64.relaxGotLoad]
    
    J --> J1{检查符号的tlv_ptr}
    J1 -->|是| J2[写入S_ + A - P]
    J1 -->|否| J3[调用x86_64.relaxTlv]
    
    K --> K1[写入S + A - P]
    
    L --> L1{判断具体page类型}
    L1 -->|page| L2[计算页面数并写入指令]
    L1 -->|got_load_page| L3[计算G的页面数]
    L1 -->|tlvp_page| L4[处理tlv指针的特殊逻辑]
    
    M --> M1{判断是否是算术操作}
    M1 -->|是| M2[修改算术指令]
    M1 -->|否| M3[修改加载存储指令]
    
    N --> N1[处理got_load的页偏移]
    
    O --> O1{判断符号的tlv_ptr}
    O1 -->|是| O2[生成加载指令]
    O1 -->|否| O3[生成算术指令]
    
    E --> Z[结束]
    F2 --> Z
    F3 --> Z
    G1 --> Z
    H2 --> Z
    H3 --> Z
    I2 --> Z
    I3 --> Z
    J2 --> Z
    J3 --> Z
    K1 --> Z
    L2 --> Z
    L3 --> Z
    L4 --> Z
    M2 --> Z
    M3 --> Z
    N1 --> Z
    O2 --> Z
    O3 --> Z
