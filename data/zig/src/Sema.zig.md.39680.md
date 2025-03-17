```zig
const SwitchProngAnalysis = struct {
    sema: *Sema,
    /// The block containing the `switch_block` itself.
    parent_block: *Block,
    operand: Operand,
    /// If this switch is on an error set, this is the type to assign to the
    /// `else` prong. If `null`, the prong should be unreachable.
    else_error_ty: ?Type,
    /// The index of the `switch_block` instruction itself.
    switch_block_inst: Zir.Inst.Index,
    /// The dummy index into which inline tag captures should be placed. May be
    /// undefined if no prong has a tag capture.
    tag_capture_inst: Zir.Inst.Index,

    const Operand = union(enum) {
        /// This switch will be dispatched only once, with the given operand.
        simple: struct {
            /// The raw switch operand value. Always defined.
            by_val: Air.Inst.Ref,
            /// The switch operand *pointer*. Defined only if there is a prong
            /// with a by-ref capture.
            by_ref: Air.Inst.Ref,
            /// The switch condition value. For unions, `operand` is the union
            /// and `cond` is its enum tag value.
            cond: Air.Inst.Ref,
        },
        /// This switch may be dispatched multiple times with `continue` syntax.
        /// As such, the operand is stored in an alloc if needed.
        loop: struct {
            /// The `alloc` containing the `switch` operand for the active dispatch.
            /// Each prong must load from this `alloc` to get captures.
            /// If there are no captures, this may be undefined.
            operand_alloc: Air.Inst.Ref,
            /// Whether `operand_alloc` contains a by-val operand or a by-ref
            /// operand.
            operand_is_ref: bool,
            /// The switch condition value for the *initial* dispatch. For
            /// unions, this is the enum tag value.
            init_cond: Air.Inst.Ref,
        },
    };

    /// Resolve a switch prong which is determined at comptime to have no peers.
    /// Uses `resolveBlockBody`. Sets up captures as needed.
    fn resolveProngComptime(
        spa: SwitchProngAnalysis,
        child_block: *Block,
        prong_type: enum { normal, special },
        prong_body: []const Zir.Inst.Index,
        capture: Zir.Inst.SwitchBlock.ProngInfo.Capture,
        /// Must use the `switch_capture` field in `offset`.
        capture_src: LazySrcLoc,
        /// The set of all values which can reach this prong. May be undefined
        /// if the prong is special or contains ranges.
        case_vals: []const Air.Inst.Ref,
        /// The inline capture of this prong. If this is not an inline prong,
        /// this is `.none`.
        inline_case_capture: Air.Inst.Ref,
        /// Whether this prong has an inline tag capture. If `true`, then
        /// `inline_case_capture` cannot be `.none`.
        has_tag_capture: bool,
        merges: *Block.Merges,
    ) CompileError!Air.Inst.Ref {
        const sema = spa.sema;
        const src = spa.parent_block.nodeOffset(
            sema.code.instructions.items(.data)[@intFromEnum(spa.switch_block_inst)].pl_node.src_node,
        );

        // We can propagate `.cold` hints from this branch since it's comptime-known
        // to be taken from the parent branch.
        const parent_hint = sema.branch_hint;
        defer sema.branch_hint = parent_hint orelse if (sema.branch_hint == .cold) .cold else null;

        if (has_tag_capture) {
            const tag_ref = try spa.analyzeTagCapture(child_block, capture_src, inline_case_capture);
            sema.inst_map.putAssumeCapacity(spa.tag_capture_inst, tag_ref);
        }
        defer if (has_tag_capture) assert(sema.inst_map.remove(spa.tag_capture_inst));

        switch (capture) {
            .none => {
                return sema.resolveBlockBody(spa.parent_block, src, child_block, prong_body, spa.switch_block_inst, merges);
            },

            .by_val, .by_ref => {
                const capture_ref = try spa.analyzeCapture(
                    child_block,
                    capture == .by_ref,
                    prong_type == .special,
                    capture_src,
                    case_vals,
                    inline_case_capture,
                );

                if (sema.typeOf(capture_ref).isNoReturn(sema.pt.zcu)) {
                    // This prong should be unreachable!
                    return .unreachable_value;
                }

                sema.inst_map.putAssumeCapacity(spa.switch_block_inst, capture_ref);
                defer assert(sema.inst_map.remove(spa.switch_block_inst));

                return sema.resolveBlockBody(spa.parent_block, src, child_block, prong_body, spa.switch_block_inst, merges);
            },
        }
    }

    /// Analyze a switch prong which may have peers at runtime.
    /// Uses `analyzeBodyRuntimeBreak`. Sets up captures as needed.
    /// Returns the `BranchHint` for the prong.
    fn analyzeProngRuntime(
        spa: SwitchProngAnalysis,
        case_block: *Block,
        prong_type: enum { normal, special },
        prong_body: []const Zir.Inst.Index,
        capture: Zir.Inst.SwitchBlock.ProngInfo.Capture,
        /// Must use the `switch_capture` field in `offset`.
        capture_src: LazySrcLoc,
        /// The set of all values which can reach this prong. May be undefined
        /// if the prong is special or contains ranges.
        case_vals: []const Air.Inst.Ref,
        /// The inline capture of this prong. If this is not an inline prong,
        /// this is `.none`.
        inline_case_capture: Air.Inst.Ref,
        /// Whether this prong has an inline tag capture. If `true`, then
        /// `inline_case_capture` cannot be `.none`.
        has_tag_capture: bool,
    ) CompileError!std.builtin.BranchHint {
        const sema = spa.sema;

        if (has_tag_capture) {
            const tag_ref = try spa.analyzeTagCapture(case_block, capture_src, inline_case_capture);
            sema.inst_map.putAssumeCapacity(spa.tag_capture_inst, tag_ref);
        }
        defer if (has_tag_capture) assert(sema.inst_map.remove(spa.tag_capture_inst));

        switch (capture) {
            .none => {
                return sema.analyzeBodyRuntimeBreak(case_block, prong_body);
            },

            .by_val, .by_ref => {
                const capture_ref = try spa.analyzeCapture(
                    case_block,
                    capture == .by_ref,
                    prong_type == .special,
                    capture_src,
                    case_vals,
                    inline_case_capture,
                );

                if (sema.typeOf(capture_ref).isNoReturn(sema.pt.zcu)) {
                    // No need to analyze any further, the prong is unreachable
                    return .none;
                }

                sema.inst_map.putAssumeCapacity(spa.switch_block_inst, capture_ref);
                defer assert(sema.inst_map.remove(spa.switch_block_inst));

                return sema.analyzeBodyRuntimeBreak(case_block, prong_body);
            },
        }
    }

    fn analyzeTagCapture(
        spa: SwitchProngAnalysis,
        block: *Block,
        capture_src: LazySrcLoc,
        inline_case_capture: Air.Inst.Ref,
    ) CompileError!Air.Inst.Ref {
        const sema = spa.sema;
        const pt = sema.pt;
        const zcu = pt.zcu;
        const operand_ty = switch (spa.operand) {
            .simple => |s| sema.typeOf(s.by_val),
            .loop => |l| ty: {
                const alloc_ty = sema.typeOf(l.operand_alloc);
                const alloc_child = alloc_ty.childType(zcu);
                if (l.operand_is_ref) break :ty alloc_child.childType(zcu);
                break :ty alloc_child;
            },
        };
        if (operand_ty.zigTypeTag(zcu) != .@"union") {
            const tag_capture_src: LazySrcLoc = .{
                .base_node_inst = capture_src.base_node_inst,
                .offset = .{ .switch_tag_capture = capture_src.offset.switch_capture },
            };
            return sema.fail(block, tag_capture_src, "cannot capture tag of non-union type '{}'", .{
                operand_ty.fmt(pt),
            });
        }
        assert(inline_case_capture != .none);
        return inline_case_capture;
    }

    fn analyzeCapture(
        spa: SwitchProngAnalysis,
        block: *Block,
        capture_byref: bool,
        is_special_prong: bool,
        capture_src: LazySrcLoc,
        case_vals: []const Air.Inst.Ref,
        inline_case_capture: Air.Inst.Ref,
    ) CompileError!Air.Inst.Ref {
        const sema = spa.sema;
        const pt = sema.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;

        const zir_datas = sema.code.instructions.items(.data);
        const switch_node_offset = zir_datas[@intFromEnum(spa.switch_block_inst)].pl_node.src_node;

        const operand_src = block.src(.{ .node_offset_switch_operand = switch_node_offset });

        const operand_val, const operand_ptr = switch (spa.operand) {
            .simple => |s| .{ s.by_val, s.by_ref },
            .loop => |l| op: {
                const loaded = try sema.analyzeLoad(block, operand_src, l.operand_alloc, operand_src);
                if (l.operand_is_ref) {
                    const by_val = try sema.analyzeLoad(block, operand_src, loaded, operand_src);
                    break :op .{ by_val, loaded };
                } else {
                    break :op .{ loaded, undefined };
                }
            },
        };

        const operand_ty = sema.typeOf(operand_val);
        const operand_ptr_ty = if (capture_byref) sema.typeOf(operand_ptr) else undefined;

        if (inline_case_capture != .none) {
            const item_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, inline_case_capture, undefined) catch unreachable;
            if (operand_ty.zigTypeTag(zcu) == .@"union") {
                const field_index: u32 = @intCast(operand_ty.unionTagFieldIndex(item_val, zcu).?);
                const union_obj = zcu.typeToUnion(operand_ty).?;
                const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_index]);
                if (capture_byref) {
                    const ptr_field_ty = try pt.ptrTypeSema(.{
                        .child = field_ty.toIntern(),
                        .flags = .{
                            .is_const = !operand_ptr_ty.ptrIsMutable(zcu),
                            .is_volatile = operand_ptr_ty.isVolatilePtr(zcu),
                            .address_space = operand_ptr_ty.ptrAddressSpace(zcu),
                        },
                    });
                    if (try sema.resolveDefinedValue(block, operand_src, operand_ptr)) |union_ptr| {
                        return Air.internedToRef((try union_ptr.ptrField(field_index, pt)).toIntern());
                    }
                    return block.addStructFieldPtr(operand_ptr, field_index, ptr_field_ty);
                } else {
                    if (try sema.resolveDefinedValue(block, operand_src, operand_val)) |union_val| {
                        const tag_and_val = ip.indexToKey(union_val.toIntern()).un;
                        return Air.internedToRef(tag_and_val.val);
                    }
                    return block.addStructFieldVal(operand_val, field_index, field_ty);
                }
            } else if (capture_byref) {
                return sema.uavRef(item_val.toIntern());
            } else {
                return inline_case_capture;
            }
        }

        if (is_special_prong) {
            if (capture_byref) {
                return operand_ptr;
            }

            switch (operand_ty.zigTypeTag(zcu)) {
                .error_set => if (spa.else_error_ty) |ty| {
                    return sema.bitCast(block, ty, operand_val, operand_src, null);
                } else {
                    try sema.analyzeUnreachable(block, operand_src, false);
                    return .unreachable_value;
                },
                else => return operand_val,
            }
        }

        switch (operand_ty.zigTypeTag(zcu)) {
            .@"union" => {
                const union_obj = zcu.typeToUnion(operand_ty).?;
                const first_item_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, case_vals[0], undefined) catch unreachable;

                const first_field_index: u32 = zcu.unionTagFieldIndex(union_obj, first_item_val).?;
                const first_field_ty = Type.fromInterned(union_obj.field_types.get(ip)[first_field_index]);

                const field_indices = try sema.arena.alloc(u32, case_vals.len);
                for (case_vals, field_indices) |item, *field_idx| {
                    const item_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, item, undefined) catch unreachable;
                    field_idx.* = zcu.unionTagFieldIndex(union_obj, item_val).?;
                }

                // Fast path: if all the operands are the same type already, we don't need to hit
                // PTR! This will also allow us to emit simpler code.
                const same_types = for (field_indices[1..]) |field_idx| {
                    const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_idx]);
                    if (!field_ty.eql(first_field_ty, zcu)) break false;
                } else true;

                const capture_ty = if (same_types) first_field_ty else capture_ty: {
                    // We need values to run PTR on, so make a bunch of undef constants.
                    const dummy_captures = try sema.arena.alloc(Air.Inst.Ref, case_vals.len);
                    for (dummy_captures, field_indices) |*dummy, field_idx| {
                        const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_idx]);
                        dummy.* = try pt.undefRef(field_ty);
                    }

                    const case_srcs = try sema.arena.alloc(?LazySrcLoc, case_vals.len);
                    for (case_srcs, 0..) |*case_src, i| {
                        case_src.* = .{
                            .base_node_inst = capture_src.base_node_inst,
                            .offset = .{ .switch_case_item = .{
                                .switch_node_offset = switch_node_offset,
                                .case_idx = capture_src.offset.switch_capture.case_idx,
                                .item_idx = .{ .kind = .single, .index = @intCast(i) },
                            } },
                        };
                    }

                    break :capture_ty sema.resolvePeerTypes(block, capture_src, dummy_captures, .{ .override = case_srcs }) catch |err| switch (err) {
                        error.AnalysisFail => {
                            const msg = sema.err orelse return error.AnalysisFail;
                            try sema.reparentOwnedErrorMsg(capture_src, msg, "capture group with incompatible types", .{});
                            return error.AnalysisFail;
                        },
                        else => |e| return e,
                    };
                };

                // By-reference captures have some further restrictions which make them easier to emit
                if (capture_byref) {
                    const operand_ptr_info = operand_ptr_ty.ptrInfo(zcu);
                    const capture_ptr_ty = resolve: {
                        // By-ref captures of hetereogeneous types are only allowed if all field
                        // pointer types are peer resolvable to each other.
                        // We need values to run PTR on, so make a bunch of undef constants.
                        const dummy_captures = try sema.arena.alloc(Air.Inst.Ref, case_vals.len);
                        for (field_indices, dummy_captures) |field_idx, *dummy| {
                            const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_idx]);
                            const field_ptr_ty = try pt.ptrTypeSema(.{
                                .child = field_ty.toIntern(),
                                .flags = .{
                                    .is_const = operand_ptr_info.flags.is_const,
                                    .is_volatile = operand_ptr_info.flags.is_volatile,
                                    .address_space = operand_ptr_info.flags.address_space,
                                    .alignment = union_obj.fieldAlign(ip, field_idx),
                                },
                            });
                            dummy.* = try pt.undefRef(field_ptr_ty);
                        }
                        const case_srcs = try sema.arena.alloc(?LazySrcLoc, case_vals.len);
                        for (case_srcs, 0..) |*case_src, i| {
                            case_src.* = .{
                                .base_node_inst = capture_src.base_node_inst,
                                .offset = .{ .switch_case_item = .{
                                    .switch_node_offset = switch_node_offset,
                                    .case_idx = capture_src.offset.switch_capture.case_idx,
                                    .item_idx = .{ .kind = .single, .index = @intCast(i) },
                                } },
                            };
                        }

                        break :resolve sema.resolvePeerTypes(block, capture_src, dummy_captures, .{ .override = case_srcs }) catch |err| switch (err) {
                            error.AnalysisFail => {
                                const msg = sema.err orelse return error.AnalysisFail;
                                try sema.errNote(capture_src, msg, "this coercion is only possible when capturing by value", .{});
                                try sema.reparentOwnedErrorMsg(capture_src, msg, "capture group with incompatible types", .{});
                                return error.AnalysisFail;
                            },
                            else => |e| return e,
                        };
                    };

                    if (try sema.resolveDefinedValue(block, operand_src, operand_ptr)) |op_ptr_val| {
                        if (op_ptr_val.isUndef(zcu)) return pt.undefRef(capture_ptr_ty);
                        const field_ptr_val = try op_ptr_val.ptrField(first_field_index, pt);
                        return Air.internedToRef((try pt.getCoerced(field_ptr_val, capture_ptr_ty)).toIntern());
                    }

                    try sema.requireRuntimeBlock(block, operand_src, null);
                    return block.addStructFieldPtr(operand_ptr, first_field_index, capture_ptr_ty);
                }

                if (try sema.resolveDefinedValue(block, operand_src, operand_val)) |operand_val_val| {
                    if (operand_val_val.isUndef(zcu)) return pt.undefRef(capture_ty);
                    const union_val = ip.indexToKey(operand_val_val.toIntern()).un;
                    if (Value.fromInterned(union_val.tag).isUndef(zcu)) return pt.undefRef(capture_ty);
                    const uncoerced = Air.internedToRef(union_val.val);
                    return sema.coerce(block, capture_ty, uncoerced, operand_src);
                }

                try sema.requireRuntimeBlock(block, operand_src, null);

                if (same_types) {
                    return block.addStructFieldVal(operand_val, first_field_index, capture_ty);
                }

                // We may have to emit a switch block which coerces the operand to the capture type.
                // If we can, try to avoid that using in-memory coercions.
                const first_non_imc = in_mem: {
                    for (field_indices, 0..) |field_idx, i| {
                        const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_idx]);
                        if (.ok != try sema.coerceInMemoryAllowed(block, capture_ty, field_ty, false, zcu.getTarget(), LazySrcLoc.unneeded, LazySrcLoc.unneeded, null)) {
                            break :in_mem i;
                        }
                    }
                    // All fields are in-memory coercible to the resolved type!
                    // Just take the first field and bitcast the result.
                    const uncoerced = try block.addStructFieldVal(operand_val, first_field_index, first_field_ty);
                    return block.addBitCast(capture_ty, uncoerced);
                };

                // By-val capture with heterogeneous types which are not all in-memory coercible to
                // the resolved capture type. We finally have to fall back to the ugly method.

                // However, let's first track which operands are in-memory coercible. There may well
                // be several, and we can squash all of these cases into the same switch prong using
                // a simple bitcast. We'll make this the 'else' prong.

                var in_mem_coercible = try std.DynamicBitSet.initFull(sema.arena, field_indices.len);
                in_mem_coercible.unset(first_non_imc);
                {
                    const next = first_non_imc + 1;
                    for (field_indices[next..], next..) |field_idx, i| {
                        const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_idx]);
                        if (.ok != try sema.coerceInMemoryAllowed(block, capture_ty, field_ty, false, zcu.getTarget(), LazySrcLoc.unneeded, LazySrcLoc.unneeded, null)) {
                            in_mem_coercible.unset(i);
                        }
                    }
                }

                const capture_block_inst = try block.addInstAsIndex(.{
                    .tag = .block,
                    .data = .{
                        .ty_pl = .{
                            .ty = Air.internedToRef(capture_ty.toIntern()),
                            .payload = undefined, // updated below
                        },
                    },
                });

                const prong_count = field_indices.len - in_mem_coercible.count();

                const estimated_extra = prong_count * 6 + (prong_count / 10); // 2 for Case, 1 item, probably 3 insts; plus hints
                var cases_extra = try std.ArrayList(u32).initCapacity(sema.gpa, estimated_extra);
                defer cases_extra.deinit();

                {
                    // All branch hints are `.none`, so just add zero elems.
                    comptime assert(@intFromEnum(std.builtin.BranchHint.none) == 0);
                    const need_elems = std.math.divCeil(usize, prong_count + 1, 10) catch unreachable;
                    try cases_extra.appendNTimes(0, need_elems);
                }

                {
                    // Non-bitcast cases
                    var it = in_mem_coercible.iterator(.{ .kind = .unset });
                    while (it.next()) |idx| {
                        var coerce_block = block.makeSubBlock();
                        defer coerce_block.instructions.deinit(sema.gpa);

                        const case_src: LazySrcLoc = .{
                            .base_node_inst = capture_src.base_node_inst,
                            .offset = .{ .switch_case_item = .{
                                .switch_node_offset = switch_node_offset,
                                .case_idx = capture_src.offset.switch_capture.case_idx,
                                .item_idx = .{ .kind = .single, .index = @intCast(idx) },
                            } },
                        };

                        const field_idx = field_indices[idx];
                        const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_idx]);
                        const uncoerced = try coerce_block.addStructFieldVal(operand_val, field_idx, field_ty);
                        const coerced = try sema.coerce(&coerce_block, capture_ty, uncoerced, case_src);
                        _ = try coerce_block.addBr(capture_block_inst, coerced);

                        try cases_extra.ensureUnusedCapacity(@typeInfo(Air.SwitchBr.Case).@"struct".fields.len +
                            1 + // `item`, no ranges
                            coerce_block.instructions.items.len);
                        cases_extra.appendSliceAssumeCapacity(&payloadToExtraItems(Air.SwitchBr.Case{
                            .items_len = 1,
                            .ranges_len = 0,
                            .body_len = @intCast(coerce_block.instructions.items.len),
                        }));
                        cases_extra.appendAssumeCapacity(@intFromEnum(case_vals[idx])); // item
                        cases_extra.appendSliceAssumeCapacity(@ptrCast(coerce_block.instructions.items)); // body
                    }
                }
                const else_body_len = len: {
                    // 'else' prong uses a bitcast
                    var coerce_block = block.makeSubBlock();
                    defer coerce_block.instructions.deinit(sema.gpa);

                    const first_imc_item_idx = in_mem_coercible.findFirstSet().?;
                    const first_imc_field_idx = field_indices[first_imc_item_idx];
                    const first_imc_field_ty = Type.fromInterned(union_obj.field_types.get(ip)[first_imc_field_idx]);
                    const uncoerced = try coerce_block.addStructFieldVal(operand_val, first_imc_field_idx, first_imc_field_ty);
                    const coerced = try coerce_block.addBitCast(capture_ty, uncoerced);
                    _ = try coerce_block.addBr(capture_block_inst, coerced);

                    try cases_extra.appendSlice(@ptrCast(coerce_block.instructions.items));
                    break :len coerce_block.instructions.items.len;
                };

                try sema.air_extra.ensureUnusedCapacity(sema.gpa, @typeInfo(Air.SwitchBr).@"struct".fields.len +
                    cases_extra.items.len +
                    @typeInfo(Air.Block).@"struct".fields.len +
                    1);

                const switch_br_inst: u32 = @intCast(sema.air_instructions.len);
                try sema.air_instructions.append(sema.gpa, .{
                    .tag = .switch_br,
                    .data = .{
                        .pl_op = .{
                            .operand = undefined, // set by switch below
                            .payload = sema.addExtraAssumeCapacity(Air.SwitchBr{
                                .cases_len = @intCast(prong_count),
                                .else_body_len = @intCast(else_body_len),
                            }),
                        },
                    },
                });
                sema.air_extra.appendSliceAssumeCapacity(cases_extra.items);

                // Set up block body
                switch (spa.operand) {
                    .simple => |s| {
                        const air_datas = sema.air_instructions.items(.data);
                        air_datas[switch_br_inst].pl_op.operand = s.cond;
                        air_datas[@intFromEnum(capture_block_inst)].ty_pl.payload = sema.addExtraAssumeCapacity(Air.Block{
                            .body_len = 1,
                        });
                        sema.air_extra.appendAssumeCapacity(switch_br_inst);
                    },
                    .loop => {
                        // The block must first extract the tag from the loaded union.
                        const tag_inst: Air.Inst.Index = @enumFromInt(sema.air_instructions.len);
                        try sema.air_instructions.append(sema.gpa, .{
                            .tag = .get_union_tag,
                            .data = .{ .ty_op = .{
                                .ty = Air.internedToRef(union_obj.enum_tag_ty),
                                .operand = operand_val,
                            } },
                        });
                        const air_datas = sema.air_instructions.items(.data);
                        air_datas[switch_br_inst].pl_op.operand = tag_inst.toRef();
                        air_datas[@intFromEnum(capture_block_inst)].ty_pl.payload = sema.addExtraAssumeCapacity(Air.Block{
                            .body_len = 2,
                        });
                        sema.air_extra.appendAssumeCapacity(@intFromEnum(tag_inst));
                        sema.air_extra.appendAssumeCapacity(switch_br_inst);
                    },
                }

                return capture_block_inst.toRef();
            },
            .error_set => {
                if (capture_byref) {
                    return sema.fail(
                        block,
                        capture_src,
                        "error set cannot be captured by reference",
                        .{},
                    );
                }

                if (case_vals.len == 1) {
                    const item_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, case_vals[0], undefined) catch unreachable;
                    const item_ty = try pt.singleErrorSetType(item_val.getErrorName(zcu).unwrap().?);
                    return sema.bitCast(block, item_ty, operand_val, operand_src, null);
                }

                var names: InferredErrorSet.NameMap = .{};
                try names.ensureUnusedCapacity(sema.arena, case_vals.len);
                for (case_vals) |err| {
                    const err_val = sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, err, undefined) catch unreachable;
                    names.putAssumeCapacityNoClobber(err_val.getErrorName(zcu).unwrap().?, {});
                }
                const error_ty = try pt.errorSetFromUnsortedNames(names.keys());
                return sema.bitCast(block, error_ty, operand_val, operand_src, null);
            },
            else => {
                // In this case the capture value is just the passed-through value
                // of the switch condition.
                if (capture_byref) {
                    return operand_ptr;
                } else {
                    return operand_val;
                }
            },
        }
    }
}```
好的，我现在要尝试理解这段Zig代码的思维模型。首先，这段代码属于SwitchProngAnalysis结构体，主要用于处理switch语句的不同分支（prong）的分析，包括编译时和运行时的处理，以及如何处理不同类型的捕获（capture）。

