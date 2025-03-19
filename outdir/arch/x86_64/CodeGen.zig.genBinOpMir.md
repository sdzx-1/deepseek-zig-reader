graph TD
    A[开始] --> B[初始化abi_size, spillEflags]
    B --> C{检查dst_mcv类型}
    C -->|寄存器类型| D[遍历每个寄存器]
    D --> E[确定mir_limb_tag]
    E --> F{处理src_mcv类型}
    F -->|寄存器/立即数| G[生成asmRegisterRegister或asmRegisterImmediate]
    F -->|内存/间接寻址| H[生成asmRegisterMemory]
    F -->|复杂地址| I[分配临时寄存器并递归调用genBinOpMir]
    C -->|内存类型| J[处理目标地址和源地址]
    J --> K[分块处理数据]
    K --> L[确定mir_limb_tag]
    L --> M{处理src_mcv分块}
    M -->|立即数| N[生成asmMemoryImmediate]
    M -->|寄存器/内存| O[生成asmMemoryRegister]
    M -->|复杂地址| P[分配临时寄存器并加载数据]
    G --> Q[继续下一个寄存器]
    H --> Q
    I --> Q
    Q --> R{是否处理完所有寄存器?}
    R -->|是| S[结束]
    R -->|否| D
    N --> T[继续下一个数据块]
    O --> T
    P --> T
    T --> U{是否处理完所有数据块?}
    U -->|是| S
    U -->|否| K
    C -->|其他类型| V[触发unreachable]
    V --> S
    S[结束]
