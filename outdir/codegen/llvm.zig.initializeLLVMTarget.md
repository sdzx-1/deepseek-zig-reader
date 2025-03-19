graph TD
    Start[开始] --> CheckArch{检查 arch 类型}
    
    CheckArch -->|aarch64/aarch64_be| AArch64[初始化AArch64目标]
    CheckArch -->|amdgcn| AMDGPU[初始化AMDGPU目标]
    CheckArch -->|thumb/thumbeb/arm/armeb| ARM[初始化ARM目标]
    CheckArch -->|avr| AVR[初始化AVR目标]
    CheckArch -->|bpfel/bpfeb| BPF[初始化BPF目标]
    CheckArch -->|hexagon| Hexagon[初始化Hexagon目标]
    CheckArch -->|lanai| Lanai[初始化Lanai目标]
    CheckArch -->|mips/mipsel/mips64/mips64el| MIPS[初始化MIPS目标]
    CheckArch -->|msp430| MSP430[初始化MSP430目标]
    CheckArch -->|nvptx/nvptx64| NVPTX[初始化NVPTX目标<br>（无AsmParser）]
    CheckArch -->|powerpc系列| PowerPC[初始化PowerPC目标]
    CheckArch -->|riscv32/riscv64| RISC-V[初始化RISC-V目标]
    CheckArch -->|sparc/sparc64| Sparc[初始化Sparc目标]
    CheckArch -->|s390x| SystemZ[初始化SystemZ目标]
    CheckArch -->|wasm32/wasm64| WebAssembly[初始化WebAssembly目标]
    CheckArch -->|x86/x86_64| X86[初始化X86目标]
    CheckArch -->|xtensa| Xtensa{检查llvm_has_xtensa}
    CheckArch -->|m68k| M68k{检查llvm_has_m68k}
    CheckArch -->|csky| CSKY{检查llvm_has_csky}
    CheckArch -->|ve| VE[初始化VE目标]
    CheckArch -->|arc| ARC{检查llvm_has_arc}
    CheckArch -->|loongarch32/loongarch64| LoongArch[初始化LoongArch目标]
    CheckArch -->|spirv系列| SPIRV[空操作]
    CheckArch -->|kalimba/propeller| Unreachable[触发unreachable错误]

    Xtensa -->|是| XtensaInit[初始化Xtensa目标<br>（无AsmPrinter）]
    Xtensa -->|否| SkipXtensa[跳过]
    
    M68k -->|是| M68kInit[初始化M68k目标]
    M68k -->|否| SkipM68k[跳过]
    
    CSKY -->|是| CSKYInit[初始化CSKY目标<br>（无AsmPrinter）]
    CSKY -->|否| SkipCSKY[跳过]
    
    ARC -->|是| ARCInit[初始化ARC目标<br>（无AsmParser）]
    ARC -->|否| SkipARC[跳过]

    subgraph 通用初始化流程
        AArch64 --> CallLLVM[调用5个LLVM初始化函数]
        AMDGPU --> CallLLVM
        ARM --> CallLLVM
        AVR --> CallLLVM
        BPF --> CallLLVM
        Hexagon --> CallLLVM
        Lanai --> CallLLVM
        MIPS --> CallLLVM
        MSP430 --> CallLLVM
        NVPTX --> CallLLVM_NVPTX[调用4个LLVM初始化函数]
        PowerPC --> CallLLVM
        RISC-V --> CallLLVM
        Sparc --> CallLLVM
        SystemZ --> CallLLVM
        WebAssembly --> CallLLVM
        X86 --> CallLLVM
        XtensaInit --> CallLLVM_Xtensa[调用3个初始化函数+AsmParser]
        M68kInit --> CallLLVM
        CSKYInit --> CallLLVM_CSKY[调用3个初始化函数+AsmParser]
        VE --> CallLLVM
        ARCInit --> CallLLVM_ARC[调用3个初始化函数+AsmPrinter]
        LoongArch --> CallLLVM
    end

    CallLLVM --> End[结束]
    CallLLVM_NVPTX --> End
    CallLLVM_Xtensa --> End
    CallLLVM_CSKY --> End
    CallLLVM_ARC --> End
    SPIRV --> End
    SkipXtensa --> End
    SkipM68k --> End
    SkipCSKY --> End
    SkipARC --> End
    Unreachable --> End[触发错误终止]
