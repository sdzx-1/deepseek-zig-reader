```zig
fn analyzeBodyInner(
    sema: *Sema,
    block: *Block,
    body: []const Zir.Inst.Index,
) CompileError!void {
    // No tracy calls here, to avoid interfering with the tail call mechanism.

    try sema.inst_map.ensureSpaceForInstructions(sema.gpa, body);

    const pt = sema.pt;
    const zcu = pt.zcu;
    const map = &sema.inst_map;
    const tags = sema.code.instructions.items(.tag);
    const datas = sema.code.instructions.items(.data);

    var crash_info = crash_report.prepAnalyzeBody(sema, block, body);
    crash_info.push();
    defer crash_info.pop();

    // We use a while (true) loop here to avoid a redundant way of breaking out of
    // the loop. The only way to break out of the loop is with a `noreturn`
    // instruction.
    var i: u32 = 0;
    while (true) {
        crash_info.setBodyIndex(i);
        const inst = body[i];

        // The hashmap lookup in here is a little expensive, and LLVM fails to optimize it away.
        if (build_options.enable_logging) {
            std.log.scoped(.sema_zir).debug("sema ZIR {s} %{d}", .{ sub_file_path: {
                const file_index = block.src_base_inst.resolveFile(&zcu.intern_pool);
                const file = zcu.fileByIndex(file_index);
                break :sub_file_path file.sub_file_path;
            }, inst });
        }

        const air_inst: Air.Inst.Ref = inst: switch (tags[@intFromEnum(inst)]) {
            // zig fmt: off
            .alloc                        => try sema.zirAlloc(block, inst),
            .alloc_inferred               => try sema.zirAllocInferred(block, true),
            .alloc_inferred_mut           => try sema.zirAllocInferred(block, false),
            .alloc_inferred_comptime      => try sema.zirAllocInferredComptime(true),
            .alloc_inferred_comptime_mut  => try sema.zirAllocInferredComptime(false),
            .resolve_inferred_alloc       => try sema.zirResolveInferredAlloc(block, inst),
            .alloc_mut                    => try sema.zirAllocMut(block, inst),
            .alloc_comptime_mut           => try sema.zirAllocComptime(block, inst),
            .make_ptr_const               => try sema.zirMakePtrConst(block, inst),
            .anyframe_type                => try sema.zirAnyframeType(block, inst),
            .array_cat                    => try sema.zirArrayCat(block, inst),
            .array_mul                    => try sema.zirArrayMul(block, inst),
            .array_type                   => try sema.zirArrayType(block, inst),
            .array_type_sentinel          => try sema.zirArrayTypeSentinel(block, inst),
            .vector_type                  => try sema.zirVectorType(block, inst),
            .as_node                      => try sema.zirAsNode(block, inst),
            .as_shift_operand             => try sema.zirAsShiftOperand(block, inst),
            .bit_and                      => try sema.zirBitwise(block, inst, .bit_and),
            .bit_not                      => try sema.zirBitNot(block, inst),
            .bit_or                       => try sema.zirBitwise(block, inst, .bit_or),
            .bitcast                      => try sema.zirBitcast(block, inst),
            .suspend_block                => try sema.zirSuspendBlock(block, inst),
            .bool_not                     => try sema.zirBoolNot(block, inst),
            .bool_br_and                  => try sema.zirBoolBr(block, inst, false),
            .bool_br_or                   => try sema.zirBoolBr(block, inst, true),
            .c_import                     => try sema.zirCImport(block, inst),
            .call                         => try sema.zirCall(block, inst, .direct),
            .field_call                   => try sema.zirCall(block, inst, .field),
            .cmp_lt                       => try sema.zirCmp(block, inst, .lt),
            .cmp_lte                      => try sema.zirCmp(block, inst, .lte),
            .cmp_eq                       => try sema.zirCmpEq(block, inst, .eq, Air.Inst.Tag.fromCmpOp(.eq, block.float_mode == .optimized)),
            .cmp_gte                      => try sema.zirCmp(block, inst, .gte),
            .cmp_gt                       => try sema.zirCmp(block, inst, .gt),
            .cmp_neq                      => try sema.zirCmpEq(block, inst, .neq, Air.Inst.Tag.fromCmpOp(.neq, block.float_mode == .optimized)),
            .decl_ref                     => try sema.zirDeclRef(block, inst),
            .decl_val                     => try sema.zirDeclVal(block, inst),
            .load                         => try sema.zirLoad(block, inst),
            .elem_ptr                     => try sema.zirElemPtr(block, inst),
            .elem_ptr_node                => try sema.zirElemPtrNode(block, inst),
            .elem_val                     => try sema.zirElemVal(block, inst),
            .elem_val_node                => try sema.zirElemValNode(block, inst),
            .elem_val_imm                 => try sema.zirElemValImm(block, inst),
            .elem_type                    => try sema.zirElemType(block, inst),
            .indexable_ptr_elem_type      => try sema.zirIndexablePtrElemType(block, inst),
            .vec_arr_elem_type            => try sema.zirVecArrElemType(block, inst),
            .enum_literal                 => try sema.zirEnumLiteral(block, inst),
            .decl_literal                 => try sema.zirDeclLiteral(block, inst, true),
            .decl_literal_no_coerce       => try sema.zirDeclLiteral(block, inst, false),
            .int_from_enum                => try sema.zirIntFromEnum(block, inst),
            .enum_from_int                => try sema.zirEnumFromInt(block, inst),
            .err_union_code               => try sema.zirErrUnionCode(block, inst),
            .err_union_code_ptr           => try sema.zirErrUnionCodePtr(block, inst),
            .err_union_payload_unsafe     => try sema.zirErrUnionPayload(block, inst),
            .err_union_payload_unsafe_ptr => try sema.zirErrUnionPayloadPtr(block, inst),
            .error_union_type             => try sema.zirErrorUnionType(block, inst),
            .error_value                  => try sema.zirErrorValue(block, inst),
            .field_ptr                    => try sema.zirFieldPtr(block, inst),
            .field_ptr_named              => try sema.zirFieldPtrNamed(block, inst),
            .field_val                    => try sema.zirFieldVal(block, inst),
            .field_val_named              => try sema.zirFieldValNamed(block, inst),
            .func                         => try sema.zirFunc(block, inst, false),
            .func_inferred                => try sema.zirFunc(block, inst, true),
            .func_fancy                   => try sema.zirFuncFancy(block, inst),
            .import                       => try sema.zirImport(block, inst),
            .indexable_ptr_len            => try sema.zirIndexablePtrLen(block, inst),
            .int                          => try sema.zirInt(block, inst),
            .int_big                      => try sema.zirIntBig(block, inst),
            .float                        => try sema.zirFloat(block, inst),
            .float128                     => try sema.zirFloat128(block, inst),
            .int_type                     => try sema.zirIntType(inst),
            .is_non_err                   => try sema.zirIsNonErr(block, inst),
            .is_non_err_ptr               => try sema.zirIsNonErrPtr(block, inst),
            .ret_is_non_err               => try sema.zirRetIsNonErr(block, inst),
            .is_non_null                  => try sema.zirIsNonNull(block, inst),
            .is_non_null_ptr              => try sema.zirIsNonNullPtr(block, inst),
            .merge_error_sets             => try sema.zirMergeErrorSets(block, inst),
            .negate                       => try sema.zirNegate(block, inst),
            .negate_wrap                  => try sema.zirNegateWrap(block, inst),
            .optional_payload_safe        => try sema.zirOptionalPayload(block, inst, true),
            .optional_payload_safe_ptr    => try sema.zirOptionalPayloadPtr(block, inst, true),
            .optional_payload_unsafe      => try sema.zirOptionalPayload(block, inst, false),
            .optional_payload_unsafe_ptr  => try sema.zirOptionalPayloadPtr(block, inst, false),
            .optional_type                => try sema.zirOptionalType(block, inst),
            .ptr_type                     => try sema.zirPtrType(block, inst),
            .ref                          => try sema.zirRef(block, inst),
            .ret_err_value_code           => try sema.zirRetErrValueCode(inst),
            .shr                          => try sema.zirShr(block, inst, .shr),
            .shr_exact                    => try sema.zirShr(block, inst, .shr_exact),
            .slice_end                    => try sema.zirSliceEnd(block, inst),
            .slice_sentinel               => try sema.zirSliceSentinel(block, inst),
            .slice_start                  => try sema.zirSliceStart(block, inst),
            .slice_length                 => try sema.zirSliceLength(block, inst),
            .slice_sentinel_ty            => try sema.zirSliceSentinelTy(block, inst),
            .str                          => try sema.zirStr(inst),
            .switch_block                 => try sema.zirSwitchBlock(block, inst, false),
            .switch_block_ref             => try sema.zirSwitchBlock(block, inst, true),
            .switch_block_err_union       => try sema.zirSwitchBlockErrUnion(block, inst),
            .type_info                    => try sema.zirTypeInfo(block, inst),
            .size_of                      => try sema.zirSizeOf(block, inst),
            .bit_size_of                  => try sema.zirBitSizeOf(block, inst),
            .typeof                       => try sema.zirTypeof(block, inst),
            .typeof_builtin               => try sema.zirTypeofBuiltin(block, inst),
            .typeof_log2_int_type         => try sema.zirTypeofLog2IntType(block, inst),
            .xor                          => try sema.zirBitwise(block, inst, .xor),
            .struct_init_empty            => try sema.zirStructInitEmpty(block, inst),
            .struct_init_empty_result     => try sema.zirStructInitEmptyResult(block, inst, false),
            .struct_init_empty_ref_result => try sema.zirStructInitEmptyResult(block, inst, true),
            .struct_init_anon             => try sema.zirStructInitAnon(block, inst),
            .struct_init                  => try sema.zirStructInit(block, inst, false),
            .struct_init_ref              => try sema.zirStructInit(block, inst, true),
            .struct_init_field_type       => try sema.zirStructInitFieldType(block, inst),
            .struct_init_field_ptr        => try sema.zirStructInitFieldPtr(block, inst),
            .array_init_anon              => try sema.zirArrayInitAnon(block, inst),
            .array_init                   => try sema.zirArrayInit(block, inst, false),
            .array_init_ref               => try sema.zirArrayInit(block, inst, true),
            .array_init_elem_type         => try sema.zirArrayInitElemType(block, inst),
            .array_init_elem_ptr          => try sema.zirArrayInitElemPtr(block, inst),
            .union_init                   => try sema.zirUnionInit(block, inst),
            .field_type_ref               => try sema.zirFieldTypeRef(block, inst),
            .int_from_ptr                 => try sema.zirIntFromPtr(block, inst),
            .align_of                     => try sema.zirAlignOf(block, inst),
            .int_from_bool                => try sema.zirIntFromBool(block, inst),
            .embed_file                   => try sema.zirEmbedFile(block, inst),
            .error_name                   => try sema.zirErrorName(block, inst),
            .tag_name                     => try sema.zirTagName(block, inst),
            .type_name                    => try sema.zirTypeName(block, inst),
            .frame_type                   => try sema.zirFrameType(block, inst),
            .frame_size                   => try sema.zirFrameSize(block, inst),
            .int_from_float               => try sema.zirIntFromFloat(block, inst),
            .float_from_int               => try sema.zirFloatFromInt(block, inst),
            .ptr_from_int                 => try sema.zirPtrFromInt(block, inst),
            .float_cast                   => try sema.zirFloatCast(block, inst),
            .int_cast                     => try sema.zirIntCast(block, inst),
            .ptr_cast                     => try sema.zirPtrCast(block, inst),
            .truncate                     => try sema.zirTruncate(block, inst),
            .has_decl                     => try sema.zirHasDecl(block, inst),
            .has_field                    => try sema.zirHasField(block, inst),
            .byte_swap                    => try sema.zirByteSwap(block, inst),
            .bit_reverse                  => try sema.zirBitReverse(block, inst),
            .bit_offset_of                => try sema.zirBitOffsetOf(block, inst),
            .offset_of                    => try sema.zirOffsetOf(block, inst),
            .splat                        => try sema.zirSplat(block, inst),
            .reduce                       => try sema.zirReduce(block, inst),
            .shuffle                      => try sema.zirShuffle(block, inst),
            .atomic_load                  => try sema.zirAtomicLoad(block, inst),
            .atomic_rmw                   => try sema.zirAtomicRmw(block, inst),
            .mul_add                      => try sema.zirMulAdd(block, inst),
            .builtin_call                 => try sema.zirBuiltinCall(block, inst),
            .@"resume"                    => try sema.zirResume(block, inst),
            .@"await"                     => try sema.zirAwait(block, inst),
            .for_len                      => try sema.zirForLen(block, inst),
            .validate_array_init_ref_ty   => try sema.zirValidateArrayInitRefTy(block, inst),
            .opt_eu_base_ptr_init         => try sema.zirOptEuBasePtrInit(block, inst),
            .coerce_ptr_elem_ty           => try sema.zirCoercePtrElemTy(block, inst),

            .clz       => try sema.zirBitCount(block, inst, .clz,      Value.clz),
            .ctz       => try sema.zirBitCount(block, inst, .ctz,      Value.ctz),
            .pop_count => try sema.zirBitCount(block, inst, .popcount, Value.popCount),
            .abs       => try sema.zirAbs(block, inst),

            .sqrt  => try sema.zirUnaryMath(block, inst, .sqrt, Value.sqrt),
            .sin   => try sema.zirUnaryMath(block, inst, .sin, Value.sin),
            .cos   => try sema.zirUnaryMath(block, inst, .cos, Value.cos),
            .tan   => try sema.zirUnaryMath(block, inst, .tan, Value.tan),
            .exp   => try sema.zirUnaryMath(block, inst, .exp, Value.exp),
            .exp2  => try sema.zirUnaryMath(block, inst, .exp2, Value.exp2),
            .log   => try sema.zirUnaryMath(block, inst, .log, Value.log),
            .log2  => try sema.zirUnaryMath(block, inst, .log2, Value.log2),
            .log10 => try sema.zirUnaryMath(block, inst, .log10, Value.log10),
            .floor => try sema.zirUnaryMath(block, inst, .floor, Value.floor),
            .ceil  => try sema.zirUnaryMath(block, inst, .ceil, Value.ceil),
            .round => try sema.zirUnaryMath(block, inst, .round, Value.round),
            .trunc => try sema.zirUnaryMath(block, inst, .trunc_float, Value.trunc),

            .error_set_decl => try sema.zirErrorSetDecl(inst),

            .add        => try sema.zirArithmetic(block, inst, .add,        true),
            .addwrap    => try sema.zirArithmetic(block, inst, .addwrap,    true),
            .add_sat    => try sema.zirArithmetic(block, inst, .add_sat,    true),
            .add_unsafe => try sema.zirArithmetic(block, inst, .add_unsafe, false),
            .mul        => try sema.zirArithmetic(block, inst, .mul,        true),
            .mulwrap    => try sema.zirArithmetic(block, inst, .mulwrap,    true),
            .mul_sat    => try sema.zirArithmetic(block, inst, .mul_sat,    true),
            .sub        => try sema.zirArithmetic(block, inst, .sub,        true),
            .subwrap    => try sema.zirArithmetic(block, inst, .subwrap,    true),
            .sub_sat    => try sema.zirArithmetic(block, inst, .sub_sat,    true),

            .div       => try sema.zirDiv(block, inst),
            .div_exact => try sema.zirDivExact(block, inst),
            .div_floor => try sema.zirDivFloor(block, inst),
            .div_trunc => try sema.zirDivTrunc(block, inst),

            .mod_rem => try sema.zirModRem(block, inst),
            .mod     => try sema.zirMod(block, inst),
            .rem     => try sema.zirRem(block, inst),

            .max => try sema.zirMinMax(block, inst, .max),
            .min => try sema.zirMinMax(block, inst, .min),

            .shl       => try sema.zirShl(block, inst, .shl),
            .shl_exact => try sema.zirShl(block, inst, .shl_exact),
            .shl_sat   => try sema.zirShl(block, inst, .shl_sat),

            .ret_ptr  => try sema.zirRetPtr(block, inst),
            .ret_type => Air.internedToRef(sema.fn_ret_ty.toIntern()),

            // Instructions that we know to *always* be noreturn based solely on their tag.
            // These functions match the return type of analyzeBody so that we can
            // tail call them here.
            .compile_error  => break try sema.zirCompileError(block, inst),
            .ret_implicit   => break try sema.zirRetImplicit(block, inst),
            .ret_node       => break try sema.zirRetNode(block, inst),
            .ret_load       => break try sema.zirRetLoad(block, inst),
            .ret_err_value  => break try sema.zirRetErrValue(block, inst),
            .@"unreachable" => break try sema.zirUnreachable(block, inst),
            .panic          => break try sema.zirPanic(block, inst),
            .trap           => break try sema.zirTrap(block, inst),
            // zig fmt: on

            // This instruction never exists in an analyzed body. It exists only in the declaration
            // list for a container type.
            .declaration => unreachable,

            .extended => ext: {
                const extended = datas[@intFromEnum(inst)].extended;
                break :ext switch (extended.opcode) {
                    // zig fmt: off
                    .struct_decl        => try sema.zirStructDecl(        block, extended, inst),
                    .enum_decl          => try sema.zirEnumDecl(          block, extended, inst),
                    .union_decl         => try sema.zirUnionDecl(         block, extended, inst),
                    .opaque_decl        => try sema.zirOpaqueDecl(        block, extended, inst),
                    .tuple_decl         => try sema.zirTupleDecl(         block, extended),
                    .this               => try sema.zirThis(              block, extended),
                    .ret_addr           => try sema.zirRetAddr(           block, extended),
                    .builtin_src        => try sema.zirBuiltinSrc(        block, extended),
                    .error_return_trace => try sema.zirErrorReturnTrace(  block),
                    .frame              => try sema.zirFrame(             block, extended),
                    .frame_address      => try sema.zirFrameAddress(      block, extended),
                    .alloc              => try sema.zirAllocExtended(     block, extended),
                    .builtin_extern     => try sema.zirBuiltinExtern(     block, extended),
                    .@"asm"             => try sema.zirAsm(               block, extended, false),
                    .asm_expr           => try sema.zirAsm(               block, extended, true),
                    .typeof_peer        => try sema.zirTypeofPeer(        block, extended, inst),
                    .compile_log        => try sema.zirCompileLog(        block, extended),
                    .min_multi          => try sema.zirMinMaxMulti(       block, extended, .min),
                    .max_multi          => try sema.zirMinMaxMulti(       block, extended, .max),
                    .add_with_overflow  => try sema.zirOverflowArithmetic(block, extended, extended.opcode),
                    .sub_with_overflow  => try sema.zirOverflowArithmetic(block, extended, extended.opcode),
                    .mul_with_overflow  => try sema.zirOverflowArithmetic(block, extended, extended.opcode),
                    .shl_with_overflow  => try sema.zirOverflowArithmetic(block, extended, extended.opcode),
                    .c_undef            => try sema.zirCUndef(            block, extended),
                    .c_include          => try sema.zirCInclude(          block, extended),
                    .c_define           => try sema.zirCDefine(           block, extended),
                    .wasm_memory_size   => try sema.zirWasmMemorySize(    block, extended),
                    .wasm_memory_grow   => try sema.zirWasmMemoryGrow(    block, extended),
                    .prefetch           => try sema.zirPrefetch(          block, extended),
                    .error_cast         => try sema.zirErrorCast(         block, extended),
                    .await_nosuspend    => try sema.zirAwaitNosuspend(    block, extended),
                    .select             => try sema.zirSelect(            block, extended),
                    .int_from_error     => try sema.zirIntFromError(      block, extended),
                    .error_from_int     => try sema.zirErrorFromInt(      block, extended),
                    .reify              => try sema.zirReify(             block, extended, inst),
                    .builtin_async_call => try sema.zirBuiltinAsyncCall(  block, extended),
                    .cmpxchg            => try sema.zirCmpxchg(           block, extended),
                    .c_va_arg           => try sema.zirCVaArg(            block, extended),
                    .c_va_copy          => try sema.zirCVaCopy(           block, extended),
                    .c_va_end           => try sema.zirCVaEnd(            block, extended),
                    .c_va_start         => try sema.zirCVaStart(          block, extended),
                    .ptr_cast_full      => try sema.zirPtrCastFull(       block, extended),
                    .ptr_cast_no_dest   => try sema.zirPtrCastNoDest(     block, extended),
                    .work_item_id       => try sema.zirWorkItem(          block, extended, extended.opcode),
                    .work_group_size    => try sema.zirWorkItem(          block, extended, extended.opcode),
                    .work_group_id      => try sema.zirWorkItem(          block, extended, extended.opcode),
                    .in_comptime        => try sema.zirInComptime(        block),
                    .closure_get        => try sema.zirClosureGet(        block, extended),
                    // zig fmt: on

                    .set_float_mode => {
                        try sema.zirSetFloatMode(block, extended);
                        i += 1;
                        continue;
                    },
                    .breakpoint => {
                        if (!block.isComptime()) {
                            _ = try block.addNoOp(.breakpoint);
                        }
                        i += 1;
                        continue;
                    },
                    .disable_instrumentation => {
                        try sema.zirDisableInstrumentation();
                        i += 1;
                        continue;
                    },
                    .disable_intrinsics => {
                        try sema.zirDisableIntrinsics();
                        i += 1;
                        continue;
                    },
                    .restore_err_ret_index => {
                        try sema.zirRestoreErrRetIndex(block, extended);
                        i += 1;
                        continue;
                    },
                    .branch_hint => {
                        try sema.zirBranchHint(block, extended);
                        i += 1;
                        continue;
                    },
                    .value_placeholder => unreachable, // never appears in a body
                    .field_parent_ptr => try sema.zirFieldParentPtr(block, extended),
                    .builtin_value => try sema.zirBuiltinValue(block, extended),
                    .inplace_arith_result_ty => try sema.zirInplaceArithResultTy(extended),
                    .dbg_empty_stmt => {
                        try sema.zirDbgEmptyStmt(block, inst);
                        i += 1;
                        continue;
                    },
                    .astgen_error => return error.AnalysisFail,
                };
            },

            // Instructions that we know can *never* be noreturn based solely on
            // their tag. We avoid needlessly checking if they are noreturn and
            // continue the loop.
            // We also know that they cannot be referenced later, so we avoid
            // putting them into the map.
            .dbg_stmt => {
                try sema.zirDbgStmt(block, inst);
                i += 1;
                continue;
            },
            .dbg_var_ptr => {
                try sema.zirDbgVar(block, inst, .dbg_var_ptr);
                i += 1;
                continue;
            },
            .dbg_var_val => {
                try sema.zirDbgVar(block, inst, .dbg_var_val);
                i += 1;
                continue;
            },
            .ensure_err_union_payload_void => {
                try sema.zirEnsureErrUnionPayloadVoid(block, inst);
                i += 1;
                continue;
            },
            .ensure_result_non_error => {
                try sema.zirEnsureResultNonError(block, inst);
                i += 1;
                continue;
            },
            .ensure_result_used => {
                try sema.zirEnsureResultUsed(block, inst);
                i += 1;
                continue;
            },
            .set_eval_branch_quota => {
                try sema.zirSetEvalBranchQuota(block, inst);
                i += 1;
                continue;
            },
            .atomic_store => {
                try sema.zirAtomicStore(block, inst);
                i += 1;
                continue;
            },
            .store_node => {
                try sema.zirStoreNode(block, inst);
                i += 1;
                continue;
            },
            .store_to_inferred_ptr => {
                try sema.zirStoreToInferredPtr(block, inst);
                i += 1;
                continue;
            },
            .validate_struct_init_ty => {
                try sema.zirValidateStructInitTy(block, inst, false);
                i += 1;
                continue;
            },
            .validate_struct_init_result_ty => {
                try sema.zirValidateStructInitTy(block, inst, true);
                i += 1;
                continue;
            },
            .validate_array_init_ty => {
                try sema.zirValidateArrayInitTy(block, inst, false);
                i += 1;
                continue;
            },
            .validate_array_init_result_ty => {
                try sema.zirValidateArrayInitTy(block, inst, true);
                i += 1;
                continue;
            },
            .validate_ptr_struct_init => {
                try sema.zirValidatePtrStructInit(block, inst);
                i += 1;
                continue;
            },
            .validate_ptr_array_init => {
                try sema.zirValidatePtrArrayInit(block, inst);
                i += 1;
                continue;
            },
            .validate_deref => {
                try sema.zirValidateDeref(block, inst);
                i += 1;
                continue;
            },
            .validate_destructure => {
                try sema.zirValidateDestructure(block, inst);
                i += 1;
                continue;
            },
            .validate_ref_ty => {
                try sema.zirValidateRefTy(block, inst);
                i += 1;
                continue;
            },
            .validate_const => {
                try sema.zirValidateConst(block, inst);
                i += 1;
                continue;
            },
            .@"export" => {
                try sema.zirExport(block, inst);
                i += 1;
                continue;
            },
            .set_runtime_safety => {
                try sema.zirSetRuntimeSafety(block, inst);
                i += 1;
                continue;
            },
            .param => {
                try sema.zirParam(block, inst, false);
                i += 1;
                continue;
            },
            .param_comptime => {
                try sema.zirParam(block, inst, true);
                i += 1;
                continue;
            },
            .param_anytype => {
                try sema.zirParamAnytype(block, inst, false);
                i += 1;
                continue;
            },
            .param_anytype_comptime => {
                try sema.zirParamAnytype(block, inst, true);
                i += 1;
                continue;
            },
            .memcpy => {
                try sema.zirMemcpy(block, inst);
                i += 1;
                continue;
            },
            .memset => {
                try sema.zirMemset(block, inst);
                i += 1;
                continue;
            },
            .check_comptime_control_flow => {
                if (!block.isComptime()) {
                    const inst_data = sema.code.instructions.items(.data)[@intFromEnum(inst)].un_node;
                    const src = block.nodeOffset(inst_data.src_node);
                    const inline_block = inst_data.operand.toIndex().?;

                    var check_block = block;
                    const target_runtime_index = while (true) {
                        if (check_block.inline_block == inline_block.toOptional()) {
                            break check_block.runtime_index;
                        }
                        check_block = check_block.parent.?;
                    };

                    if (@intFromEnum(target_runtime_index) < @intFromEnum(block.runtime_index)) {
                        const runtime_src = block.runtime_cond orelse block.runtime_loop.?;
                        const msg = msg: {
                            const msg = try sema.errMsg(src, "comptime control flow inside runtime block", .{});
                            errdefer msg.destroy(sema.gpa);
                            try sema.errNote(runtime_src, msg, "runtime control flow here", .{});
                            break :msg msg;
                        };
                        return sema.failWithOwnedErrorMsg(block, msg);
                    }
                }
                i += 1;
                continue;
            },
            .save_err_ret_index => {
                try sema.zirSaveErrRetIndex(block, inst);
                i += 1;
                continue;
            },
            .restore_err_ret_index_unconditional => {
                const un_node = datas[@intFromEnum(inst)].un_node;
                try sema.restoreErrRetIndex(block, block.nodeOffset(un_node.src_node), un_node.operand, .none);
                i += 1;
                continue;
            },
            .restore_err_ret_index_fn_entry => {
                const un_node = datas[@intFromEnum(inst)].un_node;
                try sema.restoreErrRetIndex(block, block.nodeOffset(un_node.src_node), .none, un_node.operand);
                i += 1;
                continue;
            },

            // Special case instructions to handle comptime control flow.
            .@"break" => {
                if (block.isComptime()) {
                    sema.comptime_break_inst = inst;
                    return error.ComptimeBreak;
                } else {
                    try sema.zirBreak(block, inst);
                    break;
                }
            },
            .break_inline => {
                sema.comptime_break_inst = inst;
                return error.ComptimeBreak;
            },
            .repeat => {
                if (block.isComptime()) {
                    // Send comptime control flow back to the beginning of this block.
                    const src = block.nodeOffset(datas[@intFromEnum(inst)].node);
                    try sema.emitBackwardBranch(block, src);
                    i = 0;
                    continue;
                } else {
                    // We are definitely called by `zirLoop`, which will treat the
                    // fact that this body does not terminate `noreturn` as an
                    // implicit repeat.
                    // TODO: since AIR has `repeat` now, we could change ZIR to generate
                    // more optimal code utilizing `repeat` instructions across blocks!
                    break;
                }
            },
            .repeat_inline => {
                // Send comptime control flow back to the beginning of this block.
                const src = block.nodeOffset(datas[@intFromEnum(inst)].node);
                try sema.emitBackwardBranch(block, src);
                i = 0;
                continue;
            },
            .switch_continue => if (block.isComptime()) {
                sema.comptime_break_inst = inst;
                return error.ComptimeBreak;
            } else {
                try sema.zirSwitchContinue(block, inst);
                break;
            },

            .loop => if (block.isComptime()) {
                continue :inst .block_inline;
            } else try sema.zirLoop(block, inst),

            .block => if (block.isComptime()) {
                continue :inst .block_inline;
            } else try sema.zirBlock(block, inst),

            .block_comptime => {
                const pl_node = datas[@intFromEnum(inst)].pl_node;
                const src = block.nodeOffset(pl_node.src_node);
                const extra = sema.code.extraData(Zir.Inst.BlockComptime, pl_node.payload_index);
                const block_body = sema.code.bodySlice(extra.end, extra.data.body_len);

                var child_block = block.makeSubBlock();
                defer child_block.instructions.deinit(sema.gpa);

                // We won't have any merges, but we must ensure this block is properly labeled for
                // any `.restore_err_ret_index_*` instructions.
                var label: Block.Label = .{
                    .zir_block = inst,
                    .merges = undefined,
                };
                child_block.label = &label;

                child_block.comptime_reason = .{ .reason = .{
                    .src = src,
                    .r = .{ .simple = extra.data.reason },
                } };

                const result = try sema.resolveInlineBody(&child_block, block_body, inst);

                // Only check for the result being comptime-known in the outermost `block_comptime`.
                // That way, AstGen can safely elide redundant `block_comptime` without affecting semantics.
                if (!block.isComptime() and !try sema.isComptimeKnown(result)) {
                    return sema.failWithNeededComptime(&child_block, src, null);
                }

                break :inst result;
            },

            .block_inline => blk: {
                // Directly analyze the block body without introducing a new block.
                // However, in the case of a corresponding break_inline which reaches
                // through a runtime conditional branch, we must retroactively emit
                // a block, so we remember the block index here just in case.
                const block_index = block.instructions.items.len;
                const inst_data = datas[@intFromEnum(inst)].pl_node;
                const extra = sema.code.extraData(Zir.Inst.Block, inst_data.payload_index);
                const inline_body = sema.code.bodySlice(extra.end, extra.data.body_len);
                const gpa = sema.gpa;

                const BreakResult = struct {
                    block_inst: Zir.Inst.Index,
                    operand: Zir.Inst.Ref,
                };

                const opt_break_data: ?BreakResult, const need_debug_scope = b: {
                    // Create a temporary child block so that this inline block is properly
                    // labeled for any .restore_err_ret_index instructions
                    var child_block = block.makeSubBlock();
                    var need_debug_scope = false;
                    child_block.need_debug_scope = &need_debug_scope;

                    // If this block contains a function prototype, we need to reset the
                    // current list of parameters and restore it later.
                    // Note: this probably needs to be resolved in a more general manner.
                    const tag_index = @intFromEnum(inline_body[inline_body.len - 1]);
                    child_block.inline_block = (if (tags[tag_index] == .repeat_inline)
                        inline_body[0]
                    else
                        inst).toOptional();

                    var label: Block.Label = .{
                        .zir_block = inst,
                        .merges = undefined,
                    };
                    child_block.label = &label;

                    // Write these instructions directly into the parent block
                    child_block.instructions = block.instructions;
                    defer block.instructions = child_block.instructions;

                    const break_result: ?BreakResult = if (sema.analyzeBodyInner(&child_block, inline_body)) |_| r: {
                        break :r null;
                    } else |err| switch (err) {
                        error.ComptimeBreak => brk_res: {
                            const break_inst = sema.comptime_break_inst;
                            const break_data = sema.code.instructions.items(.data)[@intFromEnum(break_inst)].@"break";
                            const break_extra = sema.code.extraData(Zir.Inst.Break, break_data.payload_index).data;
                            break :brk_res .{
                                .block_inst = break_extra.block_inst,
                                .operand = break_data.operand,
                            };
                        },
                        else => |e| return e,
                    };

                    if (need_debug_scope) {
                        _ = try sema.ensurePostHoc(block, inst);
                    }

                    break :b .{ break_result, need_debug_scope };
                };

                // A runtime conditional branch that needs a post-hoc block to be
                // emitted communicates this by mapping the block index into the inst map.
                if (map.get(inst)) |new_block_ref| ph: {
                    // Comptime control flow populates the map, so we don't actually know
                    // if this is a post-hoc runtime block until we check the
                    // post_hoc_block map.
                    const new_block_inst = new_block_ref.toIndex() orelse break :ph;
                    const labeled_block = sema.post_hoc_blocks.get(new_block_inst) orelse
                        break :ph;

                    // In this case we need to move all the instructions starting at
                    // block_index from the current block into this new one.

                    if (opt_break_data) |break_data| {
                        // This is a comptime break which we now change to a runtime break
                        // since it crosses a runtime branch.
                        // It may pass through our currently being analyzed block_inline or it
                        // may point directly to it. In the latter case, this modifies the
                        // block that we looked up in the post_hoc_blocks map above.
                        try sema.addRuntimeBreak(block, break_data.block_inst, break_data.operand);
                    }

                    try labeled_block.block.instructions.appendSlice(gpa, block.instructions.items[block_index..]);
                    block.instructions.items.len = block_index;

                    const block_result = try sema.resolveAnalyzedBlock(block, block.nodeOffset(inst_data.src_node), &labeled_block.block, &labeled_block.label.merges, need_debug_scope);
                    {
                        // Destroy the ad-hoc block entry so that it does not interfere with
                        // the next iteration of comptime control flow, if any.
                        labeled_block.destroy(gpa);
                        assert(sema.post_hoc_blocks.remove(new_block_inst));
                    }

                    break :blk block_result;
                }

                const break_data = opt_break_data orelse break;
                if (inst == break_data.block_inst) {
                    break :blk try sema.resolveInst(break_data.operand);
                } else {
                    // `comptime_break_inst` preserved from `analyzeBodyInner` above.
                    return error.ComptimeBreak;
                }
            },
            .condbr => if (block.isComptime()) {
                continue :inst .condbr_inline;
            } else {
                try sema.zirCondbr(block, inst);
                break;
            },
            .condbr_inline => {
                const inst_data = datas[@intFromEnum(inst)].pl_node;
                const cond_src = block.src(.{ .node_offset_if_cond = inst_data.src_node });
                const extra = sema.code.extraData(Zir.Inst.CondBr, inst_data.payload_index);
                const then_body = sema.code.bodySlice(extra.end, extra.data.then_body_len);
                const else_body = sema.code.bodySlice(
                    extra.end + then_body.len,
                    extra.data.else_body_len,
                );
                const uncasted_cond = try sema.resolveInst(extra.data.condition);
                const cond = try sema.coerce(block, Type.bool, uncasted_cond, cond_src);
                const cond_val = try sema.resolveConstDefinedValue(
                    block,
                    cond_src,
                    cond,
                    // If this block is comptime, it's more helpful to just give the outer message.
                    // This is particularly true if this came from a comptime `condbr` above.
                    if (block.isComptime()) null else .{ .simple = .inline_loop_operand },
                );
                const inline_body = if (cond_val.toBool()) then_body else else_body;

                try sema.maybeErrorUnwrapCondbr(block, inline_body, extra.data.condition, cond_src);
                const old_runtime_index = block.runtime_index;
                defer block.runtime_index = old_runtime_index;

                const result = try sema.analyzeInlineBody(block, inline_body, inst) orelse break;
                break :inst result;
            },
            .@"try" => blk: {
                if (!block.isComptime()) break :blk try sema.zirTry(block, inst);
                const inst_data = sema.code.instructions.items(.data)[@intFromEnum(inst)].pl_node;
                const src = block.nodeOffset(inst_data.src_node);
                const operand_src = block.src(.{ .node_offset_try_operand = inst_data.src_node });
                const extra = sema.code.extraData(Zir.Inst.Try, inst_data.payload_index);
                const inline_body = sema.code.bodySlice(extra.end, extra.data.body_len);
                const err_union = try sema.resolveInst(extra.data.operand);
                const err_union_ty = sema.typeOf(err_union);
                if (err_union_ty.zigTypeTag(zcu) != .error_union) {
                    return sema.fail(block, operand_src, "expected error union type, found '{}'", .{
                        err_union_ty.fmt(pt),
                    });
                }
                const is_non_err = try sema.analyzeIsNonErrComptimeOnly(block, operand_src, err_union);
                assert(is_non_err != .none);
                const is_non_err_val = try sema.resolveConstDefinedValue(block, operand_src, is_non_err, null);
                if (is_non_err_val.toBool()) {
                    break :blk try sema.analyzeErrUnionPayload(block, src, err_union_ty, err_union, operand_src, false);
                }
                const result = try sema.analyzeInlineBody(block, inline_body, inst) orelse break;
                break :blk result;
            },
            .try_ptr => blk: {
                if (!block.isComptime()) break :blk try sema.zirTryPtr(block, inst);
                const inst_data = sema.code.instructions.items(.data)[@intFromEnum(inst)].pl_node;
                const src = block.nodeOffset(inst_data.src_node);
                const operand_src = block.src(.{ .node_offset_try_operand = inst_data.src_node });
                const extra = sema.code.extraData(Zir.Inst.Try, inst_data.payload_index);
                const inline_body = sema.code.bodySlice(extra.end, extra.data.body_len);
                const operand = try sema.resolveInst(extra.data.operand);
                const err_union = try sema.analyzeLoad(block, src, operand, operand_src);
                const is_non_err = try sema.analyzeIsNonErrComptimeOnly(block, operand_src, err_union);
                assert(is_non_err != .none);
                const is_non_err_val = try sema.resolveConstDefinedValue(block, operand_src, is_non_err, null);
                if (is_non_err_val.toBool()) {
                    break :blk try sema.analyzeErrUnionPayloadPtr(block, src, operand, false, false);
                }
                const result = try sema.analyzeInlineBody(block, inline_body, inst) orelse break;
                break :blk result;
            },
            .@"defer" => blk: {
                const inst_data = sema.code.instructions.items(.data)[@intFromEnum(inst)].@"defer";
                const defer_body = sema.code.bodySlice(inst_data.index, inst_data.len);
                if (sema.analyzeBodyInner(block, defer_body)) |_| {
                    // The defer terminated noreturn - no more analysis needed.
                    break;
                } else |err| switch (err) {
                    error.ComptimeBreak => {},
                    else => |e| return e,
                }
                if (sema.comptime_break_inst != defer_body[defer_body.len - 1]) {
                    return error.ComptimeBreak;
                }
                break :blk .void_value;
            },
            .defer_err_code => blk: {
                const inst_data = sema.code.instructions.items(.data)[@intFromEnum(inst)].defer_err_code;
                const extra = sema.code.extraData(Zir.Inst.DeferErrCode, inst_data.payload_index).data;
                const defer_body = sema.code.bodySlice(extra.index, extra.len);
                const err_code = try sema.resolveInst(inst_data.err_code);
                try map.ensureSpaceForInstructions(sema.gpa, defer_body);
                map.putAssumeCapacity(extra.remapped_err_code, err_code);
                if (sema.analyzeBodyInner(block, defer_body)) |_| {
                    // The defer terminated noreturn - no more analysis needed.
                    break;
                } else |err| switch (err) {
                    error.ComptimeBreak => {},
                    else => |e| return e,
                }
                if (sema.comptime_break_inst != defer_body[defer_body.len - 1]) {
                    return error.ComptimeBreak;
                }
                break :blk .void_value;
            },
        };
        if (sema.isNoReturn(air_inst)) {
            // We're going to assume that the body itself is noreturn, so let's ensure that now
            assert(block.instructions.items.len > 0);
            assert(sema.isNoReturn(block.instructions.items[block.instructions.items.len - 1].toRef()));
            break;
        }
        map.putAssumeCapacity(inst, air_inst);
        i += 1;
    }
}```
嗯，我现在要总结一下这段Zig代码的思维模型。首先，这个函数叫做analyzeBodyInner，看起来是语义分析的一部分，处理ZIR（Zig Intermediate Representation）指令。代码里有很多switch语句，根据不同的ZIR指令标签调用不同的处理函数，比如zirAlloc、zirBitwise等等。

