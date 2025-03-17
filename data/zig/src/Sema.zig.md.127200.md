```zig
fn resolvePeerTypesInner(
    sema: *Sema,
    block: *Block,
    src: LazySrcLoc,
    peer_tys: []?Type,
    peer_vals: []?Value,
) !PeerResolveResult {
    const pt = sema.pt;
    const zcu = pt.zcu;
    const ip = &zcu.intern_pool;

    var strat_reason: usize = 0;
    var s: PeerResolveStrategy = .unknown;
    for (peer_tys, 0..) |opt_ty, i| {
        const ty = opt_ty orelse continue;
        s = s.merge(PeerResolveStrategy.select(ty, zcu), &strat_reason, i);
    }

    if (s == .unknown) {
        // The whole thing was noreturn or undefined - try to do an exact match
        s = .exact;
    } else {
        // There was something other than noreturn and undefined, so we can ignore those peers
        for (peer_tys) |*ty_ptr| {
            const ty = ty_ptr.* orelse continue;
            switch (ty.zigTypeTag(zcu)) {
                .noreturn, .undefined => ty_ptr.* = null,
                else => {},
            }
        }
    }

    const target = zcu.getTarget();

    switch (s) {
        .unknown => unreachable,

        .error_set => {
            var final_set: ?Type = null;
            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                if (ty.zigTypeTag(zcu) != .error_set) return .{ .conflict = .{
                    .peer_idx_a = strat_reason,
                    .peer_idx_b = i,
                } };
                if (final_set) |cur_set| {
                    final_set = try sema.maybeMergeErrorSets(block, src, cur_set, ty);
                } else {
                    final_set = ty;
                }
            }
            return .{ .success = final_set.? };
        },

        .error_union => {
            var final_set: ?Type = null;
            for (peer_tys, peer_vals) |*ty_ptr, *val_ptr| {
                const ty = ty_ptr.* orelse continue;
                const set_ty = switch (ty.zigTypeTag(zcu)) {
                    .error_set => blk: {
                        ty_ptr.* = null; // no payload to decide on
                        val_ptr.* = null;
                        break :blk ty;
                    },
                    .error_union => blk: {
                        const set_ty = ty.errorUnionSet(zcu);
                        ty_ptr.* = ty.errorUnionPayload(zcu);
                        if (val_ptr.*) |eu_val| switch (ip.indexToKey(eu_val.toIntern())) {
                            .error_union => |eu| switch (eu.val) {
                                .payload => |payload_ip| val_ptr.* = Value.fromInterned(payload_ip),
                                .err_name => val_ptr.* = null,
                            },
                            .undef => val_ptr.* = Value.fromInterned(try pt.intern(.{ .undef = ty_ptr.*.?.toIntern() })),
                            else => unreachable,
                        };
                        break :blk set_ty;
                    },
                    else => continue, // whole type is the payload
                };
                if (final_set) |cur_set| {
                    final_set = try sema.maybeMergeErrorSets(block, src, cur_set, set_ty);
                } else {
                    final_set = set_ty;
                }
            }
            assert(final_set != null);
            const final_payload = switch (try sema.resolvePeerTypesInner(
                block,
                src,
                peer_tys,
                peer_vals,
            )) {
                .success => |ty| ty,
                else => |result| return result,
            };
            return .{ .success = try pt.errorUnionType(final_set.?, final_payload) };
        },

        .nullable => {
            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                if (!ty.eql(Type.null, zcu)) return .{ .conflict = .{
                    .peer_idx_a = strat_reason,
                    .peer_idx_b = i,
                } };
            }
            return .{ .success = Type.null };
        },

        .optional => {
            for (peer_tys, peer_vals) |*ty_ptr, *val_ptr| {
                const ty = ty_ptr.* orelse continue;
                switch (ty.zigTypeTag(zcu)) {
                    .null => {
                        ty_ptr.* = null;
                        val_ptr.* = null;
                    },
                    .optional => {
                        ty_ptr.* = ty.optionalChild(zcu);
                        if (val_ptr.*) |opt_val| val_ptr.* = if (!opt_val.isUndef(zcu)) opt_val.optionalValue(zcu) else null;
                    },
                    else => {},
                }
            }
            const child_ty = switch (try sema.resolvePeerTypesInner(
                block,
                src,
                peer_tys,
                peer_vals,
            )) {
                .success => |ty| ty,
                else => |result| return result,
            };
            return .{ .success = try pt.optionalType(child_ty.toIntern()) };
        },

        .array => {
            // Index of the first non-null peer
            var opt_first_idx: ?usize = null;
            // Index of the first array or vector peer (i.e. not a tuple)
            var opt_first_arr_idx: ?usize = null;
            // Set to non-null once we see any peer, even a tuple
            var len: u64 = undefined;
            var sentinel: ?Value = undefined;
            // Only set once we see a non-tuple peer
            var elem_ty: Type = undefined;

            for (peer_tys, 0..) |*ty_ptr, i| {
                const ty = ty_ptr.* orelse continue;

                if (!ty.isArrayOrVector(zcu)) {
                    // We allow tuples of the correct length. We won't validate their elem type, since the elements can be coerced.
                    const arr_like = sema.typeIsArrayLike(ty) orelse return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } };

                    if (opt_first_idx) |first_idx| {
                        if (arr_like.len != len) return .{ .conflict = .{
                            .peer_idx_a = first_idx,
                            .peer_idx_b = i,
                        } };
                    } else {
                        opt_first_idx = i;
                        len = arr_like.len;
                    }

                    sentinel = null;

                    continue;
                }

                const first_arr_idx = opt_first_arr_idx orelse {
                    if (opt_first_idx == null) {
                        opt_first_idx = i;
                        len = ty.arrayLen(zcu);
                        sentinel = ty.sentinel(zcu);
                    }
                    opt_first_arr_idx = i;
                    elem_ty = ty.childType(zcu);
                    continue;
                };

                if (ty.arrayLen(zcu) != len) return .{ .conflict = .{
                    .peer_idx_a = first_arr_idx,
                    .peer_idx_b = i,
                } };

                const peer_elem_ty = ty.childType(zcu);
                if (!peer_elem_ty.eql(elem_ty, zcu)) coerce: {
                    const peer_elem_coerces_to_elem =
                        try sema.coerceInMemoryAllowed(block, elem_ty, peer_elem_ty, false, zcu.getTarget(), src, src, null);
                    if (peer_elem_coerces_to_elem == .ok) {
                        break :coerce;
                    }

                    const elem_coerces_to_peer_elem =
                        try sema.coerceInMemoryAllowed(block, peer_elem_ty, elem_ty, false, zcu.getTarget(), src, src, null);
                    if (elem_coerces_to_peer_elem == .ok) {
                        elem_ty = peer_elem_ty;
                        break :coerce;
                    }

                    return .{ .conflict = .{
                        .peer_idx_a = first_arr_idx,
                        .peer_idx_b = i,
                    } };
                }

                if (sentinel) |cur_sent| {
                    if (ty.sentinel(zcu)) |peer_sent| {
                        if (!peer_sent.eql(cur_sent, elem_ty, zcu)) sentinel = null;
                    } else {
                        sentinel = null;
                    }
                }
            }

            // There should always be at least one array or vector peer
            assert(opt_first_arr_idx != null);

            return .{ .success = try pt.arrayType(.{
                .len = len,
                .child = elem_ty.toIntern(),
                .sentinel = if (sentinel) |sent_val| sent_val.toIntern() else .none,
            }) };
        },

        .vector => {
            var len: ?u64 = null;
            var first_idx: usize = undefined;
            for (peer_tys, peer_vals, 0..) |*ty_ptr, *val_ptr, i| {
                const ty = ty_ptr.* orelse continue;

                if (!ty.isArrayOrVector(zcu)) {
                    // Allow tuples of the correct length
                    const arr_like = sema.typeIsArrayLike(ty) orelse return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } };

                    if (len) |expect_len| {
                        if (arr_like.len != expect_len) return .{ .conflict = .{
                            .peer_idx_a = first_idx,
                            .peer_idx_b = i,
                        } };
                    } else {
                        len = arr_like.len;
                        first_idx = i;
                    }

                    // Tuples won't participate in the child type resolution. We'll resolve without
                    // them, and if the tuples have a bad type, we'll get a coercion error later.
                    ty_ptr.* = null;
                    val_ptr.* = null;

                    continue;
                }

                if (len) |expect_len| {
                    if (ty.arrayLen(zcu) != expect_len) return .{ .conflict = .{
                        .peer_idx_a = first_idx,
                        .peer_idx_b = i,
                    } };
                } else {
                    len = ty.arrayLen(zcu);
                    first_idx = i;
                }

                ty_ptr.* = ty.childType(zcu);
                val_ptr.* = null; // multiple child vals, so we can't easily use them in PTR
            }

            const child_ty = switch (try sema.resolvePeerTypesInner(
                block,
                src,
                peer_tys,
                peer_vals,
            )) {
                .success => |ty| ty,
                else => |result| return result,
            };

            return .{ .success = try pt.vectorType(.{
                .len = @intCast(len.?),
                .child = child_ty.toIntern(),
            }) };
        },

        .c_ptr => {
            var opt_ptr_info: ?InternPool.Key.PtrType = null;
            var first_idx: usize = undefined;
            for (peer_tys, peer_vals, 0..) |opt_ty, opt_val, i| {
                const ty = opt_ty orelse continue;
                switch (ty.zigTypeTag(zcu)) {
                    .comptime_int => continue, // comptime-known integers can always coerce to C pointers
                    .int => {
                        if (opt_val != null) {
                            // Always allow the coercion for comptime-known ints
                            continue;
                        } else {
                            // Runtime-known, so check if the type is no bigger than a usize
                            const ptr_bits = target.ptrBitWidth();
                            const bits = ty.intInfo(zcu).bits;
                            if (bits <= ptr_bits) continue;
                        }
                    },
                    .null => continue,
                    else => {},
                }

                if (!ty.isPtrAtRuntime(zcu)) return .{ .conflict = .{
                    .peer_idx_a = strat_reason,
                    .peer_idx_b = i,
                } };

                // Goes through optionals
                const peer_info = ty.ptrInfo(zcu);

                var ptr_info = opt_ptr_info orelse {
                    opt_ptr_info = peer_info;
                    opt_ptr_info.?.flags.size = .c;
                    first_idx = i;
                    continue;
                };

                // Try peer -> cur, then cur -> peer
                ptr_info.child = ((try sema.resolvePairInMemoryCoercible(block, src, Type.fromInterned(ptr_info.child), Type.fromInterned(peer_info.child))) orelse {
                    return .{ .conflict = .{
                        .peer_idx_a = first_idx,
                        .peer_idx_b = i,
                    } };
                }).toIntern();

                if (ptr_info.sentinel != .none and peer_info.sentinel != .none) {
                    const peer_sent = try ip.getCoerced(sema.gpa, pt.tid, ptr_info.sentinel, ptr_info.child);
                    const ptr_sent = try ip.getCoerced(sema.gpa, pt.tid, peer_info.sentinel, ptr_info.child);
                    if (ptr_sent == peer_sent) {
                        ptr_info.sentinel = ptr_sent;
                    } else {
                        ptr_info.sentinel = .none;
                    }
                } else {
                    ptr_info.sentinel = .none;
                }

                // Note that the align can be always non-zero; Zcu.ptrType will canonicalize it
                ptr_info.flags.alignment = InternPool.Alignment.min(
                    if (ptr_info.flags.alignment != .none)
                        ptr_info.flags.alignment
                    else
                        Type.fromInterned(ptr_info.child).abiAlignment(zcu),

                    if (peer_info.flags.alignment != .none)
                        peer_info.flags.alignment
                    else
                        Type.fromInterned(peer_info.child).abiAlignment(zcu),
                );
                if (ptr_info.flags.address_space != peer_info.flags.address_space) {
                    return .{ .conflict = .{
                        .peer_idx_a = first_idx,
                        .peer_idx_b = i,
                    } };
                }

                if (ptr_info.packed_offset.bit_offset != peer_info.packed_offset.bit_offset or
                    ptr_info.packed_offset.host_size != peer_info.packed_offset.host_size)
                {
                    return .{ .conflict = .{
                        .peer_idx_a = first_idx,
                        .peer_idx_b = i,
                    } };
                }

                ptr_info.flags.is_const = ptr_info.flags.is_const or peer_info.flags.is_const;
                ptr_info.flags.is_volatile = ptr_info.flags.is_volatile or peer_info.flags.is_volatile;

                opt_ptr_info = ptr_info;
            }
            return .{ .success = try pt.ptrTypeSema(opt_ptr_info.?) };
        },

        .ptr => {
            // If we've resolved to a `[]T` but then see a `[*]T`, we can resolve to a `[*]T` only
            // if there were no actual slices. Else, we want the slice index to report a conflict.
            var opt_slice_idx: ?usize = null;

            var any_abi_aligned = false;
            var opt_ptr_info: ?InternPool.Key.PtrType = null;
            var first_idx: usize = undefined;
            var other_idx: usize = undefined; // We sometimes need a second peer index to report a generic error

            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                const peer_info: InternPool.Key.PtrType = switch (ty.zigTypeTag(zcu)) {
                    .pointer => ty.ptrInfo(zcu),
                    .@"fn" => .{
                        .child = ty.toIntern(),
                        .flags = .{
                            .address_space = target_util.defaultAddressSpace(target, .global_constant),
                        },
                    },
                    else => return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } },
                };

                switch (peer_info.flags.size) {
                    .one, .many => {},
                    .slice => opt_slice_idx = i,
                    .c => return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } },
                }

                var ptr_info = opt_ptr_info orelse {
                    opt_ptr_info = peer_info;
                    first_idx = i;
                    continue;
                };

                other_idx = i;

                // We want to return this in a lot of cases, so alias it here for convenience
                const generic_err: PeerResolveResult = .{ .conflict = .{
                    .peer_idx_a = first_idx,
                    .peer_idx_b = i,
                } };

                // Note that the align can be always non-zero; Type.ptr will canonicalize it
                if (peer_info.flags.alignment == .none) {
                    any_abi_aligned = true;
                } else if (ptr_info.flags.alignment == .none) {
                    any_abi_aligned = true;
                    ptr_info.flags.alignment = peer_info.flags.alignment;
                } else {
                    ptr_info.flags.alignment = ptr_info.flags.alignment.minStrict(peer_info.flags.alignment);
                }

                if (ptr_info.flags.address_space != peer_info.flags.address_space) {
                    return generic_err;
                }

                if (ptr_info.packed_offset.bit_offset != peer_info.packed_offset.bit_offset or
                    ptr_info.packed_offset.host_size != peer_info.packed_offset.host_size)
                {
                    return generic_err;
                }

                ptr_info.flags.is_const = ptr_info.flags.is_const or peer_info.flags.is_const;
                ptr_info.flags.is_volatile = ptr_info.flags.is_volatile or peer_info.flags.is_volatile;
                ptr_info.flags.is_allowzero = ptr_info.flags.is_allowzero or peer_info.flags.is_allowzero;

                const peer_sentinel: InternPool.Index = switch (peer_info.flags.size) {
                    .one => switch (ip.indexToKey(peer_info.child)) {
                        .array_type => |array_type| array_type.sentinel,
                        else => .none,
                    },
                    .many, .slice => peer_info.sentinel,
                    .c => unreachable,
                };

                const cur_sentinel: InternPool.Index = switch (ptr_info.flags.size) {
                    .one => switch (ip.indexToKey(ptr_info.child)) {
                        .array_type => |array_type| array_type.sentinel,
                        else => .none,
                    },
                    .many, .slice => ptr_info.sentinel,
                    .c => unreachable,
                };

                // We abstract array handling slightly so that tuple pointers can work like array pointers
                const peer_pointee_array = sema.typeIsArrayLike(Type.fromInterned(peer_info.child));
                const cur_pointee_array = sema.typeIsArrayLike(Type.fromInterned(ptr_info.child));

                // This switch is just responsible for deciding the size and pointee (not including
                // single-pointer array sentinel).
                good: {
                    switch (peer_info.flags.size) {
                        .one => switch (ptr_info.flags.size) {
                            .one => {
                                if (try sema.resolvePairInMemoryCoercible(block, src, Type.fromInterned(ptr_info.child), Type.fromInterned(peer_info.child))) |pointee| {
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }

                                const cur_arr = cur_pointee_array orelse return generic_err;
                                const peer_arr = peer_pointee_array orelse return generic_err;

                                if (try sema.resolvePairInMemoryCoercible(block, src, cur_arr.elem_ty, peer_arr.elem_ty)) |elem_ty| {
                                    // *[n:x]T + *[n:y]T = *[n]T
                                    if (cur_arr.len == peer_arr.len) {
                                        ptr_info.child = (try pt.arrayType(.{
                                            .len = cur_arr.len,
                                            .child = elem_ty.toIntern(),
                                        })).toIntern();
                                        break :good;
                                    }
                                    // *[a]T + *[b]T = []T
                                    ptr_info.flags.size = .slice;
                                    ptr_info.child = elem_ty.toIntern();
                                    break :good;
                                }

                                if (peer_arr.elem_ty.toIntern() == .noreturn_type) {
                                    // *struct{} + *[a]T = []T
                                    ptr_info.flags.size = .slice;
                                    ptr_info.child = cur_arr.elem_ty.toIntern();
                                    break :good;
                                }

                                if (cur_arr.elem_ty.toIntern() == .noreturn_type) {
                                    // *[a]T + *struct{} = []T
                                    ptr_info.flags.size = .slice;
                                    ptr_info.child = peer_arr.elem_ty.toIntern();
                                    break :good;
                                }

                                return generic_err;
                            },
                            .many => {
                                // Only works for *[n]T + [*]T -> [*]T
                                const arr = peer_pointee_array orelse return generic_err;
                                if (try sema.resolvePairInMemoryCoercible(block, src, Type.fromInterned(ptr_info.child), arr.elem_ty)) |pointee| {
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }
                                if (arr.elem_ty.toIntern() == .noreturn_type) {
                                    // *struct{} + [*]T -> [*]T
                                    break :good;
                                }
                                return generic_err;
                            },
                            .slice => {
                                // Only works for *[n]T + []T -> []T
                                const arr = peer_pointee_array orelse return generic_err;
                                if (try sema.resolvePairInMemoryCoercible(block, src, Type.fromInterned(ptr_info.child), arr.elem_ty)) |pointee| {
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }
                                if (arr.elem_ty.toIntern() == .noreturn_type) {
                                    // *struct{} + []T -> []T
                                    break :good;
                                }
                                return generic_err;
                            },
                            .c => unreachable,
                        },
                        .many => switch (ptr_info.flags.size) {
                            .one => {
                                // Only works for [*]T + *[n]T -> [*]T
                                const arr = cur_pointee_array orelse return generic_err;
                                if (try sema.resolvePairInMemoryCoercible(block, src, arr.elem_ty, Type.fromInterned(peer_info.child))) |pointee| {
                                    ptr_info.flags.size = .many;
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }
                                if (arr.elem_ty.toIntern() == .noreturn_type) {
                                    // [*]T + *struct{} -> [*]T
                                    ptr_info.flags.size = .many;
                                    ptr_info.child = peer_info.child;
                                    break :good;
                                }
                                return generic_err;
                            },
                            .many => {
                                if (try sema.resolvePairInMemoryCoercible(block, src, Type.fromInterned(ptr_info.child), Type.fromInterned(peer_info.child))) |pointee| {
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }
                                return generic_err;
                            },
                            .slice => {
                                // Only works if no peers are actually slices
                                if (opt_slice_idx) |slice_idx| {
                                    return .{ .conflict = .{
                                        .peer_idx_a = slice_idx,
                                        .peer_idx_b = i,
                                    } };
                                }
                                // Okay, then works for [*]T + "[]T" -> [*]T
                                if (try sema.resolvePairInMemoryCoercible(block, src, Type.fromInterned(ptr_info.child), Type.fromInterned(peer_info.child))) |pointee| {
                                    ptr_info.flags.size = .many;
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }
                                return generic_err;
                            },
                            .c => unreachable,
                        },
                        .slice => switch (ptr_info.flags.size) {
                            .one => {
                                // Only works for []T + *[n]T -> []T
                                const arr = cur_pointee_array orelse return generic_err;
                                if (try sema.resolvePairInMemoryCoercible(block, src, arr.elem_ty, Type.fromInterned(peer_info.child))) |pointee| {
                                    ptr_info.flags.size = .slice;
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }
                                if (arr.elem_ty.toIntern() == .noreturn_type) {
                                    // []T + *struct{} -> []T
                                    ptr_info.flags.size = .slice;
                                    ptr_info.child = peer_info.child;
                                    break :good;
                                }
                                return generic_err;
                            },
                            .many => {
                                // Impossible! (current peer is an actual slice)
                                return generic_err;
                            },
                            .slice => {
                                if (try sema.resolvePairInMemoryCoercible(block, src, Type.fromInterned(ptr_info.child), Type.fromInterned(peer_info.child))) |pointee| {
                                    ptr_info.child = pointee.toIntern();
                                    break :good;
                                }
                                return generic_err;
                            },
                            .c => unreachable,
                        },
                        .c => unreachable,
                    }
                }

                const sentinel_ty = switch (ptr_info.flags.size) {
                    .one => switch (ip.indexToKey(ptr_info.child)) {
                        .array_type => |array_type| array_type.child,
                        else => ptr_info.child,
                    },
                    .many, .slice, .c => ptr_info.child,
                };

                sentinel: {
                    no_sentinel: {
                        if (peer_sentinel == .none) break :no_sentinel;
                        if (cur_sentinel == .none) break :no_sentinel;
                        const peer_sent_coerced = try ip.getCoerced(sema.gpa, pt.tid, peer_sentinel, sentinel_ty);
                        const cur_sent_coerced = try ip.getCoerced(sema.gpa, pt.tid, cur_sentinel, sentinel_ty);
                        if (peer_sent_coerced != cur_sent_coerced) break :no_sentinel;
                        // Sentinels match
                        if (ptr_info.flags.size == .one) switch (ip.indexToKey(ptr_info.child)) {
                            .array_type => |array_type| ptr_info.child = (try pt.arrayType(.{
                                .len = array_type.len,
                                .child = array_type.child,
                                .sentinel = cur_sent_coerced,
                            })).toIntern(),
                            else => unreachable,
                        } else {
                            ptr_info.sentinel = cur_sent_coerced;
                        }
                        break :sentinel;
                    }
                    // Clear existing sentinel
                    ptr_info.sentinel = .none;
                    if (ptr_info.flags.size == .one) switch (ip.indexToKey(ptr_info.child)) {
                        .array_type => |array_type| ptr_info.child = (try pt.arrayType(.{
                            .len = array_type.len,
                            .child = array_type.child,
                            .sentinel = .none,
                        })).toIntern(),
                        else => {},
                    };
                }

                opt_ptr_info = ptr_info;
            }

            // Before we succeed, check the pointee type. If we tried to apply PTR to (for instance)
            // &.{} and &.{}, we'll currently have a pointer type of `*[0]noreturn` - we wanted to
            // coerce the empty struct to a specific type, but no peer provided one. We need to
            // detect this case and emit an error.
            const pointee = opt_ptr_info.?.child;
            switch (pointee) {
                .noreturn_type => return .{ .conflict = .{
                    .peer_idx_a = first_idx,
                    .peer_idx_b = other_idx,
                } },
                else => switch (ip.indexToKey(pointee)) {
                    .array_type => |array_type| if (array_type.child == .noreturn_type) return .{ .conflict = .{
                        .peer_idx_a = first_idx,
                        .peer_idx_b = other_idx,
                    } },
                    else => {},
                },
            }

            if (any_abi_aligned and opt_ptr_info.?.flags.alignment != .none) {
                opt_ptr_info.?.flags.alignment = opt_ptr_info.?.flags.alignment.minStrict(
                    try Type.fromInterned(pointee).abiAlignmentSema(pt),
                );
            }

            return .{ .success = try pt.ptrTypeSema(opt_ptr_info.?) };
        },

        .func => {
            var opt_cur_ty: ?Type = null;
            var first_idx: usize = undefined;
            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                const cur_ty = opt_cur_ty orelse {
                    opt_cur_ty = ty;
                    first_idx = i;
                    continue;
                };
                if (ty.zigTypeTag(zcu) != .@"fn") return .{ .conflict = .{
                    .peer_idx_a = strat_reason,
                    .peer_idx_b = i,
                } };
                // ty -> cur_ty
                if (.ok == try sema.coerceInMemoryAllowedFns(block, cur_ty, ty, false, target, src, src)) {
                    continue;
                }
                // cur_ty -> ty
                if (.ok == try sema.coerceInMemoryAllowedFns(block, ty, cur_ty, false, target, src, src)) {
                    opt_cur_ty = ty;
                    continue;
                }
                return .{ .conflict = .{
                    .peer_idx_a = first_idx,
                    .peer_idx_b = i,
                } };
            }
            return .{ .success = opt_cur_ty.? };
        },

        .enum_or_union => {
            var opt_cur_ty: ?Type = null;
            // The peer index which gave the current type
            var cur_ty_idx: usize = undefined;

            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                switch (ty.zigTypeTag(zcu)) {
                    .enum_literal, .@"enum", .@"union" => {},
                    else => return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } },
                }
                const cur_ty = opt_cur_ty orelse {
                    opt_cur_ty = ty;
                    cur_ty_idx = i;
                    continue;
                };

                // We want to return this in a lot of cases, so alias it here for convenience
                const generic_err: PeerResolveResult = .{ .conflict = .{
                    .peer_idx_a = cur_ty_idx,
                    .peer_idx_b = i,
                } };

                switch (cur_ty.zigTypeTag(zcu)) {
                    .enum_literal => {
                        opt_cur_ty = ty;
                        cur_ty_idx = i;
                    },
                    .@"enum" => switch (ty.zigTypeTag(zcu)) {
                        .enum_literal => {},
                        .@"enum" => {
                            if (!ty.eql(cur_ty, zcu)) return generic_err;
                        },
                        .@"union" => {
                            const tag_ty = ty.unionTagTypeHypothetical(zcu);
                            if (!tag_ty.eql(cur_ty, zcu)) return generic_err;
                            opt_cur_ty = ty;
                            cur_ty_idx = i;
                        },
                        else => unreachable,
                    },
                    .@"union" => switch (ty.zigTypeTag(zcu)) {
                        .enum_literal => {},
                        .@"enum" => {
                            const cur_tag_ty = cur_ty.unionTagTypeHypothetical(zcu);
                            if (!ty.eql(cur_tag_ty, zcu)) return generic_err;
                        },
                        .@"union" => {
                            if (!ty.eql(cur_ty, zcu)) return generic_err;
                        },
                        else => unreachable,
                    },
                    else => unreachable,
                }
            }
            return .{ .success = opt_cur_ty.? };
        },

        .comptime_int => {
            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                switch (ty.zigTypeTag(zcu)) {
                    .comptime_int => {},
                    else => return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } },
                }
            }
            return .{ .success = Type.comptime_int };
        },

        .comptime_float => {
            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                switch (ty.zigTypeTag(zcu)) {
                    .comptime_int, .comptime_float => {},
                    else => return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } },
                }
            }
            return .{ .success = Type.comptime_float };
        },

        .fixed_int => {
            var idx_unsigned: ?usize = null;
            var idx_signed: ?usize = null;

            // TODO: this is for compatibility with legacy behavior. See beneath the loop.
            var any_comptime_known = false;

            for (peer_tys, peer_vals, 0..) |opt_ty, *ptr_opt_val, i| {
                const ty = opt_ty orelse continue;
                const opt_val = ptr_opt_val.*;

                const peer_tag = ty.zigTypeTag(zcu);
                switch (peer_tag) {
                    .comptime_int => {
                        // If the value is undefined, we can't refine to a fixed-width int
                        if (opt_val == null or opt_val.?.isUndef(zcu)) return .{ .conflict = .{
                            .peer_idx_a = strat_reason,
                            .peer_idx_b = i,
                        } };
                        any_comptime_known = true;
                        ptr_opt_val.* = try sema.resolveLazyValue(opt_val.?);
                        continue;
                    },
                    .int => {},
                    else => return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } },
                }

                if (opt_val != null) any_comptime_known = true;

                const info = ty.intInfo(zcu);

                const idx_ptr = switch (info.signedness) {
                    .unsigned => &idx_unsigned,
                    .signed => &idx_signed,
                };

                const largest_idx = idx_ptr.* orelse {
                    idx_ptr.* = i;
                    continue;
                };

                const cur_info = peer_tys[largest_idx].?.intInfo(zcu);
                if (info.bits > cur_info.bits) {
                    idx_ptr.* = i;
                }
            }

            if (idx_signed == null) {
                return .{ .success = peer_tys[idx_unsigned.?].? };
            }

            if (idx_unsigned == null) {
                return .{ .success = peer_tys[idx_signed.?].? };
            }

            const unsigned_info = peer_tys[idx_unsigned.?].?.intInfo(zcu);
            const signed_info = peer_tys[idx_signed.?].?.intInfo(zcu);
            if (signed_info.bits > unsigned_info.bits) {
                return .{ .success = peer_tys[idx_signed.?].? };
            }

            // TODO: this is for compatibility with legacy behavior. Before this version of PTR was
            // implemented, the algorithm very often returned false positives, with the expectation
            // that you'd just hit a coercion error later. One of these was that for integers, the
            // largest type would always be returned, even if it couldn't fit everything. This had
            // an unintentional consequence to semantics, which is that if values were known at
            // comptime, they would be coerced down to the smallest type where possible. This
            // behavior is unintuitive and order-dependent, so in my opinion should be eliminated,
            // but for now we'll retain compatibility.
            if (any_comptime_known) {
                if (unsigned_info.bits > signed_info.bits) {
                    return .{ .success = peer_tys[idx_unsigned.?].? };
                }
                const idx = @min(idx_unsigned.?, idx_signed.?);
                return .{ .success = peer_tys[idx].? };
            }

            return .{ .conflict = .{
                .peer_idx_a = idx_unsigned.?,
                .peer_idx_b = idx_signed.?,
            } };
        },

        .fixed_float => {
            var opt_cur_ty: ?Type = null;

            for (peer_tys, peer_vals, 0..) |opt_ty, opt_val, i| {
                const ty = opt_ty orelse continue;
                switch (ty.zigTypeTag(zcu)) {
                    .comptime_float, .comptime_int => {},
                    .int => {
                        if (opt_val == null) return .{ .conflict = .{
                            .peer_idx_a = strat_reason,
                            .peer_idx_b = i,
                        } };
                    },
                    .float => {
                        if (opt_cur_ty) |cur_ty| {
                            if (cur_ty.eql(ty, zcu)) continue;
                            // Recreate the type so we eliminate any c_longdouble
                            const bits = @max(cur_ty.floatBits(target), ty.floatBits(target));
                            opt_cur_ty = switch (bits) {
                                16 => Type.f16,
                                32 => Type.f32,
                                64 => Type.f64,
                                80 => Type.f80,
                                128 => Type.f128,
                                else => unreachable,
                            };
                        } else {
                            opt_cur_ty = ty;
                        }
                    },
                    else => return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } },
                }
            }

            // Note that fixed_float is only chosen if there is at least one fixed-width float peer,
            // so opt_cur_ty must be non-null.
            return .{ .success = opt_cur_ty.? };
        },

        .tuple => {
            // First, check that every peer has the same approximate structure (field count)

            var opt_first_idx: ?usize = null;
            var is_tuple: bool = undefined;
            var field_count: usize = undefined;

            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;

                if (!ty.isTuple(zcu)) {
                    return .{ .conflict = .{
                        .peer_idx_a = strat_reason,
                        .peer_idx_b = i,
                    } };
                }

                const first_idx = opt_first_idx orelse {
                    opt_first_idx = i;
                    is_tuple = ty.isTuple(zcu);
                    field_count = ty.structFieldCount(zcu);
                    continue;
                };

                if (ty.structFieldCount(zcu) != field_count) {
                    return .{ .conflict = .{
                        .peer_idx_a = first_idx,
                        .peer_idx_b = i,
                    } };
                }
            }

            assert(opt_first_idx != null);

            // Now, we'll recursively resolve the field types
            const field_types = try sema.arena.alloc(InternPool.Index, field_count);
            // Values for `comptime` fields - `.none` used for non-comptime fields
            const field_vals = try sema.arena.alloc(InternPool.Index, field_count);
            const sub_peer_tys = try sema.arena.alloc(?Type, peer_tys.len);
            const sub_peer_vals = try sema.arena.alloc(?Value, peer_vals.len);

            for (field_types, field_vals, 0..) |*field_ty, *field_val, field_index| {
                // Fill buffers with types and values of the field
                for (peer_tys, peer_vals, sub_peer_tys, sub_peer_vals) |opt_ty, opt_val, *peer_field_ty, *peer_field_val| {
                    const ty = opt_ty orelse {
                        peer_field_ty.* = null;
                        peer_field_val.* = null;
                        continue;
                    };
                    peer_field_ty.* = ty.fieldType(field_index, zcu);
                    peer_field_val.* = if (opt_val) |val| try val.fieldValue(pt, field_index) else null;
                }

                // Resolve field type recursively
                field_ty.* = switch (try sema.resolvePeerTypesInner(block, src, sub_peer_tys, sub_peer_vals)) {
                    .success => |ty| ty.toIntern(),
                    else => |result| {
                        const result_buf = try sema.arena.create(PeerResolveResult);
                        result_buf.* = result;
                        const field_name = try ip.getOrPutStringFmt(sema.gpa, pt.tid, "{d}", .{field_index}, .no_embedded_nulls);

                        // The error info needs the field types, but we can't reuse sub_peer_tys
                        // since the recursive call may have clobbered it.
                        const peer_field_tys = try sema.arena.alloc(Type, peer_tys.len);
                        for (peer_tys, peer_field_tys) |opt_ty, *peer_field_ty| {
                            // Already-resolved types won't be referenced by the error so it's fine
                            // to leave them undefined.
                            const ty = opt_ty orelse continue;
                            peer_field_ty.* = ty.fieldType(field_index, zcu);
                        }

                        return .{ .field_error = .{
                            .field_name = field_name,
                            .field_types = peer_field_tys,
                            .sub_result = result_buf,
                        } };
                    },
                };

                // Decide if this is a comptime field. If it is comptime in all peers, and the
                // coerced comptime values are all the same, we say it is comptime, else not.

                var comptime_val: ?Value = null;
                for (peer_tys) |opt_ty| {
                    const struct_ty = opt_ty orelse continue;
                    try struct_ty.resolveStructFieldInits(pt);

                    const uncoerced_field_val = try struct_ty.structFieldValueComptime(pt, field_index) orelse {
                        comptime_val = null;
                        break;
                    };
                    const uncoerced_field = Air.internedToRef(uncoerced_field_val.toIntern());
                    const coerced_inst = sema.coerceExtra(block, Type.fromInterned(field_ty.*), uncoerced_field, src, .{ .report_err = false }) catch |err| switch (err) {
                        // It's possible for PTR to give false positives. Just give up on making this a comptime field, we'll get an error later anyway
                        error.NotCoercible => {
                            comptime_val = null;
                            break;
                        },
                        else => |e| return e,
                    };
                    const coerced_val = (try sema.resolveValue(coerced_inst)) orelse continue;
                    const existing = comptime_val orelse {
                        comptime_val = coerced_val;
                        continue;
                    };
                    if (!coerced_val.eql(existing, Type.fromInterned(field_ty.*), zcu)) {
                        comptime_val = null;
                        break;
                    }
                }

                field_val.* = if (comptime_val) |v| v.toIntern() else .none;
            }

            const final_ty = try ip.getTupleType(zcu.gpa, pt.tid, .{
                .types = field_types,
                .values = field_vals,
            });

            return .{ .success = Type.fromInterned(final_ty) };
        },

        .exact => {
            var expect_ty: ?Type = null;
            var first_idx: usize = undefined;
            for (peer_tys, 0..) |opt_ty, i| {
                const ty = opt_ty orelse continue;
                if (expect_ty) |expect| {
                    if (!ty.eql(expect, zcu)) return .{ .conflict = .{
                        .peer_idx_a = first_idx,
                        .peer_idx_b = i,
                    } };
                } else {
                    expect_ty = ty;
                    first_idx = i;
                }
            }
            return .{ .success = expect_ty.? };
        },
    }
}
```
嗯，我现在要理解这段Zig代码，看起来是一个叫做resolvePeerTypesInner的函数。这个函数似乎是在处理类型解析的问题，特别是当有多个“peer”类型需要协调时。让我先大致看一下代码的结构和各个部分的功能。

