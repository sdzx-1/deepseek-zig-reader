```zig
pub const SrcLoc = struct {
    file_scope: *File,
    base_node: Ast.Node.Index,
    /// Relative to `base_node`.
    lazy: LazySrcLoc.Offset,

    pub fn baseSrcToken(src_loc: SrcLoc) Ast.TokenIndex {
        const tree = src_loc.file_scope.tree.?;
        return tree.firstToken(src_loc.base_node);
    }

    pub const Span = Ast.Span;

    pub fn span(src_loc: SrcLoc, gpa: Allocator) !Span {
        switch (src_loc.lazy) {
            .unneeded => unreachable,
            .entire_file => return Span{ .start = 0, .end = 1, .main = 0 },

            .byte_abs => |byte_index| return Span{ .start = byte_index, .end = byte_index + 1, .main = byte_index },

            .token_abs => |tok_index| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const start = tree.tokenStart(tok_index);
                const end = start + @as(u32, @intCast(tree.tokenSlice(tok_index).len));
                return Span{ .start = start, .end = end, .main = start };
            },
            .node_abs => |node| {
                const tree = try src_loc.file_scope.getTree(gpa);
                return tree.nodeToSpan(node);
            },
            .byte_offset => |byte_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const tok_index = src_loc.baseSrcToken();
                const start = tree.tokenStart(tok_index) + byte_off;
                const end = start + @as(u32, @intCast(tree.tokenSlice(tok_index).len));
                return Span{ .start = start, .end = end, .main = start };
            },
            .token_offset => |tok_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const tok_index = tok_off.toAbsolute(src_loc.baseSrcToken());
                const start = tree.tokenStart(tok_index);
                const end = start + @as(u32, @intCast(tree.tokenSlice(tok_index).len));
                return Span{ .start = start, .end = end, .main = start };
            },
            .node_offset => |traced_off| {
                const node_off = traced_off.x;
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(node);
            },
            .node_offset_main_token => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const main_token = tree.nodeMainToken(node);
                return tree.tokensToSpan(main_token, main_token, main_token);
            },
            .node_offset_bin_op => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(node);
            },
            .node_offset_initializer => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.tokensToSpan(
                    tree.firstToken(node) - 3,
                    tree.lastToken(node),
                    tree.nodeMainToken(node) - 2,
                );
            },
            .node_offset_var_decl_ty => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const full = switch (tree.nodeTag(node)) {
                    .global_var_decl,
                    .local_var_decl,
                    .simple_var_decl,
                    .aligned_var_decl,
                    => tree.fullVarDecl(node).?,
                    .@"usingnamespace" => {
                        return tree.nodeToSpan(tree.nodeData(node).node);
                    },
                    else => unreachable,
                };
                if (full.ast.type_node.unwrap()) |type_node| {
                    return tree.nodeToSpan(type_node);
                }
                const tok_index = full.ast.mut_token + 1; // the name token
                const start = tree.tokenStart(tok_index);
                const end = start + @as(u32, @intCast(tree.tokenSlice(tok_index).len));
                return Span{ .start = start, .end = end, .main = start };
            },
            .node_offset_var_decl_align => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const align_node = if (tree.fullVarDecl(node)) |v|
                    v.ast.align_node.unwrap().?
                else if (tree.fullFnProto(&buf, node)) |f|
                    f.ast.align_expr.unwrap().?
                else
                    unreachable;
                return tree.nodeToSpan(align_node);
            },
            .node_offset_var_decl_section => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const section_node = if (tree.fullVarDecl(node)) |v|
                    v.ast.section_node.unwrap().?
                else if (tree.fullFnProto(&buf, node)) |f|
                    f.ast.section_expr.unwrap().?
                else
                    unreachable;
                return tree.nodeToSpan(section_node);
            },
            .node_offset_var_decl_addrspace => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const addrspace_node = if (tree.fullVarDecl(node)) |v|
                    v.ast.addrspace_node.unwrap().?
                else if (tree.fullFnProto(&buf, node)) |f|
                    f.ast.addrspace_expr.unwrap().?
                else
                    unreachable;
                return tree.nodeToSpan(addrspace_node);
            },
            .node_offset_var_decl_init => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const init_node = switch (tree.nodeTag(node)) {
                    .global_var_decl,
                    .local_var_decl,
                    .aligned_var_decl,
                    .simple_var_decl,
                    => tree.fullVarDecl(node).?.ast.init_node.unwrap().?,
                    .assign_destructure => tree.assignDestructure(node).ast.value_expr,
                    else => unreachable,
                };
                return tree.nodeToSpan(init_node);
            },
            .node_offset_builtin_call_arg => |builtin_arg| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = builtin_arg.builtin_call_node.toAbsolute(src_loc.base_node);
                var buf: [2]Ast.Node.Index = undefined;
                const params = tree.builtinCallParams(&buf, node).?;
                return tree.nodeToSpan(params[builtin_arg.arg_index]);
            },
            .node_offset_ptrcast_operand => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);

                var node = node_off.toAbsolute(src_loc.base_node);
                while (true) {
                    switch (tree.nodeTag(node)) {
                        .builtin_call_two, .builtin_call_two_comma => {},
                        else => break,
                    }

                    const first_arg, const second_arg = tree.nodeData(node).opt_node_and_opt_node;
                    if (first_arg == .none) break; // 0 args
                    if (second_arg != .none) break; // 2 args

                    const builtin_token = tree.nodeMainToken(node);
                    const builtin_name = tree.tokenSlice(builtin_token);
                    const info = BuiltinFn.list.get(builtin_name) orelse break;

                    switch (info.tag) {
                        else => break,
                        .ptr_cast,
                        .align_cast,
                        .addrspace_cast,
                        .const_cast,
                        .volatile_cast,
                        => {},
                    }

                    node = first_arg.unwrap().?;
                }

                return tree.nodeToSpan(node);
            },
            .node_offset_array_access_index => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(tree.nodeData(node).node_and_node[1]);
            },
            .node_offset_slice_ptr,
            .node_offset_slice_start,
            .node_offset_slice_end,
            .node_offset_slice_sentinel,
            => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const full = tree.fullSlice(node).?;
                const part_node = switch (src_loc.lazy) {
                    .node_offset_slice_ptr => full.ast.sliced,
                    .node_offset_slice_start => full.ast.start,
                    .node_offset_slice_end => full.ast.end.unwrap().?,
                    .node_offset_slice_sentinel => full.ast.sentinel.unwrap().?,
                    else => unreachable,
                };
                return tree.nodeToSpan(part_node);
            },
            .node_offset_call_func => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullCall(&buf, node).?;
                return tree.nodeToSpan(full.ast.fn_expr);
            },
            .node_offset_field_name => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const tok_index = switch (tree.nodeTag(node)) {
                    .field_access => tree.nodeData(node).node_and_token[1],
                    .call_one,
                    .call_one_comma,
                    .async_call_one,
                    .async_call_one_comma,
                    .call,
                    .call_comma,
                    .async_call,
                    .async_call_comma,
                    => blk: {
                        const full = tree.fullCall(&buf, node).?;
                        break :blk tree.lastToken(full.ast.fn_expr);
                    },
                    else => tree.firstToken(node) - 2,
                };
                const start = tree.tokenStart(tok_index);
                const end = start + @as(u32, @intCast(tree.tokenSlice(tok_index).len));
                return Span{ .start = start, .end = end, .main = start };
            },
            .node_offset_field_name_init => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const tok_index = tree.firstToken(node) - 2;
                const start = tree.tokenStart(tok_index);
                const end = start + @as(u32, @intCast(tree.tokenSlice(tok_index).len));
                return Span{ .start = start, .end = end, .main = start };
            },
            .node_offset_deref_ptr => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(node);
            },
            .node_offset_asm_source => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const full = tree.fullAsm(node).?;
                return tree.nodeToSpan(full.ast.template);
            },
            .node_offset_asm_ret_ty => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const full = tree.fullAsm(node).?;
                const asm_output = full.outputs[0];
                return tree.nodeToSpan(tree.nodeData(asm_output).opt_node_and_token[0].unwrap().?);
            },

            .node_offset_if_cond => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const src_node = switch (tree.nodeTag(node)) {
                    .if_simple,
                    .@"if",
                    => tree.fullIf(node).?.ast.cond_expr,

                    .while_simple,
                    .while_cont,
                    .@"while",
                    => tree.fullWhile(node).?.ast.cond_expr,

                    .for_simple,
                    .@"for",
                    => {
                        const inputs = tree.fullFor(node).?.ast.inputs;
                        const start = tree.firstToken(inputs[0]);
                        const end = tree.lastToken(inputs[inputs.len - 1]);
                        return tree.tokensToSpan(start, end, start);
                    },

                    .@"orelse" => node,
                    .@"catch" => node,
                    else => unreachable,
                };
                return tree.nodeToSpan(src_node);
            },
            .for_input => |for_input| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = for_input.for_node_offset.toAbsolute(src_loc.base_node);
                const for_full = tree.fullFor(node).?;
                const src_node = for_full.ast.inputs[for_input.input_index];
                return tree.nodeToSpan(src_node);
            },
            .for_capture_from_input => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const input_node = node_off.toAbsolute(src_loc.base_node);
                // We have to actually linear scan the whole AST to find the for loop
                // that contains this input.
                const node_tags = tree.nodes.items(.tag);
                for (node_tags, 0..) |node_tag, node_usize| {
                    const node: Ast.Node.Index = @enumFromInt(node_usize);
                    switch (node_tag) {
                        .for_simple, .@"for" => {
                            const for_full = tree.fullFor(node).?;
                            for (for_full.ast.inputs, 0..) |input, input_index| {
                                if (input_node == input) {
                                    var count = input_index;
                                    var tok = for_full.payload_token;
                                    while (true) {
                                        switch (tree.tokenTag(tok)) {
                                            .comma => {
                                                count -= 1;
                                                tok += 1;
                                            },
                                            .identifier => {
                                                if (count == 0)
                                                    return tree.tokensToSpan(tok, tok + 1, tok);
                                                tok += 1;
                                            },
                                            .asterisk => {
                                                if (count == 0)
                                                    return tree.tokensToSpan(tok, tok + 2, tok);
                                                tok += 1;
                                            },
                                            else => unreachable,
                                        }
                                    }
                                }
                            }
                        },
                        else => continue,
                    }
                } else unreachable;
            },
            .call_arg => |call_arg| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = call_arg.call_node_offset.toAbsolute(src_loc.base_node);
                var buf: [2]Ast.Node.Index = undefined;
                const call_full = tree.fullCall(buf[0..1], node) orelse {
                    assert(tree.nodeTag(node) == .builtin_call);
                    const call_args_node: Ast.Node.Index = @enumFromInt(tree.extra_data[@intFromEnum(tree.nodeData(node).extra_range.end) - 1]);
                    switch (tree.nodeTag(call_args_node)) {
                        .array_init_one,
                        .array_init_one_comma,
                        .array_init_dot_two,
                        .array_init_dot_two_comma,
                        .array_init_dot,
                        .array_init_dot_comma,
                        .array_init,
                        .array_init_comma,
                        => {
                            const full = tree.fullArrayInit(&buf, call_args_node).?.ast.elements;
                            return tree.nodeToSpan(full[call_arg.arg_index]);
                        },
                        .struct_init_one,
                        .struct_init_one_comma,
                        .struct_init_dot_two,
                        .struct_init_dot_two_comma,
                        .struct_init_dot,
                        .struct_init_dot_comma,
                        .struct_init,
                        .struct_init_comma,
                        => {
                            const full = tree.fullStructInit(&buf, call_args_node).?.ast.fields;
                            return tree.nodeToSpan(full[call_arg.arg_index]);
                        },
                        else => return tree.nodeToSpan(call_args_node),
                    }
                };
                return tree.nodeToSpan(call_full.ast.params[call_arg.arg_index]);
            },
            .fn_proto_param, .fn_proto_param_type => |fn_proto_param| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = fn_proto_param.fn_proto_node_offset.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, node).?;
                var it = full.iterate(tree);
                var i: usize = 0;
                while (it.next()) |param| : (i += 1) {
                    if (i != fn_proto_param.param_index) continue;

                    switch (src_loc.lazy) {
                        .fn_proto_param_type => if (param.anytype_ellipsis3) |tok| {
                            return tree.tokenToSpan(tok);
                        } else {
                            return tree.nodeToSpan(param.type_expr.?);
                        },
                        .fn_proto_param => if (param.anytype_ellipsis3) |tok| {
                            const first = param.comptime_noalias orelse param.name_token orelse tok;
                            return tree.tokensToSpan(first, tok, first);
                        } else {
                            const first = param.comptime_noalias orelse param.name_token orelse tree.firstToken(param.type_expr.?);
                            return tree.tokensToSpan(first, tree.lastToken(param.type_expr.?), first);
                        },
                        else => unreachable,
                    }
                }
                unreachable;
            },
            .node_offset_bin_lhs => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(tree.nodeData(node).node_and_node[0]);
            },
            .node_offset_bin_rhs => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(tree.nodeData(node).node_and_node[1]);
            },
            .array_cat_lhs, .array_cat_rhs => |cat| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = cat.array_cat_offset.toAbsolute(src_loc.base_node);
                const arr_node = if (src_loc.lazy == .array_cat_lhs)
                    tree.nodeData(node).node_and_node[0]
                else
                    tree.nodeData(node).node_and_node[1];

                var buf: [2]Ast.Node.Index = undefined;
                switch (tree.nodeTag(arr_node)) {
                    .array_init_one,
                    .array_init_one_comma,
                    .array_init_dot_two,
                    .array_init_dot_two_comma,
                    .array_init_dot,
                    .array_init_dot_comma,
                    .array_init,
                    .array_init_comma,
                    => {
                        const full = tree.fullArrayInit(&buf, arr_node).?.ast.elements;
                        return tree.nodeToSpan(full[cat.elem_index]);
                    },
                    else => return tree.nodeToSpan(arr_node),
                }
            },

            .node_offset_try_operand => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(tree.nodeData(node).node);
            },

            .node_offset_switch_operand => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                const condition, _ = tree.nodeData(node).node_and_extra;
                return tree.nodeToSpan(condition);
            },

            .node_offset_switch_special_prong => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const switch_node = node_off.toAbsolute(src_loc.base_node);
                _, const extra_index = tree.nodeData(switch_node).node_and_extra;
                const case_nodes = tree.extraDataSlice(tree.extraData(extra_index, Ast.Node.SubRange), Ast.Node.Index);
                for (case_nodes) |case_node| {
                    const case = tree.fullSwitchCase(case_node).?;
                    const is_special = (case.ast.values.len == 0) or
                        (case.ast.values.len == 1 and
                            tree.nodeTag(case.ast.values[0]) == .identifier and
                            mem.eql(u8, tree.tokenSlice(tree.nodeMainToken(case.ast.values[0])), "_"));
                    if (!is_special) continue;

                    return tree.nodeToSpan(case_node);
                } else unreachable;
            },

            .node_offset_switch_range => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const switch_node = node_off.toAbsolute(src_loc.base_node);
                _, const extra_index = tree.nodeData(switch_node).node_and_extra;
                const case_nodes = tree.extraDataSlice(tree.extraData(extra_index, Ast.Node.SubRange), Ast.Node.Index);
                for (case_nodes) |case_node| {
                    const case = tree.fullSwitchCase(case_node).?;
                    const is_special = (case.ast.values.len == 0) or
                        (case.ast.values.len == 1 and
                            tree.nodeTag(case.ast.values[0]) == .identifier and
                            mem.eql(u8, tree.tokenSlice(tree.nodeMainToken(case.ast.values[0])), "_"));
                    if (is_special) continue;

                    for (case.ast.values) |item_node| {
                        if (tree.nodeTag(item_node) == .switch_range) {
                            return tree.nodeToSpan(item_node);
                        }
                    }
                } else unreachable;
            },
            .node_offset_fn_type_align => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, node).?;
                return tree.nodeToSpan(full.ast.align_expr.unwrap().?);
            },
            .node_offset_fn_type_addrspace => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, node).?;
                return tree.nodeToSpan(full.ast.addrspace_expr.unwrap().?);
            },
            .node_offset_fn_type_section => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, node).?;
                return tree.nodeToSpan(full.ast.section_expr.unwrap().?);
            },
            .node_offset_fn_type_cc => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, node).?;
                return tree.nodeToSpan(full.ast.callconv_expr.unwrap().?);
            },

            .node_offset_fn_type_ret_ty => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, node).?;
                return tree.nodeToSpan(full.ast.return_type.unwrap().?);
            },
            .node_offset_param => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);

                var first_tok = tree.firstToken(node);
                while (true) switch (tree.tokenTag(first_tok - 1)) {
                    .colon, .identifier, .keyword_comptime, .keyword_noalias => first_tok -= 1,
                    else => break,
                };
                return tree.tokensToSpan(
                    first_tok,
                    tree.lastToken(node),
                    first_tok,
                );
            },
            .token_offset_param => |token_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const main_token = tree.nodeMainToken(src_loc.base_node);
                const tok_index = token_off.toAbsolute(main_token);

                var first_tok = tok_index;
                while (true) switch (tree.tokenTag(first_tok - 1)) {
                    .colon, .identifier, .keyword_comptime, .keyword_noalias => first_tok -= 1,
                    else => break,
                };
                return tree.tokensToSpan(
                    first_tok,
                    tok_index,
                    first_tok,
                );
            },

            .node_offset_anyframe_type => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);
                _, const child_type = tree.nodeData(parent_node).token_and_node;
                return tree.nodeToSpan(child_type);
            },

            .node_offset_lib_name => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, parent_node).?;
                const tok_index = full.lib_name.?;
                const start = tree.tokenStart(tok_index);
                const end = start + @as(u32, @intCast(tree.tokenSlice(tok_index).len));
                return Span{ .start = start, .end = end, .main = start };
            },

            .node_offset_array_type_len => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullArrayType(parent_node).?;
                return tree.nodeToSpan(full.ast.elem_count);
            },
            .node_offset_array_type_sentinel => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullArrayType(parent_node).?;
                return tree.nodeToSpan(full.ast.sentinel.unwrap().?);
            },
            .node_offset_array_type_elem => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullArrayType(parent_node).?;
                return tree.nodeToSpan(full.ast.elem_type);
            },
            .node_offset_un_op => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                return tree.nodeToSpan(tree.nodeData(node).node);
            },
            .node_offset_ptr_elem => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullPtrType(parent_node).?;
                return tree.nodeToSpan(full.ast.child_type);
            },
            .node_offset_ptr_sentinel => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullPtrType(parent_node).?;
                return tree.nodeToSpan(full.ast.sentinel.unwrap().?);
            },
            .node_offset_ptr_align => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullPtrType(parent_node).?;
                return tree.nodeToSpan(full.ast.align_node.unwrap().?);
            },
            .node_offset_ptr_addrspace => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullPtrType(parent_node).?;
                return tree.nodeToSpan(full.ast.addrspace_node.unwrap().?);
            },
            .node_offset_ptr_bitoffset => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullPtrType(parent_node).?;
                return tree.nodeToSpan(full.ast.bit_range_start.unwrap().?);
            },
            .node_offset_ptr_hostsize => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full = tree.fullPtrType(parent_node).?;
                return tree.nodeToSpan(full.ast.bit_range_end.unwrap().?);
            },
            .node_offset_container_tag => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                switch (tree.nodeTag(parent_node)) {
                    .container_decl_arg, .container_decl_arg_trailing => {
                        const full = tree.containerDeclArg(parent_node);
                        const arg_node = full.ast.arg.unwrap().?;
                        return tree.nodeToSpan(arg_node);
                    },
                    .tagged_union_enum_tag, .tagged_union_enum_tag_trailing => {
                        const full = tree.taggedUnionEnumTag(parent_node);
                        const arg_node = full.ast.arg.unwrap().?;

                        return tree.tokensToSpan(
                            tree.firstToken(arg_node) - 2,
                            tree.lastToken(arg_node) + 1,
                            tree.nodeMainToken(arg_node),
                        );
                    },
                    else => unreachable,
                }
            },
            .node_offset_field_default => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                const full: Ast.full.ContainerField = switch (tree.nodeTag(parent_node)) {
                    .container_field => tree.containerField(parent_node),
                    .container_field_init => tree.containerFieldInit(parent_node),
                    else => unreachable,
                };
                return tree.nodeToSpan(full.ast.value_expr.unwrap().?);
            },
            .node_offset_init_ty => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const parent_node = node_off.toAbsolute(src_loc.base_node);

                var buf: [2]Ast.Node.Index = undefined;
                const type_expr = if (tree.fullArrayInit(&buf, parent_node)) |array_init|
                    array_init.ast.type_expr.unwrap().?
                else
                    tree.fullStructInit(&buf, parent_node).?.ast.type_expr.unwrap().?;
                return tree.nodeToSpan(type_expr);
            },
            .node_offset_store_ptr => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);

                switch (tree.nodeTag(node)) {
                    .assign => {
                        return tree.nodeToSpan(tree.nodeData(node).node_and_node[0]);
                    },
                    else => return tree.nodeToSpan(node),
                }
            },
            .node_offset_store_operand => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);

                switch (tree.nodeTag(node)) {
                    .assign => {
                        return tree.nodeToSpan(tree.nodeData(node).node_and_node[1]);
                    },
                    else => return tree.nodeToSpan(node),
                }
            },
            .node_offset_return_operand => |node_off| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = node_off.toAbsolute(src_loc.base_node);
                if (tree.nodeTag(node) == .@"return") {
                    if (tree.nodeData(node).opt_node.unwrap()) |lhs| {
                        return tree.nodeToSpan(lhs);
                    }
                }
                return tree.nodeToSpan(node);
            },
            .container_field_name,
            .container_field_value,
            .container_field_type,
            .container_field_align,
            => |field_idx| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = src_loc.base_node;
                var buf: [2]Ast.Node.Index = undefined;
                const container_decl = tree.fullContainerDecl(&buf, node) orelse
                    return tree.nodeToSpan(node);

                var cur_field_idx: usize = 0;
                for (container_decl.ast.members) |member_node| {
                    const field = tree.fullContainerField(member_node) orelse continue;
                    if (cur_field_idx < field_idx) {
                        cur_field_idx += 1;
                        continue;
                    }
                    const field_component_node = switch (src_loc.lazy) {
                        .container_field_name => .none,
                        .container_field_value => field.ast.value_expr,
                        .container_field_type => field.ast.type_expr,
                        .container_field_align => field.ast.align_expr,
                        else => unreachable,
                    };
                    if (field_component_node.unwrap()) |component_node| {
                        return tree.nodeToSpan(component_node);
                    } else {
                        return tree.tokenToSpan(field.ast.main_token);
                    }
                } else unreachable;
            },
            .tuple_field_type, .tuple_field_init => |field_info| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = src_loc.base_node;
                var buf: [2]Ast.Node.Index = undefined;
                const container_decl = tree.fullContainerDecl(&buf, node) orelse
                    return tree.nodeToSpan(node);

                const field = tree.fullContainerField(container_decl.ast.members[field_info.elem_index]).?;
                return tree.nodeToSpan(switch (src_loc.lazy) {
                    .tuple_field_type => field.ast.type_expr.unwrap().?,
                    .tuple_field_init => field.ast.value_expr.unwrap().?,
                    else => unreachable,
                });
            },
            .init_elem => |init_elem| {
                const tree = try src_loc.file_scope.getTree(gpa);
                const init_node = init_elem.init_node_offset.toAbsolute(src_loc.base_node);
                var buf: [2]Ast.Node.Index = undefined;
                if (tree.fullArrayInit(&buf, init_node)) |full| {
                    const elem_node = full.ast.elements[init_elem.elem_index];
                    return tree.nodeToSpan(elem_node);
                } else if (tree.fullStructInit(&buf, init_node)) |full| {
                    const field_node = full.ast.fields[init_elem.elem_index];
                    return tree.tokensToSpan(
                        tree.firstToken(field_node) - 3,
                        tree.lastToken(field_node),
                        tree.nodeMainToken(field_node) - 2,
                    );
                } else unreachable;
            },
            .init_field_name,
            .init_field_linkage,
            .init_field_section,
            .init_field_visibility,
            .init_field_rw,
            .init_field_locality,
            .init_field_cache,
            .init_field_library,
            .init_field_thread_local,
            .init_field_dll_import,
            => |builtin_call_node| {
                const wanted = switch (src_loc.lazy) {
                    .init_field_name => "name",
                    .init_field_linkage => "linkage",
                    .init_field_section => "section",
                    .init_field_visibility => "visibility",
                    .init_field_rw => "rw",
                    .init_field_locality => "locality",
                    .init_field_cache => "cache",
                    .init_field_library => "library",
                    .init_field_thread_local => "thread_local",
                    .init_field_dll_import => "dll_import",
                    else => unreachable,
                };
                const tree = try src_loc.file_scope.getTree(gpa);
                const node = builtin_call_node.toAbsolute(src_loc.base_node);
                var builtin_buf: [2]Ast.Node.Index = undefined;
                const args = tree.builtinCallParams(&builtin_buf, node).?;
                const arg_node = args[1];
                var buf: [2]Ast.Node.Index = undefined;
                const full = tree.fullStructInit(&buf, arg_node) orelse
                    return tree.nodeToSpan(arg_node);
                for (full.ast.fields) |field_node| {
                    // . IDENTIFIER = field_node
                    const name_token = tree.firstToken(field_node) - 2;
                    const name = tree.tokenSlice(name_token);
                    if (std.mem.eql(u8, name, wanted)) {
                        return tree.tokensToSpan(
                            name_token - 1,
                            tree.lastToken(field_node),
                            tree.nodeMainToken(field_node) - 2,
                        );
                    }
                }
                return tree.nodeToSpan(arg_node);
            },
            .switch_case_item,
            .switch_case_item_range_first,
            .switch_case_item_range_last,
            .switch_capture,
            .switch_tag_capture,
            => {
                const switch_node_offset, const want_case_idx = switch (src_loc.lazy) {
                    .switch_case_item,
                    .switch_case_item_range_first,
                    .switch_case_item_range_last,
                    => |x| .{ x.switch_node_offset, x.case_idx },
                    .switch_capture,
                    .switch_tag_capture,
                    => |x| .{ x.switch_node_offset, x.case_idx },
                    else => unreachable,
                };

                const tree = try src_loc.file_scope.getTree(gpa);
                const switch_node = switch_node_offset.toAbsolute(src_loc.base_node);
                _, const extra_index = tree.nodeData(switch_node).node_and_extra;
                const case_nodes = tree.extraDataSlice(tree.extraData(extra_index, Ast.Node.SubRange), Ast.Node.Index);

                var multi_i: u32 = 0;
                var scalar_i: u32 = 0;
                const case = for (case_nodes) |case_node| {
                    const case = tree.fullSwitchCase(case_node).?;
                    const is_special = special: {
                        if (case.ast.values.len == 0) break :special true;
                        if (case.ast.values.len == 1 and tree.nodeTag(case.ast.values[0]) == .identifier) {
                            break :special mem.eql(u8, tree.tokenSlice(tree.nodeMainToken(case.ast.values[0])), "_");
                        }
                        break :special false;
                    };
                    if (is_special) {
                        if (want_case_idx.isSpecial()) {
                            break case;
                        }
                        continue;
                    }

                    const is_multi = case.ast.values.len != 1 or
                        tree.nodeTag(case.ast.values[0]) == .switch_range;

                    switch (want_case_idx.kind) {
                        .scalar => if (!is_multi and want_case_idx.index == scalar_i) break case,
                        .multi => if (is_multi and want_case_idx.index == multi_i) break case,
                    }

                    if (is_multi) {
                        multi_i += 1;
                    } else {
                        scalar_i += 1;
                    }
                } else unreachable;

                const want_item = switch (src_loc.lazy) {
                    .switch_case_item,
                    .switch_case_item_range_first,
                    .switch_case_item_range_last,
                    => |x| x.item_idx,
                    .switch_capture, .switch_tag_capture => {
                        const start = switch (src_loc.lazy) {
                            .switch_capture => case.payload_token.?,
                            .switch_tag_capture => tok: {
                                var tok = case.payload_token.?;
                                if (tree.tokenTag(tok) == .asterisk) tok += 1;
                                tok = tok + 2; // skip over comma
                                break :tok tok;
                            },
                            else => unreachable,
                        };
                        const end = switch (tree.tokenTag(start)) {
                            .asterisk => start + 1,
                            else => start,
                        };
                        return tree.tokensToSpan(start, end, start);
                    },
                    else => unreachable,
                };

                switch (want_item.kind) {
                    .single => {
                        var item_i: u32 = 0;
                        for (case.ast.values) |item_node| {
                            if (tree.nodeTag(item_node) == .switch_range) continue;
                            if (item_i != want_item.index) {
                                item_i += 1;
                                continue;
                            }
                            return tree.nodeToSpan(item_node);
                        } else unreachable;
                    },
                    .range => {
                        var range_i: u32 = 0;
                        for (case.ast.values) |item_node| {
                            if (tree.nodeTag(item_node) != .switch_range) continue;
                            if (range_i != want_item.index) {
                                range_i += 1;
                                continue;
                            }
                            const first, const last = tree.nodeData(item_node).node_and_node;
                            return switch (src_loc.lazy) {
                                .switch_case_item => tree.nodeToSpan(item_node),
                                .switch_case_item_range_first => tree.nodeToSpan(first),
                                .switch_case_item_range_last => tree.nodeToSpan(last),
                                else => unreachable,
                            };
                        } else unreachable;
                    },
                }
            },
            .func_decl_param_comptime => |param_idx| {
                const tree = try src_loc.file_scope.getTree(gpa);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, src_loc.base_node).?;
                var param_it = full.iterate(tree);
                for (0..param_idx) |_| assert(param_it.next() != null);
                const param = param_it.next().?;
                return tree.tokenToSpan(param.comptime_noalias.?);
            },
            .func_decl_param_ty => |param_idx| {
                const tree = try src_loc.file_scope.getTree(gpa);
                var buf: [1]Ast.Node.Index = undefined;
                const full = tree.fullFnProto(&buf, src_loc.base_node).?;
                var param_it = full.iterate(tree);
                for (0..param_idx) |_| assert(param_it.next() != null);
                const param = param_it.next().?;
                return tree.nodeToSpan(param.type_expr.?);
            },
        }
    }
}
```
嗯，我现在要总结这段Zig代码的思维模型。首先，我需要理解这段代码的结构和功能。看起来这是一个定义`SrcLoc`结构体的模块，用于处理源代码位置信息。`SrcLoc`包含文件作用域、基础节点和懒加载的源位置偏移。

