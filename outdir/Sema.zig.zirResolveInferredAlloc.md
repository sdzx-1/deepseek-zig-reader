graph TD
    A[开始] --> B[初始化变量: tracy, pt, zcu, gpa等]
    B --> C[解析inst_data, src, ty_src, ptr]
    C --> D{检查ptr_inst的tag}
    D --> |.inferred_alloc_comptime| E[处理已完成的推断分配]
    D --> |.inferred_alloc| F[处理动态推断分配]
    
    E --> E1[安全检查: 清空原指令]
    E1 --> E2[解析resolved_ptr的类型]
    E2 --> E3[处理函数体分析]
    E3 --> E4[返回resolved_ptr]
    
    F --> F1[提取ia1和ia2]
    F1 --> F2[解析peer_vals和final_elem_ty]
    F2 --> F3[构造final_ptr_ty]
    F3 --> F4{是否为常量?}
    F4 --> |是| F5[尝试解析comptime已知指针]
    F4 --> |否| F6[验证变量类型]
    
    F5 --> F7[是否需要完全解析类型?]
    F7 --> F8[返回const指针]
    
    F6 --> F9[检查comptime-only类型错误]
    F9 --> F10[检查naked函数限制]
    F10 --> F11[修改指令为.alloc]
    
    F11 --> F12[遍历存储指令]
    F12 --> F13{替换为单指令?}
    F13 --> |是| F14[直接替换指令]
    F13 --> |否| F15[替换为block结构]
    F14 & F15 --> F16{处理完所有prongs?}
    F16 --> |否| F12
    F16 --> |是| F17{是否为常量?}
    F17 --> |是| F18[返回const指针]
    F17 --> |否| F19[返回原始指针]
    
    E4 --> Z[结束]
    F8 --> Z
    F18 --> Z
    F19 --> Z
