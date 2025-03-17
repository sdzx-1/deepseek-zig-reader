```zig
fn analyzeSwitchRuntimeBlock(
    sema: *Sema,
    spa: SwitchProngAnalysis,
    child_block: *Block,
    src: LazySrcLoc,
    operand: Air.Inst.Ref,
    operand_ty: Type,
    operand_src: LazySrcLoc,
    case_vals: std.ArrayListUnmanaged(Air.Inst.Ref),
    special: SpecialProng,
    scalar_cases_len: usize,
    multi_cases_len: usize,
    union_originally: bool,
    maybe_union_ty: Type,
    err_set: bool,
    switch_node_offset: std.zig.Ast.Node.Offset,
    special_prong_src: LazySrcLoc,
    seen_enum_fields: []?LazySrcLoc,
    seen_errors: SwitchErrorSet,
    range_set: RangeSet,
    true_count: u8,
    false_count: u8,
    cond_dbg_node_index: Zir.Inst.Index,
    allow_err_code_unwrap: bool,
) CompileError!Air.Inst.Ref {
    const pt = sema.pt;
    const zcu = pt.zcu;
    const gpa = sema.gpa;
    const ip = &zcu.intern_pool;

    const block = child_block.parent.?;

    const estimated_cases_extra = (scalar_cases_len + multi_cases_len) *
        @typeInfo(Air.SwitchBr.Case).@"struct".fields.len + 2;
    var cases_extra = try std.ArrayListUnmanaged(u32).initCapacity(gpa, estimated_cases_extra);
    defer cases_extra.deinit(gpa);

    var branch_hints = try std.ArrayListUnmanaged(std.builtin.BranchHint).initCapacity(gpa, scalar_cases_len);
    defer branch_hints.deinit(gpa);

    var case_block = child_block.makeSubBlock();
    case_block.runtime_loop = null;
    case_block.runtime_cond = operand_src;
    case_block.runtime_index.increment();
    case_block.need_debug_scope = null; // this body is emitted regardless
    defer case_block.instructions.deinit(gpa);

    var extra_index: usize = special.end;

    var scalar_i: usize = 0;
    while (scalar_i < scalar_cases_len) : (scalar_i += 1) {
        extra_index += 1;
        const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
        extra_index += 1;
        const body = sema.code.bodySlice(extra_index, info.body_len);
        extra_index += info.body_len;

        case_block.instructions.shrinkRetainingCapacity(0);
        case_block.error_return_trace_index = child_block.error_return_trace_index;

        const item = case_vals.items[scalar_i];
        // `item` is already guaranteed to be constant known.

        const analyze_body = if (union_originally) blk: {
            const unresolved_item_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, item, undefined) catch unreachable;
            const item_val = sema.resolveLazyValue(unresolved_item_val) catch unreachable;
            const field_ty = maybe_union_ty.unionFieldType(item_val, zcu).?;
            break :blk field_ty.zigTypeTag(zcu) != .noreturn;
        } else true;

        const prong_hint: std.builtin.BranchHint = if (err_set and
            try sema.maybeErrorUnwrap(&case_block, body, operand, operand_src, allow_err_code_unwrap))
        h: {
            // nothing to do here. weight against error branch
            break :h .unlikely;
        } else if (analyze_body) h: {
            break :h try spa.analyzeProngRuntime(
                &case_block,
                .normal,
                body,
                info.capture,
                child_block.src(.{ .switch_capture = .{
                    .switch_node_offset = switch_node_offset,
                    .case_idx = .{ .kind = .scalar, .index = @intCast(scalar_i) },
                } }),
                &.{item},
                if (info.is_inline) item else .none,
                info.has_tag_capture,
            );
        } else h: {
            _ = try case_block.addNoOp(.unreach);
            break :h .none;
        };

        try branch_hints.append(gpa, prong_hint);
        try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
            1 + // `item`, no ranges
            case_block.instructions.items.len);
        cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
            .items_len = 1,
            .ranges_len = 0,
            .body_len = @intCast(case_block.instructions.items.len),
        }));
        cases_extra.appendAssumeCapacity(@intFromEnum(item));
        cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
    }

    var cases_len = scalar_cases_len;
    var case_val_idx: usize = scalar_cases_len;
    var multi_i: u32 = 0;
    while (multi_i < multi_cases_len) : (multi_i += 1) {
        const items_len = sema.code.extra[extra_index];
        extra_index += 1;
        const ranges_len = sema.code.extra[extra_index];
        extra_index += 1;
        const info: Zir.Inst.SwitchBlock.ProngInfo = @bitCast(sema.code.extra[extra_index]);
        extra_index += 1 + items_len + 2 * ranges_len;

        const items = case_vals.items[case_val_idx..][0..items_len];
        case_val_idx += items_len;
        // TODO: @ptrCast slice once Sema supports it
        const ranges: []const [2]Air.Inst.Ref = @as([*]const [2]Air.Inst.Ref, @ptrCast(case_vals.items[case_val_idx..]))[0..ranges_len];
        case_val_idx += ranges_len * 2;

        const body = sema.code.bodySlice(extra_index, info.body_len);
        extra_index += info.body_len;

        case_block.instructions.shrinkRetainingCapacity(0);
        case_block.error_return_trace_index = child_block.error_return_trace_index;

        // Generate all possible cases as scalar prongs.
        if (info.is_inline) {
            var emit_bb = false;

            for (ranges, 0..) |range_items, range_i| {
                var item = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, range_items[0], undefined) catch unreachable;
                const item_last = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, range_items[1], undefined) catch unreachable;

                while (item.compareScalar(.lte, item_last, operand_ty, zcu)) : ({
                    // Previous validation has resolved any possible lazy values.
                    item = sema.intAddScalar(item, try pt.intValue(operand_ty, 1), operand_ty) catch |err| switch (err) {
                        error.Overflow => unreachable,
                        else => |e| return e,
                    };
                }) {
                    cases_len += 1;

                    const item_ref = Air.internedToRef(item.toIntern());

                    case_block.instructions.shrinkRetainingCapacity(0);
                    case_block.error_return_trace_index = child_block.error_return_trace_index;

                    if (emit_bb) try sema.emitBackwardBranch(block, block.src(.{ .switch_case_item = .{
                        .switch_node_offset = switch_node_offset,
                        .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                        .item_idx = .{ .kind = .range, .index = @intCast(range_i) },
                    } }));
                    emit_bb = true;

                    const prong_hint = try spa.analyzeProngRuntime(
                        &case_block,
                        .normal,
                        body,
                        info.capture,
                        child_block.src(.{ .switch_capture = .{
                            .switch_node_offset = switch_node_offset,
                            .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                        } }),
                        undefined, // case_vals may be undefined for ranges
                        item_ref,
                        info.has_tag_capture,
                    );
                    try branch_hints.append(gpa, prong_hint);

                    try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                        1 + // `item`, no ranges
                        case_block.instructions.items.len);
                    cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                        .items_len = 1,
                        .ranges_len = 0,
                        .body_len = @intCast(case_block.instructions.items.len),
                    }));
                    cases_extra.appendAssumeCapacity(@intFromEnum(item_ref));
                    cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));

                    if (item.compareScalar(.eq, item_last, operand_ty, zcu)) break;
                }
            }

            for (items, 0..) |item, item_i| {
                cases_len += 1;

                case_block.instructions.shrinkRetainingCapacity(0);
                case_block.error_return_trace_index = child_block.error_return_trace_index;

                const analyze_body = if (union_originally) blk: {
                    const item_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, item, undefined) catch unreachable;
                    const field_ty = maybe_union_ty.unionFieldType(item_val, zcu).?;
                    break :blk field_ty.zigTypeTag(zcu) != .noreturn;
                } else true;

                if (emit_bb) try sema.emitBackwardBranch(block, block.src(.{ .switch_case_item = .{
                    .switch_node_offset = switch_node_offset,
                    .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                    .item_idx = .{ .kind = .single, .index = @intCast(item_i) },
                } }));
                emit_bb = true;

                const prong_hint: std.builtin.BranchHint = if (analyze_body) h: {
                    break :h try spa.analyzeProngRuntime(
                        &case_block,
                        .normal,
                        body,
                        info.capture,
                        child_block.src(.{ .switch_capture = .{
                            .switch_node_offset = switch_node_offset,
                            .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                        } }),
                        &.{item},
                        item,
                        info.has_tag_capture,
                    );
                } else h: {
                    _ = try case_block.addNoOp(.unreach);
                    break :h .none;
                };
                try branch_hints.append(gpa, prong_hint);

                try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                    1 + // `item`, no ranges
                    case_block.instructions.items.len);
                cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                    .items_len = 1,
                    .ranges_len = 0,
                    .body_len = @intCast(case_block.instructions.items.len),
                }));
                cases_extra.appendAssumeCapacity(@intFromEnum(item));
                cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
            }

            continue;
        }

        cases_len += 1;

        const analyze_body = if (union_originally)
            for (items) |item| {
                const item_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, item, undefined) catch unreachable;
                const field_ty = maybe_union_ty.unionFieldType(item_val, zcu).?;
                if (field_ty.zigTypeTag(zcu) != .noreturn) break true;
            } else false
        else
            true;

        const prong_hint: std.builtin.BranchHint = if (err_set and
            try sema.maybeErrorUnwrap(&case_block, body, operand, operand_src, allow_err_code_unwrap))
        h: {
            // nothing to do here. weight against error branch
            break :h .unlikely;
        } else if (analyze_body) h: {
            break :h try spa.analyzeProngRuntime(
                &case_block,
                .normal,
                body,
                info.capture,
                child_block.src(.{ .switch_capture = .{
                    .switch_node_offset = switch_node_offset,
                    .case_idx = .{ .kind = .multi, .index = @intCast(multi_i) },
                } }),
                items,
                .none,
                false,
            );
        } else h: {
            _ = try case_block.addNoOp(.unreach);
            break :h .none;
        };

        try branch_hints.append(gpa, prong_hint);

        try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
            items.len + 2 * ranges_len +
            case_block.instructions.items.len);
        cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
            .items_len = @intCast(items.len),
            .ranges_len = @intCast(ranges_len),
            .body_len = @intCast(case_block.instructions.items.len),
        }));

        for (items) |item| {
            cases_extra.appendAssumeCapacity(@intFromEnum(item));
        }
        for (ranges) |range| {
            cases_extra.appendSliceAssumeCapacity(&.{
                @intFromEnum(range[0]),
                @intFromEnum(range[1]),
            });
        }

        cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
    }

    const else_body: []const Air.Inst.Index = if (special.body.len != 0 or case_block.wantSafety()) else_body: {
        var emit_bb = false;
        if (special.is_inline) switch (operand_ty.zigTypeTag(zcu)) {
            .@"enum" => {
                if (operand_ty.isNonexhaustiveEnum(zcu) and !union_originally) {
                    return sema.fail(block, special_prong_src, "cannot enumerate values of type '{}' for 'inline else'", .{
                        operand_ty.fmt(pt),
                    });
                }
                for (seen_enum_fields, 0..) |f, i| {
                    if (f != null) continue;
                    cases_len += 1;

                    const item_val = try pt.enumValueFieldIndex(operand_ty, @intCast(i));
                    const item_ref = Air.internedToRef(item_val.toIntern());

                    case_block.instructions.shrinkRetainingCapacity(0);
                    case_block.error_return_trace_index = child_block.error_return_trace_index;

                    const analyze_body = if (union_originally) blk: {
                        const field_ty = maybe_union_ty.unionFieldType(item_val, zcu).?;
                        break :blk field_ty.zigTypeTag(zcu) != .noreturn;
                    } else true;

                    if (emit_bb) try sema.emitBackwardBranch(block, special_prong_src);
                    emit_bb = true;

                    const prong_hint: std.builtin.BranchHint = if (analyze_body) h: {
                        break :h try spa.analyzeProngRuntime(
                            &case_block,
                            .special,
                            special.body,
                            special.capture,
                            child_block.src(.{ .switch_capture = .{
                                .switch_node_offset = switch_node_offset,
                                .case_idx = LazySrcLoc.Offset.SwitchCaseIndex.special,
                            } }),
                            &.{item_ref},
                            item_ref,
                            special.has_tag_capture,
                        );
                    } else h: {
                        _ = try case_block.addNoOp(.unreach);
                        break :h .none;
                    };
                    try branch_hints.append(gpa, prong_hint);

                    try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                        1 + // `item`, no ranges
                        case_block.instructions.items.len);
                    cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                        .items_len = 1,
                        .ranges_len = 0,
                        .body_len = @intCast(case_block.instructions.items.len),
                    }));
                    cases_extra.appendAssumeCapacity(@intFromEnum(item_ref));
                    cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
                }
            },
            .error_set => {
                if (operand_ty.isAnyError(zcu)) {
                    return sema.fail(block, special_prong_src, "cannot enumerate values of type '{}' for 'inline else'", .{
                        operand_ty.fmt(pt),
                    });
                }
                const error_names = operand_ty.errorSetNames(zcu);
                for (0..error_names.len) |name_index| {
                    const error_name = error_names.get(ip)[name_index];
                    if (seen_errors.contains(error_name)) continue;
                    cases_len += 1;

                    const item_val = try pt.intern(.{ .err = .{
                        .ty = operand_ty.toIntern(),
                        .name = error_name,
                    } });
                    const item_ref = Air.internedToRef(item_val);

                    case_block.instructions.shrinkRetainingCapacity(0);
                    case_block.error_return_trace_index = child_block.error_return_trace_index;

                    if (emit_bb) try sema.emitBackwardBranch(block, special_prong_src);
                    emit_bb = true;

                    const prong_hint = try spa.analyzeProngRuntime(
                        &case_block,
                        .special,
                        special.body,
                        special.capture,
                        child_block.src(.{ .switch_capture = .{
                            .switch_node_offset = switch_node_offset,
                            .case_idx = LazySrcLoc.Offset.SwitchCaseIndex.special,
                        } }),
                        &.{item_ref},
                        item_ref,
                        special.has_tag_capture,
                    );
                    try branch_hints.append(gpa, prong_hint);

                    try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                        1 + // `item`, no ranges
                        case_block.instructions.items.len);
                    cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                        .items_len = 1,
                        .ranges_len = 0,
                        .body_len = @intCast(case_block.instructions.items.len),
                    }));
                    cases_extra.appendAssumeCapacity(@intFromEnum(item_ref));
                    cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
                }
            },
            .int => {
                var it = try RangeSetUnhandledIterator.init(sema, operand_ty, range_set);
                while (try it.next()) |cur| {
                    cases_len += 1;

                    const item_ref = Air.internedToRef(cur);

                    case_block.instructions.shrinkRetainingCapacity(0);
                    case_block.error_return_trace_index = child_block.error_return_trace_index;

                    if (emit_bb) try sema.emitBackwardBranch(block, special_prong_src);
                    emit_bb = true;

                    const prong_hint = try spa.analyzeProngRuntime(
                        &case_block,
                        .special,
                        special.body,
                        special.capture,
                        child_block.src(.{ .switch_capture = .{
                            .switch_node_offset = switch_node_offset,
                            .case_idx = LazySrcLoc.Offset.SwitchCaseIndex.special,
                        } }),
                        &.{item_ref},
                        item_ref,
                        special.has_tag_capture,
                    );
                    try branch_hints.append(gpa, prong_hint);

                    try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                        1 + // `item`, no ranges
                        case_block.instructions.items.len);
                    cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                        .items_len = 1,
                        .ranges_len = 0,
                        .body_len = @intCast(case_block.instructions.items.len),
                    }));
                    cases_extra.appendAssumeCapacity(@intFromEnum(item_ref));
                    cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
                }
            },
            .bool => {
                if (true_count == 0) {
                    cases_len += 1;

                    case_block.instructions.shrinkRetainingCapacity(0);
                    case_block.error_return_trace_index = child_block.error_return_trace_index;

                    if (emit_bb) try sema.emitBackwardBranch(block, special_prong_src);
                    emit_bb = true;

                    const prong_hint = try spa.analyzeProngRuntime(
                        &case_block,
                        .special,
                        special.body,
                        special.capture,
                        child_block.src(.{ .switch_capture = .{
                            .switch_node_offset = switch_node_offset,
                            .case_idx = LazySrcLoc.Offset.SwitchCaseIndex.special,
                        } }),
                        &.{.bool_true},
                        .bool_true,
                        special.has_tag_capture,
                    );
                    try branch_hints.append(gpa, prong_hint);

                    try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                        1 + // `item`, no ranges
                        case_block.instructions.items.len);
                    cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                        .items_len = 1,
                        .ranges_len = 0,
                        .body_len = @intCast(case_block.instructions.items.len),
                    }));
                    cases_extra.appendAssumeCapacity(@intFromEnum(Air.Inst.Ref.bool_true));
                    cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
                }
                if (false_count == 0) {
                    cases_len += 1;

                    case_block.instructions.shrinkRetainingCapacity(0);
                    case_block.error_return_trace_index = child_block.error_return_trace_index;

                    if (emit_bb) try sema.emitBackwardBranch(block, special_prong_src);
                    emit_bb = true;

                    const prong_hint = try spa.analyzeProngRuntime(
                        &case_block,
                        .special,
                        special.body,
                        special.capture,
                        child_block.src(.{ .switch_capture = .{
                            .switch_node_offset = switch_node_offset,
                            .case_idx = LazySrcLoc.Offset.SwitchCaseIndex.special,
                        } }),
                        &.{.bool_false},
                        .bool_false,
                        special.has_tag_capture,
                    );
                    try branch_hints.append(gpa, prong_hint);

                    try cases_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                        1 + // `item`, no ranges
                        case_block.instructions.items.len);
                    cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                        .items_len = 1,
                        .ranges_len = 0,
                        .body_len = @intCast(case_block.instructions.items.len),
                    }));
                    cases_extra.appendAssumeCapacity(@intFromEnum(Air.Inst.Ref.bool_false));
                    cases_extra.appendSliceAssumeCapacity(@ptrCast(case_block.instructions.items));
                }
            },
            else => return sema.fail(block, special_prong_src, "cannot enumerate values of type '{}' for 'inline else'", .{
                operand_ty.fmt(pt),
            }),
        };

        case_block.instructions.shrinkRetainingCapacity(0);
        case_block.error_return_trace_index = child_block.error_return_trace_index;

        if (zcu.backendSupportsFeature(.is_named_enum_value) and
            special.body.len != 0 and block.wantSafety() and
            operand_ty.zigTypeTag(zcu) == .@"enum" and
            (!operand_ty.isNonexhaustiveEnum(zcu) or union_originally))
        {
            try sema.zirDbgStmt(&case_block, cond_dbg_node_index);
            const ok = try case_block.addUnOp(.is_named_enum_value, operand);
            try sema.addSafetyCheck(&case_block, src, ok, .corrupt_switch);
        }

        const analyze_body = if (union_originally and !special.is_inline)
            for (seen_enum_fields, 0..) |seen_field, index| {
                if (seen_field != null) continue;
                const union_obj = zcu.typeToUnion(maybe_union_ty).?;
                const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[index]);
                if (field_ty.zigTypeTag(zcu) != .noreturn) break true;
            } else false
        else
            true;
        const else_hint: std.builtin.BranchHint = if (special.body.len != 0 and err_set and
            try sema.maybeErrorUnwrap(&case_block, special.body, operand, operand_src, allow_err_code_unwrap))
        h: {
            // nothing to do here. weight against error branch
            break :h .unlikely;
        } else if (special.body.len != 0 and analyze_body and !special.is_inline) h: {
            break :h try spa.analyzeProngRuntime(
                &case_block,
                .special,
                special.body,
                special.capture,
                child_block.src(.{ .switch_capture = .{
                    .switch_node_offset = switch_node_offset,
                    .case_idx = LazySrcLoc.Offset.SwitchCaseIndex.special,
                } }),
                undefined, // case_vals may be undefined for special prongs
                .none,
                false,
            );
        } else h: {
            // We still need a terminator in this block, but we have proven
            // that it is unreachable.
            if (case_block.wantSafety()) {
                try sema.zirDbgStmt(&case_block, cond_dbg_node_index);
                try sema.safetyPanic(&case_block, src, .corrupt_switch);
            } else {
                _ = try case_block.addNoOp(.unreach);
            }
            // Safety check / unreachable branches are cold.
            break :h .cold;
        };

        try branch_hints.append(gpa, else_hint);
        break :else_body case_block.instructions.items;
    } else else_body: {
        try branch_hints.append(gpa, .none);
        break :else_body &.{};
    };

    assert(branch_hints.items.len == cases_len + 1);

    try sema.air_extra.ensureUnusedCapacity(gpa, @typeInfo(Air.SwitchBr).@"struct".fields.len +
        cases_extra.items.len + else_body.len +
        (std.math.divCeil(usize, branch_hints.items.len, 10) catch unreachable)); // branch hints

    const payload_index = sema.addExtraAssumeCapacity(Air.SwitchBr{
        .cases_len = @intCast(cases_len),
        .else_body_len = @intCast(else_body.len),
    });

    {
        // Add branch hints.
        var cur_bag: u32 = 0;
        for (branch_hints.items, 0..) |hint, idx| {
            const idx_in_bag = idx % 10;
            cur_bag |= @as(u32, @intFromEnum(hint)) << @intCast(idx_in_bag * 3);
            if (idx_in_bag == 9) {
                sema.air_extra.appendAssumeCapacity(cur_bag);
                cur_bag = 0;
            }
        }
        if (branch_hints.items.len % 10 != 0) {
            sema.air_extra.appendAssumeCapacity(cur_bag);
        }
    }
    sema.air_extra.appendSliceAssumeCapacity(@ptrCast(cases_extra.items));
    sema.air_extra.appendSliceAssumeCapacity(@ptrCast(else_body));

    return try child_block.addInst(.{
        .tag = if (spa.operand == .loop) .loop_switch_br else .switch_br,
        .data = .{ .pl_op = .{
            .operand = operand,
            .payload = payload_index,
        } },
    });
}
```
嗯，我现在要理解这段Zig代码。首先，这个函数的名字是analyzeSwitchRuntimeBlock，看起来是处理switch语句的运行时块分析的。参数很多，可能涉及到语义分析（Sema）、switch的分支分析（SwitchProngAnalysis）、子块（child_block）等等。

