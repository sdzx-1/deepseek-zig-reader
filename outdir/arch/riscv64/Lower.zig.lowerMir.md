graph TD
    A[开始lowerMir] --> B[初始化和错误处理设置]
    B --> C{检查inst.tag}
    
    C -->|pseudo_load_rm/pseudo_store_rm| D[处理内存操作]
    D --> D1[解析FrameLoc]
    D1 --> D2{判断操作类型}
    D2 -->|load| D3[生成加载指令]
    D2 -->|store| D4[生成存储指令]
    D3/D4 --> E
    
    C -->|pseudo_mv| F[处理寄存器移动]
    F --> F1[判断源/目标寄存器类型]
    F1 -->|int/float/vector| F2[生成对应移动指令]
    F2 --> E
    
    C -->|pseudo_j| G[生成跳转指令]
    G --> E
    
    C -->|pseudo_compare| H[处理比较操作]
    H --> H1[解析操作类型和数据类型]
    H1 --> H2{数值类型}
    H2 -->|int| H3[生成整数比较逻辑]
    H2 -->|float| H4[生成浮点比较指令]
    H2 -->|vector| H5[抛出未实现错误]
    H3/H4/H5 --> E
    
    C -->|pseudo_not| I[生成逻辑非指令]
    I --> E
    
    C -->|其他标签| J[特殊处理]
    J --> J1[如pseudo_load_symbol/pseudo_spill_regs等]
    J1 --> E
    
    E[返回insts和relocs]
    
    classDef logic fill:#f9f,stroke:#333;
    classDef terminal fill:#f96,stroke:#333;
    class A,B,E terminal
    class C,D,F,G,H,I,J logic
