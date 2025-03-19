graph TD
    A[开始 genSetMem] --> B[生成 dst_ptr_mcv 根据 base 类型]
    B --> C{检查 src_mcv 类型}
    C -->|immediate| D[处理立即数]
    D --> D1{abi_size 大小}
    D1 -->|1,2,4| D2[生成 mov 指令]
    D1 -->|3,5-7| D3[Unreachachable]
    D1 -->|其他| D4[分块写入内存]
    C -->|register| E[处理寄存器]
    E --> E1[确定内存大小和对齐]
    E1 --> E2{src_size > mem_size?}
    E2 -->|是| E3[分配帧索引并写入]
    E2 -->|否| E4[直接写入寄存器]
    C -->|eflags| F[生成 setcc 指令]
    C -->|register_pair等| G[拆分类型并逐个写入]
    C -->|register_overflow| H{类型判断}
    H -->|struct| H1[写入寄存器和标志位]
    H -->|optional| H2[写入子类型和bool]
    C -->|register_offset| I[复制到临时寄存器并处理]
    C -->|memory等| J{abi_size 大小}
    J -->|0| J1[无操作]
    J -->|1-8| J2[复制到临时寄存器并写入]
    J -->|其他| J3[调用 genInlineMemcpy]
    C -->|air_ref| K[解析指令并递归调用]
    D2 --> L[结束]
    D4 --> L
    E3 --> L
    E4 --> L
    F --> L
    G --> L
    H1 --> L
    H2 --> L
    I --> L
    J1 --> L
    J2 --> L
    J3 --> L
    K --> L
