graph TD
    A[开始] --> B{args.len <=1?}
    B -->|是| C[输出用法信息]
    C --> D[报错: 需要命令参数]
    B -->|否| E{检测到ZIG_IS_DETECTING_LIBC_PATHS?}
    E -->|是| F{存在无限循环环境变量?}
    F -->|是| G[报错: 无法找到libc]
    F -->|否| H[设置环境变量防止递归]
    H --> I{args[1]是'cc'?}
    I -->|是| J[执行cc命令]
    I -->|否| K[修改参数为cc并执行]
    E -->|否| L[解析命令cmd=args[1]]
    L --> M{命令匹配列表}
    M -->|build-exe/build-lib/build-obj| N[调用buildOutputType]
    M -->|test| O[调用buildOutputType为测试]
    M -->|run| P[调用buildOutputType运行]
    M -->|dlltool/ranlib/lib/ar| Q[调用llvmArMain]
    M -->|build| R[调用cmdBuild]
    M -->|clang/-cc1/-cc1as| S[调用clangMain]
    M -->|ld.lld/lld-link/wasm-ld| T[调用lldMain]
    M -->|cc/c++| U[调用buildOutputType为C/C++]
    M -->|translate-c| V[调用translate-c逻辑]
    M -->|rc| W[调用resinator工具]
    M -->|fmt| X[调用格式化模块]
    M -->|objcopy/fetch/libc/std/init/targets| Y[调用对应工具或模块]
    M -->|version/env/zen/help/ast-check/detect-cpu| Z[执行简单输出或检查]
    M -->|调试扩展命令| AA[执行调试相关逻辑]
    M -->|其他命令| AB[输出用法并报错]
    N --> AC[结束]
    O --> AC
    P --> AC
    Q --> AC
    R --> AC
    S --> AC
    T --> AC
    U --> AC
    V --> AC
    W --> AC
    X --> AC
    Y --> AC
    Z --> AC
    AA --> AC
    AB --> AC
    D --> AC
    G --> AC
    J --> AC
    K --> AC
