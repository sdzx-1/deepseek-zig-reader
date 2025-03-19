graph TD
    A[开始] --> B[初始化result变量]
    B --> C{检查调用约定cc}
    C -->|naked| D[断言参数为空]
    D --> E[设置return_value为unreach]
    E --> F[设置stack_byte_count=0]
    F --> G[设置stack_align=1]
    G --> H[返回result]
    
    C -->|aarch64_aapcs系列| I[初始化ncrn和nsaa]
    I --> J{检查返回类型}
    J -->|noreturn| K[设置return_value为unreach]
    J -->|无运行时位| K
    J -->|错误类型| L[设置立即数0]
    J -->|大小<=8字节| M[设置寄存器返回值]
    J -->|大小>8字节| N[返回错误]
    
    I --> O[循环处理参数]
    O --> P{参数大小是否为0}
    P -->|是| Q[设置arg为none]
    P -->|否| R[处理对齐]
    R --> S{参数是否适合寄存器}
    S -->|是| T[分配寄存器并更新ncrn]
    S -->|否| U{是否部分寄存器分配}
    U -->|是| V[返回错误]
    U -->|否| W[计算栈偏移并更新nsaa]
    O --> X[更新stack_byte_count和stack_align]
    X --> H
    
    C -->|auto| Y[处理返回值类型]
    Y --> Z{是否为noreturn}
    Z -->|是| AA[设置unreach]
    Z -->|否| AB[检查无运行时位]
    AB -->|是| AC[设置none]
    AB -->|否| AD[处理返回值大小]
    AD -->|<=8字节| AE[分配x0寄存器]
    AD -->|>8字节| AF[设置栈偏移]
    
    Y --> AG[循环处理参数]
    AG --> AH{是否有运行时位}
    AH -->|否| AI[设置arg为none]
    AH -->|是| AJ[计算栈偏移和对齐]
    AJ --> AK[更新stack_offset]
    AG --> AL[更新stack_byte_count和stack_align]
    AL --> H
    
    C -->|其他约定| AM[返回错误]
