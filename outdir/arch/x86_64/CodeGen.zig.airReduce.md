flowchart TD
    A[开始] --> B[获取pt, zcu, reduce]
    B --> C{操作数类型是布尔向量?}
    C -->|是| D[mask_len-1可转为u6?]
    D -->|否| E[分配acc_reg, 处理内存操作]
    E --> F[计算limb_offset, 处理need_mask]
    F --> G[循环处理limb_offset]
    G --> H{操作类型?}
    H -->|Or| I[生成OR测试指令, 设置eflags为nz]
    H -->|And| J[生成AND测试指令, 设置eflags为z]
    D -->|是| K[生成mask]
    K --> L{操作类型?}
    L -->|Or| M[生成OR测试指令, 设置eflags为nz]
    L -->|And| N[分配tmp_reg, 生成AND测试指令, 设置eflags为z]
    C -->|否| O[返回未实现错误]
    I --> P[返回result]
    J --> P
    M --> P
    N --> P
    O --> P
    P --> Q[结束]
