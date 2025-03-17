```zig
pub const DeclGen = struct {
    gpa: mem.Allocator,
    pt: Zcu.PerThread,
    mod: *Module,
    pass: Pass,
    is_naked_fn: bool,
    expected_block: ?u32,
    /// This is a borrowed reference from `link.C`.
    fwd_decl: std.ArrayList(u8),
    error_msg: ?*Zcu.ErrorMsg,
    ctype_pool: CType.Pool,
    scratch: std.ArrayListUnmanaged(u32),
    /// Keeps track of anonymous decls that need to be rendered before this
    /// (named) Decl in the output C code.
    uav_deps: std.AutoArrayHashMapUnmanaged(InternPool.Index, C.AvBlock),
    aligned_uavs: std.AutoArrayHashMapUnmanaged(InternPool.Index, Alignment),

    pub const Pass = union(enum) {
        nav: InternPool.Nav.Index,
        uav: InternPool.Index,
        flush,
    };

    fn fwdDeclWriter(dg: *DeclGen) ArrayListWriter {
        return arrayListWriter(&dg.fwd_decl);
    }

    fn fail(dg: *DeclGen, comptime format: []const u8, args: anytype) error{ AnalysisFail, OutOfMemory } {
        @branchHint(.cold);
        const zcu = dg.pt.zcu;
        const src_loc = zcu.navSrcLoc(dg.pass.nav);
        dg.error_msg = try Zcu.ErrorMsg.create(dg.gpa, src_loc, format, args);
        return error.AnalysisFail;
    }

    fn renderUav(
        dg: *DeclGen,
        writer: anytype,
        uav: InternPool.Key.Ptr.BaseAddr.Uav,
        location: ValueRenderLocation,
    ) error{ OutOfMemory, AnalysisFail }!void {
        const pt = dg.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const ctype_pool = &dg.ctype_pool;
        const uav_val = Value.fromInterned(uav.val);
        const uav_ty = uav_val.typeOf(zcu);

        // Render an undefined pointer if we have a pointer to a zero-bit or comptime type.
        const ptr_ty: Type = .fromInterned(uav.orig_ty);
        if (ptr_ty.isPtrAtRuntime(zcu) and !uav_ty.isFnOrHasRuntimeBits(zcu)) {
            return dg.writeCValue(writer, .{ .undef = ptr_ty });
        }

        // Chase function values in order to be able to reference the original function.
        switch (ip.indexToKey(uav.val)) {
            .variable => unreachable,
            .func => |func| return dg.renderNav(writer, func.owner_nav, location),
            .@"extern" => |@"extern"| return dg.renderNav(writer, @"extern".owner_nav, location),
            else => {},
        }

        // We shouldn't cast C function pointers as this is UB (when you call
        // them).  The analysis until now should ensure that the C function
        // pointers are compatible.  If they are not, then there is a bug
        // somewhere and we should let the C compiler tell us about it.
        const ptr_ctype = try dg.ctypeFromType(ptr_ty, .complete);
        const elem_ctype = ptr_ctype.info(ctype_pool).pointer.elem_ctype;
        const uav_ctype = try dg.ctypeFromType(uav_ty, .complete);
        const need_cast = !elem_ctype.eql(uav_ctype) and
            (elem_ctype.info(ctype_pool) != .function or uav_ctype.info(ctype_pool) != .function);
        if (need_cast) {
            try writer.writeAll("((");
            try dg.renderCType(writer, ptr_ctype);
            try writer.writeByte(')');
        }
        try writer.writeByte('&');
        try renderUavName(writer, uav_val);
        if (need_cast) try writer.writeByte(')');

        // Indicate that the anon decl should be rendered to the output so that
        // our reference above is not undefined.
        const ptr_type = ip.indexToKey(uav.orig_ty).ptr_type;
        const gop = try dg.uav_deps.getOrPut(dg.gpa, uav.val);
        if (!gop.found_existing) gop.value_ptr.* = .{};

        // Only insert an alignment entry if the alignment is greater than ABI
        // alignment. If there is already an entry, keep the greater alignment.
        const explicit_alignment = ptr_type.flags.alignment;
        if (explicit_alignment != .none) {
            const abi_alignment = Type.fromInterned(ptr_type.child).abiAlignment(zcu);
            if (explicit_alignment.order(abi_alignment).compare(.gt)) {
                const aligned_gop = try dg.aligned_uavs.getOrPut(dg.gpa, uav.val);
                aligned_gop.value_ptr.* = if (aligned_gop.found_existing)
                    aligned_gop.value_ptr.maxStrict(explicit_alignment)
                else
                    explicit_alignment;
            }
        }
    }

    fn renderNav(
        dg: *DeclGen,
        writer: anytype,
        nav_index: InternPool.Nav.Index,
        location: ValueRenderLocation,
    ) error{ OutOfMemory, AnalysisFail }!void {
        _ = location;
        const pt = dg.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const ctype_pool = &dg.ctype_pool;

        // Chase function values in order to be able to reference the original function.
        const owner_nav = switch (ip.getNav(nav_index).status) {
            .unresolved => unreachable,
            .type_resolved => nav_index, // this can't be an extern or a function
            .fully_resolved => |r| switch (ip.indexToKey(r.val)) {
                .func => |f| f.owner_nav,
                .@"extern" => |e| e.owner_nav,
                else => nav_index,
            },
        };

        // Render an undefined pointer if we have a pointer to a zero-bit or comptime type.
        const nav_ty: Type = .fromInterned(ip.getNav(owner_nav).typeOf(ip));
        const ptr_ty = try pt.navPtrType(owner_nav);
        if (!nav_ty.isFnOrHasRuntimeBits(zcu)) {
            return dg.writeCValue(writer, .{ .undef = ptr_ty });
        }

        // We shouldn't cast C function pointers as this is UB (when you call
        // them).  The analysis until now should ensure that the C function
        // pointers are compatible.  If they are not, then there is a bug
        // somewhere and we should let the C compiler tell us about it.
        const ctype = try dg.ctypeFromType(ptr_ty, .complete);
        const elem_ctype = ctype.info(ctype_pool).pointer.elem_ctype;
        const nav_ctype = try dg.ctypeFromType(nav_ty, .complete);
        const need_cast = !elem_ctype.eql(nav_ctype) and
            (elem_ctype.info(ctype_pool) != .function or nav_ctype.info(ctype_pool) != .function);
        if (need_cast) {
            try writer.writeAll("((");
            try dg.renderCType(writer, ctype);
            try writer.writeByte(')');
        }
        try writer.writeByte('&');
        try dg.renderNavName(writer, owner_nav);
        if (need_cast) try writer.writeByte(')');
    }

    fn renderPointer(
        dg: *DeclGen,
        writer: anytype,
        derivation: Value.PointerDeriveStep,
        location: ValueRenderLocation,
    ) error{ OutOfMemory, AnalysisFail }!void {
        const pt = dg.pt;
        const zcu = pt.zcu;
        switch (derivation) {
            .comptime_alloc_ptr, .comptime_field_ptr => unreachable,
            .int => |int| {
                const ptr_ctype = try dg.ctypeFromType(int.ptr_ty, .complete);
                const addr_val = try pt.intValue(.usize, int.addr);
                try writer.writeByte('(');
                try dg.renderCType(writer, ptr_ctype);
                try writer.print("){x}", .{try dg.fmtIntLiteral(addr_val, .Other)});
            },

            .nav_ptr => |nav| try dg.renderNav(writer, nav, location),
            .uav_ptr => |uav| try dg.renderUav(writer, uav, location),

            inline .eu_payload_ptr, .opt_payload_ptr => |info| {
                try writer.writeAll("&(");
                try dg.renderPointer(writer, info.parent.*, location);
                try writer.writeAll(")->payload");
            },

            .field_ptr => |field| {
                const parent_ptr_ty = try field.parent.ptrType(pt);

                // Ensure complete type definition is available before accessing fields.
                _ = try dg.ctypeFromType(parent_ptr_ty.childType(zcu), .complete);

                switch (fieldLocation(parent_ptr_ty, field.result_ptr_ty, field.field_idx, pt)) {
                    .begin => {
                        const ptr_ctype = try dg.ctypeFromType(field.result_ptr_ty, .complete);
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ptr_ctype);
                        try writer.writeByte(')');
                        try dg.renderPointer(writer, field.parent.*, location);
                    },
                    .field => |name| {
                        try writer.writeAll("&(");
                        try dg.renderPointer(writer, field.parent.*, location);
                        try writer.writeAll(")->");
                        try dg.writeCValue(writer, name);
                    },
                    .byte_offset => |byte_offset| {
                        const ptr_ctype = try dg.ctypeFromType(field.result_ptr_ty, .complete);
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ptr_ctype);
                        try writer.writeByte(')');
                        const offset_val = try pt.intValue(.usize, byte_offset);
                        try writer.writeAll("((char *)");
                        try dg.renderPointer(writer, field.parent.*, location);
                        try writer.print(" + {})", .{try dg.fmtIntLiteral(offset_val, .Other)});
                    },
                }
            },

            .elem_ptr => |elem| if (!(try elem.parent.ptrType(pt)).childType(zcu).hasRuntimeBits(zcu)) {
                // Element type is zero-bit, so lowers to `void`. The index is irrelevant; just cast the pointer.
                const ptr_ctype = try dg.ctypeFromType(elem.result_ptr_ty, .complete);
                try writer.writeByte('(');
                try dg.renderCType(writer, ptr_ctype);
                try writer.writeByte(')');
                try dg.renderPointer(writer, elem.parent.*, location);
            } else {
                const index_val = try pt.intValue(.usize, elem.elem_idx);
                // We want to do pointer arithmetic on a pointer to the element type.
                // We might have a pointer-to-array. In this case, we must cast first.
                const result_ctype = try dg.ctypeFromType(elem.result_ptr_ty, .complete);
                const parent_ctype = try dg.ctypeFromType(try elem.parent.ptrType(pt), .complete);
                if (result_ctype.eql(parent_ctype)) {
                    // The pointer already has an appropriate type - just do the arithmetic.
                    try writer.writeByte('(');
                    try dg.renderPointer(writer, elem.parent.*, location);
                    try writer.print(" + {})", .{try dg.fmtIntLiteral(index_val, .Other)});
                } else {
                    // We probably have an array pointer `T (*)[n]`. Cast to an element pointer,
                    // and *then* apply the index.
                    try writer.writeAll("((");
                    try dg.renderCType(writer, result_ctype);
                    try writer.writeByte(')');
                    try dg.renderPointer(writer, elem.parent.*, location);
                    try writer.print(" + {})", .{try dg.fmtIntLiteral(index_val, .Other)});
                }
            },

            .offset_and_cast => |oac| {
                const ptr_ctype = try dg.ctypeFromType(oac.new_ptr_ty, .complete);
                try writer.writeByte('(');
                try dg.renderCType(writer, ptr_ctype);
                try writer.writeByte(')');
                if (oac.byte_offset == 0) {
                    try dg.renderPointer(writer, oac.parent.*, location);
                } else {
                    const offset_val = try pt.intValue(.usize, oac.byte_offset);
                    try writer.writeAll("((char *)");
                    try dg.renderPointer(writer, oac.parent.*, location);
                    try writer.print(" + {})", .{try dg.fmtIntLiteral(offset_val, .Other)});
                }
            },
        }
    }

    fn renderErrorName(dg: *DeclGen, writer: anytype, err_name: InternPool.NullTerminatedString) !void {
        const ip = &dg.pt.zcu.intern_pool;
        try writer.print("zig_error_{}", .{fmtIdent(err_name.toSlice(ip))});
    }

    fn renderValue(
        dg: *DeclGen,
        writer: anytype,
        val: Value,
        location: ValueRenderLocation,
    ) error{ OutOfMemory, AnalysisFail }!void {
        const pt = dg.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const target = &dg.mod.resolved_target.result;
        const ctype_pool = &dg.ctype_pool;

        const initializer_type: ValueRenderLocation = switch (location) {
            .StaticInitializer => .StaticInitializer,
            else => .Initializer,
        };

        const ty = val.typeOf(zcu);
        if (val.isUndefDeep(zcu)) return dg.renderUndefValue(writer, ty, location);
        const ctype = try dg.ctypeFromType(ty, location.toCTypeKind());
        switch (ip.indexToKey(val.toIntern())) {
            // types, not values
            .int_type,
            .ptr_type,
            .array_type,
            .vector_type,
            .opt_type,
            .anyframe_type,
            .error_union_type,
            .simple_type,
            .struct_type,
            .tuple_type,
            .union_type,
            .opaque_type,
            .enum_type,
            .func_type,
            .error_set_type,
            .inferred_error_set_type,
            // memoization, not values
            .memoized_call,
            => unreachable,

            .undef => unreachable, // handled above
            .simple_value => |simple_value| switch (simple_value) {
                // non-runtime values
                .undefined => unreachable,
                .void => unreachable,
                .null => unreachable,
                .empty_tuple => unreachable,
                .@"unreachable" => unreachable,

                .false => try writer.writeAll("false"),
                .true => try writer.writeAll("true"),
            },
            .variable,
            .@"extern",
            .func,
            .enum_literal,
            .empty_enum_value,
            => unreachable, // non-runtime values
            .int => |int| switch (int.storage) {
                .u64, .i64, .big_int => try writer.print("{}", .{try dg.fmtIntLiteral(val, location)}),
                .lazy_align, .lazy_size => {
                    try writer.writeAll("((");
                    try dg.renderCType(writer, ctype);
                    try writer.print("){x})", .{try dg.fmtIntLiteral(
                        try pt.intValue(.usize, val.toUnsignedInt(zcu)),
                        .Other,
                    )});
                },
            },
            .err => |err| try dg.renderErrorName(writer, err.name),
            .error_union => |error_union| switch (ctype.info(ctype_pool)) {
                .basic => switch (error_union.val) {
                    .err_name => |err_name| try dg.renderErrorName(writer, err_name),
                    .payload => try writer.writeAll("0"),
                },
                .pointer, .aligned, .array, .vector, .fwd_decl, .function => unreachable,
                .aggregate => |aggregate| {
                    if (!location.isInitializer()) {
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ctype);
                        try writer.writeByte(')');
                    }
                    try writer.writeByte('{');
                    for (0..aggregate.fields.len) |field_index| {
                        if (field_index > 0) try writer.writeByte(',');
                        switch (aggregate.fields.at(field_index, ctype_pool).name.index) {
                            .@"error" => switch (error_union.val) {
                                .err_name => |err_name| try dg.renderErrorName(writer, err_name),
                                .payload => try writer.writeByte('0'),
                            },
                            .payload => switch (error_union.val) {
                                .err_name => try dg.renderUndefValue(
                                    writer,
                                    ty.errorUnionPayload(zcu),
                                    initializer_type,
                                ),
                                .payload => |payload| try dg.renderValue(
                                    writer,
                                    Value.fromInterned(payload),
                                    initializer_type,
                                ),
                            },
                            else => unreachable,
                        }
                    }
                    try writer.writeByte('}');
                },
            },
            .enum_tag => |enum_tag| try dg.renderValue(writer, Value.fromInterned(enum_tag.int), location),
            .float => {
                const bits = ty.floatBits(target.*);
                const f128_val = val.toFloat(f128, zcu);

                // All unsigned ints matching float types are pre-allocated.
                const repr_ty = pt.intType(.unsigned, bits) catch unreachable;

                assert(bits <= 128);
                var repr_val_limbs: [BigInt.calcTwosCompLimbCount(128)]BigIntLimb = undefined;
                var repr_val_big = BigInt.Mutable{
                    .limbs = &repr_val_limbs,
                    .len = undefined,
                    .positive = undefined,
                };

                switch (bits) {
                    16 => repr_val_big.set(@as(u16, @bitCast(val.toFloat(f16, zcu)))),
                    32 => repr_val_big.set(@as(u32, @bitCast(val.toFloat(f32, zcu)))),
                    64 => repr_val_big.set(@as(u64, @bitCast(val.toFloat(f64, zcu)))),
                    80 => repr_val_big.set(@as(u80, @bitCast(val.toFloat(f80, zcu)))),
                    128 => repr_val_big.set(@as(u128, @bitCast(f128_val))),
                    else => unreachable,
                }

                var empty = true;
                if (std.math.isFinite(f128_val)) {
                    try writer.writeAll("zig_make_");
                    try dg.renderTypeForBuiltinFnName(writer, ty);
                    try writer.writeByte('(');
                    switch (bits) {
                        16 => try writer.print("{x}", .{val.toFloat(f16, zcu)}),
                        32 => try writer.print("{x}", .{val.toFloat(f32, zcu)}),
                        64 => try writer.print("{x}", .{val.toFloat(f64, zcu)}),
                        80 => try writer.print("{x}", .{val.toFloat(f80, zcu)}),
                        128 => try writer.print("{x}", .{f128_val}),
                        else => unreachable,
                    }
                    try writer.writeAll(", ");
                    empty = false;
                } else {
                    // isSignalNan is equivalent to isNan currently, and MSVC doesn't have nans, so prefer nan
                    const operation = if (std.math.isNan(f128_val))
                        "nan"
                    else if (std.math.isSignalNan(f128_val))
                        "nans"
                    else if (std.math.isInf(f128_val))
                        "inf"
                    else
                        unreachable;

                    if (location == .StaticInitializer) {
                        if (!std.math.isNan(f128_val) and std.math.isSignalNan(f128_val))
                            return dg.fail("TODO: C backend: implement nans rendering in static initializers", .{});

                        // MSVC doesn't have a way to define a custom or signaling NaN value in a constant expression

                        // TODO: Re-enable this check, otherwise we're writing qnan bit patterns on msvc incorrectly
                        // if (std.math.isNan(f128_val) and f128_val != std.math.nan(f128))
                        //     return dg.fail("Only quiet nans are supported in global variable initializers", .{});
                    }

                    try writer.writeAll("zig_");
                    try writer.writeAll(if (location == .StaticInitializer) "init" else "make");
                    try writer.writeAll("_special_");
                    try dg.renderTypeForBuiltinFnName(writer, ty);
                    try writer.writeByte('(');
                    if (std.math.signbit(f128_val)) try writer.writeByte('-');
                    try writer.writeAll(", ");
                    try writer.writeAll(operation);
                    try writer.writeAll(", ");
                    if (std.math.isNan(f128_val)) switch (bits) {
                        // We only actually need to pass the significand, but it will get
                        // properly masked anyway, so just pass the whole value.
                        16 => try writer.print("\"0x{x}\"", .{@as(u16, @bitCast(val.toFloat(f16, zcu)))}),
                        32 => try writer.print("\"0x{x}\"", .{@as(u32, @bitCast(val.toFloat(f32, zcu)))}),
                        64 => try writer.print("\"0x{x}\"", .{@as(u64, @bitCast(val.toFloat(f64, zcu)))}),
                        80 => try writer.print("\"0x{x}\"", .{@as(u80, @bitCast(val.toFloat(f80, zcu)))}),
                        128 => try writer.print("\"0x{x}\"", .{@as(u128, @bitCast(f128_val))}),
                        else => unreachable,
                    };
                    try writer.writeAll(", ");
                    empty = false;
                }
                try writer.print("{x}", .{try dg.fmtIntLiteral(
                    try pt.intValue_big(repr_ty, repr_val_big.toConst()),
                    location,
                )});
                if (!empty) try writer.writeByte(')');
            },
            .slice => |slice| {
                const aggregate = ctype.info(ctype_pool).aggregate;
                if (!location.isInitializer()) {
                    try writer.writeByte('(');
                    try dg.renderCType(writer, ctype);
                    try writer.writeByte(')');
                }
                try writer.writeByte('{');
                for (0..aggregate.fields.len) |field_index| {
                    if (field_index > 0) try writer.writeByte(',');
                    try dg.renderValue(writer, Value.fromInterned(
                        switch (aggregate.fields.at(field_index, ctype_pool).name.index) {
                            .ptr => slice.ptr,
                            .len => slice.len,
                            else => unreachable,
                        },
                    ), initializer_type);
                }
                try writer.writeByte('}');
            },
            .ptr => {
                var arena = std.heap.ArenaAllocator.init(zcu.gpa);
                defer arena.deinit();
                const derivation = try val.pointerDerivation(arena.allocator(), pt);
                try dg.renderPointer(writer, derivation, location);
            },
            .opt => |opt| switch (ctype.info(ctype_pool)) {
                .basic => if (ctype.isBool()) try writer.writeAll(switch (opt.val) {
                    .none => "true",
                    else => "false",
                }) else switch (opt.val) {
                    .none => try writer.writeAll("0"),
                    else => |payload| switch (ip.indexToKey(payload)) {
                        .undef => |err_ty| try dg.renderUndefValue(
                            writer,
                            .fromInterned(err_ty),
                            location,
                        ),
                        .err => |err| try dg.renderErrorName(writer, err.name),
                        else => unreachable,
                    },
                },
                .pointer => switch (opt.val) {
                    .none => try writer.writeAll("NULL"),
                    else => |payload| try dg.renderValue(writer, Value.fromInterned(payload), location),
                },
                .aligned, .array, .vector, .fwd_decl, .function => unreachable,
                .aggregate => |aggregate| {
                    switch (opt.val) {
                        .none => {},
                        else => |payload| switch (aggregate.fields.at(0, ctype_pool).name.index) {
                            .is_null, .payload => {},
                            .ptr, .len => return dg.renderValue(
                                writer,
                                Value.fromInterned(payload),
                                location,
                            ),
                            else => unreachable,
                        },
                    }
                    if (!location.isInitializer()) {
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ctype);
                        try writer.writeByte(')');
                    }
                    try writer.writeByte('{');
                    for (0..aggregate.fields.len) |field_index| {
                        if (field_index > 0) try writer.writeByte(',');
                        switch (aggregate.fields.at(field_index, ctype_pool).name.index) {
                            .is_null => try writer.writeAll(switch (opt.val) {
                                .none => "true",
                                else => "false",
                            }),
                            .payload => switch (opt.val) {
                                .none => try dg.renderUndefValue(
                                    writer,
                                    ty.optionalChild(zcu),
                                    initializer_type,
                                ),
                                else => |payload| try dg.renderValue(
                                    writer,
                                    Value.fromInterned(payload),
                                    initializer_type,
                                ),
                            },
                            .ptr => try writer.writeAll("NULL"),
                            .len => try dg.renderUndefValue(writer, .usize, initializer_type),
                            else => unreachable,
                        }
                    }
                    try writer.writeByte('}');
                },
            },
            .aggregate => switch (ip.indexToKey(ty.toIntern())) {
                .array_type, .vector_type => {
                    if (location == .FunctionArgument) {
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ctype);
                        try writer.writeByte(')');
                    }
                    const ai = ty.arrayInfo(zcu);
                    if (ai.elem_type.eql(.u8, zcu)) {
                        var literal = stringLiteral(writer, ty.arrayLenIncludingSentinel(zcu));
                        try literal.start();
                        var index: usize = 0;
                        while (index < ai.len) : (index += 1) {
                            const elem_val = try val.elemValue(pt, index);
                            const elem_val_u8: u8 = if (elem_val.isUndef(zcu))
                                undefPattern(u8)
                            else
                                @intCast(elem_val.toUnsignedInt(zcu));
                            try literal.writeChar(elem_val_u8);
                        }
                        if (ai.sentinel) |s| {
                            const s_u8: u8 = @intCast(s.toUnsignedInt(zcu));
                            if (s_u8 != 0) try literal.writeChar(s_u8);
                        }
                        try literal.end();
                    } else {
                        try writer.writeByte('{');
                        var index: usize = 0;
                        while (index < ai.len) : (index += 1) {
                            if (index != 0) try writer.writeByte(',');
                            const elem_val = try val.elemValue(pt, index);
                            try dg.renderValue(writer, elem_val, initializer_type);
                        }
                        if (ai.sentinel) |s| {
                            if (index != 0) try writer.writeByte(',');
                            try dg.renderValue(writer, s, initializer_type);
                        }
                        try writer.writeByte('}');
                    }
                },
                .tuple_type => |tuple| {
                    if (!location.isInitializer()) {
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ctype);
                        try writer.writeByte(')');
                    }

                    try writer.writeByte('{');
                    var empty = true;
                    for (0..tuple.types.len) |field_index| {
                        const comptime_val = tuple.values.get(ip)[field_index];
                        if (comptime_val != .none) continue;
                        const field_ty: Type = .fromInterned(tuple.types.get(ip)[field_index]);
                        if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;

                        if (!empty) try writer.writeByte(',');

                        const field_val = Value.fromInterned(
                            switch (ip.indexToKey(val.toIntern()).aggregate.storage) {
                                .bytes => |bytes| try pt.intern(.{ .int = .{
                                    .ty = field_ty.toIntern(),
                                    .storage = .{ .u64 = bytes.at(field_index, ip) },
                                } }),
                                .elems => |elems| elems[field_index],
                                .repeated_elem => |elem| elem,
                            },
                        );
                        try dg.renderValue(writer, field_val, initializer_type);

                        empty = false;
                    }
                    try writer.writeByte('}');
                },
                .struct_type => {
                    const loaded_struct = ip.loadStructType(ty.toIntern());
                    switch (loaded_struct.layout) {
                        .auto, .@"extern" => {
                            if (!location.isInitializer()) {
                                try writer.writeByte('(');
                                try dg.renderCType(writer, ctype);
                                try writer.writeByte(')');
                            }

                            try writer.writeByte('{');
                            var field_it = loaded_struct.iterateRuntimeOrder(ip);
                            var need_comma = false;
                            while (field_it.next()) |field_index| {
                                const field_ty: Type = .fromInterned(loaded_struct.field_types.get(ip)[field_index]);
                                if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;

                                if (need_comma) try writer.writeByte(',');
                                need_comma = true;
                                const field_val = switch (ip.indexToKey(val.toIntern()).aggregate.storage) {
                                    .bytes => |bytes| try pt.intern(.{ .int = .{
                                        .ty = field_ty.toIntern(),
                                        .storage = .{ .u64 = bytes.at(field_index, ip) },
                                    } }),
                                    .elems => |elems| elems[field_index],
                                    .repeated_elem => |elem| elem,
                                };
                                try dg.renderValue(writer, Value.fromInterned(field_val), initializer_type);
                            }
                            try writer.writeByte('}');
                        },
                        .@"packed" => {
                            const int_info = ty.intInfo(zcu);

                            const bits = Type.smallestUnsignedBits(int_info.bits - 1);
                            const bit_offset_ty = try pt.intType(.unsigned, bits);

                            var bit_offset: u64 = 0;
                            var eff_num_fields: usize = 0;

                            for (0..loaded_struct.field_types.len) |field_index| {
                                const field_ty: Type = .fromInterned(loaded_struct.field_types.get(ip)[field_index]);
                                if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;
                                eff_num_fields += 1;
                            }

                            if (eff_num_fields == 0) {
                                try writer.writeByte('(');
                                try dg.renderUndefValue(writer, ty, location);
                                try writer.writeByte(')');
                            } else if (ty.bitSize(zcu) > 64) {
                                // zig_or_u128(zig_or_u128(zig_shl_u128(a, a_off), zig_shl_u128(b, b_off)), zig_shl_u128(c, c_off))
                                var num_or = eff_num_fields - 1;
                                while (num_or > 0) : (num_or -= 1) {
                                    try writer.writeAll("zig_or_");
                                    try dg.renderTypeForBuiltinFnName(writer, ty);
                                    try writer.writeByte('(');
                                }

                                var eff_index: usize = 0;
                                var needs_closing_paren = false;
                                for (0..loaded_struct.field_types.len) |field_index| {
                                    const field_ty: Type = .fromInterned(loaded_struct.field_types.get(ip)[field_index]);
                                    if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;

                                    const field_val = switch (ip.indexToKey(val.toIntern()).aggregate.storage) {
                                        .bytes => |bytes| try pt.intern(.{ .int = .{
                                            .ty = field_ty.toIntern(),
                                            .storage = .{ .u64 = bytes.at(field_index, ip) },
                                        } }),
                                        .elems => |elems| elems[field_index],
                                        .repeated_elem => |elem| elem,
                                    };
                                    const cast_context = IntCastContext{ .value = .{ .value = Value.fromInterned(field_val) } };
                                    if (bit_offset != 0) {
                                        try writer.writeAll("zig_shl_");
                                        try dg.renderTypeForBuiltinFnName(writer, ty);
                                        try writer.writeByte('(');
                                        try dg.renderIntCast(writer, ty, cast_context, field_ty, .FunctionArgument);
                                        try writer.writeAll(", ");
                                        try dg.renderValue(writer, try pt.intValue(bit_offset_ty, bit_offset), .FunctionArgument);
                                        try writer.writeByte(')');
                                    } else {
                                        try dg.renderIntCast(writer, ty, cast_context, field_ty, .FunctionArgument);
                                    }

                                    if (needs_closing_paren) try writer.writeByte(')');
                                    if (eff_index != eff_num_fields - 1) try writer.writeAll(", ");

                                    bit_offset += field_ty.bitSize(zcu);
                                    needs_closing_paren = true;
                                    eff_index += 1;
                                }
                            } else {
                                try writer.writeByte('(');
                                // a << a_off | b << b_off | c << c_off
                                var empty = true;
                                for (0..loaded_struct.field_types.len) |field_index| {
                                    const field_ty: Type = .fromInterned(loaded_struct.field_types.get(ip)[field_index]);
                                    if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;

                                    if (!empty) try writer.writeAll(" | ");
                                    try writer.writeByte('(');
                                    try dg.renderCType(writer, ctype);
                                    try writer.writeByte(')');

                                    const field_val = switch (ip.indexToKey(val.toIntern()).aggregate.storage) {
                                        .bytes => |bytes| try pt.intern(.{ .int = .{
                                            .ty = field_ty.toIntern(),
                                            .storage = .{ .u64 = bytes.at(field_index, ip) },
                                        } }),
                                        .elems => |elems| elems[field_index],
                                        .repeated_elem => |elem| elem,
                                    };

                                    const field_int_info: std.builtin.Type.Int = if (field_ty.isAbiInt(zcu))
                                        field_ty.intInfo(zcu)
                                    else
                                        .{ .signedness = .unsigned, .bits = undefined };
                                    switch (field_int_info.signedness) {
                                        .signed => {
                                            try writer.writeByte('(');
                                            try dg.renderValue(writer, Value.fromInterned(field_val), .Other);
                                            try writer.writeAll(" & ");
                                            const field_uint_ty = try pt.intType(.unsigned, field_int_info.bits);
                                            try dg.renderValue(writer, try field_uint_ty.maxIntScalar(pt, field_uint_ty), .Other);
                                            try writer.writeByte(')');
                                        },
                                        .unsigned => try dg.renderValue(writer, Value.fromInterned(field_val), .Other),
                                    }
                                    if (bit_offset != 0) {
                                        try writer.writeAll(" << ");
                                        try dg.renderValue(writer, try pt.intValue(bit_offset_ty, bit_offset), .FunctionArgument);
                                    }

                                    bit_offset += field_ty.bitSize(zcu);
                                    empty = false;
                                }
                                try writer.writeByte(')');
                            }
                        },
                    }
                },
                else => unreachable,
            },
            .un => |un| {
                const loaded_union = ip.loadUnionType(ty.toIntern());
                if (un.tag == .none) {
                    const backing_ty = try ty.unionBackingType(pt);
                    switch (loaded_union.flagsUnordered(ip).layout) {
                        .@"packed" => {
                            if (!location.isInitializer()) {
                                try writer.writeByte('(');
                                try dg.renderType(writer, backing_ty);
                                try writer.writeByte(')');
                            }
                            try dg.renderValue(writer, Value.fromInterned(un.val), location);
                        },
                        .@"extern" => {
                            if (location == .StaticInitializer) {
                                return dg.fail("TODO: C backend: implement extern union backing type rendering in static initializers", .{});
                            }

                            const ptr_ty = try pt.singleConstPtrType(ty);
                            try writer.writeAll("*((");
                            try dg.renderType(writer, ptr_ty);
                            try writer.writeAll(")(");
                            try dg.renderType(writer, backing_ty);
                            try writer.writeAll("){");
                            try dg.renderValue(writer, Value.fromInterned(un.val), location);
                            try writer.writeAll("})");
                        },
                        else => unreachable,
                    }
                } else {
                    if (!location.isInitializer()) {
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ctype);
                        try writer.writeByte(')');
                    }

                    const field_index = zcu.unionTagFieldIndex(loaded_union, Value.fromInterned(un.tag)).?;
                    const field_ty: Type = .fromInterned(loaded_union.field_types.get(ip)[field_index]);
                    const field_name = loaded_union.loadTagType(ip).names.get(ip)[field_index];
                    if (loaded_union.flagsUnordered(ip).layout == .@"packed") {
                        if (field_ty.hasRuntimeBits(zcu)) {
                            if (field_ty.isPtrAtRuntime(zcu)) {
                                try writer.writeByte('(');
                                try dg.renderCType(writer, ctype);
                                try writer.writeByte(')');
                            } else if (field_ty.zigTypeTag(zcu) == .float) {
                                try writer.writeByte('(');
                                try dg.renderCType(writer, ctype);
                                try writer.writeByte(')');
                            }
                            try dg.renderValue(writer, Value.fromInterned(un.val), location);
                        } else try writer.writeAll("0");
                        return;
                    }

                    const has_tag = loaded_union.hasTag(ip);
                    if (has_tag) try writer.writeByte('{');
                    const aggregate = ctype.info(ctype_pool).aggregate;
                    for (0..if (has_tag) aggregate.fields.len else 1) |outer_field_index| {
                        if (outer_field_index > 0) try writer.writeByte(',');
                        switch (if (has_tag)
                            aggregate.fields.at(outer_field_index, ctype_pool).name.index
                        else
                            .payload) {
                            .tag => try dg.renderValue(
                                writer,
                                Value.fromInterned(un.tag),
                                initializer_type,
                            ),
                            .payload => {
                                try writer.writeByte('{');
                                if (field_ty.hasRuntimeBits(zcu)) {
                                    try writer.print(" .{ } = ", .{fmtIdent(field_name.toSlice(ip))});
                                    try dg.renderValue(
                                        writer,
                                        Value.fromInterned(un.val),
                                        initializer_type,
                                    );
                                    try writer.writeByte(' ');
                                } else for (0..loaded_union.field_types.len) |inner_field_index| {
                                    const inner_field_ty: Type = .fromInterned(
                                        loaded_union.field_types.get(ip)[inner_field_index],
                                    );
                                    if (!inner_field_ty.hasRuntimeBits(zcu)) continue;
                                    try dg.renderUndefValue(writer, inner_field_ty, initializer_type);
                                    break;
                                }
                                try writer.writeByte('}');
                            },
                            else => unreachable,
                        }
                    }
                    if (has_tag) try writer.writeByte('}');
                }
            },
        }
    }

    fn renderUndefValue(
        dg: *DeclGen,
        writer: anytype,
        ty: Type,
        location: ValueRenderLocation,
    ) error{ OutOfMemory, AnalysisFail }!void {
        const pt = dg.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const target = &dg.mod.resolved_target.result;
        const ctype_pool = &dg.ctype_pool;

        const initializer_type: ValueRenderLocation = switch (location) {
            .StaticInitializer => .StaticInitializer,
            else => .Initializer,
        };

        const safety_on = switch (zcu.optimizeMode()) {
            .Debug, .ReleaseSafe => true,
            .ReleaseFast, .ReleaseSmall => false,
        };

        const ctype = try dg.ctypeFromType(ty, location.toCTypeKind());
        switch (ty.toIntern()) {
            .c_longdouble_type,
            .f16_type,
            .f32_type,
            .f64_type,
            .f80_type,
            .f128_type,
            => {
                const bits = ty.floatBits(target.*);
                // All unsigned ints matching float types are pre-allocated.
                const repr_ty = dg.pt.intType(.unsigned, bits) catch unreachable;

                try writer.writeAll("zig_make_");
                try dg.renderTypeForBuiltinFnName(writer, ty);
                try writer.writeByte('(');
                switch (bits) {
                    16 => try writer.print("{x}", .{@as(f16, @bitCast(undefPattern(i16)))}),
                    32 => try writer.print("{x}", .{@as(f32, @bitCast(undefPattern(i32)))}),
                    64 => try writer.print("{x}", .{@as(f64, @bitCast(undefPattern(i64)))}),
                    80 => try writer.print("{x}", .{@as(f80, @bitCast(undefPattern(i80)))}),
                    128 => try writer.print("{x}", .{@as(f128, @bitCast(undefPattern(i128)))}),
                    else => unreachable,
                }
                try writer.writeAll(", ");
                try dg.renderUndefValue(writer, repr_ty, .FunctionArgument);
                return writer.writeByte(')');
            },
            .bool_type => try writer.writeAll(if (safety_on) "0xaa" else "false"),
            else => switch (ip.indexToKey(ty.toIntern())) {
                .simple_type,
                .int_type,
                .enum_type,
                .error_set_type,
                .inferred_error_set_type,
                => return writer.print("{x}", .{
                    try dg.fmtIntLiteral(try pt.undefValue(ty), location),
                }),
                .ptr_type => |ptr_type| switch (ptr_type.flags.size) {
                    .one, .many, .c => {
                        try writer.writeAll("((");
                        try dg.renderCType(writer, ctype);
                        return writer.print("){x})", .{
                            try dg.fmtIntLiteral(try pt.undefValue(.usize), .Other),
                        });
                    },
                    .slice => {
                        if (!location.isInitializer()) {
                            try writer.writeByte('(');
                            try dg.renderCType(writer, ctype);
                            try writer.writeByte(')');
                        }

                        try writer.writeAll("{(");
                        const ptr_ty = ty.slicePtrFieldType(zcu);
                        try dg.renderType(writer, ptr_ty);
                        return writer.print("){x}, {0x}}}", .{
                            try dg.fmtIntLiteral(try dg.pt.undefValue(.usize), .Other),
                        });
                    },
                },
                .opt_type => |child_type| switch (ctype.info(ctype_pool)) {
                    .basic, .pointer => try dg.renderUndefValue(
                        writer,
                        .fromInterned(if (ctype.isBool()) .bool_type else child_type),
                        location,
                    ),
                    .aligned, .array, .vector, .fwd_decl, .function => unreachable,
                    .aggregate => |aggregate| {
                        switch (aggregate.fields.at(0, ctype_pool).name.index) {
                            .is_null, .payload => {},
                            .ptr, .len => return dg.renderUndefValue(
                                writer,
                                .fromInterned(child_type),
                                location,
                            ),
                            else => unreachable,
                        }
                        if (!location.isInitializer()) {
                            try writer.writeByte('(');
                            try dg.renderCType(writer, ctype);
                            try writer.writeByte(')');
                        }
                        try writer.writeByte('{');
                        for (0..aggregate.fields.len) |field_index| {
                            if (field_index > 0) try writer.writeByte(',');
                            try dg.renderUndefValue(writer, .fromInterned(
                                switch (aggregate.fields.at(field_index, ctype_pool).name.index) {
                                    .is_null => .bool_type,
                                    .payload => child_type,
                                    else => unreachable,
                                },
                            ), initializer_type);
                        }
                        try writer.writeByte('}');
                    },
                },
                .struct_type => {
                    const loaded_struct = ip.loadStructType(ty.toIntern());
                    switch (loaded_struct.layout) {
                        .auto, .@"extern" => {
                            if (!location.isInitializer()) {
                                try writer.writeByte('(');
                                try dg.renderCType(writer, ctype);
                                try writer.writeByte(')');
                            }

                            try writer.writeByte('{');
                            var field_it = loaded_struct.iterateRuntimeOrder(ip);
                            var need_comma = false;
                            while (field_it.next()) |field_index| {
                                const field_ty: Type = .fromInterned(loaded_struct.field_types.get(ip)[field_index]);
                                if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;

                                if (need_comma) try writer.writeByte(',');
                                need_comma = true;
                                try dg.renderUndefValue(writer, field_ty, initializer_type);
                            }
                            return writer.writeByte('}');
                        },
                        .@"packed" => return writer.print("{x}", .{
                            try dg.fmtIntLiteral(try pt.undefValue(ty), .Other),
                        }),
                    }
                },
                .tuple_type => |tuple_info| {
                    if (!location.isInitializer()) {
                        try writer.writeByte('(');
                        try dg.renderCType(writer, ctype);
                        try writer.writeByte(')');
                    }

                    try writer.writeByte('{');
                    var need_comma = false;
                    for (0..tuple_info.types.len) |field_index| {
                        if (tuple_info.values.get(ip)[field_index] != .none) continue;
                        const field_ty: Type = .fromInterned(tuple_info.types.get(ip)[field_index]);
                        if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;

                        if (need_comma) try writer.writeByte(',');
                        need_comma = true;
                        try dg.renderUndefValue(writer, field_ty, initializer_type);
                    }
                    return writer.writeByte('}');
                },
                .union_type => {
                    const loaded_union = ip.loadUnionType(ty.toIntern());
                    switch (loaded_union.flagsUnordered(ip).layout) {
                        .auto, .@"extern" => {
                            if (!location.isInitializer()) {
                                try writer.writeByte('(');
                                try dg.renderCType(writer, ctype);
                                try writer.writeByte(')');
                            }

                            const has_tag = loaded_union.hasTag(ip);
                            if (has_tag) try writer.writeByte('{');
                            const aggregate = ctype.info(ctype_pool).aggregate;
                            for (0..if (has_tag) aggregate.fields.len else 1) |outer_field_index| {
                                if (outer_field_index > 0) try writer.writeByte(',');
                                switch (if (has_tag)
                                    aggregate.fields.at(outer_field_index, ctype_pool).name.index
                                else
                                    .payload) {
                                    .tag => try dg.renderUndefValue(
                                        writer,
                                        .fromInterned(loaded_union.enum_tag_ty),
                                        initializer_type,
                                    ),
                                    .payload => {
                                        try writer.writeByte('{');
                                        for (0..loaded_union.field_types.len) |inner_field_index| {
                                            const inner_field_ty: Type = .fromInterned(
                                                loaded_union.field_types.get(ip)[inner_field_index],
                                            );
                                            if (!inner_field_ty.hasRuntimeBits(pt.zcu)) continue;
                                            try dg.renderUndefValue(
                                                writer,
                                                inner_field_ty,
                                                initializer_type,
                                            );
                                            break;
                                        }
                                        try writer.writeByte('}');
                                    },
                                    else => unreachable,
                                }
                            }
                            if (has_tag) try writer.writeByte('}');
                        },
                        .@"packed" => return writer.print("{x}", .{
                            try dg.fmtIntLiteral(try pt.undefValue(ty), .Other),
                        }),
                    }
                },
                .error_union_type => |error_union_type| switch (ctype.info(ctype_pool)) {
                    .basic => try dg.renderUndefValue(
                        writer,
                        .fromInterned(error_union_type.error_set_type),
                        location,
                    ),
                    .pointer, .aligned, .array, .vector, .fwd_decl, .function => unreachable,
                    .aggregate => |aggregate| {
                        if (!location.isInitializer()) {
                            try writer.writeByte('(');
                            try dg.renderCType(writer, ctype);
                            try writer.writeByte(')');
                        }
                        try writer.writeByte('{');
                        for (0..aggregate.fields.len) |field_index| {
                            if (field_index > 0) try writer.writeByte(',');
                            try dg.renderUndefValue(
                                writer,
                                .fromInterned(
                                    switch (aggregate.fields.at(field_index, ctype_pool).name.index) {
                                        .@"error" => error_union_type.error_set_type,
                                        .payload => error_union_type.payload_type,
                                        else => unreachable,
                                    },
                                ),
                                initializer_type,
                            );
                        }
                        try writer.writeByte('}');
                    },
                },
                .array_type, .vector_type => {
                    const ai = ty.arrayInfo(zcu);
                    if (ai.elem_type.eql(.u8, zcu)) {
                        const c_len = ty.arrayLenIncludingSentinel(zcu);
                        var literal = stringLiteral(writer, c_len);
                        try literal.start();
                        var index: u64 = 0;
                        while (index < c_len) : (index += 1)
                            try literal.writeChar(0xaa);
                        return literal.end();
                    } else {
                        if (!location.isInitializer()) {
                            try writer.writeByte('(');
                            try dg.renderCType(writer, ctype);
                            try writer.writeByte(')');
                        }

                        try writer.writeByte('{');
                        const c_len = ty.arrayLenIncludingSentinel(zcu);
                        var index: u64 = 0;
                        while (index < c_len) : (index += 1) {
                            if (index > 0) try writer.writeAll(", ");
                            try dg.renderUndefValue(writer, ty.childType(zcu), initializer_type);
                        }
                        return writer.writeByte('}');
                    }
                },
                .anyframe_type,
                .opaque_type,
                .func_type,
                => unreachable,

                .undef,
                .simple_value,
                .variable,
                .@"extern",
                .func,
                .int,
                .err,
                .error_union,
                .enum_literal,
                .enum_tag,
                .empty_enum_value,
                .float,
                .ptr,
                .slice,
                .opt,
                .aggregate,
                .un,
                .memoized_call,
                => unreachable, // values, not types
            },
        }
    }

    fn renderFunctionSignature(
        dg: *DeclGen,
        w: anytype,
        fn_val: Value,
        fn_align: InternPool.Alignment,
        kind: CType.Kind,
        name: union(enum) {
            nav: InternPool.Nav.Index,
            fmt_ctype_pool_string: std.fmt.Formatter(formatCTypePoolString),
            @"export": struct {
                main_name: InternPool.NullTerminatedString,
                extern_name: InternPool.NullTerminatedString,
            },
        },
    ) !void {
        const zcu = dg.pt.zcu;
        const ip = &zcu.intern_pool;

        const fn_ty = fn_val.typeOf(zcu);
        const fn_ctype = try dg.ctypeFromType(fn_ty, kind);

        const fn_info = zcu.typeToFunc(fn_ty).?;
        if (fn_info.cc == .naked) {
            switch (kind) {
                .forward => try w.writeAll("zig_naked_decl "),
                .complete => try w.writeAll("zig_naked "),
                else => unreachable,
            }
        }

        if (fn_val.getFunction(zcu)) |func| {
            const func_analysis = func.analysisUnordered(ip);

            if (func_analysis.branch_hint == .cold)
                try w.writeAll("zig_cold ");

            if (kind == .complete and func_analysis.disable_intrinsics or dg.mod.no_builtin)
                try w.writeAll("zig_no_builtin ");
        }

        if (fn_info.return_type == .noreturn_type) try w.writeAll("zig_noreturn ");

        var trailing = try renderTypePrefix(dg.pass, &dg.ctype_pool, zcu, w, fn_ctype, .suffix, .{});

        if (toCallingConvention(fn_info.cc, zcu)) |call_conv| {
            try w.print("{}zig_callconv({s})", .{ trailing, call_conv });
            trailing = .maybe_space;
        }

        try w.print("{}", .{trailing});
        switch (name) {
            .nav => |nav| try dg.renderNavName(w, nav),
            .fmt_ctype_pool_string => |fmt| try w.print("{ }", .{fmt}),
            .@"export" => |@"export"| try w.print("{ }", .{fmtIdent(@"export".extern_name.toSlice(ip))}),
        }

        try renderTypeSuffix(
            dg.pass,
            &dg.ctype_pool,
            zcu,
            w,
            fn_ctype,
            .suffix,
            CQualifiers.init(.{ .@"const" = switch (kind) {
                .forward => false,
                .complete => true,
                else => unreachable,
            } }),
        );

        switch (kind) {
            .forward => {
                if (fn_align.toByteUnits()) |a| try w.print(" zig_align_fn({})", .{a});
                switch (name) {
                    .nav, .fmt_ctype_pool_string => {},
                    .@"export" => |@"export"| {
                        const extern_name = @"export".extern_name.toSlice(ip);
                        const is_mangled = isMangledIdent(extern_name, true);
                        const is_export = @"export".extern_name != @"export".main_name;
                        if (is_mangled and is_export) {
                            try w.print(" zig_mangled_export({ }, {s}, {s})", .{
                                fmtIdent(extern_name),
                                fmtStringLiteral(extern_name, null),
                                fmtStringLiteral(@"export".main_name.toSlice(ip), null),
                            });
                        } else if (is_mangled) {
                            try w.print(" zig_mangled({ }, {s})", .{
                                fmtIdent(extern_name), fmtStringLiteral(extern_name, null),
                            });
                        } else if (is_export) {
                            try w.print(" zig_export({s}, {s})", .{
                                fmtStringLiteral(@"export".main_name.toSlice(ip), null),
                                fmtStringLiteral(extern_name, null),
                            });
                        }
                    },
                }
            },
            .complete => {},
            else => unreachable,
        }
    }

    fn ctypeFromType(dg: *DeclGen, ty: Type, kind: CType.Kind) !CType {
        defer std.debug.assert(dg.scratch.items.len == 0);
        return dg.ctype_pool.fromType(dg.gpa, &dg.scratch, ty, dg.pt, dg.mod, kind);
    }

    fn byteSize(dg: *DeclGen, ctype: CType) u64 {
        return ctype.byteSize(&dg.ctype_pool, dg.mod);
    }

    /// Renders a type as a single identifier, generating intermediate typedefs
    /// if necessary.
    ///
    /// This is guaranteed to be valid in both typedefs and declarations/definitions.
    ///
    /// There are three type formats in total that we support rendering:
    ///   | Function            | Example 1 (*u8) | Example 2 ([10]*u8) |
    ///   |---------------------|-----------------|---------------------|
    ///   | `renderTypeAndName` | "uint8_t *name" | "uint8_t *name[10]" |
    ///   | `renderType`        | "uint8_t *"     | "uint8_t *[10]"     |
    ///
    fn renderType(dg: *DeclGen, w: anytype, t: Type) error{OutOfMemory}!void {
        try dg.renderCType(w, try dg.ctypeFromType(t, .complete));
    }

    fn renderCType(dg: *DeclGen, w: anytype, ctype: CType) error{OutOfMemory}!void {
        _ = try renderTypePrefix(dg.pass, &dg.ctype_pool, dg.pt.zcu, w, ctype, .suffix, .{});
        try renderTypeSuffix(dg.pass, &dg.ctype_pool, dg.pt.zcu, w, ctype, .suffix, .{});
    }

    const IntCastContext = union(enum) {
        c_value: struct {
            f: *Function,
            value: CValue,
            v: Vectorize,
        },
        value: struct {
            value: Value,
        },

        pub fn writeValue(self: *const IntCastContext, dg: *DeclGen, w: anytype, location: ValueRenderLocation) !void {
            switch (self.*) {
                .c_value => |v| {
                    try v.f.writeCValue(w, v.value, location);
                    try v.v.elem(v.f, w);
                },
                .value => |v| try dg.renderValue(w, v.value, location),
            }
        }
    };
    fn intCastIsNoop(dg: *DeclGen, dest_ty: Type, src_ty: Type) bool {
        const pt = dg.pt;
        const zcu = pt.zcu;
        const dest_bits = dest_ty.bitSize(zcu);
        const dest_int_info = dest_ty.intInfo(pt.zcu);

        const src_is_ptr = src_ty.isPtrAtRuntime(pt.zcu);
        const src_eff_ty: Type = if (src_is_ptr) switch (dest_int_info.signedness) {
            .unsigned => .usize,
            .signed => .isize,
        } else src_ty;

        const src_bits = src_eff_ty.bitSize(zcu);
        const src_int_info = if (src_eff_ty.isAbiInt(pt.zcu)) src_eff_ty.intInfo(pt.zcu) else null;
        if (dest_bits <= 64 and src_bits <= 64) {
            const needs_cast = src_int_info == null or
                (toCIntBits(dest_int_info.bits) != toCIntBits(src_int_info.?.bits) or
                    dest_int_info.signedness != src_int_info.?.signedness);
            return !needs_cast and !src_is_ptr;
        } else return false;
    }
    /// Renders a cast to an int type, from either an int or a pointer.
    ///
    /// Some platforms don't have 128 bit integers, so we need to use
    /// the zig_make_ and zig_lo_ macros in those cases.
    ///
    ///   | Dest type bits   | Src type         | Result
    ///   |------------------|------------------|---------------------------|
    ///   | < 64 bit integer | pointer          | (zig_<dest_ty>)(zig_<u|i>size)src
    ///   | < 64 bit integer | < 64 bit integer | (zig_<dest_ty>)src
    ///   | < 64 bit integer | > 64 bit integer | zig_lo(src)
    ///   | > 64 bit integer | pointer          | zig_make_<dest_ty>(0, (zig_<u|i>size)src)
    ///   | > 64 bit integer | < 64 bit integer | zig_make_<dest_ty>(0, src)
    ///   | > 64 bit integer | > 64 bit integer | zig_make_<dest_ty>(zig_hi_<src_ty>(src), zig_lo_<src_ty>(src))
    fn renderIntCast(
        dg: *DeclGen,
        w: anytype,
        dest_ty: Type,
        context: IntCastContext,
        src_ty: Type,
        location: ValueRenderLocation,
    ) !void {
        const pt = dg.pt;
        const zcu = pt.zcu;
        const dest_bits = dest_ty.bitSize(zcu);
        const dest_int_info = dest_ty.intInfo(zcu);

        const src_is_ptr = src_ty.isPtrAtRuntime(zcu);
        const src_eff_ty: Type = if (src_is_ptr) switch (dest_int_info.signedness) {
            .unsigned => .usize,
            .signed => .isize,
        } else src_ty;

        const src_bits = src_eff_ty.bitSize(zcu);
        const src_int_info = if (src_eff_ty.isAbiInt(zcu)) src_eff_ty.intInfo(zcu) else null;
        if (dest_bits <= 64 and src_bits <= 64) {
            const needs_cast = src_int_info == null or
                (toCIntBits(dest_int_info.bits) != toCIntBits(src_int_info.?.bits) or
                    dest_int_info.signedness != src_int_info.?.signedness);

            if (needs_cast) {
                try w.writeByte('(');
                try dg.renderType(w, dest_ty);
                try w.writeByte(')');
            }
            if (src_is_ptr) {
                try w.writeByte('(');
                try dg.renderType(w, src_eff_ty);
                try w.writeByte(')');
            }
            try context.writeValue(dg, w, location);
        } else if (dest_bits <= 64 and src_bits > 64) {
            assert(!src_is_ptr);
            if (dest_bits < 64) {
                try w.writeByte('(');
                try dg.renderType(w, dest_ty);
                try w.writeByte(')');
            }
            try w.writeAll("zig_lo_");
            try dg.renderTypeForBuiltinFnName(w, src_eff_ty);
            try w.writeByte('(');
            try context.writeValue(dg, w, .FunctionArgument);
            try w.writeByte(')');
        } else if (dest_bits > 64 and src_bits <= 64) {
            try w.writeAll("zig_make_");
            try dg.renderTypeForBuiltinFnName(w, dest_ty);
            try w.writeAll("(0, "); // TODO: Should the 0 go through fmtIntLiteral?
            if (src_is_ptr) {
                try w.writeByte('(');
                try dg.renderType(w, src_eff_ty);
                try w.writeByte(')');
            }
            try context.writeValue(dg, w, .FunctionArgument);
            try w.writeByte(')');
        } else {
            assert(!src_is_ptr);
            try w.writeAll("zig_make_");
            try dg.renderTypeForBuiltinFnName(w, dest_ty);
            try w.writeAll("(zig_hi_");
            try dg.renderTypeForBuiltinFnName(w, src_eff_ty);
            try w.writeByte('(');
            try context.writeValue(dg, w, .FunctionArgument);
            try w.writeAll("), zig_lo_");
            try dg.renderTypeForBuiltinFnName(w, src_eff_ty);
            try w.writeByte('(');
            try context.writeValue(dg, w, .FunctionArgument);
            try w.writeAll("))");
        }
    }

    /// Renders a type and name in field declaration/definition format.
    ///
    /// There are three type formats in total that we support rendering:
    ///   | Function            | Example 1 (*u8) | Example 2 ([10]*u8) |
    ///   |---------------------|-----------------|---------------------|
    ///   | `renderTypeAndName` | "uint8_t *name" | "uint8_t *name[10]" |
    ///   | `renderType`        | "uint8_t *"     | "uint8_t *[10]"     |
    ///
    fn renderTypeAndName(
        dg: *DeclGen,
        w: anytype,
        ty: Type,
        name: CValue,
        qualifiers: CQualifiers,
        alignment: Alignment,
        kind: CType.Kind,
    ) error{ OutOfMemory, AnalysisFail }!void {
        try dg.renderCTypeAndName(
            w,
            try dg.ctypeFromType(ty, kind),
            name,
            qualifiers,
            CType.AlignAs.fromAlignment(.{
                .@"align" = alignment,
                .abi = ty.abiAlignment(dg.pt.zcu),
            }),
        );
    }

    fn renderCTypeAndName(
        dg: *DeclGen,
        w: anytype,
        ctype: CType,
        name: CValue,
        qualifiers: CQualifiers,
        alignas: CType.AlignAs,
    ) error{ OutOfMemory, AnalysisFail }!void {
        const zcu = dg.pt.zcu;
        switch (alignas.abiOrder()) {
            .lt => try w.print("zig_under_align({}) ", .{alignas.toByteUnits()}),
            .eq => {},
            .gt => try w.print("zig_align({}) ", .{alignas.toByteUnits()}),
        }

        try w.print("{}", .{
            try renderTypePrefix(dg.pass, &dg.ctype_pool, zcu, w, ctype, .suffix, qualifiers),
        });
        try dg.writeName(w, name);
        try renderTypeSuffix(dg.pass, &dg.ctype_pool, zcu, w, ctype, .suffix, .{});
    }

    fn writeName(dg: *DeclGen, w: anytype, c_value: CValue) !void {
        switch (c_value) {
            .new_local, .local => |i| try w.print("t{d}", .{i}),
            .constant => |uav| try renderUavName(w, uav),
            .nav => |nav| try dg.renderNavName(w, nav),
            .identifier => |ident| try w.print("{ }", .{fmtIdent(ident)}),
            else => unreachable,
        }
    }

    fn writeCValue(dg: *DeclGen, w: anytype, c_value: CValue) !void {
        switch (c_value) {
            .none, .new_local, .local, .local_ref => unreachable,
            .constant => |uav| try renderUavName(w, uav),
            .arg, .arg_array => unreachable,
            .field => |i| try w.print("f{d}", .{i}),
            .nav => |nav| try dg.renderNavName(w, nav),
            .nav_ref => |nav| {
                try w.writeByte('&');
                try dg.renderNavName(w, nav);
            },
            .undef => |ty| try dg.renderUndefValue(w, ty, .Other),
            .identifier => |ident| try w.print("{ }", .{fmtIdent(ident)}),
            .payload_identifier => |ident| try w.print("{ }.{ }", .{
                fmtIdent("payload"),
                fmtIdent(ident),
            }),
            .ctype_pool_string => |string| try w.print("{ }", .{
                fmtCTypePoolString(string, &dg.ctype_pool),
            }),
        }
    }

    fn writeCValueDeref(dg: *DeclGen, w: anytype, c_value: CValue) !void {
        switch (c_value) {
            .none,
            .new_local,
            .local,
            .local_ref,
            .constant,
            .arg,
            .arg_array,
            .ctype_pool_string,
            => unreachable,
            .field => |i| try w.print("f{d}", .{i}),
            .nav => |nav| {
                try w.writeAll("(*");
                try dg.renderNavName(w, nav);
                try w.writeByte(')');
            },
            .nav_ref => |nav| try dg.renderNavName(w, nav),
            .undef => unreachable,
            .identifier => |ident| try w.print("(*{ })", .{fmtIdent(ident)}),
            .payload_identifier => |ident| try w.print("(*{ }.{ })", .{
                fmtIdent("payload"),
                fmtIdent(ident),
            }),
        }
    }

    fn writeCValueMember(
        dg: *DeclGen,
        writer: anytype,
        c_value: CValue,
        member: CValue,
    ) error{ OutOfMemory, AnalysisFail }!void {
        try dg.writeCValue(writer, c_value);
        try writer.writeByte('.');
        try dg.writeCValue(writer, member);
    }

    fn writeCValueDerefMember(dg: *DeclGen, writer: anytype, c_value: CValue, member: CValue) !void {
        switch (c_value) {
            .none,
            .new_local,
            .local,
            .local_ref,
            .constant,
            .field,
            .undef,
            .arg,
            .arg_array,
            .ctype_pool_string,
            => unreachable,
            .nav, .identifier, .payload_identifier => {
                try dg.writeCValue(writer, c_value);
                try writer.writeAll("->");
            },
            .nav_ref => {
                try dg.writeCValueDeref(writer, c_value);
                try writer.writeByte('.');
            },
        }
        try dg.writeCValue(writer, member);
    }

    fn renderFwdDecl(
        dg: *DeclGen,
        nav_index: InternPool.Nav.Index,
        flags: struct {
            is_extern: bool,
            is_const: bool,
            is_threadlocal: bool,
            is_weak_linkage: bool,
        },
    ) !void {
        const zcu = dg.pt.zcu;
        const ip = &zcu.intern_pool;
        const nav = ip.getNav(nav_index);
        const fwd = dg.fwdDeclWriter();
        try fwd.writeAll(if (flags.is_extern) "zig_extern " else "static ");
        if (flags.is_weak_linkage) try fwd.writeAll("zig_weak_linkage ");
        if (flags.is_threadlocal and !dg.mod.single_threaded) try fwd.writeAll("zig_threadlocal ");
        try dg.renderTypeAndName(
            fwd,
            .fromInterned(nav.typeOf(ip)),
            .{ .nav = nav_index },
            CQualifiers.init(.{ .@"const" = flags.is_const }),
            nav.getAlignment(),
            .complete,
        );
        try fwd.writeAll(";\n");
    }

    fn renderNavName(dg: *DeclGen, writer: anytype, nav_index: InternPool.Nav.Index) !void {
        const zcu = dg.pt.zcu;
        const ip = &zcu.intern_pool;
        const nav = ip.getNav(nav_index);
        if (nav.getExtern(ip)) |@"extern"| {
            try writer.print("{ }", .{
                fmtIdent(ip.getNav(@"extern".owner_nav).name.toSlice(ip)),
            });
        } else {
            // MSVC has a limit of 4095 character token length limit, and fmtIdent can (worst case),
            // expand to 3x the length of its input, but let's cut it off at a much shorter limit.
            const fqn_slice = ip.getNav(nav_index).fqn.toSlice(ip);
            try writer.print("{}__{d}", .{
                fmtIdent(fqn_slice[0..@min(fqn_slice.len, 100)]),
                @intFromEnum(nav_index),
            });
        }
    }

    fn renderUavName(writer: anytype, uav: Value) !void {
        try writer.print("__anon_{d}", .{@intFromEnum(uav.toIntern())});
    }

    fn renderTypeForBuiltinFnName(dg: *DeclGen, writer: anytype, ty: Type) !void {
        try dg.renderCTypeForBuiltinFnName(writer, try dg.ctypeFromType(ty, .complete));
    }

    fn renderCTypeForBuiltinFnName(dg: *DeclGen, writer: anytype, ctype: CType) !void {
        switch (ctype.info(&dg.ctype_pool)) {
            else => |ctype_info| try writer.print("{c}{d}", .{
                if (ctype.isBool())
                    signAbbrev(.unsigned)
                else if (ctype.isInteger())
                    signAbbrev(ctype.signedness(dg.mod))
                else if (ctype.isFloat())
                    @as(u8, 'f')
                else if (ctype_info == .pointer)
                    @as(u8, 'p')
                else
                    return dg.fail("TODO: CBE: implement renderTypeForBuiltinFnName for {s} type", .{@tagName(ctype_info)}),
                if (ctype.isFloat()) ctype.floatActiveBits(dg.mod) else dg.byteSize(ctype) * 8,
            }),
            .array => try writer.writeAll("big"),
        }
    }

    fn renderBuiltinInfo(dg: *DeclGen, writer: anytype, ty: Type, info: BuiltinInfo) !void {
        const ctype = try dg.ctypeFromType(ty, .complete);
        const is_big = ctype.info(&dg.ctype_pool) == .array;
        switch (info) {
            .none => if (!is_big) return,
            .bits => {},
        }

        const pt = dg.pt;
        const zcu = pt.zcu;
        const int_info = if (ty.isAbiInt(zcu)) ty.intInfo(zcu) else std.builtin.Type.Int{
            .signedness = .unsigned,
            .bits = @as(u16, @intCast(ty.bitSize(zcu))),
        };

        if (is_big) try writer.print(", {}", .{int_info.signedness == .signed});
        try writer.print(", {}", .{try dg.fmtIntLiteral(
            try pt.intValue(if (is_big) .u16 else .u8, int_info.bits),
            .FunctionArgument,
        )});
    }

    fn fmtIntLiteral(
        dg: *DeclGen,
        val: Value,
        loc: ValueRenderLocation,
    ) !std.fmt.Formatter(formatIntLiteral) {
        const zcu = dg.pt.zcu;
        const kind = loc.toCTypeKind();
        const ty = val.typeOf(zcu);
        return std.fmt.Formatter(formatIntLiteral){ .data = .{
            .dg = dg,
            .int_info = ty.intInfo(zcu),
            .kind = kind,
            .ctype = try dg.ctypeFromType(ty, kind),
            .val = val,
        } };
    }
}
```
嗯，我现在需要总结这段Zig代码的思维模型。首先，这段代码看起来是一个Zig模块中的结构体DeclGen，主要用于生成C语言的声明和代码。让我先大致浏览一下代码的结构和各个部分的功能。

