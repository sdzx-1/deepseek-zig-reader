```zig
pub const Object = struct {
    gpa: Allocator,
    builder: Builder,

    pt: Zcu.PerThread,

    debug_compile_unit: Builder.Metadata,

    debug_enums_fwd_ref: Builder.Metadata,
    debug_globals_fwd_ref: Builder.Metadata,

    debug_enums: std.ArrayListUnmanaged(Builder.Metadata),
    debug_globals: std.ArrayListUnmanaged(Builder.Metadata),

    debug_file_map: std.AutoHashMapUnmanaged(Zcu.File.Index, Builder.Metadata),
    debug_type_map: std.AutoHashMapUnmanaged(Type, Builder.Metadata),

    debug_unresolved_namespace_scopes: std.AutoArrayHashMapUnmanaged(InternPool.NamespaceIndex, Builder.Metadata),

    target: std.Target,
    /// Ideally we would use `llvm_module.getNamedFunction` to go from *Decl to LLVM function,
    /// but that has some downsides:
    /// * we have to compute the fully qualified name every time we want to do the lookup
    /// * for externally linked functions, the name is not fully qualified, but when
    ///   a Decl goes from exported to not exported and vice-versa, we would use the wrong
    ///   version of the name and incorrectly get function not found in the llvm module.
    /// * it works for functions not all globals.
    /// Therefore, this table keeps track of the mapping.
    nav_map: std.AutoHashMapUnmanaged(InternPool.Nav.Index, Builder.Global.Index),
    /// Same deal as `decl_map` but for anonymous declarations, which are always global constants.
    uav_map: std.AutoHashMapUnmanaged(InternPool.Index, Builder.Global.Index),
    /// Maps enum types to their corresponding LLVM functions for implementing the `tag_name` instruction.
    enum_tag_name_map: std.AutoHashMapUnmanaged(InternPool.Index, Builder.Global.Index),
    /// Serves the same purpose as `enum_tag_name_map` but for the `is_named_enum_value` instruction.
    named_enum_map: std.AutoHashMapUnmanaged(InternPool.Index, Builder.Function.Index),
    /// Maps Zig types to LLVM types. The table memory is backed by the GPA of
    /// the compiler.
    /// TODO when InternPool garbage collection is implemented, this map needs
    /// to be garbage collected as well.
    type_map: TypeMap,
    /// The LLVM global table which holds the names corresponding to Zig errors.
    /// Note that the values are not added until `emit`, when all errors in
    /// the compilation are known.
    error_name_table: Builder.Variable.Index,

    /// Memoizes a null `?usize` value.
    null_opt_usize: Builder.Constant,

    /// When an LLVM struct type is created, an entry is inserted into this
    /// table for every zig source field of the struct that has a corresponding
    /// LLVM struct field. comptime fields are not included. Zero-bit fields are
    /// mapped to a field at the correct byte, which may be a padding field, or
    /// are not mapped, in which case they are semantically at the end of the
    /// struct.
    /// The value is the LLVM struct field index.
    /// This is denormalized data.
    struct_field_map: std.AutoHashMapUnmanaged(ZigStructField, c_uint),

    /// Values for `@llvm.used`.
    used: std.ArrayListUnmanaged(Builder.Constant),

    const ZigStructField = struct {
        struct_ty: InternPool.Index,
        field_index: u32,
    };

    pub const Ptr = if (dev.env.supports(.llvm_backend)) *Object else noreturn;

    pub const TypeMap = std.AutoHashMapUnmanaged(InternPool.Index, Builder.Type);

    pub fn create(arena: Allocator, comp: *Compilation) !Ptr {
        dev.check(.llvm_backend);
        const gpa = comp.gpa;
        const target = comp.root_mod.resolved_target.result;
        const llvm_target_triple = try targetTriple(arena, target);

        var builder = try Builder.init(.{
            .allocator = gpa,
            .strip = comp.config.debug_format == .strip,
            .name = comp.root_name,
            .target = target,
            .triple = llvm_target_triple,
        });
        errdefer builder.deinit();

        builder.data_layout = try builder.string(dataLayout(target));

        const debug_compile_unit, const debug_enums_fwd_ref, const debug_globals_fwd_ref =
            if (!builder.strip) debug_info: {
                // We fully resolve all paths at this point to avoid lack of
                // source line info in stack traces or lack of debugging
                // information which, if relative paths were used, would be
                // very location dependent.
                // TODO: the only concern I have with this is WASI as either host or target, should
                // we leave the paths as relative then?
                // TODO: This is totally wrong. In dwarf, paths are encoded as relative to
                // a particular directory, and then the directory path is specified elsewhere.
                // In the compiler frontend we have it stored correctly in this
                // way already, but here we throw all that sweet information
                // into the garbage can by converting into absolute paths. What
                // a terrible tragedy.
                const compile_unit_dir = blk: {
                    if (comp.zcu) |zcu| m: {
                        const d = try zcu.main_mod.root.joinString(arena, "");
                        if (d.len == 0) break :m;
                        if (std.fs.path.isAbsolute(d)) break :blk d;
                        break :blk std.fs.realpathAlloc(arena, d) catch break :blk d;
                    }
                    break :blk try std.process.getCwdAlloc(arena);
                };

                const debug_file = try builder.debugFile(
                    try builder.metadataString(comp.root_name),
                    try builder.metadataString(compile_unit_dir),
                );

                const debug_enums_fwd_ref = try builder.debugForwardReference();
                const debug_globals_fwd_ref = try builder.debugForwardReference();

                const debug_compile_unit = try builder.debugCompileUnit(
                    debug_file,
                    // Don't use the version string here; LLVM misparses it when it
                    // includes the git revision.
                    try builder.metadataStringFmt("zig {d}.{d}.{d}", .{
                        build_options.semver.major,
                        build_options.semver.minor,
                        build_options.semver.patch,
                    }),
                    debug_enums_fwd_ref,
                    debug_globals_fwd_ref,
                    .{ .optimized = comp.root_mod.optimize_mode != .Debug },
                );

                try builder.metadataNamed(try builder.metadataString("llvm.dbg.cu"), &.{debug_compile_unit});
                break :debug_info .{ debug_compile_unit, debug_enums_fwd_ref, debug_globals_fwd_ref };
            } else .{.none} ** 3;

        const obj = try arena.create(Object);
        obj.* = .{
            .gpa = gpa,
            .builder = builder,
            .pt = .{
                .zcu = comp.zcu.?,
                .tid = .main,
            },
            .debug_compile_unit = debug_compile_unit,
            .debug_enums_fwd_ref = debug_enums_fwd_ref,
            .debug_globals_fwd_ref = debug_globals_fwd_ref,
            .debug_enums = .{},
            .debug_globals = .{},
            .debug_file_map = .{},
            .debug_type_map = .{},
            .debug_unresolved_namespace_scopes = .{},
            .target = target,
            .nav_map = .{},
            .uav_map = .{},
            .enum_tag_name_map = .{},
            .named_enum_map = .{},
            .type_map = .{},
            .error_name_table = .none,
            .null_opt_usize = .no_init,
            .struct_field_map = .{},
            .used = .{},
        };
        return obj;
    }

    pub fn deinit(self: *Object) void {
        const gpa = self.gpa;
        self.debug_enums.deinit(gpa);
        self.debug_globals.deinit(gpa);
        self.debug_file_map.deinit(gpa);
        self.debug_type_map.deinit(gpa);
        self.debug_unresolved_namespace_scopes.deinit(gpa);
        self.nav_map.deinit(gpa);
        self.uav_map.deinit(gpa);
        self.enum_tag_name_map.deinit(gpa);
        self.named_enum_map.deinit(gpa);
        self.type_map.deinit(gpa);
        self.builder.deinit();
        self.struct_field_map.deinit(gpa);
        self.* = undefined;
    }

    fn genErrorNameTable(o: *Object) Allocator.Error!void {
        // If o.error_name_table is null, then it was not referenced by any instructions.
        if (o.error_name_table == .none) return;

        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;

        const error_name_list = ip.global_error_set.getNamesFromMainThread();
        const llvm_errors = try zcu.gpa.alloc(Builder.Constant, 1 + error_name_list.len);
        defer zcu.gpa.free(llvm_errors);

        // TODO: Address space
        const slice_ty = Type.slice_const_u8_sentinel_0;
        const llvm_usize_ty = try o.lowerType(Type.usize);
        const llvm_slice_ty = try o.lowerType(slice_ty);
        const llvm_table_ty = try o.builder.arrayType(1 + error_name_list.len, llvm_slice_ty);

        llvm_errors[0] = try o.builder.undefConst(llvm_slice_ty);
        for (llvm_errors[1..], error_name_list) |*llvm_error, name| {
            const name_string = try o.builder.stringNull(name.toSlice(ip));
            const name_init = try o.builder.stringConst(name_string);
            const name_variable_index =
                try o.builder.addVariable(.empty, name_init.typeOf(&o.builder), .default);
            try name_variable_index.setInitializer(name_init, &o.builder);
            name_variable_index.setLinkage(.private, &o.builder);
            name_variable_index.setMutability(.constant, &o.builder);
            name_variable_index.setUnnamedAddr(.unnamed_addr, &o.builder);
            name_variable_index.setAlignment(comptime Builder.Alignment.fromByteUnits(1), &o.builder);

            llvm_error.* = try o.builder.structConst(llvm_slice_ty, &.{
                name_variable_index.toConst(&o.builder),
                try o.builder.intConst(llvm_usize_ty, name_string.slice(&o.builder).?.len - 1),
            });
        }

        const table_variable_index = try o.builder.addVariable(.empty, llvm_table_ty, .default);
        try table_variable_index.setInitializer(
            try o.builder.arrayConst(llvm_table_ty, llvm_errors),
            &o.builder,
        );
        table_variable_index.setLinkage(.private, &o.builder);
        table_variable_index.setMutability(.constant, &o.builder);
        table_variable_index.setUnnamedAddr(.unnamed_addr, &o.builder);
        table_variable_index.setAlignment(
            slice_ty.abiAlignment(zcu).toLlvm(),
            &o.builder,
        );

        try o.error_name_table.setInitializer(table_variable_index.toConst(&o.builder), &o.builder);
    }

    fn genCmpLtErrorsLenFunction(o: *Object) !void {
        // If there is no such function in the module, it means the source code does not need it.
        const name = o.builder.strtabStringIfExists(lt_errors_fn_name) orelse return;
        const llvm_fn = o.builder.getGlobal(name) orelse return;
        const errors_len = o.pt.zcu.intern_pool.global_error_set.getNamesFromMainThread().len;

        var wip = try Builder.WipFunction.init(&o.builder, .{
            .function = llvm_fn.ptrConst(&o.builder).kind.function,
            .strip = true,
        });
        defer wip.deinit();
        wip.cursor = .{ .block = try wip.block(0, "Entry") };

        // Example source of the following LLVM IR:
        // fn __zig_lt_errors_len(index: u16) bool {
        //     return index <= total_errors_len;
        // }

        const lhs = wip.arg(0);
        const rhs = try o.builder.intValue(try o.errorIntType(), errors_len);
        const is_lt = try wip.icmp(.ule, lhs, rhs, "");
        _ = try wip.ret(is_lt);
        try wip.finish();
    }

    fn genModuleLevelAssembly(object: *Object) !void {
        const writer = object.builder.setModuleAsm();
        for (object.pt.zcu.global_assembly.values()) |assembly| {
            try writer.print("{s}\n", .{assembly});
        }
        try object.builder.finishModuleAsm();
    }

    pub const EmitOptions = struct {
        pre_ir_path: ?[]const u8,
        pre_bc_path: ?[]const u8,
        bin_path: ?[*:0]const u8,
        asm_path: ?[*:0]const u8,
        post_ir_path: ?[*:0]const u8,
        post_bc_path: ?[*:0]const u8,

        is_debug: bool,
        is_small: bool,
        time_report: bool,
        sanitize_thread: bool,
        fuzz: bool,
        lto: Compilation.Config.LtoMode,
    };

    pub fn emit(o: *Object, options: EmitOptions) error{ LinkFailure, OutOfMemory }!void {
        const zcu = o.pt.zcu;
        const comp = zcu.comp;
        const diags = &comp.link_diags;

        {
            try o.genErrorNameTable();
            try o.genCmpLtErrorsLenFunction();
            try o.genModuleLevelAssembly();

            if (o.used.items.len > 0) {
                const array_llvm_ty = try o.builder.arrayType(o.used.items.len, .ptr);
                const init_val = try o.builder.arrayConst(array_llvm_ty, o.used.items);
                const compiler_used_variable = try o.builder.addVariable(
                    try o.builder.strtabString("llvm.used"),
                    array_llvm_ty,
                    .default,
                );
                compiler_used_variable.setLinkage(.appending, &o.builder);
                compiler_used_variable.setSection(try o.builder.string("llvm.metadata"), &o.builder);
                try compiler_used_variable.setInitializer(init_val, &o.builder);
            }

            if (!o.builder.strip) {
                {
                    var i: usize = 0;
                    while (i < o.debug_unresolved_namespace_scopes.count()) : (i += 1) {
                        const namespace_index = o.debug_unresolved_namespace_scopes.keys()[i];
                        const fwd_ref = o.debug_unresolved_namespace_scopes.values()[i];

                        const namespace = zcu.namespacePtr(namespace_index);
                        const debug_type = try o.lowerDebugType(Type.fromInterned(namespace.owner_type));

                        o.builder.debugForwardReferenceSetType(fwd_ref, debug_type);
                    }
                }

                o.builder.debugForwardReferenceSetType(
                    o.debug_enums_fwd_ref,
                    try o.builder.metadataTuple(o.debug_enums.items),
                );

                o.builder.debugForwardReferenceSetType(
                    o.debug_globals_fwd_ref,
                    try o.builder.metadataTuple(o.debug_globals.items),
                );
            }
        }

        {
            var module_flags = try std.ArrayList(Builder.Metadata).initCapacity(o.gpa, 7);
            defer module_flags.deinit();

            const behavior_error = try o.builder.metadataConstant(try o.builder.intConst(.i32, 1));
            const behavior_warning = try o.builder.metadataConstant(try o.builder.intConst(.i32, 2));
            const behavior_max = try o.builder.metadataConstant(try o.builder.intConst(.i32, 7));
            const behavior_min = try o.builder.metadataConstant(try o.builder.intConst(.i32, 8));

            const pic_level = target_util.picLevel(comp.root_mod.resolved_target.result);
            if (comp.root_mod.pic) {
                module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                    behavior_min,
                    try o.builder.metadataString("PIC Level"),
                    try o.builder.metadataConstant(try o.builder.intConst(.i32, pic_level)),
                ));
            }

            if (comp.config.pie) {
                module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                    behavior_max,
                    try o.builder.metadataString("PIE Level"),
                    try o.builder.metadataConstant(try o.builder.intConst(.i32, pic_level)),
                ));
            }

            if (comp.root_mod.code_model != .default) {
                module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                    behavior_error,
                    try o.builder.metadataString("Code Model"),
                    try o.builder.metadataConstant(try o.builder.intConst(.i32, @as(i32, switch (comp.root_mod.code_model) {
                        .tiny => 0,
                        .small => 1,
                        .kernel => 2,
                        .medium => 3,
                        .large => 4,
                        else => unreachable,
                    }))),
                ));
            }

            if (!o.builder.strip) {
                module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                    behavior_warning,
                    try o.builder.metadataString("Debug Info Version"),
                    try o.builder.metadataConstant(try o.builder.intConst(.i32, 3)),
                ));

                switch (comp.config.debug_format) {
                    .strip => unreachable,
                    .dwarf => |f| {
                        module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                            behavior_max,
                            try o.builder.metadataString("Dwarf Version"),
                            try o.builder.metadataConstant(try o.builder.intConst(.i32, 4)),
                        ));

                        if (f == .@"64") {
                            module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                                behavior_max,
                                try o.builder.metadataString("DWARF64"),
                                try o.builder.metadataConstant(.@"1"),
                            ));
                        }
                    },
                    .code_view => {
                        module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                            behavior_warning,
                            try o.builder.metadataString("CodeView"),
                            try o.builder.metadataConstant(.@"1"),
                        ));
                    },
                }
            }

            const target = comp.root_mod.resolved_target.result;
            if (target.os.tag == .windows and (target.cpu.arch == .x86_64 or target.cpu.arch == .x86)) {
                // Add the "RegCallv4" flag so that any functions using `x86_regcallcc` use regcall
                // v4, which is essentially a requirement on Windows. See corresponding logic in
                // `toLlvmCallConvTag`.
                module_flags.appendAssumeCapacity(try o.builder.metadataModuleFlag(
                    behavior_max,
                    try o.builder.metadataString("RegCallv4"),
                    try o.builder.metadataConstant(.@"1"),
                ));
            }

            try o.builder.metadataNamed(try o.builder.metadataString("llvm.module.flags"), module_flags.items);
        }

        const target_triple_sentinel =
            try o.gpa.dupeZ(u8, o.builder.target_triple.slice(&o.builder).?);
        defer o.gpa.free(target_triple_sentinel);

        const emit_asm_msg = options.asm_path orelse "(none)";
        const emit_bin_msg = options.bin_path orelse "(none)";
        const post_llvm_ir_msg = options.post_ir_path orelse "(none)";
        const post_llvm_bc_msg = options.post_bc_path orelse "(none)";
        log.debug("emit LLVM object asm={s} bin={s} ir={s} bc={s}", .{
            emit_asm_msg, emit_bin_msg, post_llvm_ir_msg, post_llvm_bc_msg,
        });

        const context, const module = emit: {
            if (options.pre_ir_path) |path| {
                if (std.mem.eql(u8, path, "-")) {
                    o.builder.dump();
                } else {
                    _ = try o.builder.printToFile(path);
                }
            }

            const bitcode = try o.builder.toBitcode(o.gpa, .{
                .name = "zig",
                .version = build_options.semver,
            });
            defer o.gpa.free(bitcode);
            o.builder.clearAndFree();

            if (options.pre_bc_path) |path| {
                var file = std.fs.cwd().createFile(path, .{}) catch |err|
                    return diags.fail("failed to create '{s}': {s}", .{ path, @errorName(err) });
                defer file.close();

                const ptr: [*]const u8 = @ptrCast(bitcode.ptr);
                file.writeAll(ptr[0..(bitcode.len * 4)]) catch |err|
                    return diags.fail("failed to write to '{s}': {s}", .{ path, @errorName(err) });
            }

            if (options.asm_path == null and options.bin_path == null and
                options.post_ir_path == null and options.post_bc_path == null) return;

            if (options.post_bc_path) |path| {
                var file = std.fs.cwd().createFileZ(path, .{}) catch |err|
                    return diags.fail("failed to create '{s}': {s}", .{ path, @errorName(err) });
                defer file.close();

                const ptr: [*]const u8 = @ptrCast(bitcode.ptr);
                file.writeAll(ptr[0..(bitcode.len * 4)]) catch |err|
                    return diags.fail("failed to write to '{s}': {s}", .{ path, @errorName(err) });
            }

            if (!build_options.have_llvm or !comp.config.use_lib_llvm) {
                return diags.fail("emitting without libllvm not implemented", .{});
            }

            initializeLLVMTarget(comp.root_mod.resolved_target.result.cpu.arch);

            const context: *llvm.Context = llvm.Context.create();
            errdefer context.dispose();

            const bitcode_memory_buffer = llvm.MemoryBuffer.createMemoryBufferWithMemoryRange(
                @ptrCast(bitcode.ptr),
                bitcode.len * 4,
                "BitcodeBuffer",
                llvm.Bool.False,
            );
            defer bitcode_memory_buffer.dispose();

            context.enableBrokenDebugInfoCheck();

            var module: *llvm.Module = undefined;
            if (context.parseBitcodeInContext2(bitcode_memory_buffer, &module).toBool() or context.getBrokenDebugInfo()) {
                return diags.fail("Failed to parse bitcode", .{});
            }
            break :emit .{ context, module };
        };
        defer context.dispose();

        var target: *llvm.Target = undefined;
        var error_message: [*:0]const u8 = undefined;
        if (llvm.Target.getFromTriple(target_triple_sentinel, &target, &error_message).toBool()) {
            defer llvm.disposeMessage(error_message);
            return diags.fail("LLVM failed to parse '{s}': {s}", .{ target_triple_sentinel, error_message });
        }

        const optimize_mode = comp.root_mod.optimize_mode;

        const opt_level: llvm.CodeGenOptLevel = if (optimize_mode == .Debug)
            .None
        else
            .Aggressive;

        const reloc_mode: llvm.RelocMode = if (comp.root_mod.pic)
            .PIC
        else if (comp.config.link_mode == .dynamic)
            llvm.RelocMode.DynamicNoPIC
        else
            .Static;

        const code_model: llvm.CodeModel = switch (comp.root_mod.code_model) {
            .default => .Default,
            .tiny => .Tiny,
            .small => .Small,
            .kernel => .Kernel,
            .medium => .Medium,
            .large => .Large,
        };

        const float_abi: llvm.TargetMachine.FloatABI = if (comp.root_mod.resolved_target.result.abi.float() == .hard)
            .Hard
        else
            .Soft;

        var target_machine = llvm.TargetMachine.create(
            target,
            target_triple_sentinel,
            if (comp.root_mod.resolved_target.result.cpu.model.llvm_name) |s| s.ptr else null,
            comp.root_mod.resolved_target.llvm_cpu_features.?,
            opt_level,
            reloc_mode,
            code_model,
            comp.function_sections,
            comp.data_sections,
            float_abi,
            if (target_util.llvmMachineAbi(comp.root_mod.resolved_target.result)) |s| s.ptr else null,
        );
        errdefer target_machine.dispose();

        if (comp.llvm_opt_bisect_limit >= 0) {
            context.setOptBisectLimit(comp.llvm_opt_bisect_limit);
        }

        // Unfortunately, LLVM shits the bed when we ask for both binary and assembly.
        // So we call the entire pipeline multiple times if this is requested.
        // var error_message: [*:0]const u8 = undefined;
        var lowered_options: llvm.TargetMachine.EmitOptions = .{
            .is_debug = options.is_debug,
            .is_small = options.is_small,
            .time_report = options.time_report,
            .tsan = options.sanitize_thread,
            .lto = options.lto != .none,
            // https://github.com/ziglang/zig/issues/21215
            .allow_fast_isel = !comp.root_mod.resolved_target.result.cpu.arch.isMIPS(),
            .asm_filename = null,
            .bin_filename = options.bin_path,
            .llvm_ir_filename = options.post_ir_path,
            .bitcode_filename = null,

            // `.coverage` value is only used when `.sancov` is enabled.
            .sancov = options.fuzz or comp.config.san_cov_trace_pc_guard,
            .coverage = .{
                .CoverageType = .Edge,
                // Works in tandem with Inline8bitCounters or InlineBoolFlag.
                // Zig does not yet implement its own version of this but it
                // needs to for better fuzzing logic.
                .IndirectCalls = false,
                .TraceBB = false,
                .TraceCmp = options.fuzz,
                .TraceDiv = false,
                .TraceGep = false,
                .Use8bitCounters = false,
                .TracePC = false,
                .TracePCGuard = comp.config.san_cov_trace_pc_guard,
                // Zig emits its own inline 8-bit counters instrumentation.
                .Inline8bitCounters = false,
                .InlineBoolFlag = false,
                // Zig emits its own PC table instrumentation.
                .PCTable = false,
                .NoPrune = false,
                // Workaround for https://github.com/llvm/llvm-project/pull/106464
                .StackDepth = true,
                .TraceLoads = false,
                .TraceStores = false,
                .CollectControlFlow = false,
            },
        };
        if (options.asm_path != null and options.bin_path != null) {
            if (target_machine.emitToFile(module, &error_message, &lowered_options)) {
                defer llvm.disposeMessage(error_message);
                return diags.fail("LLVM failed to emit bin={s} ir={s}: {s}", .{
                    emit_bin_msg, post_llvm_ir_msg, error_message,
                });
            }
            lowered_options.bin_filename = null;
            lowered_options.llvm_ir_filename = null;
        }

        lowered_options.asm_filename = options.asm_path;
        if (target_machine.emitToFile(module, &error_message, &lowered_options)) {
            defer llvm.disposeMessage(error_message);
            return diags.fail("LLVM failed to emit asm={s} bin={s} ir={s} bc={s}: {s}", .{
                emit_asm_msg, emit_bin_msg, post_llvm_ir_msg, post_llvm_bc_msg, error_message,
            });
        }
    }

    pub fn updateFunc(
        o: *Object,
        pt: Zcu.PerThread,
        func_index: InternPool.Index,
        air: Air,
        liveness: Liveness,
    ) !void {
        assert(std.meta.eql(pt, o.pt));
        const zcu = pt.zcu;
        const comp = zcu.comp;
        const ip = &zcu.intern_pool;
        const func = zcu.funcInfo(func_index);
        const nav = ip.getNav(func.owner_nav);
        const file_scope = zcu.navFileScopeIndex(func.owner_nav);
        const owner_mod = zcu.fileByIndex(file_scope).mod;
        const fn_ty = Type.fromInterned(func.ty);
        const fn_info = zcu.typeToFunc(fn_ty).?;
        const target = owner_mod.resolved_target.result;

        var ng: NavGen = .{
            .object = o,
            .nav_index = func.owner_nav,
            .err_msg = null,
        };

        const function_index = try o.resolveLlvmFunction(func.owner_nav);

        var attributes = try function_index.ptrConst(&o.builder).attributes.toWip(&o.builder);
        defer attributes.deinit(&o.builder);

        const func_analysis = func.analysisUnordered(ip);
        if (func_analysis.is_noinline) {
            try attributes.addFnAttr(.@"noinline", &o.builder);
        } else {
            _ = try attributes.removeFnAttr(.@"noinline");
        }

        if (func_analysis.branch_hint == .cold) {
            try attributes.addFnAttr(.cold, &o.builder);
        } else {
            _ = try attributes.removeFnAttr(.cold);
        }

        if (owner_mod.sanitize_thread and !func_analysis.disable_instrumentation) {
            try attributes.addFnAttr(.sanitize_thread, &o.builder);
        } else {
            _ = try attributes.removeFnAttr(.sanitize_thread);
        }
        const is_naked = fn_info.cc == .naked;
        if (owner_mod.fuzz and !func_analysis.disable_instrumentation and !is_naked) {
            try attributes.addFnAttr(.optforfuzzing, &o.builder);
            _ = try attributes.removeFnAttr(.skipprofile);
            _ = try attributes.removeFnAttr(.nosanitize_coverage);
        } else {
            _ = try attributes.removeFnAttr(.optforfuzzing);
            try attributes.addFnAttr(.skipprofile, &o.builder);
            try attributes.addFnAttr(.nosanitize_coverage, &o.builder);
        }

        const disable_intrinsics = func_analysis.disable_intrinsics or owner_mod.no_builtin;
        if (disable_intrinsics) {
            // The intent here is for compiler-rt and libc functions to not generate
            // infinite recursion. For example, if we are compiling the memcpy function,
            // and llvm detects that the body is equivalent to memcpy, it may replace the
            // body of memcpy with a call to memcpy, which would then cause a stack
            // overflow instead of performing memcpy.
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("no-builtins"),
                .value = .empty,
            } }, &o.builder);
        }

        // TODO: disable this if safety is off for the function scope
        const ssp_buf_size = owner_mod.stack_protector;
        if (ssp_buf_size != 0) {
            try attributes.addFnAttr(.sspstrong, &o.builder);
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("stack-protector-buffer-size"),
                .value = try o.builder.fmt("{d}", .{ssp_buf_size}),
            } }, &o.builder);
        }

        // TODO: disable this if safety is off for the function scope
        if (owner_mod.stack_check) {
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("probe-stack"),
                .value = try o.builder.string("__zig_probe_stack"),
            } }, &o.builder);
        } else if (target.os.tag == .uefi) {
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("no-stack-arg-probe"),
                .value = .empty,
            } }, &o.builder);
        }

        if (nav.status.fully_resolved.@"linksection".toSlice(ip)) |section|
            function_index.setSection(try o.builder.string(section), &o.builder);

        var deinit_wip = true;
        var wip = try Builder.WipFunction.init(&o.builder, .{
            .function = function_index,
            .strip = owner_mod.strip,
        });
        defer if (deinit_wip) wip.deinit();
        wip.cursor = .{ .block = try wip.block(0, "Entry") };

        var llvm_arg_i: u32 = 0;

        // This gets the LLVM values from the function and stores them in `ng.args`.
        const sret = firstParamSRet(fn_info, zcu, target);
        const ret_ptr: Builder.Value = if (sret) param: {
            const param = wip.arg(llvm_arg_i);
            llvm_arg_i += 1;
            break :param param;
        } else .none;

        if (ccAbiPromoteInt(fn_info.cc, zcu, Type.fromInterned(fn_info.return_type))) |s| switch (s) {
            .signed => try attributes.addRetAttr(.signext, &o.builder),
            .unsigned => try attributes.addRetAttr(.zeroext, &o.builder),
        };

        const err_return_tracing = fn_info.cc == .auto and comp.config.any_error_tracing;

        const err_ret_trace: Builder.Value = if (err_return_tracing) param: {
            const param = wip.arg(llvm_arg_i);
            llvm_arg_i += 1;
            break :param param;
        } else .none;

        // This is the list of args we will use that correspond directly to the AIR arg
        // instructions. Depending on the calling convention, this list is not necessarily
        // a bijection with the actual LLVM parameters of the function.
        const gpa = o.gpa;
        var args: std.ArrayListUnmanaged(Builder.Value) = .empty;
        defer args.deinit(gpa);

        {
            var it = iterateParamTypes(o, fn_info);
            while (try it.next()) |lowering| {
                try args.ensureUnusedCapacity(gpa, 1);

                switch (lowering) {
                    .no_bits => continue,
                    .byval => {
                        assert(!it.byval_attr);
                        const param_index = it.zig_index - 1;
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[param_index]);
                        const param = wip.arg(llvm_arg_i);

                        if (isByRef(param_ty, zcu)) {
                            const alignment = param_ty.abiAlignment(zcu).toLlvm();
                            const param_llvm_ty = param.typeOfWip(&wip);
                            const arg_ptr = try buildAllocaInner(&wip, param_llvm_ty, alignment, target);
                            _ = try wip.store(.normal, param, arg_ptr, alignment);
                            args.appendAssumeCapacity(arg_ptr);
                        } else {
                            args.appendAssumeCapacity(param);

                            try o.addByValParamAttrs(&attributes, param_ty, param_index, fn_info, llvm_arg_i);
                        }
                        llvm_arg_i += 1;
                    },
                    .byref => {
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                        const param_llvm_ty = try o.lowerType(param_ty);
                        const param = wip.arg(llvm_arg_i);
                        const alignment = param_ty.abiAlignment(zcu).toLlvm();

                        try o.addByRefParamAttrs(&attributes, llvm_arg_i, alignment, it.byval_attr, param_llvm_ty);
                        llvm_arg_i += 1;

                        if (isByRef(param_ty, zcu)) {
                            args.appendAssumeCapacity(param);
                        } else {
                            args.appendAssumeCapacity(try wip.load(.normal, param_llvm_ty, param, alignment, ""));
                        }
                    },
                    .byref_mut => {
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                        const param_llvm_ty = try o.lowerType(param_ty);
                        const param = wip.arg(llvm_arg_i);
                        const alignment = param_ty.abiAlignment(zcu).toLlvm();

                        try attributes.addParamAttr(llvm_arg_i, .noundef, &o.builder);
                        llvm_arg_i += 1;

                        if (isByRef(param_ty, zcu)) {
                            args.appendAssumeCapacity(param);
                        } else {
                            args.appendAssumeCapacity(try wip.load(.normal, param_llvm_ty, param, alignment, ""));
                        }
                    },
                    .abi_sized_int => {
                        assert(!it.byval_attr);
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                        const param = wip.arg(llvm_arg_i);
                        llvm_arg_i += 1;

                        const param_llvm_ty = try o.lowerType(param_ty);
                        const alignment = param_ty.abiAlignment(zcu).toLlvm();
                        const arg_ptr = try buildAllocaInner(&wip, param_llvm_ty, alignment, target);
                        _ = try wip.store(.normal, param, arg_ptr, alignment);

                        args.appendAssumeCapacity(if (isByRef(param_ty, zcu))
                            arg_ptr
                        else
                            try wip.load(.normal, param_llvm_ty, arg_ptr, alignment, ""));
                    },
                    .slice => {
                        assert(!it.byval_attr);
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                        const ptr_info = param_ty.ptrInfo(zcu);

                        if (math.cast(u5, it.zig_index - 1)) |i| {
                            if (@as(u1, @truncate(fn_info.noalias_bits >> i)) != 0) {
                                try attributes.addParamAttr(llvm_arg_i, .@"noalias", &o.builder);
                            }
                        }
                        if (param_ty.zigTypeTag(zcu) != .optional) {
                            try attributes.addParamAttr(llvm_arg_i, .nonnull, &o.builder);
                        }
                        if (ptr_info.flags.is_const) {
                            try attributes.addParamAttr(llvm_arg_i, .readonly, &o.builder);
                        }
                        const elem_align = (if (ptr_info.flags.alignment != .none)
                            @as(InternPool.Alignment, ptr_info.flags.alignment)
                        else
                            Type.fromInterned(ptr_info.child).abiAlignment(zcu).max(.@"1")).toLlvm();
                        try attributes.addParamAttr(llvm_arg_i, .{ .@"align" = elem_align }, &o.builder);
                        const ptr_param = wip.arg(llvm_arg_i);
                        llvm_arg_i += 1;
                        const len_param = wip.arg(llvm_arg_i);
                        llvm_arg_i += 1;

                        const slice_llvm_ty = try o.lowerType(param_ty);
                        args.appendAssumeCapacity(
                            try wip.buildAggregate(slice_llvm_ty, &.{ ptr_param, len_param }, ""),
                        );
                    },
                    .multiple_llvm_types => {
                        assert(!it.byval_attr);
                        const field_types = it.types_buffer[0..it.types_len];
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                        const param_llvm_ty = try o.lowerType(param_ty);
                        const param_alignment = param_ty.abiAlignment(zcu).toLlvm();
                        const arg_ptr = try buildAllocaInner(&wip, param_llvm_ty, param_alignment, target);
                        const llvm_ty = try o.builder.structType(.normal, field_types);
                        for (0..field_types.len) |field_i| {
                            const param = wip.arg(llvm_arg_i);
                            llvm_arg_i += 1;
                            const field_ptr = try wip.gepStruct(llvm_ty, arg_ptr, field_i, "");
                            const alignment =
                                Builder.Alignment.fromByteUnits(@divExact(target.ptrBitWidth(), 8));
                            _ = try wip.store(.normal, param, field_ptr, alignment);
                        }

                        const is_by_ref = isByRef(param_ty, zcu);
                        args.appendAssumeCapacity(if (is_by_ref)
                            arg_ptr
                        else
                            try wip.load(.normal, param_llvm_ty, arg_ptr, param_alignment, ""));
                    },
                    .float_array => {
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                        const param_llvm_ty = try o.lowerType(param_ty);
                        const param = wip.arg(llvm_arg_i);
                        llvm_arg_i += 1;

                        const alignment = param_ty.abiAlignment(zcu).toLlvm();
                        const arg_ptr = try buildAllocaInner(&wip, param_llvm_ty, alignment, target);
                        _ = try wip.store(.normal, param, arg_ptr, alignment);

                        args.appendAssumeCapacity(if (isByRef(param_ty, zcu))
                            arg_ptr
                        else
                            try wip.load(.normal, param_llvm_ty, arg_ptr, alignment, ""));
                    },
                    .i32_array, .i64_array => {
                        const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                        const param_llvm_ty = try o.lowerType(param_ty);
                        const param = wip.arg(llvm_arg_i);
                        llvm_arg_i += 1;

                        const alignment = param_ty.abiAlignment(zcu).toLlvm();
                        const arg_ptr = try buildAllocaInner(&wip, param.typeOfWip(&wip), alignment, target);
                        _ = try wip.store(.normal, param, arg_ptr, alignment);

                        args.appendAssumeCapacity(if (isByRef(param_ty, zcu))
                            arg_ptr
                        else
                            try wip.load(.normal, param_llvm_ty, arg_ptr, alignment, ""));
                    },
                }
            }
        }

        function_index.setAttributes(try attributes.finish(&o.builder), &o.builder);

        const file, const subprogram = if (!wip.strip) debug_info: {
            const file = try o.getDebugFile(file_scope);

            const line_number = zcu.navSrcLine(func.owner_nav) + 1;
            const is_internal_linkage = ip.indexToKey(nav.status.fully_resolved.val) != .@"extern";
            const debug_decl_type = try o.lowerDebugType(fn_ty);

            const subprogram = try o.builder.debugSubprogram(
                file,
                try o.builder.metadataString(nav.name.toSlice(ip)),
                try o.builder.metadataStringFromStrtabString(function_index.name(&o.builder)),
                line_number,
                line_number + func.lbrace_line,
                debug_decl_type,
                .{
                    .di_flags = .{
                        .StaticMember = true,
                        .NoReturn = fn_info.return_type == .noreturn_type,
                    },
                    .sp_flags = .{
                        .Optimized = owner_mod.optimize_mode != .Debug,
                        .Definition = true,
                        .LocalToUnit = is_internal_linkage,
                    },
                },
                o.debug_compile_unit,
            );
            function_index.setSubprogram(subprogram, &o.builder);
            break :debug_info .{ file, subprogram };
        } else .{.none} ** 2;

        const fuzz: ?FuncGen.Fuzz = f: {
            if (!owner_mod.fuzz) break :f null;
            if (func_analysis.disable_instrumentation) break :f null;
            if (is_naked) break :f null;
            if (comp.config.san_cov_trace_pc_guard) break :f null;

            // The void type used here is a placeholder to be replaced with an
            // array of the appropriate size after the POI count is known.

            // Due to error "members of llvm.compiler.used must be named", this global needs a name.
            const anon_name = try o.builder.strtabStringFmt("__sancov_gen_.{d}", .{o.used.items.len});
            const counters_variable = try o.builder.addVariable(anon_name, .void, .default);
            try o.used.append(gpa, counters_variable.toConst(&o.builder));
            counters_variable.setLinkage(.private, &o.builder);
            counters_variable.setAlignment(comptime Builder.Alignment.fromByteUnits(1), &o.builder);
            counters_variable.setSection(try o.builder.string("__sancov_cntrs"), &o.builder);

            break :f .{
                .counters_variable = counters_variable,
                .pcs = .{},
            };
        };

        var fg: FuncGen = .{
            .gpa = gpa,
            .air = air,
            .liveness = liveness,
            .ng = &ng,
            .wip = wip,
            .is_naked = fn_info.cc == .naked,
            .fuzz = fuzz,
            .ret_ptr = ret_ptr,
            .args = args.items,
            .arg_index = 0,
            .arg_inline_index = 0,
            .func_inst_table = .{},
            .blocks = .{},
            .loops = .{},
            .switch_dispatch_info = .{},
            .sync_scope = if (owner_mod.single_threaded) .singlethread else .system,
            .file = file,
            .scope = subprogram,
            .base_line = zcu.navSrcLine(func.owner_nav),
            .prev_dbg_line = 0,
            .prev_dbg_column = 0,
            .err_ret_trace = err_ret_trace,
            .disable_intrinsics = disable_intrinsics,
        };
        defer fg.deinit();
        deinit_wip = false;

        fg.genBody(air.getMainBody(), .poi) catch |err| switch (err) {
            error.CodegenFail => {
                try zcu.failed_codegen.put(gpa, func.owner_nav, ng.err_msg.?);
                ng.err_msg = null;
                return;
            },
            else => |e| return e,
        };

        if (fg.fuzz) |*f| {
            {
                const array_llvm_ty = try o.builder.arrayType(f.pcs.items.len, .i8);
                f.counters_variable.ptrConst(&o.builder).global.ptr(&o.builder).type = array_llvm_ty;
                const zero_init = try o.builder.zeroInitConst(array_llvm_ty);
                try f.counters_variable.setInitializer(zero_init, &o.builder);
            }

            const array_llvm_ty = try o.builder.arrayType(f.pcs.items.len, .ptr);
            const init_val = try o.builder.arrayConst(array_llvm_ty, f.pcs.items);
            // Due to error "members of llvm.compiler.used must be named", this global needs a name.
            const anon_name = try o.builder.strtabStringFmt("__sancov_gen_.{d}", .{o.used.items.len});
            const pcs_variable = try o.builder.addVariable(anon_name, array_llvm_ty, .default);
            try o.used.append(gpa, pcs_variable.toConst(&o.builder));
            pcs_variable.setLinkage(.private, &o.builder);
            pcs_variable.setMutability(.constant, &o.builder);
            pcs_variable.setAlignment(Type.usize.abiAlignment(zcu).toLlvm(), &o.builder);
            pcs_variable.setSection(try o.builder.string("__sancov_pcs1"), &o.builder);
            try pcs_variable.setInitializer(init_val, &o.builder);
        }

        try fg.wip.finish();
    }

    pub fn updateNav(self: *Object, pt: Zcu.PerThread, nav_index: InternPool.Nav.Index) !void {
        assert(std.meta.eql(pt, self.pt));
        var ng: NavGen = .{
            .object = self,
            .nav_index = nav_index,
            .err_msg = null,
        };
        ng.genDecl() catch |err| switch (err) {
            error.CodegenFail => {
                try pt.zcu.failed_codegen.put(pt.zcu.gpa, nav_index, ng.err_msg.?);
                ng.err_msg = null;
                return;
            },
            else => |e| return e,
        };
    }

    pub fn updateExports(
        self: *Object,
        pt: Zcu.PerThread,
        exported: Zcu.Exported,
        export_indices: []const Zcu.Export.Index,
    ) link.File.UpdateExportsError!void {
        assert(std.meta.eql(pt, self.pt));
        const zcu = pt.zcu;
        const nav_index = switch (exported) {
            .nav => |nav| nav,
            .uav => |uav| return updateExportedValue(self, zcu, uav, export_indices),
        };
        const ip = &zcu.intern_pool;
        const global_index = self.nav_map.get(nav_index).?;
        const comp = zcu.comp;

        if (export_indices.len != 0) {
            return updateExportedGlobal(self, zcu, global_index, export_indices);
        } else {
            const fqn = try self.builder.strtabString(ip.getNav(nav_index).fqn.toSlice(ip));
            try global_index.rename(fqn, &self.builder);
            global_index.setLinkage(.internal, &self.builder);
            if (comp.config.dll_export_fns)
                global_index.setDllStorageClass(.default, &self.builder);
            global_index.setUnnamedAddr(.unnamed_addr, &self.builder);
        }
    }

    fn updateExportedValue(
        o: *Object,
        zcu: *Zcu,
        exported_value: InternPool.Index,
        export_indices: []const Zcu.Export.Index,
    ) link.File.UpdateExportsError!void {
        const gpa = zcu.gpa;
        const ip = &zcu.intern_pool;
        const main_exp_name = try o.builder.strtabString(export_indices[0].ptr(zcu).opts.name.toSlice(ip));
        const global_index = i: {
            const gop = try o.uav_map.getOrPut(gpa, exported_value);
            if (gop.found_existing) {
                const global_index = gop.value_ptr.*;
                try global_index.rename(main_exp_name, &o.builder);
                break :i global_index;
            }
            const llvm_addr_space = toLlvmAddressSpace(.generic, o.target);
            const variable_index = try o.builder.addVariable(
                main_exp_name,
                try o.lowerType(Type.fromInterned(ip.typeOf(exported_value))),
                llvm_addr_space,
            );
            const global_index = variable_index.ptrConst(&o.builder).global;
            gop.value_ptr.* = global_index;
            // This line invalidates `gop`.
            const init_val = o.lowerValue(exported_value) catch |err| switch (err) {
                error.OutOfMemory => return error.OutOfMemory,
                error.CodegenFail => return error.AnalysisFail,
            };
            try variable_index.setInitializer(init_val, &o.builder);
            break :i global_index;
        };
        return updateExportedGlobal(o, zcu, global_index, export_indices);
    }

    fn updateExportedGlobal(
        o: *Object,
        zcu: *Zcu,
        global_index: Builder.Global.Index,
        export_indices: []const Zcu.Export.Index,
    ) link.File.UpdateExportsError!void {
        const comp = zcu.comp;
        const ip = &zcu.intern_pool;
        const first_export = export_indices[0].ptr(zcu);

        // We will rename this global to have a name matching `first_export`.
        // Successive exports become aliases.
        // If the first export name already exists, then there is a corresponding
        // extern global - we replace it with this global.
        const first_exp_name = try o.builder.strtabString(first_export.opts.name.toSlice(ip));
        if (o.builder.getGlobal(first_exp_name)) |other_global| replace: {
            if (other_global.toConst().getBase(&o.builder) == global_index.toConst().getBase(&o.builder)) {
                break :replace; // this global already has the name we want
            }
            try global_index.takeName(other_global, &o.builder);
            try other_global.replace(global_index, &o.builder);
            // Problem: now we need to replace in the decl_map that
            // the extern decl index points to this new global. However we don't
            // know the decl index.
            // Even if we did, a future incremental update to the extern would then
            // treat the LLVM global as an extern rather than an export, so it would
            // need a way to check that.
            // This is a TODO that needs to be solved when making
            // the LLVM backend support incremental compilation.
        } else {
            try global_index.rename(first_exp_name, &o.builder);
        }

        global_index.setUnnamedAddr(.default, &o.builder);
        if (comp.config.dll_export_fns)
            global_index.setDllStorageClass(.dllexport, &o.builder);
        global_index.setLinkage(switch (first_export.opts.linkage) {
            .internal => unreachable,
            .strong => .external,
            .weak => .weak_odr,
            .link_once => .linkonce_odr,
        }, &o.builder);
        global_index.setVisibility(switch (first_export.opts.visibility) {
            .default => .default,
            .hidden => .hidden,
            .protected => .protected,
        }, &o.builder);
        if (first_export.opts.section.toSlice(ip)) |section|
            switch (global_index.ptrConst(&o.builder).kind) {
                .variable => |impl_index| impl_index.setSection(
                    try o.builder.string(section),
                    &o.builder,
                ),
                .function => unreachable,
                .alias => unreachable,
                .replaced => unreachable,
            };

        // If a Decl is exported more than one time (which is rare),
        // we add aliases for all but the first export.
        // TODO LLVM C API does not support deleting aliases.
        // The planned solution to this is https://github.com/ziglang/zig/issues/13265
        // Until then we iterate over existing aliases and make them point
        // to the correct decl, or otherwise add a new alias. Old aliases are leaked.
        for (export_indices[1..]) |export_idx| {
            const exp = export_idx.ptr(zcu);
            const exp_name = try o.builder.strtabString(exp.opts.name.toSlice(ip));
            if (o.builder.getGlobal(exp_name)) |global| {
                switch (global.ptrConst(&o.builder).kind) {
                    .alias => |alias| {
                        alias.setAliasee(global_index.toConst(), &o.builder);
                        continue;
                    },
                    .variable, .function => {
                        // This existing global is an `extern` corresponding to this export.
                        // Replace it with the global being exported.
                        // This existing global must be replaced with the alias.
                        try global.rename(.empty, &o.builder);
                        try global.replace(global_index, &o.builder);
                    },
                    .replaced => unreachable,
                }
            }
            const alias_index = try o.builder.addAlias(
                .empty,
                global_index.typeOf(&o.builder),
                .default,
                global_index.toConst(),
            );
            try alias_index.rename(exp_name, &o.builder);
        }
    }

    fn getDebugFile(o: *Object, file_index: Zcu.File.Index) Allocator.Error!Builder.Metadata {
        const gpa = o.gpa;
        const gop = try o.debug_file_map.getOrPut(gpa, file_index);
        errdefer assert(o.debug_file_map.remove(file_index));
        if (gop.found_existing) return gop.value_ptr.*;
        const file = o.pt.zcu.fileByIndex(file_index);
        gop.value_ptr.* = try o.builder.debugFile(
            try o.builder.metadataString(std.fs.path.basename(file.sub_file_path)),
            dir_path: {
                const sub_path = std.fs.path.dirname(file.sub_file_path) orelse "";
                const dir_path = try file.mod.root.joinString(gpa, sub_path);
                defer gpa.free(dir_path);
                if (std.fs.path.isAbsolute(dir_path))
                    break :dir_path try o.builder.metadataString(dir_path);
                var abs_buffer: [std.fs.max_path_bytes]u8 = undefined;
                const abs_path = std.fs.realpath(dir_path, &abs_buffer) catch
                    break :dir_path try o.builder.metadataString(dir_path);
                break :dir_path try o.builder.metadataString(abs_path);
            },
        );
        return gop.value_ptr.*;
    }

    pub fn lowerDebugType(
        o: *Object,
        ty: Type,
    ) Allocator.Error!Builder.Metadata {
        assert(!o.builder.strip);

        const gpa = o.gpa;
        const target = o.target;
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;

        if (o.debug_type_map.get(ty)) |debug_type| return debug_type;

        switch (ty.zigTypeTag(zcu)) {
            .void,
            .noreturn,
            => {
                const debug_void_type = try o.builder.debugSignedType(
                    try o.builder.metadataString("void"),
                    0,
                );
                try o.debug_type_map.put(gpa, ty, debug_void_type);
                return debug_void_type;
            },
            .int => {
                const info = ty.intInfo(zcu);
                assert(info.bits != 0);
                const name = try o.allocTypeName(ty);
                defer gpa.free(name);
                const builder_name = try o.builder.metadataString(name);
                const debug_bits = ty.abiSize(zcu) * 8; // lldb cannot handle non-byte sized types
                const debug_int_type = switch (info.signedness) {
                    .signed => try o.builder.debugSignedType(builder_name, debug_bits),
                    .unsigned => try o.builder.debugUnsignedType(builder_name, debug_bits),
                };
                try o.debug_type_map.put(gpa, ty, debug_int_type);
                return debug_int_type;
            },
            .@"enum" => {
                if (!ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                    const debug_enum_type = try o.makeEmptyNamespaceDebugType(ty);
                    try o.debug_type_map.put(gpa, ty, debug_enum_type);
                    return debug_enum_type;
                }

                const enum_type = ip.loadEnumType(ty.toIntern());
                const enumerators = try gpa.alloc(Builder.Metadata, enum_type.names.len);
                defer gpa.free(enumerators);

                const int_ty = Type.fromInterned(enum_type.tag_ty);
                const int_info = ty.intInfo(zcu);
                assert(int_info.bits != 0);

                for (enum_type.names.get(ip), 0..) |field_name_ip, i| {
                    var bigint_space: Value.BigIntSpace = undefined;
                    const bigint = if (enum_type.values.len != 0)
                        Value.fromInterned(enum_type.values.get(ip)[i]).toBigInt(&bigint_space, zcu)
                    else
                        std.math.big.int.Mutable.init(&bigint_space.limbs, i).toConst();

                    enumerators[i] = try o.builder.debugEnumerator(
                        try o.builder.metadataString(field_name_ip.toSlice(ip)),
                        int_info.signedness == .unsigned,
                        int_info.bits,
                        bigint,
                    );
                }

                const file = try o.getDebugFile(ty.typeDeclInstAllowGeneratedTag(zcu).?.resolveFile(ip));
                const scope = if (ty.getParentNamespace(zcu).unwrap()) |parent_namespace|
                    try o.namespaceToDebugScope(parent_namespace)
                else
                    file;

                const name = try o.allocTypeName(ty);
                defer gpa.free(name);

                const debug_enum_type = try o.builder.debugEnumerationType(
                    try o.builder.metadataString(name),
                    file,
                    scope,
                    ty.typeDeclSrcLine(zcu).? + 1, // Line
                    try o.lowerDebugType(int_ty),
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(enumerators),
                );

                try o.debug_type_map.put(gpa, ty, debug_enum_type);
                try o.debug_enums.append(gpa, debug_enum_type);
                return debug_enum_type;
            },
            .float => {
                const bits = ty.floatBits(target);
                const name = try o.allocTypeName(ty);
                defer gpa.free(name);
                const debug_float_type = try o.builder.debugFloatType(
                    try o.builder.metadataString(name),
                    bits,
                );
                try o.debug_type_map.put(gpa, ty, debug_float_type);
                return debug_float_type;
            },
            .bool => {
                const debug_bool_type = try o.builder.debugBoolType(
                    try o.builder.metadataString("bool"),
                    8, // lldb cannot handle non-byte sized types
                );
                try o.debug_type_map.put(gpa, ty, debug_bool_type);
                return debug_bool_type;
            },
            .pointer => {
                // Normalize everything that the debug info does not represent.
                const ptr_info = ty.ptrInfo(zcu);

                if (ptr_info.sentinel != .none or
                    ptr_info.flags.address_space != .generic or
                    ptr_info.packed_offset.bit_offset != 0 or
                    ptr_info.packed_offset.host_size != 0 or
                    ptr_info.flags.vector_index != .none or
                    ptr_info.flags.is_allowzero or
                    ptr_info.flags.is_const or
                    ptr_info.flags.is_volatile or
                    ptr_info.flags.size == .many or ptr_info.flags.size == .c or
                    !Type.fromInterned(ptr_info.child).hasRuntimeBitsIgnoreComptime(zcu))
                {
                    const bland_ptr_ty = try pt.ptrType(.{
                        .child = if (!Type.fromInterned(ptr_info.child).hasRuntimeBitsIgnoreComptime(zcu))
                            .anyopaque_type
                        else
                            ptr_info.child,
                        .flags = .{
                            .alignment = ptr_info.flags.alignment,
                            .size = switch (ptr_info.flags.size) {
                                .many, .c, .one => .one,
                                .slice => .slice,
                            },
                        },
                    });
                    const debug_ptr_type = try o.lowerDebugType(bland_ptr_ty);
                    try o.debug_type_map.put(gpa, ty, debug_ptr_type);
                    return debug_ptr_type;
                }

                const debug_fwd_ref = try o.builder.debugForwardReference();

                // Set as forward reference while the type is lowered in case it references itself
                try o.debug_type_map.put(gpa, ty, debug_fwd_ref);

                if (ty.isSlice(zcu)) {
                    const ptr_ty = ty.slicePtrFieldType(zcu);
                    const len_ty = Type.usize;

                    const name = try o.allocTypeName(ty);
                    defer gpa.free(name);
                    const line = 0;

                    const ptr_size = ptr_ty.abiSize(zcu);
                    const ptr_align = ptr_ty.abiAlignment(zcu);
                    const len_size = len_ty.abiSize(zcu);
                    const len_align = len_ty.abiAlignment(zcu);

                    const len_offset = len_align.forward(ptr_size);

                    const debug_ptr_type = try o.builder.debugMemberType(
                        try o.builder.metadataString("ptr"),
                        .none, // File
                        debug_fwd_ref,
                        0, // Line
                        try o.lowerDebugType(ptr_ty),
                        ptr_size * 8,
                        (ptr_align.toByteUnits() orelse 0) * 8,
                        0, // Offset
                    );

                    const debug_len_type = try o.builder.debugMemberType(
                        try o.builder.metadataString("len"),
                        .none, // File
                        debug_fwd_ref,
                        0, // Line
                        try o.lowerDebugType(len_ty),
                        len_size * 8,
                        (len_align.toByteUnits() orelse 0) * 8,
                        len_offset * 8,
                    );

                    const debug_slice_type = try o.builder.debugStructType(
                        try o.builder.metadataString(name),
                        .none, // File
                        o.debug_compile_unit, // Scope
                        line,
                        .none, // Underlying type
                        ty.abiSize(zcu) * 8,
                        (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                        try o.builder.metadataTuple(&.{
                            debug_ptr_type,
                            debug_len_type,
                        }),
                    );

                    o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_slice_type);

                    // Set to real type now that it has been lowered fully
                    const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                    map_ptr.* = debug_slice_type;

                    return debug_slice_type;
                }

                const debug_elem_ty = try o.lowerDebugType(Type.fromInterned(ptr_info.child));

                const name = try o.allocTypeName(ty);
                defer gpa.free(name);

                const debug_ptr_type = try o.builder.debugPointerType(
                    try o.builder.metadataString(name),
                    .none, // File
                    .none, // Scope
                    0, // Line
                    debug_elem_ty,
                    target.ptrBitWidth(),
                    (ty.ptrAlignment(zcu).toByteUnits() orelse 0) * 8,
                    0, // Offset
                );

                o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_ptr_type);

                // Set to real type now that it has been lowered fully
                const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                map_ptr.* = debug_ptr_type;

                return debug_ptr_type;
            },
            .@"opaque" => {
                if (ty.toIntern() == .anyopaque_type) {
                    const debug_opaque_type = try o.builder.debugSignedType(
                        try o.builder.metadataString("anyopaque"),
                        0,
                    );
                    try o.debug_type_map.put(gpa, ty, debug_opaque_type);
                    return debug_opaque_type;
                }

                const name = try o.allocTypeName(ty);
                defer gpa.free(name);

                const file = try o.getDebugFile(ty.typeDeclInstAllowGeneratedTag(zcu).?.resolveFile(ip));
                const scope = if (ty.getParentNamespace(zcu).unwrap()) |parent_namespace|
                    try o.namespaceToDebugScope(parent_namespace)
                else
                    file;

                const debug_opaque_type = try o.builder.debugStructType(
                    try o.builder.metadataString(name),
                    file,
                    scope,
                    ty.typeDeclSrcLine(zcu).? + 1, // Line
                    .none, // Underlying type
                    0, // Size
                    0, // Align
                    .none, // Fields
                );
                try o.debug_type_map.put(gpa, ty, debug_opaque_type);
                return debug_opaque_type;
            },
            .array => {
                const debug_array_type = try o.builder.debugArrayType(
                    .none, // Name
                    .none, // File
                    .none, // Scope
                    0, // Line
                    try o.lowerDebugType(ty.childType(zcu)),
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(&.{
                        try o.builder.debugSubrange(
                            try o.builder.metadataConstant(try o.builder.intConst(.i64, 0)),
                            try o.builder.metadataConstant(try o.builder.intConst(.i64, ty.arrayLen(zcu))),
                        ),
                    }),
                );
                try o.debug_type_map.put(gpa, ty, debug_array_type);
                return debug_array_type;
            },
            .vector => {
                const elem_ty = ty.elemType2(zcu);
                // Vector elements cannot be padded since that would make
                // @bitSizOf(elem) * len > @bitSizOf(vec).
                // Neither gdb nor lldb seem to be able to display non-byte sized
                // vectors properly.
                const debug_elem_type = switch (elem_ty.zigTypeTag(zcu)) {
                    .int => blk: {
                        const info = elem_ty.intInfo(zcu);
                        assert(info.bits != 0);
                        const name = try o.allocTypeName(ty);
                        defer gpa.free(name);
                        const builder_name = try o.builder.metadataString(name);
                        break :blk switch (info.signedness) {
                            .signed => try o.builder.debugSignedType(builder_name, info.bits),
                            .unsigned => try o.builder.debugUnsignedType(builder_name, info.bits),
                        };
                    },
                    .bool => try o.builder.debugBoolType(
                        try o.builder.metadataString("bool"),
                        1,
                    ),
                    else => try o.lowerDebugType(ty.childType(zcu)),
                };

                const debug_vector_type = try o.builder.debugVectorType(
                    .none, // Name
                    .none, // File
                    .none, // Scope
                    0, // Line
                    debug_elem_type,
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(&.{
                        try o.builder.debugSubrange(
                            try o.builder.metadataConstant(try o.builder.intConst(.i64, 0)),
                            try o.builder.metadataConstant(try o.builder.intConst(.i64, ty.vectorLen(zcu))),
                        ),
                    }),
                );

                try o.debug_type_map.put(gpa, ty, debug_vector_type);
                return debug_vector_type;
            },
            .optional => {
                const name = try o.allocTypeName(ty);
                defer gpa.free(name);
                const child_ty = ty.optionalChild(zcu);
                if (!child_ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                    const debug_bool_type = try o.builder.debugBoolType(
                        try o.builder.metadataString(name),
                        8,
                    );
                    try o.debug_type_map.put(gpa, ty, debug_bool_type);
                    return debug_bool_type;
                }

                const debug_fwd_ref = try o.builder.debugForwardReference();

                // Set as forward reference while the type is lowered in case it references itself
                try o.debug_type_map.put(gpa, ty, debug_fwd_ref);

                if (ty.optionalReprIsPayload(zcu)) {
                    const debug_optional_type = try o.lowerDebugType(child_ty);

                    o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_optional_type);

                    // Set to real type now that it has been lowered fully
                    const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                    map_ptr.* = debug_optional_type;

                    return debug_optional_type;
                }

                const non_null_ty = Type.u8;
                const payload_size = child_ty.abiSize(zcu);
                const payload_align = child_ty.abiAlignment(zcu);
                const non_null_size = non_null_ty.abiSize(zcu);
                const non_null_align = non_null_ty.abiAlignment(zcu);
                const non_null_offset = non_null_align.forward(payload_size);

                const debug_data_type = try o.builder.debugMemberType(
                    try o.builder.metadataString("data"),
                    .none, // File
                    debug_fwd_ref,
                    0, // Line
                    try o.lowerDebugType(child_ty),
                    payload_size * 8,
                    (payload_align.toByteUnits() orelse 0) * 8,
                    0, // Offset
                );

                const debug_some_type = try o.builder.debugMemberType(
                    try o.builder.metadataString("some"),
                    .none,
                    debug_fwd_ref,
                    0,
                    try o.lowerDebugType(non_null_ty),
                    non_null_size * 8,
                    (non_null_align.toByteUnits() orelse 0) * 8,
                    non_null_offset * 8,
                );

                const debug_optional_type = try o.builder.debugStructType(
                    try o.builder.metadataString(name),
                    .none, // File
                    o.debug_compile_unit, // Scope
                    0, // Line
                    .none, // Underlying type
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(&.{
                        debug_data_type,
                        debug_some_type,
                    }),
                );

                o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_optional_type);

                // Set to real type now that it has been lowered fully
                const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                map_ptr.* = debug_optional_type;

                return debug_optional_type;
            },
            .error_union => {
                const payload_ty = ty.errorUnionPayload(zcu);
                if (!payload_ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                    // TODO: Maybe remove?
                    const debug_error_union_type = try o.lowerDebugType(Type.anyerror);
                    try o.debug_type_map.put(gpa, ty, debug_error_union_type);
                    return debug_error_union_type;
                }

                const name = try o.allocTypeName(ty);
                defer gpa.free(name);

                const error_size = Type.anyerror.abiSize(zcu);
                const error_align = Type.anyerror.abiAlignment(zcu);
                const payload_size = payload_ty.abiSize(zcu);
                const payload_align = payload_ty.abiAlignment(zcu);

                var error_index: u32 = undefined;
                var payload_index: u32 = undefined;
                var error_offset: u64 = undefined;
                var payload_offset: u64 = undefined;
                if (error_align.compare(.gt, payload_align)) {
                    error_index = 0;
                    payload_index = 1;
                    error_offset = 0;
                    payload_offset = payload_align.forward(error_size);
                } else {
                    payload_index = 0;
                    error_index = 1;
                    payload_offset = 0;
                    error_offset = error_align.forward(payload_size);
                }

                const debug_fwd_ref = try o.builder.debugForwardReference();

                var fields: [2]Builder.Metadata = undefined;
                fields[error_index] = try o.builder.debugMemberType(
                    try o.builder.metadataString("tag"),
                    .none, // File
                    debug_fwd_ref,
                    0, // Line
                    try o.lowerDebugType(Type.anyerror),
                    error_size * 8,
                    (error_align.toByteUnits() orelse 0) * 8,
                    error_offset * 8,
                );
                fields[payload_index] = try o.builder.debugMemberType(
                    try o.builder.metadataString("value"),
                    .none, // File
                    debug_fwd_ref,
                    0, // Line
                    try o.lowerDebugType(payload_ty),
                    payload_size * 8,
                    (payload_align.toByteUnits() orelse 0) * 8,
                    payload_offset * 8,
                );

                const debug_error_union_type = try o.builder.debugStructType(
                    try o.builder.metadataString(name),
                    .none, // File
                    o.debug_compile_unit, // Sope
                    0, // Line
                    .none, // Underlying type
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(&fields),
                );

                o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_error_union_type);

                try o.debug_type_map.put(gpa, ty, debug_error_union_type);
                return debug_error_union_type;
            },
            .error_set => {
                const debug_error_set = try o.builder.debugUnsignedType(
                    try o.builder.metadataString("anyerror"),
                    16,
                );
                try o.debug_type_map.put(gpa, ty, debug_error_set);
                return debug_error_set;
            },
            .@"struct" => {
                const name = try o.allocTypeName(ty);
                defer gpa.free(name);

                if (zcu.typeToPackedStruct(ty)) |struct_type| {
                    const backing_int_ty = struct_type.backingIntTypeUnordered(ip);
                    if (backing_int_ty != .none) {
                        const info = Type.fromInterned(backing_int_ty).intInfo(zcu);
                        const builder_name = try o.builder.metadataString(name);
                        const debug_int_type = switch (info.signedness) {
                            .signed => try o.builder.debugSignedType(builder_name, ty.abiSize(zcu) * 8),
                            .unsigned => try o.builder.debugUnsignedType(builder_name, ty.abiSize(zcu) * 8),
                        };
                        try o.debug_type_map.put(gpa, ty, debug_int_type);
                        return debug_int_type;
                    }
                }

                switch (ip.indexToKey(ty.toIntern())) {
                    .tuple_type => |tuple| {
                        var fields: std.ArrayListUnmanaged(Builder.Metadata) = .empty;
                        defer fields.deinit(gpa);

                        try fields.ensureUnusedCapacity(gpa, tuple.types.len);

                        comptime assert(struct_layout_version == 2);
                        var offset: u64 = 0;

                        const debug_fwd_ref = try o.builder.debugForwardReference();

                        for (tuple.types.get(ip), tuple.values.get(ip), 0..) |field_ty, field_val, i| {
                            if (field_val != .none or !Type.fromInterned(field_ty).hasRuntimeBits(zcu)) continue;

                            const field_size = Type.fromInterned(field_ty).abiSize(zcu);
                            const field_align = Type.fromInterned(field_ty).abiAlignment(zcu);
                            const field_offset = field_align.forward(offset);
                            offset = field_offset + field_size;

                            var name_buf: [32]u8 = undefined;
                            const field_name = std.fmt.bufPrint(&name_buf, "{d}", .{i}) catch unreachable;

                            fields.appendAssumeCapacity(try o.builder.debugMemberType(
                                try o.builder.metadataString(field_name),
                                .none, // File
                                debug_fwd_ref,
                                0,
                                try o.lowerDebugType(Type.fromInterned(field_ty)),
                                field_size * 8,
                                (field_align.toByteUnits() orelse 0) * 8,
                                field_offset * 8,
                            ));
                        }

                        const debug_struct_type = try o.builder.debugStructType(
                            try o.builder.metadataString(name),
                            .none, // File
                            o.debug_compile_unit, // Scope
                            0, // Line
                            .none, // Underlying type
                            ty.abiSize(zcu) * 8,
                            (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                            try o.builder.metadataTuple(fields.items),
                        );

                        o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_struct_type);

                        try o.debug_type_map.put(gpa, ty, debug_struct_type);
                        return debug_struct_type;
                    },
                    .struct_type => {
                        if (!ip.loadStructType(ty.toIntern()).haveFieldTypes(ip)) {
                            // This can happen if a struct type makes it all the way to
                            // flush() without ever being instantiated or referenced (even
                            // via pointer). The only reason we are hearing about it now is
                            // that it is being used as a namespace to put other debug types
                            // into. Therefore we can satisfy this by making an empty namespace,
                            // rather than changing the frontend to unnecessarily resolve the
                            // struct field types.
                            const debug_struct_type = try o.makeEmptyNamespaceDebugType(ty);
                            try o.debug_type_map.put(gpa, ty, debug_struct_type);
                            return debug_struct_type;
                        }
                    },
                    else => {},
                }

                if (!ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                    const debug_struct_type = try o.makeEmptyNamespaceDebugType(ty);
                    try o.debug_type_map.put(gpa, ty, debug_struct_type);
                    return debug_struct_type;
                }

                const struct_type = zcu.typeToStruct(ty).?;

                var fields: std.ArrayListUnmanaged(Builder.Metadata) = .empty;
                defer fields.deinit(gpa);

                try fields.ensureUnusedCapacity(gpa, struct_type.field_types.len);

                const debug_fwd_ref = try o.builder.debugForwardReference();

                // Set as forward reference while the type is lowered in case it references itself
                try o.debug_type_map.put(gpa, ty, debug_fwd_ref);

                comptime assert(struct_layout_version == 2);
                var it = struct_type.iterateRuntimeOrder(ip);
                while (it.next()) |field_index| {
                    const field_ty = Type.fromInterned(struct_type.field_types.get(ip)[field_index]);
                    if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;
                    const field_size = field_ty.abiSize(zcu);
                    const field_align = ty.fieldAlignment(field_index, zcu);
                    const field_offset = ty.structFieldOffset(field_index, zcu);
                    const field_name = struct_type.fieldName(ip, field_index).unwrap() orelse
                        try ip.getOrPutStringFmt(gpa, pt.tid, "{d}", .{field_index}, .no_embedded_nulls);
                    fields.appendAssumeCapacity(try o.builder.debugMemberType(
                        try o.builder.metadataString(field_name.toSlice(ip)),
                        .none, // File
                        debug_fwd_ref,
                        0, // Line
                        try o.lowerDebugType(field_ty),
                        field_size * 8,
                        (field_align.toByteUnits() orelse 0) * 8,
                        field_offset * 8,
                    ));
                }

                const debug_struct_type = try o.builder.debugStructType(
                    try o.builder.metadataString(name),
                    .none, // File
                    o.debug_compile_unit, // Scope
                    0, // Line
                    .none, // Underlying type
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(fields.items),
                );

                o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_struct_type);

                // Set to real type now that it has been lowered fully
                const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                map_ptr.* = debug_struct_type;

                return debug_struct_type;
            },
            .@"union" => {
                const name = try o.allocTypeName(ty);
                defer gpa.free(name);

                const union_type = ip.loadUnionType(ty.toIntern());
                if (!union_type.haveFieldTypes(ip) or
                    !ty.hasRuntimeBitsIgnoreComptime(zcu) or
                    !union_type.haveLayout(ip))
                {
                    const debug_union_type = try o.makeEmptyNamespaceDebugType(ty);
                    try o.debug_type_map.put(gpa, ty, debug_union_type);
                    return debug_union_type;
                }

                const layout = Type.getUnionLayout(union_type, zcu);

                const debug_fwd_ref = try o.builder.debugForwardReference();

                // Set as forward reference while the type is lowered in case it references itself
                try o.debug_type_map.put(gpa, ty, debug_fwd_ref);

                if (layout.payload_size == 0) {
                    const debug_union_type = try o.builder.debugStructType(
                        try o.builder.metadataString(name),
                        .none, // File
                        o.debug_compile_unit, // Scope
                        0, // Line
                        .none, // Underlying type
                        ty.abiSize(zcu) * 8,
                        (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                        try o.builder.metadataTuple(
                            &.{try o.lowerDebugType(Type.fromInterned(union_type.enum_tag_ty))},
                        ),
                    );

                    // Set to real type now that it has been lowered fully
                    const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                    map_ptr.* = debug_union_type;

                    return debug_union_type;
                }

                var fields: std.ArrayListUnmanaged(Builder.Metadata) = .empty;
                defer fields.deinit(gpa);

                try fields.ensureUnusedCapacity(gpa, union_type.loadTagType(ip).names.len);

                const debug_union_fwd_ref = if (layout.tag_size == 0)
                    debug_fwd_ref
                else
                    try o.builder.debugForwardReference();

                const tag_type = union_type.loadTagType(ip);

                for (0..tag_type.names.len) |field_index| {
                    const field_ty = union_type.field_types.get(ip)[field_index];
                    if (!Type.fromInterned(field_ty).hasRuntimeBitsIgnoreComptime(zcu)) continue;

                    const field_size = Type.fromInterned(field_ty).abiSize(zcu);
                    const field_align: InternPool.Alignment = switch (union_type.flagsUnordered(ip).layout) {
                        .@"packed" => .none,
                        .auto, .@"extern" => ty.fieldAlignment(field_index, zcu),
                    };

                    const field_name = tag_type.names.get(ip)[field_index];
                    fields.appendAssumeCapacity(try o.builder.debugMemberType(
                        try o.builder.metadataString(field_name.toSlice(ip)),
                        .none, // File
                        debug_union_fwd_ref,
                        0, // Line
                        try o.lowerDebugType(Type.fromInterned(field_ty)),
                        field_size * 8,
                        (field_align.toByteUnits() orelse 0) * 8,
                        0, // Offset
                    ));
                }

                var union_name_buf: ?[:0]const u8 = null;
                defer if (union_name_buf) |buf| gpa.free(buf);
                const union_name = if (layout.tag_size == 0) name else name: {
                    union_name_buf = try std.fmt.allocPrintZ(gpa, "{s}:Payload", .{name});
                    break :name union_name_buf.?;
                };

                const debug_union_type = try o.builder.debugUnionType(
                    try o.builder.metadataString(union_name),
                    .none, // File
                    o.debug_compile_unit, // Scope
                    0, // Line
                    .none, // Underlying type
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(fields.items),
                );

                o.builder.debugForwardReferenceSetType(debug_union_fwd_ref, debug_union_type);

                if (layout.tag_size == 0) {
                    // Set to real type now that it has been lowered fully
                    const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                    map_ptr.* = debug_union_type;

                    return debug_union_type;
                }

                var tag_offset: u64 = undefined;
                var payload_offset: u64 = undefined;
                if (layout.tag_align.compare(.gte, layout.payload_align)) {
                    tag_offset = 0;
                    payload_offset = layout.payload_align.forward(layout.tag_size);
                } else {
                    payload_offset = 0;
                    tag_offset = layout.tag_align.forward(layout.payload_size);
                }

                const debug_tag_type = try o.builder.debugMemberType(
                    try o.builder.metadataString("tag"),
                    .none, // File
                    debug_fwd_ref,
                    0, // Line
                    try o.lowerDebugType(Type.fromInterned(union_type.enum_tag_ty)),
                    layout.tag_size * 8,
                    (layout.tag_align.toByteUnits() orelse 0) * 8,
                    tag_offset * 8,
                );

                const debug_payload_type = try o.builder.debugMemberType(
                    try o.builder.metadataString("payload"),
                    .none, // File
                    debug_fwd_ref,
                    0, // Line
                    debug_union_type,
                    layout.payload_size * 8,
                    (layout.payload_align.toByteUnits() orelse 0) * 8,
                    payload_offset * 8,
                );

                const full_fields: [2]Builder.Metadata =
                    if (layout.tag_align.compare(.gte, layout.payload_align))
                        .{ debug_tag_type, debug_payload_type }
                    else
                        .{ debug_payload_type, debug_tag_type };

                const debug_tagged_union_type = try o.builder.debugStructType(
                    try o.builder.metadataString(name),
                    .none, // File
                    o.debug_compile_unit, // Scope
                    0, // Line
                    .none, // Underlying type
                    ty.abiSize(zcu) * 8,
                    (ty.abiAlignment(zcu).toByteUnits() orelse 0) * 8,
                    try o.builder.metadataTuple(&full_fields),
                );

                o.builder.debugForwardReferenceSetType(debug_fwd_ref, debug_tagged_union_type);

                // Set to real type now that it has been lowered fully
                const map_ptr = o.debug_type_map.getPtr(ty) orelse unreachable;
                map_ptr.* = debug_tagged_union_type;

                return debug_tagged_union_type;
            },
            .@"fn" => {
                const fn_info = zcu.typeToFunc(ty).?;

                var debug_param_types = std.ArrayList(Builder.Metadata).init(gpa);
                defer debug_param_types.deinit();

                try debug_param_types.ensureUnusedCapacity(3 + fn_info.param_types.len);

                // Return type goes first.
                if (Type.fromInterned(fn_info.return_type).hasRuntimeBitsIgnoreComptime(zcu)) {
                    const sret = firstParamSRet(fn_info, zcu, target);
                    const ret_ty = if (sret) Type.void else Type.fromInterned(fn_info.return_type);
                    debug_param_types.appendAssumeCapacity(try o.lowerDebugType(ret_ty));

                    if (sret) {
                        const ptr_ty = try pt.singleMutPtrType(Type.fromInterned(fn_info.return_type));
                        debug_param_types.appendAssumeCapacity(try o.lowerDebugType(ptr_ty));
                    }
                } else {
                    debug_param_types.appendAssumeCapacity(try o.lowerDebugType(Type.void));
                }

                if (fn_info.cc == .auto and zcu.comp.config.any_error_tracing) {
                    const ptr_ty = try pt.singleMutPtrType(try o.getStackTraceType());
                    debug_param_types.appendAssumeCapacity(try o.lowerDebugType(ptr_ty));
                }

                for (0..fn_info.param_types.len) |i| {
                    const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[i]);
                    if (!param_ty.hasRuntimeBitsIgnoreComptime(zcu)) continue;

                    if (isByRef(param_ty, zcu)) {
                        const ptr_ty = try pt.singleMutPtrType(param_ty);
                        debug_param_types.appendAssumeCapacity(try o.lowerDebugType(ptr_ty));
                    } else {
                        debug_param_types.appendAssumeCapacity(try o.lowerDebugType(param_ty));
                    }
                }

                const debug_function_type = try o.builder.debugSubroutineType(
                    try o.builder.metadataTuple(debug_param_types.items),
                );

                try o.debug_type_map.put(gpa, ty, debug_function_type);
                return debug_function_type;
            },
            .comptime_int => unreachable,
            .comptime_float => unreachable,
            .type => unreachable,
            .undefined => unreachable,
            .null => unreachable,
            .enum_literal => unreachable,

            .frame => @panic("TODO implement lowerDebugType for Frame types"),
            .@"anyframe" => @panic("TODO implement lowerDebugType for AnyFrame types"),
        }
    }

    fn namespaceToDebugScope(o: *Object, namespace_index: InternPool.NamespaceIndex) !Builder.Metadata {
        const zcu = o.pt.zcu;
        const namespace = zcu.namespacePtr(namespace_index);
        if (namespace.parent == .none) return try o.getDebugFile(namespace.file_scope);

        const gop = try o.debug_unresolved_namespace_scopes.getOrPut(o.gpa, namespace_index);

        if (!gop.found_existing) gop.value_ptr.* = try o.builder.debugForwardReference();

        return gop.value_ptr.*;
    }

    fn makeEmptyNamespaceDebugType(o: *Object, ty: Type) !Builder.Metadata {
        const zcu = o.pt.zcu;
        const ip = &zcu.intern_pool;
        const file = try o.getDebugFile(ty.typeDeclInstAllowGeneratedTag(zcu).?.resolveFile(ip));
        const scope = if (ty.getParentNamespace(zcu).unwrap()) |parent_namespace|
            try o.namespaceToDebugScope(parent_namespace)
        else
            file;
        return o.builder.debugStructType(
            try o.builder.metadataString(ty.containerTypeName(ip).toSlice(ip)), // TODO use fully qualified name
            file,
            scope,
            ty.typeDeclSrcLine(zcu).? + 1,
            .none,
            0,
            0,
            .none,
        );
    }

    fn getStackTraceType(o: *Object) Allocator.Error!Type {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;

        const std_mod = zcu.std_mod;
        const std_file_imported = pt.importPkg(std_mod) catch unreachable;

        const builtin_str = try ip.getOrPutString(zcu.gpa, pt.tid, "builtin", .no_embedded_nulls);
        const std_file_root_type = Type.fromInterned(zcu.fileRootType(std_file_imported.file_index));
        const std_namespace = ip.namespacePtr(std_file_root_type.getNamespaceIndex(zcu));
        const builtin_nav = std_namespace.pub_decls.getKeyAdapted(builtin_str, Zcu.Namespace.NameAdapter{ .zcu = zcu }).?;

        const stack_trace_str = try ip.getOrPutString(zcu.gpa, pt.tid, "StackTrace", .no_embedded_nulls);
        // buffer is only used for int_type, `builtin` is a struct.
        const builtin_ty = zcu.navValue(builtin_nav).toType();
        const builtin_namespace = zcu.namespacePtr(builtin_ty.getNamespaceIndex(zcu));
        const stack_trace_nav = builtin_namespace.pub_decls.getKeyAdapted(stack_trace_str, Zcu.Namespace.NameAdapter{ .zcu = zcu }).?;

        // Sema should have ensured that StackTrace was analyzed.
        return zcu.navValue(stack_trace_nav).toType();
    }

    fn allocTypeName(o: *Object, ty: Type) Allocator.Error![:0]const u8 {
        var buffer = std.ArrayList(u8).init(o.gpa);
        errdefer buffer.deinit();
        try ty.print(buffer.writer(), o.pt);
        return buffer.toOwnedSliceSentinel(0);
    }

    /// If the llvm function does not exist, create it.
    /// Note that this can be called before the function's semantic analysis has
    /// completed, so if any attributes rely on that, they must be done in updateFunc, not here.
    fn resolveLlvmFunction(
        o: *Object,
        nav_index: InternPool.Nav.Index,
    ) Allocator.Error!Builder.Function.Index {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const gpa = o.gpa;
        const nav = ip.getNav(nav_index);
        const owner_mod = zcu.navFileScope(nav_index).mod;
        const ty: Type = .fromInterned(nav.typeOf(ip));
        const gop = try o.nav_map.getOrPut(gpa, nav_index);
        if (gop.found_existing) return gop.value_ptr.ptr(&o.builder).kind.function;

        const fn_info = zcu.typeToFunc(ty).?;
        const target = owner_mod.resolved_target.result;
        const sret = firstParamSRet(fn_info, zcu, target);

        const is_extern, const lib_name = if (nav.getExtern(ip)) |@"extern"|
            .{ true, @"extern".lib_name }
        else
            .{ false, .none };
        const function_index = try o.builder.addFunction(
            try o.lowerType(ty),
            try o.builder.strtabString((if (is_extern) nav.name else nav.fqn).toSlice(ip)),
            toLlvmAddressSpace(nav.getAddrspace(), target),
        );
        gop.value_ptr.* = function_index.ptrConst(&o.builder).global;

        var attributes: Builder.FunctionAttributes.Wip = .{};
        defer attributes.deinit(&o.builder);

        if (!is_extern) {
            function_index.setLinkage(.internal, &o.builder);
            function_index.setUnnamedAddr(.unnamed_addr, &o.builder);
        } else {
            if (target.cpu.arch.isWasm()) {
                try attributes.addFnAttr(.{ .string = .{
                    .kind = try o.builder.string("wasm-import-name"),
                    .value = try o.builder.string(nav.name.toSlice(ip)),
                } }, &o.builder);
                if (lib_name.toSlice(ip)) |lib_name_slice| {
                    if (!std.mem.eql(u8, lib_name_slice, "c")) try attributes.addFnAttr(.{ .string = .{
                        .kind = try o.builder.string("wasm-import-module"),
                        .value = try o.builder.string(lib_name_slice),
                    } }, &o.builder);
                }
            }
        }

        var llvm_arg_i: u32 = 0;
        if (sret) {
            // Sret pointers must not be address 0
            try attributes.addParamAttr(llvm_arg_i, .nonnull, &o.builder);
            try attributes.addParamAttr(llvm_arg_i, .@"noalias", &o.builder);

            const raw_llvm_ret_ty = try o.lowerType(Type.fromInterned(fn_info.return_type));
            try attributes.addParamAttr(llvm_arg_i, .{ .sret = raw_llvm_ret_ty }, &o.builder);

            llvm_arg_i += 1;
        }

        const err_return_tracing = fn_info.cc == .auto and zcu.comp.config.any_error_tracing;

        if (err_return_tracing) {
            try attributes.addParamAttr(llvm_arg_i, .nonnull, &o.builder);
            llvm_arg_i += 1;
        }

        if (fn_info.cc == .@"async") {
            @panic("TODO: LLVM backend lower async function");
        }

        {
            const cc_info = toLlvmCallConv(fn_info.cc, target).?;

            function_index.setCallConv(cc_info.llvm_cc, &o.builder);

            if (cc_info.align_stack) {
                try attributes.addFnAttr(.{ .alignstack = .fromByteUnits(target.stackAlignment()) }, &o.builder);
            } else {
                _ = try attributes.removeFnAttr(.alignstack);
            }

            if (cc_info.naked) {
                try attributes.addFnAttr(.naked, &o.builder);
            } else {
                _ = try attributes.removeFnAttr(.naked);
            }

            for (0..cc_info.inreg_param_count) |param_idx| {
                try attributes.addParamAttr(param_idx, .inreg, &o.builder);
            }
            for (cc_info.inreg_param_count..std.math.maxInt(u2)) |param_idx| {
                _ = try attributes.removeParamAttr(param_idx, .inreg);
            }

            switch (fn_info.cc) {
                inline .riscv64_interrupt,
                .riscv32_interrupt,
                .mips_interrupt,
                .mips64_interrupt,
                => |info| {
                    try attributes.addFnAttr(.{ .string = .{
                        .kind = try o.builder.string("interrupt"),
                        .value = try o.builder.string(@tagName(info.mode)),
                    } }, &o.builder);
                },
                .arm_interrupt,
                => |info| {
                    try attributes.addFnAttr(.{ .string = .{
                        .kind = try o.builder.string("interrupt"),
                        .value = try o.builder.string(switch (info.type) {
                            .generic => "",
                            .irq => "IRQ",
                            .fiq => "FIQ",
                            .swi => "SWI",
                            .abort => "ABORT",
                            .undef => "UNDEF",
                        }),
                    } }, &o.builder);
                },
                // these function attributes serve as a backup against any mistakes LLVM makes.
                // clang sets both the function's calling convention and the function attributes
                // in its backend, so future patches to the AVR backend could end up checking only one,
                // possibly breaking our support. it's safer to just emit both.
                .avr_interrupt, .avr_signal, .csky_interrupt => {
                    try attributes.addFnAttr(.{ .string = .{
                        .kind = try o.builder.string(switch (fn_info.cc) {
                            .avr_interrupt,
                            .csky_interrupt,
                            => "interrupt",
                            .avr_signal => "signal",
                            else => unreachable,
                        }),
                        .value = .empty,
                    } }, &o.builder);
                },
                else => {},
            }
        }

        if (nav.getAlignment() != .none)
            function_index.setAlignment(nav.getAlignment().toLlvm(), &o.builder);

        // Function attributes that are independent of analysis results of the function body.
        try o.addCommonFnAttributes(
            &attributes,
            owner_mod,
            // Some backends don't respect the `naked` attribute in `TargetFrameLowering::hasFP()`,
            // so for these backends, LLVM will happily emit code that accesses the stack through
            // the frame pointer. This is nonsensical since what the `naked` attribute does is
            // suppress generation of the prologue and epilogue, and the prologue is where the
            // frame pointer normally gets set up. At time of writing, this is the case for at
            // least x86 and RISC-V.
            owner_mod.omit_frame_pointer or fn_info.cc == .naked,
        );

        if (fn_info.return_type == .noreturn_type) try attributes.addFnAttr(.noreturn, &o.builder);

        // Add parameter attributes. We handle only the case of extern functions (no body)
        // because functions with bodies are handled in `updateFunc`.
        if (is_extern) {
            var it = iterateParamTypes(o, fn_info);
            it.llvm_index = llvm_arg_i;
            while (try it.next()) |lowering| switch (lowering) {
                .byval => {
                    const param_index = it.zig_index - 1;
                    const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[param_index]);
                    if (!isByRef(param_ty, zcu)) {
                        try o.addByValParamAttrs(&attributes, param_ty, param_index, fn_info, it.llvm_index - 1);
                    }
                },
                .byref => {
                    const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                    const param_llvm_ty = try o.lowerType(param_ty);
                    const alignment = param_ty.abiAlignment(zcu);
                    try o.addByRefParamAttrs(&attributes, it.llvm_index - 1, alignment.toLlvm(), it.byval_attr, param_llvm_ty);
                },
                .byref_mut => try attributes.addParamAttr(it.llvm_index - 1, .noundef, &o.builder),
                // No attributes needed for these.
                .no_bits,
                .abi_sized_int,
                .multiple_llvm_types,
                .float_array,
                .i32_array,
                .i64_array,
                => continue,

                .slice => unreachable, // extern functions do not support slice types.

            };
        }

        function_index.setAttributes(try attributes.finish(&o.builder), &o.builder);
        return function_index;
    }

    fn addCommonFnAttributes(
        o: *Object,
        attributes: *Builder.FunctionAttributes.Wip,
        owner_mod: *Package.Module,
        omit_frame_pointer: bool,
    ) Allocator.Error!void {
        if (!owner_mod.red_zone) {
            try attributes.addFnAttr(.noredzone, &o.builder);
        }
        if (omit_frame_pointer) {
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("frame-pointer"),
                .value = try o.builder.string("none"),
            } }, &o.builder);
        } else {
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("frame-pointer"),
                .value = try o.builder.string("all"),
            } }, &o.builder);
        }
        try attributes.addFnAttr(.nounwind, &o.builder);
        if (owner_mod.unwind_tables != .none) {
            try attributes.addFnAttr(
                .{ .uwtable = if (owner_mod.unwind_tables == .@"async") .@"async" else .sync },
                &o.builder,
            );
        }
        if (owner_mod.optimize_mode == .ReleaseSmall) {
            try attributes.addFnAttr(.minsize, &o.builder);
            try attributes.addFnAttr(.optsize, &o.builder);
        }
        const target = owner_mod.resolved_target.result;
        if (target.cpu.model.llvm_name) |s| {
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("target-cpu"),
                .value = try o.builder.string(s),
            } }, &o.builder);
        }
        if (owner_mod.resolved_target.llvm_cpu_features) |s| {
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("target-features"),
                .value = try o.builder.string(std.mem.span(s)),
            } }, &o.builder);
        }
        if (target.abi.float() == .soft) {
            // `use-soft-float` means "use software routines for floating point computations". In
            // other words, it configures how LLVM lowers basic float instructions like `fcmp`,
            // `fadd`, etc. The float calling convention is configured on `TargetMachine` and is
            // mostly an orthogonal concept, although obviously we do need hardware float operations
            // to actually be able to pass float values in float registers.
            //
            // Ideally, we would support something akin to the `-mfloat-abi=softfp` option that GCC
            // and Clang support for Arm32 and CSKY. We don't currently expose such an option in
            // Zig, and using CPU features as the source of truth for this makes for a miserable
            // user experience since people expect e.g. `arm-linux-gnueabi` to mean full soft float
            // unless the compiler has explicitly been told otherwise. (And note that our baseline
            // CPU models almost all include FPU features!)
            //
            // Revisit this at some point.
            try attributes.addFnAttr(.{ .string = .{
                .kind = try o.builder.string("use-soft-float"),
                .value = try o.builder.string("true"),
            } }, &o.builder);

            // This prevents LLVM from using FPU/SIMD code for things like `memcpy`. As for the
            // above, this should be revisited if `softfp` support is added.
            try attributes.addFnAttr(.noimplicitfloat, &o.builder);
        }
    }

    fn resolveGlobalUav(
        o: *Object,
        uav: InternPool.Index,
        llvm_addr_space: Builder.AddrSpace,
        alignment: InternPool.Alignment,
    ) Error!Builder.Variable.Index {
        assert(alignment != .none);
        // TODO: Add address space to the anon_decl_map
        const gop = try o.uav_map.getOrPut(o.gpa, uav);
        if (gop.found_existing) {
            // Keep the greater of the two alignments.
            const variable_index = gop.value_ptr.ptr(&o.builder).kind.variable;
            const old_alignment = InternPool.Alignment.fromLlvm(variable_index.getAlignment(&o.builder));
            const max_alignment = old_alignment.maxStrict(alignment);
            variable_index.setAlignment(max_alignment.toLlvm(), &o.builder);
            return variable_index;
        }
        errdefer assert(o.uav_map.remove(uav));

        const zcu = o.pt.zcu;
        const decl_ty = zcu.intern_pool.typeOf(uav);

        const variable_index = try o.builder.addVariable(
            try o.builder.strtabStringFmt("__anon_{d}", .{@intFromEnum(uav)}),
            try o.lowerType(Type.fromInterned(decl_ty)),
            llvm_addr_space,
        );
        gop.value_ptr.* = variable_index.ptrConst(&o.builder).global;

        try variable_index.setInitializer(try o.lowerValue(uav), &o.builder);
        variable_index.setLinkage(.internal, &o.builder);
        variable_index.setMutability(.constant, &o.builder);
        variable_index.setUnnamedAddr(.unnamed_addr, &o.builder);
        variable_index.setAlignment(alignment.toLlvm(), &o.builder);
        return variable_index;
    }

    fn resolveGlobalNav(
        o: *Object,
        nav_index: InternPool.Nav.Index,
    ) Allocator.Error!Builder.Variable.Index {
        const gop = try o.nav_map.getOrPut(o.gpa, nav_index);
        if (gop.found_existing) return gop.value_ptr.ptr(&o.builder).kind.variable;
        errdefer assert(o.nav_map.remove(nav_index));

        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const nav = ip.getNav(nav_index);
        const is_extern, const is_threadlocal, const is_weak_linkage, const is_dll_import = switch (nav.status) {
            .unresolved => unreachable,
            .fully_resolved => |r| switch (ip.indexToKey(r.val)) {
                .variable => |variable| .{ false, variable.is_threadlocal, variable.is_weak_linkage, false },
                .@"extern" => |@"extern"| .{ true, @"extern".is_threadlocal, @"extern".is_weak_linkage, @"extern".is_dll_import },
                else => .{ false, false, false, false },
            },
            // This means it's a source declaration which is not `extern`!
            .type_resolved => |r| .{ false, r.is_threadlocal, false, false },
        };

        const variable_index = try o.builder.addVariable(
            try o.builder.strtabString((if (is_extern) nav.name else nav.fqn).toSlice(ip)),
            try o.lowerType(Type.fromInterned(nav.typeOf(ip))),
            toLlvmGlobalAddressSpace(nav.getAddrspace(), zcu.getTarget()),
        );
        gop.value_ptr.* = variable_index.ptrConst(&o.builder).global;

        // This is needed for declarations created by `@extern`.
        if (is_extern) {
            variable_index.setLinkage(.external, &o.builder);
            variable_index.setUnnamedAddr(.default, &o.builder);
            if (is_threadlocal and !zcu.navFileScope(nav_index).mod.single_threaded)
                variable_index.setThreadLocal(.generaldynamic, &o.builder);
            if (is_weak_linkage) variable_index.setLinkage(.extern_weak, &o.builder);
            if (is_dll_import) variable_index.setDllStorageClass(.dllimport, &o.builder);
        } else {
            variable_index.setLinkage(.internal, &o.builder);
            variable_index.setUnnamedAddr(.unnamed_addr, &o.builder);
        }
        return variable_index;
    }

    fn errorIntType(o: *Object) Allocator.Error!Builder.Type {
        return o.builder.intType(o.pt.zcu.errorSetBits());
    }

    fn lowerType(o: *Object, t: Type) Allocator.Error!Builder.Type {
        const pt = o.pt;
        const zcu = pt.zcu;
        const target = zcu.getTarget();
        const ip = &zcu.intern_pool;
        return switch (t.toIntern()) {
            .u0_type, .i0_type => unreachable,
            inline .u1_type,
            .u8_type,
            .i8_type,
            .u16_type,
            .i16_type,
            .u29_type,
            .u32_type,
            .i32_type,
            .u64_type,
            .i64_type,
            .u80_type,
            .u128_type,
            .i128_type,
            => |tag| @field(Builder.Type, "i" ++ @tagName(tag)[1 .. @tagName(tag).len - "_type".len]),
            .usize_type, .isize_type => try o.builder.intType(target.ptrBitWidth()),
            inline .c_char_type,
            .c_short_type,
            .c_ushort_type,
            .c_int_type,
            .c_uint_type,
            .c_long_type,
            .c_ulong_type,
            .c_longlong_type,
            .c_ulonglong_type,
            => |tag| try o.builder.intType(target.cTypeBitSize(
                @field(std.Target.CType, @tagName(tag)["c_".len .. @tagName(tag).len - "_type".len]),
            )),
            .c_longdouble_type,
            .f16_type,
            .f32_type,
            .f64_type,
            .f80_type,
            .f128_type,
            => switch (t.floatBits(target)) {
                16 => if (backendSupportsF16(target)) .half else .i16,
                32 => .float,
                64 => .double,
                80 => if (backendSupportsF80(target)) .x86_fp80 else .i80,
                128 => .fp128,
                else => unreachable,
            },
            .anyopaque_type => {
                // This is unreachable except when used as the type for an extern global.
                // For example: `@extern(*anyopaque, .{ .name = "foo"})` should produce
                // @foo = external global i8
                return .i8;
            },
            .bool_type => .i1,
            .void_type => .void,
            .type_type => unreachable,
            .anyerror_type => try o.errorIntType(),
            .comptime_int_type,
            .comptime_float_type,
            .noreturn_type,
            => unreachable,
            .anyframe_type => @panic("TODO implement lowerType for AnyFrame types"),
            .null_type,
            .undefined_type,
            .enum_literal_type,
            => unreachable,
            .manyptr_u8_type,
            .manyptr_const_u8_type,
            .manyptr_const_u8_sentinel_0_type,
            .single_const_pointer_to_comptime_int_type,
            => .ptr,
            .slice_const_u8_type,
            .slice_const_u8_sentinel_0_type,
            => try o.builder.structType(.normal, &.{ .ptr, try o.lowerType(Type.usize) }),
            .optional_noreturn_type => unreachable,
            .anyerror_void_error_union_type,
            .adhoc_inferred_error_set_type,
            => try o.errorIntType(),
            .generic_poison_type,
            .empty_tuple_type,
            => unreachable,
            // values, not types
            .undef,
            .zero,
            .zero_usize,
            .zero_u8,
            .one,
            .one_usize,
            .one_u8,
            .four_u8,
            .negative_one,
            .void_value,
            .unreachable_value,
            .null_value,
            .bool_true,
            .bool_false,
            .empty_tuple,
            .none,
            => unreachable,
            else => switch (ip.indexToKey(t.toIntern())) {
                .int_type => |int_type| try o.builder.intType(int_type.bits),
                .ptr_type => |ptr_type| type: {
                    const ptr_ty = try o.builder.ptrType(
                        toLlvmAddressSpace(ptr_type.flags.address_space, target),
                    );
                    break :type switch (ptr_type.flags.size) {
                        .one, .many, .c => ptr_ty,
                        .slice => try o.builder.structType(.normal, &.{
                            ptr_ty,
                            try o.lowerType(Type.usize),
                        }),
                    };
                },
                .array_type => |array_type| o.builder.arrayType(
                    array_type.lenIncludingSentinel(),
                    try o.lowerType(Type.fromInterned(array_type.child)),
                ),
                .vector_type => |vector_type| o.builder.vectorType(
                    .normal,
                    vector_type.len,
                    try o.lowerType(Type.fromInterned(vector_type.child)),
                ),
                .opt_type => |child_ty| {
                    // Must stay in sync with `opt_payload` logic in `lowerPtr`.
                    if (!Type.fromInterned(child_ty).hasRuntimeBitsIgnoreComptime(zcu)) return .i8;

                    const payload_ty = try o.lowerType(Type.fromInterned(child_ty));
                    if (t.optionalReprIsPayload(zcu)) return payload_ty;

                    comptime assert(optional_layout_version == 3);
                    var fields: [3]Builder.Type = .{ payload_ty, .i8, undefined };
                    var fields_len: usize = 2;
                    const offset = Type.fromInterned(child_ty).abiSize(zcu) + 1;
                    const abi_size = t.abiSize(zcu);
                    const padding_len = abi_size - offset;
                    if (padding_len > 0) {
                        fields[2] = try o.builder.arrayType(padding_len, .i8);
                        fields_len = 3;
                    }
                    return o.builder.structType(.normal, fields[0..fields_len]);
                },
                .anyframe_type => @panic("TODO implement lowerType for AnyFrame types"),
                .error_union_type => |error_union_type| {
                    // Must stay in sync with `codegen.errUnionPayloadOffset`.
                    // See logic in `lowerPtr`.
                    const error_type = try o.errorIntType();
                    if (!Type.fromInterned(error_union_type.payload_type).hasRuntimeBitsIgnoreComptime(zcu))
                        return error_type;
                    const payload_type = try o.lowerType(Type.fromInterned(error_union_type.payload_type));
                    const err_int_ty = try o.pt.errorIntType();

                    const payload_align = Type.fromInterned(error_union_type.payload_type).abiAlignment(zcu);
                    const error_align = err_int_ty.abiAlignment(zcu);

                    const payload_size = Type.fromInterned(error_union_type.payload_type).abiSize(zcu);
                    const error_size = err_int_ty.abiSize(zcu);

                    var fields: [3]Builder.Type = undefined;
                    var fields_len: usize = 2;
                    const padding_len = if (error_align.compare(.gt, payload_align)) pad: {
                        fields[0] = error_type;
                        fields[1] = payload_type;
                        const payload_end =
                            payload_align.forward(error_size) +
                            payload_size;
                        const abi_size = error_align.forward(payload_end);
                        break :pad abi_size - payload_end;
                    } else pad: {
                        fields[0] = payload_type;
                        fields[1] = error_type;
                        const error_end =
                            error_align.forward(payload_size) +
                            error_size;
                        const abi_size = payload_align.forward(error_end);
                        break :pad abi_size - error_end;
                    };
                    if (padding_len > 0) {
                        fields[2] = try o.builder.arrayType(padding_len, .i8);
                        fields_len = 3;
                    }
                    return o.builder.structType(.normal, fields[0..fields_len]);
                },
                .simple_type => unreachable,
                .struct_type => {
                    if (o.type_map.get(t.toIntern())) |value| return value;

                    const struct_type = ip.loadStructType(t.toIntern());

                    if (struct_type.layout == .@"packed") {
                        const int_ty = try o.lowerType(Type.fromInterned(struct_type.backingIntTypeUnordered(ip)));
                        try o.type_map.put(o.gpa, t.toIntern(), int_ty);
                        return int_ty;
                    }

                    var llvm_field_types: std.ArrayListUnmanaged(Builder.Type) = .empty;
                    defer llvm_field_types.deinit(o.gpa);
                    // Although we can estimate how much capacity to add, these cannot be
                    // relied upon because of the recursive calls to lowerType below.
                    try llvm_field_types.ensureUnusedCapacity(o.gpa, struct_type.field_types.len);
                    try o.struct_field_map.ensureUnusedCapacity(o.gpa, struct_type.field_types.len);

                    comptime assert(struct_layout_version == 2);
                    var offset: u64 = 0;
                    var big_align: InternPool.Alignment = .@"1";
                    var struct_kind: Builder.Type.Structure.Kind = .normal;
                    // When we encounter a zero-bit field, we place it here so we know to map it to the next non-zero-bit field (if any).
                    var it = struct_type.iterateRuntimeOrder(ip);
                    while (it.next()) |field_index| {
                        const field_ty = Type.fromInterned(struct_type.field_types.get(ip)[field_index]);
                        const field_align = t.fieldAlignment(field_index, zcu);
                        const field_ty_align = field_ty.abiAlignment(zcu);
                        if (field_align.compare(.lt, field_ty_align)) struct_kind = .@"packed";
                        big_align = big_align.max(field_align);
                        const prev_offset = offset;
                        offset = field_align.forward(offset);

                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) try llvm_field_types.append(
                            o.gpa,
                            try o.builder.arrayType(padding_len, .i8),
                        );

                        if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                            // This is a zero-bit field. If there are runtime bits after this field,
                            // map to the next LLVM field (which we know exists): otherwise, don't
                            // map the field, indicating it's at the end of the struct.
                            if (offset != struct_type.sizeUnordered(ip)) {
                                try o.struct_field_map.put(o.gpa, .{
                                    .struct_ty = t.toIntern(),
                                    .field_index = field_index,
                                }, @intCast(llvm_field_types.items.len));
                            }
                            continue;
                        }

                        try o.struct_field_map.put(o.gpa, .{
                            .struct_ty = t.toIntern(),
                            .field_index = field_index,
                        }, @intCast(llvm_field_types.items.len));
                        try llvm_field_types.append(o.gpa, try o.lowerType(field_ty));

                        offset += field_ty.abiSize(zcu);
                    }
                    {
                        const prev_offset = offset;
                        offset = big_align.forward(offset);
                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) try llvm_field_types.append(
                            o.gpa,
                            try o.builder.arrayType(padding_len, .i8),
                        );
                    }

                    const ty = try o.builder.opaqueType(try o.builder.string(t.containerTypeName(ip).toSlice(ip)));
                    try o.type_map.put(o.gpa, t.toIntern(), ty);

                    o.builder.namedTypeSetBody(
                        ty,
                        try o.builder.structType(struct_kind, llvm_field_types.items),
                    );
                    return ty;
                },
                .tuple_type => |tuple_type| {
                    var llvm_field_types: std.ArrayListUnmanaged(Builder.Type) = .empty;
                    defer llvm_field_types.deinit(o.gpa);
                    // Although we can estimate how much capacity to add, these cannot be
                    // relied upon because of the recursive calls to lowerType below.
                    try llvm_field_types.ensureUnusedCapacity(o.gpa, tuple_type.types.len);
                    try o.struct_field_map.ensureUnusedCapacity(o.gpa, tuple_type.types.len);

                    comptime assert(struct_layout_version == 2);
                    var offset: u64 = 0;
                    var big_align: InternPool.Alignment = .none;

                    const struct_size = t.abiSize(zcu);

                    for (
                        tuple_type.types.get(ip),
                        tuple_type.values.get(ip),
                        0..,
                    ) |field_ty, field_val, field_index| {
                        if (field_val != .none) continue;

                        const field_align = Type.fromInterned(field_ty).abiAlignment(zcu);
                        big_align = big_align.max(field_align);
                        const prev_offset = offset;
                        offset = field_align.forward(offset);

                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) try llvm_field_types.append(
                            o.gpa,
                            try o.builder.arrayType(padding_len, .i8),
                        );
                        if (!Type.fromInterned(field_ty).hasRuntimeBitsIgnoreComptime(zcu)) {
                            // This is a zero-bit field. If there are runtime bits after this field,
                            // map to the next LLVM field (which we know exists): otherwise, don't
                            // map the field, indicating it's at the end of the struct.
                            if (offset != struct_size) {
                                try o.struct_field_map.put(o.gpa, .{
                                    .struct_ty = t.toIntern(),
                                    .field_index = @intCast(field_index),
                                }, @intCast(llvm_field_types.items.len));
                            }
                            continue;
                        }
                        try o.struct_field_map.put(o.gpa, .{
                            .struct_ty = t.toIntern(),
                            .field_index = @intCast(field_index),
                        }, @intCast(llvm_field_types.items.len));
                        try llvm_field_types.append(o.gpa, try o.lowerType(Type.fromInterned(field_ty)));

                        offset += Type.fromInterned(field_ty).abiSize(zcu);
                    }
                    {
                        const prev_offset = offset;
                        offset = big_align.forward(offset);
                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) try llvm_field_types.append(
                            o.gpa,
                            try o.builder.arrayType(padding_len, .i8),
                        );
                    }
                    return o.builder.structType(.normal, llvm_field_types.items);
                },
                .union_type => {
                    if (o.type_map.get(t.toIntern())) |value| return value;

                    const union_obj = ip.loadUnionType(t.toIntern());
                    const layout = Type.getUnionLayout(union_obj, zcu);

                    if (union_obj.flagsUnordered(ip).layout == .@"packed") {
                        const int_ty = try o.builder.intType(@intCast(t.bitSize(zcu)));
                        try o.type_map.put(o.gpa, t.toIntern(), int_ty);
                        return int_ty;
                    }

                    if (layout.payload_size == 0) {
                        const enum_tag_ty = try o.lowerType(Type.fromInterned(union_obj.enum_tag_ty));
                        try o.type_map.put(o.gpa, t.toIntern(), enum_tag_ty);
                        return enum_tag_ty;
                    }

                    const aligned_field_ty = Type.fromInterned(union_obj.field_types.get(ip)[layout.most_aligned_field]);
                    const aligned_field_llvm_ty = try o.lowerType(aligned_field_ty);

                    const payload_ty = ty: {
                        if (layout.most_aligned_field_size == layout.payload_size) {
                            break :ty aligned_field_llvm_ty;
                        }
                        const padding_len = if (layout.tag_size == 0)
                            layout.abi_size - layout.most_aligned_field_size
                        else
                            layout.payload_size - layout.most_aligned_field_size;
                        break :ty try o.builder.structType(.@"packed", &.{
                            aligned_field_llvm_ty,
                            try o.builder.arrayType(padding_len, .i8),
                        });
                    };

                    if (layout.tag_size == 0) {
                        const ty = try o.builder.opaqueType(try o.builder.string(t.containerTypeName(ip).toSlice(ip)));
                        try o.type_map.put(o.gpa, t.toIntern(), ty);

                        o.builder.namedTypeSetBody(
                            ty,
                            try o.builder.structType(.normal, &.{payload_ty}),
                        );
                        return ty;
                    }
                    const enum_tag_ty = try o.lowerType(Type.fromInterned(union_obj.enum_tag_ty));

                    // Put the tag before or after the payload depending on which one's
                    // alignment is greater.
                    var llvm_fields: [3]Builder.Type = undefined;
                    var llvm_fields_len: usize = 2;

                    if (layout.tag_align.compare(.gte, layout.payload_align)) {
                        llvm_fields = .{ enum_tag_ty, payload_ty, .none };
                    } else {
                        llvm_fields = .{ payload_ty, enum_tag_ty, .none };
                    }

                    // Insert padding to make the LLVM struct ABI size match the Zig union ABI size.
                    if (layout.padding != 0) {
                        llvm_fields[llvm_fields_len] = try o.builder.arrayType(layout.padding, .i8);
                        llvm_fields_len += 1;
                    }

                    const ty = try o.builder.opaqueType(try o.builder.string(t.containerTypeName(ip).toSlice(ip)));
                    try o.type_map.put(o.gpa, t.toIntern(), ty);

                    o.builder.namedTypeSetBody(
                        ty,
                        try o.builder.structType(.normal, llvm_fields[0..llvm_fields_len]),
                    );
                    return ty;
                },
                .opaque_type => {
                    const gop = try o.type_map.getOrPut(o.gpa, t.toIntern());
                    if (!gop.found_existing) {
                        gop.value_ptr.* = try o.builder.opaqueType(try o.builder.string(t.containerTypeName(ip).toSlice(ip)));
                    }
                    return gop.value_ptr.*;
                },
                .enum_type => try o.lowerType(Type.fromInterned(ip.loadEnumType(t.toIntern()).tag_ty)),
                .func_type => |func_type| try o.lowerTypeFn(func_type),
                .error_set_type, .inferred_error_set_type => try o.errorIntType(),
                // values, not types
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
                // memoization, not types
                .memoized_call,
                => unreachable,
            },
        };
    }

    /// Use this instead of lowerType when you want to handle correctly the case of elem_ty
    /// being a zero bit type, but it should still be lowered as an i8 in such case.
    /// There are other similar cases handled here as well.
    fn lowerPtrElemTy(o: *Object, elem_ty: Type) Allocator.Error!Builder.Type {
        const pt = o.pt;
        const zcu = pt.zcu;
        const lower_elem_ty = switch (elem_ty.zigTypeTag(zcu)) {
            .@"opaque" => true,
            .@"fn" => !zcu.typeToFunc(elem_ty).?.is_generic,
            .array => elem_ty.childType(zcu).hasRuntimeBitsIgnoreComptime(zcu),
            else => elem_ty.hasRuntimeBitsIgnoreComptime(zcu),
        };
        return if (lower_elem_ty) try o.lowerType(elem_ty) else .i8;
    }

    fn lowerTypeFn(o: *Object, fn_info: InternPool.Key.FuncType) Allocator.Error!Builder.Type {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const target = zcu.getTarget();
        const ret_ty = try lowerFnRetTy(o, fn_info);

        var llvm_params: std.ArrayListUnmanaged(Builder.Type) = .empty;
        defer llvm_params.deinit(o.gpa);

        if (firstParamSRet(fn_info, zcu, target)) {
            try llvm_params.append(o.gpa, .ptr);
        }

        if (fn_info.cc == .auto and zcu.comp.config.any_error_tracing) {
            const ptr_ty = try pt.singleMutPtrType(try o.getStackTraceType());
            try llvm_params.append(o.gpa, try o.lowerType(ptr_ty));
        }

        var it = iterateParamTypes(o, fn_info);
        while (try it.next()) |lowering| switch (lowering) {
            .no_bits => continue,
            .byval => {
                const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                try llvm_params.append(o.gpa, try o.lowerType(param_ty));
            },
            .byref, .byref_mut => {
                try llvm_params.append(o.gpa, .ptr);
            },
            .abi_sized_int => {
                const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                try llvm_params.append(o.gpa, try o.builder.intType(
                    @intCast(param_ty.abiSize(zcu) * 8),
                ));
            },
            .slice => {
                const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                try llvm_params.appendSlice(o.gpa, &.{
                    try o.builder.ptrType(toLlvmAddressSpace(param_ty.ptrAddressSpace(zcu), target)),
                    try o.lowerType(Type.usize),
                });
            },
            .multiple_llvm_types => {
                try llvm_params.appendSlice(o.gpa, it.types_buffer[0..it.types_len]);
            },
            .float_array => |count| {
                const param_ty = Type.fromInterned(fn_info.param_types.get(ip)[it.zig_index - 1]);
                const float_ty = try o.lowerType(aarch64_c_abi.getFloatArrayType(param_ty, zcu).?);
                try llvm_params.append(o.gpa, try o.builder.arrayType(count, float_ty));
            },
            .i32_array, .i64_array => |arr_len| {
                try llvm_params.append(o.gpa, try o.builder.arrayType(arr_len, switch (lowering) {
                    .i32_array => .i32,
                    .i64_array => .i64,
                    else => unreachable,
                }));
            },
        };

        return o.builder.fnType(
            ret_ty,
            llvm_params.items,
            if (fn_info.is_var_args) .vararg else .normal,
        );
    }

    fn lowerValueToInt(o: *Object, llvm_int_ty: Builder.Type, arg_val: InternPool.Index) Error!Builder.Constant {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const target = zcu.getTarget();

        const val = Value.fromInterned(arg_val);
        const val_key = ip.indexToKey(val.toIntern());

        if (val.isUndefDeep(zcu)) return o.builder.undefConst(llvm_int_ty);

        const ty = Type.fromInterned(val_key.typeOf());
        switch (val_key) {
            .@"extern" => |@"extern"| {
                const function_index = try o.resolveLlvmFunction(@"extern".owner_nav);
                const ptr = function_index.ptrConst(&o.builder).global.toConst();
                return o.builder.convConst(ptr, llvm_int_ty);
            },
            .func => |func| {
                const function_index = try o.resolveLlvmFunction(func.owner_nav);
                const ptr = function_index.ptrConst(&o.builder).global.toConst();
                return o.builder.convConst(ptr, llvm_int_ty);
            },
            .ptr => return o.builder.convConst(try o.lowerPtr(arg_val, 0), llvm_int_ty),
            .aggregate => switch (ip.indexToKey(ty.toIntern())) {
                .struct_type, .vector_type => {},
                else => unreachable,
            },
            .un => |un| {
                const layout = ty.unionGetLayout(zcu);
                if (layout.payload_size == 0) return o.lowerValue(un.tag);

                const union_obj = zcu.typeToUnion(ty).?;
                const container_layout = union_obj.flagsUnordered(ip).layout;

                assert(container_layout == .@"packed");

                var need_unnamed = false;
                if (un.tag == .none) {
                    assert(layout.tag_size == 0);
                    const union_val = try o.lowerValueToInt(llvm_int_ty, un.val);

                    need_unnamed = true;
                    return union_val;
                }
                const field_index = zcu.unionTagFieldIndex(union_obj, Value.fromInterned(un.tag)).?;
                const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_index]);
                if (!field_ty.hasRuntimeBits(zcu)) return o.builder.intConst(llvm_int_ty, 0);
                return o.lowerValueToInt(llvm_int_ty, un.val);
            },
            .simple_value => |simple_value| switch (simple_value) {
                .false, .true => {},
                else => unreachable,
            },
            .int,
            .float,
            .enum_tag,
            => {},
            .opt => {}, // pointer like optional expected
            else => unreachable,
        }
        const bits = ty.bitSize(zcu);
        const bytes: usize = @intCast(std.mem.alignForward(u64, bits, 8) / 8);

        var stack = std.heap.stackFallback(32, o.gpa);
        const allocator = stack.get();

        const limbs = try allocator.alloc(
            std.math.big.Limb,
            std.mem.alignForward(usize, bytes, @sizeOf(std.math.big.Limb)) /
                @sizeOf(std.math.big.Limb),
        );
        defer allocator.free(limbs);
        @memset(limbs, 0);

        val.writeToPackedMemory(ty, pt, std.mem.sliceAsBytes(limbs)[0..bytes], 0) catch unreachable;

        if (builtin.target.cpu.arch.endian() == .little) {
            if (target.cpu.arch.endian() == .big)
                std.mem.reverse(u8, std.mem.sliceAsBytes(limbs)[0..bytes]);
        } else if (target.cpu.arch.endian() == .little) {
            for (limbs) |*limb| {
                limb.* = std.mem.nativeToLittle(usize, limb.*);
            }
        }

        return o.builder.bigIntConst(llvm_int_ty, .{
            .limbs = limbs,
            .positive = true,
        });
    }

    fn lowerValue(o: *Object, arg_val: InternPool.Index) Error!Builder.Constant {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const target = zcu.getTarget();

        const val = Value.fromInterned(arg_val);
        const val_key = ip.indexToKey(val.toIntern());

        if (val.isUndefDeep(zcu)) {
            return o.builder.undefConst(try o.lowerType(Type.fromInterned(val_key.typeOf())));
        }

        const ty = Type.fromInterned(val_key.typeOf());
        return switch (val_key) {
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
            => unreachable, // types, not values

            .undef => unreachable, // handled above
            .simple_value => |simple_value| switch (simple_value) {
                .undefined => unreachable, // non-runtime value
                .void => unreachable, // non-runtime value
                .null => unreachable, // non-runtime value
                .empty_tuple => unreachable, // non-runtime value
                .@"unreachable" => unreachable, // non-runtime value

                .false => .false,
                .true => .true,
            },
            .variable,
            .enum_literal,
            .empty_enum_value,
            => unreachable, // non-runtime values
            .@"extern" => |@"extern"| {
                const function_index = try o.resolveLlvmFunction(@"extern".owner_nav);
                return function_index.ptrConst(&o.builder).global.toConst();
            },
            .func => |func| {
                const function_index = try o.resolveLlvmFunction(func.owner_nav);
                return function_index.ptrConst(&o.builder).global.toConst();
            },
            .int => {
                var bigint_space: Value.BigIntSpace = undefined;
                const bigint = val.toBigInt(&bigint_space, zcu);
                return lowerBigInt(o, ty, bigint);
            },
            .err => |err| {
                const int = try pt.getErrorValue(err.name);
                const llvm_int = try o.builder.intConst(try o.errorIntType(), int);
                return llvm_int;
            },
            .error_union => |error_union| {
                const err_val = switch (error_union.val) {
                    .err_name => |err_name| try pt.intern(.{ .err = .{
                        .ty = ty.errorUnionSet(zcu).toIntern(),
                        .name = err_name,
                    } }),
                    .payload => (try pt.intValue(try pt.errorIntType(), 0)).toIntern(),
                };
                const err_int_ty = try pt.errorIntType();
                const payload_type = ty.errorUnionPayload(zcu);
                if (!payload_type.hasRuntimeBitsIgnoreComptime(zcu)) {
                    // We use the error type directly as the type.
                    return o.lowerValue(err_val);
                }

                const payload_align = payload_type.abiAlignment(zcu);
                const error_align = err_int_ty.abiAlignment(zcu);
                const llvm_error_value = try o.lowerValue(err_val);
                const llvm_payload_value = try o.lowerValue(switch (error_union.val) {
                    .err_name => try pt.intern(.{ .undef = payload_type.toIntern() }),
                    .payload => |payload| payload,
                });

                var fields: [3]Builder.Type = undefined;
                var vals: [3]Builder.Constant = undefined;
                if (error_align.compare(.gt, payload_align)) {
                    vals[0] = llvm_error_value;
                    vals[1] = llvm_payload_value;
                } else {
                    vals[0] = llvm_payload_value;
                    vals[1] = llvm_error_value;
                }
                fields[0] = vals[0].typeOf(&o.builder);
                fields[1] = vals[1].typeOf(&o.builder);

                const llvm_ty = try o.lowerType(ty);
                const llvm_ty_fields = llvm_ty.structFields(&o.builder);
                if (llvm_ty_fields.len > 2) {
                    assert(llvm_ty_fields.len == 3);
                    fields[2] = llvm_ty_fields[2];
                    vals[2] = try o.builder.undefConst(fields[2]);
                }
                return o.builder.structConst(try o.builder.structType(
                    llvm_ty.structKind(&o.builder),
                    fields[0..llvm_ty_fields.len],
                ), vals[0..llvm_ty_fields.len]);
            },
            .enum_tag => |enum_tag| o.lowerValue(enum_tag.int),
            .float => switch (ty.floatBits(target)) {
                16 => if (backendSupportsF16(target))
                    try o.builder.halfConst(val.toFloat(f16, zcu))
                else
                    try o.builder.intConst(.i16, @as(i16, @bitCast(val.toFloat(f16, zcu)))),
                32 => try o.builder.floatConst(val.toFloat(f32, zcu)),
                64 => try o.builder.doubleConst(val.toFloat(f64, zcu)),
                80 => if (backendSupportsF80(target))
                    try o.builder.x86_fp80Const(val.toFloat(f80, zcu))
                else
                    try o.builder.intConst(.i80, @as(i80, @bitCast(val.toFloat(f80, zcu)))),
                128 => try o.builder.fp128Const(val.toFloat(f128, zcu)),
                else => unreachable,
            },
            .ptr => try o.lowerPtr(arg_val, 0),
            .slice => |slice| return o.builder.structConst(try o.lowerType(ty), &.{
                try o.lowerValue(slice.ptr),
                try o.lowerValue(slice.len),
            }),
            .opt => |opt| {
                comptime assert(optional_layout_version == 3);
                const payload_ty = ty.optionalChild(zcu);

                const non_null_bit = try o.builder.intConst(.i8, @intFromBool(opt.val != .none));
                if (!payload_ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                    return non_null_bit;
                }
                const llvm_ty = try o.lowerType(ty);
                if (ty.optionalReprIsPayload(zcu)) return switch (opt.val) {
                    .none => switch (llvm_ty.tag(&o.builder)) {
                        .integer => try o.builder.intConst(llvm_ty, 0),
                        .pointer => try o.builder.nullConst(llvm_ty),
                        .structure => try o.builder.zeroInitConst(llvm_ty),
                        else => unreachable,
                    },
                    else => |payload| try o.lowerValue(payload),
                };
                assert(payload_ty.zigTypeTag(zcu) != .@"fn");

                var fields: [3]Builder.Type = undefined;
                var vals: [3]Builder.Constant = undefined;
                vals[0] = try o.lowerValue(switch (opt.val) {
                    .none => try pt.intern(.{ .undef = payload_ty.toIntern() }),
                    else => |payload| payload,
                });
                vals[1] = non_null_bit;
                fields[0] = vals[0].typeOf(&o.builder);
                fields[1] = vals[1].typeOf(&o.builder);

                const llvm_ty_fields = llvm_ty.structFields(&o.builder);
                if (llvm_ty_fields.len > 2) {
                    assert(llvm_ty_fields.len == 3);
                    fields[2] = llvm_ty_fields[2];
                    vals[2] = try o.builder.undefConst(fields[2]);
                }
                return o.builder.structConst(try o.builder.structType(
                    llvm_ty.structKind(&o.builder),
                    fields[0..llvm_ty_fields.len],
                ), vals[0..llvm_ty_fields.len]);
            },
            .aggregate => |aggregate| switch (ip.indexToKey(ty.toIntern())) {
                .array_type => |array_type| switch (aggregate.storage) {
                    .bytes => |bytes| try o.builder.stringConst(try o.builder.string(
                        bytes.toSlice(array_type.lenIncludingSentinel(), ip),
                    )),
                    .elems => |elems| {
                        const array_ty = try o.lowerType(ty);
                        const elem_ty = array_ty.childType(&o.builder);
                        assert(elems.len == array_ty.aggregateLen(&o.builder));

                        const ExpectedContents = extern struct {
                            vals: [Builder.expected_fields_len]Builder.Constant,
                            fields: [Builder.expected_fields_len]Builder.Type,
                        };
                        var stack align(@max(
                            @alignOf(std.heap.StackFallbackAllocator(0)),
                            @alignOf(ExpectedContents),
                        )) = std.heap.stackFallback(@sizeOf(ExpectedContents), o.gpa);
                        const allocator = stack.get();
                        const vals = try allocator.alloc(Builder.Constant, elems.len);
                        defer allocator.free(vals);
                        const fields = try allocator.alloc(Builder.Type, elems.len);
                        defer allocator.free(fields);

                        var need_unnamed = false;
                        for (vals, fields, elems) |*result_val, *result_field, elem| {
                            result_val.* = try o.lowerValue(elem);
                            result_field.* = result_val.typeOf(&o.builder);
                            if (result_field.* != elem_ty) need_unnamed = true;
                        }
                        return if (need_unnamed) try o.builder.structConst(
                            try o.builder.structType(.normal, fields),
                            vals,
                        ) else try o.builder.arrayConst(array_ty, vals);
                    },
                    .repeated_elem => |elem| {
                        const len: usize = @intCast(array_type.len);
                        const len_including_sentinel: usize = @intCast(array_type.lenIncludingSentinel());
                        const array_ty = try o.lowerType(ty);
                        const elem_ty = array_ty.childType(&o.builder);

                        const ExpectedContents = extern struct {
                            vals: [Builder.expected_fields_len]Builder.Constant,
                            fields: [Builder.expected_fields_len]Builder.Type,
                        };
                        var stack align(@max(
                            @alignOf(std.heap.StackFallbackAllocator(0)),
                            @alignOf(ExpectedContents),
                        )) = std.heap.stackFallback(@sizeOf(ExpectedContents), o.gpa);
                        const allocator = stack.get();
                        const vals = try allocator.alloc(Builder.Constant, len_including_sentinel);
                        defer allocator.free(vals);
                        const fields = try allocator.alloc(Builder.Type, len_including_sentinel);
                        defer allocator.free(fields);

                        var need_unnamed = false;
                        @memset(vals[0..len], try o.lowerValue(elem));
                        @memset(fields[0..len], vals[0].typeOf(&o.builder));
                        if (fields[0] != elem_ty) need_unnamed = true;

                        if (array_type.sentinel != .none) {
                            vals[len] = try o.lowerValue(array_type.sentinel);
                            fields[len] = vals[len].typeOf(&o.builder);
                            if (fields[len] != elem_ty) need_unnamed = true;
                        }

                        return if (need_unnamed) try o.builder.structConst(
                            try o.builder.structType(.@"packed", fields),
                            vals,
                        ) else try o.builder.arrayConst(array_ty, vals);
                    },
                },
                .vector_type => |vector_type| {
                    const vector_ty = try o.lowerType(ty);
                    switch (aggregate.storage) {
                        .bytes, .elems => {
                            const ExpectedContents = [Builder.expected_fields_len]Builder.Constant;
                            var stack align(@max(
                                @alignOf(std.heap.StackFallbackAllocator(0)),
                                @alignOf(ExpectedContents),
                            )) = std.heap.stackFallback(@sizeOf(ExpectedContents), o.gpa);
                            const allocator = stack.get();
                            const vals = try allocator.alloc(Builder.Constant, vector_type.len);
                            defer allocator.free(vals);

                            switch (aggregate.storage) {
                                .bytes => |bytes| for (vals, bytes.toSlice(vector_type.len, ip)) |*result_val, byte| {
                                    result_val.* = try o.builder.intConst(.i8, byte);
                                },
                                .elems => |elems| for (vals, elems) |*result_val, elem| {
                                    result_val.* = try o.lowerValue(elem);
                                },
                                .repeated_elem => unreachable,
                            }
                            return o.builder.vectorConst(vector_ty, vals);
                        },
                        .repeated_elem => |elem| return o.builder.splatConst(
                            vector_ty,
                            try o.lowerValue(elem),
                        ),
                    }
                },
                .tuple_type => |tuple| {
                    const struct_ty = try o.lowerType(ty);
                    const llvm_len = struct_ty.aggregateLen(&o.builder);

                    const ExpectedContents = extern struct {
                        vals: [Builder.expected_fields_len]Builder.Constant,
                        fields: [Builder.expected_fields_len]Builder.Type,
                    };
                    var stack align(@max(
                        @alignOf(std.heap.StackFallbackAllocator(0)),
                        @alignOf(ExpectedContents),
                    )) = std.heap.stackFallback(@sizeOf(ExpectedContents), o.gpa);
                    const allocator = stack.get();
                    const vals = try allocator.alloc(Builder.Constant, llvm_len);
                    defer allocator.free(vals);
                    const fields = try allocator.alloc(Builder.Type, llvm_len);
                    defer allocator.free(fields);

                    comptime assert(struct_layout_version == 2);
                    var llvm_index: usize = 0;
                    var offset: u64 = 0;
                    var big_align: InternPool.Alignment = .none;
                    var need_unnamed = false;
                    for (
                        tuple.types.get(ip),
                        tuple.values.get(ip),
                        0..,
                    ) |field_ty, field_val, field_index| {
                        if (field_val != .none) continue;
                        if (!Type.fromInterned(field_ty).hasRuntimeBitsIgnoreComptime(zcu)) continue;

                        const field_align = Type.fromInterned(field_ty).abiAlignment(zcu);
                        big_align = big_align.max(field_align);
                        const prev_offset = offset;
                        offset = field_align.forward(offset);

                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) {
                            // TODO make this and all other padding elsewhere in debug
                            // builds be 0xaa not undef.
                            fields[llvm_index] = try o.builder.arrayType(padding_len, .i8);
                            vals[llvm_index] = try o.builder.undefConst(fields[llvm_index]);
                            assert(fields[llvm_index] == struct_ty.structFields(&o.builder)[llvm_index]);
                            llvm_index += 1;
                        }

                        vals[llvm_index] =
                            try o.lowerValue((try val.fieldValue(pt, field_index)).toIntern());
                        fields[llvm_index] = vals[llvm_index].typeOf(&o.builder);
                        if (fields[llvm_index] != struct_ty.structFields(&o.builder)[llvm_index])
                            need_unnamed = true;
                        llvm_index += 1;

                        offset += Type.fromInterned(field_ty).abiSize(zcu);
                    }
                    {
                        const prev_offset = offset;
                        offset = big_align.forward(offset);
                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) {
                            fields[llvm_index] = try o.builder.arrayType(padding_len, .i8);
                            vals[llvm_index] = try o.builder.undefConst(fields[llvm_index]);
                            assert(fields[llvm_index] == struct_ty.structFields(&o.builder)[llvm_index]);
                            llvm_index += 1;
                        }
                    }
                    assert(llvm_index == llvm_len);

                    return o.builder.structConst(if (need_unnamed)
                        try o.builder.structType(struct_ty.structKind(&o.builder), fields)
                    else
                        struct_ty, vals);
                },
                .struct_type => {
                    const struct_type = ip.loadStructType(ty.toIntern());
                    assert(struct_type.haveLayout(ip));
                    const struct_ty = try o.lowerType(ty);
                    if (struct_type.layout == .@"packed") {
                        comptime assert(Type.packed_struct_layout_version == 2);

                        const bits = ty.bitSize(zcu);
                        const llvm_int_ty = try o.builder.intType(@intCast(bits));

                        return o.lowerValueToInt(llvm_int_ty, arg_val);
                    }
                    const llvm_len = struct_ty.aggregateLen(&o.builder);

                    const ExpectedContents = extern struct {
                        vals: [Builder.expected_fields_len]Builder.Constant,
                        fields: [Builder.expected_fields_len]Builder.Type,
                    };
                    var stack align(@max(
                        @alignOf(std.heap.StackFallbackAllocator(0)),
                        @alignOf(ExpectedContents),
                    )) = std.heap.stackFallback(@sizeOf(ExpectedContents), o.gpa);
                    const allocator = stack.get();
                    const vals = try allocator.alloc(Builder.Constant, llvm_len);
                    defer allocator.free(vals);
                    const fields = try allocator.alloc(Builder.Type, llvm_len);
                    defer allocator.free(fields);

                    comptime assert(struct_layout_version == 2);
                    var llvm_index: usize = 0;
                    var offset: u64 = 0;
                    var big_align: InternPool.Alignment = .@"1";
                    var need_unnamed = false;
                    var field_it = struct_type.iterateRuntimeOrder(ip);
                    while (field_it.next()) |field_index| {
                        const field_ty = Type.fromInterned(struct_type.field_types.get(ip)[field_index]);
                        const field_align = ty.fieldAlignment(field_index, zcu);
                        big_align = big_align.max(field_align);
                        const prev_offset = offset;
                        offset = field_align.forward(offset);

                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) {
                            // TODO make this and all other padding elsewhere in debug
                            // builds be 0xaa not undef.
                            fields[llvm_index] = try o.builder.arrayType(padding_len, .i8);
                            vals[llvm_index] = try o.builder.undefConst(fields[llvm_index]);
                            assert(fields[llvm_index] ==
                                struct_ty.structFields(&o.builder)[llvm_index]);
                            llvm_index += 1;
                        }

                        if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                            // This is a zero-bit field - we only needed it for the alignment.
                            continue;
                        }

                        vals[llvm_index] = try o.lowerValue(
                            (try val.fieldValue(pt, field_index)).toIntern(),
                        );
                        fields[llvm_index] = vals[llvm_index].typeOf(&o.builder);
                        if (fields[llvm_index] != struct_ty.structFields(&o.builder)[llvm_index])
                            need_unnamed = true;
                        llvm_index += 1;

                        offset += field_ty.abiSize(zcu);
                    }
                    {
                        const prev_offset = offset;
                        offset = big_align.forward(offset);
                        const padding_len = offset - prev_offset;
                        if (padding_len > 0) {
                            fields[llvm_index] = try o.builder.arrayType(padding_len, .i8);
                            vals[llvm_index] = try o.builder.undefConst(fields[llvm_index]);
                            assert(fields[llvm_index] == struct_ty.structFields(&o.builder)[llvm_index]);
                            llvm_index += 1;
                        }
                    }
                    assert(llvm_index == llvm_len);

                    return o.builder.structConst(if (need_unnamed)
                        try o.builder.structType(struct_ty.structKind(&o.builder), fields)
                    else
                        struct_ty, vals);
                },
                else => unreachable,
            },
            .un => |un| {
                const union_ty = try o.lowerType(ty);
                const layout = ty.unionGetLayout(zcu);
                if (layout.payload_size == 0) return o.lowerValue(un.tag);

                const union_obj = zcu.typeToUnion(ty).?;
                const container_layout = union_obj.flagsUnordered(ip).layout;

                var need_unnamed = false;
                const payload = if (un.tag != .none) p: {
                    const field_index = zcu.unionTagFieldIndex(union_obj, Value.fromInterned(un.tag)).?;
                    const field_ty = Type.fromInterned(union_obj.field_types.get(ip)[field_index]);
                    if (container_layout == .@"packed") {
                        if (!field_ty.hasRuntimeBits(zcu)) return o.builder.intConst(union_ty, 0);
                        const bits = ty.bitSize(zcu);
                        const llvm_int_ty = try o.builder.intType(@intCast(bits));

                        return o.lowerValueToInt(llvm_int_ty, arg_val);
                    }

                    // Sometimes we must make an unnamed struct because LLVM does
                    // not support bitcasting our payload struct to the true union payload type.
                    // Instead we use an unnamed struct and every reference to the global
                    // must pointer cast to the expected type before accessing the union.
                    need_unnamed = layout.most_aligned_field != field_index;

                    if (!field_ty.hasRuntimeBitsIgnoreComptime(zcu)) {
                        const padding_len = layout.payload_size;
                        break :p try o.builder.undefConst(try o.builder.arrayType(padding_len, .i8));
                    }
                    const payload = try o.lowerValue(un.val);
                    const payload_ty = payload.typeOf(&o.builder);
                    if (payload_ty != union_ty.structFields(&o.builder)[
                        @intFromBool(layout.tag_align.compare(.gte, layout.payload_align))
                    ]) need_unnamed = true;
                    const field_size = field_ty.abiSize(zcu);
                    if (field_size == layout.payload_size) break :p payload;
                    const padding_len = layout.payload_size - field_size;
                    const padding_ty = try o.builder.arrayType(padding_len, .i8);
                    break :p try o.builder.structConst(
                        try o.builder.structType(.@"packed", &.{ payload_ty, padding_ty }),
                        &.{ payload, try o.builder.undefConst(padding_ty) },
                    );
                } else p: {
                    assert(layout.tag_size == 0);
                    if (container_layout == .@"packed") {
                        const bits = ty.bitSize(zcu);
                        const llvm_int_ty = try o.builder.intType(@intCast(bits));

                        return o.lowerValueToInt(llvm_int_ty, arg_val);
                    }

                    const union_val = try o.lowerValue(un.val);
                    need_unnamed = true;
                    break :p union_val;
                };

                const payload_ty = payload.typeOf(&o.builder);
                if (layout.tag_size == 0) return o.builder.structConst(if (need_unnamed)
                    try o.builder.structType(union_ty.structKind(&o.builder), &.{payload_ty})
                else
                    union_ty, &.{payload});
                const tag = try o.lowerValue(un.tag);
                const tag_ty = tag.typeOf(&o.builder);
                var fields: [3]Builder.Type = undefined;
                var vals: [3]Builder.Constant = undefined;
                var len: usize = 2;
                if (layout.tag_align.compare(.gte, layout.payload_align)) {
                    fields = .{ tag_ty, payload_ty, undefined };
                    vals = .{ tag, payload, undefined };
                } else {
                    fields = .{ payload_ty, tag_ty, undefined };
                    vals = .{ payload, tag, undefined };
                }
                if (layout.padding != 0) {
                    fields[2] = try o.builder.arrayType(layout.padding, .i8);
                    vals[2] = try o.builder.undefConst(fields[2]);
                    len = 3;
                }
                return o.builder.structConst(if (need_unnamed)
                    try o.builder.structType(union_ty.structKind(&o.builder), fields[0..len])
                else
                    union_ty, vals[0..len]);
            },
            .memoized_call => unreachable,
        };
    }

    fn lowerBigInt(
        o: *Object,
        ty: Type,
        bigint: std.math.big.int.Const,
    ) Allocator.Error!Builder.Constant {
        const zcu = o.pt.zcu;
        return o.builder.bigIntConst(try o.builder.intType(ty.intInfo(zcu).bits), bigint);
    }

    fn lowerPtr(
        o: *Object,
        ptr_val: InternPool.Index,
        prev_offset: u64,
    ) Error!Builder.Constant {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ptr = zcu.intern_pool.indexToKey(ptr_val).ptr;
        const offset: u64 = prev_offset + ptr.byte_offset;
        return switch (ptr.base_addr) {
            .nav => |nav| {
                const base_ptr = try o.lowerNavRefValue(nav);
                return o.builder.gepConst(.inbounds, .i8, base_ptr, null, &.{
                    try o.builder.intConst(.i64, offset),
                });
            },
            .uav => |uav| {
                const base_ptr = try o.lowerUavRef(uav);
                return o.builder.gepConst(.inbounds, .i8, base_ptr, null, &.{
                    try o.builder.intConst(.i64, offset),
                });
            },
            .int => try o.builder.castConst(
                .inttoptr,
                try o.builder.intConst(try o.lowerType(Type.usize), offset),
                try o.lowerType(Type.fromInterned(ptr.ty)),
            ),
            .eu_payload => |eu_ptr| try o.lowerPtr(
                eu_ptr,
                offset + @import("../codegen.zig").errUnionPayloadOffset(
                    Value.fromInterned(eu_ptr).typeOf(zcu).childType(zcu),
                    zcu,
                ),
            ),
            .opt_payload => |opt_ptr| try o.lowerPtr(opt_ptr, offset),
            .field => |field| {
                const agg_ty = Value.fromInterned(field.base).typeOf(zcu).childType(zcu);
                const field_off: u64 = switch (agg_ty.zigTypeTag(zcu)) {
                    .pointer => off: {
                        assert(agg_ty.isSlice(zcu));
                        break :off switch (field.index) {
                            Value.slice_ptr_index => 0,
                            Value.slice_len_index => @divExact(zcu.getTarget().ptrBitWidth(), 8),
                            else => unreachable,
                        };
                    },
                    .@"struct", .@"union" => switch (agg_ty.containerLayout(zcu)) {
                        .auto => agg_ty.structFieldOffset(@intCast(field.index), zcu),
                        .@"extern", .@"packed" => unreachable,
                    },
                    else => unreachable,
                };
                return o.lowerPtr(field.base, offset + field_off);
            },
            .arr_elem, .comptime_field, .comptime_alloc => unreachable,
        };
    }

    /// This logic is very similar to `lowerNavRefValue` but for anonymous declarations.
    /// Maybe the logic could be unified.
    fn lowerUavRef(
        o: *Object,
        uav: InternPool.Key.Ptr.BaseAddr.Uav,
    ) Error!Builder.Constant {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const uav_val = uav.val;
        const uav_ty = Type.fromInterned(ip.typeOf(uav_val));
        const target = zcu.getTarget();

        switch (ip.indexToKey(uav_val)) {
            .func => @panic("TODO"),
            .@"extern" => @panic("TODO"),
            else => {},
        }

        const ptr_ty = Type.fromInterned(uav.orig_ty);

        const is_fn_body = uav_ty.zigTypeTag(zcu) == .@"fn";
        if ((!is_fn_body and !uav_ty.hasRuntimeBits(zcu)) or
            (is_fn_body and zcu.typeToFunc(uav_ty).?.is_generic)) return o.lowerPtrToVoid(ptr_ty);

        if (is_fn_body)
            @panic("TODO");

        const llvm_addr_space = toLlvmAddressSpace(ptr_ty.ptrAddressSpace(zcu), target);
        const alignment = ptr_ty.ptrAlignment(zcu);
        const llvm_global = (try o.resolveGlobalUav(uav.val, llvm_addr_space, alignment)).ptrConst(&o.builder).global;

        const llvm_val = try o.builder.convConst(
            llvm_global.toConst(),
            try o.builder.ptrType(llvm_addr_space),
        );

        return o.builder.convConst(llvm_val, try o.lowerType(ptr_ty));
    }

    fn lowerNavRefValue(o: *Object, nav_index: InternPool.Nav.Index) Allocator.Error!Builder.Constant {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;

        const nav = ip.getNav(nav_index);

        const nav_ty = Type.fromInterned(nav.typeOf(ip));
        const ptr_ty = try pt.navPtrType(nav_index);

        const is_fn_body = nav_ty.zigTypeTag(zcu) == .@"fn";
        if ((!is_fn_body and !nav_ty.hasRuntimeBits(zcu)) or
            (is_fn_body and zcu.typeToFunc(nav_ty).?.is_generic))
        {
            return o.lowerPtrToVoid(ptr_ty);
        }

        const llvm_global = if (is_fn_body)
            (try o.resolveLlvmFunction(nav_index)).ptrConst(&o.builder).global
        else
            (try o.resolveGlobalNav(nav_index)).ptrConst(&o.builder).global;

        const llvm_val = try o.builder.convConst(
            llvm_global.toConst(),
            try o.builder.ptrType(toLlvmAddressSpace(nav.getAddrspace(), zcu.getTarget())),
        );

        return o.builder.convConst(llvm_val, try o.lowerType(ptr_ty));
    }

    fn lowerPtrToVoid(o: *Object, ptr_ty: Type) Allocator.Error!Builder.Constant {
        const zcu = o.pt.zcu;
        // Even though we are pointing at something which has zero bits (e.g. `void`),
        // Pointers are defined to have bits. So we must return something here.
        // The value cannot be undefined, because we use the `nonnull` annotation
        // for non-optional pointers. We also need to respect the alignment, even though
        // the address will never be dereferenced.
        const int: u64 = ptr_ty.ptrInfo(zcu).flags.alignment.toByteUnits() orelse
            // Note that these 0xaa values are appropriate even in release-optimized builds
            // because we need a well-defined value that is not null, and LLVM does not
            // have an "undef_but_not_null" attribute. As an example, if this `alloc` AIR
            // instruction is followed by a `wrap_optional`, it will return this value
            // verbatim, and the result should test as non-null.
            switch (zcu.getTarget().ptrBitWidth()) {
                16 => 0xaaaa,
                32 => 0xaaaaaaaa,
                64 => 0xaaaaaaaa_aaaaaaaa,
                else => unreachable,
            };
        const llvm_usize = try o.lowerType(Type.usize);
        const llvm_ptr_ty = try o.lowerType(ptr_ty);
        return o.builder.castConst(.inttoptr, try o.builder.intConst(llvm_usize, int), llvm_ptr_ty);
    }

    /// If the operand type of an atomic operation is not byte sized we need to
    /// widen it before using it and then truncate the result.
    /// RMW exchange of floating-point values is bitcasted to same-sized integer
    /// types to work around a LLVM deficiency when targeting ARM/AArch64.
    fn getAtomicAbiType(o: *Object, ty: Type, is_rmw_xchg: bool) Allocator.Error!Builder.Type {
        const pt = o.pt;
        const zcu = pt.zcu;
        const int_ty = switch (ty.zigTypeTag(zcu)) {
            .int => ty,
            .@"enum" => ty.intTagType(zcu),
            .float => {
                if (!is_rmw_xchg) return .none;
                return o.builder.intType(@intCast(ty.abiSize(zcu) * 8));
            },
            .bool => return .i8,
            else => return .none,
        };
        const bit_count = int_ty.intInfo(zcu).bits;
        if (!std.math.isPowerOfTwo(bit_count) or (bit_count % 8) != 0) {
            return o.builder.intType(@intCast(int_ty.abiSize(zcu) * 8));
        } else {
            return .none;
        }
    }

    fn addByValParamAttrs(
        o: *Object,
        attributes: *Builder.FunctionAttributes.Wip,
        param_ty: Type,
        param_index: u32,
        fn_info: InternPool.Key.FuncType,
        llvm_arg_i: u32,
    ) Allocator.Error!void {
        const pt = o.pt;
        const zcu = pt.zcu;
        if (param_ty.isPtrAtRuntime(zcu)) {
            const ptr_info = param_ty.ptrInfo(zcu);
            if (math.cast(u5, param_index)) |i| {
                if (@as(u1, @truncate(fn_info.noalias_bits >> i)) != 0) {
                    try attributes.addParamAttr(llvm_arg_i, .@"noalias", &o.builder);
                }
            }
            if (!param_ty.isPtrLikeOptional(zcu) and !ptr_info.flags.is_allowzero) {
                try attributes.addParamAttr(llvm_arg_i, .nonnull, &o.builder);
            }
            switch (fn_info.cc) {
                else => {},
                .x86_64_interrupt,
                .x86_interrupt,
                => {
                    const child_type = try lowerType(o, Type.fromInterned(ptr_info.child));
                    try attributes.addParamAttr(llvm_arg_i, .{ .byval = child_type }, &o.builder);
                },
            }
            if (ptr_info.flags.is_const) {
                try attributes.addParamAttr(llvm_arg_i, .readonly, &o.builder);
            }
            const elem_align = if (ptr_info.flags.alignment != .none)
                ptr_info.flags.alignment
            else
                Type.fromInterned(ptr_info.child).abiAlignment(zcu).max(.@"1");
            try attributes.addParamAttr(llvm_arg_i, .{ .@"align" = elem_align.toLlvm() }, &o.builder);
        } else if (ccAbiPromoteInt(fn_info.cc, zcu, param_ty)) |s| switch (s) {
            .signed => try attributes.addParamAttr(llvm_arg_i, .signext, &o.builder),
            .unsigned => try attributes.addParamAttr(llvm_arg_i, .zeroext, &o.builder),
        };
    }

    fn addByRefParamAttrs(
        o: *Object,
        attributes: *Builder.FunctionAttributes.Wip,
        llvm_arg_i: u32,
        alignment: Builder.Alignment,
        byval: bool,
        param_llvm_ty: Builder.Type,
    ) Allocator.Error!void {
        try attributes.addParamAttr(llvm_arg_i, .nonnull, &o.builder);
        try attributes.addParamAttr(llvm_arg_i, .readonly, &o.builder);
        try attributes.addParamAttr(llvm_arg_i, .{ .@"align" = alignment }, &o.builder);
        if (byval) try attributes.addParamAttr(llvm_arg_i, .{ .byval = param_llvm_ty }, &o.builder);
    }

    fn llvmFieldIndex(o: *Object, struct_ty: Type, field_index: usize) ?c_uint {
        return o.struct_field_map.get(.{
            .struct_ty = struct_ty.toIntern(),
            .field_index = @intCast(field_index),
        });
    }

    fn getCmpLtErrorsLenFunction(o: *Object) !Builder.Function.Index {
        const name = try o.builder.strtabString(lt_errors_fn_name);
        if (o.builder.getGlobal(name)) |llvm_fn| return llvm_fn.ptrConst(&o.builder).kind.function;

        const zcu = o.pt.zcu;
        const target = zcu.root_mod.resolved_target.result;
        const function_index = try o.builder.addFunction(
            try o.builder.fnType(.i1, &.{try o.errorIntType()}, .normal),
            name,
            toLlvmAddressSpace(.generic, target),
        );

        var attributes: Builder.FunctionAttributes.Wip = .{};
        defer attributes.deinit(&o.builder);
        try o.addCommonFnAttributes(&attributes, zcu.root_mod, zcu.root_mod.omit_frame_pointer);

        function_index.setLinkage(.internal, &o.builder);
        function_index.setCallConv(.fastcc, &o.builder);
        function_index.setAttributes(try attributes.finish(&o.builder), &o.builder);
        return function_index;
    }

    fn getEnumTagNameFunction(o: *Object, enum_ty: Type) !Builder.Function.Index {
        const pt = o.pt;
        const zcu = pt.zcu;
        const ip = &zcu.intern_pool;
        const enum_type = ip.loadEnumType(enum_ty.toIntern());

        const gop = try o.enum_tag_name_map.getOrPut(o.gpa, enum_ty.toIntern());
        if (gop.found_existing) return gop.value_ptr.ptrConst(&o.builder).kind.function;
        errdefer assert(o.enum_tag_name_map.remove(enum_ty.toIntern()));

        const usize_ty = try o.lowerType(Type.usize);
        const ret_ty = try o.lowerType(Type.slice_const_u8_sentinel_0);
        const target = zcu.root_mod.resolved_target.result;
        const function_index = try o.builder.addFunction(
            try o.builder.fnType(ret_ty, &.{try o.lowerType(Type.fromInterned(enum_type.tag_ty))}, .normal),
            try o.builder.strtabStringFmt("__zig_tag_name_{}", .{enum_type.name.fmt(ip)}),
            toLlvmAddressSpace(.generic, target),
        );

        var attributes: Builder.FunctionAttributes.Wip = .{};
        defer attributes.deinit(&o.builder);
        try o.addCommonFnAttributes(&attributes, zcu.root_mod, zcu.root_mod.omit_frame_pointer);

        function_index.setLinkage(.internal, &o.builder);
        function_index.setCallConv(.fastcc, &o.builder);
        function_index.setAttributes(try attributes.finish(&o.builder), &o.builder);
        gop.value_ptr.* = function_index.ptrConst(&o.builder).global;

        var wip = try Builder.WipFunction.init(&o.builder, .{
            .function = function_index,
            .strip = true,
        });
        defer wip.deinit();
        wip.cursor = .{ .block = try wip.block(0, "Entry") };

        const bad_value_block = try wip.block(1, "BadValue");
        const tag_int_value = wip.arg(0);
        var wip_switch =
            try wip.@"switch"(tag_int_value, bad_value_block, @intCast(enum_type.names.len), .none);
        defer wip_switch.finish(&wip);

        for (0..enum_type.names.len) |field_index| {
            const name = try o.builder.stringNull(enum_type.names.get(ip)[field_index].toSlice(ip));
            const name_init = try o.builder.stringConst(name);
            const name_variable_index =
                try o.builder.addVariable(.empty, name_init.typeOf(&o.builder), .default);
            try name_variable_index.setInitializer(name_init, &o.builder);
            name_variable_index.setLinkage(.private, &o.builder);
            name_variable_index.setMutability(.constant, &o.builder);
            name_variable_index.setUnnamedAddr(.unnamed_addr, &o.builder);
            name_variable_index.setAlignment(comptime Builder.Alignment.fromByteUnits(1), &o.builder);

            const name_val = try o.builder.structValue(ret_ty, &.{
                name_variable_index.toConst(&o.builder),
                try o.builder.intConst(usize_ty, name.slice(&o.builder).?.len - 1),
            });

            const return_block = try wip.block(1, "Name");
            const this_tag_int_value = try o.lowerValue(
                (try pt.enumValueFieldIndex(enum_ty, @intCast(field_index))).toIntern(),
            );
            try wip_switch.addCase(this_tag_int_value, return_block, &wip);

            wip.cursor = .{ .block = return_block };
            _ = try wip.ret(name_val);
        }

        wip.cursor = .{ .block = bad_value_block };
        _ = try wip.@"unreachable"();

        try wip.finish();
        return function_index;
    }
}```
嗯，我现在需要总结这段Zig代码的思维模型。首先，我得看看这段代码属于哪个模块，实现了什么功能。看起来这是一个Zig语言中使用LLVM后端的对象生成模块，可能和编译器代码生成有关。

