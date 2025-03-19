flowchart TD
    A[开始] --> B{检查 safety 和 rhs 是否为 undef?}
    B -->|是| C[结束]
    B -->|否| D[保存寄存器并锁定 rax, rdi, rsi, rcx]
    D --> E[解析目标地址 (dst) 和类型]
    E --> F[解析源值 (src_val) 和元素类型]
    F --> G{元素大小 (elem_abi_size) 是否为 1?}
    G -->|是| H[调用 genInlineMemset]
    H --> I[结束]
    G -->|否| J{指针类型是 slice 还是 one?}
    J -->|slice| K[获取指针和长度]
    K --> L[检查长度是否为 0]
    L -->|是| M[跳过复制]
    L -->|否| N[计算字节数并调用 genInlineMemcpy]
    N --> O[结束]
    J -->|one| P[确保数组长度不为 0]
    P --> Q[存储第一个元素]
    Q --> R[计算剩余元素地址]
    R --> S[调用 genInlineMemcpy]
    S --> O
