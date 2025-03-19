graph TD
    A[开始] --> B[参数检查]
    B --> C{缓存检查}
    C -->|命中| D[使用缓存结果]
    C -->|未命中| E[准备编译参数]
    E --> F{选择C前端}
    F -->|aro| G[调用aro_translate_c]
    F -->|clang| H[准备clang参数并调用translate_c]
    G --> I[捕获输出]
    H --> I
    I --> J{是否成功?}
    J -->|失败| K[处理错误并退出]
    J -->|成功| L[写入缓存文件]
    L --> M[更新缓存清单]
    M --> N[输出结果]
    D --> N
    K --> Z[结束]
    N --> Z