首先，我需要理解这个结构体的各个字段的作用：

- sema: 指向Sema的指针，可能是语义分析器的上下文。
- parent_block: 包含switch语句的父块。
- operand: 联合类型，可能是简单操作数或循环操作数，用于表示switch的条件操作数。
- else_error_ty: 如果switch在错误集上，这是else分支的类型，否则为null。
- switch_block_inst: switch块本身的指令索引。
- tag_capture_inst: 用于内联标签捕获的虚拟指令索引。

接下来是两个主要函数：resolveProngComptime和analyzeProngRuntime。它们分别处理编译时确定的分支和运行时可能有多分支的情况。

resolveProngComptime函数处理那些在编译时可以确定没有其他可能分支的情况。这里会设置捕获（如标签捕获和值捕获），并调用resolveBlockBody来处理分支体。如果捕获是by_ref或by_val，会调用analyzeCapture来获取捕获的引用，并处理可能的未到达情况。

analyzeProngRuntime处理运行时可能有多个分支的情况，使用analyzeBodyRuntimeBreak来分析分支体，并返回分支提示。同样，这里也会处理标签和值的捕获，如果类型是不归类型（NoReturn），则认为该分支不可达。

analyzeTagCapture函数处理标签捕获，特别是联合类型的标签。如果不是联合类型，会报错。这里需要确保操作数是联合类型，否则无法捕获标签。

