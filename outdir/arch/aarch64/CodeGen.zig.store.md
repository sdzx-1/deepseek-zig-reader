flowchart TD
    A[Start: store] --> B[计算abi_size = value_ty.abiSize]
    B --> C{switch(ptr)}
    
    C -->|immediate| D[调用setRegOrMem]
    C -->|ptr_stack_offset| E[调用genSetStack]
    C -->|register| F[锁定addr_reg]
    F --> G{switch(value)}
    
    G -->|register| H[调用genStrRegister]
    G -->|其他类型| I{abi_size <=8?}
    
    I -->|是| J[分配tmp_reg]
    J --> K[调用genSetReg]
    K --> L[递归调用store]
    
    I -->|否| M[分配4个regs]
    M --> N{switch(value)}
    
    N -->|stack_offset| O[设置src_reg为栈偏移]
    N -->|stack_argument_offset| P[生成ldr指令]
    N -->|memory| Q[设置src_reg为立即数地址]
    N -->|linker_load| R[处理符号加载]
    N -->|其他| S[报错]
    
    O --> T[设置len_reg为abi_size]
    P --> T
    Q --> T
    R --> T
    T --> U[调用genInlineMemcpy]
    
    C -->|memory/stack_offset等| V[复制到临时寄存器addr_reg]
    V --> W[递归调用store]
    
    classDef decision fill:#f9f,stroke:#333;
    classDef process fill:#bbf,stroke:#333;
    class C,G,I,N decision;
    class D,E,H,J,K,L,M,O,P,Q,R,S,T,U,V,W process;