首先，函数接收的参数有sema（可能是语义分析器）、block（代码块）、src（源代码位置）、peer_tys（可能是需要协调的类型列表）和peer_vals（对应的值列表）。返回类型是PeerResolveResult，可能表示解析成功或冲突。

函数一开始初始化了一些变量，比如pt、zcu、ip，然后遍历peer_tys来确定一个PeerResolveStrategy的策略s。这里调用了s.merge方法，可能是在合并不同peer类型的策略。如果所有peer类型都是unknown（比如noreturn或undefined），则设置为exact策略，否则过滤掉这些类型。

接下来是一个大的switch语句，根据不同的策略s处理不同的情况。比如，当策略是error_set时，合并错误集；如果是error_union，处理错误联合类型；nullable处理可空类型；array处理数组或向量类型；ptr处理指针类型等等。

每个case内部都有详细的处理逻辑，比如检查类型是否兼容，合并错误集，处理数组长度和元素类型，指针的对齐和地址空间，函数类型的兼容性，枚举或联合类型的一致性，整数和浮点类型的处理，元组类型的字段匹配，最后是exact策略下的类型完全匹配。

在阅读过程中，我注意到很多地方都有错误处理，返回.conflict表示类型冲突，比如数组长度不一致，指针类型不兼容，函数参数不匹配等。此外，有些部分涉及到递归调用resolvePeerTypesInner来处理嵌套的类型结构，比如元组类型的每个字段类型都需要递归解析。

