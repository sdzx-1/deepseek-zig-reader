```zig
pub fn get(ip: *InternPool, gpa: Allocator, tid: Zcu.PerThread.Id, key: Key) Allocator.Error!Index {
    var gop = try ip.getOrPutKey(gpa, tid, key);
    defer gop.deinit();
    if (gop == .existing) return gop.existing;
    const local = ip.getLocal(tid);
    const items = local.getMutableItems(gpa);
    const extra = local.getMutableExtra(gpa);
    try items.ensureUnusedCapacity(1);
    switch (key) {
        .int_type => |int_type| {
            const t: Tag = switch (int_type.signedness) {
                .signed => .type_int_signed,
                .unsigned => .type_int_unsigned,
            };
            items.appendAssumeCapacity(.{
                .tag = t,
                .data = int_type.bits,
            });
        },
        .ptr_type => |ptr_type| {
            assert(ptr_type.child != .none);
            assert(ptr_type.sentinel == .none or ip.typeOf(ptr_type.sentinel) == ptr_type.child);

            if (ptr_type.flags.size == .slice) {
                gop.cancel();
                var new_key = key;
                new_key.ptr_type.flags.size = .many;
                const ptr_type_index = try ip.get(gpa, tid, new_key);
                gop = try ip.getOrPutKey(gpa, tid, key);

                try items.ensureUnusedCapacity(1);
                items.appendAssumeCapacity(.{
                    .tag = .type_slice,
                    .data = @intFromEnum(ptr_type_index),
                });
                return gop.put();
            }

            var ptr_type_adjusted = ptr_type;
            if (ptr_type.flags.size == .c) ptr_type_adjusted.flags.is_allowzero = true;

            items.appendAssumeCapacity(.{
                .tag = .type_pointer,
                .data = try addExtra(extra, ptr_type_adjusted),
            });
        },
        .array_type => |array_type| {
            assert(array_type.child != .none);
            assert(array_type.sentinel == .none or ip.typeOf(array_type.sentinel) == array_type.child);

            if (std.math.cast(u32, array_type.len)) |len| {
                if (array_type.sentinel == .none) {
                    items.appendAssumeCapacity(.{
                        .tag = .type_array_small,
                        .data = try addExtra(extra, Vector{
                            .len = len,
                            .child = array_type.child,
                        }),
                    });
                    return gop.put();
                }
            }

            const length = Array.Length.init(array_type.len);
            items.appendAssumeCapacity(.{
                .tag = .type_array_big,
                .data = try addExtra(extra, Array{
                    .len0 = length.a,
                    .len1 = length.b,
                    .child = array_type.child,
                    .sentinel = array_type.sentinel,
                }),
            });
        },
        .vector_type => |vector_type| {
            items.appendAssumeCapacity(.{
                .tag = .type_vector,
                .data = try addExtra(extra, Vector{
                    .len = vector_type.len,
                    .child = vector_type.child,
                }),
            });
        },
        .opt_type => |payload_type| {
            assert(payload_type != .none);
            items.appendAssumeCapacity(.{
                .tag = .type_optional,
                .data = @intFromEnum(payload_type),
            });
        },
        .anyframe_type => |payload_type| {
            // payload_type might be none, indicating the type is `anyframe`.
            items.appendAssumeCapacity(.{
                .tag = .type_anyframe,
                .data = @intFromEnum(payload_type),
            });
        },
        .error_union_type => |error_union_type| {
            items.appendAssumeCapacity(if (error_union_type.error_set_type == .anyerror_type) .{
                .tag = .type_anyerror_union,
                .data = @intFromEnum(error_union_type.payload_type),
            } else .{
                .tag = .type_error_union,
                .data = try addExtra(extra, error_union_type),
            });
        },
        .error_set_type => |error_set_type| {
            assert(error_set_type.names_map == .none);
            assert(std.sort.isSorted(NullTerminatedString, error_set_type.names.get(ip), {}, NullTerminatedString.indexLessThan));
            const names = error_set_type.names.get(ip);
            const names_map = try ip.addMap(gpa, tid, names.len);
            ip.addStringsToMap(names_map, names);
            const names_len = error_set_type.names.len;
            try extra.ensureUnusedCapacity(@typeInfo(Tag.ErrorSet).@"struct".fields.len + names_len);
            items.appendAssumeCapacity(.{
                .tag = .type_error_set,
                .data = addExtraAssumeCapacity(extra, Tag.ErrorSet{
                    .names_len = names_len,
                    .names_map = names_map,
                }),
            });
            extra.appendSliceAssumeCapacity(.{@ptrCast(error_set_type.names.get(ip))});
        },
        .inferred_error_set_type => |ies_index| {
            items.appendAssumeCapacity(.{
                .tag = .type_inferred_error_set,
                .data = @intFromEnum(ies_index),
            });
        },
        .simple_type => |simple_type| {
            assert(@intFromEnum(simple_type) == items.mutate.len);
            items.appendAssumeCapacity(.{
                .tag = .simple_type,
                .data = 0, // avoid writing `undefined` bits to a file
            });
        },
        .simple_value => |simple_value| {
            assert(@intFromEnum(simple_value) == items.mutate.len);
            items.appendAssumeCapacity(.{
                .tag = .simple_value,
                .data = 0, // avoid writing `undefined` bits to a file
            });
        },
        .undef => |ty| {
            assert(ty != .none);
            items.appendAssumeCapacity(.{
                .tag = .undef,
                .data = @intFromEnum(ty),
            });
        },

        .struct_type => unreachable, // use getStructType() instead
        .tuple_type => unreachable, // use getTupleType() instead
        .union_type => unreachable, // use getUnionType() instead
        .opaque_type => unreachable, // use getOpaqueType() instead

        .enum_type => unreachable, // use getEnumType() instead
        .func_type => unreachable, // use getFuncType() instead
        .@"extern" => unreachable, // use getExtern() instead
        .func => unreachable, // use getFuncInstance() or getFuncDecl() instead
        .un => unreachable, // use getUnion instead

        .variable => |variable| {
            const has_init = variable.init != .none;
            if (has_init) assert(variable.ty == ip.typeOf(variable.init));
            items.appendAssumeCapacity(.{
                .tag = .variable,
                .data = try addExtra(extra, Tag.Variable{
                    .ty = variable.ty,
                    .init = variable.init,
                    .owner_nav = variable.owner_nav,
                    .flags = .{
                        .is_const = false,
                        .is_threadlocal = variable.is_threadlocal,
                        .is_weak_linkage = variable.is_weak_linkage,
                        .is_dll_import = false,
                    },
                }),
            });
        },

        .slice => |slice| {
            assert(ip.indexToKey(slice.ty).ptr_type.flags.size == .slice);
            assert(ip.indexToKey(ip.typeOf(slice.ptr)).ptr_type.flags.size == .many);
            items.appendAssumeCapacity(.{
                .tag = .ptr_slice,
                .data = try addExtra(extra, PtrSlice{
                    .ty = slice.ty,
                    .ptr = slice.ptr,
                    .len = slice.len,
                }),
            });
        },

        .ptr => |ptr| {
            const ptr_type = ip.indexToKey(ptr.ty).ptr_type;
            assert(ptr_type.flags.size != .slice);
            items.appendAssumeCapacity(switch (ptr.base_addr) {
                .nav => |nav| .{
                    .tag = .ptr_nav,
                    .data = try addExtra(extra, PtrNav.init(ptr.ty, nav, ptr.byte_offset)),
                },
                .comptime_alloc => |alloc_index| .{
                    .tag = .ptr_comptime_alloc,
                    .data = try addExtra(extra, PtrComptimeAlloc.init(ptr.ty, alloc_index, ptr.byte_offset)),
                },
                .uav => |uav| if (ptrsHaveSameAlignment(ip, ptr.ty, ptr_type, uav.orig_ty)) item: {
                    if (ptr.ty != uav.orig_ty) {
                        gop.cancel();
                        var new_key = key;
                        new_key.ptr.base_addr.uav.orig_ty = ptr.ty;
                        gop = try ip.getOrPutKey(gpa, tid, new_key);
                        if (gop == .existing) return gop.existing;
                    }
                    break :item .{
                        .tag = .ptr_uav,
                        .data = try addExtra(extra, PtrUav.init(ptr.ty, uav.val, ptr.byte_offset)),
                    };
                } else .{
                    .tag = .ptr_uav_aligned,
                    .data = try addExtra(extra, PtrUavAligned.init(ptr.ty, uav.val, uav.orig_ty, ptr.byte_offset)),
                },
                .comptime_field => |field_val| item: {
                    assert(field_val != .none);
                    break :item .{
                        .tag = .ptr_comptime_field,
                        .data = try addExtra(extra, PtrComptimeField.init(ptr.ty, field_val, ptr.byte_offset)),
                    };
                },
                .eu_payload, .opt_payload => |base| item: {
                    switch (ptr.base_addr) {
                        .eu_payload => assert(ip.indexToKey(
                            ip.indexToKey(ip.typeOf(base)).ptr_type.child,
                        ) == .error_union_type),
                        .opt_payload => assert(ip.indexToKey(
                            ip.indexToKey(ip.typeOf(base)).ptr_type.child,
                        ) == .opt_type),
                        else => unreachable,
                    }
                    break :item .{
                        .tag = switch (ptr.base_addr) {
                            .eu_payload => .ptr_eu_payload,
                            .opt_payload => .ptr_opt_payload,
                            else => unreachable,
                        },
                        .data = try addExtra(extra, PtrBase.init(ptr.ty, base, ptr.byte_offset)),
                    };
                },
                .int => .{
                    .tag = .ptr_int,
                    .data = try addExtra(extra, PtrInt.init(ptr.ty, ptr.byte_offset)),
                },
                .arr_elem, .field => |base_index| {
                    const base_ptr_type = ip.indexToKey(ip.typeOf(base_index.base)).ptr_type;
                    switch (ptr.base_addr) {
                        .arr_elem => assert(base_ptr_type.flags.size == .many),
                        .field => {
                            assert(base_ptr_type.flags.size == .one);
                            switch (ip.indexToKey(base_ptr_type.child)) {
                                .tuple_type => |tuple_type| {
                                    assert(ptr.base_addr == .field);
                                    assert(base_index.index < tuple_type.types.len);
                                },
                                .struct_type => {
                                    assert(ptr.base_addr == .field);
                                    assert(base_index.index < ip.loadStructType(base_ptr_type.child).field_types.len);
                                },
                                .union_type => {
                                    const union_type = ip.loadUnionType(base_ptr_type.child);
                                    assert(ptr.base_addr == .field);
                                    assert(base_index.index < union_type.field_types.len);
                                },
                                .ptr_type => |slice_type| {
                                    assert(ptr.base_addr == .field);
                                    assert(slice_type.flags.size == .slice);
                                    assert(base_index.index < 2);
                                },
                                else => unreachable,
                            }
                        },
                        else => unreachable,
                    }
                    gop.cancel();
                    const index_index = try ip.get(gpa, tid, .{ .int = .{
                        .ty = .usize_type,
                        .storage = .{ .u64 = base_index.index },
                    } });
                    gop = try ip.getOrPutKey(gpa, tid, key);
                    try items.ensureUnusedCapacity(1);
                    items.appendAssumeCapacity(.{
                        .tag = switch (ptr.base_addr) {
                            .arr_elem => .ptr_elem,
                            .field => .ptr_field,
                            else => unreachable,
                        },
                        .data = try addExtra(extra, PtrBaseIndex.init(ptr.ty, base_index.base, index_index, ptr.byte_offset)),
                    });
                    return gop.put();
                },
            });
        },

        .opt => |opt| {
            assert(ip.isOptionalType(opt.ty));
            assert(opt.val == .none or ip.indexToKey(opt.ty).opt_type == ip.typeOf(opt.val));
            items.appendAssumeCapacity(if (opt.val == .none) .{
                .tag = .opt_null,
                .data = @intFromEnum(opt.ty),
            } else .{
                .tag = .opt_payload,
                .data = try addExtra(extra, Tag.TypeValue{
                    .ty = opt.ty,
                    .val = opt.val,
                }),
            });
        },

        .int => |int| b: {
            assert(ip.isIntegerType(int.ty));
            switch (int.storage) {
                .u64, .i64, .big_int => {},
                .lazy_align, .lazy_size => |lazy_ty| {
                    items.appendAssumeCapacity(.{
                        .tag = switch (int.storage) {
                            else => unreachable,
                            .lazy_align => .int_lazy_align,
                            .lazy_size => .int_lazy_size,
                        },
                        .data = try addExtra(extra, IntLazy{
                            .ty = int.ty,
                            .lazy_ty = lazy_ty,
                        }),
                    });
                    return gop.put();
                },
            }
            switch (int.ty) {
                .u8_type => switch (int.storage) {
                    .big_int => |big_int| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_u8,
                            .data = big_int.toInt(u8) catch unreachable,
                        });
                        break :b;
                    },
                    inline .u64, .i64 => |x| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_u8,
                            .data = @as(u8, @intCast(x)),
                        });
                        break :b;
                    },
                    .lazy_align, .lazy_size => unreachable,
                },
                .u16_type => switch (int.storage) {
                    .big_int => |big_int| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_u16,
                            .data = big_int.toInt(u16) catch unreachable,
                        });
                        break :b;
                    },
                    inline .u64, .i64 => |x| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_u16,
                            .data = @as(u16, @intCast(x)),
                        });
                        break :b;
                    },
                    .lazy_align, .lazy_size => unreachable,
                },
                .u32_type => switch (int.storage) {
                    .big_int => |big_int| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_u32,
                            .data = big_int.toInt(u32) catch unreachable,
                        });
                        break :b;
                    },
                    inline .u64, .i64 => |x| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_u32,
                            .data = @as(u32, @intCast(x)),
                        });
                        break :b;
                    },
                    .lazy_align, .lazy_size => unreachable,
                },
                .i32_type => switch (int.storage) {
                    .big_int => |big_int| {
                        const casted = big_int.toInt(i32) catch unreachable;
                        items.appendAssumeCapacity(.{
                            .tag = .int_i32,
                            .data = @as(u32, @bitCast(casted)),
                        });
                        break :b;
                    },
                    inline .u64, .i64 => |x| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_i32,
                            .data = @as(u32, @bitCast(@as(i32, @intCast(x)))),
                        });
                        break :b;
                    },
                    .lazy_align, .lazy_size => unreachable,
                },
                .usize_type => switch (int.storage) {
                    .big_int => |big_int| {
                        if (big_int.toInt(u32)) |casted| {
                            items.appendAssumeCapacity(.{
                                .tag = .int_usize,
                                .data = casted,
                            });
                            break :b;
                        } else |_| {}
                    },
                    inline .u64, .i64 => |x| {
                        if (std.math.cast(u32, x)) |casted| {
                            items.appendAssumeCapacity(.{
                                .tag = .int_usize,
                                .data = casted,
                            });
                            break :b;
                        }
                    },
                    .lazy_align, .lazy_size => unreachable,
                },
                .comptime_int_type => switch (int.storage) {
                    .big_int => |big_int| {
                        if (big_int.toInt(u32)) |casted| {
                            items.appendAssumeCapacity(.{
                                .tag = .int_comptime_int_u32,
                                .data = casted,
                            });
                            break :b;
                        } else |_| {}
                        if (big_int.toInt(i32)) |casted| {
                            items.appendAssumeCapacity(.{
                                .tag = .int_comptime_int_i32,
                                .data = @as(u32, @bitCast(casted)),
                            });
                            break :b;
                        } else |_| {}
                    },
                    inline .u64, .i64 => |x| {
                        if (std.math.cast(u32, x)) |casted| {
                            items.appendAssumeCapacity(.{
                                .tag = .int_comptime_int_u32,
                                .data = casted,
                            });
                            break :b;
                        }
                        if (std.math.cast(i32, x)) |casted| {
                            items.appendAssumeCapacity(.{
                                .tag = .int_comptime_int_i32,
                                .data = @as(u32, @bitCast(casted)),
                            });
                            break :b;
                        }
                    },
                    .lazy_align, .lazy_size => unreachable,
                },
                else => {},
            }
            switch (int.storage) {
                .big_int => |big_int| {
                    if (big_int.toInt(u32)) |casted| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_small,
                            .data = try addExtra(extra, IntSmall{
                                .ty = int.ty,
                                .value = casted,
                            }),
                        });
                        return gop.put();
                    } else |_| {}

                    const tag: Tag = if (big_int.positive) .int_positive else .int_negative;
                    try addInt(ip, gpa, tid, int.ty, tag, big_int.limbs);
                },
                inline .u64, .i64 => |x| {
                    if (std.math.cast(u32, x)) |casted| {
                        items.appendAssumeCapacity(.{
                            .tag = .int_small,
                            .data = try addExtra(extra, IntSmall{
                                .ty = int.ty,
                                .value = casted,
                            }),
                        });
                        return gop.put();
                    }

                    var buf: [2]Limb = undefined;
                    const big_int = BigIntMutable.init(&buf, x).toConst();
                    const tag: Tag = if (big_int.positive) .int_positive else .int_negative;
                    try addInt(ip, gpa, tid, int.ty, tag, big_int.limbs);
                },
                .lazy_align, .lazy_size => unreachable,
            }
        },

        .err => |err| {
            assert(ip.isErrorSetType(err.ty));
            items.appendAssumeCapacity(.{
                .tag = .error_set_error,
                .data = try addExtra(extra, err),
            });
        },

        .error_union => |error_union| {
            assert(ip.isErrorUnionType(error_union.ty));
            items.appendAssumeCapacity(switch (error_union.val) {
                .err_name => |err_name| .{
                    .tag = .error_union_error,
                    .data = try addExtra(extra, Key.Error{
                        .ty = error_union.ty,
                        .name = err_name,
                    }),
                },
                .payload => |payload| .{
                    .tag = .error_union_payload,
                    .data = try addExtra(extra, Tag.TypeValue{
                        .ty = error_union.ty,
                        .val = payload,
                    }),
                },
            });
        },

        .enum_literal => |enum_literal| items.appendAssumeCapacity(.{
            .tag = .enum_literal,
            .data = @intFromEnum(enum_literal),
        }),

        .enum_tag => |enum_tag| {
            assert(ip.isEnumType(enum_tag.ty));
            switch (ip.indexToKey(enum_tag.ty)) {
                .simple_type => assert(ip.isIntegerType(ip.typeOf(enum_tag.int))),
                .enum_type => assert(ip.typeOf(enum_tag.int) == ip.loadEnumType(enum_tag.ty).tag_ty),
                else => unreachable,
            }
            items.appendAssumeCapacity(.{
                .tag = .enum_tag,
                .data = try addExtra(extra, enum_tag),
            });
        },

        .empty_enum_value => |enum_or_union_ty| items.appendAssumeCapacity(.{
            .tag = .only_possible_value,
            .data = @intFromEnum(enum_or_union_ty),
        }),

        .float => |float| {
            switch (float.ty) {
                .f16_type => items.appendAssumeCapacity(.{
                    .tag = .float_f16,
                    .data = @as(u16, @bitCast(float.storage.f16)),
                }),
                .f32_type => items.appendAssumeCapacity(.{
                    .tag = .float_f32,
                    .data = @as(u32, @bitCast(float.storage.f32)),
                }),
                .f64_type => items.appendAssumeCapacity(.{
                    .tag = .float_f64,
                    .data = try addExtra(extra, Float64.pack(float.storage.f64)),
                }),
                .f80_type => items.appendAssumeCapacity(.{
                    .tag = .float_f80,
                    .data = try addExtra(extra, Float80.pack(float.storage.f80)),
                }),
                .f128_type => items.appendAssumeCapacity(.{
                    .tag = .float_f128,
                    .data = try addExtra(extra, Float128.pack(float.storage.f128)),
                }),
                .c_longdouble_type => switch (float.storage) {
                    .f80 => |x| items.appendAssumeCapacity(.{
                        .tag = .float_c_longdouble_f80,
                        .data = try addExtra(extra, Float80.pack(x)),
                    }),
                    inline .f16, .f32, .f64, .f128 => |x| items.appendAssumeCapacity(.{
                        .tag = .float_c_longdouble_f128,
                        .data = try addExtra(extra, Float128.pack(x)),
                    }),
                },
                .comptime_float_type => items.appendAssumeCapacity(.{
                    .tag = .float_comptime_float,
                    .data = try addExtra(extra, Float128.pack(float.storage.f128)),
                }),
                else => unreachable,
            }
        },

        .aggregate => |aggregate| {
            const ty_key = ip.indexToKey(aggregate.ty);
            const len = ip.aggregateTypeLen(aggregate.ty);
            const child = switch (ty_key) {
                .array_type => |array_type| array_type.child,
                .vector_type => |vector_type| vector_type.child,
                .tuple_type, .struct_type => .none,
                else => unreachable,
            };
            const sentinel = switch (ty_key) {
                .array_type => |array_type| array_type.sentinel,
                .vector_type, .tuple_type, .struct_type => .none,
                else => unreachable,
            };
            const len_including_sentinel = len + @intFromBool(sentinel != .none);
            switch (aggregate.storage) {
                .bytes => |bytes| {
                    assert(child == .u8_type);
                    if (sentinel != .none) {
                        assert(bytes.at(@intCast(len), ip) == ip.indexToKey(sentinel).int.storage.u64);
                    }
                },
                .elems => |elems| {
                    if (elems.len != len) {
                        assert(elems.len == len_including_sentinel);
                        assert(elems[@intCast(len)] == sentinel);
                    }
                },
                .repeated_elem => |elem| {
                    assert(sentinel == .none or elem == sentinel);
                },
            }
            switch (ty_key) {
                .array_type, .vector_type => {
                    for (aggregate.storage.values()) |elem| {
                        assert(ip.typeOf(elem) == child);
                    }
                },
                .struct_type => {
                    for (aggregate.storage.values(), ip.loadStructType(aggregate.ty).field_types.get(ip)) |elem, field_ty| {
                        assert(ip.typeOf(elem) == field_ty);
                    }
                },
                .tuple_type => |tuple_type| {
                    for (aggregate.storage.values(), tuple_type.types.get(ip)) |elem, ty| {
                        assert(ip.typeOf(elem) == ty);
                    }
                },
                else => unreachable,
            }

            if (len == 0) {
                items.appendAssumeCapacity(.{
                    .tag = .only_possible_value,
                    .data = @intFromEnum(aggregate.ty),
                });
                return gop.put();
            }

            switch (ty_key) {
                .tuple_type => |tuple_type| opv: {
                    switch (aggregate.storage) {
                        .bytes => |bytes| for (tuple_type.values.get(ip), bytes.at(0, ip)..) |value, byte| {
                            if (value == .none) break :opv;
                            switch (ip.indexToKey(value)) {
                                .undef => break :opv,
                                .int => |int| switch (int.storage) {
                                    .u64 => |x| if (x != byte) break :opv,
                                    else => break :opv,
                                },
                                else => unreachable,
                            }
                        },
                        .elems => |elems| if (!std.mem.eql(
                            Index,
                            tuple_type.values.get(ip),
                            elems,
                        )) break :opv,
                        .repeated_elem => |elem| for (tuple_type.values.get(ip)) |value| {
                            if (value != elem) break :opv;
                        },
                    }
                    // This encoding works thanks to the fact that, as we just verified,
                    // the type itself contains a slice of values that can be provided
                    // in the aggregate fields.
                    items.appendAssumeCapacity(.{
                        .tag = .only_possible_value,
                        .data = @intFromEnum(aggregate.ty),
                    });
                    return gop.put();
                },
                else => {},
            }

            repeated: {
                switch (aggregate.storage) {
                    .bytes => |bytes| for (bytes.toSlice(len, ip)[1..]) |byte|
                        if (byte != bytes.at(0, ip)) break :repeated,
                    .elems => |elems| for (elems[1..@intCast(len)]) |elem|
                        if (elem != elems[0]) break :repeated,
                    .repeated_elem => {},
                }
                const elem = switch (aggregate.storage) {
                    .bytes => |bytes| elem: {
                        gop.cancel();
                        const elem = try ip.get(gpa, tid, .{ .int = .{
                            .ty = .u8_type,
                            .storage = .{ .u64 = bytes.at(0, ip) },
                        } });
                        gop = try ip.getOrPutKey(gpa, tid, key);
                        try items.ensureUnusedCapacity(1);
                        break :elem elem;
                    },
                    .elems => |elems| elems[0],
                    .repeated_elem => |elem| elem,
                };

                try extra.ensureUnusedCapacity(@typeInfo(Repeated).@"struct".fields.len);
                items.appendAssumeCapacity(.{
                    .tag = .repeated,
                    .data = addExtraAssumeCapacity(extra, Repeated{
                        .ty = aggregate.ty,
                        .elem_val = elem,
                    }),
                });
                return gop.put();
            }

            if (child == .u8_type) bytes: {
                const strings = ip.getLocal(tid).getMutableStrings(gpa);
                const start = strings.mutate.len;
                try strings.ensureUnusedCapacity(@intCast(len_including_sentinel + 1));
                try extra.ensureUnusedCapacity(@typeInfo(Bytes).@"struct".fields.len);
                switch (aggregate.storage) {
                    .bytes => |bytes| strings.appendSliceAssumeCapacity(.{bytes.toSlice(len, ip)}),
                    .elems => |elems| for (elems[0..@intCast(len)]) |elem| switch (ip.indexToKey(elem)) {
                        .undef => {
                            strings.shrinkRetainingCapacity(start);
                            break :bytes;
                        },
                        .int => |int| strings.appendAssumeCapacity(.{@intCast(int.storage.u64)}),
                        else => unreachable,
                    },
                    .repeated_elem => |elem| switch (ip.indexToKey(elem)) {
                        .undef => break :bytes,
                        .int => |int| @memset(
                            strings.addManyAsSliceAssumeCapacity(@intCast(len))[0],
                            @intCast(int.storage.u64),
                        ),
                        else => unreachable,
                    },
                }
                if (sentinel != .none) strings.appendAssumeCapacity(.{
                    @intCast(ip.indexToKey(sentinel).int.storage.u64),
                });
                const string = try ip.getOrPutTrailingString(
                    gpa,
                    tid,
                    @intCast(len_including_sentinel),
                    .maybe_embedded_nulls,
                );
                items.appendAssumeCapacity(.{
                    .tag = .bytes,
                    .data = addExtraAssumeCapacity(extra, Bytes{
                        .ty = aggregate.ty,
                        .bytes = string,
                    }),
                });
                return gop.put();
            }

            try extra.ensureUnusedCapacity(
                @typeInfo(Tag.Aggregate).@"struct".fields.len + @as(usize, @intCast(len_including_sentinel + 1)),
            );
            items.appendAssumeCapacity(.{
                .tag = .aggregate,
                .data = addExtraAssumeCapacity(extra, Tag.Aggregate{
                    .ty = aggregate.ty,
                }),
            });
            extra.appendSliceAssumeCapacity(.{@ptrCast(aggregate.storage.elems)});
            if (sentinel != .none) extra.appendAssumeCapacity(.{@intFromEnum(sentinel)});
        },

        .memoized_call => |memoized_call| {
            for (memoized_call.arg_values) |arg| assert(arg != .none);
            try extra.ensureUnusedCapacity(@typeInfo(MemoizedCall).@"struct".fields.len +
                memoized_call.arg_values.len);
            items.appendAssumeCapacity(.{
                .tag = .memoized_call,
                .data = addExtraAssumeCapacity(extra, MemoizedCall{
                    .func = memoized_call.func,
                    .args_len = @intCast(memoized_call.arg_values.len),
                    .result = memoized_call.result,
                    .branch_count = memoized_call.branch_count,
                }),
            });
            extra.appendSliceAssumeCapacity(.{@ptrCast(memoized_call.arg_values)});
        },
    }
    return gop.put();
}
```
嗯，我现在需要理解这段Zig代码。首先，这个函数叫做`get`，属于`InternPool`模块。看起来它的作用是根据给定的`key`返回一个索引，可能用于内部池化（interning）不同类型的值，以减少重复存储。

