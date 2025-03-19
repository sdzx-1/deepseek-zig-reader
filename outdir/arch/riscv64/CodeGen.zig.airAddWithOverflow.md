flowchart TD
    A[开始] --> B[获取类型和操作数信息]
    B --> C{检查lhs类型是否为向量}
    C -->|是| D[返回错误'TODO implement add with overflow for Vector type']
    C -->|否| E{检查是否为整数类型}
    E -->|是| F[获取整数类型信息]
    F --> G{检查位宽≥8且是2的幂?}
    G -->|是| H[生成加法结果add_result]
    H --> I[存储加法结果到内存位置]
    I --> J[将add_result复制到临时寄存器trunc_reg]
    J --> K[截断trunc_reg]
    K --> L[比较原始结果与截断结果是否不等]
    L --> M[存储溢出标志到内存]
    G -->|否| N[加载操作数到寄存器]
    N --> O[截断操作数寄存器]
    O --> P[执行加法指令]
    P --> Q[截断结果寄存器]
    Q --> R[存储加法结果到内存]
    R --> S[复制结果到临时寄存器]
    S --> T[截断并比较溢出]
    T --> M
    M --> U[返回结果result_mcv]
    E -->|否| V[触发unreachable]
    D --> W[结束]
    V --> W
    U --> W
    W --> X[结束并返回最终结果]
