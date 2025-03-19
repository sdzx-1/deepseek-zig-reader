graph TD
    A[开始] --> B[遍历relocs的tags, pointees, offsets, addends]
    B --> C{offset >= relocs.end?}
    C -->|是| D[跳出循环]
    C -->|否| E[计算sliced_code = code[offset - code_offset ..]]
    E --> F{根据tag选择处理分支}
    
    F -->|function_index_i32| G[调用reloc_u32_function]
    F -->|function_index_leb| H[调用reloc_leb_function]
    F -->|function_offset_i32| I[@panic TODO]
    F -->|function_offset_i64| I
    F -->|table_index_i32| J[调用reloc_u32_table_index]
    F -->|table_index_i64| K[调用reloc_u64_table_index]
    F -->|table_index_rel_sleb| I
    F -->|table_index_rel_sleb64| I
    F -->|table_index_sleb| L[调用reloc_sleb_table_index]
    F -->|table_index_sleb64| M[调用reloc_sleb64_table_index]
    
    F -->|function_import_index_i32| G
    F -->|function_import_index_leb| H
    F -->|function_import_offset_i32| I
    F -->|function_import_offset_i64| I
    F -->|table_import_index_i32| J
    F -->|table_import_index_i64| K
    F -->|table_import_index_rel_sleb| I
    F -->|table_import_index_rel_sleb64| I
    F -->|table_import_index_sleb| L
    F -->|table_import_index_sleb64| M
    
    F -->|global_index_i32| N[调用reloc_u32_global]
    F -->|global_index_leb| O[调用reloc_leb_global]
    F -->|global_import_index_i32| N
    F -->|global_import_index_leb| O
    
    F -->|memory_addr_i32| P[调用reloc_u32_addr]
    F -->|memory_addr_i64| Q[调用reloc_u64_addr]
    F -->|memory_addr_leb| R[调用reloc_leb_addr]
    F -->|memory_addr_leb64| S[调用reloc_leb64_addr]
    F -->|memory_addr_locrel_i32| I
    F -->|memory_addr_rel_sleb| I
    F -->|memory_addr_rel_sleb64| I
    F -->|memory_addr_sleb| T[调用reloc_sleb_addr]
    F -->|memory_addr_sleb64| U[调用reloc_sleb64_addr]
    F -->|memory_addr_tls_sleb| T
    F -->|memory_addr_tls_sleb64| U
    
    F -->|memory_addr_import_...| V[...类似内存地址处理分支]
    F -->|section_offset_i32| I
    
    F -->|table_number_leb| W[调用reloc_leb_table]
    F -->|table_import_number_leb| W
    
    F -->|type_index_leb| X[调用reloc_leb_type]
    
    G --> B
    H --> B
    I --> B
    J --> B
    K --> B
    L --> B
    M --> B
    N --> B
    O --> B
    P --> B
    Q --> B
    R --> B
    S --> B
    T --> B
    U --> B
    V --> B
    W --> B
    X --> B
    
    D --> Z[结束]
