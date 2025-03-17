```zig
fn buildOutputType(
    gpa: Allocator,
    arena: Allocator,
    all_args: []const []const u8,
    arg_mode: ArgMode,
) !void {
    var provided_name: ?[]const u8 = null;
    var root_src_file: ?[]const u8 = null;
    var version: std.SemanticVersion = .{ .major = 0, .minor = 0, .patch = 0 };
    var have_version = false;
    var compatibility_version: ?std.SemanticVersion = null;
    var function_sections = false;
    var data_sections = false;
    var listen: Listen = .none;
    var debug_compile_errors = false;
    var verbose_link = (native_os != .wasi or builtin.link_libc) and
        EnvVar.ZIG_VERBOSE_LINK.isSet();
    var verbose_cc = (native_os != .wasi or builtin.link_libc) and
        EnvVar.ZIG_VERBOSE_CC.isSet();
    var verbose_air = false;
    var verbose_intern_pool = false;
    var verbose_generic_instances = false;
    var verbose_llvm_ir: ?[]const u8 = null;
    var verbose_llvm_bc: ?[]const u8 = null;
    var verbose_cimport = false;
    var verbose_llvm_cpu_features = false;
    var time_report = false;
    var stack_report = false;
    var show_builtin = false;
    var emit_bin: EmitBin = .yes_default_path;
    var emit_asm: Emit = .no;
    var emit_llvm_ir: Emit = .no;
    var emit_llvm_bc: Emit = .no;
    var emit_docs: Emit = .no;
    var emit_implib: Emit = .yes_default_path;
    var emit_implib_arg_provided = false;
    var target_arch_os_abi: ?[]const u8 = null;
    var target_mcpu: ?[]const u8 = null;
    var emit_h: Emit = .no;
    var soname: SOName = undefined;
    var want_compiler_rt: ?bool = null;
    var want_ubsan_rt: ?bool = null;
    var linker_script: ?[]const u8 = null;
    var version_script: ?[]const u8 = null;
    var linker_repro: ?bool = null;
    var linker_allow_undefined_version: bool = false;
    var linker_enable_new_dtags: ?bool = null;
    var disable_c_depfile = false;
    var linker_sort_section: ?link.File.Elf.SortSection = null;
    var linker_gc_sections: ?bool = null;
    var linker_compress_debug_sections: ?link.File.Elf.CompressDebugSections = null;
    var linker_allow_shlib_undefined: ?bool = null;
    var allow_so_scripts: bool = false;
    var linker_bind_global_refs_locally: ?bool = null;
    var linker_import_symbols: bool = false;
    var linker_import_table: bool = false;
    var linker_export_table: bool = false;
    var linker_initial_memory: ?u64 = null;
    var linker_max_memory: ?u64 = null;
    var linker_global_base: ?u64 = null;
    var linker_print_gc_sections: bool = false;
    var linker_print_icf_sections: bool = false;
    var linker_print_map: bool = false;
    var llvm_opt_bisect_limit: c_int = -1;
    var linker_z_nocopyreloc = false;
    var linker_z_nodelete = false;
    var linker_z_notext = false;
    var linker_z_defs = false;
    var linker_z_origin = false;
    var linker_z_now = true;
    var linker_z_relro = true;
    var linker_z_common_page_size: ?u64 = null;
    var linker_z_max_page_size: ?u64 = null;
    var linker_tsaware = false;
    var linker_nxcompat = false;
    var linker_dynamicbase = true;
    var linker_optimization: ?[]const u8 = null;
    var linker_module_definition_file: ?[]const u8 = null;
    var test_no_exec = false;
    var entry: Compilation.CreateOptions.Entry = .default;
    var force_undefined_symbols: std.StringArrayHashMapUnmanaged(void) = .empty;
    var stack_size: ?u64 = null;
    var image_base: ?u64 = null;
    var link_eh_frame_hdr = false;
    var link_emit_relocs = false;
    var build_id: ?std.zig.BuildId = null;
    var runtime_args_start: ?usize = null;
    var test_filters: std.ArrayListUnmanaged([]const u8) = .empty;
    var test_name_prefix: ?[]const u8 = null;
    var test_runner_path: ?[]const u8 = null;
    var override_local_cache_dir: ?[]const u8 = try EnvVar.ZIG_LOCAL_CACHE_DIR.get(arena);
    var override_global_cache_dir: ?[]const u8 = try EnvVar.ZIG_GLOBAL_CACHE_DIR.get(arena);
    var override_lib_dir: ?[]const u8 = try EnvVar.ZIG_LIB_DIR.get(arena);
    var clang_preprocessor_mode: Compilation.ClangPreprocessorMode = .no;
    var subsystem: ?std.Target.SubSystem = null;
    var major_subsystem_version: ?u16 = null;
    var minor_subsystem_version: ?u16 = null;
    var mingw_unicode_entry_point: bool = false;
    var enable_link_snapshots: bool = false;
    var debug_compiler_runtime_libs = false;
    var opt_incremental: ?bool = null;
    var install_name: ?[]const u8 = null;
    var hash_style: link.File.Elf.HashStyle = .both;
    var entitlements: ?[]const u8 = null;
    var pagezero_size: ?u64 = null;
    var lib_search_strategy: link.UnresolvedInput.SearchStrategy = .paths_first;
    var lib_preferred_mode: std.builtin.LinkMode = .dynamic;
    var headerpad_size: ?u32 = null;
    var headerpad_max_install_names: bool = false;
    var dead_strip_dylibs: bool = false;
    var force_load_objc: bool = false;
    var discard_local_symbols: bool = false;
    var contains_res_file: bool = false;
    var reference_trace: ?u32 = null;
    var pdb_out_path: ?[]const u8 = null;
    var error_limit: ?Zcu.ErrorInt = null;
    // These are before resolving sysroot.
    var extra_cflags: std.ArrayListUnmanaged([]const u8) = .empty;
    var extra_rcflags: std.ArrayListUnmanaged([]const u8) = .empty;
    var symbol_wrap_set: std.StringArrayHashMapUnmanaged(void) = .empty;
    var rc_includes: Compilation.RcIncludes = .any;
    var manifest_file: ?[]const u8 = null;
    var linker_export_symbol_names: std.ArrayListUnmanaged([]const u8) = .empty;

    // Tracks the position in c_source_files which have already their owner populated.
    var c_source_files_owner_index: usize = 0;
    // Tracks the position in rc_source_files which have already their owner populated.
    var rc_source_files_owner_index: usize = 0;

    // null means replace with the test executable binary
    var test_exec_args: std.ArrayListUnmanaged(?[]const u8) = .empty;

    // These get set by CLI flags and then snapshotted when a `-M` flag is
    // encountered.
    var mod_opts: Package.Module.CreateOptions.Inherited = .{};

    // These get appended to by CLI flags and then slurped when a `-M` flag
    // is encountered.
    var cssan: ClangSearchSanitizer = .{};
    var cc_argv: std.ArrayListUnmanaged([]const u8) = .empty;
    var deps: std.ArrayListUnmanaged(CliModule.Dep) = .empty;

    // Contains every module specified via -M. The dependencies are added
    // after argument parsing is completed. We use a StringArrayHashMap to make
    // error output consistent. "root" is special.
    var create_module: CreateModule = .{
        // Populated just before the call to `createModule`.
        .global_cache_directory = undefined,
        .object_format = null,
        .dynamic_linker = null,
        .modules = .{},
        .opts = .{
            .is_test = arg_mode == .zig_test,
            // Populated while parsing CLI args.
            .output_mode = undefined,
            // Populated in the call to `createModule` for the root module.
            .resolved_target = undefined,
            .have_zcu = false,
            // Populated just before the call to `createModule`.
            .emit_llvm_ir = undefined,
            // Populated just before the call to `createModule`.
            .emit_llvm_bc = undefined,
            // Populated just before the call to `createModule`.
            .emit_bin = undefined,
            // Populated just before the call to `createModule`.
            .any_c_source_files = undefined,
        },
        // Populated in the call to `createModule` for the root module.
        .resolved_options = undefined,

        .cli_link_inputs = .empty,
        .windows_libs = .empty,
        .link_inputs = .empty,

        .wasi_emulated_libs = .{},

        .c_source_files = .{},
        .rc_source_files = .{},

        .llvm_m_args = .{},
        .sysroot = null,
        .lib_directories = .{}, // populated by createModule()
        .lib_dir_args = .{}, // populated from CLI arg parsing
        .libc_installation = null,
        .want_native_include_dirs = false,
        .frameworks = .{},
        .framework_dirs = .{},
        .rpath_list = .{},
        .each_lib_rpath = null,
        .libc_paths_file = try EnvVar.ZIG_LIBC.get(arena),
        .native_system_include_paths = &.{},
    };
    defer create_module.link_inputs.deinit(gpa);

    // before arg parsing, check for the NO_COLOR and CLICOLOR_FORCE environment variables
    // if set, default the color setting to .off or .on, respectively
    // explicit --color arguments will still override this setting.
    // Disable color on WASI per https://github.com/WebAssembly/WASI/issues/162
    var color: Color = if (native_os == .wasi or EnvVar.NO_COLOR.isSet())
        .off
    else if (EnvVar.CLICOLOR_FORCE.isSet())
        .on
    else
        .auto;
    var n_jobs: ?u32 = null;

    switch (arg_mode) {
        .build, .translate_c, .zig_test, .run => {
            switch (arg_mode) {
                .build => |m| {
                    create_module.opts.output_mode = m;
                },
                .translate_c => {
                    emit_bin = .no;
                    create_module.opts.output_mode = .Obj;
                },
                .zig_test, .run => {
                    create_module.opts.output_mode = .Exe;
                },
                else => unreachable,
            }

            soname = .yes_default_value;

            var args_iter = ArgsIterator{
                .args = all_args[2..],
            };

            var file_ext: ?Compilation.FileExt = null;
            args_loop: while (args_iter.next()) |arg| {
                if (mem.startsWith(u8, arg, "@")) {
                    // This is a "compiler response file". We must parse the file and treat its
                    // contents as command line parameters.
                    const resp_file_path = arg[1..];
                    args_iter.resp_file = initArgIteratorResponseFile(arena, resp_file_path) catch |err| {
                        fatal("unable to read response file '{s}': {s}", .{ resp_file_path, @errorName(err) });
                    };
                } else if (mem.startsWith(u8, arg, "-")) {
                    if (mem.eql(u8, arg, "-h") or mem.eql(u8, arg, "--help")) {
                        try io.getStdOut().writeAll(usage_build_generic);
                        return cleanExit();
                    } else if (mem.eql(u8, arg, "--")) {
                        if (arg_mode == .run) {
                            // args_iter.i is 1, referring the next arg after "--" in ["--", ...]
                            // Add +2 to the index so it is relative to all_args
                            runtime_args_start = args_iter.i + 2;
                            break :args_loop;
                        } else {
                            fatal("unexpected end-of-parameter mark: --", .{});
                        }
                    } else if (mem.eql(u8, arg, "--dep")) {
                        var it = mem.splitScalar(u8, args_iter.nextOrFatal(), '=');
                        const key = it.next().?;
                        const value = it.next() orelse key;
                        if (mem.eql(u8, key, "std") and !mem.eql(u8, value, "std")) {
                            fatal("unable to import as '{s}': conflicts with builtin module", .{
                                key,
                            });
                        }
                        for ([_][]const u8{ "root", "builtin" }) |name| {
                            if (mem.eql(u8, key, name)) {
                                fatal("unable to import as '{s}': conflicts with builtin module", .{
                                    key,
                                });
                            }
                        }
                        try deps.append(arena, .{
                            .key = key,
                            .value = value,
                        });
                    } else if (mem.startsWith(u8, arg, "-M")) {
                        var it = mem.splitScalar(u8, arg["-M".len..], '=');
                        const mod_name = it.next().?;
                        const root_src_orig = it.next();
                        try handleModArg(
                            arena,
                            mod_name,
                            root_src_orig,
                            &create_module,
                            &mod_opts,
                            &cc_argv,
                            &target_arch_os_abi,
                            &target_mcpu,
                            &deps,
                            &c_source_files_owner_index,
                            &rc_source_files_owner_index,
                            &cssan,
                        );
                    } else if (mem.eql(u8, arg, "--error-limit")) {
                        const next_arg = args_iter.nextOrFatal();
                        error_limit = std.fmt.parseUnsigned(Zcu.ErrorInt, next_arg, 0) catch |err| {
                            fatal("unable to parse error limit '{s}': {s}", .{ next_arg, @errorName(err) });
                        };
                    } else if (mem.eql(u8, arg, "-cflags")) {
                        extra_cflags.shrinkRetainingCapacity(0);
                        while (true) {
                            const next_arg = args_iter.next() orelse {
                                fatal("expected -- after -cflags", .{});
                            };
                            if (mem.eql(u8, next_arg, "--")) break;
                            try extra_cflags.append(arena, next_arg);
                        }
                    } else if (mem.eql(u8, arg, "-rcincludes")) {
                        rc_includes = parseRcIncludes(args_iter.nextOrFatal());
                    } else if (mem.startsWith(u8, arg, "-rcincludes=")) {
                        rc_includes = parseRcIncludes(arg["-rcincludes=".len..]);
                    } else if (mem.eql(u8, arg, "-rcflags")) {
                        extra_rcflags.shrinkRetainingCapacity(0);
                        while (true) {
                            const next_arg = args_iter.next() orelse {
                                fatal("expected -- after -rcflags", .{});
                            };
                            if (mem.eql(u8, next_arg, "--")) break;
                            try extra_rcflags.append(arena, next_arg);
                        }
                    } else if (mem.startsWith(u8, arg, "-fstructured-cfg")) {
                        mod_opts.structured_cfg = true;
                    } else if (mem.startsWith(u8, arg, "-fno-structured-cfg")) {
                        mod_opts.structured_cfg = false;
                    } else if (mem.eql(u8, arg, "--color")) {
                        const next_arg = args_iter.next() orelse {
                            fatal("expected [auto|on|off] after --color", .{});
                        };
                        color = std.meta.stringToEnum(Color, next_arg) orelse {
                            fatal("expected [auto|on|off] after --color, found '{s}'", .{next_arg});
                        };
                    } else if (mem.startsWith(u8, arg, "-j")) {
                        const str = arg["-j".len..];
                        const num = std.fmt.parseUnsigned(u32, str, 10) catch |err| {
                            fatal("unable to parse jobs count '{s}': {s}", .{
                                str, @errorName(err),
                            });
                        };
                        if (num < 1) {
                            fatal("number of jobs must be at least 1\n", .{});
                        }
                        n_jobs = num;
                    } else if (mem.eql(u8, arg, "--subsystem")) {
                        subsystem = try parseSubSystem(args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "-O")) {
                        mod_opts.optimize_mode = parseOptimizeMode(args_iter.nextOrFatal());
                    } else if (mem.startsWith(u8, arg, "-fentry=")) {
                        entry = .{ .named = arg["-fentry=".len..] };
                    } else if (mem.eql(u8, arg, "--force_undefined")) {
                        try force_undefined_symbols.put(arena, args_iter.nextOrFatal(), {});
                    } else if (mem.eql(u8, arg, "--discard-all")) {
                        discard_local_symbols = true;
                    } else if (mem.eql(u8, arg, "--stack")) {
                        stack_size = parseStackSize(args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "--image-base")) {
                        image_base = parseImageBase(args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "--name")) {
                        provided_name = args_iter.nextOrFatal();
                        if (!mem.eql(u8, provided_name.?, fs.path.basename(provided_name.?)))
                            fatal("invalid package name '{s}': cannot contain folder separators", .{provided_name.?});
                    } else if (mem.eql(u8, arg, "-rpath")) {
                        try create_module.rpath_list.append(arena, args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "--library-directory") or mem.eql(u8, arg, "-L")) {
                        try create_module.lib_dir_args.append(arena, args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "-F")) {
                        try create_module.framework_dirs.append(arena, args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "-framework")) {
                        try create_module.frameworks.put(arena, args_iter.nextOrFatal(), .{});
                    } else if (mem.eql(u8, arg, "-weak_framework")) {
                        try create_module.frameworks.put(arena, args_iter.nextOrFatal(), .{ .weak = true });
                    } else if (mem.eql(u8, arg, "-needed_framework")) {
                        try create_module.frameworks.put(arena, args_iter.nextOrFatal(), .{ .needed = true });
                    } else if (mem.eql(u8, arg, "-install_name")) {
                        install_name = args_iter.nextOrFatal();
                    } else if (mem.startsWith(u8, arg, "--compress-debug-sections=")) {
                        const param = arg["--compress-debug-sections=".len..];
                        linker_compress_debug_sections = std.meta.stringToEnum(link.File.Elf.CompressDebugSections, param) orelse {
                            fatal("expected --compress-debug-sections=[none|zlib|zstd], found '{s}'", .{param});
                        };
                    } else if (mem.eql(u8, arg, "--compress-debug-sections")) {
                        linker_compress_debug_sections = link.File.Elf.CompressDebugSections.zlib;
                    } else if (mem.eql(u8, arg, "-pagezero_size")) {
                        const next_arg = args_iter.nextOrFatal();
                        pagezero_size = std.fmt.parseUnsigned(u64, eatIntPrefix(next_arg, 16), 16) catch |err| {
                            fatal("unable to parse pagezero size'{s}': {s}", .{ next_arg, @errorName(err) });
                        };
                    } else if (mem.eql(u8, arg, "-search_paths_first")) {
                        lib_search_strategy = .paths_first;
                        lib_preferred_mode = .dynamic;
                    } else if (mem.eql(u8, arg, "-search_paths_first_static")) {
                        lib_search_strategy = .paths_first;
                        lib_preferred_mode = .static;
                    } else if (mem.eql(u8, arg, "-search_dylibs_first")) {
                        lib_search_strategy = .mode_first;
                        lib_preferred_mode = .dynamic;
                    } else if (mem.eql(u8, arg, "-search_static_first")) {
                        lib_search_strategy = .mode_first;
                        lib_preferred_mode = .static;
                    } else if (mem.eql(u8, arg, "-search_dylibs_only")) {
                        lib_search_strategy = .no_fallback;
                        lib_preferred_mode = .dynamic;
                    } else if (mem.eql(u8, arg, "-search_static_only")) {
                        lib_search_strategy = .no_fallback;
                        lib_preferred_mode = .static;
                    } else if (mem.eql(u8, arg, "-headerpad")) {
                        const next_arg = args_iter.nextOrFatal();
                        headerpad_size = std.fmt.parseUnsigned(u32, eatIntPrefix(next_arg, 16), 16) catch |err| {
                            fatal("unable to parse headerpad size '{s}': {s}", .{ next_arg, @errorName(err) });
                        };
                    } else if (mem.eql(u8, arg, "-headerpad_max_install_names")) {
                        headerpad_max_install_names = true;
                    } else if (mem.eql(u8, arg, "-dead_strip")) {
                        linker_gc_sections = true;
                    } else if (mem.eql(u8, arg, "-dead_strip_dylibs")) {
                        dead_strip_dylibs = true;
                    } else if (mem.eql(u8, arg, "-ObjC")) {
                        force_load_objc = true;
                    } else if (mem.eql(u8, arg, "-T") or mem.eql(u8, arg, "--script")) {
                        linker_script = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "-version-script") or mem.eql(u8, arg, "--version-script")) {
                        version_script = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--undefined-version")) {
                        linker_allow_undefined_version = true;
                    } else if (mem.eql(u8, arg, "--no-undefined-version")) {
                        linker_allow_undefined_version = false;
                    } else if (mem.eql(u8, arg, "--enable-new-dtags")) {
                        linker_enable_new_dtags = true;
                    } else if (mem.eql(u8, arg, "--disable-new-dtags")) {
                        linker_enable_new_dtags = false;
                    } else if (mem.eql(u8, arg, "--library") or mem.eql(u8, arg, "-l")) {
                        // We don't know whether this library is part of libc
                        // or libc++ until we resolve the target, so we append
                        // to the list for now.
                        try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                            .name = args_iter.nextOrFatal(),
                            .query = .{
                                .needed = false,
                                .weak = false,
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                    } else if (mem.eql(u8, arg, "--needed-library") or
                        mem.eql(u8, arg, "-needed-l") or
                        mem.eql(u8, arg, "-needed_library"))
                    {
                        const next_arg = args_iter.nextOrFatal();
                        try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                            .name = next_arg,
                            .query = .{
                                .needed = true,
                                .weak = false,
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                    } else if (mem.eql(u8, arg, "-weak_library") or mem.eql(u8, arg, "-weak-l")) {
                        try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                            .name = args_iter.nextOrFatal(),
                            .query = .{
                                .needed = false,
                                .weak = true,
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                    } else if (mem.eql(u8, arg, "-D")) {
                        try cc_argv.appendSlice(arena, &.{ arg, args_iter.nextOrFatal() });
                    } else if (mem.eql(u8, arg, "-I")) {
                        try cssan.addIncludePath(arena, &cc_argv, .I, arg, args_iter.nextOrFatal(), false);
                    } else if (mem.eql(u8, arg, "-isystem")) {
                        try cssan.addIncludePath(arena, &cc_argv, .isystem, arg, args_iter.nextOrFatal(), false);
                    } else if (mem.eql(u8, arg, "-iwithsysroot")) {
                        try cssan.addIncludePath(arena, &cc_argv, .iwithsysroot, arg, args_iter.nextOrFatal(), false);
                    } else if (mem.eql(u8, arg, "-idirafter")) {
                        try cssan.addIncludePath(arena, &cc_argv, .idirafter, arg, args_iter.nextOrFatal(), false);
                    } else if (mem.eql(u8, arg, "-iframework")) {
                        const path = args_iter.nextOrFatal();
                        try cssan.addIncludePath(arena, &cc_argv, .iframework, arg, path, false);
                        try create_module.framework_dirs.append(arena, path); // Forward to the backend as -F
                    } else if (mem.eql(u8, arg, "-iframeworkwithsysroot")) {
                        const path = args_iter.nextOrFatal();
                        try cssan.addIncludePath(arena, &cc_argv, .iframeworkwithsysroot, arg, path, false);
                        try create_module.framework_dirs.append(arena, path); // Forward to the backend as -F
                    } else if (mem.eql(u8, arg, "--version")) {
                        const next_arg = args_iter.nextOrFatal();
                        version = std.SemanticVersion.parse(next_arg) catch |err| {
                            fatal("unable to parse --version '{s}': {s}", .{ next_arg, @errorName(err) });
                        };
                        have_version = true;
                    } else if (mem.eql(u8, arg, "-target")) {
                        target_arch_os_abi = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "-mcpu")) {
                        target_mcpu = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "-mcmodel")) {
                        mod_opts.code_model = parseCodeModel(args_iter.nextOrFatal());
                    } else if (mem.startsWith(u8, arg, "-mcmodel=")) {
                        mod_opts.code_model = parseCodeModel(arg["-mcmodel=".len..]);
                    } else if (mem.startsWith(u8, arg, "-ofmt=")) {
                        create_module.object_format = arg["-ofmt=".len..];
                    } else if (mem.startsWith(u8, arg, "-mcpu=")) {
                        target_mcpu = arg["-mcpu=".len..];
                    } else if (mem.startsWith(u8, arg, "-O")) {
                        mod_opts.optimize_mode = parseOptimizeMode(arg["-O".len..]);
                    } else if (mem.eql(u8, arg, "--dynamic-linker")) {
                        create_module.dynamic_linker = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--sysroot")) {
                        const next_arg = args_iter.nextOrFatal();
                        create_module.sysroot = next_arg;
                        try cc_argv.appendSlice(arena, &.{ "-isysroot", next_arg });
                    } else if (mem.eql(u8, arg, "--libc")) {
                        create_module.libc_paths_file = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--test-filter")) {
                        try test_filters.append(arena, args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "--test-name-prefix")) {
                        test_name_prefix = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--test-runner")) {
                        test_runner_path = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--test-cmd")) {
                        try test_exec_args.append(arena, args_iter.nextOrFatal());
                    } else if (mem.eql(u8, arg, "--cache-dir")) {
                        override_local_cache_dir = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--global-cache-dir")) {
                        override_global_cache_dir = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--zig-lib-dir")) {
                        override_lib_dir = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "--debug-log")) {
                        if (!build_options.enable_logging) {
                            warn("Zig was compiled without logging enabled (-Dlog). --debug-log has no effect.", .{});
                            _ = args_iter.nextOrFatal();
                        } else {
                            try log_scopes.append(arena, args_iter.nextOrFatal());
                        }
                    } else if (mem.eql(u8, arg, "--listen")) {
                        const next_arg = args_iter.nextOrFatal();
                        if (mem.eql(u8, next_arg, "-")) {
                            dev.check(.stdio_listen);
                            listen = .stdio;
                        } else {
                            dev.check(.network_listen);
                            // example: --listen 127.0.0.1:9000
                            var it = std.mem.splitScalar(u8, next_arg, ':');
                            const host = it.next().?;
                            const port_text = it.next() orelse "14735";
                            const port = std.fmt.parseInt(u16, port_text, 10) catch |err|
                                fatal("invalid port number: '{s}': {s}", .{ port_text, @errorName(err) });
                            listen = .{ .ip4 = std.net.Ip4Address.parse(host, port) catch |err|
                                fatal("invalid host: '{s}': {s}", .{ host, @errorName(err) }) };
                        }
                    } else if (mem.eql(u8, arg, "--listen=-")) {
                        dev.check(.stdio_listen);
                        listen = .stdio;
                    } else if (mem.eql(u8, arg, "--debug-link-snapshot")) {
                        if (!build_options.enable_link_snapshots) {
                            warn("Zig was compiled without linker snapshots enabled (-Dlink-snapshot). --debug-link-snapshot has no effect.", .{});
                        } else {
                            enable_link_snapshots = true;
                        }
                    } else if (mem.eql(u8, arg, "--debug-rt")) {
                        debug_compiler_runtime_libs = true;
                    } else if (mem.eql(u8, arg, "-fincremental")) {
                        dev.check(.incremental);
                        opt_incremental = true;
                    } else if (mem.eql(u8, arg, "-fno-incremental")) {
                        opt_incremental = false;
                    } else if (mem.eql(u8, arg, "--entitlements")) {
                        entitlements = args_iter.nextOrFatal();
                    } else if (mem.eql(u8, arg, "-fcompiler-rt")) {
                        want_compiler_rt = true;
                    } else if (mem.eql(u8, arg, "-fno-compiler-rt")) {
                        want_compiler_rt = false;
                    } else if (mem.eql(u8, arg, "-fubsan-rt")) {
                        want_ubsan_rt = true;
                    } else if (mem.eql(u8, arg, "-fno-ubsan-rt")) {
                        want_ubsan_rt = false;
                    } else if (mem.eql(u8, arg, "-feach-lib-rpath")) {
                        create_module.each_lib_rpath = true;
                    } else if (mem.eql(u8, arg, "-fno-each-lib-rpath")) {
                        create_module.each_lib_rpath = false;
                    } else if (mem.eql(u8, arg, "--test-cmd-bin")) {
                        try test_exec_args.append(arena, null);
                    } else if (mem.eql(u8, arg, "--test-no-exec")) {
                        test_no_exec = true;
                    } else if (mem.eql(u8, arg, "-ftime-report")) {
                        time_report = true;
                    } else if (mem.eql(u8, arg, "-fstack-report")) {
                        stack_report = true;
                    } else if (mem.eql(u8, arg, "-fPIC")) {
                        mod_opts.pic = true;
                    } else if (mem.eql(u8, arg, "-fno-PIC")) {
                        mod_opts.pic = false;
                    } else if (mem.eql(u8, arg, "-fPIE")) {
                        create_module.opts.pie = true;
                    } else if (mem.eql(u8, arg, "-fno-PIE")) {
                        create_module.opts.pie = false;
                    } else if (mem.eql(u8, arg, "-flto")) {
                        create_module.opts.lto = .full;
                    } else if (mem.startsWith(u8, arg, "-flto=")) {
                        const mode = arg["-flto=".len..];
                        if (mem.eql(u8, mode, "full")) {
                            create_module.opts.lto = .full;
                        } else if (mem.eql(u8, mode, "thin")) {
                            create_module.opts.lto = .thin;
                        } else {
                            fatal("Invalid -flto mode: '{s}'. Must be 'full'or 'thin'.", .{mode});
                        }
                    } else if (mem.eql(u8, arg, "-fno-lto")) {
                        create_module.opts.lto = .none;
                    } else if (mem.eql(u8, arg, "-funwind-tables")) {
                        mod_opts.unwind_tables = .sync;
                    } else if (mem.eql(u8, arg, "-fasync-unwind-tables")) {
                        mod_opts.unwind_tables = .@"async";
                    } else if (mem.eql(u8, arg, "-fno-unwind-tables")) {
                        mod_opts.unwind_tables = .none;
                    } else if (mem.eql(u8, arg, "-fstack-check")) {
                        mod_opts.stack_check = true;
                    } else if (mem.eql(u8, arg, "-fno-stack-check")) {
                        mod_opts.stack_check = false;
                    } else if (mem.eql(u8, arg, "-fstack-protector")) {
                        mod_opts.stack_protector = Compilation.default_stack_protector_buffer_size;
                    } else if (mem.eql(u8, arg, "-fno-stack-protector")) {
                        mod_opts.stack_protector = 0;
                    } else if (mem.eql(u8, arg, "-mred-zone")) {
                        mod_opts.red_zone = true;
                    } else if (mem.eql(u8, arg, "-mno-red-zone")) {
                        mod_opts.red_zone = false;
                    } else if (mem.eql(u8, arg, "-fomit-frame-pointer")) {
                        mod_opts.omit_frame_pointer = true;
                    } else if (mem.eql(u8, arg, "-fno-omit-frame-pointer")) {
                        mod_opts.omit_frame_pointer = false;
                    } else if (mem.eql(u8, arg, "-fsanitize-c")) {
                        mod_opts.sanitize_c = true;
                    } else if (mem.eql(u8, arg, "-fno-sanitize-c")) {
                        mod_opts.sanitize_c = false;
                    } else if (mem.eql(u8, arg, "-fvalgrind")) {
                        mod_opts.valgrind = true;
                    } else if (mem.eql(u8, arg, "-fno-valgrind")) {
                        mod_opts.valgrind = false;
                    } else if (mem.eql(u8, arg, "-fsanitize-thread")) {
                        mod_opts.sanitize_thread = true;
                    } else if (mem.eql(u8, arg, "-fno-sanitize-thread")) {
                        mod_opts.sanitize_thread = false;
                    } else if (mem.eql(u8, arg, "-ffuzz")) {
                        mod_opts.fuzz = true;
                    } else if (mem.eql(u8, arg, "-fno-fuzz")) {
                        mod_opts.fuzz = false;
                    } else if (mem.eql(u8, arg, "-fllvm")) {
                        create_module.opts.use_llvm = true;
                    } else if (mem.eql(u8, arg, "-fno-llvm")) {
                        create_module.opts.use_llvm = false;
                    } else if (mem.eql(u8, arg, "-flibllvm")) {
                        create_module.opts.use_lib_llvm = true;
                    } else if (mem.eql(u8, arg, "-fno-libllvm")) {
                        create_module.opts.use_lib_llvm = false;
                    } else if (mem.eql(u8, arg, "-flld")) {
                        create_module.opts.use_lld = true;
                    } else if (mem.eql(u8, arg, "-fno-lld")) {
                        create_module.opts.use_lld = false;
                    } else if (mem.eql(u8, arg, "-fclang")) {
                        create_module.opts.use_clang = true;
                    } else if (mem.eql(u8, arg, "-fno-clang")) {
                        create_module.opts.use_clang = false;
                    } else if (mem.eql(u8, arg, "-fsanitize-coverage-trace-pc-guard")) {
                        create_module.opts.san_cov_trace_pc_guard = true;
                    } else if (mem.eql(u8, arg, "-fno-sanitize-coverage-trace-pc-guard")) {
                        create_module.opts.san_cov_trace_pc_guard = false;
                    } else if (mem.eql(u8, arg, "-freference-trace")) {
                        reference_trace = 256;
                    } else if (mem.startsWith(u8, arg, "-freference-trace=")) {
                        const num = arg["-freference-trace=".len..];
                        reference_trace = std.fmt.parseUnsigned(u32, num, 10) catch |err| {
                            fatal("unable to parse reference_trace count '{s}': {s}", .{ num, @errorName(err) });
                        };
                    } else if (mem.eql(u8, arg, "-fno-reference-trace")) {
                        reference_trace = null;
                    } else if (mem.eql(u8, arg, "-ferror-tracing")) {
                        mod_opts.error_tracing = true;
                    } else if (mem.eql(u8, arg, "-fno-error-tracing")) {
                        mod_opts.error_tracing = false;
                    } else if (mem.eql(u8, arg, "-rdynamic")) {
                        create_module.opts.rdynamic = true;
                    } else if (mem.eql(u8, arg, "-fsoname")) {
                        soname = .yes_default_value;
                    } else if (mem.startsWith(u8, arg, "-fsoname=")) {
                        soname = .{ .yes = arg["-fsoname=".len..] };
                    } else if (mem.eql(u8, arg, "-fno-soname")) {
                        soname = .no;
                    } else if (mem.eql(u8, arg, "-femit-bin")) {
                        emit_bin = .yes_default_path;
                    } else if (mem.startsWith(u8, arg, "-femit-bin=")) {
                        emit_bin = .{ .yes = arg["-femit-bin=".len..] };
                    } else if (mem.eql(u8, arg, "-fno-emit-bin")) {
                        emit_bin = .no;
                    } else if (mem.eql(u8, arg, "-femit-h")) {
                        emit_h = .yes_default_path;
                    } else if (mem.startsWith(u8, arg, "-femit-h=")) {
                        emit_h = .{ .yes = arg["-femit-h=".len..] };
                    } else if (mem.eql(u8, arg, "-fno-emit-h")) {
                        emit_h = .no;
                    } else if (mem.eql(u8, arg, "-femit-asm")) {
                        emit_asm = .yes_default_path;
                    } else if (mem.startsWith(u8, arg, "-femit-asm=")) {
                        emit_asm = .{ .yes = arg["-femit-asm=".len..] };
                    } else if (mem.eql(u8, arg, "-fno-emit-asm")) {
                        emit_asm = .no;
                    } else if (mem.eql(u8, arg, "-femit-llvm-ir")) {
                        emit_llvm_ir = .yes_default_path;
                    } else if (mem.startsWith(u8, arg, "-femit-llvm-ir=")) {
                        emit_llvm_ir = .{ .yes = arg["-femit-llvm-ir=".len..] };
                    } else if (mem.eql(u8, arg, "-fno-emit-llvm-ir")) {
                        emit_llvm_ir = .no;
                    } else if (mem.eql(u8, arg, "-femit-llvm-bc")) {
                        emit_llvm_bc = .yes_default_path;
                    } else if (mem.startsWith(u8, arg, "-femit-llvm-bc=")) {
                        emit_llvm_bc = .{ .yes = arg["-femit-llvm-bc=".len..] };
                    } else if (mem.eql(u8, arg, "-fno-emit-llvm-bc")) {
                        emit_llvm_bc = .no;
                    } else if (mem.eql(u8, arg, "-femit-docs")) {
                        emit_docs = .yes_default_path;
                    } else if (mem.startsWith(u8, arg, "-femit-docs=")) {
                        emit_docs = .{ .yes = arg["-femit-docs=".len..] };
                    } else if (mem.eql(u8, arg, "-fno-emit-docs")) {
                        emit_docs = .no;
                    } else if (mem.eql(u8, arg, "-femit-implib")) {
                        emit_implib = .yes_default_path;
                        emit_implib_arg_provided = true;
                    } else if (mem.startsWith(u8, arg, "-femit-implib=")) {
                        emit_implib = .{ .yes = arg["-femit-implib=".len..] };
                        emit_implib_arg_provided = true;
                    } else if (mem.eql(u8, arg, "-fno-emit-implib")) {
                        emit_implib = .no;
                        emit_implib_arg_provided = true;
                    } else if (mem.eql(u8, arg, "-dynamic")) {
                        create_module.opts.link_mode = .dynamic;
                        lib_preferred_mode = .dynamic;
                        lib_search_strategy = .mode_first;
                    } else if (mem.eql(u8, arg, "-static")) {
                        create_module.opts.link_mode = .static;
                        lib_preferred_mode = .static;
                        lib_search_strategy = .no_fallback;
                    } else if (mem.eql(u8, arg, "-fdll-export-fns")) {
                        create_module.opts.dll_export_fns = true;
                    } else if (mem.eql(u8, arg, "-fno-dll-export-fns")) {
                        create_module.opts.dll_export_fns = false;
                    } else if (mem.eql(u8, arg, "--show-builtin")) {
                        show_builtin = true;
                        emit_bin = .no;
                    } else if (mem.eql(u8, arg, "-fstrip")) {
                        mod_opts.strip = true;
                    } else if (mem.eql(u8, arg, "-fno-strip")) {
                        mod_opts.strip = false;
                    } else if (mem.eql(u8, arg, "-gdwarf32")) {
                        create_module.opts.debug_format = .{ .dwarf = .@"32" };
                    } else if (mem.eql(u8, arg, "-gdwarf64")) {
                        create_module.opts.debug_format = .{ .dwarf = .@"64" };
                    } else if (mem.eql(u8, arg, "-fformatted-panics")) {
                        // Remove this after 0.15.0 is tagged.
                        warn("-fformatted-panics is deprecated and does nothing", .{});
                    } else if (mem.eql(u8, arg, "-fno-formatted-panics")) {
                        // Remove this after 0.15.0 is tagged.
                        warn("-fno-formatted-panics is deprecated and does nothing", .{});
                    } else if (mem.eql(u8, arg, "-fsingle-threaded")) {
                        mod_opts.single_threaded = true;
                    } else if (mem.eql(u8, arg, "-fno-single-threaded")) {
                        mod_opts.single_threaded = false;
                    } else if (mem.eql(u8, arg, "-ffunction-sections")) {
                        function_sections = true;
                    } else if (mem.eql(u8, arg, "-fno-function-sections")) {
                        function_sections = false;
                    } else if (mem.eql(u8, arg, "-fdata-sections")) {
                        data_sections = true;
                    } else if (mem.eql(u8, arg, "-fno-data-sections")) {
                        data_sections = false;
                    } else if (mem.eql(u8, arg, "-fbuiltin")) {
                        mod_opts.no_builtin = false;
                    } else if (mem.eql(u8, arg, "-fno-builtin")) {
                        mod_opts.no_builtin = true;
                    } else if (mem.startsWith(u8, arg, "-fopt-bisect-limit=")) {
                        const next_arg = arg["-fopt-bisect-limit=".len..];
                        llvm_opt_bisect_limit = std.fmt.parseInt(c_int, next_arg, 0) catch |err|
                            fatal("unable to parse '{s}': {s}", .{ arg, @errorName(err) });
                    } else if (mem.eql(u8, arg, "--eh-frame-hdr")) {
                        link_eh_frame_hdr = true;
                    } else if (mem.eql(u8, arg, "--no-eh-frame-hdr")) {
                        link_eh_frame_hdr = false;
                    } else if (mem.eql(u8, arg, "--dynamicbase")) {
                        linker_dynamicbase = true;
                    } else if (mem.eql(u8, arg, "--no-dynamicbase")) {
                        linker_dynamicbase = false;
                    } else if (mem.eql(u8, arg, "--emit-relocs")) {
                        link_emit_relocs = true;
                    } else if (mem.eql(u8, arg, "-fallow-shlib-undefined")) {
                        linker_allow_shlib_undefined = true;
                    } else if (mem.eql(u8, arg, "-fno-allow-shlib-undefined")) {
                        linker_allow_shlib_undefined = false;
                    } else if (mem.eql(u8, arg, "-fallow-so-scripts")) {
                        allow_so_scripts = true;
                    } else if (mem.eql(u8, arg, "-fno-allow-so-scripts")) {
                        allow_so_scripts = false;
                    } else if (mem.eql(u8, arg, "-z")) {
                        const z_arg = args_iter.nextOrFatal();
                        if (mem.eql(u8, z_arg, "nodelete")) {
                            linker_z_nodelete = true;
                        } else if (mem.eql(u8, z_arg, "notext")) {
                            linker_z_notext = true;
                        } else if (mem.eql(u8, z_arg, "defs")) {
                            linker_z_defs = true;
                        } else if (mem.eql(u8, z_arg, "undefs")) {
                            linker_z_defs = false;
                        } else if (mem.eql(u8, z_arg, "origin")) {
                            linker_z_origin = true;
                        } else if (mem.eql(u8, z_arg, "nocopyreloc")) {
                            linker_z_nocopyreloc = true;
                        } else if (mem.eql(u8, z_arg, "now")) {
                            linker_z_now = true;
                        } else if (mem.eql(u8, z_arg, "lazy")) {
                            linker_z_now = false;
                        } else if (mem.eql(u8, z_arg, "relro")) {
                            linker_z_relro = true;
                        } else if (mem.eql(u8, z_arg, "norelro")) {
                            linker_z_relro = false;
                        } else if (mem.startsWith(u8, z_arg, "common-page-size=")) {
                            linker_z_common_page_size = parseIntSuffix(z_arg, "common-page-size=".len);
                        } else if (mem.startsWith(u8, z_arg, "max-page-size=")) {
                            linker_z_max_page_size = parseIntSuffix(z_arg, "max-page-size=".len);
                        } else {
                            fatal("unsupported linker extension flag: -z {s}", .{z_arg});
                        }
                    } else if (mem.eql(u8, arg, "--import-memory")) {
                        create_module.opts.import_memory = true;
                    } else if (mem.eql(u8, arg, "-fentry")) {
                        switch (entry) {
                            .default, .disabled => entry = .enabled,
                            .enabled, .named => {},
                        }
                    } else if (mem.eql(u8, arg, "-fno-entry")) {
                        entry = .disabled;
                    } else if (mem.eql(u8, arg, "--export-memory")) {
                        create_module.opts.export_memory = true;
                    } else if (mem.eql(u8, arg, "--import-symbols")) {
                        linker_import_symbols = true;
                    } else if (mem.eql(u8, arg, "--import-table")) {
                        linker_import_table = true;
                    } else if (mem.eql(u8, arg, "--export-table")) {
                        linker_export_table = true;
                    } else if (mem.startsWith(u8, arg, "--initial-memory=")) {
                        linker_initial_memory = parseIntSuffix(arg, "--initial-memory=".len);
                    } else if (mem.startsWith(u8, arg, "--max-memory=")) {
                        linker_max_memory = parseIntSuffix(arg, "--max-memory=".len);
                    } else if (mem.eql(u8, arg, "--shared-memory")) {
                        create_module.opts.shared_memory = true;
                    } else if (mem.startsWith(u8, arg, "--global-base=")) {
                        linker_global_base = parseIntSuffix(arg, "--global-base=".len);
                    } else if (mem.startsWith(u8, arg, "--export=")) {
                        try linker_export_symbol_names.append(arena, arg["--export=".len..]);
                    } else if (mem.eql(u8, arg, "-Bsymbolic")) {
                        linker_bind_global_refs_locally = true;
                    } else if (mem.eql(u8, arg, "--gc-sections")) {
                        linker_gc_sections = true;
                    } else if (mem.eql(u8, arg, "--no-gc-sections")) {
                        linker_gc_sections = false;
                    } else if (mem.eql(u8, arg, "--build-id")) {
                        build_id = .fast;
                    } else if (mem.startsWith(u8, arg, "--build-id=")) {
                        const style = arg["--build-id=".len..];
                        build_id = std.zig.BuildId.parse(style) catch |err| {
                            fatal("unable to parse --build-id style '{s}': {s}", .{
                                style, @errorName(err),
                            });
                        };
                    } else if (mem.eql(u8, arg, "--debug-compile-errors")) {
                        if (build_options.enable_debug_extensions) {
                            debug_compile_errors = true;
                        } else {
                            warn("Zig was compiled without debug extensions. --debug-compile-errors has no effect.", .{});
                        }
                    } else if (mem.eql(u8, arg, "--verbose-link")) {
                        verbose_link = true;
                    } else if (mem.eql(u8, arg, "--verbose-cc")) {
                        verbose_cc = true;
                    } else if (mem.eql(u8, arg, "--verbose-air")) {
                        verbose_air = true;
                    } else if (mem.eql(u8, arg, "--verbose-intern-pool")) {
                        verbose_intern_pool = true;
                    } else if (mem.eql(u8, arg, "--verbose-generic-instances")) {
                        verbose_generic_instances = true;
                    } else if (mem.eql(u8, arg, "--verbose-llvm-ir")) {
                        verbose_llvm_ir = "-";
                    } else if (mem.startsWith(u8, arg, "--verbose-llvm-ir=")) {
                        verbose_llvm_ir = arg["--verbose-llvm-ir=".len..];
                    } else if (mem.startsWith(u8, arg, "--verbose-llvm-bc=")) {
                        verbose_llvm_bc = arg["--verbose-llvm-bc=".len..];
                    } else if (mem.eql(u8, arg, "--verbose-cimport")) {
                        verbose_cimport = true;
                    } else if (mem.eql(u8, arg, "--verbose-llvm-cpu-features")) {
                        verbose_llvm_cpu_features = true;
                    } else if (mem.startsWith(u8, arg, "-T")) {
                        linker_script = arg[2..];
                    } else if (mem.startsWith(u8, arg, "-L")) {
                        try create_module.lib_dir_args.append(arena, arg[2..]);
                    } else if (mem.startsWith(u8, arg, "-F")) {
                        try create_module.framework_dirs.append(arena, arg[2..]);
                    } else if (mem.startsWith(u8, arg, "-l")) {
                        // We don't know whether this library is part of libc
                        // or libc++ until we resolve the target, so we append
                        // to the list for now.
                        try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                            .name = arg["-l".len..],
                            .query = .{
                                .needed = false,
                                .weak = false,
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                    } else if (mem.startsWith(u8, arg, "-needed-l")) {
                        try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                            .name = arg["-needed-l".len..],
                            .query = .{
                                .needed = true,
                                .weak = false,
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                    } else if (mem.startsWith(u8, arg, "-weak-l")) {
                        try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                            .name = arg["-weak-l".len..],
                            .query = .{
                                .needed = false,
                                .weak = true,
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                    } else if (mem.startsWith(u8, arg, "-D")) {
                        try cc_argv.append(arena, arg);
                    } else if (mem.startsWith(u8, arg, "-I")) {
                        try cssan.addIncludePath(arena, &cc_argv, .I, arg, arg[2..], true);
                    } else if (mem.startsWith(u8, arg, "-x")) {
                        const lang = if (arg.len == "-x".len)
                            args_iter.nextOrFatal()
                        else
                            arg["-x".len..];
                        if (mem.eql(u8, lang, "none")) {
                            file_ext = null;
                        } else if (Compilation.LangToExt.get(lang)) |got_ext| {
                            file_ext = got_ext;
                        } else {
                            fatal("language not recognized: '{s}'", .{lang});
                        }
                    } else if (mem.startsWith(u8, arg, "-mexec-model=")) {
                        create_module.opts.wasi_exec_model = parseWasiExecModel(arg["-mexec-model=".len..]);
                    } else if (mem.eql(u8, arg, "-municode")) {
                        mingw_unicode_entry_point = true;
                    } else {
                        fatal("unrecognized parameter: '{s}'", .{arg});
                    }
                } else switch (file_ext orelse Compilation.classifyFileExt(arg)) {
                    .shared_library, .object, .static_library => {
                        try create_module.cli_link_inputs.append(arena, .{ .path_query = .{
                            .path = Path.initCwd(arg),
                            .query = .{
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                        // We do not set `any_dyn_libs` yet because a .so file
                        // may actually resolve to a GNU ld script which ends
                        // up being a static library.
                    },
                    .res => {
                        try create_module.cli_link_inputs.append(arena, .{ .path_query = .{
                            .path = Path.initCwd(arg),
                            .query = .{
                                .preferred_mode = lib_preferred_mode,
                                .search_strategy = lib_search_strategy,
                                .allow_so_scripts = allow_so_scripts,
                            },
                        } });
                        contains_res_file = true;
                    },
                    .manifest => {
                        if (manifest_file) |other| {
                            fatal("only one manifest file can be specified, found '{s}' after '{s}'", .{ arg, other });
                        } else manifest_file = arg;
                    },
                    .assembly, .assembly_with_cpp, .c, .cpp, .h, .hpp, .hm, .hmm, .ll, .bc, .m, .mm => {
                        try create_module.c_source_files.append(arena, .{
                            // Populated after module creation.
                            .owner = undefined,
                            .src_path = arg,
                            .extra_flags = try arena.dupe([]const u8, extra_cflags.items),
                            // duped when parsing the args.
                            .ext = file_ext,
                        });
                    },
                    .rc => {
                        try create_module.rc_source_files.append(arena, .{
                            // Populated after module creation.
                            .owner = undefined,
                            .src_path = arg,
                            .extra_flags = try arena.dupe([]const u8, extra_rcflags.items),
                        });
                    },
                    .zig => {
                        if (root_src_file) |other| {
                            fatal("found another zig file '{s}' after root source file '{s}'", .{ arg, other });
                        } else root_src_file = arg;
                    },
                    .def, .unknown => {
                        if (std.ascii.eqlIgnoreCase(".xml", std.fs.path.extension(arg))) {
                            warn("embedded manifest files must have the extension '.manifest'", .{});
                        }
                        fatal("unrecognized file extension of parameter '{s}'", .{arg});
                    },
                }
            }
        },
        .cc, .cpp => {
            dev.check(.cc_command);

            emit_h = .no;
            soname = .no;
            create_module.opts.ensure_libc_on_non_freestanding = true;
            create_module.opts.ensure_libcpp_on_non_freestanding = arg_mode == .cpp;
            create_module.want_native_include_dirs = true;
            // Clang's driver enables this switch unconditionally.
            // Disabling the emission of .eh_frame_hdr can unexpectedly break
            // some functionality that depend on it, such as C++ exceptions and
            // DWARF-based stack traces.
            link_eh_frame_hdr = true;
            allow_so_scripts = true;

            const COutMode = enum {
                link,
                object,
                assembly,
                preprocessor,
            };
            var c_out_mode: ?COutMode = null;
            var out_path: ?[]const u8 = null;
            var is_shared_lib = false;
            var preprocessor_args = std.ArrayList([]const u8).init(arena);
            var linker_args = std.ArrayList([]const u8).init(arena);
            var it = ClangArgIterator.init(arena, all_args);
            var emit_llvm = false;
            var needed = false;
            var must_link = false;
            var file_ext: ?Compilation.FileExt = null;
            while (it.has_next) {
                it.next() catch |err| {
                    fatal("unable to parse command line parameters: {s}", .{@errorName(err)});
                };
                switch (it.zig_equivalent) {
                    .target => target_arch_os_abi = it.only_arg, // example: -target riscv64-linux-unknown
                    .o => {
                        // We handle -o /dev/null equivalent to -fno-emit-bin because
                        // otherwise our atomic rename into place will fail. This also
                        // makes Zig do less work, avoiding pointless file system operations.
                        if (mem.eql(u8, it.only_arg, "/dev/null")) {
                            emit_bin = .no;
                        } else {
                            out_path = it.only_arg;
                        }
                    },
                    .c, .r => c_out_mode = .object, // -c or -r
                    .asm_only => c_out_mode = .assembly, // -S
                    .preprocess_only => c_out_mode = .preprocessor, // -E
                    .emit_llvm => emit_llvm = true,
                    .x => {
                        const lang = mem.sliceTo(it.only_arg, 0);
                        if (mem.eql(u8, lang, "none")) {
                            file_ext = null;
                        } else if (Compilation.LangToExt.get(lang)) |got_ext| {
                            file_ext = got_ext;
                        } else {
                            fatal("language not recognized: '{s}'", .{lang});
                        }
                    },
                    .other => {
                        try cc_argv.appendSlice(arena, it.other_args);
                    },
                    .positional => switch (file_ext orelse Compilation.classifyFileExt(mem.sliceTo(it.only_arg, 0))) {
                        .assembly, .assembly_with_cpp, .c, .cpp, .ll, .bc, .h, .hpp, .hm, .hmm, .m, .mm => {
                            try create_module.c_source_files.append(arena, .{
                                // Populated after module creation.
                                .owner = undefined,
                                .src_path = it.only_arg,
                                .ext = file_ext, // duped while parsing the args.
                            });
                        },
                        .unknown, .object, .static_library, .shared_library => {
                            try create_module.cli_link_inputs.append(arena, .{ .path_query = .{
                                .path = Path.initCwd(it.only_arg),
                                .query = .{
                                    .must_link = must_link,
                                    .needed = needed,
                                    .preferred_mode = lib_preferred_mode,
                                    .search_strategy = lib_search_strategy,
                                    .allow_so_scripts = allow_so_scripts,
                                },
                            } });
                            // We do not set `any_dyn_libs` yet because a .so file
                            // may actually resolve to a GNU ld script which ends
                            // up being a static library.
                        },
                        .res => {
                            try create_module.cli_link_inputs.append(arena, .{ .path_query = .{
                                .path = Path.initCwd(it.only_arg),
                                .query = .{
                                    .must_link = must_link,
                                    .needed = needed,
                                    .preferred_mode = lib_preferred_mode,
                                    .search_strategy = lib_search_strategy,
                                    .allow_so_scripts = allow_so_scripts,
                                },
                            } });
                            contains_res_file = true;
                        },
                        .manifest => {
                            if (manifest_file) |other| {
                                fatal("only one manifest file can be specified, found '{s}' after previously specified manifest '{s}'", .{ it.only_arg, other });
                            } else manifest_file = it.only_arg;
                        },
                        .def => {
                            linker_module_definition_file = it.only_arg;
                        },
                        .rc => {
                            try create_module.rc_source_files.append(arena, .{
                                // Populated after module creation.
                                .owner = undefined,
                                .src_path = it.only_arg,
                            });
                        },
                        .zig => {
                            if (root_src_file) |other| {
                                fatal("found another zig file '{s}' after root source file '{s}'", .{ it.only_arg, other });
                            } else root_src_file = it.only_arg;
                        },
                    },
                    .l => {
                        // -l
                        // We don't know whether this library is part of libc or libc++ until
                        // we resolve the target, so we simply append to the list for now.
                        if (mem.startsWith(u8, it.only_arg, ":")) {
                            // -l :path/to/filename is used when callers need
                            // more control over what's in the resulting
                            // binary: no extra rpaths and DSO filename exactly
                            // as provided. CGo compilation depends on this.
                            try create_module.cli_link_inputs.append(arena, .{ .dso_exact = .{
                                .name = it.only_arg,
                            } });
                        } else {
                            try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                                .name = it.only_arg,
                                .query = .{
                                    .must_link = must_link,
                                    .needed = needed,
                                    .weak = false,
                                    .preferred_mode = lib_preferred_mode,
                                    .search_strategy = lib_search_strategy,
                                    .allow_so_scripts = allow_so_scripts,
                                },
                            } });
                        }
                    },
                    .ignore => {},
                    .driver_punt => {
                        // Never mind what we're doing, just pass the args directly. For example --help.
                        return process.exit(try clangMain(arena, all_args));
                    },
                    .pic => mod_opts.pic = true,
                    .no_pic => mod_opts.pic = false,
                    .pie => create_module.opts.pie = true,
                    .no_pie => create_module.opts.pie = false,
                    .lto => {
                        if (mem.eql(u8, it.only_arg, "flto") or
                            mem.eql(u8, it.only_arg, "auto") or
                            mem.eql(u8, it.only_arg, "full") or
                            mem.eql(u8, it.only_arg, "jobserver"))
                        {
                            create_module.opts.lto = .full;
                        } else if (mem.eql(u8, it.only_arg, "thin")) {
                            create_module.opts.lto = .thin;
                        } else {
                            fatal("Invalid -flto mode: '{s}'. Must be 'auto', 'full', 'thin', or 'jobserver'.", .{it.only_arg});
                        }
                    },
                    .no_lto => create_module.opts.lto = .none,
                    .red_zone => mod_opts.red_zone = true,
                    .no_red_zone => mod_opts.red_zone = false,
                    .omit_frame_pointer => mod_opts.omit_frame_pointer = true,
                    .no_omit_frame_pointer => mod_opts.omit_frame_pointer = false,
                    .function_sections => function_sections = true,
                    .no_function_sections => function_sections = false,
                    .data_sections => data_sections = true,
                    .no_data_sections => data_sections = false,
                    .builtin => mod_opts.no_builtin = false,
                    .no_builtin => mod_opts.no_builtin = true,
                    .color_diagnostics => color = .on,
                    .no_color_diagnostics => color = .off,
                    .stack_check => mod_opts.stack_check = true,
                    .no_stack_check => mod_opts.stack_check = false,
                    .stack_protector => {
                        if (mod_opts.stack_protector == null) {
                            mod_opts.stack_protector = Compilation.default_stack_protector_buffer_size;
                        }
                    },
                    .no_stack_protector => mod_opts.stack_protector = 0,
                    // The way these unwind table options are processed in GCC and Clang is crazy
                    // convoluted, and we also don't know the target triple here, so this is all
                    // best-effort.
                    .unwind_tables => if (mod_opts.unwind_tables) |uwt| switch (uwt) {
                        .none => {
                            mod_opts.unwind_tables = .sync;
                        },
                        .sync, .@"async" => {},
                    } else {
                        mod_opts.unwind_tables = .sync;
                    },
                    .no_unwind_tables => mod_opts.unwind_tables = .none,
                    .asynchronous_unwind_tables => mod_opts.unwind_tables = .@"async",
                    .no_asynchronous_unwind_tables => if (mod_opts.unwind_tables) |uwt| switch (uwt) {
                        .none, .sync => {},
                        .@"async" => {
                            mod_opts.unwind_tables = .sync;
                        },
                    } else {
                        mod_opts.unwind_tables = .sync;
                    },
                    .nostdlib => {
                        create_module.opts.ensure_libc_on_non_freestanding = false;
                        create_module.opts.ensure_libcpp_on_non_freestanding = false;
                    },
                    .nostdlib_cpp => create_module.opts.ensure_libcpp_on_non_freestanding = false,
                    .shared => {
                        create_module.opts.link_mode = .dynamic;
                        is_shared_lib = true;
                    },
                    .rdynamic => create_module.opts.rdynamic = true,
                    .wp => {
                        var split_it = mem.splitScalar(u8, it.only_arg, ',');
                        while (split_it.next()) |preprocessor_arg| {
                            if (preprocessor_arg.len >= 3 and
                                preprocessor_arg[0] == '-' and
                                preprocessor_arg[2] != '-')
                            {
                                if (mem.indexOfScalar(u8, preprocessor_arg, '=')) |equals_pos| {
                                    const key = preprocessor_arg[0..equals_pos];
                                    const value = preprocessor_arg[equals_pos + 1 ..];
                                    try preprocessor_args.append(key);
                                    try preprocessor_args.append(value);
                                    continue;
                                }
                            }
                            try preprocessor_args.append(preprocessor_arg);
                        }
                    },
                    .wl => {
                        var split_it = mem.splitScalar(u8, it.only_arg, ',');
                        while (split_it.next()) |linker_arg| {
                            // Handle nested-joined args like `-Wl,-rpath=foo`.
                            // Must be prefixed with 1 or 2 dashes.
                            if (linker_arg.len >= 3 and
                                linker_arg[0] == '-' and
                                linker_arg[2] != '-')
                            {
                                if (mem.indexOfScalar(u8, linker_arg, '=')) |equals_pos| {
                                    const key = linker_arg[0..equals_pos];
                                    const value = linker_arg[equals_pos + 1 ..];
                                    if (mem.eql(u8, key, "--build-id")) {
                                        build_id = std.zig.BuildId.parse(value) catch |err| {
                                            fatal("unable to parse --build-id style '{s}': {s}", .{
                                                value, @errorName(err),
                                            });
                                        };
                                        continue;
                                    } else if (mem.eql(u8, key, "--sort-common")) {
                                        // this ignores --sort=common=<anything>; ignoring plain --sort-common
                                        // is done below.
                                        continue;
                                    }
                                    try linker_args.append(key);
                                    try linker_args.append(value);
                                    continue;
                                }
                            }
                            if (mem.eql(u8, linker_arg, "--build-id")) {
                                build_id = .fast;
                            } else if (mem.eql(u8, linker_arg, "--as-needed")) {
                                needed = false;
                            } else if (mem.eql(u8, linker_arg, "--no-as-needed")) {
                                needed = true;
                            } else if (mem.eql(u8, linker_arg, "-no-pie")) {
                                create_module.opts.pie = false;
                            } else if (mem.eql(u8, linker_arg, "--sort-common")) {
                                // from ld.lld(1): --sort-common is ignored for GNU compatibility,
                                // this ignores plain --sort-common
                            } else if (mem.eql(u8, linker_arg, "--whole-archive") or
                                mem.eql(u8, linker_arg, "-whole-archive"))
                            {
                                must_link = true;
                            } else if (mem.eql(u8, linker_arg, "--no-whole-archive") or
                                mem.eql(u8, linker_arg, "-no-whole-archive"))
                            {
                                must_link = false;
                            } else if (mem.eql(u8, linker_arg, "-Bdynamic") or
                                mem.eql(u8, linker_arg, "-dy") or
                                mem.eql(u8, linker_arg, "-call_shared"))
                            {
                                lib_search_strategy = .no_fallback;
                                lib_preferred_mode = .dynamic;
                            } else if (mem.eql(u8, linker_arg, "-Bstatic") or
                                mem.eql(u8, linker_arg, "-dn") or
                                mem.eql(u8, linker_arg, "-non_shared") or
                                mem.eql(u8, linker_arg, "-static"))
                            {
                                lib_search_strategy = .no_fallback;
                                lib_preferred_mode = .static;
                            } else if (mem.eql(u8, linker_arg, "-search_paths_first")) {
                                lib_search_strategy = .paths_first;
                                lib_preferred_mode = .dynamic;
                            } else if (mem.eql(u8, linker_arg, "-search_dylibs_first")) {
                                lib_search_strategy = .mode_first;
                                lib_preferred_mode = .dynamic;
                            } else {
                                try linker_args.append(linker_arg);
                            }
                        }
                    },
                    .san_cov_trace_pc_guard => create_module.opts.san_cov_trace_pc_guard = true,
                    .san_cov => {
                        var split_it = mem.splitScalar(u8, it.only_arg, ',');
                        while (split_it.next()) |san_arg| {
                            if (std.mem.eql(u8, san_arg, "trace-pc-guard")) {
                                create_module.opts.san_cov_trace_pc_guard = true;
                            }
                        }
                        try cc_argv.appendSlice(arena, it.other_args);
                    },
                    .no_san_cov => {
                        var split_it = mem.splitScalar(u8, it.only_arg, ',');
                        while (split_it.next()) |san_arg| {
                            if (std.mem.eql(u8, san_arg, "trace-pc-guard")) {
                                create_module.opts.san_cov_trace_pc_guard = false;
                            }
                        }
                        try cc_argv.appendSlice(arena, it.other_args);
                    },
                    .optimize => {
                        // Alright, what release mode do they want?
                        const level = if (it.only_arg.len >= 1 and it.only_arg[0] == 'O') it.only_arg[1..] else it.only_arg;
                        if (mem.eql(u8, level, "s") or
                            mem.eql(u8, level, "z"))
                        {
                            mod_opts.optimize_mode = .ReleaseSmall;
                        } else if (mem.eql(u8, level, "1") or
                            mem.eql(u8, level, "2") or
                            mem.eql(u8, level, "3") or
                            mem.eql(u8, level, "4") or
                            mem.eql(u8, level, "fast"))
                        {
                            mod_opts.optimize_mode = .ReleaseFast;
                        } else if (mem.eql(u8, level, "g") or
                            mem.eql(u8, level, "0"))
                        {
                            mod_opts.optimize_mode = .Debug;
                        } else {
                            try cc_argv.appendSlice(arena, it.other_args);
                        }
                    },
                    .debug => {
                        mod_opts.strip = false;
                        if (mem.eql(u8, it.only_arg, "g")) {
                            // We handled with strip = false above.
                        } else if (mem.eql(u8, it.only_arg, "g1") or
                            mem.eql(u8, it.only_arg, "gline-tables-only"))
                        {
                            // We handled with strip = false above. but we also want reduced debug info.
                            try cc_argv.append(arena, "-gline-tables-only");
                        } else {
                            try cc_argv.appendSlice(arena, it.other_args);
                        }
                    },
                    .gdwarf32 => {
                        mod_opts.strip = false;
                        create_module.opts.debug_format = .{ .dwarf = .@"32" };
                    },
                    .gdwarf64 => {
                        mod_opts.strip = false;
                        create_module.opts.debug_format = .{ .dwarf = .@"64" };
                    },
                    .sanitize => {
                        var san_it = std.mem.splitScalar(u8, it.only_arg, ',');
                        var recognized_any = false;
                        while (san_it.next()) |sub_arg| {
                            if (mem.eql(u8, sub_arg, "undefined")) {
                                mod_opts.sanitize_c = true;
                                recognized_any = true;
                            } else if (mem.eql(u8, sub_arg, "thread")) {
                                mod_opts.sanitize_thread = true;
                                recognized_any = true;
                            } else if (mem.eql(u8, sub_arg, "fuzzer") or mem.eql(u8, sub_arg, "fuzzer-no-link")) {
                                mod_opts.fuzz = true;
                                recognized_any = true;
                            }
                        }
                        if (!recognized_any) {
                            try cc_argv.appendSlice(arena, it.other_args);
                        }
                    },
                    .linker_script => linker_script = it.only_arg,
                    .verbose => {
                        verbose_link = true;
                        // Have Clang print more infos, some tools such as CMake
                        // parse this to discover any implicit include and
                        // library dir to look-up into.
                        try cc_argv.append(arena, "-v");
                    },
                    .dry_run => {
                        // This flag means "dry run". Clang will not actually output anything
                        // to the file system.
                        verbose_link = true;
                        disable_c_depfile = true;
                        try cc_argv.append(arena, "-###");
                    },
                    .for_linker => try linker_args.append(it.only_arg),
                    .linker_input_z => {
                        try linker_args.append("-z");
                        try linker_args.append(it.only_arg);
                    },
                    .lib_dir => try create_module.lib_dir_args.append(arena, it.only_arg),
                    .mcpu => target_mcpu = it.only_arg,
                    .m => try create_module.llvm_m_args.append(arena, it.only_arg),
                    .dep_file => {
                        disable_c_depfile = true;
                        try cc_argv.appendSlice(arena, it.other_args);
                    },
                    .dep_file_to_stdout => { // -M, -MM
                        // "Like -MD, but also implies -E and writes to stdout by default"
                        // "Like -MMD, but also implies -E and writes to stdout by default"
                        c_out_mode = .preprocessor;
                        disable_c_depfile = true;
                        try cc_argv.appendSlice(arena, it.other_args);
                    },
                    .framework_dir => try create_module.framework_dirs.append(arena, it.only_arg),
                    .framework => try create_module.frameworks.put(arena, it.only_arg, .{}),
                    .nostdlibinc => create_module.want_native_include_dirs = false,
                    .strip => mod_opts.strip = true,
                    .exec_model => {
                        create_module.opts.wasi_exec_model = parseWasiExecModel(it.only_arg);
                    },
                    .sysroot => {
                        create_module.sysroot = it.only_arg;
                    },
                    .entry => {
                        entry = .{ .named = it.only_arg };
                    },
                    .force_undefined_symbol => {
                        try force_undefined_symbols.put(arena, it.only_arg, {});
                    },
                    .force_load_objc => force_load_objc = true,
                    .mingw_unicode_entry_point => mingw_unicode_entry_point = true,
                    .weak_library => try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                        .name = it.only_arg,
                        .query = .{
                            .needed = false,
                            .weak = true,
                            .preferred_mode = lib_preferred_mode,
                            .search_strategy = lib_search_strategy,
                            .allow_so_scripts = allow_so_scripts,
                        },
                    } }),
                    .weak_framework => try create_module.frameworks.put(arena, it.only_arg, .{ .weak = true }),
                    .headerpad_max_install_names => headerpad_max_install_names = true,
                    .compress_debug_sections => {
                        if (it.only_arg.len == 0) {
                            linker_compress_debug_sections = .zlib;
                        } else {
                            linker_compress_debug_sections = std.meta.stringToEnum(link.File.Elf.CompressDebugSections, it.only_arg) orelse {
                                fatal("expected [none|zlib|zstd] after --compress-debug-sections, found '{s}'", .{it.only_arg});
                            };
                        }
                    },
                    .install_name => {
                        install_name = it.only_arg;
                    },
                    .undefined => {
                        if (mem.eql(u8, "dynamic_lookup", it.only_arg)) {
                            linker_allow_shlib_undefined = true;
                        } else if (mem.eql(u8, "error", it.only_arg)) {
                            linker_allow_shlib_undefined = false;
                        } else {
                            fatal("unsupported -undefined option '{s}'", .{it.only_arg});
                        }
                    },
                    .rtlib => {
                        // Unlike Clang, we support `none` for explicitly omitting compiler-rt.
                        if (mem.eql(u8, "none", it.only_arg)) {
                            want_compiler_rt = false;
                        } else if (mem.eql(u8, "compiler-rt", it.only_arg) or
                            mem.eql(u8, "libgcc", it.only_arg))
                        {
                            want_compiler_rt = true;
                        } else {
                            // Note that we don't support `platform`.
                            fatal("unsupported -rtlib option '{s}'", .{it.only_arg});
                        }
                    },
                }
            }
            // Parse linker args.
            var linker_args_it = ArgsIterator{
                .args = linker_args.items,
            };
            while (linker_args_it.next()) |arg| {
                if (mem.eql(u8, arg, "-soname") or
                    mem.eql(u8, arg, "--soname"))
                {
                    const name = linker_args_it.nextOrFatal();
                    soname = .{ .yes = name };
                    // Use it as --name.
                    // Example: libsoundio.so.2
                    var prefix: usize = 0;
                    if (mem.startsWith(u8, name, "lib")) {
                        prefix = 3;
                    }
                    var end: usize = name.len;
                    if (mem.endsWith(u8, name, ".so")) {
                        end -= 3;
                    } else {
                        var found_digit = false;
                        while (end > 0 and std.ascii.isDigit(name[end - 1])) {
                            found_digit = true;
                            end -= 1;
                        }
                        if (found_digit and end > 0 and name[end - 1] == '.') {
                            end -= 1;
                        } else {
                            end = name.len;
                        }
                        if (mem.endsWith(u8, name[prefix..end], ".so")) {
                            end -= 3;
                        }
                    }
                    provided_name = name[prefix..end];
                } else if (mem.eql(u8, arg, "-rpath") or mem.eql(u8, arg, "--rpath") or mem.eql(u8, arg, "-R")) {
                    try create_module.rpath_list.append(arena, linker_args_it.nextOrFatal());
                } else if (mem.eql(u8, arg, "--subsystem")) {
                    subsystem = try parseSubSystem(linker_args_it.nextOrFatal());
                } else if (mem.eql(u8, arg, "-I") or
                    mem.eql(u8, arg, "--dynamic-linker") or
                    mem.eql(u8, arg, "-dynamic-linker"))
                {
                    create_module.dynamic_linker = linker_args_it.nextOrFatal();
                } else if (mem.eql(u8, arg, "-E") or
                    mem.eql(u8, arg, "--export-dynamic") or
                    mem.eql(u8, arg, "-export-dynamic"))
                {
                    create_module.opts.rdynamic = true;
                } else if (mem.eql(u8, arg, "-version-script") or mem.eql(u8, arg, "--version-script")) {
                    version_script = linker_args_it.nextOrFatal();
                } else if (mem.eql(u8, arg, "--undefined-version")) {
                    linker_allow_undefined_version = true;
                } else if (mem.eql(u8, arg, "--no-undefined-version")) {
                    linker_allow_undefined_version = false;
                } else if (mem.eql(u8, arg, "--enable-new-dtags")) {
                    linker_enable_new_dtags = true;
                } else if (mem.eql(u8, arg, "--disable-new-dtags")) {
                    linker_enable_new_dtags = false;
                } else if (mem.eql(u8, arg, "-O")) {
                    linker_optimization = linker_args_it.nextOrFatal();
                } else if (mem.startsWith(u8, arg, "-O")) {
                    linker_optimization = arg["-O".len..];
                } else if (mem.eql(u8, arg, "-pagezero_size")) {
                    const next_arg = linker_args_it.nextOrFatal();
                    pagezero_size = std.fmt.parseUnsigned(u64, eatIntPrefix(next_arg, 16), 16) catch |err| {
                        fatal("unable to parse pagezero size '{s}': {s}", .{ next_arg, @errorName(err) });
                    };
                } else if (mem.eql(u8, arg, "-headerpad")) {
                    const next_arg = linker_args_it.nextOrFatal();
                    headerpad_size = std.fmt.parseUnsigned(u32, eatIntPrefix(next_arg, 16), 16) catch |err| {
                        fatal("unable to parse  headerpad size '{s}': {s}", .{ next_arg, @errorName(err) });
                    };
                } else if (mem.eql(u8, arg, "-headerpad_max_install_names")) {
                    headerpad_max_install_names = true;
                } else if (mem.eql(u8, arg, "-dead_strip")) {
                    linker_gc_sections = true;
                } else if (mem.eql(u8, arg, "-dead_strip_dylibs")) {
                    dead_strip_dylibs = true;
                } else if (mem.eql(u8, arg, "-ObjC")) {
                    force_load_objc = true;
                } else if (mem.eql(u8, arg, "--no-undefined")) {
                    linker_z_defs = true;
                } else if (mem.eql(u8, arg, "--gc-sections")) {
                    linker_gc_sections = true;
                } else if (mem.eql(u8, arg, "--no-gc-sections")) {
                    linker_gc_sections = false;
                } else if (mem.eql(u8, arg, "--print-gc-sections")) {
                    linker_print_gc_sections = true;
                } else if (mem.eql(u8, arg, "--print-icf-sections")) {
                    linker_print_icf_sections = true;
                } else if (mem.eql(u8, arg, "--print-map")) {
                    linker_print_map = true;
                } else if (mem.eql(u8, arg, "--sort-section")) {
                    const arg1 = linker_args_it.nextOrFatal();
                    linker_sort_section = std.meta.stringToEnum(link.File.Elf.SortSection, arg1) orelse {
                        fatal("expected [name|alignment] after --sort-section, found '{s}'", .{arg1});
                    };
                } else if (mem.eql(u8, arg, "--allow-shlib-undefined") or
                    mem.eql(u8, arg, "-allow-shlib-undefined"))
                {
                    linker_allow_shlib_undefined = true;
                } else if (mem.eql(u8, arg, "--no-allow-shlib-undefined") or
                    mem.eql(u8, arg, "-no-allow-shlib-undefined"))
                {
                    linker_allow_shlib_undefined = false;
                } else if (mem.eql(u8, arg, "-Bsymbolic")) {
                    linker_bind_global_refs_locally = true;
                } else if (mem.eql(u8, arg, "--import-memory")) {
                    create_module.opts.import_memory = true;
                } else if (mem.eql(u8, arg, "--export-memory")) {
                    create_module.opts.export_memory = true;
                } else if (mem.eql(u8, arg, "--import-symbols")) {
                    linker_import_symbols = true;
                } else if (mem.eql(u8, arg, "--import-table")) {
                    linker_import_table = true;
                } else if (mem.eql(u8, arg, "--export-table")) {
                    linker_export_table = true;
                } else if (mem.eql(u8, arg, "--no-entry")) {
                    entry = .disabled;
                } else if (mem.eql(u8, arg, "--initial-memory")) {
                    const next_arg = linker_args_it.nextOrFatal();
                    linker_initial_memory = std.fmt.parseUnsigned(u32, next_arg, 10) catch |err| {
                        fatal("unable to parse initial memory size '{s}': {s}", .{ next_arg, @errorName(err) });
                    };
                } else if (mem.eql(u8, arg, "--max-memory")) {
                    const next_arg = linker_args_it.nextOrFatal();
                    linker_max_memory = std.fmt.parseUnsigned(u32, next_arg, 10) catch |err| {
                        fatal("unable to parse max memory size '{s}': {s}", .{ next_arg, @errorName(err) });
                    };
                } else if (mem.eql(u8, arg, "--shared-memory")) {
                    create_module.opts.shared_memory = true;
                } else if (mem.eql(u8, arg, "--global-base")) {
                    const next_arg = linker_args_it.nextOrFatal();
                    linker_global_base = std.fmt.parseUnsigned(u32, next_arg, 10) catch |err| {
                        fatal("unable to parse global base '{s}': {s}", .{ next_arg, @errorName(err) });
                    };
                } else if (mem.eql(u8, arg, "--export")) {
                    try linker_export_symbol_names.append(arena, linker_args_it.nextOrFatal());
                } else if (mem.eql(u8, arg, "--compress-debug-sections")) {
                    const arg1 = linker_args_it.nextOrFatal();
                    linker_compress_debug_sections = std.meta.stringToEnum(link.File.Elf.CompressDebugSections, arg1) orelse {
                        fatal("expected [none|zlib|zstd] after --compress-debug-sections, found '{s}'", .{arg1});
                    };
                } else if (mem.startsWith(u8, arg, "-z")) {
                    var z_arg = arg[2..];
                    if (z_arg.len == 0) {
                        z_arg = linker_args_it.nextOrFatal();
                    }
                    if (mem.eql(u8, z_arg, "nodelete")) {
                        linker_z_nodelete = true;
                    } else if (mem.eql(u8, z_arg, "notext")) {
                        linker_z_notext = true;
                    } else if (mem.eql(u8, z_arg, "defs")) {
                        linker_z_defs = true;
                    } else if (mem.eql(u8, z_arg, "undefs")) {
                        linker_z_defs = false;
                    } else if (mem.eql(u8, z_arg, "origin")) {
                        linker_z_origin = true;
                    } else if (mem.eql(u8, z_arg, "nocopyreloc")) {
                        linker_z_nocopyreloc = true;
                    } else if (mem.eql(u8, z_arg, "noexecstack")) {
                        // noexecstack is the default when linking with LLD
                    } else if (mem.eql(u8, z_arg, "now")) {
                        linker_z_now = true;
                    } else if (mem.eql(u8, z_arg, "lazy")) {
                        linker_z_now = false;
                    } else if (mem.eql(u8, z_arg, "relro")) {
                        linker_z_relro = true;
                    } else if (mem.eql(u8, z_arg, "norelro")) {
                        linker_z_relro = false;
                    } else if (mem.startsWith(u8, z_arg, "stack-size=")) {
                        stack_size = parseStackSize(z_arg["stack-size=".len..]);
                    } else if (mem.startsWith(u8, z_arg, "common-page-size=")) {
                        linker_z_common_page_size = parseIntSuffix(z_arg, "common-page-size=".len);
                    } else if (mem.startsWith(u8, z_arg, "max-page-size=")) {
                        linker_z_max_page_size = parseIntSuffix(z_arg, "max-page-size=".len);
                    } else {
                        fatal("unsupported linker extension flag: -z {s}", .{z_arg});
                    }
                } else if (mem.eql(u8, arg, "--major-image-version")) {
                    const major = linker_args_it.nextOrFatal();
                    version.major = std.fmt.parseUnsigned(u32, major, 10) catch |err| {
                        fatal("unable to parse major image version '{s}': {s}", .{ major, @errorName(err) });
                    };
                    have_version = true;
                } else if (mem.eql(u8, arg, "--minor-image-version")) {
                    const minor = linker_args_it.nextOrFatal();
                    version.minor = std.fmt.parseUnsigned(u32, minor, 10) catch |err| {
                        fatal("unable to parse minor image version '{s}': {s}", .{ minor, @errorName(err) });
                    };
                    have_version = true;
                } else if (mem.eql(u8, arg, "-e") or mem.eql(u8, arg, "--entry")) {
                    entry = .{ .named = linker_args_it.nextOrFatal() };
                } else if (mem.eql(u8, arg, "-u")) {
                    try force_undefined_symbols.put(arena, linker_args_it.nextOrFatal(), {});
                } else if (mem.eql(u8, arg, "-x") or mem.eql(u8, arg, "--discard-all")) {
                    discard_local_symbols = true;
                } else if (mem.eql(u8, arg, "--stack") or mem.eql(u8, arg, "-stack_size")) {
                    stack_size = parseStackSize(linker_args_it.nextOrFatal());
                } else if (mem.eql(u8, arg, "--image-base")) {
                    image_base = parseImageBase(linker_args_it.nextOrFatal());
                } else if (mem.eql(u8, arg, "--enable-auto-image-base") or
                    mem.eql(u8, arg, "--disable-auto-image-base"))
                {
                    // `--enable-auto-image-base` is a flag that binutils added in ~2000 for MinGW.
                    // It does a hash of the file and uses that as part of the image base value.
                    // Presumably the idea was to avoid DLLs needing to be relocated when loaded.
                    // This is practically irrelevant today as all PEs produced since Windows Vista
                    // have ASLR enabled by default anyway, and Windows 10+ has Mandatory ASLR which
                    // doesn't even care what the PE file wants and relocates it anyway.
                    //
                    // Unfortunately, Libtool hardcodes usage of this archaic flag when targeting
                    // MinGW, so to make `zig cc` for that use case work, accept and ignore the
                    // flag, and warn the user that it has no effect.
                    warn("auto-image-base options are unimplemented and ignored", .{});
                } else if (mem.eql(u8, arg, "-T") or mem.eql(u8, arg, "--script")) {
                    linker_script = linker_args_it.nextOrFatal();
                } else if (mem.eql(u8, arg, "--eh-frame-hdr")) {
                    link_eh_frame_hdr = true;
                } else if (mem.eql(u8, arg, "--no-eh-frame-hdr")) {
                    link_eh_frame_hdr = false;
                } else if (mem.eql(u8, arg, "--tsaware")) {
                    linker_tsaware = true;
                } else if (mem.eql(u8, arg, "--nxcompat")) {
                    linker_nxcompat = true;
                } else if (mem.eql(u8, arg, "--dynamicbase")) {
                    linker_dynamicbase = true;
                } else if (mem.eql(u8, arg, "--no-dynamicbase")) {
                    linker_dynamicbase = false;
                } else if (mem.eql(u8, arg, "--high-entropy-va")) {
                    // This option does not do anything.
                } else if (mem.eql(u8, arg, "--export-all-symbols")) {
                    create_module.opts.rdynamic = true;
                } else if (mem.eql(u8, arg, "--color-diagnostics") or
                    mem.eql(u8, arg, "--color-diagnostics=always"))
                {
                    color = .on;
                } else if (mem.eql(u8, arg, "--no-color-diagnostics") or
                    mem.eql(u8, arg, "--color-diagnostics=never"))
                {
                    color = .off;
                } else if (mem.eql(u8, arg, "-s") or mem.eql(u8, arg, "--strip-all") or
                    mem.eql(u8, arg, "-S") or mem.eql(u8, arg, "--strip-debug"))
                {
                    // -s, --strip-all             Strip all symbols
                    // -S, --strip-debug           Strip debugging symbols
                    mod_opts.strip = true;
                } else if (mem.eql(u8, arg, "--start-group") or
                    mem.eql(u8, arg, "--end-group"))
                {
                    // We don't need to care about these because these args are
                    // for resolving circular dependencies but our linker takes
                    // care of this without explicit args.
                } else if (mem.eql(u8, arg, "--major-os-version") or
                    mem.eql(u8, arg, "--minor-os-version"))
                {
                    // This option does not do anything.
                    _ = linker_args_it.nextOrFatal();
                } else if (mem.eql(u8, arg, "--major-subsystem-version")) {
                    const major = linker_args_it.nextOrFatal();
                    major_subsystem_version = std.fmt.parseUnsigned(u16, major, 10) catch |err| {
                        fatal("unable to parse major subsystem version '{s}': {s}", .{
                            major, @errorName(err),
                        });
                    };
                } else if (mem.eql(u8, arg, "--minor-subsystem-version")) {
                    const minor = linker_args_it.nextOrFatal();
                    minor_subsystem_version = std.fmt.parseUnsigned(u16, minor, 10) catch |err| {
                        fatal("unable to parse minor subsystem version '{s}': {s}", .{
                            minor, @errorName(err),
                        });
                    };
                } else if (mem.eql(u8, arg, "-framework")) {
                    try create_module.frameworks.put(arena, linker_args_it.nextOrFatal(), .{});
                } else if (mem.eql(u8, arg, "-weak_framework")) {
                    try create_module.frameworks.put(arena, linker_args_it.nextOrFatal(), .{ .weak = true });
                } else if (mem.eql(u8, arg, "-needed_framework")) {
                    try create_module.frameworks.put(arena, linker_args_it.nextOrFatal(), .{ .needed = true });
                } else if (mem.eql(u8, arg, "-needed_library")) {
                    try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                        .name = linker_args_it.nextOrFatal(),
                        .query = .{
                            .weak = false,
                            .needed = true,
                            .preferred_mode = lib_preferred_mode,
                            .search_strategy = lib_search_strategy,
                            .allow_so_scripts = allow_so_scripts,
                        },
                    } });
                } else if (mem.startsWith(u8, arg, "-weak-l")) {
                    try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                        .name = arg["-weak-l".len..],
                        .query = .{
                            .weak = true,
                            .needed = false,
                            .preferred_mode = lib_preferred_mode,
                            .search_strategy = lib_search_strategy,
                            .allow_so_scripts = allow_so_scripts,
                        },
                    } });
                } else if (mem.eql(u8, arg, "-weak_library")) {
                    try create_module.cli_link_inputs.append(arena, .{ .name_query = .{
                        .name = linker_args_it.nextOrFatal(),
                        .query = .{
                            .weak = true,
                            .needed = false,
                            .preferred_mode = lib_preferred_mode,
                            .search_strategy = lib_search_strategy,
                            .allow_so_scripts = allow_so_scripts,
                        },
                    } });
                } else if (mem.eql(u8, arg, "-compatibility_version")) {
                    const compat_version = linker_args_it.nextOrFatal();
                    compatibility_version = std.SemanticVersion.parse(compat_version) catch |err| {
                        fatal("unable to parse -compatibility_version '{s}': {s}", .{ compat_version, @errorName(err) });
                    };
                } else if (mem.eql(u8, arg, "-current_version")) {
                    const curr_version = linker_args_it.nextOrFatal();
                    version = std.SemanticVersion.parse(curr_version) catch |err| {
                        fatal("unable to parse -current_version '{s}': {s}", .{ curr_version, @errorName(err) });
                    };
                    have_version = true;
                } else if (mem.eql(u8, arg, "--out-implib") or
                    mem.eql(u8, arg, "-implib"))
                {
                    emit_implib = .{ .yes = linker_args_it.nextOrFatal() };
                    emit_implib_arg_provided = true;
                } else if (mem.eql(u8, arg, "-Brepro") or mem.eql(u8, arg, "/Brepro")) {
                    linker_repro = true;
                } else if (mem.eql(u8, arg, "-undefined")) {
                    const lookup_type = linker_args_it.nextOrFatal();
                    if (mem.eql(u8, "dynamic_lookup", lookup_type)) {
                        linker_allow_shlib_undefined = true;
                    } else if (mem.eql(u8, "error", lookup_type)) {
                        linker_allow_shlib_undefined = false;
                    } else {
                        fatal("unsupported -undefined option '{s}'", .{lookup_type});
                    }
                } else if (mem.eql(u8, arg, "-install_name")) {
                    install_name = linker_args_it.nextOrFatal();
                } else if (mem.eql(u8, arg, "-force_load")) {
                    try create_module.cli_link_inputs.append(arena, .{ .path_query = .{
                        .path = Path.initCwd(linker_args_it.nextOrFatal()),
                        .query = .{
                            .must_link = true,
                            .preferred_mode = .static,
                            .search_strategy = .no_fallback,
                        },
                    } });
                } else if (mem.eql(u8, arg, "-hash-style") or
                    mem.eql(u8, arg, "--hash-style"))
                {
                    const next_arg = linker_args_it.nextOrFatal();
                    hash_style = std.meta.stringToEnum(link.File.Elf.HashStyle, next_arg) orelse {
                        fatal("expected [sysv|gnu|both] after --hash-style, found '{s}'", .{
                            next_arg,
                        });
                    };
                } else if (mem.eql(u8, arg, "-wrap")) {
                    const next_arg = linker_args_it.nextOrFatal();
                    try symbol_wrap_set.put(arena, next_arg, {});
                } else if (mem.startsWith(u8, arg, "/subsystem:")) {
                    var split_it = mem.splitBackwardsScalar(u8, arg, ':');
                    subsystem = try parseSubSystem(split_it.first());
                } else if (mem.startsWith(u8, arg, "/implib:")) {
                    var split_it = mem.splitBackwardsScalar(u8, arg, ':');
                    emit_implib = .{ .yes = split_it.first() };
                    emit_implib_arg_provided = true;
                } else if (mem.startsWith(u8, arg, "/pdb:")) {
                    var split_it = mem.splitBackwardsScalar(u8, arg, ':');
                    pdb_out_path = split_it.first();
                } else if (mem.startsWith(u8, arg, "/version:")) {
                    var split_it = mem.splitBackwardsScalar(u8, arg, ':');
                    const version_arg = split_it.first();
                    version = std.SemanticVersion.parse(version_arg) catch |err| {
                        fatal("unable to parse /version '{s}': {s}", .{ arg, @errorName(err) });
                    };
                    have_version = true;
                } else if (mem.eql(u8, arg, "-V")) {
                    warn("ignoring request for supported emulations: unimplemented", .{});
                } else if (mem.eql(u8, arg, "-v")) {
                    try std.io.getStdOut().writeAll("zig ld " ++ build_options.version ++ "\n");
                } else if (mem.eql(u8, arg, "--version")) {
                    try std.io.getStdOut().writeAll("zig ld " ++ build_options.version ++ "\n");
                    process.exit(0);
                } else {
                    fatal("unsupported linker arg: {s}", .{arg});
                }
            }

            // Parse preprocessor args.
            var preprocessor_args_it = ArgsIterator{
                .args = preprocessor_args.items,
            };
            while (preprocessor_args_it.next()) |arg| {
                if (mem.eql(u8, arg, "-MD") or mem.eql(u8, arg, "-MMD") or mem.eql(u8, arg, "-MT")) {
                    disable_c_depfile = true;
                    const cc_arg = try std.fmt.allocPrint(arena, "-Wp,{s},{s}", .{ arg, preprocessor_args_it.nextOrFatal() });
                    try cc_argv.append(arena, cc_arg);
                } else {
                    fatal("unsupported preprocessor arg: {s}", .{arg});
                }
            }

            if (mod_opts.sanitize_c) |wsc| {
                if (wsc and mod_opts.optimize_mode == .ReleaseFast) {
                    mod_opts.optimize_mode = .ReleaseSafe;
                }
            }

            // precompiled header syntax: "zig cc -x c-header test.h -o test.pch"
            const emit_pch = ((file_ext == .h or file_ext == .hpp or file_ext == .hm or file_ext == .hmm) and c_out_mode == null);
            if (emit_pch)
                c_out_mode = .preprocessor;

            switch (c_out_mode orelse .link) {
                .link => {
                    create_module.opts.output_mode = if (is_shared_lib) .Lib else .Exe;
                    if (emit_bin != .no) {
                        emit_bin = if (out_path) |p| .{ .yes = p } else EmitBin.yes_a_out;
                    }
                    if (emit_llvm) {
                        fatal("-emit-llvm cannot be used when linking", .{});
                    }
                },
                .object => {
                    create_module.opts.output_mode = .Obj;
                    if (emit_llvm) {
                        emit_bin = .no;
                        if (out_path) |p| {
                            emit_llvm_bc = .{ .yes = p };
                        } else {
                            emit_llvm_bc = .yes_default_path;
                        }
                    } else {
                        if (out_path) |p| {
                            emit_bin = .{ .yes = p };
                        } else {
                            emit_bin = .yes_default_path;
                        }
                    }
                },
                .assembly => {
                    create_module.opts.output_mode = .Obj;
                    emit_bin = .no;
                    if (emit_llvm) {
                        if (out_path) |p| {
                            emit_llvm_ir = .{ .yes = p };
                        } else {
                            emit_llvm_ir = .yes_default_path;
                        }
                    } else {
                        if (out_path) |p| {
                            emit_asm = .{ .yes = p };
                        } else {
                            emit_asm = .yes_default_path;
                        }
                    }
                },
                .preprocessor => {
                    create_module.opts.output_mode = .Obj;
                    // An error message is generated when there is more than 1 C source file.
                    if (create_module.c_source_files.items.len != 1) {
                        // For example `zig cc` and no args should print the "no input files" message.
                        return process.exit(try clangMain(arena, all_args));
                    }
                    if (emit_pch) {
                        emit_bin = if (out_path) |p| .{ .yes = p } else .yes_default_path;
                        clang_preprocessor_mode = .pch;
                    } else {
                        // If the output path is "-" (stdout), then we need to emit the preprocessed output to stdout
                        // like "clang -E main.c -o -" does.
                        if (out_path != null and !mem.eql(u8, out_path.?, "-")) {
                            emit_bin = .{ .yes = out_path.? };
                            clang_preprocessor_mode = .yes;
                        } else {
                            emit_bin = .no;
                            clang_preprocessor_mode = .stdout;
                        }
                    }
                },
            }
            if (create_module.c_source_files.items.len == 0 and
                !anyObjectLinkInputs(create_module.cli_link_inputs.items) and
                root_src_file == null)
            {
                // For example `zig cc` and no args should print the "no input files" message.
                // There could be other reasons to punt to clang, for example, --help.
                return process.exit(try clangMain(arena, all_args));
            }
        },
    }

    if (arg_mode == .translate_c and create_module.c_source_files.items.len != 1) {
        fatal("translate-c expects exactly 1 source file (found {d})", .{create_module.c_source_files.items.len});
    }

    if (show_builtin and root_src_file == null) {
        // Without this, there will be no main module created and no zig
        // compilation unit, and therefore also no builtin.zig contents
        // created.
        root_src_file = "builtin.zig";
    }

    implicit_root_mod: {
        const unresolved_src_path = b: {
            if (root_src_file) |src_path| {
                if (create_module.modules.count() != 0) {
                    fatal("main module provided both by '-M{s}={}{s}' and by positional argument '{s}'", .{
                        create_module.modules.keys()[0],
                        create_module.modules.values()[0].paths.root,
                        create_module.modules.values()[0].paths.root_src_path,
                        src_path,
                    });
                }
                create_module.opts.have_zcu = true;
                break :b src_path;
            }

            if (create_module.modules.count() != 0)
                break :implicit_root_mod;

            if (create_module.c_source_files.items.len >= 1)
                break :b create_module.c_source_files.items[0].src_path;

            for (create_module.cli_link_inputs.items) |unresolved_link_input| switch (unresolved_link_input) {
                // Intentionally includes dynamic libraries provided by file path.
                .path_query => |pq| break :b pq.path.sub_path,
                else => continue,
            };

            if (emit_bin == .yes)
                break :b emit_bin.yes;

            if (create_module.rc_source_files.items.len >= 1)
                break :b create_module.rc_source_files.items[0].src_path;

            if (arg_mode == .run)
                fatal("`zig run` expects at least one positional argument", .{});

            fatal("expected a positional argument, -femit-bin=[path], --show-builtin, or --name [name]", .{});

            break :implicit_root_mod;
        };

        // See duplicate logic: ModCreationGlobalFlags
        if (mod_opts.single_threaded == false)
            create_module.opts.any_non_single_threaded = true;
        if (mod_opts.sanitize_thread == true)
            create_module.opts.any_sanitize_thread = true;
        if (mod_opts.fuzz == true)
            create_module.opts.any_fuzz = true;
        if (mod_opts.unwind_tables) |uwt| switch (uwt) {
            .none => {},
            .sync, .@"async" => create_module.opts.any_unwind_tables = true,
        };
        if (mod_opts.strip == false)
            create_module.opts.any_non_stripped = true;
        if (mod_opts.error_tracing == true)
            create_module.opts.any_error_tracing = true;

        const src_path = try introspect.resolvePath(arena, unresolved_src_path);
        const name = if (arg_mode == .zig_test)
            "test"
        else
            fs.path.stem(fs.path.basename(src_path));

        try create_module.modules.put(arena, name, .{
            .paths = .{
                .root = .{
                    .root_dir = Cache.Directory.cwd(),
                    .sub_path = fs.path.dirname(src_path) orelse "",
                },
                .root_src_path = fs.path.basename(src_path),
            },
            .cc_argv = try cc_argv.toOwnedSlice(arena),
            .inherited = mod_opts,
            .target_arch_os_abi = target_arch_os_abi,
            .target_mcpu = target_mcpu,
            .deps = try deps.toOwnedSlice(arena),
            .resolved = null,
            .c_source_files_start = c_source_files_owner_index,
            .c_source_files_end = create_module.c_source_files.items.len,
            .rc_source_files_start = rc_source_files_owner_index,
            .rc_source_files_end = create_module.rc_source_files.items.len,
        });
        cssan.reset();
        mod_opts = .{};
        target_arch_os_abi = null;
        target_mcpu = null;
        c_source_files_owner_index = create_module.c_source_files.items.len;
        rc_source_files_owner_index = create_module.rc_source_files.items.len;
    }

    if (!create_module.opts.have_zcu and arg_mode == .zig_test) {
        fatal("`zig test` expects a zig source file argument", .{});
    }

    if (c_source_files_owner_index != create_module.c_source_files.items.len) {
        fatal("C source file '{s}' has no parent module", .{
            create_module.c_source_files.items[c_source_files_owner_index].src_path,
        });
    }

    if (rc_source_files_owner_index != create_module.rc_source_files.items.len) {
        fatal("resource file '{s}' has no parent module", .{
            create_module.rc_source_files.items[rc_source_files_owner_index].src_path,
        });
    }

    const self_exe_path: ?[]const u8 = if (!process.can_spawn)
        null
    else
        introspect.findZigExePath(arena) catch |err| {
            fatal("unable to find zig self exe path: {s}", .{@errorName(err)});
        };

    var zig_lib_directory: Directory = d: {
        if (override_lib_dir) |unresolved_lib_dir| {
            const lib_dir = try introspect.resolvePath(arena, unresolved_lib_dir);
            break :d .{
                .path = lib_dir,
                .handle = fs.cwd().openDir(lib_dir, .{}) catch |err| {
                    fatal("unable to open zig lib directory '{s}': {s}", .{ lib_dir, @errorName(err) });
                },
            };
        } else if (native_os == .wasi) {
            break :d getWasiPreopen("/lib");
        } else if (self_exe_path) |p| {
            break :d introspect.findZigLibDirFromSelfExe(arena, p) catch |err| {
                fatal("unable to find zig installation directory '{s}': {s}", .{ p, @errorName(err) });
            };
        } else {
            unreachable;
        }
    };
    defer zig_lib_directory.handle.close();

    var global_cache_directory: Directory = l: {
        if (override_global_cache_dir) |p| {
            break :l .{
                .handle = try fs.cwd().makeOpenPath(p, .{}),
                .path = p,
            };
        }
        if (native_os == .wasi) {
            break :l getWasiPreopen("/cache");
        }
        const p = try introspect.resolveGlobalCacheDir(arena);
        break :l .{
            .handle = try fs.cwd().makeOpenPath(p, .{}),
            .path = p,
        };
    };
    defer global_cache_directory.handle.close();

    if (linker_optimization) |o| {
        warn("ignoring deprecated linker optimization setting '{s}'", .{o});
    }

    create_module.global_cache_directory = global_cache_directory;
    create_module.opts.emit_llvm_ir = emit_llvm_ir != .no;
    create_module.opts.emit_llvm_bc = emit_llvm_bc != .no;
    create_module.opts.emit_bin = emit_bin != .no;
    create_module.opts.any_c_source_files = create_module.c_source_files.items.len != 0;

    var builtin_modules: std.StringHashMapUnmanaged(*Package.Module) = .empty;
    // `builtin_modules` allocated into `arena`, so no deinit
    const main_mod = try createModule(gpa, arena, &create_module, 0, null, zig_lib_directory, &builtin_modules, color);
    for (create_module.modules.keys(), create_module.modules.values()) |key, cli_mod| {
        if (cli_mod.resolved == null)
            fatal("module '{s}' declared but not used", .{key});
    }

    // When you're testing std, the main module is std. In that case,
    // we'll just set the std module to the main one, since avoiding
    // the errors caused by duplicating it is more effort than it's
    // worth.
    const main_mod_is_std = m: {
        const std_path = try fs.path.resolve(arena, &.{
            zig_lib_directory.path orelse ".", "std", "std.zig",
        });
        const main_path = try fs.path.resolve(arena, &.{
            main_mod.root.root_dir.path orelse ".",
            main_mod.root.sub_path,
            main_mod.root_src_path,
        });
        break :m mem.eql(u8, main_path, std_path);
    };

    const std_mod = m: {
        if (main_mod_is_std) break :m main_mod;
        if (create_module.modules.get("std")) |cli_mod| break :m cli_mod.resolved.?;
        break :m null;
    };

    const root_mod = if (arg_mode == .zig_test) root_mod: {
        const test_mod = if (test_runner_path) |test_runner| test_mod: {
            const test_mod = try Package.Module.create(arena, .{
                .global_cache_directory = global_cache_directory,
                .paths = .{
                    .root = .{
                        .root_dir = Cache.Directory.cwd(),
                        .sub_path = fs.path.dirname(test_runner) orelse "",
                    },
                    .root_src_path = fs.path.basename(test_runner),
                },
                .fully_qualified_name = "root",
                .cc_argv = &.{},
                .inherited = .{},
                .global = create_module.resolved_options,
                .parent = main_mod,
                .builtin_mod = main_mod.getBuiltinDependency(),
                .builtin_modules = null, // `builtin_mod` is specified
            });
            test_mod.deps = try main_mod.deps.clone(arena);
            break :test_mod test_mod;
        } else try Package.Module.create(arena, .{
            .global_cache_directory = global_cache_directory,
            .paths = .{
                .root = .{
                    .root_dir = zig_lib_directory,
                    .sub_path = "compiler",
                },
                .root_src_path = "test_runner.zig",
            },
            .fully_qualified_name = "root",
            .cc_argv = &.{},
            .inherited = .{},
            .global = create_module.resolved_options,
            .parent = main_mod,
            .builtin_mod = main_mod.getBuiltinDependency(),
            .builtin_modules = null, // `builtin_mod` is specified
        });

        break :root_mod test_mod;
    } else main_mod;

    const target = main_mod.resolved_target.result;

    if (target.os.tag == .windows and major_subsystem_version == null and minor_subsystem_version == null) {
        major_subsystem_version, minor_subsystem_version = switch (target.os.version_range.windows.min) {
            .nt4 => .{ 4, 0 },
            .win2k => .{ 5, 0 },
            .xp => if (target.cpu.arch == .x86_64) .{ 5, 2 } else .{ 5, 1 },
            .ws2003 => .{ 5, 2 },
            else => .{ null, null },
        };
    }

    if (target.ofmt != .coff) {
        if (manifest_file != null) {
            fatal("manifest file is not allowed unless the target object format is coff (Windows/UEFI)", .{});
        }
        if (create_module.rc_source_files.items.len != 0) {
            fatal("rc files are not allowed unless the target object format is coff (Windows/UEFI)", .{});
        }
        if (contains_res_file) {
            fatal("res files are not allowed unless the target object format is coff (Windows/UEFI)", .{});
        }
    }

    var resolved_frameworks = std.ArrayList(Compilation.Framework).init(arena);

    if (create_module.frameworks.keys().len > 0) {
        var test_path = std.ArrayList(u8).init(gpa);
        defer test_path.deinit();

        var checked_paths = std.ArrayList(u8).init(gpa);
        defer checked_paths.deinit();

        var failed_frameworks = std.ArrayList(struct {
            name: []const u8,
            checked_paths: []const u8,
        }).init(arena);

        framework: for (create_module.frameworks.keys(), create_module.frameworks.values()) |framework_name, info| {
            checked_paths.clearRetainingCapacity();

            for (create_module.framework_dirs.items) |framework_dir_path| {
                if (try accessFrameworkPath(
                    &test_path,
                    &checked_paths,
                    framework_dir_path,
                    framework_name,
                )) {
                    const path = Path.initCwd(try arena.dupe(u8, test_path.items));
                    try resolved_frameworks.append(.{
                        .needed = info.needed,
                        .weak = info.weak,
                        .path = path,
                    });
                    continue :framework;
                }
            }

            try failed_frameworks.append(.{
                .name = framework_name,
                .checked_paths = try arena.dupe(u8, checked_paths.items),
            });
        }

        if (failed_frameworks.items.len > 0) {
            for (failed_frameworks.items) |f| {
                const searched_paths = if (f.checked_paths.len == 0) " none" else f.checked_paths;
                std.log.err("unable to find framework '{s}'. searched paths: {s}", .{
                    f.name, searched_paths,
                });
            }
            process.exit(1);
        }
    }
    // After this point, resolved_frameworks is used instead of frameworks.

    if (create_module.resolved_options.output_mode == .Obj and target.ofmt == .coff) {
        const total_obj_count = create_module.c_source_files.items.len +
            @intFromBool(root_src_file != null) +
            create_module.rc_source_files.items.len +
            link.countObjectInputs(create_module.link_inputs.items);
        if (total_obj_count > 1) {
            fatal("{s} does not support linking multiple objects into one", .{@tagName(target.ofmt)});
        }
    }

    var cleanup_emit_bin_dir: ?fs.Dir = null;
    defer if (cleanup_emit_bin_dir) |*dir| dir.close();

    const output_to_cache = listen != .none;
    const optional_version = if (have_version) version else null;

    const root_name = if (provided_name) |n| n else main_mod.fully_qualified_name;

    const resolved_soname: ?[]const u8 = switch (soname) {
        .yes => |explicit| explicit,
        .no => null,
        .yes_default_value => switch (target.ofmt) {
            .elf => if (have_version)
                try std.fmt.allocPrint(arena, "lib{s}.so.{d}", .{ root_name, version.major })
            else
                try std.fmt.allocPrint(arena, "lib{s}.so", .{root_name}),
            else => null,
        },
    };

    const a_out_basename = switch (target.ofmt) {
        .coff => "a.exe",
        else => "a.out",
    };

    const emit_bin_loc: ?Compilation.EmitLoc = switch (emit_bin) {
        .no => null,
        .yes_default_path => Compilation.EmitLoc{
            .directory = blk: {
                switch (arg_mode) {
                    .run, .zig_test => break :blk null,
                    else => {
                        if (output_to_cache) {
                            break :blk null;
                        } else {
                            break :blk .{ .path = null, .handle = fs.cwd() };
                        }
                    },
                }
            },
            .basename = if (clang_preprocessor_mode == .pch)
                try std.fmt.allocPrint(arena, "{s}.pch", .{root_name})
            else
                try std.zig.binNameAlloc(arena, .{
                    .root_name = root_name,
                    .target = target,
                    .output_mode = create_module.resolved_options.output_mode,
                    .link_mode = create_module.resolved_options.link_mode,
                    .version = optional_version,
                }),
        },
        .yes => |full_path| b: {
            const basename = fs.path.basename(full_path);
            if (fs.path.dirname(full_path)) |dirname| {
                const handle = fs.cwd().openDir(dirname, .{}) catch |err| {
                    fatal("unable to open output directory '{s}': {s}", .{ dirname, @errorName(err) });
                };
                cleanup_emit_bin_dir = handle;
                break :b Compilation.EmitLoc{
                    .basename = basename,
                    .directory = .{
                        .path = dirname,
                        .handle = handle,
                    },
                };
            } else {
                break :b Compilation.EmitLoc{
                    .basename = basename,
                    .directory = .{ .path = null, .handle = fs.cwd() },
                };
            }
        },
        .yes_a_out => Compilation.EmitLoc{
            .directory = .{ .path = null, .handle = fs.cwd() },
            .basename = a_out_basename,
        },
    };

    const default_h_basename = try std.fmt.allocPrint(arena, "{s}.h", .{root_name});
    var emit_h_resolved = emit_h.resolve(default_h_basename, output_to_cache) catch |err| {
        switch (emit_h) {
            .yes => |p| {
                fatal("unable to open directory from argument '-femit-h', '{s}': {s}", .{
                    p, @errorName(err),
                });
            },
            .yes_default_path => {
                fatal("unable to open directory from arguments '--name' or '-fsoname', '{s}': {s}", .{
                    default_h_basename, @errorName(err),
                });
            },
            .no => unreachable,
        }
    };
    defer emit_h_resolved.deinit();

    const default_asm_basename = try std.fmt.allocPrint(arena, "{s}.s", .{root_name});
    var emit_asm_resolved = emit_asm.resolve(default_asm_basename, output_to_cache) catch |err| {
        switch (emit_asm) {
            .yes => |p| {
                fatal("unable to open directory from argument '-femit-asm', '{s}': {s}", .{
                    p, @errorName(err),
                });
            },
            .yes_default_path => {
                fatal("unable to open directory from arguments '--name' or '-fsoname', '{s}': {s}", .{
                    default_asm_basename, @errorName(err),
                });
            },
            .no => unreachable,
        }
    };
    defer emit_asm_resolved.deinit();

    const default_llvm_ir_basename = try std.fmt.allocPrint(arena, "{s}.ll", .{root_name});
    var emit_llvm_ir_resolved = emit_llvm_ir.resolve(default_llvm_ir_basename, output_to_cache) catch |err| {
        switch (emit_llvm_ir) {
            .yes => |p| {
                fatal("unable to open directory from argument '-femit-llvm-ir', '{s}': {s}", .{
                    p, @errorName(err),
                });
            },
            .yes_default_path => {
                fatal("unable to open directory from arguments '--name' or '-fsoname', '{s}': {s}", .{
                    default_llvm_ir_basename, @errorName(err),
                });
            },
            .no => unreachable,
        }
    };
    defer emit_llvm_ir_resolved.deinit();

    const default_llvm_bc_basename = try std.fmt.allocPrint(arena, "{s}.bc", .{root_name});
    var emit_llvm_bc_resolved = emit_llvm_bc.resolve(default_llvm_bc_basename, output_to_cache) catch |err| {
        switch (emit_llvm_bc) {
            .yes => |p| {
                fatal("unable to open directory from argument '-femit-llvm-bc', '{s}': {s}", .{
                    p, @errorName(err),
                });
            },
            .yes_default_path => {
                fatal("unable to open directory from arguments '--name' or '-fsoname', '{s}': {s}", .{
                    default_llvm_bc_basename, @errorName(err),
                });
            },
            .no => unreachable,
        }
    };
    defer emit_llvm_bc_resolved.deinit();

    var emit_docs_resolved = emit_docs.resolve("docs", output_to_cache) catch |err| {
        switch (emit_docs) {
            .yes => |p| {
                fatal("unable to open directory from argument '-femit-docs', '{s}': {s}", .{
                    p, @errorName(err),
                });
            },
            .yes_default_path => {
                fatal("unable to open directory 'docs': {s}", .{@errorName(err)});
            },
            .no => unreachable,
        }
    };
    defer emit_docs_resolved.deinit();

    const is_exe_or_dyn_lib = switch (create_module.resolved_options.output_mode) {
        .Obj => false,
        .Lib => create_module.resolved_options.link_mode == .dynamic,
        .Exe => true,
    };
    // Note that cmake when targeting Windows will try to execute
    // zig cc to make an executable and output an implib too.
    const implib_eligible = is_exe_or_dyn_lib and
        emit_bin_loc != null and target.os.tag == .windows;
    if (!implib_eligible) {
        if (!emit_implib_arg_provided) {
            emit_implib = .no;
        } else if (emit_implib != .no) {
            fatal("the argument -femit-implib is allowed only when building a Windows DLL", .{});
        }
    }
    const default_implib_basename = try std.fmt.allocPrint(arena, "{s}.lib", .{root_name});
    var emit_implib_resolved = switch (emit_implib) {
        .no => Emit.Resolved{ .data = null, .dir = null },
        .yes => |p| emit_implib.resolve(default_implib_basename, output_to_cache) catch |err| {
            fatal("unable to open directory from argument '-femit-implib', '{s}': {s}", .{
                p, @errorName(err),
            });
        },
        .yes_default_path => Emit.Resolved{
            .data = Compilation.EmitLoc{
                .directory = emit_bin_loc.?.directory,
                .basename = default_implib_basename,
            },
            .dir = null,
        },
    };
    defer emit_implib_resolved.deinit();

    var thread_pool: ThreadPool = undefined;
    try thread_pool.init(.{
        .allocator = gpa,
        .n_jobs = @min(@max(n_jobs orelse std.Thread.getCpuCount() catch 1, 1), std.math.maxInt(Zcu.PerThread.IdBacking)),
        .track_ids = true,
        .stack_size = thread_stack_size,
    });
    defer thread_pool.deinit();

    var cleanup_local_cache_dir: ?fs.Dir = null;
    defer if (cleanup_local_cache_dir) |*dir| dir.close();

    var local_cache_directory: Directory = l: {
        if (override_local_cache_dir) |local_cache_dir_path| {
            const dir = try fs.cwd().makeOpenPath(local_cache_dir_path, .{});
            cleanup_local_cache_dir = dir;
            break :l .{
                .handle = dir,
                .path = local_cache_dir_path,
            };
        }
        if (arg_mode == .run) {
            break :l global_cache_directory;
        }

        // search upwards from cwd until we find directory with build.zig
        const cwd_path = try process.getCwdAlloc(arena);
        var dirname: []const u8 = cwd_path;
        while (true) {
            const joined_path = try fs.path.join(arena, &.{
                dirname, Package.build_zig_basename,
            });
            if (fs.cwd().access(joined_path, .{})) |_| {
                const cache_dir_path = try fs.path.join(arena, &.{ dirname, default_local_zig_cache_basename });
                const dir = try fs.cwd().makeOpenPath(cache_dir_path, .{});
                cleanup_local_cache_dir = dir;
                break :l .{ .handle = dir, .path = cache_dir_path };
            } else |err| switch (err) {
                error.FileNotFound => {
                    dirname = fs.path.dirname(dirname) orelse {
                        break :l global_cache_directory;
                    };
                    continue;
                },
                else => break :l global_cache_directory,
            }
        }

        // Otherwise we really don't have a reasonable place to put the local cache directory,
        // so we utilize the global one.
        break :l global_cache_directory;
    };

    for (create_module.c_source_files.items) |*src| {
        if (!mem.eql(u8, src.src_path, "-")) continue;

        const ext = src.ext orelse
            fatal("-E or -x is required when reading from a non-regular file", .{});

        // "-" is stdin. Dump it to a real file.
        const sep = fs.path.sep_str;
        const dump_path = try std.fmt.allocPrint(arena, "tmp" ++ sep ++ "{x}-dump-stdin{s}", .{
            std.crypto.random.int(u64), ext.canonicalName(target),
        });
        try local_cache_directory.handle.makePath("tmp");

        // Note that in one of the happy paths, execve() is used to switch to
        // clang in which case any cleanup logic that exists for this temporary
        // file will not run and this temp file will be leaked. The filename
        // will be a hash of its contents — so multiple invocations of
        // `zig cc -` will result in the same temp file name.
        var f = try local_cache_directory.handle.createFile(dump_path, .{});
        defer f.close();

        // Re-using the hasher from Cache, since the functional requirements
        // for the hashing algorithm here and in the cache are the same.
        // We are providing our own cache key, because this file has nothing
        // to do with the cache manifest.
        var hasher = Cache.Hasher.init("0123456789abcdef");
        var w = io.multiWriter(.{ f.writer(), hasher.writer() });
        var fifo = std.fifo.LinearFifo(u8, .{ .Static = 4096 }).init();
        try fifo.pump(io.getStdIn().reader(), w.writer());

        var bin_digest: Cache.BinDigest = undefined;
        hasher.final(&bin_digest);

        const sub_path = try std.fmt.allocPrint(arena, "tmp" ++ sep ++ "{s}-stdin{s}", .{
            std.fmt.fmtSliceHexLower(&bin_digest),
            ext.canonicalName(target),
        });
        try local_cache_directory.handle.rename(dump_path, sub_path);

        // Convert `sub_path` to be relative to current working directory.
        src.src_path = try local_cache_directory.join(arena, &.{sub_path});
    }

    if (build_options.have_llvm and emit_asm != .no) {
        // LLVM has no way to set this non-globally.
        const argv = [_][*:0]const u8{ "zig (LLVM option parsing)", "--x86-asm-syntax=intel" };
        @import("codegen/llvm/bindings.zig").ParseCommandLineOptions(argv.len, &argv);
    }

    const clang_passthrough_mode = switch (arg_mode) {
        .cc, .cpp, .translate_c => true,
        else => false,
    };

    const incremental = opt_incremental orelse false;

    const disable_lld_caching = !output_to_cache;

    const cache_mode: Compilation.CacheMode = b: {
        if (incremental) break :b .incremental;
        if (disable_lld_caching) break :b .incremental;
        if (!create_module.resolved_options.have_zcu) break :b .whole;

        // TODO: once we support incremental compilation for the LLVM backend
        // via saving the LLVM module into a bitcode file and restoring it,
        // along with compiler state, this clause can be removed so that
        // incremental cache mode is used for LLVM backend too.
        if (create_module.resolved_options.use_llvm) break :b .whole;

        // Eventually, this default should be `.incremental`. However, since incremental
        // compilation is currently an opt-in feature, it makes a strictly worse default cache mode
        // than `.whole`.
        // https://github.com/ziglang/zig/issues/21165
        break :b .whole;
    };

    process.raiseFileDescriptorLimit();

    var file_system_inputs: std.ArrayListUnmanaged(u8) = .empty;
    defer file_system_inputs.deinit(gpa);

    const comp = Compilation.create(gpa, arena, .{
        .zig_lib_directory = zig_lib_directory,
        .local_cache_directory = local_cache_directory,
        .global_cache_directory = global_cache_directory,
        .thread_pool = &thread_pool,
        .self_exe_path = self_exe_path,
        .config = create_module.resolved_options,
        .root_name = root_name,
        .sysroot = create_module.sysroot,
        .main_mod = main_mod,
        .root_mod = root_mod,
        .std_mod = std_mod,
        .emit_bin = emit_bin_loc,
        .emit_h = emit_h_resolved.data,
        .emit_asm = emit_asm_resolved.data,
        .emit_llvm_ir = emit_llvm_ir_resolved.data,
        .emit_llvm_bc = emit_llvm_bc_resolved.data,
        .emit_docs = emit_docs_resolved.data,
        .emit_implib = emit_implib_resolved.data,
        .lib_directories = create_module.lib_directories.items,
        .rpath_list = create_module.rpath_list.items,
        .symbol_wrap_set = symbol_wrap_set,
        .c_source_files = create_module.c_source_files.items,
        .rc_source_files = create_module.rc_source_files.items,
        .manifest_file = manifest_file,
        .rc_includes = rc_includes,
        .mingw_unicode_entry_point = mingw_unicode_entry_point,
        .link_inputs = create_module.link_inputs.items,
        .framework_dirs = create_module.framework_dirs.items,
        .frameworks = resolved_frameworks.items,
        .windows_lib_names = create_module.windows_libs.keys(),
        .wasi_emulated_libs = create_module.wasi_emulated_libs.items,
        .want_compiler_rt = want_compiler_rt,
        .want_ubsan_rt = want_ubsan_rt,
        .hash_style = hash_style,
        .linker_script = linker_script,
        .version_script = version_script,
        .linker_allow_undefined_version = linker_allow_undefined_version,
        .linker_enable_new_dtags = linker_enable_new_dtags,
        .disable_c_depfile = disable_c_depfile,
        .soname = resolved_soname,
        .linker_sort_section = linker_sort_section,
        .linker_gc_sections = linker_gc_sections,
        .linker_repro = linker_repro,
        .linker_allow_shlib_undefined = linker_allow_shlib_undefined,
        .linker_bind_global_refs_locally = linker_bind_global_refs_locally,
        .linker_import_symbols = linker_import_symbols,
        .linker_import_table = linker_import_table,
        .linker_export_table = linker_export_table,
        .linker_initial_memory = linker_initial_memory,
        .linker_max_memory = linker_max_memory,
        .linker_print_gc_sections = linker_print_gc_sections,
        .linker_print_icf_sections = linker_print_icf_sections,
        .linker_print_map = linker_print_map,
        .llvm_opt_bisect_limit = llvm_opt_bisect_limit,
        .linker_global_base = linker_global_base,
        .linker_export_symbol_names = linker_export_symbol_names.items,
        .linker_z_nocopyreloc = linker_z_nocopyreloc,
        .linker_z_nodelete = linker_z_nodelete,
        .linker_z_notext = linker_z_notext,
        .linker_z_defs = linker_z_defs,
        .linker_z_origin = linker_z_origin,
        .linker_z_now = linker_z_now,
        .linker_z_relro = linker_z_relro,
        .linker_z_common_page_size = linker_z_common_page_size,
        .linker_z_max_page_size = linker_z_max_page_size,
        .linker_tsaware = linker_tsaware,
        .linker_nxcompat = linker_nxcompat,
        .linker_dynamicbase = linker_dynamicbase,
        .linker_compress_debug_sections = linker_compress_debug_sections,
        .linker_module_definition_file = linker_module_definition_file,
        .major_subsystem_version = major_subsystem_version,
        .minor_subsystem_version = minor_subsystem_version,
        .link_eh_frame_hdr = link_eh_frame_hdr,
        .link_emit_relocs = link_emit_relocs,
        .entry = entry,
        .force_undefined_symbols = force_undefined_symbols,
        .stack_size = stack_size,
        .image_base = image_base,
        .function_sections = function_sections,
        .data_sections = data_sections,
        .clang_passthrough_mode = clang_passthrough_mode,
        .clang_preprocessor_mode = clang_preprocessor_mode,
        .version = optional_version,
        .compatibility_version = compatibility_version,
        .libc_installation = if (create_module.libc_installation) |*lci| lci else null,
        .verbose_cc = verbose_cc,
        .verbose_link = verbose_link,
        .verbose_air = verbose_air,
        .verbose_intern_pool = verbose_intern_pool,
        .verbose_generic_instances = verbose_generic_instances,
        .verbose_llvm_ir = verbose_llvm_ir,
        .verbose_llvm_bc = verbose_llvm_bc,
        .verbose_cimport = verbose_cimport,
        .verbose_llvm_cpu_features = verbose_llvm_cpu_features,
        .time_report = time_report,
        .stack_report = stack_report,
        .build_id = build_id,
        .test_filters = test_filters.items,
        .test_name_prefix = test_name_prefix,
        .test_runner_path = test_runner_path,
        .disable_lld_caching = disable_lld_caching,
        .cache_mode = cache_mode,
        .subsystem = subsystem,
        .debug_compile_errors = debug_compile_errors,
        .incremental = incremental,
        .enable_link_snapshots = enable_link_snapshots,
        .install_name = install_name,
        .entitlements = entitlements,
        .pagezero_size = pagezero_size,
        .headerpad_size = headerpad_size,
        .headerpad_max_install_names = headerpad_max_install_names,
        .dead_strip_dylibs = dead_strip_dylibs,
        .force_load_objc = force_load_objc,
        .discard_local_symbols = discard_local_symbols,
        .reference_trace = reference_trace,
        .pdb_out_path = pdb_out_path,
        .error_limit = error_limit,
        .native_system_include_paths = create_module.native_system_include_paths,
        // Any leftover C compilation args (such as -I) apply globally rather
        // than to any particular module. This feature can greatly reduce CLI
        // noise when --search-prefix and -M are combined.
        .global_cc_argv = try cc_argv.toOwnedSlice(arena),
        .file_system_inputs = &file_system_inputs,
        .debug_compiler_runtime_libs = debug_compiler_runtime_libs,
    }) catch |err| switch (err) {
        error.LibCUnavailable => {
            const triple_name = try target.zigTriple(arena);
            std.log.err("unable to find or provide libc for target '{s}'", .{triple_name});

            for (std.zig.target.available_libcs) |t| {
                if (t.arch == target.cpu.arch and t.os == target.os.tag) {
                    // If there's a `glibc_min`, there's also an `os_ver`.
                    if (t.glibc_min) |glibc_min| {
                        std.log.info("zig can provide libc for related target {s}-{s}.{}-{s}.{d}.{d}", .{
                            @tagName(t.arch),
                            @tagName(t.os),
                            t.os_ver.?,
                            @tagName(t.abi),
                            glibc_min.major,
                            glibc_min.minor,
                        });
                    } else if (t.os_ver) |os_ver| {
                        std.log.info("zig can provide libc for related target {s}-{s}.{}-{s}", .{
                            @tagName(t.arch),
                            @tagName(t.os),
                            os_ver,
                            @tagName(t.abi),
                        });
                    } else {
                        std.log.info("zig can provide libc for related target {s}-{s}-{s}", .{
                            @tagName(t.arch),
                            @tagName(t.os),
                            @tagName(t.abi),
                        });
                    }
                }
            }
            process.exit(1);
        },
        error.ExportTableAndImportTableConflict => {
            fatal("--import-table and --export-table may not be used together", .{});
        },
        else => fatal("unable to create compilation: {s}", .{@errorName(err)}),
    };
    var comp_destroyed = false;
    defer if (!comp_destroyed) comp.destroy();

    if (show_builtin) {
        const builtin_mod = comp.root_mod.getBuiltinDependency();
        const source = builtin_mod.builtin_file.?.source.?;
        return std.io.getStdOut().writeAll(source);
    }
    switch (listen) {
        .none => {},
        .stdio => {
            try serve(
                comp,
                std.io.getStdIn(),
                std.io.getStdOut(),
                test_exec_args.items,
                self_exe_path,
                arg_mode,
                all_args,
                runtime_args_start,
            );
            return cleanExit();
        },
        .ip4 => |ip4_addr| {
            const addr: std.net.Address = .{ .in = ip4_addr };

            var server = try addr.listen(.{
                .reuse_address = true,
            });
            defer server.deinit();

            const conn = try server.accept();
            defer conn.stream.close();

            try serve(
                comp,
                .{ .handle = conn.stream.handle },
                .{ .handle = conn.stream.handle },
                test_exec_args.items,
                self_exe_path,
                arg_mode,
                all_args,
                runtime_args_start,
            );
            return cleanExit();
        },
    }

    {
        const root_prog_node = std.Progress.start(.{
            .disable_printing = (color == .off),
        });
        defer root_prog_node.end();

        if (arg_mode == .translate_c) {
            return cmdTranslateC(comp, arena, null, null, root_prog_node);
        }

        updateModule(comp, color, root_prog_node) catch |err| switch (err) {
            error.SemanticAnalyzeFail => {
                assert(listen == .none);
                saveState(comp, incremental);
                process.exit(1);
            },
            else => |e| return e,
        };
    }
    try comp.makeBinFileExecutable();
    saveState(comp, incremental);

    if (switch (arg_mode) {
        .run => true,
        .zig_test => !test_no_exec,
        else => false,
    }) {
        dev.checkAny(&.{ .run_command, .test_command });

        if (test_exec_args.items.len == 0 and target.ofmt == .c) default_exec_args: {
            // Default to using `zig run` to execute the produced .c code from `zig test`.
            const c_code_loc = emit_bin_loc orelse break :default_exec_args;
            const c_code_directory = c_code_loc.directory orelse comp.bin_file.?.emit.root_dir;
            const c_code_path = try fs.path.join(arena, &[_][]const u8{
                c_code_directory.path orelse ".", c_code_loc.basename,
            });
            try test_exec_args.appendSlice(arena, &.{ self_exe_path, "run" });
            if (zig_lib_directory.path) |p| {
                try test_exec_args.appendSlice(arena, &.{ "-I", p });
            }

            if (create_module.resolved_options.link_libc) {
                try test_exec_args.append(arena, "-lc");
            } else if (target.os.tag == .windows) {
                try test_exec_args.appendSlice(arena, &.{
                    "--subsystem", "console",
                    "-lkernel32",  "-lntdll",
                });
            }

            const first_cli_mod = create_module.modules.values()[0];
            if (first_cli_mod.target_arch_os_abi) |triple| {
                try test_exec_args.appendSlice(arena, &.{ "-target", triple });
            }
            if (first_cli_mod.target_mcpu) |mcpu| {
                try test_exec_args.append(arena, try std.fmt.allocPrint(arena, "-mcpu={s}", .{mcpu}));
            }
            if (create_module.dynamic_linker) |dl| {
                try test_exec_args.appendSlice(arena, &.{ "--dynamic-linker", dl });
            }
            try test_exec_args.append(arena, c_code_path);
        }

        try runOrTest(
            comp,
            gpa,
            arena,
            test_exec_args.items,
            self_exe_path.?,
            arg_mode,
            &target,
            &comp_destroyed,
            all_args,
            runtime_args_start,
            create_module.resolved_options.link_libc,
        );
    }

    // Skip resource deallocation in release builds; let the OS do it.
    return cleanExit();
}
```
好的，我现在要总结这段代码的思维模型。首先，我需要仔细阅读代码，理解它的结构和功能。看起来这是一个用Zig语言编写的函数，名为`buildOutputType`。函数接受多个参数，包括分配器、参数列表、参数模式等，并返回一个可能的错误或`void`。

