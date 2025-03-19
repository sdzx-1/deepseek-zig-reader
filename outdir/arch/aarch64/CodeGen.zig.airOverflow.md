graph TD
    A[开始 airOverflow] --> B[获取指令标签tag, ty_pl, extra等数据]
    B --> C{指令是否未被使用?}
    C -->|是| D[结果设为.dead]
    C -->|否| E[解析lhs/rhs绑定和类型]
    E --> F{类型是否为向量?}
    F -->|是| G[报错: TODO向量溢出操作]
    F -->|否| H[获取整数类型信息int_info]
    H --> I{整数位数范围?}
    I -->|1-31/33-63| J[分配栈空间并处理溢出位]
    J --> J1[释放比较标志寄存器]
    J1 --> J2[生成加减操作dest_reg]
    J2 --> J3[截断寄存器并比较]
    J3 --> J4[设置栈上的结果和溢出位]
    J4 --> K[返回栈偏移结果]
    I -->|32/64| L[尝试解析立即数]
    L --> M{立即数是否符合条件?}
    M -->|是| N[生成立即数操作指令]
    M -->|否| O[生成寄存器操作指令]
    N & O --> P[设置溢出标志]
    P --> Q[返回寄存器+标志结果]
    I -->|其他位数| R[报错:TODO大整数溢出操作]
    D & G & K & Q & R --> S[调用finishAir返回结果]
    S --> T[结束]
    
    style A fill:#f9f,stroke:#333
    style S fill:#bbf,stroke:#333
    style T fill:#f9f,stroke:#333
    style G fill:#fbb,stroke:#333
    style R fill:#fbb,stroke:#333
