```zig
fn zirSwitchBlock(sema: *Sema, block: *Block, inst: Zir.Inst.Index, operand_is_ref: bool) CompileError!Air.Inst.Ref {
    const tracy = trace(@src());
    defer tracy.end();

    const pt = sema.pt;
    const zcu = pt.zcu;
    const gpa = sema.gpa;
    const inst_data = sema.code.instructions.items(.data)[@intFromEnum(inst)].pl_node;
    const src = block.nodeOffset(inst_data.src_node);
    const src_node_offset = inst_data.src_node;
    const operand_src = block.src(.{ .node_offset_switch_operand = src_node_offset });
    const special_prong_src = block.src(.{ .node_offset_switch_special_prong = src_node_offset });
    const extra = sema.code.extraData(Zir.Inst.SwitchBlock, inst_data.payload_index);

    const operand: SwitchProngAnalysis.Operand, const raw_operand_ty: Type = op: {
        const maybe_ptr = try sema.resolveInst(extra.data.operand);
        const val, const ref = if (operand_is_ref)
            .{ try sema.analyzeLoad(block, src, maybe_ptr, operand_src), maybe_ptr }
        else
            .{ maybe_ptr, undefined };

        const init_cond = try sema.switchCond(block, operand_src, val);

        const operand_ty = sema.typeOf(val);

        if (extra.data.bits.has_continue and !block.isComptime()) {
            // Even if the operand is comptime-known, this `switch` is runtime.
            if (try operand_ty.comptimeOnlySema(pt)) {
                return sema.failWithOwnedErrorMsg(block, msg: {
                    const msg = try sema.errMsg(operand_src, "operand of switch loop has comptime-only type '{}'", .{operand_ty.fmt(pt)});
                    errdefer msg.destroy(gpa);
                    try sema.errNote(operand_src, msg, "switch loops are evalauted at runtime outside of comptime scopes", .{});
                    break :msg msg;
                });
            }
            try sema.validateRuntimeValue(block, operand_src, maybe_ptr);
            const operand_alloc = if (extra.data.bits.any_non_inline_capture) a: {
                const operand_ptr_ty = try pt.singleMutPtrType(sema.typeOf(maybe_ptr));
                const operand_alloc = try block.addTy(.alloc, operand_ptr_ty);
                _ = try block.addBinOp(.store, operand_alloc, maybe_ptr);
                break :a operand_alloc;
            } else undefined;
            break :op .{
                .{ .loop = .{
                    .operand_alloc = operand_alloc,
                    .operand_is_ref = operand_is_ref,
                    .init_cond = init_cond,
                } },
                operand_ty,
            };
        }

        // We always use `simple` in the comptime case, because as far as the dispatching logic
        // is concerned, it really is dispatching a single prong. `resolveSwitchComptime` will
        // be resposible for recursively resolving different prongs as needed.
        break :op .{
            .{ .simple = .{
                .by_val = val,
                .by_ref = ref,
                .cond = init_cond,
            } },
            operand_ty,
        };
    };

    const union_originally = raw_operand_ty.zigTypeTag(zcu) == .@"union";
    const err_set = raw_operand_ty.zigTypeTag(zcu) == .error_set;
    const cond_ty = switch (raw_operand_ty.zigTypeTag(zcu)) {
        .@"union" => raw_operand_ty.unionTagType(zcu).?, // validated by `switchCond` above
        else => raw_operand_ty,
    };

    // AstGen guarantees that the instruction immediately preceding
    // switch_block(_ref) is a dbg_stmt
    const cond_dbg_node_index: Zir.Inst.Index = @enumFromInt(@intFromEnum(inst) - 1);

    var header_extra_index: usize = extra.end;

    const scalar_cases_len = extra.data.bits.scalar_cases_len;
    const multi_cases_len = if (extra.data.bits.has_multi_cases) blk: {
        const multi_cases_len = sema.code.extra[header_extra_index];
        header_extra_index += 1;
        break :blk multi_cases_len;
    } else 0;

    const tag_capture_inst: Zir.Inst.Index = if (extra.data.bits.any_has_tag_capture) blk: {
        const tag_capture_inst: Zir.Inst.Index = @enumFromInt(sema.code.extra[header_extra_index]);
        header_extra_index += 1;
        // SwitchProngAnalysis wants inst_map to have space for the tag capture.
        // Note that the normal capture is referred to via the switch block
        // index, which there is already necessarily space for.
        try sema.inst_map.ensureSpaceForInstructions(gpa, &.{tag_capture_inst});
        break :blk tag_capture_inst;
    } else undefined;

    var case_vals = try std.ArrayListUnmanaged(Air.Inst.Ref).initCapacity(gpa, scalar_cases_len + 2 * multi_cases_len);
    defer case_vals.deinit(gpa);

    const special_prong = extra.data.bits.specialProng();
    const special: SpecialProng = switch (special_prong) {
        .none => .{
            .body = &.{},
            .end = header_extra_index,
            .capture = .none,
            .is_inline = false,
            .has_tag_capture = false,
        },
        .under, .@"else" => blk: {
            const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[header_extra_index]);
            const extra_body_start = header_extra_index + 1;
            break :blk .{
                .body = sema.code.bodySlice(extra_body_start, info.body_len),
                .end = extra_body_start + info.body_len,
                .capture = info.capture,
                .is_inline = info.is_inline,
                .has_tag_capture = info.has_tag_capture,
            };
        },
    };

    // Duplicate checking variables later also used for `inline else`.
    var seen_enum_fields: []?LazySrcLoc = &.{};
    var seen_errors = SwitchErrorSet.init(gpa);
    var range_set = RangeSet.init(gpa, zcu);
    var true_count: u8 = 0;
    var false_count: u8 = 0;

    defer {
        range_set.deinit();
        gpa.free(seen_enum_fields);
        seen_errors.deinit();
    }

    var empty_enum = false;

    var else_error_ty: ?Type = null;

    // Validate usage of '_' prongs.
    if (special_prong == .under and !raw_operand_ty.isNonexhaustiveEnum(zcu)) {
        const msg = msg: {
            const msg = try sema.errMsg(
                src,
                "'_' prong only allowed when switching on non-exhaustive enums",
                .{},
            );
            errdefer msg.destroy(gpa);
            try sema.errNote(
                special_prong_src,
                msg,
                "'_' prong here",
                .{},
            );
            try sema.errNote(
                src,
                msg,
                "consider using 'else'",
                .{},
            );
            break :msg msg;
        };
        return sema.failWithOwnedErrorMsg(block, msg);
    }

    // Validate for duplicate items, missing else prong, and invalid range.
    switch (cond_ty.zigTypeTag(zcu)) {
        .@"union" => unreachable, // handled in `switchCond`
        .@"enum" => {
            seen_enum_fields = try gpa.alloc(?LazySrcLoc, cond_ty.enumFieldCount(zcu));
            empty_enum = seen_enum_fields.len == 0 and !cond_ty.isNonexhaustiveEnum(zcu);
            @memset(seen_enum_fields, null);
            // `range_set` is used for non-exhaustive enum values that do not correspond to any tags.

            var extra_index: usize = special.end;
            {
                var scalar_i: u32 = 0;
                while (scalar_i < scalar_cases_len) : (scalar_i += 1) {
                    const item_ref: Zir.Inst.Ref = @enumFromInt(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1 + info.body_len;

                    case_vals.appendAssumeCapacity(try sema.validateSwitchItemEnum(
                        block,
                        seen_enum_fields,
                        &range_set,
                        item_ref,
                        cond_ty,
                        block.src(.{ .switch_case_item = .{
                            .switch_node_offset = src_node_offset,
                            .case_idx = .{ .kind = .scalar, .index = @intCast(scalar_i) },
                            .item_idx = .{ .kind = .single, .index = 0 },
                        } }),
                    ));
                }
            }
            {
                var multi_i: u32 = 0;
                while (multi_i < multi_cases_len) : (multi_i += 1) {
                    const items_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const ranges_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const items = sema.code.refSlice(extra_index, items_len);
                    extra_index += items_len + info.body_len;

                    try case_vals.ensureUnusedCapacity(gpa, items.len);
                    for (items, 0..) |item_ref, item_i| {
                        case_vals.appendAssumeCapacity(try sema.validateSwitchItemEnum(
                            block,
                            seen_enum_fields,
                            &range_set,
                            item_ref,
                            cond_ty,
                            block.src(.{ .switch_case_item = .{
                                .switch_node_offset = src_node_offset,
                                .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                                .item_idx = .{ .kind = .single, .index = @intCast(item_i) },
                            } }),
                        ));
                    }

                    try sema.validateSwitchNoRange(block, ranges_len, cond_ty, src_node_offset);
                }
            }
            const all_tags_handled = for (seen_enum_fields) |seen_src| {
                if (seen_src == null) break false;
            } else true;

            if (special_prong == .@"else") {
                if (all_tags_handled and !cond_ty.isNonexhaustiveEnum(zcu)) return sema.fail(
                    block,
                    special_prong_src,
                    "unreachable else prong; all cases already handled",
                    .{},
                );
            } else if (!all_tags_handled) {
                const msg = msg: {
                    const msg = try sema.errMsg(
                        src,
                        "switch must handle all possibilities",
                        .{},
                    );
                    errdefer msg.destroy(sema.gpa);
                    for (seen_enum_fields, 0..) |seen_src, i| {
                        if (seen_src != null) continue;

                        const field_name = cond_ty.enumFieldName(i, zcu);
                        try sema.addFieldErrNote(
                            cond_ty,
                            i,
                            msg,
                            "unhandled enumeration value: '{}'",
                            .{field_name.fmt(&zcu.intern_pool)},
                        );
                    }
                    try sema.errNote(
                        cond_ty.srcLoc(zcu),
                        msg,
                        "enum '{}' declared here",
                        .{cond_ty.fmt(pt)},
                    );
                    break :msg msg;
                };
                return sema.failWithOwnedErrorMsg(block, msg);
            } else if (special_prong == .none and cond_ty.isNonexhaustiveEnum(zcu) and !union_originally) {
                return sema.fail(
                    block,
                    src,
                    "switch on non-exhaustive enum must include 'else' or '_' prong",
                    .{},
                );
            }
        },
        .error_set => else_error_ty = try validateErrSetSwitch(
            sema,
            block,
            &seen_errors,
            &case_vals,
            cond_ty,
            inst_data,
            scalar_cases_len,
            multi_cases_len,
            .{ .body = special.body, .end = special.end, .src = special_prong_src },
            special_prong == .@"else",
        ),
        .int, .comptime_int => {
            var extra_index: usize = special.end;
            {
                var scalar_i: u32 = 0;
                while (scalar_i < scalar_cases_len) : (scalar_i += 1) {
                    const item_ref: Zir.Inst.Ref = @enumFromInt(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1 + info.body_len;

                    case_vals.appendAssumeCapacity(try sema.validateSwitchItemInt(
                        block,
                        &range_set,
                        item_ref,
                        cond_ty,
                        block.src(.{ .switch_case_item = .{
                            .switch_node_offset = src_node_offset,
                            .case_idx = .{ .kind = .scalar, .index = @intCast(scalar_i) },
                            .item_idx = .{ .kind = .single, .index = 0 },
                        } }),
                    ));
                }
            }
            {
                var multi_i: u32 = 0;
                while (multi_i < multi_cases_len) : (multi_i += 1) {
                    const items_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const ranges_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const items = sema.code.refSlice(extra_index, items_len);
                    extra_index += items_len;

                    try case_vals.ensureUnusedCapacity(gpa, items.len);
                    for (items, 0..) |item_ref, item_i| {
                        case_vals.appendAssumeCapacity(try sema.validateSwitchItemInt(
                            block,
                            &range_set,
                            item_ref,
                            cond_ty,
                            block.src(.{ .switch_case_item = .{
                                .switch_node_offset = src_node_offset,
                                .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                                .item_idx = .{ .kind = .single, .index = @intCast(item_i) },
                            } }),
                        ));
                    }

                    try case_vals.ensureUnusedCapacity(gpa, 2 * ranges_len);
                    var range_i: u32 = 0;
                    while (range_i < ranges_len) : (range_i += 1) {
                        const item_first: Zir.Inst.Ref = @enumFromInt(sema.code.extra[extra_index]);
                        extra_index += 1;
                        const item_last: Zir.Inst.Ref = @enumFromInt(sema.code.extra[extra_index]);
                        extra_index += 1;

                        const vals = try sema.validateSwitchRange(
                            block,
                            &range_set,
                            item_first,
                            item_last,
                            cond_ty,
                            block.src(.{ .switch_case_item = .{
                                .switch_node_offset = src_node_offset,
                                .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                                .item_idx = .{ .kind = .range, .index = @intCast(range_i) },
                            } }),
                        );
                        case_vals.appendAssumeCapacity(vals[0]);
                        case_vals.appendAssumeCapacity(vals[1]);
                    }

                    extra_index += info.body_len;
                }
            }

            check_range: {
                if (cond_ty.zigTypeTag(zcu) == .int) {
                    const min_int = try cond_ty.minInt(pt, cond_ty);
                    const max_int = try cond_ty.maxInt(pt, cond_ty);
                    if (try range_set.spans(min_int.toIntern(), max_int.toIntern())) {
                        if (special_prong == .@"else") {
                            return sema.fail(
                                block,
                                special_prong_src,
                                "unreachable else prong; all cases already handled",
                                .{},
                            );
                        }
                        break :check_range;
                    }
                }
                if (special_prong != .@"else") {
                    return sema.fail(
                        block,
                        src,
                        "switch must handle all possibilities",
                        .{},
                    );
                }
            }
        },
        .bool => {
            var extra_index: usize = special.end;
            {
                var scalar_i: u32 = 0;
                while (scalar_i < scalar_cases_len) : (scalar_i += 1) {
                    const item_ref: Zir.Inst.Ref = @enumFromInt(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1 + info.body_len;

                    case_vals.appendAssumeCapacity(try sema.validateSwitchItemBool(
                        block,
                        &true_count,
                        &false_count,
                        item_ref,
                        block.src(.{ .switch_case_item = .{
                            .switch_node_offset = src_node_offset,
                            .case_idx = .{ .kind = .scalar, .index = @intCast(scalar_i) },
                            .item_idx = .{ .kind = .single, .index = 0 },
                        } }),
                    ));
                }
            }
            {
                var multi_i: u32 = 0;
                while (multi_i < multi_cases_len) : (multi_i += 1) {
                    const items_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const ranges_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const items = sema.code.refSlice(extra_index, items_len);
                    extra_index += items_len + info.body_len;

                    try case_vals.ensureUnusedCapacity(gpa, items.len);
                    for (items, 0..) |item_ref, item_i| {
                        case_vals.appendAssumeCapacity(try sema.validateSwitchItemBool(
                            block,
                            &true_count,
                            &false_count,
                            item_ref,
                            block.src(.{ .switch_case_item = .{
                                .switch_node_offset = src_node_offset,
                                .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                                .item_idx = .{ .kind = .single, .index = @intCast(item_i) },
                            } }),
                        ));
                    }

                    try sema.validateSwitchNoRange(block, ranges_len, cond_ty, src_node_offset);
                }
            }
            switch (special_prong) {
                .@"else" => {
                    if (true_count + false_count == 2) {
                        return sema.fail(
                            block,
                            special_prong_src,
                            "unreachable else prong; all cases already handled",
                            .{},
                        );
                    }
                },
                .under, .none => {
                    if (true_count + false_count < 2) {
                        return sema.fail(
                            block,
                            src,
                            "switch must handle all possibilities",
                            .{},
                        );
                    }
                },
            }
        },
        .enum_literal, .void, .@"fn", .pointer, .type => {
            if (special_prong != .@"else") {
                return sema.fail(
                    block,
                    src,
                    "else prong required when switching on type '{}'",
                    .{cond_ty.fmt(pt)},
                );
            }

            var seen_values = ValueSrcMap{};
            defer seen_values.deinit(gpa);

            var extra_index: usize = special.end;
            {
                var scalar_i: u32 = 0;
                while (scalar_i < scalar_cases_len) : (scalar_i += 1) {
                    const item_ref: Zir.Inst.Ref = @enumFromInt(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1;
                    extra_index += info.body_len;

                    case_vals.appendAssumeCapacity(try sema.validateSwitchItemSparse(
                        block,
                        &seen_values,
                        item_ref,
                        cond_ty,
                        block.src(.{ .switch_case_item = .{
                            .switch_node_offset = src_node_offset,
                            .case_idx = .{ .kind = .scalar, .index = @intCast(scalar_i) },
                            .item_idx = .{ .kind = .single, .index = 0 },
                        } }),
                    ));
                }
            }
            {
                var multi_i: u32 = 0;
                while (multi_i < multi_cases_len) : (multi_i += 1) {
                    const items_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const ranges_len = sema.code.extra[extra_index];
                    extra_index += 1;
                    const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
                    extra_index += 1;
                    const items = sema.code.refSlice(extra_index, items_len);
                    extra_index += items_len + info.body_len;

                    try case_vals.ensureUnusedCapacity(gpa, items.len);
                    for (items, 0..) |item_ref, item_i| {
                        case_vals.appendAssumeCapacity(try sema.validateSwitchItemSparse(
                            block,
                            &seen_values,
                            item_ref,
                            cond_ty,
                            block.src(.{ .switch_case_item = .{
                                .switch_node_offset = src_node_offset,
                                .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                                .item_idx = .{ .kind = .single, .index = @intCast(item_i) },
                            } }),
                        ));
                    }

                    try sema.validateSwitchNoRange(block, ranges_len, cond_ty, src_node_offset);
                }
            }
        },

        .error_union,
        .noreturn,
        .array,
        .@"struct",
        .undefined,
        .null,
        .optional,
        .@"opaque",
        .vector,
        .frame,
        .@"anyframe",
        .comptime_float,
        .float,
        => return sema.fail(block, operand_src, "invalid switch operand type '{}'", .{
            raw_operand_ty.fmt(pt),
        }),
    }

    const spa: SwitchProngAnalysis = .{
        .sema = sema,
        .parent_block = block,
        .operand = operand,
        .else_error_ty = else_error_ty,
        .switch_block_inst = inst,
        .tag_capture_inst = tag_capture_inst,
    };

    const block_inst: Air.Inst.Index = @enumFromInt(sema.air_instructions.len);
    try sema.air_instructions.append(gpa, .{
        .tag = .block,
        .data = undefined,
    });
    var label: Block.Label = .{
        .zir_block = inst,
        .merges = .{
            .src_locs = .{},
            .results = .{},
            .br_list = .{},
            .block_inst = block_inst,
        },
    };

    var child_block: Block = .{
        .parent = block,
        .sema = sema,
        .namespace = block.namespace,
        .instructions = .{},
        .label = &label,
        .inlining = block.inlining,
        .comptime_reason = block.comptime_reason,
        .is_typeof = block.is_typeof,
        .c_import_buf = block.c_import_buf,
        .runtime_cond = block.runtime_cond,
        .runtime_loop = block.runtime_loop,
        .runtime_index = block.runtime_index,
        .want_safety = block.want_safety,
        .error_return_trace_index = block.error_return_trace_index,
        .src_base_inst = block.src_base_inst,
        .type_name_ctx = block.type_name_ctx,
    };
    const merges = &child_block.label.?.merges;
    defer child_block.instructions.deinit(gpa);
    defer merges.deinit(gpa);

    if (scalar_cases_len + multi_cases_len == 0 and !special.is_inline) {
        if (empty_enum) {
            return .void_value;
        }
        if (special_prong == .none) {
            return sema.fail(block, src, "switch must handle all possibilities", .{});
        }
        const init_cond = switch (operand) {
            .simple => |s| s.cond,
            .loop => |l| l.init_cond,
        };
        if (zcu.backendSupportsFeature(.is_named_enum_value) and block.wantSafety() and
            raw_operand_ty.zigTypeTag(zcu) == .@"enum" and !raw_operand_ty.isNonexhaustiveEnum(zcu))
        {
            try sema.zirDbgStmt(block, cond_dbg_node_index);
            const ok = try block.addUnOp(.is_named_enum_value, init_cond);
            try sema.addSafetyCheck(block, src, ok, .corrupt_switch);
        }
        if (err_set and try sema.maybeErrorUnwrap(block, special.body, init_cond, operand_src, false)) {
            return .unreachable_value;
        }
    }

    switch (operand) {
        .loop => {}, // always runtime; evaluation in comptime scope uses `simple`
        .simple => |s| {
            if (try sema.resolveDefinedValue(&child_block, src, s.cond)) |cond_val| {
                return resolveSwitchComptimeLoop(
                    sema,
                    spa,
                    &child_block,
                    if (operand_is_ref)
                        sema.typeOf(s.by_ref)
                    else
                        raw_operand_ty,
                    cond_ty,
                    cond_val,
                    src_node_offset,
                    special,
                    case_vals,
                    scalar_cases_len,
                    multi_cases_len,
                    err_set,
                    empty_enum,
                    operand_is_ref,
                );
            }

            if (scalar_cases_len + multi_cases_len == 0 and !special.is_inline and !extra.data.bits.has_continue) {
                return spa.resolveProngComptime(
                    &child_block,
                    .special,
                    special.body,
                    special.capture,
                    block.src(.{ .switch_capture = .{
                        .switch_node_offset = src_node_offset,
                        .case_idx = LazySrcLoc.Offset.SwitchCaseIndex.special,
                    } }),
                    undefined, // case_vals may be undefined for special prongs
                    .none,
                    false,
                    merges,
                );
            }
        },
    }

    if (child_block.isComptime()) {
        _ = try sema.resolveConstDefinedValue(&child_block, operand_src, operand.simple.cond, null);
        unreachable;
    }

    const air_switch_ref = try sema.analyzeSwitchRuntimeBlock(
        spa,
        &child_block,
        src,
        switch (operand) {
            .simple => |s| s.cond,
            .loop => |l| l.init_cond,
        },
        cond_ty,
        operand_src,
        case_vals,
        special,
        scalar_cases_len,
        multi_cases_len,
        union_originally,
        raw_operand_ty,
        err_set,
        src_node_offset,
        special_prong_src,
        seen_enum_fields,
        seen_errors,
        range_set,
        true_count,
        false_count,
        cond_dbg_node_index,
        false,
    );

    for (merges.extra_insts.items, merges.extra_src_locs.items) |placeholder_inst, dispatch_src| {
        var replacement_block = block.makeSubBlock();
        defer replacement_block.instructions.deinit(gpa);

        assert(sema.air_instructions.items(.tag)[@intFromEnum(placeholder_inst)] == .br);
        const new_operand_maybe_ref = sema.air_instructions.items(.data)[@intFromEnum(placeholder_inst)].br.operand;

        if (extra.data.bits.any_non_inline_capture) {
            _ = try replacement_block.addBinOp(.store, operand.loop.operand_alloc, new_operand_maybe_ref);
        }

        const new_operand_val = if (operand_is_ref)
            try sema.analyzeLoad(&replacement_block, dispatch_src, new_operand_maybe_ref, dispatch_src)
        else
            new_operand_maybe_ref;

        const new_cond = try sema.switchCond(&replacement_block, dispatch_src, new_operand_val);

        if (zcu.backendSupportsFeature(.is_named_enum_value) and block.wantSafety() and
            cond_ty.zigTypeTag(zcu) == .@"enum" and !cond_ty.isNonexhaustiveEnum(zcu) and
            !try sema.isComptimeKnown(new_cond))
        {
            const ok = try replacement_block.addUnOp(.is_named_enum_value, new_cond);
            try sema.addSafetyCheck(&replacement_block, src, ok, .corrupt_switch);
        }

        _ = try replacement_block.addInst(.{
            .tag = .switch_dispatch,
            .data = .{ .br = .{
                .block_inst = air_switch_ref.toIndex().?,
                .operand = new_cond,
            } },
        });

        if (replacement_block.instructions.items.len == 1) {
            // Optimization: we don't need a block!
            sema.air_instructions.set(
                @intFromEnum(placeholder_inst),
                sema.air_instructions.get(@intFromEnum(replacement_block.instructions.items[0])),
            );
            continue;
        }

        // Replace placeholder with a block.
        // No `br` is needed as the block is a switch dispatch so necessarily `noreturn`.
        try sema.air_extra.ensureUnusedCapacity(
            gpa,
            @typeInfo(Air.Block).@"struct".fields.len + replacement_block.instructions.items.len,
        );
        sema.air_instructions.set(@intFromEnum(placeholder_inst), .{
            .tag = .block,
            .data = .{ .ty_pl = .{
                .ty = .noreturn_type,
                .payload = sema.addExtraAssumeCapacity(Air.Block{
                    .body_len = @intCast(replacement_block.instructions.items.len),
                }),
            } },
        });
        sema.air_extra.appendSliceAssumeCapacity(@ptrCast(replacement_block.instructions.items));
    }

    return sema.resolveAnalyzedBlock(block, src, &child_block, merges, false);
}
```
嗯，我现在要理解这段Zig代码。看起来这是一个处理switch语句的函数，叫做zirSwitchBlock。首先，我需要理清这个函数的主要逻辑和各个部分的作用。