analyzeCapture函数处理值的捕获，根据是by_ref还是by_val，生成相应的引用或值。对于联合类型，处理字段索引，并根据字段类型进行类型解析。如果是错误集类型，处理错误类型的转换。对于其他类型，直接传递操作数的值或指针。

在处理联合类型时，代码会检查所有字段的类型是否相同，如果不同，需要进行类型解析（PTR），确保捕获类型的一致性。如果类型不同且无法在内存中强制转换，可能需要生成额外的switch块来处理不同类型的情况。

对于错误集类型，by_ref捕获是不允许的，会报错。而by_val捕获则会将错误值转换为对应的错误类型。

其他类型（如基本类型）的捕获则直接传递操作数的值或指针。

总结来说，这段代码的核心在于处理switch语句各分支的类型和捕获逻辑，确保在不同情况下（编译时确定、运行时多分支、联合类型、错误集等）都能正确生成相应的代码，处理类型转换和捕获，保证语义正确性。

可能的疑问点包括：

1. Operand联合类型的具体使用场景，简单和循环操作数有何区别？
2. 如何确定case_vals中的值对应的联合字段索引？
3. 类型解析（PTR）的具体过程是怎样的，特别是处理不同类型时的策略？
4. 为什么by_ref捕获在错误集类型下不允许？
5. 处理联合类型时，生成额外switch块的逻辑是如何确保类型安全的？