首先，代码定义了一个名为Object的结构体，包含了很多字段，比如gpa（通用分配器）、builder（LLVM构建器）、debug相关的元数据，以及各种映射表，比如nav_map、uav_map等。这些映射表可能用于跟踪不同的声明和全局变量。

接着，Object结构体有很多方法，比如create、deinit、emit、updateFunc、updateNav等。这些方法似乎是用来创建和更新LLVM对象，生成调试信息，处理导出符号等。

其中，create函数负责初始化Object实例，设置调试信息和目标平台相关的内容。deinit函数用于释放资源，清理各种映射表。emit函数可能负责将生成的LLVM模块输出成目标文件或汇编代码。

updateFunc方法看起来是更新或生成某个函数的LLVM IR，处理参数属性、调试信息，以及错误处理。这里涉及到很多LLVM的底层操作，比如设置函数参数属性、生成基本块、处理控制流等。

debug相关的部分，比如lowerDebugType，处理类型的调试信息生成，为不同的Zig类型生成对应的DWARF信息。这可能包括结构体、联合体、枚举等复杂类型的描述。

还有一些辅助函数，比如lowerType将Zig类型转换为LLVM类型，lowerValue处理值的转换，这可能包括整数、浮点数、指针等不同类型的处理。

此外，代码中涉及很多错误处理和内存管理的内容，比如使用Allocator进行内存分配，错误处理使用error union，确保资源正确释放。