首先，函数内部有很多变量声明，比如pt、zcu、gpa、ip，这些都是从sema中提取的，可能是用来处理类型、内存分配和内部池的。接下来看到child_block.parent被赋值给block，可能是在处理父块的信息。

然后，代码里有一个estimated_cases_extra的计算，估计cases_extra的容量，这可能和switch分支的数量有关。之后用cases_extra和branch_hints两个动态数组来存储分支的额外信息和分支提示。

接下来是case_block的创建，使用了child_block.makeSubBlock()，然后调整了一些属性，比如runtime_loop、runtime_cond等。这说明case_block是用来处理每个case的代码块的。

然后是两个大的循环：一个处理标量情况（scalar_i循环），另一个处理多情况（multi_i循环）。在标量循环中，从sema.code.extra中提取信息，处理每个标量case，可能包括捕获的值、body的长度等。这里还涉及到处理联合类型（union_originally）的情况，检查字段类型是否为noreturn，以决定是否分析body。

在处理每个标量case时，会调用spa.analyzeProngRuntime来分析prong的运行时行为，并将分支提示（prong_hint）添加到branch_hints数组中。然后将case的信息和指令添加到cases_extra中。

多情况循环类似，但处理的是多个item和range的情况。这里可能处理像case 1...5这样的范围匹配。对于每个多case，同样会生成多个标量prong，并处理内联（inline）的情况，可能需要展开范围到具体的值，并逐一处理。