首先，函数参数有sema（语义分析器）、block（当前的基本块）、body（ZIR指令的数组）。函数的主要结构是一个无限循环，遍历body中的每个指令，并根据指令类型进行处理。每个指令处理完后，结果会被存入inst_map中。

代码中有很多条件分支，处理不同的指令类型。例如，遇到.compile_error或.ret_implicit等指令时，会中断循环并返回相应的结果。还有些指令需要在编译时处理，比如comptime控制流，这部分可能会涉及到错误处理或跳转。

另外，代码里有很多关于错误处理的部分，比如使用try来处理可能的错误，并在遇到错误时返回。还有一些处理块（block）的逻辑，比如创建子块（subBlock），处理内联块（inline block），以及处理循环和条件分支。

需要注意的是，代码中有很多关于编译时（comptime）和运行时（runtime）的逻辑区分。例如，当block是comptime时，会执行不同的处理路径，可能涉及到生成编译时的值或进行类型检查。

总的来说，这个函数的作用是将ZIR指令转换为AIR（Abstract Intermediate Representation）指令，同时进行语义检查和编译时计算。处理过程中需要考虑控制流、错误处理、类型推导以及编译时与运行时的不同行为。
================================================
这段代码是Zig编译器语义分析阶段的核心部分，负责将ZIR（Zig Intermediate Representation）指令转换为AIR（Abstract Intermediate Representation），同时进行类型推导、控制流分析和编译时计算。其思维模型可总结如下：