接下来，`baseSrcToken`方法似乎用来获取基础源标记，通过访问文件作用域的语法树来找到对应的第一个标记。然后，`Span`结构体可能表示源代码的起始和结束位置。

`span`方法是关键，它处理不同的`lazy`偏移类型，生成对应的`Span`。这里有多个分支，比如`byte_abs`、`token_abs`、`node_abs`等等。每个分支对应不同的处理逻辑，例如直接根据字节索引、标记索引或节点索引来定位源代码范围。

我需要理清每个分支的作用。比如，`token_offset`会根据基础标记的偏移量计算位置，而`node_offset`则处理节点的绝对或相对偏移。还有一些特定的情况，比如`node_offset_var_decl_ty`处理变量声明类型的位置，或者`node_offset_if_cond`处理条件表达式的位置。

在处理这些分支时，代码多次调用`getTree`来获取语法树，然后根据不同的节点类型和偏移量计算起始和结束位置。有些情况下需要处理复杂的结构，如循环、函数参数、数组访问等。

此外，一些分支涉及到错误处理，比如`try`语句，可能抛出错误。需要确保所有可能的错误都被正确处理，比如内存分配失败的情况。

最后，代码中大量使用了模式匹配（switch语句）来处理不同的`lazy`类型，每个类型对应特定的逻辑来生成源代码范围。这可能涉及到Zig语言的AST（抽象语法树）结构和相关操作方法。

