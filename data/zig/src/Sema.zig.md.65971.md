```zig
fn zirTypeInfo(sema: *Sema, block: *Block, inst: Zir.Inst.Index) CompileError!Air.Inst.Ref {
    const pt = sema.pt;
    const zcu = pt.zcu;
    const gpa = sema.gpa;
    const ip = &zcu.intern_pool;
    const inst_data = sema.code.instructions.items(.data)[@intFromEnum(inst)].un_node;
    const src = block.nodeOffset(inst_data.src_node);
    const ty = try sema.resolveType(block, src, inst_data.operand);
    const type_info_ty = try sema.getBuiltinType(src, .Type);
    const type_info_tag_ty = type_info_ty.unionTagType(zcu).?;

    if (ty.typeDeclInst(zcu)) |type_decl_inst| {
        try sema.declareDependency(.{ .namespace = type_decl_inst });
    }

    switch (ty.zigTypeTag(zcu)) {
        .type,
        .void,
        .bool,
        .noreturn,
        .comptime_float,
        .comptime_int,
        .undefined,
        .null,
        .enum_literal,
        => |type_info_tag| return unionInitFromEnumTag(sema, block, src, type_info_ty, @intFromEnum(type_info_tag), .void_value),

        .@"fn" => {
            const fn_info_ty = try sema.getBuiltinType(src, .@"Type.Fn");
            const param_info_ty = try sema.getBuiltinType(src, .@"Type.Fn.Param");

            const func_ty_info = zcu.typeToFunc(ty).?;
            const param_vals = try sema.arena.alloc(InternPool.Index, func_ty_info.param_types.len);
            for (param_vals, 0..) |*param_val, i| {
                const param_ty = func_ty_info.param_types.get(ip)[i];
                const is_generic = param_ty == .generic_poison_type;
                const param_ty_val = try pt.intern(.{ .opt = .{
                    .ty = try pt.intern(.{ .opt_type = .type_type }),
                    .val = if (is_generic) .none else param_ty,
                } });

                const is_noalias = blk: {
                    const index = std.math.cast(u5, i) orelse break :blk false;
                    break :blk @as(u1, @truncate(func_ty_info.noalias_bits >> index)) != 0;
                };

                const param_fields = .{
                    // is_generic: bool,
                    Value.makeBool(is_generic).toIntern(),
                    // is_noalias: bool,
                    Value.makeBool(is_noalias).toIntern(),
                    // type: ?type,
                    param_ty_val,
                };
                param_val.* = try pt.intern(.{ .aggregate = .{
                    .ty = param_info_ty.toIntern(),
                    .storage = .{ .elems = &param_fields },
                } });
            }

            const args_val = v: {
                const new_decl_ty = try pt.arrayType(.{
                    .len = param_vals.len,
                    .child = param_info_ty.toIntern(),
                });
                const new_decl_val = try pt.intern(.{ .aggregate = .{
                    .ty = new_decl_ty.toIntern(),
                    .storage = .{ .elems = param_vals },
                } });
                const slice_ty = (try pt.ptrTypeSema(.{
                    .child = param_info_ty.toIntern(),
                    .flags = .{
                        .size = .slice,
                        .is_const = true,
                    },
                })).toIntern();
                const manyptr_ty = Type.fromInterned(slice_ty).slicePtrFieldType(zcu).toIntern();
                break :v try pt.intern(.{ .slice = .{
                    .ty = slice_ty,
                    .ptr = try pt.intern(.{ .ptr = .{
                        .ty = manyptr_ty,
                        .base_addr = .{ .uav = .{
                            .orig_ty = manyptr_ty,
                            .val = new_decl_val,
                        } },
                        .byte_offset = 0,
                    } }),
                    .len = (try pt.intValue(Type.usize, param_vals.len)).toIntern(),
                } });
            };

            const ret_ty_opt = try pt.intern(.{ .opt = .{
                .ty = try pt.intern(.{ .opt_type = .type_type }),
                .val = opt_val: {
                    const ret_ty: Type = .fromInterned(func_ty_info.return_type);
                    if (ret_ty.toIntern() == .generic_poison_type) break :opt_val .none;
                    if (ret_ty.zigTypeTag(zcu) == .error_union) {
                        if (ret_ty.errorUnionPayload(zcu).toIntern() == .generic_poison_type) {
                            break :opt_val .none;
                        }
                    }
                    break :opt_val ret_ty.toIntern();
                },
            } });

            const callconv_ty = try sema.getBuiltinType(src, .CallingConvention);
            const callconv_val = Value.uninterpret(func_ty_info.cc, callconv_ty, pt) catch |err| switch (err) {
                error.TypeMismatch => @panic("std.builtin is corrupt"),
                error.OutOfMemory => |e| return e,
            };

            const field_values: [5]InternPool.Index = .{
                // calling_convention: CallingConvention,
                callconv_val.toIntern(),
                // is_generic: bool,
                Value.makeBool(func_ty_info.is_generic).toIntern(),
                // is_var_args: bool,
                Value.makeBool(func_ty_info.is_var_args).toIntern(),
                // return_type: ?type,
                ret_ty_opt,
                // args: []const Fn.Param,
                args_val,
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.@"fn"))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = fn_info_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .int => {
            const int_info_ty = try sema.getBuiltinType(src, .@"Type.Int");
            const signedness_ty = try sema.getBuiltinType(src, .Signedness);
            const info = ty.intInfo(zcu);
            const field_values = .{
                // signedness: Signedness,
                (try pt.enumValueFieldIndex(signedness_ty, @intFromEnum(info.signedness))).toIntern(),
                // bits: u16,
                (try pt.intValue(Type.u16, info.bits)).toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.int))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = int_info_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .float => {
            const float_info_ty = try sema.getBuiltinType(src, .@"Type.Float");

            const field_vals = .{
                // bits: u16,
                (try pt.intValue(Type.u16, ty.bitSize(zcu))).toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.float))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = float_info_ty.toIntern(),
                    .storage = .{ .elems = &field_vals },
                } }),
            })));
        },
        .pointer => {
            const info = ty.ptrInfo(zcu);
            const alignment = if (info.flags.alignment.toByteUnits()) |alignment|
                try pt.intValue(Type.comptime_int, alignment)
            else
                try Type.fromInterned(info.child).lazyAbiAlignment(pt);

            const addrspace_ty = try sema.getBuiltinType(src, .AddressSpace);
            const pointer_ty = try sema.getBuiltinType(src, .@"Type.Pointer");
            const ptr_size_ty = try sema.getBuiltinType(src, .@"Type.Pointer.Size");

            const field_values = .{
                // size: Size,
                (try pt.enumValueFieldIndex(ptr_size_ty, @intFromEnum(info.flags.size))).toIntern(),
                // is_const: bool,
                Value.makeBool(info.flags.is_const).toIntern(),
                // is_volatile: bool,
                Value.makeBool(info.flags.is_volatile).toIntern(),
                // alignment: comptime_int,
                alignment.toIntern(),
                // address_space: AddressSpace
                (try pt.enumValueFieldIndex(addrspace_ty, @intFromEnum(info.flags.address_space))).toIntern(),
                // child: type,
                info.child,
                // is_allowzero: bool,
                Value.makeBool(info.flags.is_allowzero).toIntern(),
                // sentinel: ?*const anyopaque,
                (try sema.optRefValue(switch (info.sentinel) {
                    .none => null,
                    else => Value.fromInterned(info.sentinel),
                })).toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.pointer))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = pointer_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .array => {
            const array_field_ty = try sema.getBuiltinType(src, .@"Type.Array");

            const info = ty.arrayInfo(zcu);
            const field_values = .{
                // len: comptime_int,
                (try pt.intValue(Type.comptime_int, info.len)).toIntern(),
                // child: type,
                info.elem_type.toIntern(),
                // sentinel: ?*const anyopaque,
                (try sema.optRefValue(info.sentinel)).toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.array))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = array_field_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .vector => {
            const vector_field_ty = try sema.getBuiltinType(src, .@"Type.Vector");

            const info = ty.arrayInfo(zcu);
            const field_values = .{
                // len: comptime_int,
                (try pt.intValue(Type.comptime_int, info.len)).toIntern(),
                // child: type,
                info.elem_type.toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.vector))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = vector_field_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .optional => {
            const optional_field_ty = try sema.getBuiltinType(src, .@"Type.Optional");

            const field_values = .{
                // child: type,
                ty.optionalChild(zcu).toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.optional))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = optional_field_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .error_set => {
            // Get the Error type
            const error_field_ty = try sema.getBuiltinType(src, .@"Type.Error");

            // Build our list of Error values
            // Optional value is only null if anyerror
            // Value can be zero-length slice otherwise
            const error_field_vals = switch (try sema.resolveInferredErrorSetTy(block, src, ty.toIntern())) {
                .anyerror_type => null,
                else => |err_set_ty_index| blk: {
                    const names = ip.indexToKey(err_set_ty_index).error_set_type.names;
                    const vals = try sema.arena.alloc(InternPool.Index, names.len);
                    for (vals, 0..) |*field_val, error_index| {
                        const error_name = names.get(ip)[error_index];
                        const error_name_len = error_name.length(ip);
                        const error_name_val = v: {
                            const new_decl_ty = try pt.arrayType(.{
                                .len = error_name_len,
                                .sentinel = .zero_u8,
                                .child = .u8_type,
                            });
                            const new_decl_val = try pt.intern(.{ .aggregate = .{
                                .ty = new_decl_ty.toIntern(),
                                .storage = .{ .bytes = error_name.toString() },
                            } });
                            break :v try pt.intern(.{ .slice = .{
                                .ty = .slice_const_u8_sentinel_0_type,
                                .ptr = try pt.intern(.{ .ptr = .{
                                    .ty = .manyptr_const_u8_sentinel_0_type,
                                    .base_addr = .{ .uav = .{
                                        .val = new_decl_val,
                                        .orig_ty = .slice_const_u8_sentinel_0_type,
                                    } },
                                    .byte_offset = 0,
                                } }),
                                .len = (try pt.intValue(Type.usize, error_name_len)).toIntern(),
                            } });
                        };

                        const error_field_fields = .{
                            // name: [:0]const u8,
                            error_name_val,
                        };
                        field_val.* = try pt.intern(.{ .aggregate = .{
                            .ty = error_field_ty.toIntern(),
                            .storage = .{ .elems = &error_field_fields },
                        } });
                    }

                    break :blk vals;
                },
            };

            // Build our ?[]const Error value
            const slice_errors_ty = try pt.ptrTypeSema(.{
                .child = error_field_ty.toIntern(),
                .flags = .{
                    .size = .slice,
                    .is_const = true,
                },
            });
            const opt_slice_errors_ty = try pt.optionalType(slice_errors_ty.toIntern());
            const errors_payload_val: InternPool.Index = if (error_field_vals) |vals| v: {
                const array_errors_ty = try pt.arrayType(.{
                    .len = vals.len,
                    .child = error_field_ty.toIntern(),
                });
                const new_decl_val = try pt.intern(.{ .aggregate = .{
                    .ty = array_errors_ty.toIntern(),
                    .storage = .{ .elems = vals },
                } });
                const manyptr_errors_ty = slice_errors_ty.slicePtrFieldType(zcu).toIntern();
                break :v try pt.intern(.{ .slice = .{
                    .ty = slice_errors_ty.toIntern(),
                    .ptr = try pt.intern(.{ .ptr = .{
                        .ty = manyptr_errors_ty,
                        .base_addr = .{ .uav = .{
                            .orig_ty = manyptr_errors_ty,
                            .val = new_decl_val,
                        } },
                        .byte_offset = 0,
                    } }),
                    .len = (try pt.intValue(Type.usize, vals.len)).toIntern(),
                } });
            } else .none;
            const errors_val = try pt.intern(.{ .opt = .{
                .ty = opt_slice_errors_ty.toIntern(),
                .val = errors_payload_val,
            } });

            // Construct Type{ .error_set = errors_val }
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.error_set))).toIntern(),
                .val = errors_val,
            })));
        },
        .error_union => {
            const error_union_field_ty = try sema.getBuiltinType(src, .@"Type.ErrorUnion");

            const field_values = .{
                // error_set: type,
                ty.errorUnionSet(zcu).toIntern(),
                // payload: type,
                ty.errorUnionPayload(zcu).toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.error_union))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = error_union_field_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .@"enum" => {
            const is_exhaustive = Value.makeBool(ip.loadEnumType(ty.toIntern()).tag_mode != .nonexhaustive);

            const enum_field_ty = try sema.getBuiltinType(src, .@"Type.EnumField");

            const enum_field_vals = try sema.arena.alloc(InternPool.Index, ip.loadEnumType(ty.toIntern()).names.len);
            for (enum_field_vals, 0..) |*field_val, tag_index| {
                const enum_type = ip.loadEnumType(ty.toIntern());
                const value_val = if (enum_type.values.len > 0)
                    try ip.getCoercedInts(
                        zcu.gpa,
                        pt.tid,
                        ip.indexToKey(enum_type.values.get(ip)[tag_index]).int,
                        .comptime_int_type,
                    )
                else
                    (try pt.intValue(Type.comptime_int, tag_index)).toIntern();

                // TODO: write something like getCoercedInts to avoid needing to dupe
                const name_val = v: {
                    const tag_name = enum_type.names.get(ip)[tag_index];
                    const tag_name_len = tag_name.length(ip);
                    const new_decl_ty = try pt.arrayType(.{
                        .len = tag_name_len,
                        .sentinel = .zero_u8,
                        .child = .u8_type,
                    });
                    const new_decl_val = try pt.intern(.{ .aggregate = .{
                        .ty = new_decl_ty.toIntern(),
                        .storage = .{ .bytes = tag_name.toString() },
                    } });
                    break :v try pt.intern(.{ .slice = .{
                        .ty = .slice_const_u8_sentinel_0_type,
                        .ptr = try pt.intern(.{ .ptr = .{
                            .ty = .manyptr_const_u8_sentinel_0_type,
                            .base_addr = .{ .uav = .{
                                .val = new_decl_val,
                                .orig_ty = .slice_const_u8_sentinel_0_type,
                            } },
                            .byte_offset = 0,
                        } }),
                        .len = (try pt.intValue(Type.usize, tag_name_len)).toIntern(),
                    } });
                };

                const enum_field_fields = .{
                    // name: [:0]const u8,
                    name_val,
                    // value: comptime_int,
                    value_val,
                };
                field_val.* = try pt.intern(.{ .aggregate = .{
                    .ty = enum_field_ty.toIntern(),
                    .storage = .{ .elems = &enum_field_fields },
                } });
            }

            const fields_val = v: {
                const fields_array_ty = try pt.arrayType(.{
                    .len = enum_field_vals.len,
                    .child = enum_field_ty.toIntern(),
                });
                const new_decl_val = try pt.intern(.{ .aggregate = .{
                    .ty = fields_array_ty.toIntern(),
                    .storage = .{ .elems = enum_field_vals },
                } });
                const slice_ty = (try pt.ptrTypeSema(.{
                    .child = enum_field_ty.toIntern(),
                    .flags = .{
                        .size = .slice,
                        .is_const = true,
                    },
                })).toIntern();
                const manyptr_ty = Type.fromInterned(slice_ty).slicePtrFieldType(zcu).toIntern();
                break :v try pt.intern(.{ .slice = .{
                    .ty = slice_ty,
                    .ptr = try pt.intern(.{ .ptr = .{
                        .ty = manyptr_ty,
                        .base_addr = .{ .uav = .{
                            .val = new_decl_val,
                            .orig_ty = manyptr_ty,
                        } },
                        .byte_offset = 0,
                    } }),
                    .len = (try pt.intValue(Type.usize, enum_field_vals.len)).toIntern(),
                } });
            };

            const decls_val = try sema.typeInfoDecls(block, src, ip.loadEnumType(ty.toIntern()).namespace.toOptional());

            const type_enum_ty = try sema.getBuiltinType(src, .@"Type.Enum");

            const field_values = .{
                // tag_type: type,
                ip.loadEnumType(ty.toIntern()).tag_ty,
                // fields: []const EnumField,
                fields_val,
                // decls: []const Declaration,
                decls_val,
                // is_exhaustive: bool,
                is_exhaustive.toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.@"enum"))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = type_enum_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .@"union" => {
            const type_union_ty = try sema.getBuiltinType(src, .@"Type.Union");
            const union_field_ty = try sema.getBuiltinType(src, .@"Type.UnionField");

            try ty.resolveLayout(pt); // Getting alignment requires type layout
            const union_obj = zcu.typeToUnion(ty).?;
            const tag_type = union_obj.loadTagType(ip);
            const layout = union_obj.flagsUnordered(ip).layout;

            const union_field_vals = try gpa.alloc(InternPool.Index, tag_type.names.len);
            defer gpa.free(union_field_vals);

            for (union_field_vals, 0..) |*field_val, field_index| {
                const name_val = v: {
                    const field_name = tag_type.names.get(ip)[field_index];
                    const field_name_len = field_name.length(ip);
                    const new_decl_ty = try pt.arrayType(.{
                        .len = field_name_len,
                        .sentinel = .zero_u8,
                        .child = .u8_type,
                    });
                    const new_decl_val = try pt.intern(.{ .aggregate = .{
                        .ty = new_decl_ty.toIntern(),
                        .storage = .{ .bytes = field_name.toString() },
                    } });
                    break :v try pt.intern(.{ .slice = .{
                        .ty = .slice_const_u8_sentinel_0_type,
                        .ptr = try pt.intern(.{ .ptr = .{
                            .ty = .manyptr_const_u8_sentinel_0_type,
                            .base_addr = .{ .uav = .{
                                .val = new_decl_val,
                                .orig_ty = .slice_const_u8_sentinel_0_type,
                            } },
                            .byte_offset = 0,
                        } }),
                        .len = (try pt.intValue(Type.usize, field_name_len)).toIntern(),
                    } });
                };

                const alignment = switch (layout) {
                    .auto, .@"extern" => try ty.fieldAlignmentSema(field_index, pt),
                    .@"packed" => .none,
                };

                const field_ty = union_obj.field_types.get(ip)[field_index];
                const union_field_fields = .{
                    // name: [:0]const u8,
                    name_val,
                    // type: type,
                    field_ty,
                    // alignment: comptime_int,
                    (try pt.intValue(Type.comptime_int, alignment.toByteUnits() orelse 0)).toIntern(),
                };
                field_val.* = try pt.intern(.{ .aggregate = .{
                    .ty = union_field_ty.toIntern(),
                    .storage = .{ .elems = &union_field_fields },
                } });
            }

            const fields_val = v: {
                const array_fields_ty = try pt.arrayType(.{
                    .len = union_field_vals.len,
                    .child = union_field_ty.toIntern(),
                });
                const new_decl_val = try pt.intern(.{ .aggregate = .{
                    .ty = array_fields_ty.toIntern(),
                    .storage = .{ .elems = union_field_vals },
                } });
                const slice_ty = (try pt.ptrTypeSema(.{
                    .child = union_field_ty.toIntern(),
                    .flags = .{
                        .size = .slice,
                        .is_const = true,
                    },
                })).toIntern();
                const manyptr_ty = Type.fromInterned(slice_ty).slicePtrFieldType(zcu).toIntern();
                break :v try pt.intern(.{ .slice = .{
                    .ty = slice_ty,
                    .ptr = try pt.intern(.{ .ptr = .{
                        .ty = manyptr_ty,
                        .base_addr = .{ .uav = .{
                            .orig_ty = manyptr_ty,
                            .val = new_decl_val,
                        } },
                        .byte_offset = 0,
                    } }),
                    .len = (try pt.intValue(Type.usize, union_field_vals.len)).toIntern(),
                } });
            };

            const decls_val = try sema.typeInfoDecls(block, src, ty.getNamespaceIndex(zcu).toOptional());

            const enum_tag_ty_val = try pt.intern(.{ .opt = .{
                .ty = (try pt.optionalType(.type_type)).toIntern(),
                .val = if (ty.unionTagType(zcu)) |tag_ty| tag_ty.toIntern() else .none,
            } });

            const container_layout_ty = try sema.getBuiltinType(src, .@"Type.ContainerLayout");

            const field_values = .{
                // layout: ContainerLayout,
                (try pt.enumValueFieldIndex(container_layout_ty, @intFromEnum(layout))).toIntern(),

                // tag_type: ?type,
                enum_tag_ty_val,
                // fields: []const UnionField,
                fields_val,
                // decls: []const Declaration,
                decls_val,
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.@"union"))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = type_union_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .@"struct" => {
            const type_struct_ty = try sema.getBuiltinType(src, .@"Type.Struct");
            const struct_field_ty = try sema.getBuiltinType(src, .@"Type.StructField");

            try ty.resolveLayout(pt); // Getting alignment requires type layout

            var struct_field_vals: []InternPool.Index = &.{};
            defer gpa.free(struct_field_vals);
            fv: {
                const struct_type = switch (ip.indexToKey(ty.toIntern())) {
                    .tuple_type => |tuple_type| {
                        struct_field_vals = try gpa.alloc(InternPool.Index, tuple_type.types.len);
                        for (struct_field_vals, 0..) |*struct_field_val, field_index| {
                            const field_ty = tuple_type.types.get(ip)[field_index];
                            const field_val = tuple_type.values.get(ip)[field_index];
                            const name_val = v: {
                                const field_name = try ip.getOrPutStringFmt(gpa, pt.tid, "{d}", .{field_index}, .no_embedded_nulls);
                                const field_name_len = field_name.length(ip);
                                const new_decl_ty = try pt.arrayType(.{
                                    .len = field_name_len,
                                    .sentinel = .zero_u8,
                                    .child = .u8_type,
                                });
                                const new_decl_val = try pt.intern(.{ .aggregate = .{
                                    .ty = new_decl_ty.toIntern(),
                                    .storage = .{ .bytes = field_name.toString() },
                                } });
                                break :v try pt.intern(.{ .slice = .{
                                    .ty = .slice_const_u8_sentinel_0_type,
                                    .ptr = try pt.intern(.{ .ptr = .{
                                        .ty = .manyptr_const_u8_sentinel_0_type,
                                        .base_addr = .{ .uav = .{
                                            .val = new_decl_val,
                                            .orig_ty = .slice_const_u8_sentinel_0_type,
                                        } },
                                        .byte_offset = 0,
                                    } }),
                                    .len = (try pt.intValue(Type.usize, field_name_len)).toIntern(),
                                } });
                            };

                            try Type.fromInterned(field_ty).resolveLayout(pt);

                            const is_comptime = field_val != .none;
                            const opt_default_val = if (is_comptime) Value.fromInterned(field_val) else null;
                            const default_val_ptr = try sema.optRefValue(opt_default_val);
                            const struct_field_fields = .{
                                // name: [:0]const u8,
                                name_val,
                                // type: type,
                                field_ty,
                                // default_value: ?*const anyopaque,
                                default_val_ptr.toIntern(),
                                // is_comptime: bool,
                                Value.makeBool(is_comptime).toIntern(),
                                // alignment: comptime_int,
                                (try pt.intValue(Type.comptime_int, Type.fromInterned(field_ty).abiAlignment(zcu).toByteUnits() orelse 0)).toIntern(),
                            };
                            struct_field_val.* = try pt.intern(.{ .aggregate = .{
                                .ty = struct_field_ty.toIntern(),
                                .storage = .{ .elems = &struct_field_fields },
                            } });
                        }
                        break :fv;
                    },
                    .struct_type => ip.loadStructType(ty.toIntern()),
                    else => unreachable,
                };
                struct_field_vals = try gpa.alloc(InternPool.Index, struct_type.field_types.len);

                try ty.resolveStructFieldInits(pt);

                for (struct_field_vals, 0..) |*field_val, field_index| {
                    const field_name = if (struct_type.fieldName(ip, field_index).unwrap()) |field_name|
                        field_name
                    else
                        try ip.getOrPutStringFmt(gpa, pt.tid, "{d}", .{field_index}, .no_embedded_nulls);
                    const field_name_len = field_name.length(ip);
                    const field_ty = Type.fromInterned(struct_type.field_types.get(ip)[field_index]);
                    const field_init = struct_type.fieldInit(ip, field_index);
                    const field_is_comptime = struct_type.fieldIsComptime(ip, field_index);
                    const name_val = v: {
                        const new_decl_ty = try pt.arrayType(.{
                            .len = field_name_len,
                            .sentinel = .zero_u8,
                            .child = .u8_type,
                        });
                        const new_decl_val = try pt.intern(.{ .aggregate = .{
                            .ty = new_decl_ty.toIntern(),
                            .storage = .{ .bytes = field_name.toString() },
                        } });
                        break :v try pt.intern(.{ .slice = .{
                            .ty = .slice_const_u8_sentinel_0_type,
                            .ptr = try pt.intern(.{ .ptr = .{
                                .ty = .manyptr_const_u8_sentinel_0_type,
                                .base_addr = .{ .uav = .{
                                    .val = new_decl_val,
                                    .orig_ty = .slice_const_u8_sentinel_0_type,
                                } },
                                .byte_offset = 0,
                            } }),
                            .len = (try pt.intValue(Type.usize, field_name_len)).toIntern(),
                        } });
                    };

                    const opt_default_val = if (field_init == .none) null else Value.fromInterned(field_init);
                    const default_val_ptr = try sema.optRefValue(opt_default_val);
                    const alignment = switch (struct_type.layout) {
                        .@"packed" => .none,
                        else => try field_ty.structFieldAlignmentSema(
                            struct_type.fieldAlign(ip, field_index),
                            struct_type.layout,
                            pt,
                        ),
                    };

                    const struct_field_fields = .{
                        // name: [:0]const u8,
                        name_val,
                        // type: type,
                        field_ty.toIntern(),
                        // default_value: ?*const anyopaque,
                        default_val_ptr.toIntern(),
                        // is_comptime: bool,
                        Value.makeBool(field_is_comptime).toIntern(),
                        // alignment: comptime_int,
                        (try pt.intValue(Type.comptime_int, alignment.toByteUnits() orelse 0)).toIntern(),
                    };
                    field_val.* = try pt.intern(.{ .aggregate = .{
                        .ty = struct_field_ty.toIntern(),
                        .storage = .{ .elems = &struct_field_fields },
                    } });
                }
            }

            const fields_val = v: {
                const array_fields_ty = try pt.arrayType(.{
                    .len = struct_field_vals.len,
                    .child = struct_field_ty.toIntern(),
                });
                const new_decl_val = try pt.intern(.{ .aggregate = .{
                    .ty = array_fields_ty.toIntern(),
                    .storage = .{ .elems = struct_field_vals },
                } });
                const slice_ty = (try pt.ptrTypeSema(.{
                    .child = struct_field_ty.toIntern(),
                    .flags = .{
                        .size = .slice,
                        .is_const = true,
                    },
                })).toIntern();
                const manyptr_ty = Type.fromInterned(slice_ty).slicePtrFieldType(zcu).toIntern();
                break :v try pt.intern(.{ .slice = .{
                    .ty = slice_ty,
                    .ptr = try pt.intern(.{ .ptr = .{
                        .ty = manyptr_ty,
                        .base_addr = .{ .uav = .{
                            .orig_ty = manyptr_ty,
                            .val = new_decl_val,
                        } },
                        .byte_offset = 0,
                    } }),
                    .len = (try pt.intValue(Type.usize, struct_field_vals.len)).toIntern(),
                } });
            };

            const decls_val = try sema.typeInfoDecls(block, src, ty.getNamespace(zcu));

            const backing_integer_val = try pt.intern(.{ .opt = .{
                .ty = (try pt.optionalType(.type_type)).toIntern(),
                .val = if (zcu.typeToPackedStruct(ty)) |packed_struct| val: {
                    assert(Type.fromInterned(packed_struct.backingIntTypeUnordered(ip)).isInt(zcu));
                    break :val packed_struct.backingIntTypeUnordered(ip);
                } else .none,
            } });

            const container_layout_ty = try sema.getBuiltinType(src, .@"Type.ContainerLayout");

            const layout = ty.containerLayout(zcu);

            const field_values = [_]InternPool.Index{
                // layout: ContainerLayout,
                (try pt.enumValueFieldIndex(container_layout_ty, @intFromEnum(layout))).toIntern(),
                // backing_integer: ?type,
                backing_integer_val,
                // fields: []const StructField,
                fields_val,
                // decls: []const Declaration,
                decls_val,
                // is_tuple: bool,
                Value.makeBool(ty.isTuple(zcu)).toIntern(),
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.@"struct"))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = type_struct_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .@"opaque" => {
            const type_opaque_ty = try sema.getBuiltinType(src, .@"Type.Opaque");

            try ty.resolveFields(pt);
            const decls_val = try sema.typeInfoDecls(block, src, ty.getNamespace(zcu));

            const field_values = .{
                // decls: []const Declaration,
                decls_val,
            };
            return Air.internedToRef((try pt.internUnion(.{
                .ty = type_info_ty.toIntern(),
                .tag = (try pt.enumValueFieldIndex(type_info_tag_ty, @intFromEnum(std.builtin.TypeId.@"opaque"))).toIntern(),
                .val = try pt.intern(.{ .aggregate = .{
                    .ty = type_opaque_ty.toIntern(),
                    .storage = .{ .elems = &field_values },
                } }),
            })));
        },
        .frame => return sema.failWithUseOfAsync(block, src),
        .@"anyframe" => return sema.failWithUseOfAsync(block, src),
    }
}
```
好的，我现在需要总结这段Zig代码的思维模型。首先，我要理解这段代码的主要功能是什么。看起来这是一个名为`zirTypeInfo`的函数，可能用于处理类型信息，将ZIR指令转换为AIR的引用。

