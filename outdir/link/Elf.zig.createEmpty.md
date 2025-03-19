flowchart TD
    A[开始] --> B[声明变量和断言检查]
    B --> C[设置use_lld和use_llvm]
    C --> D[初始化rpath_table]
    D --> E[创建Elf实例并初始化基础属性]
    E --> F[设置image_base]
    F --> G[处理LLVM对象创建]
    G --> H{是否使用LLD或LLVM?}
    H -- 是 --> I[直接返回Elf实例]
    H -- 否 --> J[创建输出文件]
    J --> K[初始化ELF结构: 文件表/段表/符号表]
    K --> L{是否为对象文件或静态库?}
    L -- 否 --> M[初始化动态段和PHDR]
    L -- 是 --> N[跳过动态段初始化]
    M --> O[处理ZCU(Zig编译单元)]
    N --> O
    O --> P[初始化ZigObject并配置符号]
    P --> Q[返回Elf实例]
    I --> Q