有几个疑问点：比如在处理指针类型时，如何确定指针的子类型和对齐方式？在元组类型处理中，如何确保所有peer的字段数量和类型一致？还有，如何处理编译时已知的值（comptime_int/comptime_float）与固定类型（如i32/f32）之间的协调？

可能需要更仔细地分析每个策略分支的具体逻辑，特别是那些涉及类型合并和递归处理的部分。此外，理解Zig的类型系统如何表示（如错误集、联合、数组、指针等）对于理解这段代码至关重要。

总结来说，这个函数的主要作用是在给定多个peer类型的情况下，找到它们之间的共同类型，或者确定它们无法协调的原因。不同的策略处理不同的类型类别，通过递归和合并逻辑逐步构建最终的解析结果。
================================================
这段代码的思维模型可以分为以下几个关键部分：

---

### **1. 核心目标**
**动态协调多种类型**：在给定一组可能为空的 peer 类型（`peer_tys`）和对应的值（`peer_vals`）时，通过递归和策略模式，找到一个所有类型都能隐式转换的共同类型。若无法协调，则返回类型冲突的具体位置。

---

### **2. 策略选择与合并**
- **初始化策略**：遍历所有 peer 类型，通过 `PeerResolveStrategy.select` 为每个类型生成策略，并通过 `merge` 逐步合并成一个最终策略 `s`。
- **默认策略**：若所有类型均为 `noreturn` 或 `undefined`，则退化为 `exact` 策略（要求完全匹配）；否则过滤掉这些无效类型。