需要进一步查看相关类型解析和代码生成的具体实现，以及Zig语言中switch语句的语义规则，才能完全理解每个细节。但整体上，这段代码的结构和逻辑是围绕类型检查、捕获处理和分支分析展开的，确保switch语句在不同情况下的正确行为。
================================================
这段代码的思维模型围绕**处理Zig语言中`switch`语句的复杂分支逻辑**展开，主要解决编译时和运行时分支分析、类型捕获及类型转换问题。以下是核心要点：

---

### 1. **核心目标**
   - **分析`switch`分支的捕获逻辑**：处理不同场景下的值/标签捕获（`by_val`、`by_ref`、`tag`），确保类型安全和代码生成正确。
   - **支持联合（Union）和错误集（Error Set）**：处理联合类型的字段选择、错误集的隐式转换，以及动态分支的类型兼容性。
   - **区分编译时与运行时逻辑**：
     - **编译时分支**（`resolveProngComptime`）：直接解析确定的分支，无需运行时检查。
     - **运行时分支**（`analyzeProngRuntime`）：生成可能多路径的代码，处理动态分支的捕获和类型匹配。

---

### 2. **关键数据结构**
   - **`Operand`联合类型**：
     - **简单模式**（`simple`）：直接使用操作数的值和条件（如普通`switch`）。
     - **循环模式**（`loop`）：通过`alloc`存储操作数，支持多次分派（如`continue`语法）。
   - **`SwitchProngAnalysis`**：
     - 记录`switch`的语义上下文（父块、操作数、错误类型等）。
     - 处理捕获指令（如`tag_capture_inst`和`switch_block_inst`的映射）。

