flowchart TD
    A[开始] --> B[构造Key]
    B --> C{replace_existing?}
    C -->|是| D[调用putKeyReplace]
    C -->|否| E[调用getOrPutKey]
    D --> F{检查结果}
    E --> F
    F -->|existing| G[返回existing结果]
    F -->|put| H[获取local items和extra]
    H --> I{处理ini.layout}
    I -->|packed| J[填充packed结构元数据]
    J --> K{是否有captures?}
    K -->|是| L[添加captures到extra]
    K -->|否| M[跳过]
    M --> N[填充字段占位符]
    N --> O[返回wip结果]
    I -->|extern或auto| P[填充普通结构元数据]
    P --> Q{是否有captures?}
    Q -->|是| R[添加captures到extra]
    Q -->|否| S[跳过]
    S --> T[填充字段、对齐等信息]
    T --> U[返回wip结果]
