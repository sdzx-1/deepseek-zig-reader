flowchart TD
    A[开始] --> B{检查f.location类型}
    B -->|relative_path| C[检查路径是否为绝对路径]
    C -->|是| D[返回错误: 绝对路径]
    C -->|否| E[检查是否有哈希令牌]
    E -->|有| F[返回错误: 路径依赖无哈希]
    E -->|无| G[检查路径是否在项目内]
    G -->|否| H[返回错误: 路径越界]
    G -->|是| I[设置package_root并加载清单]
    I --> J[检查是否有build.zig]
    J -->|无| K[检查构建文件是否存在]
    K --> L[返回错误或继续]
    J -->|有| M[检查是否递归]
    M -->|否| N[结束]
    M -->|是| O[为依赖排队作业]
    
    B -->|remote| P[检查是否有哈希]
    P -->|有| Q[构造缓存路径]
    Q --> R[检查缓存是否存在]
    R -->|存在| S[加载清单并检查构建文件]
    S --> T[检查是否递归]
    T -->|否| U[结束]
    T -->|是| O
    R -->|不存在| V[检查是否只读模式]
    V -->|是| W[返回错误: 包未找到]
    V -->|否| X[获取并解压远程资源]
    
    B -->|path_or_url| Y[尝试打开为目录]
    Y -->|成功| Z[处理目录资源]
    Y -->|失败| AA[尝试打开为文件]
    AA -->|成功| BB[处理文件资源]
    AA -->|失败| CC[解析为URI]
    CC --> DD[初始化资源]
    DD --> X
    
    X --> EE[运行资源处理]
    EE --> F1[结束]
    
    style A fill:#4CAF50,stroke:#388E3C
    style D,F,H,W fill:#FF5252,stroke:#D32F2F
    style N,U,F1 fill:#2196F3,stroke:#1976D2
    style O,X,EE fill:#FFC107,stroke:#FFA000