首先，DeclGen结构体包含了很多字段，比如gpa（内存分配器）、pt（每个线程的状态）、mod（模块指针）、pass（编译阶段信息）等等。这些字段应该是在代码生成过程中需要用到的各种上下文信息。

接下来，结构体内部定义了一些方法，比如fwdDeclWriter、fail、renderUav、renderNav、renderPointer、renderValue、renderUndefValue等等。这些方法似乎负责将Zig的不同类型和结构转换为对应的C代码。例如，renderUav和renderNav可能用于处理指针或引用的渲染，renderValue用于处理各种值的转换，包括整数、浮点数、结构体、联合体等。

在renderValue方法中，处理了不同类型的值，比如整数、浮点数、切片、指针、可选类型（option）、结构体、联合体等。每种类型都有对应的处理逻辑，例如对于浮点数，需要考虑不同的位数（16、32、64、80、128位）以及特殊值（如NaN、无穷大）。对于结构体和联合体，可能需要递归处理各个字段，生成对应的C结构体或联合体代码。

另外，renderUndefValue方法处理未定义的值，根据类型生成对应的未初始化值，比如用0xaa填充字节，或者在特定情况下生成对应的C代码。

还有一些辅助函数，比如renderTypeAndName、renderCType，用于生成C类型的声明和名称，考虑类型的前缀和后缀，比如指针、数组、函数指针等。

