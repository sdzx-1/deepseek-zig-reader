flowchart TD
    A[开始: genShiftBinOpMir] --> B{abi_size > 16?}
    B -->|是| C[锁定RCX寄存器(如果需要)]
    C --> D[分配4个临时寄存器]
    D --> E{检查tag类型}
    E -->|左移| F[设置临时寄存器并处理移位]
    E -->|右移| G[设置临时寄存器并处理移位]
    F --> H[生成循环处理多字节移位]
    G --> H
    H --> I{移位量是否在循环范围内?}
    I -->|是| J[生成循环体指令]
    J --> K[更新索引寄存器]
    K --> I
    I -->|否| L[处理剩余移位操作]
    L --> M[释放临时寄存器和RCX锁]
    M --> N[返回]
    B -->|否| O{abi_size > 8?}
    O -->|是| P{检查操作数类型}
    P -->|寄存器对| Q[处理双寄存器移位]
    P -->|内存加载| R[处理内存操作数移位]
    Q --> S[生成双寄存器移位指令]
    R --> S
    S --> N
    O -->|否| T[处理小尺寸操作数]
    T --> U{检查操作数位置}
    U -->|寄存器| V[直接生成寄存器移位指令]
    U -->|内存/间接| W[生成内存操作移位指令]
    V --> N
    W --> N
    style A stroke:#333,stroke-width:2px
    style B stroke:#f66,stroke-width:2px
    style O stroke:#f66,stroke-width:2px
    style E stroke:#66f,stroke-width:2px
    classDef condition fill:#f9f,stroke:#333;
    class B,O,E condition
