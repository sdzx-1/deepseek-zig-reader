```zig
fn coerceExtra(
    sema: *Sema,
    block: *Block,
    dest_ty: Type,
    inst: Air.Inst.Ref,
    inst_src: LazySrcLoc,
    opts: CoerceOpts,
) CoersionError!Air.Inst.Ref {
    if (dest_ty.isGenericPoison()) return inst;
    const pt = sema.pt;
    const zcu = pt.zcu;
    const ip = &zcu.intern_pool;
    const dest_ty_src = inst_src; // TODO better source location
    try dest_ty.resolveFields(pt);
    const inst_ty = sema.typeOf(inst);
    try inst_ty.resolveFields(pt);
    const target = zcu.getTarget();
    // If the types are the same, we can return the operand.
    if (dest_ty.eql(inst_ty, zcu))
        return inst;

    const maybe_inst_val = try sema.resolveValue(inst);

    var in_memory_result = try sema.coerceInMemoryAllowed(block, dest_ty, inst_ty, false, target, dest_ty_src, inst_src, maybe_inst_val);
    if (in_memory_result == .ok) {
        if (maybe_inst_val) |val| {
            return sema.coerceInMemory(val, dest_ty);
        }
        try sema.requireRuntimeBlock(block, inst_src, null);
        const new_val = try block.addBitCast(dest_ty, inst);
        try sema.checkKnownAllocPtr(block, inst, new_val);
        return new_val;
    }

    switch (dest_ty.zigTypeTag(zcu)) {
        .optional => optional: {
            if (maybe_inst_val) |val| {
                // undefined sets the optional bit also to undefined.
                if (val.toIntern() == .undef) {
                    return pt.undefRef(dest_ty);
                }

                // null to ?T
                if (val.toIntern() == .null_value) {
                    return Air.internedToRef((try pt.intern(.{ .opt = .{
                        .ty = dest_ty.toIntern(),
                        .val = .none,
                    } })));
                }
            }

            // cast from ?*T and ?[*]T to ?*anyopaque
            // but don't do it if the source type is a double pointer
            if (dest_ty.isPtrLikeOptional(zcu) and
                dest_ty.elemType2(zcu).toIntern() == .anyopaque_type and
                inst_ty.isPtrAtRuntime(zcu))
            anyopaque_check: {
                if (!sema.checkPtrAttributes(dest_ty, inst_ty, &in_memory_result)) break :optional;
                const elem_ty = inst_ty.elemType2(zcu);
                if (elem_ty.zigTypeTag(zcu) == .pointer or elem_ty.isPtrLikeOptional(zcu)) {
                    in_memory_result = .{ .double_ptr_to_anyopaque = .{
                        .actual = inst_ty,
                        .wanted = dest_ty,
                    } };
                    break :optional;
                }
                // Let the logic below handle wrapping the optional now that
                // it has been checked to correctly coerce.
                if (!inst_ty.isPtrLikeOptional(zcu)) break :anyopaque_check;
                return sema.coerceCompatiblePtrs(block, dest_ty, inst, inst_src);
            }

            // T to ?T
            const child_type = dest_ty.optionalChild(zcu);
            const intermediate = sema.coerceExtra(block, child_type, inst, inst_src, .{ .report_err = false }) catch |err| switch (err) {
                error.NotCoercible => {
                    if (in_memory_result == .no_match) {
                        // Try to give more useful notes
                        in_memory_result = try sema.coerceInMemoryAllowed(block, child_type, inst_ty, false, target, dest_ty_src, inst_src, maybe_inst_val);
                    }
                    break :optional;
                },
                else => |e| return e,
            };
            return try sema.wrapOptional(block, dest_ty, intermediate, inst_src);
        },
        .pointer => pointer: {
            const dest_info = dest_ty.ptrInfo(zcu);

            // Function body to function pointer.
            if (inst_ty.zigTypeTag(zcu) == .@"fn") {
                const fn_val = try sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, inst, undefined);
                const fn_nav = switch (zcu.intern_pool.indexToKey(fn_val.toIntern())) {
                    .func => |f| f.owner_nav,
                    .@"extern" => |e| e.owner_nav,
                    else => unreachable,
                };
                const inst_as_ptr = try sema.analyzeNavRef(inst_src, fn_nav);
                return sema.coerce(block, dest_ty, inst_as_ptr, inst_src);
            }

            // *T to *[1]T
            single_item: {
                if (dest_info.flags.size != .one) break :single_item;
                if (!inst_ty.isSinglePointer(zcu)) break :single_item;
                if (!sema.checkPtrAttributes(dest_ty, inst_ty, &in_memory_result)) break :pointer;
                const ptr_elem_ty = inst_ty.childType(zcu);
                const array_ty = Type.fromInterned(dest_info.child);
                if (array_ty.zigTypeTag(zcu) != .array) break :single_item;
                const array_elem_ty = array_ty.childType(zcu);
                if (array_ty.arrayLen(zcu) != 1) break :single_item;
                const dest_is_mut = !dest_info.flags.is_const;
                switch (try sema.coerceInMemoryAllowed(block, array_elem_ty, ptr_elem_ty, dest_is_mut, target, dest_ty_src, inst_src, maybe_inst_val)) {
                    .ok => {},
                    else => break :single_item,
                }
                return sema.coerceCompatiblePtrs(block, dest_ty, inst, inst_src);
            }

            // Coercions where the source is a single pointer to an array.
            src_array_ptr: {
                if (!inst_ty.isSinglePointer(zcu)) break :src_array_ptr;
                if (dest_info.flags.size == .one) break :src_array_ptr; // `*[n]T` -> `*T` isn't valid
                if (!sema.checkPtrAttributes(dest_ty, inst_ty, &in_memory_result)) break :pointer;
                const array_ty = inst_ty.childType(zcu);
                if (array_ty.zigTypeTag(zcu) != .array) break :src_array_ptr;
                const array_elem_type = array_ty.childType(zcu);
                const dest_is_mut = !dest_info.flags.is_const;

                const dst_elem_type = Type.fromInterned(dest_info.child);
                const elem_res = try sema.coerceInMemoryAllowed(block, dst_elem_type, array_elem_type, dest_is_mut, target, dest_ty_src, inst_src, maybe_inst_val);
                switch (elem_res) {
                    .ok => {},
                    else => {
                        in_memory_result = .{ .ptr_child = .{
                            .child = try elem_res.dupe(sema.arena),
                            .actual = array_elem_type,
                            .wanted = dst_elem_type,
                        } };
                        break :src_array_ptr;
                    },
                }

                if (dest_info.sentinel != .none) {
                    if (array_ty.sentinel(zcu)) |inst_sent| {
                        if (Air.internedToRef(dest_info.sentinel) !=
                            try sema.coerceInMemory(inst_sent, dst_elem_type))
                        {
                            in_memory_result = .{ .ptr_sentinel = .{
                                .actual = inst_sent,
                                .wanted = Value.fromInterned(dest_info.sentinel),
                                .ty = dst_elem_type,
                            } };
                            break :src_array_ptr;
                        }
                    } else {
                        in_memory_result = .{ .ptr_sentinel = .{
                            .actual = Value.@"unreachable",
                            .wanted = Value.fromInterned(dest_info.sentinel),
                            .ty = dst_elem_type,
                        } };
                        break :src_array_ptr;
                    }
                }

                switch (dest_info.flags.size) {
                    .slice => {
                        // *[N]T to []T
                        return sema.coerceArrayPtrToSlice(block, dest_ty, inst, inst_src);
                    },
                    .c => {
                        // *[N]T to [*c]T
                        return sema.coerceCompatiblePtrs(block, dest_ty, inst, inst_src);
                    },
                    .many => {
                        // *[N]T to [*]T
                        return sema.coerceCompatiblePtrs(block, dest_ty, inst, inst_src);
                    },
                    .one => unreachable, // early exit at top of block
                }
            }

            // coercion from C pointer
            if (inst_ty.isCPtr(zcu)) src_c_ptr: {
                if (dest_info.flags.size == .slice) break :src_c_ptr;
                if (!sema.checkPtrAttributes(dest_ty, inst_ty, &in_memory_result)) break :src_c_ptr;
                // In this case we must add a safety check because the C pointer
                // could be null.
                const src_elem_ty = inst_ty.childType(zcu);
                const dest_is_mut = !dest_info.flags.is_const;
                const dst_elem_type = Type.fromInterned(dest_info.child);
                switch (try sema.coerceInMemoryAllowed(block, dst_elem_type, src_elem_ty, dest_is_mut, target, dest_ty_src, inst_src, maybe_inst_val)) {
                    .ok => {},
                    else => break :src_c_ptr,
                }
                return sema.coerceCompatiblePtrs(block, dest_ty, inst, inst_src);
            }

            // cast from *T and [*]T to *anyopaque
            // but don't do it if the source type is a double pointer
            if (dest_info.child == .anyopaque_type and inst_ty.zigTypeTag(zcu) == .pointer) to_anyopaque: {
                if (!sema.checkPtrAttributes(dest_ty, inst_ty, &in_memory_result)) break :pointer;
                const elem_ty = inst_ty.elemType2(zcu);
                if (elem_ty.zigTypeTag(zcu) == .pointer or elem_ty.isPtrLikeOptional(zcu)) {
                    in_memory_result = .{ .double_ptr_to_anyopaque = .{
                        .actual = inst_ty,
                        .wanted = dest_ty,
                    } };
                    break :pointer;
                }
                if (dest_ty.isSlice(zcu)) break :to_anyopaque;
                if (inst_ty.isSlice(zcu)) {
                    in_memory_result = .{ .slice_to_anyopaque = .{
                        .actual = inst_ty,
                        .wanted = dest_ty,
                    } };
                    break :pointer;
                }
                return sema.coerceCompatiblePtrs(block, dest_ty, inst, inst_src);
            }

            switch (dest_info.flags.size) {
                // coercion to C pointer
                .c => switch (inst_ty.zigTypeTag(zcu)) {
                    .null => return Air.internedToRef(try pt.intern(.{ .ptr = .{
                        .ty = dest_ty.toIntern(),
                        .base_addr = .int,
                        .byte_offset = 0,
                    } })),
                    .comptime_int => {
                        const addr = sema.coerceExtra(block, Type.usize, inst, inst_src, .{ .report_err = false }) catch |err| switch (err) {
                            error.NotCoercible => break :pointer,
                            else => |e| return e,
                        };
                        return try sema.coerceCompatiblePtrs(block, dest_ty, addr, inst_src);
                    },
                    .int => {
                        const ptr_size_ty = switch (inst_ty.intInfo(zcu).signedness) {
                            .signed => Type.isize,
                            .unsigned => Type.usize,
                        };
                        const addr = sema.coerceExtra(block, ptr_size_ty, inst, inst_src, .{ .report_err = false }) catch |err| switch (err) {
                            error.NotCoercible => {
                                // Try to give more useful notes
                                in_memory_result = try sema.coerceInMemoryAllowed(block, ptr_size_ty, inst_ty, false, target, dest_ty_src, inst_src, maybe_inst_val);
                                break :pointer;
                            },
                            else => |e| return e,
                        };
                        return try sema.coerceCompatiblePtrs(block, dest_ty, addr, inst_src);
                    },
                    .pointer => p: {
                        if (!sema.checkPtrAttributes(dest_ty, inst_ty, &in_memory_result)) break :p;
                        const inst_info = inst_ty.ptrInfo(zcu);
                        switch (try sema.coerceInMemoryAllowed(
                            block,
                            Type.fromInterned(dest_info.child),
                            Type.fromInterned(inst_info.child),
                            !dest_info.flags.is_const,
                            target,
                            dest_ty_src,
                            inst_src,
                            maybe_inst_val,
                        )) {
                            .ok => {},
                            else => break :p,
                        }
                        if (inst_info.flags.size == .slice) {
                            assert(dest_info.sentinel == .none);
                            if (inst_info.sentinel == .none or
                                inst_info.sentinel != (try pt.intValue(Type.fromInterned(inst_info.child), 0)).toIntern())
                                break :p;

                            const slice_ptr = try sema.analyzeSlicePtr(block, inst_src, inst, inst_ty);
                            return sema.coerceCompatiblePtrs(block, dest_ty, slice_ptr, inst_src);
                        }
                        return sema.coerceCompatiblePtrs(block, dest_ty, inst, inst_src);
                    },
                    else => {},
                },
                .one => {},
                .slice => to_slice: {
                    if (inst_ty.zigTypeTag(zcu) == .array) {
                        return sema.fail(
                            block,
                            inst_src,
                            "array literal requires address-of operator (&) to coerce to slice type '{}'",
                            .{dest_ty.fmt(pt)},
                        );
                    }

                    if (!inst_ty.isSinglePointer(zcu)) break :to_slice;
                    const inst_child_ty = inst_ty.childType(zcu);
                    if (!inst_child_ty.isTuple(zcu)) break :to_slice;

                    // empty tuple to zero-length slice
                    // note that this allows coercing to a mutable slice.
                    if (inst_child_ty.structFieldCount(zcu) == 0) {
                        const align_val = try dest_ty.ptrAlignmentSema(pt);
                        return Air.internedToRef(try pt.intern(.{ .slice = .{
                            .ty = dest_ty.toIntern(),
                            .ptr = try pt.intern(.{ .ptr = .{
                                .ty = dest_ty.slicePtrFieldType(zcu).toIntern(),
                                .base_addr = .int,
                                .byte_offset = align_val.toByteUnits().?,
                            } }),
                            .len = .zero_usize,
                        } }));
                    }

                    // pointer to tuple to slice
                    if (!dest_info.flags.is_const) {
                        const err_msg = err_msg: {
                            const err_msg = try sema.errMsg(inst_src, "cannot cast pointer to tuple to '{}'", .{dest_ty.fmt(pt)});
                            errdefer err_msg.destroy(sema.gpa);
                            try sema.errNote(dest_ty_src, err_msg, "pointers to tuples can only coerce to constant pointers", .{});
                            break :err_msg err_msg;
                        };
                        return sema.failWithOwnedErrorMsg(block, err_msg);
                    }
                    return sema.coerceTupleToSlicePtrs(block, dest_ty, dest_ty_src, inst, inst_src);
                },
                .many => p: {
                    if (!inst_ty.isSlice(zcu)) break :p;
                    if (!sema.checkPtrAttributes(dest_ty, inst_ty, &in_memory_result)) break :p;
                    const inst_info = inst_ty.ptrInfo(zcu);

                    switch (try sema.coerceInMemoryAllowed(
                        block,
                        Type.fromInterned(dest_info.child),
                        Type.fromInterned(inst_info.child),
                        !dest_info.flags.is_const,
                        target,
                        dest_ty_src,
                        inst_src,
                        maybe_inst_val,
                    )) {
                        .ok => {},
                        else => break :p,
                    }

                    if (dest_info.sentinel == .none or inst_info.sentinel == .none or
                        Air.internedToRef(dest_info.sentinel) !=
                            try sema.coerceInMemory(Value.fromInterned(inst_info.sentinel), Type.fromInterned(dest_info.child)))
                        break :p;

                    const slice_ptr = try sema.analyzeSlicePtr(block, inst_src, inst, inst_ty);
                    return sema.coerceCompatiblePtrs(block, dest_ty, slice_ptr, inst_src);
                },
            }
        },
        .int, .comptime_int => switch (inst_ty.zigTypeTag(zcu)) {
            .float, .comptime_float => float: {
                const val = maybe_inst_val orelse {
                    if (dest_ty.zigTypeTag(zcu) == .comptime_int) {
                        if (!opts.report_err) return error.NotCoercible;
                        return sema.failWithNeededComptime(block, inst_src, .{ .simple = .casted_to_comptime_int });
                    }
                    break :float;
                };
                const result_val = try sema.intFromFloat(block, inst_src, val, inst_ty, dest_ty, .exact);
                return Air.internedToRef(result_val.toIntern());
            },
            .int, .comptime_int => {
                if (maybe_inst_val) |val| {
                    // comptime-known integer to other number
                    if (!(try sema.intFitsInType(val, dest_ty, null))) {
                        if (!opts.report_err) return error.NotCoercible;
                        return sema.fail(block, inst_src, "type '{}' cannot represent integer value '{}'", .{ dest_ty.fmt(pt), val.fmtValueSema(pt, sema) });
                    }
                    return switch (zcu.intern_pool.indexToKey(val.toIntern())) {
                        .undef => try pt.undefRef(dest_ty),
                        .int => |int| Air.internedToRef(
                            try zcu.intern_pool.getCoercedInts(zcu.gpa, pt.tid, int, dest_ty.toIntern()),
                        ),
                        else => unreachable,
                    };
                }
                if (dest_ty.zigTypeTag(zcu) == .comptime_int) {
                    if (!opts.report_err) return error.NotCoercible;
                    if (opts.no_cast_to_comptime_int) return inst;
                    return sema.failWithNeededComptime(block, inst_src, .{ .simple = .casted_to_comptime_int });
                }

                // integer widening
                const dst_info = dest_ty.intInfo(zcu);
                const src_info = inst_ty.intInfo(zcu);
                if ((src_info.signedness == dst_info.signedness and dst_info.bits >= src_info.bits) or
                    // small enough unsigned ints can get casted to large enough signed ints
                    (dst_info.signedness == .signed and dst_info.bits > src_info.bits))
                {
                    try sema.requireRuntimeBlock(block, inst_src, null);
                    return block.addTyOp(.intcast, dest_ty, inst);
                }
            },
            else => {},
        },
        .float, .comptime_float => switch (inst_ty.zigTypeTag(zcu)) {
            .comptime_float => {
                const val = try sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, inst, undefined);
                const result_val = try val.floatCast(dest_ty, pt);
                return Air.internedToRef(result_val.toIntern());
            },
            .float => {
                if (maybe_inst_val) |val| {
                    const result_val = try val.floatCast(dest_ty, pt);
                    if (!val.eql(try result_val.floatCast(inst_ty, pt), inst_ty, zcu)) {
                        return sema.fail(
                            block,
                            inst_src,
                            "type '{}' cannot represent float value '{}'",
                            .{ dest_ty.fmt(pt), val.fmtValueSema(pt, sema) },
                        );
                    }
                    return Air.internedToRef(result_val.toIntern());
                } else if (dest_ty.zigTypeTag(zcu) == .comptime_float) {
                    if (!opts.report_err) return error.NotCoercible;
                    return sema.failWithNeededComptime(block, inst_src, .{ .simple = .casted_to_comptime_float });
                }

                // float widening
                const src_bits = inst_ty.floatBits(target);
                const dst_bits = dest_ty.floatBits(target);
                if (dst_bits >= src_bits) {
                    try sema.requireRuntimeBlock(block, inst_src, null);
                    return block.addTyOp(.fpext, dest_ty, inst);
                }
            },
            .int, .comptime_int => int: {
                const val = maybe_inst_val orelse {
                    if (dest_ty.zigTypeTag(zcu) == .comptime_float) {
                        if (!opts.report_err) return error.NotCoercible;
                        return sema.failWithNeededComptime(block, inst_src, .{ .simple = .casted_to_comptime_float });
                    }
                    break :int;
                };
                const result_val = try val.floatFromIntAdvanced(sema.arena, inst_ty, dest_ty, pt, .sema);
                // TODO implement this compile error
                //const int_again_val = try result_val.intFromFloat(sema.arena, inst_ty);
                //if (!int_again_val.eql(val, inst_ty, zcu)) {
                //    return sema.fail(
                //        block,
                //        inst_src,
                //        "type '{}' cannot represent integer value '{}'",
                //        .{ dest_ty.fmt(pt), val },
                //    );
                //}
                return Air.internedToRef(result_val.toIntern());
            },
            else => {},
        },
        .@"enum" => switch (inst_ty.zigTypeTag(zcu)) {
            .enum_literal => {
                // enum literal to enum
                const val = try sema.resolveConstDefinedValue(block, LazySrcLoc.unneeded, inst, undefined);
                const string = zcu.intern_pool.indexToKey(val.toIntern()).enum_literal;
                const field_index = dest_ty.enumFieldIndex(string, zcu) orelse {
                    return sema.fail(block, inst_src, "no field named '{}' in enum '{}'", .{
                        string.fmt(&zcu.intern_pool), dest_ty.fmt(pt),
                    });
                };
                return Air.internedToRef((try pt.enumValueFieldIndex(dest_ty, @intCast(field_index))).toIntern());
            },
            .@"union" => blk: {
                // union to its own tag type
                const union_tag_ty = inst_ty.unionTagType(zcu) orelse break :blk;
                if (union_tag_ty.eql(dest_ty, zcu)) {
                    return sema.unionToTag(block, dest_ty, inst, inst_src);
                }
            },
            else => {},
        },
        .error_union => switch (inst_ty.zigTypeTag(zcu)) {
            .error_union => eu: {
                if (maybe_inst_val) |inst_val| {
                    switch (inst_val.toIntern()) {
                        .undef => return pt.undefRef(dest_ty),
                        else => switch (zcu.intern_pool.indexToKey(inst_val.toIntern())) {
                            .error_union => |error_union| switch (error_union.val) {
                                .err_name => |err_name| {
                                    const error_set_ty = inst_ty.errorUnionSet(zcu);
                                    const error_set_val = Air.internedToRef((try pt.intern(.{ .err = .{
                                        .ty = error_set_ty.toIntern(),
                                        .name = err_name,
                                    } })));
                                    return sema.wrapErrorUnionSet(block, dest_ty, error_set_val, inst_src);
                                },
                                .payload => |payload| {
                                    const payload_val = Air.internedToRef(payload);
                                    return sema.wrapErrorUnionPayload(block, dest_ty, payload_val, inst_src) catch |err| switch (err) {
                                        error.NotCoercible => break :eu,
                                        else => |e| return e,
                                    };
                                },
                            },
                            else => unreachable,
                        },
                    }
                }
            },
            .error_set => {
                // E to E!T
                return sema.wrapErrorUnionSet(block, dest_ty, inst, inst_src);
            },
            else => eu: {
                // T to E!T
                return sema.wrapErrorUnionPayload(block, dest_ty, inst, inst_src) catch |err| switch (err) {
                    error.NotCoercible => {
                        if (in_memory_result == .no_match) {
                            const payload_type = dest_ty.errorUnionPayload(zcu);
                            // Try to give more useful notes
                            in_memory_result = try sema.coerceInMemoryAllowed(block, payload_type, inst_ty, false, target, dest_ty_src, inst_src, maybe_inst_val);
                        }
                        break :eu;
                    },
                    else => |e| return e,
                };
            },
        },
        .@"union" => switch (inst_ty.zigTypeTag(zcu)) {
            .@"enum", .enum_literal => return sema.coerceEnumToUnion(block, dest_ty, dest_ty_src, inst, inst_src),
            else => {},
        },
        .array => switch (inst_ty.zigTypeTag(zcu)) {
            .array => array_to_array: {
                // Array coercions are allowed only if the child is IMC and the sentinel is unchanged or removed.
                if (.ok != try sema.coerceInMemoryAllowed(
                    block,
                    dest_ty.childType(zcu),
                    inst_ty.childType(zcu),
                    false,
                    target,
                    dest_ty_src,
                    inst_src,
                    maybe_inst_val,
                )) {
                    break :array_to_array;
                }

                if (dest_ty.sentinel(zcu)) |dest_sent| {
                    const src_sent = inst_ty.sentinel(zcu) orelse break :array_to_array;
                    if (dest_sent.toIntern() != (try pt.getCoerced(src_sent, dest_ty.childType(zcu))).toIntern()) {
                        break :array_to_array;
                    }
                }

                return sema.coerceArrayLike(block, dest_ty, dest_ty_src, inst, inst_src);
            },
            .vector => return sema.coerceArrayLike(block, dest_ty, dest_ty_src, inst, inst_src),
            .@"struct" => {
                if (inst_ty.isTuple(zcu)) {
                    return sema.coerceTupleToArray(block, dest_ty, dest_ty_src, inst, inst_src);
                }
            },
            else => {},
        },
        .vector => switch (inst_ty.zigTypeTag(zcu)) {
            .array, .vector => return sema.coerceArrayLike(block, dest_ty, dest_ty_src, inst, inst_src),
            .@"struct" => {
                if (inst_ty.isTuple(zcu)) {
                    return sema.coerceTupleToArray(block, dest_ty, dest_ty_src, inst, inst_src);
                }
            },
            else => {},
        },
        .@"struct" => blk: {
            if (dest_ty.isTuple(zcu) and inst_ty.isTuple(zcu)) {
                return sema.coerceTupleToTuple(block, dest_ty, inst, inst_src) catch |err| switch (err) {
                    error.NotCoercible => break :blk,
                    else => |e| return e,
                };
            }
        },
        else => {},
    }

    const can_coerce_to = switch (dest_ty.zigTypeTag(zcu)) {
        .noreturn, .@"opaque" => false,
        else => true,
    };

    if (can_coerce_to) {
        // undefined to anything. We do this after the big switch above so that
        // special logic has a chance to run first, such as `*[N]T` to `[]T` which
        // should initialize the length field of the slice.
        if (maybe_inst_val) |val| if (val.toIntern() == .undef) return pt.undefRef(dest_ty);
    }

    if (!opts.report_err) return error.NotCoercible;

    if (opts.is_ret and dest_ty.zigTypeTag(zcu) == .noreturn) {
        const msg = msg: {
            const msg = try sema.errMsg(inst_src, "function declared 'noreturn' returns", .{});
            errdefer msg.destroy(sema.gpa);

            const ret_ty_src: LazySrcLoc = .{
                .base_node_inst = ip.getNav(zcu.funcInfo(sema.func_index).owner_nav).srcInst(ip),
                .offset = .{ .node_offset_fn_type_ret_ty = .zero },
            };
            try sema.errNote(ret_ty_src, msg, "'noreturn' declared here", .{});
            break :msg msg;
        };
        return sema.failWithOwnedErrorMsg(block, msg);
    }

    const msg = msg: {
        const msg = try sema.errMsg(inst_src, "expected type '{}', found '{}'", .{ dest_ty.fmt(pt), inst_ty.fmt(pt) });
        errdefer msg.destroy(sema.gpa);

        if (!can_coerce_to) {
            try sema.errNote(inst_src, msg, "cannot coerce to '{}'", .{dest_ty.fmt(pt)});
        }

        // E!T to T
        if (inst_ty.zigTypeTag(zcu) == .error_union and
            (try sema.coerceInMemoryAllowed(block, inst_ty.errorUnionPayload(zcu), dest_ty, false, target, dest_ty_src, inst_src, maybe_inst_val)) == .ok)
        {
            try sema.errNote(inst_src, msg, "cannot convert error union to payload type", .{});
            try sema.errNote(inst_src, msg, "consider using 'try', 'catch', or 'if'", .{});
        }

        // ?T to T
        if (inst_ty.zigTypeTag(zcu) == .optional and
            (try sema.coerceInMemoryAllowed(block, inst_ty.optionalChild(zcu), dest_ty, false, target, dest_ty_src, inst_src, maybe_inst_val)) == .ok)
        {
            try sema.errNote(inst_src, msg, "cannot convert optional to payload type", .{});
            try sema.errNote(inst_src, msg, "consider using '.?', 'orelse', or 'if'", .{});
        }

        try in_memory_result.report(sema, inst_src, msg);

        // Add notes about function return type
        if (opts.is_ret and
            !zcu.test_functions.contains(zcu.funcInfo(sema.func_index).owner_nav))
        {
            const ret_ty_src: LazySrcLoc = .{
                .base_node_inst = ip.getNav(zcu.funcInfo(sema.func_index).owner_nav).srcInst(ip),
                .offset = .{ .node_offset_fn_type_ret_ty = .zero },
            };
            if (inst_ty.isError(zcu) and !dest_ty.isError(zcu)) {
                try sema.errNote(ret_ty_src, msg, "function cannot return an error", .{});
            } else {
                try sema.errNote(ret_ty_src, msg, "function return type declared here", .{});
            }
        }

        if (try opts.param_src.get(sema)) |param_src| {
            try sema.errNote(param_src, msg, "parameter type declared here", .{});
        }

        // TODO maybe add "cannot store an error in type '{}'" note

        break :msg msg;
    };
    return sema.failWithOwnedErrorMsg(block, msg);
}```
嗯，这段代码看起来是关于Zig编程语言中的类型强制转换的函数。函数名是coerceExtra，参数包括Sema、Block、目标类型dest_ty、指令inst，还有一些选项opts。返回类型是CoersionError!Air.Inst.Ref，看起来是处理类型转换时可能出现的错误以及转换后的指令引用。

