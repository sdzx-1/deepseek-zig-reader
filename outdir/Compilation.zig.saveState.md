flowchart TD
    A[开始] --> B[检查增量编译]
    B --> C{存在bin_file?}
    C -->|否| D[返回]
    C -->|是| E[初始化bufs和pt_headers]
    E --> F{存在zcu?}
    F -->|否| L[跳过zcu处理]
    F -->|是| G[构造Header]
    G --> H[填充pt_headers]
    H --> I[添加Header和pt_headers到bufs]
    I --> J[添加zcu依赖数据到bufs]
    J --> K[处理locals数据到bufs]
    K --> L
    L --> M{处理linker state}
    M -->|lf.tag是.wasm| N[处理Wasm特定数据]
    N --> O[添加Wasm相关结构到bufs]
    O --> P{是Obj模式?}
    P -->|是| Q[添加Obj模式数据]
    P -->|否| R[添加Exe模式数据]
    Q --> S
    R --> S
    S --> T[处理剩余Wasm数据]
    T --> U[生成basename]
    U --> V[原子文件写入bufs]
    V --> W[完成]
    M -->|其他tag| X[记录错误日志]
    X --> U
