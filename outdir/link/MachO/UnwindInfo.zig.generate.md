flowchart TD
    A[开始] --> B[收集所有Unwind Records]
    B --> C[遍历每个Section的每个Atom]
    C --> D{Atom是否存活?}
    D -->|是| E[获取Atom的Unwind Records]
    D -->|否| C
    E --> F[过滤并添加有效Record到info.records]
    F --> C

    B --> G[编码Records]
    G --> H[遍历每个Record]
    H --> I{是否存在FDE?}
    I -->|是| J[处理FDE的LSDA和CIE]
    I -->|否| K{是否有Personality?}
    K -->|是| L[处理Personality索引]
    K -->|否| H
    J --> L
    L --> H

    G --> M[按地址排序Records]
    M --> N[折叠相邻相同编码的Records]
    N --> O[遍历并合并Records]
    O --> P{能否折叠?}
    P -->|是| Q[合并Record长度]
    P -->|否| R[保留当前Record]
    Q --> O
    R --> O

    N --> S[计算公共编码]
    S --> T[统计编码出现次数]
    T --> U[排序并选择前N个公共编码]
    U --> V[将公共编码存入info]

    S --> W[计算页分配]
    W --> X[遍历Records分页]
    X --> Y{是否Dwarf编码或可压缩?}
    Y -->|是| Z[分配压缩页]
    Y -->|否| AA[分配常规页]
    Z --> AB[更新页信息]
    AA --> AB
    AB --> X

    W --> AC[保存LSDA指针]
    AC --> AD[遍历Records并记录LSDA索引]
    AD --> AE[结束]