总结来说，这段代码的核心是根据不同的源代码位置描述方式，动态计算出对应的源代码范围（Span），用于编译过程中的错误报告、调试信息等。每个分支处理一种特定的位置表示方式，通过访问语法树的节点和标记来获取具体的位置信息。
================================================
这段代码定义了`SrcLoc`结构体及其方法，用于在Zig编译器中动态解析源代码位置信息，生成对应的代码范围（`Span`）。其思维模型可总结如下：

---

### **核心目标**
根据不同的**位置描述方式**（通过`lazy`字段的多种变体），动态计算源代码的起始、结束和主标识符位置，用于错误提示、调试信息等场景。

---

### **关键组件**
1. **`SrcLoc`结构体**  
   - **成员**：
     - `file_scope`：所属文件。
     - `base_node`：基准AST节点。
     - `lazy`：偏移类型（`LazySrcLoc.Offset`的枚举变体），决定如何从基准节点定位目标位置。

2. **`Span`方法**  
   核心方法`span`通过模式匹配`lazy`字段的多种变体，生成对应的代码范围：
   - **通用逻辑**：
     - 获取语法树（`tree`）。
     - 根据`lazy`类型，计算`start`（起始位置）、`end`（结束位置）、`main`（主位置）。
   - **分支逻辑**：
     - **直接定位**：如`byte_abs`（直接字节索引）、`token_abs`（直接标记索引）。
     - **相对偏移**：如`token_offset`（基于基准标记的偏移）、`node_offset`（基于基准节点的偏移）。
     - **复杂结构**：如`node_offset_var_decl_ty`（变量声明类型）、`node_offset_if_cond`（条件表达式）、`switch_case_item`（`switch`分支项）等，需解析AST节点的子结构。
     - **特殊语法**：如`ptrcast`操作符、`asm`块、函数原型参数、容器字段等，需处理特定语法规则。

