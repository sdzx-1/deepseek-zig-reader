flowchart TD
    A[开始] --> B[初始化match_stack, old_contents, new_contents]
    B --> C[处理主结构体指令]
    C --> D["填充match_stack和inst_map（显式类型和其他指令）"]
    D --> E{检查match_stack是否为空？}
    E -- 是 --> F[结束]
    E -- 否 --> G[弹出match_stack中的项]
    G --> H{"旧新类型捕获数相同？"}
    H -- 否 --> E
    H -- 是 --> I[将旧新指令映射到inst_map]
    I --> J[初始化各类声明容器（named_decls, named_tests等）]
    J --> K[遍历旧ZIR声明填充容器]
    K --> L[遍历新ZIR声明进行匹配]
    L --> M{"声明类型判断"}
    M -- comptime/usingnamespace/unnamed_test --> N["按顺序匹配"]
    M -- test/decltest/const/var --> O["按名称匹配"]
    N & O --> P{是否匹配成功？}
    P -- 否 --> L
    P -- 是 --> Q[映射声明到inst_map]
    Q --> R[处理嵌套的显式类型和其他指令]
    R --> S["填充match_stack和inst_map"]
    S --> L
    L --> T{是否处理完所有新声明？}
    T -- 否 --> L
    T -- 是 --> E
