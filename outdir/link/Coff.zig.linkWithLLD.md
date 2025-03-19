flowchart TD
    A[开始] --> B[检查LLD链接器]
    B --> C[初始化跟踪和资源]
    C --> D[获取输出路径和目录]
    D --> E{是否有Zig代码需要编译?}
    E --> |是| F[刷新模块对象并获取路径]
    E --> |否| G[模块对象路径设为空]
    F --> G
    G --> H[创建子进度节点]
    H --> I{输出模式是否为Obj?}
    I --> |是| J[直接复制对象文件]
    I --> |否| K[构建LLD命令行参数]
    K --> L[添加基础参数和调试信息]
    L --> M[处理版本和ABI设置]
    M --> N[处理栈大小和基地址]
    N --> O[设置目标架构参数]
    O --> P[处理强制符号和动态库标志]
    P --> Q[处理入口点名称]
    Q --> R[添加安全性和兼容性标志]
    R --> S[设置输出路径和导入库]
    S --> T[处理LibC路径和库目录]
    T --> U[添加链接输入和对象文件]
    U --> V[处理子系统设置]
    V --> W{子系统模式是UEFI还是Win32?}
    W --> |UEFI| X[添加UEFI特定参数]
    W --> |Win32| Y[处理CRT链接和默认库]
    X --> Z[添加依赖库和编译器运行时]
    Y --> Z
    Z --> AA[处理Windows库和错误检查]
    AA --> AB[调用LLD进行链接]
    AB --> AC[更新缓存和写清单]
    J --> AD[结束子进度节点]
    AC --> AD
    AD --> AE[结束]
    
    style A fill:#90EE90,stroke:#006400
    style AE fill:#90EE90,stroke:#006400
    style I fill:#FFA07A,stroke:#FF4500
    style W fill:#FFA07A,stroke:#FF4500
    style E fill:#FFA07A,stroke:#FF4500