首先，函数内部声明了很多变量，这些变量似乎用于处理不同的编译选项和配置，比如目标架构、优化选项、链接器设置等等。接着，代码根据不同的`arg_mode`（参数模式）进入不同的处理分支，比如`build`、`translate_c`、`zig_test`、`run`等模式。每个模式下，代码会解析命令行参数，设置相应的编译选项。

在解析参数的过程中，代码处理了各种标志，例如`-target`、`-mcpu`、`-O`等，并根据这些参数设置模块的选项。此外，还处理了库的链接、框架路径、预处理器指令等。代码中还涉及到模块的创建、缓存的配置、线程池的初始化等。

在代码的后半部分，处理了输出文件的生成，包括二进制文件、汇编文件、LLVM IR等，并根据不同的目标平台（如Windows、Linux等）设置相应的选项。最后，根据不同的参数模式（如运行测试或执行程序），调用相应的函数来执行生成的二进制文件。

总结来说，这段代码的主要功能是解析命令行参数，配置编译选项，创建模块，处理依赖关系，生成目标文件，并根据需要执行生成的程序或测试。整个流程涉及到多个步骤，包括参数解析、模块管理、编译配置、资源分配和输出处理。需要理解Zig的编译流程、模块系统以及如何处理不同的编译目标和选项。
================================================
这段代码是Zig编译器前端的一个核心函数，负责构建输出类型（如可执行文件、库等）。其思维模型可分为以下几个关键部分：

