graph TD
    Start[开始jitCmd] --> Init[初始化: 颜色/进度节点/目标解析]
    Init --> ExeName[生成可执行文件名]
    ExeName --> SelfExe[获取自身可执行路径]
    SelfExe --> Optimize[确定优化模式/strip设置]
    Optimize --> EnvVars[处理环境变量: ZIG_LIB_DIR等]
    EnvVars --> OpenDirs[打开Zig库目录和全局缓存目录]
    OpenDirs --> ThreadPool[初始化线程池]
    ThreadPool --> BuildArgv[构建子进程参数]

    BuildArgv --> ModConfig[创建主模块配置]
    ModConfig --> CheckAro{options.depend_on_aro?}
    CheckAro -->|Yes| CreateAro[创建aro模块并添加依赖]
    CheckAro -->|No| Continue

    Continue --> Compilation[创建编译实例]
    Compilation --> ServerCheck{options.server?}
    ServerCheck -->|Yes| Server[启动服务器并更新]
    ServerCheck -->|No| Update[常规更新模块]

    Server --> ErrorCheck[检查错误包]
    Update --> ErrorCheck
    ErrorCheck -->|有错误| Exit2[退出代码2]
    ErrorCheck -->|无错误| GetExePath[获取可执行路径]

    GetExePath --> PrependArgs[添加前缀参数]
    PrependArgs --> ExecCheck{process.can_execv?}
    ExecCheck -->|Yes| TryExecv[尝试execv执行]
    TryExecv -->|失败| FatalExecv[报错退出]
    ExecCheck -->|No| SpawnCheck{支持spawn?}
    SpawnCheck -->|No| FatalSpawn[报错退出]
    SpawnCheck -->|Yes| Spawn[生成子进程]

    Spawn --> CaptureCheck{options.capture?}
    CaptureCheck -->|Yes| ReadOutput[读取输出到缓冲区]
    CaptureCheck -->|No| Wait[等待进程结束]

    Wait --> ExitCode{退出状态}
    ExitCode -->|0| Success[成功退出]
    ExitCode -->|非0| FatalCode[报错退出]
    ExitCode -->|崩溃| FatalCrash[报错退出]

    style Start stroke:#333,stroke-width:2px
    style Exit2 stroke:#f00,stroke-width:2px
    style FatalExecv stroke:#f00,stroke-width:2px
    style FatalSpawn stroke:#f00,stroke-width:2px
    style FatalCode stroke:#f00,stroke-width:2px
    style FatalCrash stroke:#f00,stroke-width:2px
    style Success stroke:#0f0,stroke-width:2px