3. **辅助方法**  
   - `baseSrcToken`：获取基准节点对应的首个标记。
   - `nodeToSpan`、`tokenToSpan`等：将节点或标记转换为`Span`。

---

### **核心逻辑**
1. **语法树操作**  
   通过`file_scope.getTree(gpa)`获取语法树，遍历节点和标记，提取位置信息（如`tokenStart`、`tokenSlice`）。

2. **模式匹配**  
   通过`switch (src_loc.lazy)`处理20+种偏移类型，每种类型对应不同的AST解析逻辑：
   - **简单类型**：直接映射到固定位置（如`entire_file`表示整个文件）。
   - **节点/标记偏移**：基于`base_node`或`baseSrcToken`计算绝对位置。
   - **复杂结构**：如`for`循环的捕获变量、`switch`的特殊分支（`_`通配符）、函数参数的`comptime`修饰符等，需深度遍历AST。

3. **错误处理**  
   使用`try`确保内存分配（`gpa`）和语法树解析过程中的错误被传播。

---

### **设计模式**
1. **策略模式**  
   每个`lazy`变体对应一种定位策略，通过模式匹配选择具体实现。
   
2. **递归解析**  
   部分逻辑（如`node_offset_ptrcast_operand`）需要递归解析AST节点，直到满足条件。

3. **数据驱动**  
   `lazy`字段的变体决定了代码的行为，将逻辑与数据解耦。

---

### **典型用例**
- **变量声明类型**：通过`node_offset_var_decl_ty`定位变量声明的类型节点。
- **函数参数**：通过`fn_proto_param`解析函数原型的参数位置。
- **错误位置**：当代码抛出错误时，根据`SrcLoc`生成具体的代码范围供用户定位。

---

### **总结**
这段代码是Zig编译器前端的关键部分，通过灵活处理多种源代码位置描述方式，实现了高精度的代码位置映射。其核心是通过AST的深度解析和模式匹配，将抽象的节点/标记偏移转换为具体的字节范围，为开发者提供直观的调试和错误信息。