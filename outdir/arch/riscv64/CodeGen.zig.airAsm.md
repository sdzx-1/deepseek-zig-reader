flowchart TD
    A[开始] --> B[提取输出约束和输入]
    B --> C[初始化args和arg_map]
    C --> D[遍历outputs处理约束]
    D --> E{约束类型判断}
    E --> |'='| F[处理只写约束]
    E --> |'+'| G[处理读写约束]
    E --> |其他| H[报错]
    G --> I{检查early clobber}
    I --> |是| J[锁定寄存器]
    I --> |否| K[处理寄存器/内存约束]
    D --> L[记录到arg_map]
    L --> M[加入args列表]
    M --> N[处理输入约束]
    N --> O{约束类型判断}
    O --> |"X"| P[直接使用输入值]
    O --> |寄存器约束| Q[分配寄存器并加载]
    O --> |其他| R[报错]
    Q --> S[锁定寄存器]
    S --> T[记录到arg_map]
    T --> U[加入args列表]
    U --> V[处理clobbers]
    V --> W{是否是内存clobber}
    W --> |是| X[跳过处理]
    W --> |否| Y[分配指定寄存器]
    Y --> Z[解析汇编源代码]
    Z --> AA[逐行处理汇编指令]
    AA --> AB[处理标签定义]
    AB --> AC[处理指令/伪指令]
    AC --> AD{Mnemonic类型}
    AD --> |真实指令| AE[生成机器码]
    AD --> |伪指令| AF[特殊处理]
    AF --> AG[处理.li/.mv/.tail等]
    AG --> AH[生成对应操作]
    AE --> AI[处理操作数]
    AI --> AJ{操作数类型}
    AJ --> |寄存器| AK[生成寄存器操作]
    AJ --> |立即数| AL[生成立即数操作]
    AJ --> |符号引用| AM[生成重定位]
    AM --> AN[处理结果存储]
    AN --> AO[遍历outputs存储结果]
    AO --> AP{是否需要存储}
    AP --> |是| AQ[生成store指令]
    AP --> |否| AR[跳过]
    AR --> AS[收尾处理]
    AQ --> AS
    AS --> AT[结束]