在处理指针和引用时，代码中多次提到需要处理对齐（alignment）、类型转换（cast）以及避免未定义行为（UB），比如在渲染函数指针时不应该进行类型转换，否则可能导致UB。

此外，代码中涉及到大量的条件判断和错误处理，比如使用try和catch来捕获可能的错误，并返回错误码如error.AnalysisFail或error.OutOfMemory。

在代码生成过程中，还考虑了不同平台和编译器的特性，例如MSVC不支持某些特性时，需要生成特定的宏或代码路径。例如，在处理浮点数的特殊值时，需要判断是否是静态初始化，以及在MSVC环境下如何处理NaN等。

另外，代码中使用了大量的Zig特性，如comptime、联合体（union）、枚举（enum）、错误处理等，这些特性在生成C代码时需要被正确转换。例如，使用comptime生成的类型信息需要在运行时生成对应的C类型声明。

总结一下，这段代码的主要功能是将Zig的高级语言特性（如类型系统、错误处理、内存管理）转换为等效的C代码，同时处理不同平台和编译器的差异，确保生成的代码既正确又高效。每个方法负责处理特定的类型或结构，递归地生成对应的C代码片段，并组合成完整的声明和定义。

不过，有些部分我还不太理解，比如uav_deps和aligned_uavs的作用，可能涉及到依赖管理和对齐处理。此外，Pass联合体的具体用途也需要进一步分析，可能表示不同的编译阶段或处理模式。需要更深入地查看这些部分的具体实现和调用上下文，才能完全理解其在整个代码生成过程中的作用。