---

### **核心目标**
将ZIR指令逐条解析为AIR指令，完成以下任务：
1. **语义验证**（类型匹配、操作合法性检查等）。
2. **编译时计算**（如常量折叠、类型推导）。
3. **控制流处理**（循环、条件分支、错误处理）。
4. **符号映射**（将ZIR指令结果映射到AIR指令，存入`inst_map`）。

---

### **关键逻辑**
1. **指令分派**  
   通过`switch (tags)`匹配ZIR指令类型，调用对应的`zirXxx`函数（如`zirAlloc`、`zirCmp`等）。每个函数负责生成AIR指令并执行语义分析。

2. **控制流处理**  
   - **循环与分支**：通过`while (true)`遍历指令，遇到`noreturn`指令（如`.compile_error`）时终止循环。
   - **编译时控制流**：通过`block.isComptime()`区分编译时与运行时逻辑。例如：
     - `comptime_break`触发`error.ComptimeBreak`，回退到上层处理。
     - 内联块（`.block_inline`）直接合并到父块，避免生成冗余AIR块。

3. **错误处理**  
   - 使用`try`捕获可能的错误（如类型不匹配）。
   - 通过`crash_info`记录错误上下文，确保错误信息包含源码位置。

4. **编译时优化**  
   - **常量传播**：如`resolveConstDefinedValue`解析编译时常量。
   - **类型推导**：如`typeOf`获取操作数类型，验证操作合法性。

