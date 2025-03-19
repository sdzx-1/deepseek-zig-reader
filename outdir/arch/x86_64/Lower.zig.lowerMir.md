graph TD
    A[开始] --> B[初始化lower.result_insts和lower.result_relocs]
    B --> C[设置errdefer和defer清理]
    C --> D[获取当前指令inst]
    D --> E{检查inst.tag}
    E --> |else| F[调用lower.generic(inst)]
    E --> |.pseudo| G{检查inst.ops}
    
    G --> |.pseudo_cmov_z_and_np_rr| H[生成cmovnz和cmovnp指令]
    G --> |.pseudo_cmov_nz_or_p_rr| I[生成cmovnz和cmovp指令]
    G --> |.pseudo_cmov_nz_or_p_rm| J[生成带内存操作的cmov指令]
    G --> |.pseudo_set_z_and_np_r| K[生成setz、setnp和and指令]
    G --> |.pseudo_set_z_and_np_m| L[生成内存相关set和and指令]
    G --> |.pseudo_set_nz_or_p_r| M[生成setnz、setp和or指令]
    G --> |.pseudo_set_nz_or_p_m| N[生成内存相关set和or指令]
    G --> |.pseudo_j_z_and_np_inst| O[生成jnz和jnp跳转]
    G --> |.pseudo_j_nz_or_p_inst| P[生成jnz和jp跳转]
    G --> |.pseudo_probe_align_ri_s| Q[生成test、jz、lea、test、jmp指令链]
    G --> |.pseudo_probe_adjust_unrolled_ri_s| R[循环生成test指令并sub]
    G --> |.pseudo_probe_adjust_setup_rri_s| S[生成mov和sub指令]
    G --> |.pseudo_probe_adjust_loop_rr| T[生成test、sub、jae指令]
    G --> |.pseudo_push_reg_list| U[调用pushPopRegList]
    G --> |.pseudo_pop_reg_list| V[调用pushPopRegList]
    G --> |.pseudo_cfi_*指令| W[生成对应的.cfi指令]
    G --> |.pseudo_dbg_*指令| X[处理调试信息，无操作]
    G --> |其他| Y[空操作或断言]
    
    H --> Z
    I --> Z
    J --> Z
    K --> Z
    L --> Z
    M --> Z
    N --> Z
    O --> Z
    P --> Z
    Q --> Z
    R --> Z
    S --> Z
    T --> Z
    U --> Z
    V --> Z
    W --> Z
    X --> Z
    Y --> Z
    
    Z[返回insts和relocs切片]
    
    classDef default fill:#f9f,stroke:#333,stroke-width:2px;
    classDef condition fill:#f96,stroke:#333,stroke-width:2px;
    class E,G condition
