graph TD
    A[开始] --> B[获取zcu和ip]
    B --> C{switch ip.indexToKey}
    C -->|类型相关| D[调用Type.print]
    C -->|.undef| E[写入'undefined']
    C -->|.simple_value| F{switch simple_value}
    F -->|.void| G[写入'{}']
    F -->|.empty_tuple| H[写入'.{}']
    F -->|其他| I[写入tag名]
    C -->|.variable| J[写入'(variable)']
    C -->|.extern| K[写入'(extern ...)']
    C -->|.func| L[写入'(function ...)']
    C -->|.int| M{switch int.storage}
    M -->|数值类型| N[写入数值]
    M -->|.lazy_align| O{opt_sema存在?}
    O -->|是| P[计算对齐并写入]
    O -->|否| Q[写入@alignOf(...)]
    M -->|.lazy_size| R{opt_sema存在?}
    R -->|是| S[计算大小并写入]
    R -->|否| T[写入@sizeOf(...)]
    C -->|.err| U[写入error名]
    C -->|.error_union| V{error_union类型}
    V -->|.err_name| W[写入error名]
    V -->|.payload| X[递归调用print]
    C -->|.enum_literal| Y[写入.enum名]
    C -->|.enum_tag| Z[处理枚举标签]
    Z --> AA[查找tag_index]
    AA -->|存在| AB[写入.enum名]
    AA -->|不存在且level=0| AC[写入@enumFromInt(...)]
    AA -->|level>0| AD[递归调用print]
    C -->|.empty_enum_value| AE[写入空枚举值]
    C -->|.float| AF[写入浮点数]
    C -->|.slice| AG[处理切片]
    AG --> AH{ptr是否undef?}
    AH -->|是且len=0| AI[写入&.{}]
    AH -->|是但len≠0| AJ[递归调用print(ptr)]
    AH -->|否| AK[判断print_contents]
    AK -->|是| AL[调用printPtr]
    AK -->|否| AL
    AG --> AM[处理slice.len]
    AM -->|level=0| AN[写入(...)]
    AM -->|level>0| AO[递归调用print]
    C -->|.ptr| AP[处理指针]
    AP --> AQ[判断print_contents]
    AQ -->|是| AR[调用printPtr]
    AQ -->|否| AR
    C -->|.opt| AS{opt.val类型}
    AS -->|.none| AT[写入null]
    AS -->|其他| AU[递归调用print]
    C -->|.aggregate| AV[调用printAggregate]
    C -->|.un| AW[处理联合类型]
    AW -->|level=0| AX[写入.{ ... }]
    AW -->|tag=none| AY[写入@bitCast表达式]
    AW -->|tag存在| AZ[递归写入tag和val]
    C -->|其他| BA[结束处理]

    classDef logic fill:#f9f,stroke:#333;
    class C,F,M,O,R,V,AH,AS,AQ,AW logic;