---

### 3. **核心逻辑**
   - **标签捕获（`analyzeTagCapture`）**：
     - 仅联合类型支持标签捕获，非联合类型会报错。
     - 内联标签（`inline_case_capture`）直接映射到联合的枚举标签。
   - **值捕获（`analyzeCapture`）**：
     - **联合类型**：
       - 根据字段索引提取值/指针。
       - 检查所有分支字段类型是否一致，若不一致则通过**类型解析（PTR）**推导公共类型。
       - 生成额外的`switch_br`指令处理异构类型的运行时选择。
     - **错误集**：
       - `by_ref`捕获被禁止，`by_val`需转换为具体错误子集类型。
     - **其他类型**：直接传递操作数的值或指针。

---

### 4. **编译时优化**
   - **冷路径提示**：通过`branch_hint`标记冷门分支，优化生成代码。
   - **类型强制转换**：
     - 尽可能通过`bitCast`或内存布局兼容性（`coerceInMemoryAllowed`）避免运行时转换。
     - 对无法直接转换的类型，生成嵌套`switch`块处理不同字段类型。

---

### 5. **错误处理**
   - **不可达分支**：若捕获类型为`noreturn`，直接标记为`unreachable`。
   - **类型不兼容**：在类型解析失败时，生成详细的错误信息（如联合字段类型不一致）。

---

### 6. **设计模式**
   - **子块分割**：通过`makeSubBlock`创建临时块处理复杂逻辑（如类型转换），确保作用域隔离。
   - **指令映射**：使用`inst_map`管理捕获指令的运行时替换（如`tag_capture_inst`的临时插入/移除）。

---

### 7. **总结**
这段代码是Zig编译器语义分析阶段的核心部分，通过精细的类型推导和代码生成策略，确保`switch`语句在不同场景（联合、错误集、多分支）下的语义正确性。其思维模型围绕**类型驱动**和**分支上下文管理**展开，平衡编译时优化与运行时灵活性。