总的来说，DeclGen结构体是一个复杂的代码生成器，负责将Zig的中间表示（IR）转换为C代码，处理各种类型、内存布局、对齐、平台差异等问题，确保生成的C代码能够正确编译并保持Zig的语义。
================================================
这段代码定义了一个名为`DeclGen`的结构体，主要用于将Zig的高级语言特性（如类型系统、内存管理、错误处理等）转换为C代码。其核心思维模型可总结如下：

---

### **1. 核心目标**
- **代码生成**：将Zig的中间表示（IR）转换为符合C语言规范的代码，处理类型、对齐、内存布局、平台差异等问题。
- **语义保留**：确保生成的C代码严格遵循Zig的语义，包括运行时行为、错误处理、未定义值等。

---

### **2. 关键数据结构**
- **`DeclGen`结构体**：保存代码生成所需的上下文信息：
  - **内存管理**：`gpa`（通用分配器）、`scratch`（临时内存池）。
  - **模块与线程状态**：`mod`（当前模块）、`pt`（线程局部状态）。
  - **输出管理**：`fwd_decl`（前向声明缓冲区）、`error_msg`（错误信息）。
  - **类型映射**：`ctype_pool`（Zig类型到C类型的映射池）。
  - **依赖管理**：`uav_deps`（匿名声明依赖）、`aligned_uavs`（对齐需求管理）。