需要理解的关键点包括：

1. Object结构体如何管理LLVM构建过程中的各种状态和资源。
2. 如何将Zig的类型系统和语言特性映射到LLVM的IR。
3. 调试信息是如何生成和管理的，特别是复杂类型的处理。
4. 如何处理函数参数和返回值的ABI，比如结构体按值传递、切片类型的处理等。
5. 错误处理和内存管理的模式，确保代码的健壮性。

可能存在的难点包括理解LLVM API的使用方式，Zig类型到LLVM类型的转换逻辑，以及调试信息生成的细节。此外，代码中的多线程处理（比如PerThread结构）和并发问题也需要关注。

总结来说，这段代码是Zig编译器后端的一部分，负责将Zig的高级结构转换为LLVM IR，并生成相应的调试信息，确保生成的代码符合目标平台的ABI和优化要求。
================================================
这段代码是 Zig 编译器后端的重要组成部分，负责将 Zig 的高级语义转换为 LLVM IR，并管理调试信息、符号导出和代码生成。以下是其核心思维模型的总结：

---

### **1. 核心数据结构：`Object`**
- **功能**：管理代码生成的全局状态和资源。
- **关键字段**：
  - **`gpa`**：通用内存分配器，用于动态内存管理。
  - **`builder`**：LLVM IR 构建器，封装 LLVM 的 API 操作。
  - **调试信息**：如 `debug_compile_unit`、`debug_type_map`，用于生成 DWARF 调试信息。
  - **映射表**：如 `nav_map`（命名声明映射）、`uav_map`（匿名全局变量映射）、`type_map`（类型缓存），用于高效查询已生成的 LLVM 实体。
  - **目标信息**：`target` 包含目标平台的配置（如 CPU、ABI）。

