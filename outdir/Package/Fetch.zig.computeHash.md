graph TD
    A[开始] --> B[初始化: 分配器、错误包、线程池等]
    B --> C[创建all_files和deleted_files列表]
    C --> D[初始化目录遍历器walker]
    D --> E{遍历下一个条目?}
    E --> |是| F[条目类型判断]
    F --> |目录| E
    F --> |文件/符号链接| G[路径过滤检查]
    G --> |排除| H[记录删除文件并添加到deleted_files]
    H --> I[记录父目录到sus_dirs]
    I --> E
    G --> |包含| J[创建HashedFile并启动工作线程计算哈希]
    J --> K[添加到all_files]
    K --> E
    E --> |否| L[等待所有工作线程完成]
    L --> M[排序sus_dirs并按长度降序处理]
    M --> N{遍历sus_dirs}
    N --> |删除目录成功| O[添加父目录到sus_dirs]
    O --> N
    N --> |处理完成| P[排序all_files]
    P --> Q[初始化哈希计算器]
    Q --> R{遍历all_files}
    R --> |文件| S[收集哈希结果并更新总哈希]
    S --> T[累计文件大小]
    T --> R
    R --> |完成| U{遍历deleted_files检查删除结果}
    U --> |文件| V[检查删除是否成功]
    V --> U
    U --> |完成| W[存在错误?]
    W --> |是| X[返回FetchFailed错误]
    W --> |否| Y[调试模式输出哈希信息?]
    Y --> |是| Z[输出调试信息]
    Z --> AA[返回哈希结果和总大小]
    Y --> |否| AA