函数参数有四个：`ip`是`InternPool`的指针，`gpa`是分配器，`tid`是线程ID，`key`是要查找或创建的关键字。返回类型是一个`Index`或者错误。

首先，函数里调用了`ip.getOrPutKey`，得到一个`gop`（可能是GetOrPut的结果）。然后用`defer`确保`gop.deinit()`会被调用，不管后面怎么返回。如果`gop`是`.existing`，直接返回已有的索引，否则继续处理。

接下来，根据`key`的不同类型，进行不同的处理。比如，如果`key`是`.int_type`，会根据符号性（signedness）设置不同的`Tag`，然后添加到`items`里。如果是`.ptr_type`，则处理指针类型，可能调整指针的大小，比如将slice转换为many指针，并递归调用`ip.get`来获取调整后的指针类型索引。

对于其他类型，如数组、向量、可选类型等，都有相应的处理逻辑。比如数组类型需要检查长度和哨兵值（sentinel），结构体类型则不允许直接使用，必须通过特定函数如`getStructType`。

在处理整数类型时，会根据整数的大小和符号选择不同的标签，如`.int_u8`、`.int_i32`等，并处理大整数的情况，可能将其分解为多个limbs存储。浮点数类型则根据精度存储为不同的格式。

聚合类型（如数组、结构体、元组）需要检查元素的类型和数量，确保符合预期。对于重复的元素，会优化存储为`.repeated`类型，减少冗余。如果是字节数组，可能直接存储为字符串。

