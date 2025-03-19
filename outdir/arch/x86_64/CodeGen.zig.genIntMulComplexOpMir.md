graph TD
    A[开始: genIntMulComplexOpMir] --> B{检查dst_mcv类型}
    B --> |none, unreach, dead等| C[触发unreachable]
    B --> |register| D[确定寄存器别名和锁定]
    D --> E{检查abi_size}
    E --> |1| F[生成movzx指令扩展寄存器]
    E --> |其他| G[无需扩展]
    F --> H[处理源操作数resolved_src_mcv]
    G --> H
    H --> I{源操作数类型}
    I --> |register| J[处理寄存器源: 可能扩展并生成mul指令]
    I --> |immediate| K[检查立即数大小]
    K --> |可容纳于i32| L[生成mul立即数指令]
    K --> |过大| M[复制到临时寄存器并递归调用]
    I --> |register_offset, eflags等| N[复制到临时寄存器并生成mul指令]
    I --> |memory, indirect等| O{根据abi_size处理}
    O --> |1| P[复制到临时寄存器并扩展]
    O --> |其他| Q[直接生成内存mul指令]
    P --> R[生成mul指令]
    Q --> R
    B --> |register_pair等| S[触发unimplemented]
    B --> |memory, indirect等| T[复制到临时寄存器]
    T --> U[递归调用处理临时寄存器]
    U --> V[将结果复制回原目标]