---

### **2. 核心流程**
#### **2.1 初始化与销毁**
- **`create`**：初始化 `Object` 实例，设置调试单元、目标三元组（Triple）、数据布局等。
- **`deinit`**：释放所有资源（如映射表、LLVM 构建器），防止内存泄漏。

#### **2.2 代码生成（`emit`）**
- **步骤**：
  1. **生成错误名称表**：`genErrorNameTable` 为全局错误集合生成符号表。
  2. **生成模块级汇编**：`genModuleLevelAssembly` 处理内联汇编代码。
  3. **处理 LLVM 模块标志**：如 PIC/PIE 级别、调试信息版本。
  4. **调用 LLVM API**：将中间表示（IR/BC）转换为目标文件（`.o`）、汇编（`.s`）等。
- **关键细节**：支持多阶段输出（如预处理 IR、优化后 IR），处理跨平台差异（如 Windows 的 DLL 导出）。

#### **2.3 函数更新（`updateFunc`）**
- **功能**：将 Zig 函数转换为 LLVM 函数，处理参数传递、属性设置、调试信息。
- **关键步骤**：
  - **参数处理**：根据调用约定（如 SRet 返回值、错误追踪）调整参数顺序和属性。
  - **调试信息**：为函数生成 `subprogram` 元数据，关联源码位置。
  - **控制流生成**：通过 `FuncGen` 生成基本块、指令，处理分支、循环、Switch 等结构。
  - **错误处理**：捕获代码生成错误并反馈到编译器诊断系统。