首先，函数接收`sema`（语义分析器）、`block`（代码块）、`inst`（指令索引）作为参数，并返回`Air.Inst.Ref`或者错误。接下来，函数内部定义了一些变量，比如`pt`、`zcu`、`gpa`、`ip`等，这些都是可能用于类型解析和管理的结构体或指针。

然后，代码获取指令的数据和源位置，解析类型`ty`，并获取内置类型`type_info_ty`和`type_info_tag_ty`。接着，检查类型是否有声明依赖，并声明该依赖。

接下来是一个大的`switch`语句，根据`ty`的`zigTypeTag`处理不同类型的逻辑。例如，处理基本类型（如`.type`, `.void`等）、函数类型（`.@"fn"`）、整数类型（`.int`）、浮点类型（`.float`）、指针类型（`.pointer`）、数组（`.array`）、向量（`.vector`）、可选类型（`.optional`）、错误集合（`.error_set`）、联合体（`.union`）、结构体（`.struct`）、枚举（`.enum`）等等。

每个case中，代码会构造相应的类型信息，比如对于函数类型，会处理参数、返回类型、调用约定等，并生成对应的聚合值。对于结构体或联合体，会处理字段信息、对齐、布局等，并最终将这些信息封装到AIR的引用中。

在处理过程中，使用了很多`pt.intern`来内部化（intern）值，可能是为了重用和优化内存。同时，处理错误和内存分配时，有适当的错误处理和资源管理。

