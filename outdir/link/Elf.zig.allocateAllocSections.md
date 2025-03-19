flowchart TD
    A[开始] --> B[定义Align结构并初始化]
    B --> C[遍历所有节, 找出第一个TLS节并计算最大对齐]
    C --> D[初始化covers数组用于分组节]
    D --> E[遍历所有节, 按shdr标志分组到covers]
    E --> F[获取phdr_table的结束地址作为起始地址]
    F --> G[遍历每个cover]
    G --> H{cover是否为空?}
    H --> |是| G
    H --> |否| I[计算当前cover的最大对齐]
    I --> J[对齐当前地址]
    J --> K[初始化memsz和filesz]
    K --> L[遍历cover中的节]
    L --> M{是否是.tbss节?}
    M --> |是| N[调整tbss地址并跳过文件空间分配]
    M --> |否| O[对齐当前节地址]
    O --> P[更新shdr地址并累加memsz/filesz]
    P --> L
    N --> L
    L --> Q[处理完所有节?]
    Q --> |否| L
    Q --> |是| R[获取当前cover的phdr]
    R --> S{文件空间是否足够?}
    S --> |否| T[移动phdr并调整节偏移]
    S --> |是| U[更新phdr的地址和大小]
    T --> U
    U --> V[地址对齐到页大小]
    V --> G
    G --> W[处理完所有covers?]
    W --> |否| G
    W --> |是| X[结束]