---

### **3. 类型与值处理**
#### **3.1 类型转换（`lowerType`）**
- **目标**：将 Zig 类型映射为 LLVM 类型。
- **处理逻辑**：
  - **基本类型**：直接映射（如 `i32` → `i32`，`bool` → `i1`）。
  - **复杂类型**：
    - **结构体/联合体**：按字段对齐生成 LLVM 结构，处理填充字节。
    - **切片**：转换为 `{ ptr, len }` 结构。
    - **可选类型**：根据是否包含有效负载（Payload）生成不同布局。
  - **调试类型**：`lowerDebugType` 为 DWARF 生成类型描述（如 `DICompositeType`）。

#### **3.2 值转换（`lowerValue`）**
- **目标**：将 Zig 常量值转换为 LLVM 常量。
- **处理逻辑**：
  - **整数/浮点数**：直接生成 LLVM 常量。
  - **指针/切片**：解析基地址和偏移，生成 `gep` 指令。
  - **聚合类型**：递归处理字段，生成结构体或数组常量。

---

### **4. 调试信息生成**
- **关键方法**：`lowerDebugType` 为所有 Zig 类型生成 DWARF 元数据。
- **实现细节**：
  - **结构体/联合体**：生成成员偏移、对齐信息。
  - **枚举**：生成标签值与名称的映射。
  - **函数**：生成参数类型、返回类型和源码位置信息。