需要注意的是，代码中有很多条件判断和类型转换，例如处理可选类型的默认值、错误集合的处理、枚举的tag类型等。此外，对于复杂类型如结构体和联合体，代码会递归解析字段信息，并构建对应的类型描述。

最后，对于不支持的类型（如`.frame`, `.anyframe`），会调用`sema.failWithUseOfAsync`抛出错误。

总结来说，这段代码的核心思维模型是：根据不同的类型标签，递归或迭代地解析类型的各个组成部分，构造对应的类型信息结构，并将其转换为AIR中间表示。处理过程中需要处理各种边缘情况，如泛型参数、可选值、错误集合等，同时确保类型依赖和内存管理的正确性。
================================================
这段代码的思维模型可以概括为**类型信息的递归解析与结构化构建**，其核心逻辑如下：

---

### **1. 功能目标**
将 Zig 类型（通过 ZIR 指令表示）转换为 `std.builtin.Type` 的结构化类型信息（AIR 表示），用于编译期的反射和元编程。

---

### **2. 核心流程**
1. **初始化与依赖管理**  
   - 解析指令数据，获取类型 `ty` 和内置类型 `type_info_ty`。
   - 处理类型声明的依赖关系（如 `declareDependency`）。

2. **类型分类与分派**  
   通过 `switch (ty.zigTypeTag(zcu))` 对类型进行分类，针对不同类别（如函数、整数、指针、结构体等）生成对应的类型信息。