首先，函数参数包括sema、block、inst和operand_is_ref。sema可能是语义分析的上下文，block是当前的代码块，inst是Zir指令的索引，operand_is_ref表示操作数是否是引用。返回类型是CompileError!Air.Inst.Ref，说明这个函数可能会返回一个Air指令的引用或者编译错误。

接下来，函数开始获取一些上下文信息，比如tracy跟踪、pt（可能是解析上下文）、zcu（Zig编译单元）、gpa（通用分配器）等。然后通过inst_data获取源节点信息，包括操作数的源位置和特殊分支的源位置。extra变量用于存储Zir的SwitchBlock指令的额外数据。

然后，代码处理操作数（operand）和其类型（raw_operand_ty）。这里有一个op标签块，根据operand_is_ref决定如何处理操作数。如果是引用，就分析加载操作数的值；否则直接使用操作数。接着调用switchCond来获取初始条件（init_cond），并确定操作数的类型operand_ty。

接下来，如果存在continue并且block不是编译时，会检查操作数类型是否是comptime-only，如果是的话报错。然后处理操作数的分配和存储。这里似乎有两种情况：一种是循环的情况（loop），另一种是简单情况（simple），根据是否有continue和any_non_inline_capture来决定。

之后，根据操作数的类型（联合体、错误集等）处理不同的条件类型cond_ty。然后处理各个case的情况，包括标量case、多个case、范围case等。这里有很多验证步骤，比如检查枚举是否覆盖所有情况、错误处理、整数范围覆盖等。

