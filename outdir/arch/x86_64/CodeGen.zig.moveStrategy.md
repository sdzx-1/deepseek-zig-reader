graph TD
    A[开始] --> B{检查class}
    B -->|general_purpose/segment| C[返回load_store mov]
    B -->|x87| D[返回load_store_x87]
    B -->|mmx| E[空分支]
    B -->|sse| F{检查ty类型}
    F -->|非浮点/向量/指针等| G[分类ABI并检查abi_size]
    G --> H{abi_size范围}
    H -->|1字节| I[根据AVX/SSE4.1/SSE2选择插入提取策略]
    H -->|2字节| J[根据AVX/SSE4.1/SSE2选择插入提取策略]
    H -->|3-4字节| K[选择v_d/_d/_ss + mov]
    H -->|5-8字节| L[选择v_q/_q/_ps + mov]
    H -->|9-16字节| M[根据对齐选择v_dqa/dqu指令]
    H -->|17-32字节| N[AVX下选择对齐指令]
    
    F -->|浮点类型| O[检查浮点位数]
    O -->|16位| P[选择vp_w插入提取策略]
    O -->|32位| Q[选择v_ss/_ss mov]
    O -->|64位| R[选择v_sd/_sd mov]
    O -->|128位| S[根据对齐选择v_dqa/dqu指令]
    
    F -->|向量类型| T{检查子类型}
    T -->|bool向量| U[根据向量长度选择插入提取策略]
    T -->|int向量| V[根据子类型位数和长度选择对齐指令]
    T -->|指针/可选类型| W[根据长度选择对齐指令]
    T -->|float向量| X[根据子类型精度选择v_ps/v_pd指令]
    
    B -->|ip/cr/dr| Y[空分支]
    
    Z[所有未处理分支] --> AA[返回TODO错误]
    
    E --> Z
    Y --> Z
    M --> Z
    N --> Z
    S --> Z
    U --> Z
    V --> Z
    W --> Z
    X --> Z
