graph TD
    A[开始] --> B[初始化基础结构]
    B --> C[添加null输入节和重定位节]
    C --> D[初始化字符串表]
    D --> E[添加STT_FILE符号]
    E --> F{检查debug_format}
    
    F -->|strip| G[结束]
    F -->|dwarf| H[初始化Dwarf]
    
    H --> I[检查并创建.debug_str节]
    I --> J[检查并创建.debug_info节]
    J --> K[检查并创建.debug_abbrev节]
    K --> L[检查并创建.debug_aranges节]
    L --> M[检查并创建.debug_line节]
    M --> N[检查并创建.debug_line_str节]
    N --> O[检查并创建.debug_loclists节]
    O --> P[检查并创建.debug_rnglists节]
    P --> Q[检查并创建.eh_frame节]
    Q --> R[初始化Dwarf元数据]
    R --> S[结束]
    
    F -->|code_view| T[触发unreachable]
    T --> S