- **作用**：支持调试器查看变量、单步执行等。

---

### **5. 符号与全局变量**
- **`updateExports`**：处理符号的导出（如 `export` 关键字），生成 LLVM 全局变量或别名。
- **`resolveGlobalNav`**：将 Zig 的命名声明（如全局变量）映射到 LLVM 全局变量。
- **`resolveGlobalUav`**：处理匿名全局变量（如常量切片），确保唯一性和正确内存布局。

---

### **6. 关键设计模式**
- **缓存机制**：通过 `type_map`、`struct_field_map` 缓存类型转换结果，避免重复计算。
- **错误处理**：统一使用 Zig 的错误联合（`error{...}`），结合 `try` 和 `catch` 确保资源安全。
- **多线程支持**：`PerThread` 结构可能用于隔离线程本地状态（如编译单元）。

---

### **7. 跨平台处理**
- **目标三元组**：通过 `targetTriple` 生成 LLVM 兼容的目标描述。
- **ABI 适配**：处理不同平台的调用约定（如 Windows 的 `RegCallv4`）、结构体对齐。
- **特殊指令**：如为 MIPS 关闭快速指令选择（`allow_fast_isel`）。

---

### **总结**
这段代码是 Zig 编译器后端的核心模块，负责将高级语言特性（如错误联合、可选类型、泛型）高效转换为 LLVM IR，同时确保调试信息的完整性和跨平台兼容性。其设计强调类型系统的精确映射、资源管理的安全性，以及通过缓存和优化提升生成效率。