### 1. **参数解析与初始化**
- **变量声明**: 声明了大量配置变量（如目标架构、优化级别、链接选项、调试标志等），用于存储从命令行解析出的配置信息。
- **参数迭代**: 使用`ArgsIterator`遍历命令行参数，处理`-M`（模块）、`-target`（目标三元组）、`-O`（优化级别）等关键参数，并根据不同模式（`build`/`run`/`test`等）调整行为。
- **环境变量处理**: 读取`NO_COLOR`、`CLICOLOR_FORCE`等环境变量，设置输出颜色模式。

---

### 2. **模块与依赖管理**
- **模块创建**: 通过`create_module`结构管理模块配置，包括目标格式、动态链接器、库路径等。支持多模块依赖（通过`-M`标志）。
- **依赖处理**: 解析`--dep`参数，管理模块间的依赖关系，确保模块路径和名称无冲突。
- **C/RC文件处理**: 收集C和资源文件（`.c`/`.rc`），并关联到对应模块。

---

### 3. **编译目标与后端配置**
- **目标解析**: 根据`-target`和`-mcpu`确定目标架构和CPU特性，兼容Clang风格的参数。
- **输出控制**: 配置生成的文件类型（二进制、汇编、LLVM IR/BC、文档等），处理路径和缓存目录（全局/本地缓存）。
- **链接器选项**: 设置ELF/COFF格式的链接器参数（如`--gc-sections`、`-z now`）、动态库版本（`soname`）、Windows子系统版本等。