在处理各个case时，会验证每个项是否合法，比如枚举项是否存在重复、错误名称是否正确、整数范围是否合理等。对于布尔类型，检查true和false是否都被处理。对于其他类型如类型、函数指针等，需要else分支。

之后，创建Air指令的block，处理子块child_block，并分析各个分支。如果是编译时已知的条件值，会调用resolveSwitchComptimeLoop来处理。如果是运行时的情况，则生成相应的switch分发指令。

最后，处理可能的循环继续情况，替换占位符指令，并返回分析后的块结果。

整个函数的核心是处理switch语句的不同情况，验证所有分支是否覆盖所有可能，生成相应的中间表示（Air指令）。其中涉及类型检查、错误处理、编译时和运行时的不同处理逻辑，以及代码块的合并和控制流的处理。

可能需要注意的地方包括各种验证步骤，比如枚举的覆盖检查、错误集的处理、整数范围的覆盖，以及处理不同类型的操作数时的不同逻辑。此外，对于编译时和运行时的不同处理路径也需要特别关注，比如当操作数是编译时已知时的优化处理，以及运行时生成相应的条件分支。

在理解过程中，可能需要进一步查阅Zig语言中switch语句的语义，以及Zir和Air中间表示的结构，才能完全理解每个步骤的具体作用和实现细节。
================================================
这段代码实现了Zig编译器中对`switch`语句的语义分析和中间代码生成。其核心思维模型可概括为以下步骤：

