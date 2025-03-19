graph TD
    A[开始 airSelect] --> B[初始化变量: pt, zcu, pl_op, extra等]
    B --> C{处理 pred_mcv 类型}
    C -->|register_mask| D[处理寄存器掩码情况]
    D --> E{检查 need_xmm0 和寄存器是否匹配}
    E -->|是| F[获取 xmm0 寄存器并设置值]
    E -->|否| G[使用现有寄存器或复制到临时寄存器]
    G --> H[确定 order 和 reuse_mcv/other_mcv]
    H --> I[分配目标寄存器 dst_mcv]
    I --> J{根据指令集选择 mir_tag}
    J --> K[生成 AVX 指令]
    J --> L[生成 SSE 指令]
    J --> M[使用逻辑指令组合]
    K & L & M --> N[生成最终结果]
    C -->|register| O[处理通用寄存器情况]
    O --> P[检查是否为 SSE 寄存器]
    P -->|是| Q[处理 bool 类型元素]
    Q --> R{need_xmm0 且寄存器非 xmm0}
    R -->|是| S[复制到 xmm0]
    R -->|否| T[直接使用或复制到临时寄存器]
    T --> U[生成 blend 指令或其他指令]
    P -->|否| V[返回错误或未实现]
    C -->|其他| W[返回错误或未实现]
    D & O & W --> X[处理 lhs 和 rhs 操作数]
    X --> Y[分配并锁定寄存器]
    Y --> Z[生成最终指令序列]
    Z --> AA[返回结果 finishAir]
    N --> AA
