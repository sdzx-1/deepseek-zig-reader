flowchart TD
    A[开始] --> B{检查是否有LLVM}
    B -- 否 --> C[返回错误ZigCompilerNotBuiltWithLLVMExtensions]
    B -- 是 --> D[初始化ArenaAllocator]
    D --> E[获取目标系统glibc版本]
    E --> F[计算nonshared_stat和start_old_init_fini标志]
    F --> G{判断crt_file类型}
    
    G -- .scrt1_o --> H[构建start_o参数]
    H --> I[构建abi_note_o参数]
    I --> J[添加init_o源文件]
    J --> K[定义basename为crt1/Scrt1]
    K --> L[调用comp.build_crt_file生成目标文件]
    
    G -- .libc_nonshared_a --> M[定义依赖项deps数组]
    M --> N[遍历deps过滤需要包含的项]
    N --> O[为每个dep构建编译参数]
    O --> P[添加架构特定参数（如x86的CAN_USE_REGISTER_ASM_EBP）]
    P --> Q[填充files_buf数组]
    Q --> R[调用comp.build_crt_file生成静态库]
    
    L --> S[结束]
    R --> S
    C --> S
