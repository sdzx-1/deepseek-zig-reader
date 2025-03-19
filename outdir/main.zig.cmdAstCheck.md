flowchart TD
    Start([开始]) --> ParseArgs[解析命令行参数]
    ParseArgs --> |处理每个参数| CheckArg{参数以'-'开头?}
    CheckArg --> |是| HandleOptions[处理选项]
    HandleOptions --> |遇到-h/--help| PrintHelp[打印帮助信息并退出]
    HandleOptions --> |遇到-t| SetOutputText[设置want_output_text为true]
    HandleOptions --> |遇到--zon| SetForceZon[设置force_zon为true]
    HandleOptions --> |遇到--color| ReadColor[读取颜色参数并设置]
    HandleOptions --> |未知选项| FatalUnknown[报错并退出]
    CheckArg --> |否| CheckFile[检查是否已设置源文件]
    CheckFile --> |未设置| SetSourceFile[设置zig_source_file]
    CheckFile --> |已设置| FatalExtra[报错多余参数并退出]
    
    ParseArgs --> AfterArgs[参数处理完毕]
    AfterArgs --> OpenFile[打开源文件或读取stdin]
    OpenFile --> |文件存在| ReadFile[读取文件内容到内存]
    OpenFile --> |文件不存在| FatalFile[报错文件打开失败]
    ReadFile --> CheckSize[检查文件大小是否超限]
    CheckSize --> |超过限制| FatalSize[报错文件过大]
    CheckSize --> |正常| StoreSource[存储源码到file结构]
    
    AfterArgs --> |从stdin读取| ReadStdin[读取标准输入内容]
    ReadStdin --> StoreStdin[存储到file结构]
    
    StoreSource --> DetermineMode[确定解析模式]
    StoreStdin --> DetermineMode
    DetermineMode --> |.zon扩展名或--zon| ModeZon[设置为zon模式]
    DetermineMode --> |其他情况| ModeZig[设置为zig模式]
    
    ModeZig --> ParseZig[解析Zig AST]
    ParseZig --> GenZIR[生成ZIR中间代码]
    GenZIR --> CheckZIRErrors{有编译错误?}
    CheckZIRErrors --> |是| RenderZIRErrors[渲染ZIR错误信息]
    RenderZIRErrors --> ExitError[退出码1]
    CheckZIRErrors --> |否| CheckOutputText{需要输出文本?}
    CheckOutputText --> |是| PrintZIRStats[打印ZIR统计信息]
    PrintZIRStats --> RenderZIRText[渲染ZIR文本表示]
    RenderZIRText --> ExitClean[正常退出]
    CheckOutputText --> |否| ExitClean
    
    ModeZon --> ParseZon[解析Zon AST]
    ParseZon --> GenZOIR[生成ZOIR中间代码]
    GenZOIR --> CheckZOIRErrors{有编译错误?}
    CheckZOIRErrors --> |是| RenderZOIRErrors[渲染ZOIR错误信息]
    RenderZOIRErrors --> ExitError
    CheckZOIRErrors --> |否| CheckOutputTextZon{需要输出文本?}
    CheckOutputTextZon --> |是| PrintZOIRText[渲染ZOIR文本表示]
    PrintZOIRText --> ExitClean
    CheckOutputTextZon --> |否| ExitClean
    
    ExitError --> End([结束])
    ExitClean --> End
    
    classDef condition fill:#f9f,stroke:#333
    classDef process fill:#bbf,stroke:#333
    classDef startEnd fill:#9f9,stroke:#333
    class CheckArg,CheckFile,CheckSize,CheckZIRErrors,CheckOutputText,CheckZOIRErrors,CheckOutputTextZon condition
    class ParseArgs,HandleOptions,OpenFile,ReadFile,DetermineMode,ParseZig,GenZIR,PrintZIRStats,RenderZIRText,ParseZon,GenZOIR,PrintZOIRText process
    class Start,End,PrintHelp,FatalUnknown,FatalExtra,FatalFile,FatalSize,ExitError,ExitClean startEnd
