flowchart TD
    A[开始: genMulDivBinOp] --> B{dst_ty是向量或浮点?}
    B -- 是 --> C[返回未实现错误]
    B -- 否 --> D[计算dst_abi_size和src_abi_size]
    D --> E{dst_abi_size==16且src_abi_size==16?}
    E -- 是 --> F[处理128位乘法]
    F --> F1[锁定RAX/RDX寄存器]
    F1 --> F2[处理LHS/RHS内存符号加载]
    F2 --> F3[生成MOV/MUL/ADD指令序列]
    F3 --> F4[返回RAX/RDX寄存器对]
    E -- 否 --> G{检查操作标签和尺寸约束}
    G --> H[根据tag选择处理路径]
    
    H --> I1[标签是mul/mul_wrap?]
    I1 -- 是 --> J1[处理大整数乘法]
    J1 --> J1a[分配临时寄存器]
    J1a --> J1b[生成内存初始化和循环结构]
    J1b --> J1c[生成位操作和进位处理]
    J1c --> J1d[返回目标MCV]
    
    H --> I2[标签是div/rem/mod?]
    I2 -- 是 --> J2[处理除法/取模]
    J2 --> J2a{有符号?}
    J2a -- 是 --> K1[生成带符号除法逻辑]
    K1 --> K1a[处理余数修正]
    K1a --> K1b[生成SUB指令]
    J2a -- 否 --> K2[调用libcall实现无符号除法]
    
    H --> I3[处理常规尺寸运算]
    I3 --> M[锁定RAX/RDX寄存器]
    M --> N{操作类型判断}
    N --> O1[mul/div/rem等]
    O1 --> P1[生成MUL/DIV指令]
    P1 --> Q1[处理结果截断]
    Q1 --> R1[返回寄存器结果]
    
    N --> O2[mod/div_floor]
    O2 --> P2{有符号处理}
    P2 -- 是 --> Q2[生成带符号修正逻辑]
    Q2 --> R2[返回修正后结果]
    P2 -- 否 --> Q3[直接返回RDX/RAX]
    
    R1 --> S[结束]
    R2 --> S
    K2 --> S
    J1d --> S
    C --> S