---

### **3. 核心功能**
#### **类型与值的转换**
- **类型渲染**：
  - `renderType`和`renderCType`：将Zig类型转换为C类型声明（如`uint8_t*`、结构体、联合体等）。
  - 处理复杂类型：指针、数组、函数指针、可选类型（`Option`）、错误联合类型（`ErrorUnion`）。
- **值渲染**：
  - `renderValue`：将Zig值（整数、浮点数、切片、结构体、联合体等）转换为C表达式。
  - `renderUndefValue`：生成未初始化值的合法C表示（如用`0xaa`填充内存）。

#### **指针与内存管理**
- **指针派生**：处理指针的多种派生方式（如字段偏移、数组索引、类型转换）。
- **对齐处理**：根据Zig类型和显式对齐要求生成`alignas`或特定宏（如`zig_align`）。

#### **平台与编译器适配**
- **特殊值处理**：如浮点数的`NaN`、`Inf`在不同编译器（如MSVC）下的表示。
- **内联汇编支持**：通过`zig_naked`等宏处理裸函数（`naked function`）。

#### **错误处理**
- **错误传播**：通过`fail`方法收集错误信息，返回`AnalysisFail`或`OutOfMemory`。
- **错误值渲染**：将Zig错误类型映射为C的枚举或错误码。

