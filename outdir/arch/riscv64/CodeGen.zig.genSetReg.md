graph TD
    A[开始 genSetReg] --> B[参数检查: abi_size > max_size?]
    B -->|是| C[panic]
    B -->|否| D[switch src_mcv类型]
    
    D -->|undef| E[处理undef]
    E --> E1[检查sym_index]
    E1 -->|存在| E2[递归调用genSetReg]
    E1 -->|不存在| E3[根据abi_size生成立即数]
    
    D -->|immediate| F[处理立即数]
    F --> F1[检查数值范围]
    F1 -->|i12范围| F2[生成addi指令]
    F1 -->|i32范围| F3[生成lui+addi指令]
    F1 -->|超过i32| F4[使用临时寄存器生成64位序列]
    
    D -->|register| G[处理寄存器]
    G --> G1[是否相同寄存器?]
    G1 -->|是| G2[直接返回]
    G1 -->|否| G3[处理向量寄存器]
    G3 --> G4[设置vl寄存器]
    G4 --> G5[生成mv指令]
    
    D -->|load_frame| H[处理帧加载]
    H --> H1[是否向量寄存器?]
    H1 -->|是| H2[使用地址寄存器间接加载]
    H1 -->|否| H3[生成pseudo_load_rm指令]
    
    D -->|memory| I[处理内存地址]
    I --> I1[生成ld指令]
    
    D -->|lea_frame/register_offset| J[生成pseudo_lea_rm指令]
    
    D -->|indirect| K[处理间接加载]
    K --> K1[根据类型选择加载指令]
    K1 -->|float| K2[生成flw/fld]
    K1 -->|int| K3[生成lb/lh/lw/ld]
    K1 -->|vector| K4[设置vl后加载]
    
    D -->|其他类型| L[处理其他情况]
    L --> L1[递归调用或生成特殊指令]
    
    C --> M[结束]
    E2 --> M
    E3 --> M
    F2 --> M
    F3 --> M
    F4 --> M
    G2 --> M
    G5 --> M
    H2 --> M
    H3 --> M
    I1 --> M
    J --> M
    K2 --> M
    K3 --> M
    K4 --> M
    L1 --> M
