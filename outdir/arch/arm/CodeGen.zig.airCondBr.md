graph TD
    A[开始 airCondBr] --> B[解析条件指令和分支信息]
    B --> C{条件操作数是否死亡?}
    C -->|是| D[处理操作数死亡]
    C -->|否| E[保存父分支状态]
    D --> E
    E --> F[生成then分支代码]
    F --> G[恢复父分支状态]
    G --> H[生成else分支代码]
    H --> I[合并分支状态]
    I --> J[处理寄存器分配冲突]
    J --> K[清理临时分支数据]
    K --> L[结束并返回结果]

    subgraph 分支处理
        F --> F1[处理then分支的死亡指令]
        F1 --> F2[生成then主体代码]
        H --> H1[处理else分支的死亡指令]
        H1 --> H2[生成else主体代码]
    end

    subgraph 状态管理
        E --> E1[保存寄存器状态]
        E --> E2[保存栈状态]
        E --> E3[保存CPSR标志]
        G --> G1[恢复寄存器]
        G --> G2[恢复栈]
        G --> G3[恢复CPSR]
    end

    subgraph 状态合并
        I --> I1[遍历else条目]
        I --> I2[遍历then条目]
        I1 --> I3[对比并统一MCV值]
        I2 --> I4[更新父分支状态表]
    end

    L --> M[返回.unreach操作数]
    style A fill:#c0ffc0,stroke:#333
    style L fill:#ffc0c0,stroke:#333