首先，我注意到代码中很多条件判断，比如检查dest_ty是否是泛型毒药类型，如果是的话就直接返回inst。然后比较目标类型和实例类型是否相同，如果相同就直接返回实例，不需要转换。这可能是一个优化步骤，避免不必要的转换操作。

接下来，代码处理了很多不同的类型情况，比如可选类型（optional）、指针类型（pointer）、整数类型（int）、浮点类型（float）、枚举类型（enum）、错误联合类型（error_union）等等。每个类型分支下都有相应的处理逻辑，例如将T类型转换为可选类型?T，处理指针的各种情况（如单指针到数组指针的转换），以及整数和浮点数之间的转换。

在处理指针时，代码检查了指针的属性，比如是否是常量指针、指针的大小（如slice、c指针等），并进行相应的转换。例如，将*[N]T转换为[]T的切片类型，或者处理C指针到Zig指针的转换。

对于错误联合类型（error_union），代码处理了将错误集或有效负载类型转换为错误联合类型的情况，并确保错误类型和目标类型的一致性。

此外，代码还处理了一些特殊情况，比如将undefined值转换为任意类型，或者在类型无法转换时生成相应的错误信息，包括提供有用的错误提示，例如建议使用try、catch或if来处理错误联合类型。

整体来看，这个函数的主要作用是根据目标类型和实例类型的具体情况，执行相应的类型转换逻辑，处理各种可能的类型组合，并在无法转换时生成详细的错误信息。逻辑非常复杂，涉及大量的类型检查和转换规则，需要深入理解Zig的类型系统才能完全掌握。

