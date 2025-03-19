flowchart TD
    Start([Start]) --> CheckArch{Check target.cpu.arch}
    
    CheckArch -->|.arc| ReturnARC["Return ARC data layout"]
    CheckArch -->|.xcore| ReturnXcore["Return Xcore data layout"]
    CheckArch -->|.hexagon| ReturnHexagon["Return Hexagon data layout"]
    CheckArch -->|.lanai| ReturnLanai["Return Lanai data layout"]
    CheckArch -->|.aarch64| CheckAarch64{Check target.ofmt and OS}
    CheckArch -->|.aarch64_be| ReturnAarch64BE["Return aarch64_be data layout"]
    CheckArch -->|.arm| CheckArmMacho{target.ofmt == .macho?}
    CheckArch -->|.armeb, .thumbeb| CheckArmebMacho{target.ofmt == .macho?}
    CheckArch -->|.thumb| CheckThumbMacho{target.ofmt == .macho?}
    CheckArch -->|.avr| ReturnAvr["Return AVR data layout"]
    CheckArch -->|.bpfeb| ReturnBpfeb["Return BPFEB data layout"]
    CheckArch -->|.bpfel| ReturnBpfel["Return BPFEL data layout"]
    CheckArch -->|.msp430| ReturnMsp430["Return MSP430 data layout"]
    CheckArch -->|.mips| ReturnMips["Return MIPS data layout"]
    CheckArch -->|.mipsel| ReturnMipsel["Return MIPSEL data layout"]
    CheckArch -->|.mips64| CheckMips64ABI{Check target.abi}
    CheckArch -->|.mips64el| CheckMips64elABI{Check target.abi}
    CheckArch -->|.m68k| ReturnM68k["Return M68k data layout"]
    CheckArch -->|.powerpc| CheckPowerpcOS{Check target.os.tag}
    CheckArch -->|.powerpcle| ReturnPowerpcle["Return PowerPCLE data layout"]
    CheckArch -->|.powerpc64| CheckPowerpc64OS{Check target.os.tag}
    CheckArch -->|.powerpc64le| CheckPowerpc64leOS{Check target.os.tag}
    CheckArch -->|.nvptx| ReturnNvptx["Return NVPTX data layout"]
    CheckArch -->|.nvptx64| ReturnNvptx64["Return NVPTX64 data layout"]
    CheckArch -->|.amdgcn| ReturnAmdgcn["Return AMDGCN data layout"]
    CheckArch -->|.riscv32| CheckRiscv32E{Check RISC-V E extension}
    CheckArch -->|.riscv64| CheckRiscv64E{Check RISC-V E extension}
    CheckArch -->|.sparc| ReturnSparc["Return SPARC data layout"]
    CheckArch -->|.sparc64| ReturnSparc64["Return SPARC64 data layout"]
    CheckArch -->|.s390x| CheckS390xOS{Check target.os.tag}
    CheckArch -->|.x86| CheckX86OS{Check target.os.tag}
    CheckArch -->|.x86_64| CheckX86_64OS{Check OS and ABI}
    CheckArch -->|.spirv| ReturnSpirv["Return SPIR-V data layout"]
    CheckArch -->|.spirv32| ReturnSpirv32["Return SPIRV32 data layout"]
    CheckArch -->|.spirv64| ReturnSpirv64["Return SPIRV64 data layout"]
    CheckArch -->|.wasm32| CheckWasm32Emscripten{target.os.tag == .emscripten?}
    CheckArch -->|.wasm64| CheckWasm64Emscripten{target.os.tag == .emscripten?}
    CheckArch -->|.ve| ReturnVE["Return VE data layout"]
    CheckArch -->|.csky| ReturnCsky["Return CSKY data layout"]
    CheckArch -->|.loongarch32| ReturnLoongarch32["Return LoongArch32 data layout"]
    CheckArch -->|.loongarch64| ReturnLoongarch64["Return LoongArch64 data layout"]
    CheckArch -->|.xtensa| ReturnXtensa["Return Xtensa data layout"]
    CheckArch -->|.kalimba, .propeller| Unreachable[unreachable]

    %% AArch64分支
    CheckAarch64 -->|.macho| CheckAarch64Windows{target.os.tag == .windows?}
    CheckAarch64 -->|Windows| ReturnAarch64Win["Return Windows aarch64 layout"]
    CheckAarch64 -->|Other| CheckAarch64ABI{target.abi == .ilp32?}
    CheckAarch64ABI -->|Yes| ReturnAarch64ILP32["Return ILP32 aarch64 layout"]
    CheckAarch64ABI -->|No| ReturnAarch64Default["Return default aarch64 layout"]

    %% Arm分支
    CheckArmMacho -->|Yes| ReturnArmMacho["Return ARM Mach-O layout"]
    CheckArmMacho -->|No| ReturnArmDefault["Return default ARM layout"]

    %% X86分支
    CheckX86OS -->|.windows| CheckX86ABI{Check target.abi}
    CheckX86OS -->|Other| CheckX86Ofmt{target.ofmt == .macho?}
    CheckX86Ofmt -->|Yes| ReturnX86Macho["Return X86 Mach-O layout"]
    CheckX86Ofmt -->|No| ReturnX86Default["Return default X86 layout"]

    %% X86_64分支
    CheckX86_64OS -->|Darwin/Macho| ReturnX64Macho["Return x86_64 Mach-O layout"]
    CheckX86_64OS -->|Windows| ReturnX64Win["Return x86_64 Windows layout"]
    CheckX86_64OS -->|Other| ReturnX64Default["Return default x86_64 layout"]

    %% 其他分支逻辑类似，用条件判断连接最终返回值...
    
    %% 终止节点
    ReturnARC --> End([End])
    ReturnXcore --> End
    ReturnHexagon --> End
    ReturnLanai --> End
    ReturnAarch64Win --> End
    ReturnAarch64ILP32 --> End
    ReturnAarch64Default --> End
    ReturnAarch64BE --> End
    ReturnArmMacho --> End
    ReturnArmDefault --> End
    ReturnAvr --> End
    ReturnBpfeb --> End
    ReturnBpfel --> End
    %% ...（其他返回节点均连接到End）

    style Start fill:#90EE90
    style End fill:#FFA07A
    style CheckArch fill:#87CEEB
    style ReturnARC fill:#FFD700