---

### **3. 策略分派（Switch-Case 逻辑）**
根据策略 `s` 分派到不同的处理逻辑，核心分支包括：

#### **基础类型**
- **`error_set`**：合并多个错误集类型，若存在非错误集类型则冲突。
- **`error_union`**：提取错误联合的负载类型，递归协调负载类型后重新包装为错误联合。
- **`nullable`/`optional`**：处理可空类型的子类型，递归协调子类型后包装为可空类型。

#### **复合类型**
- **`array`/`vector`**：
  - 校验所有数组/向量的长度一致。
  - 递归协调元素类型，处理哨兵值（sentinel）和内存对齐。
- **`c_ptr`/`ptr`**：
  - 协调指针的子类型、对齐、地址空间等属性。
  - 处理指针的特殊情况（如 `*[n]T` 与 `[*]T` 的兼容性）。
- **`tuple`**：
  - 确保所有元组的字段数量一致。
  - 递归协调每个字段的类型和编译时值。

#### **函数与特殊类型**
- **`func`**：校验函数签名的兼容性（参数、返回类型等）。
- **`enum_or_union`**：确保枚举或联合类型的一致性（如联合的标签类型匹配）。
- **`comptime_int`/`fixed_int`**：处理编译时整数与固定宽度整数的隐式转换。
- **`exact`**：要求所有类型完全一致，否则冲突。

