graph TD
    A[开始 updateLazyType] --> B[初始化 zcu, ip, ty]
    B --> C[根据 type_index 记录日志]
    C --> D[创建 wip_nav 结构体并初始化]
    D --> E[处理类型索引]
    E -->|int_type| F["生成数值类型信息\n(写入 signedness、bits、大小、对齐)"]
    E -->|ptr_type| G["处理指针类型\n(根据 size 分 slice/普通指针)"]
    G -->|slice| H["生成切片结构体\n(ptr 和 len 字段)"]
    G -->|普通指针| I["写入指针属性\n(对齐、地址空间、const/volatile)"]
    E -->|array_type| J["生成数组类型\n(处理元素类型、长度)"]
    E -->|vector_type| K["生成向量类型\n(元素类型、长度)"]
    E -->|opt_type| L["生成可选类型\n(根据 opt_repr 处理不同布局)"]
    L -->|unpacked/error_set/pointer| M["生成带标签的联合体\n(has_value 字段和值/null)"]
    E -->|error_union_type| N["生成错误联合类型\n(is_error 和 payload/error 字段)"]
    E -->|simple_type| O["处理基础类型\n(数值类型/void类型)"]
    E -->|tuple_type| P["生成元组结构体\n(遍历字段并处理编译时值)"]
    P --> Q[循环处理每个元组字段]
    E -->|enum_type| R["生成枚举类型\n(处理标签类型和枚举值)"]
    E -->|func_type| S["生成函数类型\n(调用约定、参数类型、返回类型)"]
    E -->|error_set_type| T["生成错误集合\n(错误码和名称)"]
    E -->|其他类型| U[触发 unreachable 或特殊处理]
    U -->|anyerror/adhoc_inferred| V[延迟处理或终止]
    F --> W[写入调试信息]
    H --> W
    I --> W
    J --> W
    K --> W
    M --> W
    N --> W
    O --> W
    Q --> W
    R --> W
    S --> W
    T --> W
    W --> X[释放资源并更新条目]
    X --> Y[结束]
    V --> Y