3. **信息提取与构造**  
   - **基本类型**（如 `void`、`bool`）：直接映射到预定义的 `TypeId` 标签。
   - **复合类型**（如函数、结构体、枚举）：
     - 提取子类型（如函数参数、结构体字段、枚举标签）。
     - 构造聚合值（如数组、切片）描述复合结构的细节（如参数列表、字段对齐、默认值）。
   - **特殊类型**（如错误集合、可选类型）：处理可选值和错误集的动态特性。

4. **内存与资源管理**  
   - 使用 `InternPool` 缓存重复值（如字符串、类型标签）。
   - 通过 `sema.arena` 分配临时内存，避免泄漏。

5. **错误处理与边界条件**  
   - 对异步帧（`.frame`）等不支持的类型直接报错。
   - 处理泛型参数（如 `is_generic`）、对齐（`alignment`）、可变参数（`is_var_args`）等边缘情况。

---

### **3. 关键数据结构**
- **`TypeInfo` 的联合体**：  
  对应 `std.builtin.Type`，每种类型对应一个联合分支（如 `.int`、`.struct`）。
- **字段描述对象**：  
  如 `Type.Fn.Param`（函数参数）、`Type.StructField`（结构体字段），通过聚合值（`aggregate`）构造。
- **切片与指针**：  
  用于动态集合（如参数列表 `[]const Fn.Param`）的高效表示。

---

### **4. 递归与组合模式**
- **递归解析**：  
  如结构体字段的类型可能嵌套其他复杂类型，需递归调用类型解析逻辑。
- **组合模式**：  
  通过 `InternPool` 将子类型信息组合成完整的类型描述（如 `internUnion` 和 `internAggregate`）。

---

### **5. 设计模式**
- **工厂模式**：  
  通过 `sema.getBuiltinType` 获取预定义类型（如 `Type.Int`、`Type.Pointer`）。
- **策略模式**：  
  不同类型的分支逻辑独立实现，但统一返回 `Air.Inst.Ref`。

---

### **6. 关键挑战**
- **类型系统的复杂性**：  
  处理 Zig 类型系统的所有特性（如泛型、错误联合、编译期值）。
- **内存与性能优化**：  
  通过 `InternPool` 和切片避免重复数据，减少内存占用。
- **错误恢复**：  
  确保错误路径（如 `try` 和 `catch`）不会泄露资源。

---

### **总结**
这段代码是一个**类型信息编译器**，将高级类型定义转换为底层的结构化数据。其核心是通过递归解析类型结构，动态构造符合 `std.builtin.Type` 的内存表示，同时处理 Zig 类型系统的复杂性和编译期计算的约束。