然后处理特殊prong（else或者default的情况），根据不同的类型（enum、error_set、int、bool等）生成相应的case。例如，对于enum类型，检查未处理的字段，并为每个生成case。对于error_set，处理未覆盖的错误名。对于int类型，使用RangeSetUnhandledIterator遍历未处理的范围。对于bool类型，处理true和false未覆盖的情况。

最后，处理else_body，根据是否还有未覆盖的情况生成相应的指令，可能包括安全检查和panic。然后将所有收集到的cases_extra和else_body的信息整合到Air.SwitchBr结构中，并生成相应的Air指令（switch_br或loop_switch_br）。

总结来说，这个函数的主要作用是分析switch语句的各个分支，生成对应的中间表示（Air指令），处理不同类型的case（标量、范围、多情况、特殊else），并收集分支提示，确保所有可能的情况都被覆盖，同时处理联合类型和错误集等特殊情况。
================================================
这段代码是Zig编译器中对`switch`语句进行运行时分析的函数，主要任务是将`switch`的各个分支转换为中间表示（Air指令），并处理各种复杂情况。其思维模型可总结如下：

---

### **核心逻辑**
1. **初始化与上下文准备**  
   - 从语义分析器（`Sema`）中提取类型、内存分配器、内部池等上下文信息。
   - 创建子块（`case_block`）用于处理每个`case`的代码生成，并关联父块信息。

