graph TD
    A[开始] --> B{检查构建选项}
    B -->|skip_non_native且非COFF格式| C[panic]
    B -->|否则| D[初始化zcu, ip, comp, target]
    D --> E{使用LLVM?}
    E -->|是| F[遍历export_indices]
    F --> G{检查导出类型}
    G -->|.nav| H[获取导出符号的调用约定]
    G -->|.uav| I[跳过]
    H --> J{匹配调用约定和符号名}
    J -->|main且link_libc| K[设置have_c_main标志]
    J -->|WinMain等Windows入口点| L[设置对应标志]
    E -->|否| M{存在llvm_object?}
    M -->|是| N[调用llvm_object.updateExports并返回]
    M -->|否| O[处理元数据]
    O --> P{exported类型}
    P -->|.nav| Q[获取或创建Atom]
    P -->|.uav| R[尝试降低UAV]
    R --> S{处理结果}
    S -->|成功| T[获取元数据]
    S -->|失败| U[记录错误并返回]
    T --> V[获取atom_index和atom]
    V --> W[遍历export_indices]
    W --> X{检查section合法性}
    X -->|非.text段| Y[记录错误并继续]
    X -->|合法| Z{检查linkage类型}
    Z -->|link_once| AA[记录错误并继续]
    Z -->|其他| AB[分配符号索引]
    AB --> AC[设置符号名称、值、段号、类型]
    AC --> AD{处理linkage类型}
    AD -->|strong| AE[设置EXTERNAL存储类]
    AD -->|internal| AF[panic: TODO]
    AD -->|weak| AG[panic: TODO]
    AG --> AH[解析全局符号]
    AE --> AH
    AH --> AI[结束循环]
    AI --> AJ[函数结束]