不过，有些部分我还不太明白，比如如何处理泛型类型，或者某些特定的指针转换逻辑。可能需要进一步查阅Zig的类型系统文档或相关代码来理解这些细节。此外，错误处理部分也比较复杂，需要确保在所有可能的错误路径上都正确地生成错误信息，并返回相应的错误代码。

总之，这段代码展示了类型强制转换在编译器中的实现细节，涉及到多种类型的处理和各种边界情况的考虑，是一个复杂但关键的函数。
================================================
这段代码是 Zig 编译器中进行类型强制转换的核心逻辑，其思维模型可以总结为以下层次：

---

### **1. 基础逻辑**
- **目标**：将输入值 `inst` 的类型 `inst_ty` 强制转换为目标类型 `dest_ty`。
- **快速返回**：
  - 若 `dest_ty` 是泛型毒药（无法具体化），直接返回原值。
  - 若 `dest_ty` 和 `inst_ty` 类型严格相等，直接返回原值。

---

### **2. 内存兼容性检查**
- 通过 `coerceInMemoryAllowed` 检查类型是否可以直接在内存中兼容（例如布局、对齐等）。
- 若兼容：
  - 对编译期已知值（`maybe_inst_val`），直接转换。
  - 对运行时值，插入位转换指令（`addBitCast`）或生成运行时检查逻辑。