---

### **4. 递归与错误处理**
- **递归调用**：在复合类型（如元组、数组、错误联合）中，递归协调子类型（如元组字段、数组元素）。
- **冲突定位**：通过 `peer_idx_a` 和 `peer_idx_b` 精确报告类型冲突的位置。
- **编译时值协调**：若值在编译时已知（如 `comptime_int`），尝试隐式转换或触发错误。

---

### **5. 内存与对齐处理**
- **指针对齐**：对齐值取所有指针的最小对齐值，并考虑 ABI 对齐约束。
- **内存兼容性**：通过 `coerceInMemoryAllowed` 检查类型是否可在内存中隐式转换。

---

### **6. 特殊逻辑**
- **空类型兼容性**：如 `noreturn` 或零长度数组（`[0]T`）可兼容任意类型。
- **哨兵值消除**：若哨兵值不一致，则丢弃哨兵（如数组的哨兵）。
- **编译时值优化**：若所有字段的编译时值一致，则标记为编译时字段。

---

### **总结**
该函数通过分层的策略模式，逐步分解复杂类型（如指针、元组、错误联合），递归协调子类型，最终生成一个统一的类型或报告冲突。其核心思想是：
1. **策略驱动**：不同策略处理不同类型类别的协调逻辑。
2. **递归分解**：复合类型逐层拆解为子类型问题。
3. **编译时与运行时结合**：优先利用编译时已知值优化类型推断，同时确保运行时兼容性。