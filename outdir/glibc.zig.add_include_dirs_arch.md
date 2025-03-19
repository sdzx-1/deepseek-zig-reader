flowchart TD
    Start[开始] --> GetArch[获取arch和is_64等变量]
    GetArch --> CheckX86{arch是x86?}
    
    CheckX86 --> |是| CheckX86_64{是x86_64?}
    CheckX86_64 --> |是| CheckNPTL_X86_64{opt_nptl存在?}
    CheckNPTL_X86_64 --> |是| AddX86_64_NPTL[添加x86_64/nptl路径]
    CheckNPTL_X86_64 --> |否| CheckX32{ABI是gnux32?}
    CheckX32 --> |是| AddX86_64_X32[添加x86_64/x32路径]
    CheckX32 --> |否| AddX86_64[添加x86_64路径]
    
    CheckX86_64 --> |否| CheckX86{是x86?}
    CheckX86 --> |是| CheckNPTL_X86{opt_nptl存在?}
    CheckNPTL_X86 --> |是| AddI386_NPTL[添加i386/nptl路径]
    CheckNPTL_X86 --> |否| AddI386[添加i386路径]
    
    CheckX86 --> |后续公共逻辑| CheckX86_NPTL{opt_nptl存在?}
    CheckX86_NPTL --> |是| AddX86_NPTL[添加x86/nptl路径]
    CheckX86_NPTL --> |否| AddX86[添加x86路径]
    
    CheckX86 --> |否| CheckArm{arch是Arm?}
    CheckArm --> |是| CheckNPTL_Arm{opt_nptl存在?}
    CheckNPTL_Arm --> |是| AddArm_NPTL[添加arm/nptl路径]
    CheckNPTL_Arm --> |否| AddArm[添加arm路径]
    
    CheckArm --> |否| CheckMIPS{arch是MIPS?}
    CheckMIPS --> |是| CheckNPTL_MIPS{opt_nptl存在?}
    CheckNPTL_MIPS --> |是| AddMIPS_NPTL[添加mips/nptl路径]
    CheckNPTL_MIPS --> |否| CheckMIPS64{is_64?}
    CheckMIPS64 --> |是| AddMIPS64[添加mips/mips64路径]
    CheckMIPS64 --> |否| AddMIPS32[添加mips/mips32路径]
    CheckMIPS64 --> AddMIPS[添加mips路径]
    
    CheckMIPS --> |否| CheckSPARC{arch是SPARC?}
    CheckSPARC --> |是| CheckNPTL_SPARC{opt_nptl存在?}
    CheckNPTL_SPARC --> |是| AddSPARC_NPTL[添加sparc/nptl路径]
    CheckNPTL_SPARC --> |否| CheckSPARC64{is_64?}
    CheckSPARC64 --> |是| AddSPARC64[添加sparc/sparc64路径]
    CheckSPARC64 --> |否| AddSPARC32[添加sparc/sparc32路径]
    CheckSPARC64 --> AddSPARC[添加sparc路径]
    
    CheckSPARC --> |否| CheckAArch64{arch是AArch64?}
    CheckAArch64 --> |是| CheckNPTL_AArch64{opt_nptl存在?}
    CheckNPTL_AArch64 --> |是| AddAArch64_NPTL[添加aarch64/nptl路径]
    CheckNPTL_AArch64 --> |否| AddAArch64[添加aarch64路径]
    
    CheckAArch64 --> |否| CheckPPC{arch是PowerPC?}
    CheckPPC --> |是| CheckNPTL_PPC{opt_nptl存在?}
    CheckNPTL_PPC --> |是| AddPPC_NPTL[添加powerpc/nptl路径]
    CheckNPTL_PPC --> |否| CheckPPC64{is_64?}
    CheckPPC64 --> |是| AddPPC64[添加powerpc/powerpc64路径]
    CheckPPC64 --> |否| AddPPC32[添加powerpc/powerpc32路径]
    CheckPPC64 --> AddPPC[添加powerpc路径]
    
    CheckPPC --> |否| CheckRISCV{arch是RISC-V?}
    CheckRISCV --> |是| CheckNPTL_RISCV{opt_nptl存在?}
    CheckNPTL_RISCV --> |是| AddRISCV_NPTL[添加riscv/nptl路径]
    CheckNPTL_RISCV --> |否| AddRISCV[添加riscv路径]
    
    CheckRISCV --> |否| CheckS390X{arch是s390x?}
    CheckS390X --> |是| CheckNPTL_S390{opt_nptl存在?}
    CheckNPTL_S390 --> |是| AddS390_NPTL[添加s390/nptl路径]
    CheckNPTL_S390 --> |否| AddS390_64[添加s390/s390-64路径]
    CheckNPTL_S390 --> AddS390[添加s390路径]
    
    CheckS390X --> |否| CheckLoongArch{arch是LoongArch?}
    CheckLoongArch --> |是| AddLoongArch[添加loongarch路径]
    
    CheckLoongArch --> |否| CheckM68K{arch是m68k?}
    CheckM68K --> |是| CheckNPTL_M68K{opt_nptl存在?}
    CheckNPTL_M68K --> |是| AddM68K_NPTL[添加m68k/nptl路径]
    CheckNPTL_M68K --> |否| AddM68K_CF[添加m68k/m680x0路径]
    CheckNPTL_M68K --> AddM68K[添加m68k路径]
    
    CheckM68K --> |否| CheckARC{arch是ARC?}
    CheckARC --> |是| AddARC[添加arc路径]
    
    CheckARC --> |否| CheckCSKY{arch是CSKY?}
    CheckCSKY --> |是| AddCSKY[添加csky路径]
    
    CheckCSKY --> |否| End[结束]
    
    style Start fill:#9f9,stroke:#333,stroke-width:2px
    style End fill:#f99,stroke:#333,stroke-width:2px
