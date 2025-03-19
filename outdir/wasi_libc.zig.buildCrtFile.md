graph TD
    Start[开始] --> CheckLLVM{检查LLVM支持}
    CheckLLVM -- 不支持 --> Error[返回错误ZigCompilerNotBuiltWithLLVMExtensions]
    CheckLLVM -- 支持 --> InitArena[初始化Arena分配器]
    InitArena --> SwitchCrtFile{根据crt_file类型分支}
    
    SwitchCrtFile -- crt1_reactor_o --> Crt1Reactor[准备参数和源文件]
    Crt1Reactor --> AddCCArgsReactor[添加CC参数]
    AddCCArgsReactor --> AddLibcBottomIncludesReactor[添加libc-bottom-half头文件]
    AddLibcBottomIncludesReactor --> BuildCrt1Reactor[调用comp.build_crt_file生成crt1-reactor.o]
    
    SwitchCrtFile -- crt1_command_o --> Crt1Command[准备参数和源文件]
    Crt1Command --> AddCCArgsCommand[添加CC参数]
    AddCCArgsCommand --> AddLibcBottomIncludesCommand[添加libc-bottom-half头文件]
    AddLibcBottomIncludesCommand --> BuildCrt1Command[调用comp.build_crt_file生成crt1-command.o]
    
    SwitchCrtFile -- libc_a --> LibcA[初始化libc_sources列表]
    LibcA --> CompileEmmalloc[编译emmalloc部分]
    CompileEmmalloc --> AddCCArgsEmmalloc[添加CC参数（O3优化+无严格别名）]
    AddCCArgsEmmalloc --> AddEmmallocFiles[添加emmalloc源文件]
    
    LibcA --> CompileBottomHalf[编译libc-bottom-half]
    CompileBottomHalf --> AddCCArgsBottom[添加CC参数（O3优化）]
    AddCCArgsBottom --> AddLibcBottomIncludes[添加libc-bottom头文件]
    AddLibcBottomIncludes --> AddBottomHalfFiles[添加bottom-half源文件]
    
    LibcA --> CompileTopHalf[编译libc-top-half]
    CompileTopHalf --> AddCCArgsTop[添加CC参数（O3优化）]
    AddCCArgsTop --> AddLibcTopIncludes[添加libc-top头文件]
    AddLibcTopIncludes --> AddTopHalfFiles[添加top-half源文件]
    
    AddTopHalfFiles --> BuildLibcA[调用comp.build_crt_file生成libc.a]
    
    SwitchCrtFile -- libdl_a --> LibDlA[准备参数和源文件]
    LibDlA --> AddCCArgsLibdl[添加CC参数（O3优化）]
    AddCCArgsLibdl --> AddLibcBottomIncludesLibdl[添加libc-bottom头文件]
    AddLibcBottomIncludesLibdl --> AddEmuDlFiles[添加emulated_dl源文件]
    AddEmuDlFiles --> BuildLibDlA[调用comp.build_crt_file生成libdl.a]
    
    SwitchCrtFile -- 其他库类型* --> OtherCases[类似流程：准备参数/添加头文件/收集源文件]
    OtherCases --> BuildOtherLib[调用comp.build_crt_file生成对应库]
    
    BuildCrt1Reactor --> End[返回结果]
    BuildCrt1Command --> End
    BuildLibcA --> End
    BuildLibDlA --> End
    BuildOtherLib --> End
    Error --> End
    
    style Start fill:#90EE90,stroke:#333
    style CheckLLVM fill:#FFD700,stroke:#333
    style Error fill:#FF6347,stroke:#333
    style InitArena fill:#87CEEB,stroke:#333
    style SwitchCrtFile fill:#FFA07A,stroke:#333
    style End fill:#90EE90,stroke:#333
