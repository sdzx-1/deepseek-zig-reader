graph TD
    A[开始] --> B[初始化: inst_map, tags, datas等]
    B --> C[进入while(true)循环]
    C --> D[处理指令i]
    D --> E{检查指令类型}
    E -->|普通指令| F[调用对应的处理函数]
    E -->|noreturn指令| G[处理并结束循环]
    E -->|控制流指令| H[处理控制流如break, loop等]
    E -->|编译时控制流| I[处理comptime逻辑]
    F --> J{是否noreturn?}
    J -->|是| K[结束循环]
    J -->|否| L[存入inst_map, i++]
    H --> M{是否改变流程?}
    M -->|是, 如break| K
    M -->|否, 继续循环| C
    I --> N{comptime_break?}
    N -->|是| O[返回ComptimeBreak错误]
    N -->|否| P[继续循环]
    L --> C
    K --> Q[结束函数]
    O --> Q
    style A fill:#90EE90
    style Q fill:#FFA07A
