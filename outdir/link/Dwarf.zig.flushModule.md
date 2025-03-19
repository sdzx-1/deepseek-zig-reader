graph TD
    A[开始flushModule] --> B[处理类型信息]
    B --> C[处理模块根目录路径]
    C --> D{debug_aranges.dirty?}
    D -->|是| E[更新debug_aranges段]
    D -->|否| F{debug_frame.dirty?}
    E --> F
    F -->|是| G[更新debug_frame段]
    F -->|否| H{debug_info.dirty?}
    G --> H
    H -->|是| I[更新debug_info段]
    H -->|否| J{debug_abbrev.dirty?}
    I --> J
    J -->|是| K[更新debug_abbrev段]
    J -->|否| L{debug_str.dirty?}
    K --> L
    L -->|是| M[更新debug_str段]
    L -->|否| N{debug_line.dirty?}
    M --> N
    N -->|是| O[更新debug_line段]
    N -->|否| P{debug_line_str.dirty?}
    O --> P
    P -->|是| Q[更新debug_line_str段]
    P -->|否| R{debug_loclists.dirty?}
    Q --> R
    R -->|是| S[标记debug_loclists干净]
    R -->|否| T{debug_rnglists.dirty?}
    S --> T
    T -->|是| U[更新debug_rnglists段]
    T -->|否| V[验证所有段已清理]
    U --> V
    V --> W[结束flushModule]

    subgraph 类型信息处理
        B --> B1[获取或创建anyerror类型]
        B1 --> B2[初始化WipNav结构]
        B2 --> B3[生成枚举类型信息]
        B3 --> B4[写入全局错误集条目]
        B4 --> B5[更新调试信息条目]
    end

    subgraph 模块路径处理
        C --> C1[遍历所有模块]
        C1 --> C2[解析根目录路径]
        C2 --> C3[存储到debug_line_str]
    end

    subgraph 段更新公共逻辑
        E & G & I & K & M & O & Q & U --> |段操作| X[清空旧数据]
        X --> Y[生成头部信息]
        Y --> Z[处理重定位条目]
        Z --> AA[写入最终内容]
        AA --> AB[标记段为干净]
    end