2. **分支分类处理**  
   - **标量分支（Scalar Cases）**  
     - 遍历每个标量值（如具体枚举值、整数等），生成对应的代码块。
     - 处理联合类型（Union）的特殊情况：若字段类型为`noreturn`，则跳过该分支。
     - 收集分支提示（如`likely`/`unlikely`），用于优化代码生成。
   - **多分支（Multi Cases）**  
     - 处理范围（如`1..5`）和多个值的组合分支。
     - 若为内联（`inline`）分支，将范围展开为多个标量值，逐一生成代码块。
     - 处理联合类型的有效性检查，确保分支代码有意义。

3. **特殊分支（Else/Default）**  
   - 根据操作数类型（枚举、错误集、整数、布尔等），生成未覆盖的默认分支：
     - **枚举**：检查未处理的枚举字段，生成对应分支。
     - **错误集**：遍历未覆盖的错误名。
     - **整数范围**：通过`RangeSetUnhandledIterator`生成未覆盖的整数值。
     - **布尔**：补充未处理的`true`或`false`分支。
   - 若未覆盖分支不可达，插入安全检查或`unreachable`指令。

4. **代码生成与指令整合**  
   - 将所有分支的指令和元数据（`cases_extra`）合并到`Air.SwitchBr`结构中。
   - 处理分支提示的打包（每10个提示压缩为一个`u32`）。
   - 生成最终的`switch_br`或`loop_switch_br`指令，表示完整的`switch`逻辑。

---

### **关键细节**
- **联合类型与错误处理**  
  - 若操作数是联合类型，需检查其字段类型是否为`noreturn`，以跳过无效分支。
  - 对错误集（`error_set`）的处理需要确保所有可能的错误名被覆盖。

- **内联分支展开**  
  - 对`inline`修饰的分支，需展开范围（如`1..5`为`1, 2, 3, 4, 5`），生成多个标量分支。

- **安全性与可达性**  
  - 若`else`分支未覆盖所有情况且开启安全检查（`wantSafety`），插入`corrupt_switch`的运行时检查。
  - 不可达分支用`unreachable`指令标记。

---

### **流程总结**
1. **收集分支信息** → 2. **生成每个分支的代码块** → 3. **处理默认分支** → 4. **整合为中间表示**  
最终目标是生成高效且安全的`switch`指令，覆盖所有可能路径，同时为编译器优化提供分支提示。

--- 

**代码作用**：将高级`switch`语义转换为底层中间表示，处理类型特化、分支展开、安全性检查，确保语义正确性与生成代码的效率。