---

### **3. 类型分派处理**
代码按 `dest_ty` 的类别分派处理逻辑：

#### **可选类型（Optional）**
- **`null` 或 `undef`**：直接转换为目标可选类型。
- **子类型转换**：将 `T` 转换为 `?T`，递归处理子类型 `T` 的兼容性。
- **指针类可选类型**：处理 `?*T` 到 `?*anyopaque` 的特殊转换。

#### **指针类型（Pointer）**
- **函数到函数指针**：将函数体转换为函数指针。
- **指针到数组**：如 `*T` 转 `*[1]T`。
- **数组指针到切片**：如 `*[N]T` 转 `[]T`。
- **C 指针兼容性**：处理 `[*c]T` 的转换和空指针检查。
- **`*anyopaque` 转换**：禁止双重指针（如 `**T`）到 `*anyopaque`。

#### **整数与浮点**
- **编译期值**：直接检查值范围是否匹配目标类型。
- **运行时值**：插入类型转换指令（如 `intcast`、`fpext`）。
- **跨类型转换**：如整数转浮点，需检查精度和范围。

#### **枚举与联合**
- **枚举字面量转枚举**：通过字段名匹配。
- **联合到标签类型**：提取联合的标签值。

#### **错误联合（Error Union）**
- **错误集转错误联合**：包装错误到 `E!T`。
- **有效负载转错误联合**：包装有效负载到 `E!T`。

#### **数组与元组**
- **数组到数组**：检查元素类型和哨兵值兼容性。
- **元组到数组**：将元组字段按顺序转换为数组元素。

---

### **4. 特殊处理**
- **`undefined` 值**：可直接转换为任意类型。
- **`noreturn` 类型**：检查函数返回值是否非法。

---

### **5. 错误处理**
- **类型不匹配**：生成详细的错误信息，包括：
  - 预期类型与实际类型对比。
  - 指针子类型、哨兵值、内存布局等不匹配原因。
  - 建议操作（如使用 `try`、`catch` 处理错误联合）。
- **上下文补充**：关联函数返回类型或参数声明位置，辅助定位问题。

---

### **总结**
该函数通过递归类型检查、内存布局验证和分派处理，实现了 Zig 复杂的类型强制规则。核心思想是：
1. **尽可能直接转换**（如内存兼容或编译期已知值）。
2. **分类型细化规则**（如指针、可选类型的特殊逻辑）。
3. **精确错误引导**（通过详细错误信息帮助开发者理解类型不匹配的原因）。

这种设计确保了类型安全，同时兼顾编译时优化和用户友好的错误提示。