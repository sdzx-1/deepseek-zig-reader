graph TD
    A[开始] --> B[初始化变量和参数]
    B --> C{检查arg_mode}
    C -->|build/translate_c/zig_test/run| D[处理命令行参数]
    C -->|cc/cpp| E[处理Clang兼容参数]
    D --> F[配置模块选项]
    E --> F
    F --> G[创建全局缓存目录]
    G --> H[处理目标架构和优化选项]
    H --> I[处理链接器参数]
    I --> J[处理C/Rust源文件]
    J --> K[创建编译实例Compilation]
    K --> L{是否有错误}
    L -->|是| M[输出错误并退出]
    L -->|否| N[配置输出文件和缓存]
    N --> O{是否为translate_c模式}
    O -->|是| P[执行translate-c命令]
    O -->|否| Q[更新模块并编译]
    Q --> R{是否成功}
    R -->|否| S[保存状态并退出]
    R -->|是| T[生成可执行文件]
    T --> U{是否为run/test模式}
    U -->|是| V[执行运行或测试]
    U -->|否| W[清理资源并退出]
    V --> W
    M --> W
    S --> W
    W[结束]