---

### 4. **编译流程与优化**
- **优化模式**: 解析`-O`参数，设置调试/发布模式，控制符号剥离、栈保护、LTO等。
- **多线程处理**: 初始化线程池，利用并行加速编译（如C文件编译）。
- **缓存机制**: 支持增量编译（`incremental`模式）和全量缓存（`whole`模式），依赖文件哈希和全局缓存。

---

### 5. **运行时与执行**
- **测试/运行模式**: 处理`zig test`和`zig run`，生成测试运行器或直接执行二进制文件。
- **跨平台支持**: 处理Windows PE选项（如`.manifest`文件）、WASI执行模型、MinGW入口点等。
- **调试支持**: 提供详细的编译错误跟踪（`--debug-compile-errors`）、LLVM IR/BC输出（`--verbose-llvm-ir`）等。

---

### 6. **错误处理与资源管理**
- **错误处理**: 对非法参数（如冲突的链接选项）触发`fatal`错误，确保配置一致性。
- **资源清理**: 使用`defer`确保文件描述符、缓存目录等资源的释放，避免泄漏。

---

### 核心逻辑流程图
```
参数解析 → 模块初始化 → 目标配置 → 编译/链接 → 输出生成 → 执行/测试
```
- **关键路径**: 命令行参数驱动模块和目标的配置，编译后端（LLVM/自研）生成代码，链接器整合对象文件，最终生成可执行文件或库。
- **扩展性**: 通过`clang_passthrough_mode`支持Clang兼容参数，便于迁移现有构建系统。

---

### 总结
这段代码是Zig编译器的前端核心，负责将用户输入（参数/文件）转换为编译任务，协调模块、目标配置、优化、链接等步骤，最终生成可执行输出。其设计强调灵活性和跨平台支持，同时通过缓存和并行化提升性能。