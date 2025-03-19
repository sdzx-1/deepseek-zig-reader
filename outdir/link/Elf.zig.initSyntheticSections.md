graph TD
    A[开始] --> B[获取comp, target, ptr_size, shared_objects]
    B --> C{需要.eh_frame?}
    C -->|是| D[添加.eh_frame节区]
    D --> E{comp.link_eh_frame_hdr?}
    E -->|是| F[添加.eh_frame_hdr节区]
    C -->|否| G[跳过.eh_frame]
    E -->|否| G
    G --> H{got.entries.len > 0?}
    H -->|是| I[添加.got节区]
    H -->|否| J[跳过.got]
    J --> K[检查.got_plt是否存在]
    K -->|不存在| L[添加.got.plt节区]
    K -->|存在| M[跳过.got.plt]
    M --> N{需要.rela_dyn?}
    N -->|是| O[添加.rela.dyn节区]
    N -->|否| P[跳过.rela_dyn]
    P --> Q{plt.symbols.len > 0?}
    Q -->|是| R[添加.plt和.rela.plt节区]
    Q -->|否| S[跳过.plt]
    S --> T{plt_got.symbols.len > 0?}
    T -->|是| U[添加.plt.got节区]
    T -->|否| V[跳过.plt.got]
    V --> W{copy_rel.symbols.len > 0?}
    W -->|是| X[添加.copy_rel节区]
    W -->|否| Y[跳过.copy_rel]
    Y --> Z{需要.interp节区?}
    Z -->|是| AA[添加.interp节区]
    Z -->|否| AB[跳过.interp]
    AB --> AC{是否是动态库/有共享对象/PIE?}
    AC -->|是| AD[添加.dynstrtab, .dynamic, .dynsymtab等动态节区]
    AD --> AE{需要版本节区?}
    AE -->|是| AF[添加.versym和.verneed节区]
    AE -->|否| AG[跳过版本节区]
    AC -->|否| AH[跳过动态节区]
    AG --> AI[调用initSymtab和initShStrtab]
    AH --> AI
    AI --> AJ[结束]
