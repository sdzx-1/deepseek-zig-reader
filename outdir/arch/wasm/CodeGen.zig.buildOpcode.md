flowchart TD
    Start[开始] --> CheckOp{检查 args.op}
    CheckOp --> |unreachable| Unreachable[返回 unreachable]
    CheckOp --> |nop| Nop[返回 unreachable]
    CheckOp --> |block| Block[返回 unreachable]
    CheckOp --> |loop| Loop[返回 unreachable]
    CheckOp --> |if| If[返回 unreachable]
    CheckOp --> |else| Else[返回 unreachable]
    CheckOp --> |end| End[返回 unreachable]
    CheckOp --> |br| Br[返回 unreachable]
    CheckOp --> |br_if| BrIf[返回 unreachable]
    CheckOp --> |br_table| BrTable[返回 unreachable]
    CheckOp --> |return| Return[返回 unreachable]
    CheckOp --> |call| Call[返回 unreachable]
    CheckOp --> |drop| Drop[返回 unreachable]
    CheckOp --> |select| Select[返回 unreachable]
    CheckOp --> |global_get| GlobalGet[返回 unreachable]
    CheckOp --> |global_set| GlobalSet[返回 unreachable]
    
    CheckOp --> |load| LoadWidth{检查 width}
    LoadWidth --> |8| Load8[检查 valtype1]
    Load8 --> |i32| Load8i32{检查 signedness}
    Load8i32 --> |signed| i32_load8_s[返回 .i32_load8_s]
    Load8i32 --> |unsigned| i32_load8_u[返回 .i32_load8_u]
    Load8 --> |i64| Load8i64{检查 signedness}
    Load8i64 --> |signed| i64_load8_s[返回 .i64_load8_s]
    Load8i64 --> |unsigned| i64_load8_u[返回 .i64_load8_u]
    
    LoadWidth --> |16| Load16[检查 valtype1]
    Load16 --> |i32| Load16i32{检查 signedness}
    Load16i32 --> |signed| i32_load16_s[返回 .i32_load16_s]
    Load16i32 --> |unsigned| i32_load16_u[返回 .i32_load16_u]
    Load16 --> |i64| Load16i64{检查 signedness}
    Load16i64 --> |signed| i64_load16_s[返回 .i64_load16_s]
    Load16i64 --> |unsigned| i64_load16_u[返回 .i64_load16_u]
    
    LoadWidth --> |32| Load32[检查 valtype1]
    Load32 --> |i64| Load32i64{检查 signedness}
    Load32i64 --> |signed| i64_load32_s[返回 .i64_load32_s]
    Load32i64 --> |unsigned| i64_load32_u[返回 .i64_load32_u]
    Load32 --> |i32| i32_load[返回 .i32_load]
    Load32 --> |f32| f32_load[返回 .f32_load]
    
    LoadWidth --> |64| Load64[检查 valtype1]
    Load64 --> |i64| i64_load[返回 .i64_load]
    Load64 --> |f64| f64_load[返回 .f64_load]
    
    LoadWidth --> |无 width| LoadNoWidth[检查 valtype1]
    LoadNoWidth --> |i32| i32_load[返回 .i32_load]
    LoadNoWidth --> |i64| i64_load[返回 .i64_load]
    LoadNoWidth --> |f32| f32_load[返回 .f32_load]
    LoadNoWidth --> |f64| f64_load[返回 .f64_load]
    
    CheckOp --> |store| StoreWidth{检查 width}
    StoreWidth --> |8| Store8[检查 valtype1]
    Store8 --> |i32| i32_store8[返回 .i32_store8]
    Store8 --> |i64| i64_store8[返回 .i64_store8]
    
    StoreWidth --> |16| Store16[检查 valtype1]
    Store16 --> |i32| i32_store16[返回 .i32_store16]
    Store16 --> |i64| i64_store16[返回 .i64_store16]
    
    StoreWidth --> |32| Store32[检查 valtype1]
    Store32 --> |i64| i64_store32[返回 .i64_store32]
    Store32 --> |i32| i32_store[返回 .i32_store]
    Store32 --> |f32| f32_store[返回 .f32_store]
    
    StoreWidth --> |64| Store64[检查 valtype1]
    Store64 --> |i64| i64_store[返回 .i64_store]
    Store64 --> |f64| f64_store[返回 .f64_store]
    
    StoreWidth --> |无 width| StoreNoWidth[检查 valtype1]
    StoreNoWidth --> |i32| i32_store[返回 .i32_store]
    StoreNoWidth --> |i64| i64_store[返回 .i64_store]
    StoreNoWidth --> |f32| f32_store[返回 .f32_store]
    StoreNoWidth --> |f64| f64_store[返回 .f64_store]
    
    CheckOp --> |memory_size| MemorySize[返回 .memory_size]
    CheckOp --> |memory_grow| MemoryGrow[返回 .memory_grow]
    
    CheckOp --> |const| ConstType{检查 valtype1}
    ConstType --> |i32| i32_const[返回 .i32_const]
    ConstType --> |i64| i64_const[返回 .i64_const]
    ConstType --> |f32| f32_const[返回 .f32_const]
    ConstType --> |f64| f64_const[返回 .f64_const]
    
    CheckOp --> |eqz| EqzType{检查 valtype1}
    EqzType --> |i32| i32_eqz[返回 .i32_eqz]
    EqzType --> |i64| i64_eqz[返回 .i64_eqz]
    
    CheckOp --> |eq| EqType{检查 valtype1}
    EqType --> |i32| i32_eq[返回 .i32_eq]
    EqType --> |i64| i64_eq[返回 .i64_eq]
    EqType --> |f32| f32_eq[返回 .f32_eq]
    EqType --> |f64| f64_eq[返回 .f64_eq]
    
    CheckOp --> |ne| NeType{检查 valtype1}
    NeType --> |i32| i32_ne[返回 .i32_ne]
    NeType --> |i64| i64_ne[返回 .i64_ne]
    NeType --> |f32| f32_ne[返回 .f32_ne]
    NeType --> |f64| f64_ne[返回 .f64_ne]
    
    CheckOp --> |lt| LtType{检查 valtype1}
    LtType --> |i32| LtSignedness_i32{检查 signedness}
    LtSignedness_i32 --> |signed| i32_lt_s[返回 .i32_lt_s]
    LtSignedness_i32 --> |unsigned| i32_lt_u[返回 .i32_lt_u]
    LtType --> |i64| LtSignedness_i64{检查 signedness}
    LtSignedness_i64 --> |signed| i64_lt_s[返回 .i64_lt_s]
    LtSignedness_i64 --> |unsigned| i64_lt_u[返回 .i64_lt_u]
    LtType --> |f32| f32_lt[返回 .f32_lt]
    LtType --> |f64| f64_lt[返回 .f64_lt]
    
    CheckOp --> |gt| GtType{检查 valtype1}
    GtType --> |i32| GtSignedness_i32{检查 signedness}
    GtSignedness_i32 --> |signed| i32_gt_s[返回 .i32_gt_s]
    GtSignedness_i32 --> |unsigned| i32_gt_u[返回 .i32_gt_u]
    GtType --> |i64| GtSignedness_i64{检查 signedness}
    GtSignedness_i64 --> |signed| i64_gt_s[返回 .i64_gt_s]
    GtSignedness_i64 --> |unsigned| i64_gt_u[返回 .i64_gt_u]
    GtType --> |f32| f32_gt[返回 .f32_gt]
    GtType --> |f64| f64_gt[返回 .f64_gt]
    
    CheckOp --> |le| LeType{检查 valtype1}
    LeType --> |i32| LeSignedness_i32{检查 signedness}
    LeSignedness_i32 --> |signed| i32_le_s[返回 .i32_le_s]
    LeSignedness_i32 --> |unsigned| i32_le_u[返回 .i32_le_u]
    LeType --> |i64| LeSignedness_i64{检查 signedness}
    LeSignedness_i64 --> |signed| i64_le_s[返回 .i64_le_s]
    LeSignedness_i64 --> |unsigned| i64_le_u[返回 .i64_le_u]
    LeType --> |f32| f32_le[返回 .f32_le]
    LeType --> |f64| f64_le[返回 .f64_le]
    
    CheckOp --> |ge| GeType{检查 valtype1}
    GeType --> |i32| GeSignedness_i32{检查 signedness}
    GeSignedness_i32 --> |signed| i32_ge_s[返回 .i32_ge_s]
    GeSignedness_i32 --> |unsigned| i32_ge_u[返回 .i32_ge_u]
    GeType --> |i64| GeSignedness_i64{检查 signedness}
    GeSignedness_i64 --> |signed| i64_ge_s[返回 .i64_ge_s]
    GeSignedness_i64 --> |unsigned| i64_ge_u[返回 .i64_ge_u]
    GeType --> |f32| f32_ge[返回 .f32_ge]
    GeType --> |f64| f64_ge[返回 .f64_ge]
    
    CheckOp --> |clz| ClzType{检查 valtype1}
    ClzType --> |i32| i32_clz[返回 .i32_clz]
    ClzType --> |i64| i64_clz[返回 .i64_clz]
    
    CheckOp --> |ctz| CtzType{检查 valtype1}
    CtzType --> |i32| i32_ctz[返回 .i32_ctz]
    CtzType --> |i64| i64_ctz[返回 .i64_ctz]
    
    CheckOp --> |popcnt| PopcntType{检查 valtype1}
    PopcntType --> |i32| i32_popcnt[返回 .i32_popcnt]
    PopcntType --> |i64| i64_popcnt[返回 .i64_popcnt]
    
    CheckOp --> |add| AddType{检查 valtype1}
    AddType --> |i32| i32_add[返回 .i32_add]
    AddType --> |i64| i64_add[返回 .i64_add]
    AddType --> |f32| f32_add[返回 .f32_add]
    AddType --> |f64| f64_add[返回 .f64_add]
    
    CheckOp --> |sub| SubType{检查 valtype1}
    SubType --> |i32| i32_sub[返回 .i32_sub]
    SubType --> |i64| i64_sub[返回 .i64_sub]
    SubType --> |f32| f32_sub[返回 .f32_sub]
    SubType --> |f64| f64_sub[返回 .f64_sub]
    
    CheckOp --> |mul| MulType{检查 valtype1}
    MulType --> |i32| i32_mul[返回 .i32_mul]
    MulType --> |i64| i64_mul[返回 .i64_mul]
    MulType --> |f32| f32_mul[返回 .f32_mul]
    MulType --> |f64| f64_mul[返回 .f64_mul]
    
    CheckOp --> |div| DivType{检查 valtype1}
    DivType --> |i32| DivSignedness_i32{检查 signedness}
    DivSignedness_i32 --> |signed| i32_div_s[返回 .i32_div_s]
    DivSignedness_i32 --> |unsigned| i32_div_u[返回 .i32_div_u]
    DivType --> |i64| DivSignedness_i64{检查 signedness}
    DivSignedness_i64 --> |signed| i64_div_s[返回 .i64_div_s]
    DivSignedness_i64 --> |unsigned| i64_div_u[返回 .i64_div_u]
    DivType --> |f32| f32_div[返回 .f32_div]
    DivType --> |f64| f64_div[返回 .f64_div]
    
    CheckOp --> |rem| RemType{检查 valtype1}
    RemType --> |i32| RemSignedness_i32{检查 signedness}
    RemSignedness_i32 --> |signed| i32_rem_s[返回 .i32_rem_s]
    RemSignedness_i32 --> |unsigned| i32_rem_u[返回 .i32_rem_u]
    RemType --> |i64| RemSignedness_i64{检查 signedness}
    RemSignedness_i64 --> |signed| i64_rem_s[返回 .i64_rem_s]
    RemSignedness_i64 --> |unsigned| i64_rem_u[返回 .i64_rem_u]
    
    CheckOp --> |and| AndType{检查 valtype1}
    AndType --> |i32| i32_and[返回 .i32_and]
    AndType --> |i64| i64_and[返回 .i64_and]
    
    CheckOp --> |or| OrType{检查 valtype1}
    OrType --> |i32| i32_or[返回 .i32_or]
    OrType --> |i64| i64_or[返回 .i64_or]
    
    CheckOp --> |xor| XorType{检查 valtype1}
    XorType --> |i32| i32_xor[返回 .i32_xor]
    XorType --> |i64| i64_xor[返回 .i64_xor]
    
    CheckOp --> |shl| ShlType{检查 valtype1}
    ShlType --> |i32| i32_shl[返回 .i32_shl]
    ShlType --> |i64| i64_shl[返回 .i64_shl]
    
    CheckOp --> |shr| ShrType{检查 valtype1}
    ShrType --> |i32| ShrSignedness_i32{检查 signedness}
    ShrSignedness_i32 --> |signed| i32_shr_s[返回 .i32_shr_s]
    ShrSignedness_i32 --> |unsigned| i32_shr_u[返回 .i32_shr_u]
    ShrType --> |i64| ShrSignedness_i64{检查 signedness}
    ShrSignedness_i64 --> |signed| i64_shr_s[返回 .i64_shr_s]
    ShrSignedness_i64 --> |unsigned| i64_shr_u[返回 .i64_shr_u]
    
    CheckOp --> |rotl| RotlType{检查 valtype1}
    RotlType --> |i32| i32_rotl[返回 .i32_rotl]
    RotlType --> |i64| i64_rotl[返回 .i64_rotl]
    
    CheckOp --> |rotr| RotrType{检查 valtype1}
    RotrType --> |i32| i32_rotr[返回 .i32_rotr]
    RotrType --> |i64| i64_rotr[返回 .i64_rotr]
    
    CheckOp --> |abs| AbsType{检查 valtype1}
    AbsType --> |f32| f32_abs[返回 .f32_abs]
    AbsType --> |f64| f64_abs[返回 .f64_abs]
    
    CheckOp --> |neg| NegType{检查 valtype1}
    NegType --> |f32| f32_neg[返回 .f32_neg]
    NegType --> |f64| f64_neg[返回 .f64_neg]
    
    CheckOp --> |ceil| CeilType{检查 valtype1}
    CeilType --> |i32| f32_ceil[返回 .f32_ceil]
    CeilType --> |f32| f32_ceil[返回 .f32_ceil]
    CeilType --> |f64| f64_ceil[返回 .f64_ceil]
    
    CheckOp --> |floor| FloorType{检查 valtype1}
    FloorType --> |i32| f32_floor[返回 .f32_floor]
    FloorType --> |f32| f32_floor[返回 .f32_floor]
    FloorType --> |f64| f64_floor[返回 .f64_floor]
    
    CheckOp --> |trunc| TruncType{检查 valtype1}
    TruncType --> |i32| TruncCheckValtype2{检查 valtype2}
    TruncCheckValtype2 --> |f32| TruncSignedness_f32{检查 signedness}
    TruncSignedness_f32 --> |signed| i32_trunc_f32_s[返回 .i32_trunc_f32_s]
    TruncSignedness_f32 --> |unsigned| i32_trunc_f32_u[返回 .i32_trunc_f32_u]
    TruncCheckValtype2 --> |f64| TruncSignedness_f64{检查 signedness}
    TruncSignedness_f64 --> |signed| i32_trunc_f64_s[返回 .i32_trunc_f64_s]
    TruncSignedness_f64 --> |unsigned| i32_trunc_f64_u[返回 .i32_trunc_f64_u]
    TruncType --> |i64| TruncCheckValtype2_i64{检查 valtype2}
    TruncCheckValtype2_i64 --> |f32| TruncSignedness_f32_i64{检查 signedness}
    TruncSignedness_f32_i64 --> |signed| i64_trunc_f32_s[返回 .i64_trunc_f32_s]
    TruncSignedness_f32_i64 --> |unsigned| i64_trunc_f32_u[返回 .i64_trunc_f32_u]
    TruncCheckValtype2_i64 --> |f64| TruncSignedness_f64_i64{检查 signedness}
    TruncSignedness_f64_i64 --> |signed| i64_trunc_f64_s[返回 .i64_trunc_f64_s]
    TruncSignedness_f64_i64 --> |unsigned| i64_trunc_f64_u[返回 .i64_trunc_f64_u]
    TruncType --> |f32| f32_trunc[返回 .f32_trunc]
    TruncType --> |f64| f64_trunc[返回 .f64_trunc]
    
    CheckOp --> |nearest| NearestType{检查 valtype1}
    NearestType --> |f32| f32_nearest[返回 .f32_nearest]
    NearestType --> |f64| f64_nearest[返回 .f64_nearest]
    
    CheckOp --> |sqrt| SqrtType{检查 valtype1}
    SqrtType --> |f32| f32_sqrt[返回 .f32_sqrt]
    SqrtType --> |f64| f64_sqrt[返回 .f64_sqrt]
    
    CheckOp --> |min| MinType{检查 valtype1}
    MinType --> |f32| f32_min[返回 .f32_min]
    MinType --> |f64| f64_min[返回 .f64_min]
    
    CheckOp --> |max| MaxType{检查 valtype1}
    MaxType --> |f32| f32_max[返回 .f32_max]
    MaxType --> |f64| f64_max[返回 .f64_max]
    
    CheckOp --> |copysign| CopysignType{检查 valtype1}
    CopysignType --> |f32| f32_copysign[返回 .f32_copysign]
    CopysignType --> |f64| f64_copysign[返回 .f64_copysign]
    
    CheckOp --> |wrap| WrapType{检查 valtype1}
    WrapType --> |i32| WrapValtype2{检查 valtype2}
    WrapValtype2 --> |i64| i32_wrap_i64[返回 .i32_wrap_i64]
    
    CheckOp --> |convert| ConvertType{检查 valtype1}
    ConvertType --> |f32| ConvertValtype2_f32{检查 valtype2}
    ConvertValtype2_f32 --> |i32| ConvertSignedness_i32_f32{检查 signedness}
    ConvertSignedness_i32_f32 --> |signed| f32_convert_i32_s[返回 .f32_convert_i32_s]
    ConvertSignedness_i32_f32 --> |unsigned| f32_convert_i32_u[返回 .f32_convert_i32_u]
    ConvertValtype2_f32 --> |i64| ConvertSignedness_i64_f32{检查 signedness}
    ConvertSignedness_i64_f32 --> |signed| f32_convert_i64_s[返回 .f32_convert_i64_s]
    ConvertSignedness_i64_f32 --> |unsigned| f32_convert_i64_u[返回 .f32_convert_i64_u]
    ConvertType --> |f64| ConvertValtype2_f64{检查 valtype2}
    ConvertValtype2_f64 --> |i32| ConvertSignedness_i32_f64{检查 signedness}
    ConvertSignedness_i32_f64 --> |signed| f64_convert_i32_s[返回 .f64_convert_i32_s]
    ConvertSignedness_i32_f64 --> |unsigned| f64_convert_i32_u[返回 .f64_convert_i32_u]
    ConvertValtype2_f64 --> |i64| ConvertSignedness_i64_f64{检查 signedness}
    ConvertSignedness_i64_f64 --> |signed| f64_convert_i64_s[返回 .f64_convert_i64_s]
    ConvertSignedness_i64_f64 --> |unsigned| f64_convert_i64_u[返回 .f64_convert_i64_u]
    
    CheckOp --> |demote| DemoteCheck{valtype1=f32 and valtype2=f64}
    DemoteCheck --> YesDemote[f32_demote_f64[返回 .f32_demote_f64]]
    
    CheckOp --> |promote| PromoteCheck{valtype1=f64 and valtype2=f32}
    PromoteCheck --> YesPromote[f64_promote_f32[返回 .f64_promote_f32]]
    
    CheckOp --> |reinterpret| ReinterpretType{检查 valtype1}
    ReinterpretType --> |i32| ReinterpretValtype2_i32{检查 valtype2}
    ReinterpretValtype2_i32 --> |f32| i32_reinterpret_f32[返回 .i32_reinterpret_f32]
    ReinterpretType --> |i64| ReinterpretValtype2_i64{检查 valtype2}
    ReinterpretValtype2_i64 --> |f64| i64_reinterpret_f64[返回 .i64_reinterpret_f64]
    ReinterpretType --> |f32| ReinterpretValtype2_f32{检查 valtype2}
    ReinterpretValtype2_f32 --> |i32| f32_reinterpret_i32[返回 .f32_reinterpret_i32]
    ReinterpretType --> |f64| ReinterpretValtype2_f64{检查 valtype2}
    ReinterpretValtype2_f64 --> |i64| f64_reinterpret_i64[返回 .f64_reinterpret_i64]
    
    CheckOp --> |extend| ExtendType{检查 valtype1}
    ExtendType --> |i32| ExtendWidth_i32{检查 width}
    ExtendWidth_i32 --> |8| ExtendSignedness_8_i32{检查 signedness}
    ExtendSignedness_8_i32 --> |signed| i32_extend8_s[返回 .i32_extend8_s]
    ExtendWidth_i32 --> |16| ExtendSignedness_16_i32{检查 signedness}
    ExtendSignedness_16_i32 --> |signed| i32_extend16_s[返回 .i32_extend16_s]
    ExtendType --> |i64| ExtendWidth_i64{检查 width}
    ExtendWidth_i64 --> |8| ExtendSignedness_8_i64{检查 signedness}
    ExtendSignedness_8_i64 --> |signed| i64_extend8_s[返回 .i64_extend8_s]
    ExtendWidth_i64 --> |16| ExtendSignedness_16_i64{检查 signedness}
    ExtendSignedness_16_i64 --> |signed| i64_extend16_s[返回 .i64_extend16_s]
    ExtendWidth_i64 --> |32| ExtendSignedness_32_i64{检查 signedness}
    ExtendSignedness_32_i64 --> |signed| i64_extend32_s[返回 .i64_extend32_s]