还有一些特殊情况的处理，比如`memoized_call`，需要存储函数参数和结果，用于缓存调用结果，避免重复计算。

在代码中，很多地方用了`assert`来确保前置条件，例如指针类型是否正确，元素是否存在等。这有助于在开发阶段捕获错误。

不过，有些地方不太明白。比如，在`.ptr_type`处理中，当指针的大小是`.slice`时，会创建一个新的`new_key`，将大小改为`.many`，然后递归调用`ip.get`获取`ptr_type_index`，之后再重新获取`gop`。为什么要这样做？可能是为了统一处理slice指针为many指针，避免重复代码？

另外，在整数处理中，如果无法将大整数转换为较小的类型，就调用`addInt`函数。但`addInt`函数没有在这段代码中定义，可能在别处实现。需要确认它的作用是否是处理大整数的存储。

总的来说，这个函数的主要逻辑是根据不同的`key`类型，构造相应的数据结构，并将它们添加到内部池中，返回对应的索引。这样可以确保相同的值只存储一次，提高内存效率。
================================================
这段代码实现了一个内部池（InternPool）的`get`函数，用于根据不同的类型和值生成唯一的索引（`Index`），避免重复存储。其核心思维模型如下：

---

### **核心逻辑**
1. **键值查找与创建**  
   通过 `getOrPutKey` 尝试从池中获取现有值；若存在（`.existing`），直接返回索引；否则进入创建流程。