---

### **1. 上下文初始化**
- **语义分析上下文**：获取编译单元（`zcu`）、内存分配器（`gpa`）、解析上下文（`pt`）等关键对象。
- **指令解析**：从ZIR（Zig中间表示）中提取`switch`指令的元数据（`inst_data`、`extra`），定位源码位置（`src`）。
- **操作数处理**：根据`operand_is_ref`决定操作数的加载方式（直接值或间接引用），生成初始条件（`init_cond`）和类型（`operand_ty`）。

---

### **2. 分支类型与条件验证**
- **条件类型推导**：根据操作数类型（如枚举、错误集、整数、布尔等）确定`cond_ty`，并验证其合法性。
- **特殊分支处理**：
  - **`_`分支**：仅允许非穷尽枚举。
  - **`else`分支**：检查是否冗余（如所有情况已覆盖）。
- **分支项验证**：
  - **枚举**：确保所有字段被处理，标记重复项。
  - **错误集**：检查错误名称是否存在，标记未处理错误。
  - **整数/布尔**：验证范围覆盖性（如整数是否全范围覆盖、布尔是否处理`true`/`false`）。
  - **其他类型**（如类型、函数指针）：强制要求`else`分支。

---

### **3. 中间代码生成**
- **代码块构建**：创建`Air`指令的`block`，管理子块（`child_block`）的合并点（`merges`）。
- **编译时优化**：
  - 若条件值为编译时已知（`comptime`），递归解析分支逻辑（`resolveSwitchComptimeLoop`）。
  - 避免生成冗余代码（如空`switch`直接返回`void`）。
- **运行时处理**：
  - 生成条件分发指令（`.switch_dispatch`）。
  - 处理循环`continue`逻辑，替换占位符指令为实际代码块。
  - 安全性检查（如枚举值合法性验证）。

---

### **4. 控制流合并与结果返回**
- **块合并**：通过`merges`管理所有分支的跳转目标，合并子块结果到父块。
- **错误处理**：在无效类型（如`error_union`、`noreturn`）或未覆盖分支时抛出编译错误。

---

### **关键设计**
- **穷尽性检查**：确保所有可能的输入值均有对应分支，避免未定义行为。
- **类型驱动**：根据操作数类型动态调整验证逻辑（如枚举与错误集的不同处理）。
- **编译时/运行时分离**：通过`comptime`标记优化编译时分支，生成高效的运行时代码。
- **安全性**：插入运行时检查（如`is_named_enum_value`）防止非法值。

---

### **总结**
此函数是Zig编译器处理`switch`语义的核心模块，通过类型推导、分支验证、代码生成三个层次确保`switch`的正确性与高效性。其设计紧密结合Zig语言特性（如编译时计算、穷尽检查），体现了静态类型语言编译器的典型工作流程。