5. **符号映射**  
   - `inst_map`记录ZIR指令到AIR指令的映射，确保后续指令能引用先前结果。

---

### **特殊处理**
- **`noreturn`指令**：如`.unreachable`或`.ret`会终止当前块的控制流。
- **内联块（Inline Block）**：直接展开到父块中，减少运行时开销。
- **错误联合类型**：处理`try`和`err_union`时，需验证错误是否可忽略。
- **调试信息**：如`.dbg_stmt`生成调试符号，支持源码级调试。

---

### **运行时与编译时分离**
- **编译时逻辑**（`block.isComptime()`为`true`）：
  - 执行常量折叠、类型检查。
  - 处理`comptime`控制流（如`break_inline`）。
- **运行时逻辑**：
  - 生成实际机器码相关的AIR指令（如内存分配、循环）。

---

### **性能优化**
- **避免冗余操作**：如`build_options.enable_logging`控制调试日志。
- **尾调用优化**：注释提到“No tracy calls here, to avoid interfering with the tail call mechanism”。

---

### **总结**
此函数是Zig编译器将高阶ZIR指令转换为低阶AIR的核心枢纽，通过精细的分派逻辑、编译时/运行时分离、错误处理和控制流管理，确保代码的语义正确性和执行效率。其设计充分体现了Zig对编译时元编程和零成本抽象的重视。