---

### **4. 关键方法**
- **`renderUav`和`renderNav`**：
  - 处理Zig中的“匿名值”（Unnamed Anonymous Values, UAV）和“具名值”（Named Anonymous Values, NAV），生成C变量名（如`__anon_123`）。
- **`renderPointer`**：
  - 递归处理指针的派生步骤（如结构体字段访问、数组索引、类型转换）。
- **`renderFunctionSignature`**：
  - 生成函数签名，处理调用约定（`callconv`）、内联属性（`zig_cold`）、对齐（`zig_align_fn`）。

---

### **5. 设计模式**
- **递归处理**：如结构体、联合体的字段递归渲染。
- **类型驱动生成**：根据类型信息动态生成C代码（如`ctypeFromType`）。
- **条件编译**：通过平台宏（如`zig_make_special_f32`）适配不同编译器和目标。

---

### **6. 重要细节**
- **字符串字面量**：对`u8`数组优化为C字符串字面量（如`"hello"`）。
- **未定义行为规避**：避免对函数指针进行类型转换（可能导致UB）。
- **对齐优化**：仅对显式对齐高于ABI对齐的情况生成对齐指令。

---

### **7. 总结**
`DeclGen`是Zig编译器后端的关键组件，负责将高级Zig语义精准映射到C代码，同时处理底层细节（如内存布局、平台差异）。其设计核心是**类型驱动的递归代码生成**，结合复杂的上下文管理和错误处理，确保生成的C代码既高效又符合Zig的严格语义。