2. **类型分派与数据构造**  
   根据 `key` 的类型（如整数、指针、数组、错误联合等），分派到不同的处理逻辑，构造对应的数据结构（如 `Tag` 和附加数据 `extra`），并确保内存容量。
3. **递归处理与优化**  
   - **递归调用**：某些类型（如指针调整、重复元素）需要递归调用 `get` 或调整 `key` 后再处理。
   - **存储优化**：对重复值（如全相同元素的数组）、小整数、固定浮点数等，选择紧凑的存储方式（如 `.repeated`、`.int_u8`）。
4. **断言与校验**  
   使用 `assert` 确保类型和值的合法性（如指针子类型、数组长度、哨兵值等），避免无效状态。
5. **内存管理**  
   通过 `gpa` 分配器动态管理内存，`defer` 确保资源释放，`ensureUnusedCapacity` 预分配空间提升性能。

---

### **关键处理逻辑**
- **整数类型**  
  根据符号性和位宽选择标签（如 `.type_int_signed`），处理大整数时分解为 `limbs`（通过 `addInt`）。
- **指针类型**  
  - **Slice 指针**：转换为 `many` 指针后递归处理。
  - **对齐与标志**：调整指针属性（如 `is_allowzero`），存储为 `.type_pointer`。
- **数组与向量**  
  根据长度和哨兵值选择 `.type_array_small`（小长度）或 `.type_array_big`（大长度）。
- **错误联合与枚举**  
  区分错误和有效负载（`.error_union_error` vs. `.error_union_payload`），确保错误集名称有序。
- **聚合类型**  
  - **重复值优化**：全相同元素存储为 `.repeated`。
  - **字节数组**：直接存储为字符串（`.bytes`）。
- **缓存调用**  
  `memoized_call` 缓存函数参数和结果，避免重复计算。

---

### **特殊机制**
- **递归键调整**  
  例如将 `slice` 指针转为 `many` 指针后重新生成 `key`，确保类型统一。
- **延迟计算**  
  对某些值（如 `lazy_align`）延迟到运行时计算。
- **线程本地存储**  
  通过 `tid` 区分线程本地数据，避免竞争。

---

### **总结**
该函数通过类型分派和递归处理，将复杂类型和值统一编码为内部池的索引，兼顾内存效率和类型安全。其设计核心是 **类型驱动** 和 **递归归一化**，通过预校验、内存预分配和存储优化，确保高效且正确的池化操作。