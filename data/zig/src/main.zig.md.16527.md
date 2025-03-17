```zig
fn cmdBuild(gpa: Allocator, arena: Allocator, args: []const []const u8) !void {
    dev.check(.build_command);

    var build_file: ?[]const u8 = null;
    var override_lib_dir: ?[]const u8 = try EnvVar.ZIG_LIB_DIR.get(arena);
    var override_global_cache_dir: ?[]const u8 = try EnvVar.ZIG_GLOBAL_CACHE_DIR.get(arena);
    var override_local_cache_dir: ?[]const u8 = try EnvVar.ZIG_LOCAL_CACHE_DIR.get(arena);
    var override_build_runner: ?[]const u8 = try EnvVar.ZIG_BUILD_RUNNER.get(arena);
    var child_argv = std.ArrayList([]const u8).init(arena);
    var reference_trace: ?u32 = null;
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
    var fetch_only = false;
    var system_pkg_dir_path: ?[]const u8 = null;
    var debug_target: ?[]const u8 = null;

    const argv_index_exe = child_argv.items.len;
    _ = try child_argv.addOne();

    const self_exe_path = try introspect.findZigExePath(arena);
    try child_argv.append(self_exe_path);

    const argv_index_zig_lib_dir = child_argv.items.len;
    _ = try child_argv.addOne();

    const argv_index_build_file = child_argv.items.len;
    _ = try child_argv.addOne();

    const argv_index_cache_dir = child_argv.items.len;
    _ = try child_argv.addOne();

    const argv_index_global_cache_dir = child_argv.items.len;
    _ = try child_argv.addOne();

    try child_argv.appendSlice(&.{
        "--seed",
        try std.fmt.allocPrint(arena, "0x{x}", .{std.crypto.random.int(u32)}),
    });
    const argv_index_seed = child_argv.items.len - 1;

    // This parent process needs a way to obtain results from the configuration
    // phase of the child process. In the future, the make phase will be
    // executed in a separate process than the configure phase, and we can then
    // use stdout from the configuration phase for this purpose.
    //
    // However, currently, both phases are in the same process, and Run Step
    // provides API for making the runned subprocesses inherit stdout and stderr
    // which means these streams are not available for passing metadata back
    // to the parent.
    //
    // Until make and configure phases are separated into different processes,
    // the strategy is to choose a temporary file name ahead of time, and then
    // read this file in the parent to obtain the results, in the case the child
    // exits with code 3.
    const results_tmp_file_nonce = std.fmt.hex(std.crypto.random.int(u64));
    try child_argv.append("-Z" ++ results_tmp_file_nonce);

    var color: Color = .auto;
    var n_jobs: ?u32 = null;

    {
        var i: usize = 0;
        while (i < args.len) : (i += 1) {
            const arg = args[i];
            if (mem.startsWith(u8, arg, "-")) {
                if (mem.eql(u8, arg, "--build-file")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    build_file = args[i];
                    continue;
                } else if (mem.eql(u8, arg, "--zig-lib-dir")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    override_lib_dir = args[i];
                    continue;
                } else if (mem.eql(u8, arg, "--build-runner")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    override_build_runner = args[i];
                    continue;
                } else if (mem.eql(u8, arg, "--cache-dir")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    override_local_cache_dir = args[i];
                    continue;
                } else if (mem.eql(u8, arg, "--global-cache-dir")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    override_global_cache_dir = args[i];
                    continue;
                } else if (mem.eql(u8, arg, "-freference-trace")) {
                    reference_trace = 256;
                } else if (mem.eql(u8, arg, "--fetch")) {
                    fetch_only = true;
                } else if (mem.eql(u8, arg, "--system")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    system_pkg_dir_path = args[i];
                    try child_argv.append("--system");
                    continue;
                } else if (mem.startsWith(u8, arg, "-freference-trace=")) {
                    const num = arg["-freference-trace=".len..];
                    reference_trace = std.fmt.parseUnsigned(u32, num, 10) catch |err| {
                        fatal("unable to parse reference_trace count '{s}': {s}", .{ num, @errorName(err) });
                    };
                } else if (mem.eql(u8, arg, "-fno-reference-trace")) {
                    reference_trace = null;
                } else if (mem.eql(u8, arg, "--debug-log")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    try child_argv.appendSlice(args[i .. i + 2]);
                    i += 1;
                    if (!build_options.enable_logging) {
                        warn("Zig was compiled without logging enabled (-Dlog). --debug-log has no effect.", .{});
                    } else {
                        try log_scopes.append(arena, args[i]);
                    }
                    continue;
                } else if (mem.eql(u8, arg, "--debug-compile-errors")) {
                    if (build_options.enable_debug_extensions) {
                        debug_compile_errors = true;
                    } else {
                        warn("Zig was compiled without debug extensions. --debug-compile-errors has no effect.", .{});
                    }
                } else if (mem.eql(u8, arg, "--debug-target")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    if (build_options.enable_debug_extensions) {
                        debug_target = args[i];
                    } else {
                        warn("Zig was compiled without debug extensions. --debug-target has no effect.", .{});
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
                } else if (mem.eql(u8, arg, "--color")) {
                    if (i + 1 >= args.len) fatal("expected [auto|on|off] after {s}", .{arg});
                    i += 1;
                    color = std.meta.stringToEnum(Color, args[i]) orelse {
                        fatal("expected [auto|on|off] after {s}, found '{s}'", .{ arg, args[i] });
                    };
                    try child_argv.appendSlice(&.{ arg, args[i] });
                    continue;
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
                } else if (mem.eql(u8, arg, "--seed")) {
                    if (i + 1 >= args.len) fatal("expected argument after '{s}'", .{arg});
                    i += 1;
                    child_argv.items[argv_index_seed] = args[i];
                    continue;
                } else if (mem.eql(u8, arg, "--")) {
                    // The rest of the args are supposed to get passed onto
                    // build runner's `build.args`
                    try child_argv.appendSlice(args[i..]);
                    break;
                }
            }
            try child_argv.append(arg);
        }
    }

    const work_around_btrfs_bug = native_os == .linux and
        EnvVar.ZIG_BTRFS_WORKAROUND.isSet();
    const root_prog_node = std.Progress.start(.{
        .disable_printing = (color == .off),
        .root_name = "Compile Build Script",
    });
    defer root_prog_node.end();

    // Normally the build runner is compiled for the host target but here is
    // some code to help when debugging edits to the build runner so that you
    // can make sure it compiles successfully on other targets.
    const resolved_target: Package.Module.ResolvedTarget = t: {
        if (build_options.enable_debug_extensions) {
            if (debug_target) |triple| {
                const target_query = try std.Target.Query.parse(.{
                    .arch_os_abi = triple,
                });
                break :t .{
                    .result = std.zig.resolveTargetQueryOrFatal(target_query),
                    .is_native_os = false,
                    .is_native_abi = false,
                };
            }
        }
        break :t .{
            .result = std.zig.resolveTargetQueryOrFatal(.{}),
            .is_native_os = true,
            .is_native_abi = true,
        };
    };

    const exe_basename = try std.zig.binNameAlloc(arena, .{
        .root_name = "build",
        .target = resolved_target.result,
        .output_mode = .Exe,
    });
    const emit_bin: Compilation.EmitLoc = .{
        .directory = null, // Use the local zig-cache.
        .basename = exe_basename,
    };

    process.raiseFileDescriptorLimit();

    var zig_lib_directory: Directory = if (override_lib_dir) |lib_dir| .{
        .path = lib_dir,
        .handle = fs.cwd().openDir(lib_dir, .{}) catch |err| {
            fatal("unable to open zig lib directory from 'zig-lib-dir' argument: '{s}': {s}", .{ lib_dir, @errorName(err) });
        },
    } else introspect.findZigLibDirFromSelfExe(arena, self_exe_path) catch |err| {
        fatal("unable to find zig installation directory '{s}': {s}", .{ self_exe_path, @errorName(err) });
    };
    defer zig_lib_directory.handle.close();

    const cwd_path = try process.getCwdAlloc(arena);
    child_argv.items[argv_index_zig_lib_dir] = zig_lib_directory.path orelse cwd_path;

    const build_root = try findBuildRoot(arena, .{
        .cwd_path = cwd_path,
        .build_file = build_file,
    });
    child_argv.items[argv_index_build_file] = build_root.directory.path orelse cwd_path;

    var global_cache_directory: Directory = l: {
        const p = override_global_cache_dir orelse try introspect.resolveGlobalCacheDir(arena);
        const dir = fs.cwd().makeOpenPath(p, .{}) catch |err| {
            const base_msg = "unable to open or create the global Zig cache at '{s}': {s}.{s}";
            const extra = "\nIf this location is not writable then consider specifying an " ++
                "alternative with the ZIG_GLOBAL_CACHE_DIR environment variable or the " ++
                "--global-cache-dir option.";
            const show_extra = err == error.AccessDenied or err == error.ReadOnlyFileSystem;
            fatal(base_msg, .{ p, @errorName(err), if (show_extra) extra else "" });
        };
        break :l .{
            .handle = dir,
            .path = p,
        };
    };
    defer global_cache_directory.handle.close();

    child_argv.items[argv_index_global_cache_dir] = global_cache_directory.path orelse cwd_path;

    var local_cache_directory: Directory = l: {
        if (override_local_cache_dir) |local_cache_dir_path| {
            break :l .{
                .handle = try fs.cwd().makeOpenPath(local_cache_dir_path, .{}),
                .path = local_cache_dir_path,
            };
        }
        const cache_dir_path = try build_root.directory.join(arena, &.{default_local_zig_cache_basename});
        break :l .{
            .handle = try build_root.directory.handle.makeOpenPath(default_local_zig_cache_basename, .{}),
            .path = cache_dir_path,
        };
    };
    defer local_cache_directory.handle.close();

    child_argv.items[argv_index_cache_dir] = local_cache_directory.path orelse cwd_path;

    var thread_pool: ThreadPool = undefined;
    try thread_pool.init(.{
        .allocator = gpa,
        .n_jobs = @min(@max(n_jobs orelse std.Thread.getCpuCount() catch 1, 1), std.math.maxInt(Zcu.PerThread.IdBacking)),
        .track_ids = true,
        .stack_size = thread_stack_size,
    });
    defer thread_pool.deinit();

    // Dummy http client that is not actually used when fetch_command is unsupported.
    // Prevents bootstrap from depending on a bunch of unnecessary stuff.
    var http_client: if (dev.env.supports(.fetch_command)) std.http.Client else struct {
        allocator: Allocator,
        fn deinit(_: @This()) void {}
    } = .{ .allocator = gpa };
    defer http_client.deinit();

    var unlazy_set: Package.Fetch.JobQueue.UnlazySet = .{};

    // This loop is re-evaluated when the build script exits with an indication that it
    // could not continue due to missing lazy dependencies.
    while (true) {
        // We want to release all the locks before executing the child process, so we make a nice
        // big block here to ensure the cleanup gets run when we extract out our argv.
        {
            const main_mod_paths: Package.Module.CreateOptions.Paths = if (override_build_runner) |runner| .{
                .root = .{
                    .root_dir = Cache.Directory.cwd(),
                    .sub_path = fs.path.dirname(runner) orelse "",
                },
                .root_src_path = fs.path.basename(runner),
            } else .{
                .root = .{
                    .root_dir = zig_lib_directory,
                    .sub_path = "compiler",
                },
                .root_src_path = "build_runner.zig",
            };

            const config = try Compilation.Config.resolve(.{
                .output_mode = .Exe,
                .resolved_target = resolved_target,
                .have_zcu = true,
                .emit_bin = true,
                .is_test = false,
            });

            const root_mod = try Package.Module.create(arena, .{
                .global_cache_directory = global_cache_directory,
                .paths = main_mod_paths,
                .fully_qualified_name = "root",
                .cc_argv = &.{},
                .inherited = .{
                    .resolved_target = resolved_target,
                },
                .global = config,
                .parent = null,
                .builtin_mod = null,
                .builtin_modules = null, // all modules will inherit this one's builtin
            });

            const builtin_mod = root_mod.getBuiltinDependency();

            const build_mod = try Package.Module.create(arena, .{
                .global_cache_directory = global_cache_directory,
                .paths = .{
                    .root = .{ .root_dir = build_root.directory },
                    .root_src_path = build_root.build_zig_basename,
                },
                .fully_qualified_name = "root.@build",
                .cc_argv = &.{},
                .inherited = .{},
                .global = config,
                .parent = root_mod,
                .builtin_mod = builtin_mod,
                .builtin_modules = null, // `builtin_mod` is specified
            });

            var cleanup_build_dir: ?fs.Dir = null;
            defer if (cleanup_build_dir) |*dir| dir.close();

            if (dev.env.supports(.fetch_command)) {
                const fetch_prog_node = root_prog_node.start("Fetch Packages", 0);
                defer fetch_prog_node.end();

                var job_queue: Package.Fetch.JobQueue = .{
                    .http_client = &http_client,
                    .thread_pool = &thread_pool,
                    .global_cache = global_cache_directory,
                    .read_only = false,
                    .recursive = true,
                    .debug_hash = false,
                    .work_around_btrfs_bug = work_around_btrfs_bug,
                    .unlazy_set = unlazy_set,
                };
                defer job_queue.deinit();

                if (system_pkg_dir_path) |p| {
                    job_queue.global_cache = .{
                        .path = p,
                        .handle = fs.cwd().openDir(p, .{}) catch |err| {
                            fatal("unable to open system package directory '{s}': {s}", .{
                                p, @errorName(err),
                            });
                        },
                    };
                    job_queue.read_only = true;
                    cleanup_build_dir = job_queue.global_cache.handle;
                } else {
                    try http_client.initDefaultProxies(arena);
                }

                try job_queue.all_fetches.ensureUnusedCapacity(gpa, 1);
                try job_queue.table.ensureUnusedCapacity(gpa, 1);

                var fetch: Package.Fetch = .{
                    .arena = std.heap.ArenaAllocator.init(gpa),
                    .location = .{ .relative_path = build_mod.root },
                    .location_tok = 0,
                    .hash_tok = .none,
                    .name_tok = 0,
                    .lazy_status = .eager,
                    .parent_package_root = build_mod.root,
                    .parent_manifest_ast = null,
                    .prog_node = fetch_prog_node,
                    .job_queue = &job_queue,
                    .omit_missing_hash_error = true,
                    .allow_missing_paths_field = false,
                    .allow_missing_fingerprint = false,
                    .allow_name_string = false,
                    .use_latest_commit = false,

                    .package_root = undefined,
                    .error_bundle = undefined,
                    .manifest = null,
                    .manifest_ast = undefined,
                    .computed_hash = undefined,
                    .has_build_zig = true,
                    .oom_flag = false,
                    .latest_commit = null,

                    .module = build_mod,
                };
                job_queue.all_fetches.appendAssumeCapacity(&fetch);

                job_queue.table.putAssumeCapacityNoClobber(
                    Package.Fetch.relativePathDigest(build_mod.root, global_cache_directory),
                    &fetch,
                );

                job_queue.thread_pool.spawnWg(&job_queue.wait_group, Package.Fetch.workerRun, .{
                    &fetch, "root",
                });
                job_queue.wait_group.wait();

                try job_queue.consolidateErrors();

                if (fetch.error_bundle.root_list.items.len > 0) {
                    var errors = try fetch.error_bundle.toOwnedBundle("");
                    errors.renderToStdErr(color.renderOptions());
                    process.exit(1);
                }

                if (fetch_only) return cleanExit();

                var source_buf = std.ArrayList(u8).init(gpa);
                defer source_buf.deinit();
                try job_queue.createDependenciesSource(&source_buf);
                const deps_mod = try createDependenciesModule(
                    arena,
                    source_buf.items,
                    root_mod,
                    global_cache_directory,
                    local_cache_directory,
                    builtin_mod,
                    config,
                );

                {
                    // We need a Module for each package's build.zig.
                    const hashes = job_queue.table.keys();
                    const fetches = job_queue.table.values();
                    try deps_mod.deps.ensureUnusedCapacity(arena, @intCast(hashes.len));
                    for (hashes, fetches) |*hash, f| {
                        if (f == &fetch) {
                            // The first one is a dummy package for the current project.
                            continue;
                        }
                        if (!f.has_build_zig)
                            continue;
                        const hash_slice = hash.toSlice();
                        const m = try Package.Module.create(arena, .{
                            .global_cache_directory = global_cache_directory,
                            .paths = .{
                                .root = try f.package_root.clone(arena),
                                .root_src_path = Package.build_zig_basename,
                            },
                            .fully_qualified_name = try std.fmt.allocPrint(
                                arena,
                                "root.@dependencies.{s}",
                                .{hash_slice},
                            ),
                            .cc_argv = &.{},
                            .inherited = .{},
                            .global = config,
                            .parent = root_mod,
                            .builtin_mod = builtin_mod,
                            .builtin_modules = null, // `builtin_mod` is specified
                        });
                        const hash_cloned = try arena.dupe(u8, hash_slice);
                        deps_mod.deps.putAssumeCapacityNoClobber(hash_cloned, m);
                        f.module = m;
                    }

                    // Each build.zig module needs access to each of its
                    // dependencies' build.zig modules by name.
                    for (fetches) |f| {
                        const mod = f.module orelse continue;
                        const man = f.manifest orelse continue;
                        const dep_names = man.dependencies.keys();
                        try mod.deps.ensureUnusedCapacity(arena, @intCast(dep_names.len));
                        for (dep_names, man.dependencies.values()) |name, dep| {
                            const dep_digest = Package.Fetch.depDigest(
                                f.package_root,
                                global_cache_directory,
                                dep,
                            ) orelse continue;
                            const dep_mod = job_queue.table.get(dep_digest).?.module orelse continue;
                            const name_cloned = try arena.dupe(u8, name);
                            mod.deps.putAssumeCapacityNoClobber(name_cloned, dep_mod);
                        }
                    }
                }
            } else try createEmptyDependenciesModule(
                arena,
                root_mod,
                global_cache_directory,
                local_cache_directory,
                builtin_mod,
                config,
            );

            try root_mod.deps.put(arena, "@build", build_mod);

            const comp = Compilation.create(gpa, arena, .{
                .zig_lib_directory = zig_lib_directory,
                .local_cache_directory = local_cache_directory,
                .global_cache_directory = global_cache_directory,
                .root_name = "build",
                .config = config,
                .root_mod = root_mod,
                .main_mod = build_mod,
                .emit_bin = emit_bin,
                .emit_h = null,
                .self_exe_path = self_exe_path,
                .thread_pool = &thread_pool,
                .verbose_cc = verbose_cc,
                .verbose_link = verbose_link,
                .verbose_air = verbose_air,
                .verbose_intern_pool = verbose_intern_pool,
                .verbose_generic_instances = verbose_generic_instances,
                .verbose_llvm_ir = verbose_llvm_ir,
                .verbose_llvm_bc = verbose_llvm_bc,
                .verbose_cimport = verbose_cimport,
                .verbose_llvm_cpu_features = verbose_llvm_cpu_features,
                .cache_mode = .whole,
                .reference_trace = reference_trace,
                .debug_compile_errors = debug_compile_errors,
            }) catch |err| {
                fatal("unable to create compilation: {s}", .{@errorName(err)});
            };
            defer comp.destroy();

            updateModule(comp, color, root_prog_node) catch |err| switch (err) {
                error.SemanticAnalyzeFail => process.exit(2),
                else => |e| return e,
            };

            // Since incremental compilation isn't done yet, we use cache_mode = whole
            // above, and thus the output file is already closed.
            //try comp.makeBinFileExecutable();
            child_argv.items[argv_index_exe] =
                try local_cache_directory.join(arena, &.{comp.cache_use.whole.bin_sub_path.?});
        }

        if (process.can_spawn) {
            var child = std.process.Child.init(child_argv.items, gpa);
            child.stdin_behavior = .Inherit;
            child.stdout_behavior = .Inherit;
            child.stderr_behavior = .Inherit;

            const term = t: {
                std.debug.lockStdErr();
                defer std.debug.unlockStdErr();
                break :t child.spawnAndWait() catch |err| {
                    fatal("failed to spawn build runner {s}: {s}", .{ child_argv.items[0], @errorName(err) });
                };
            };

            switch (term) {
                .Exited => |code| {
                    if (code == 0) return cleanExit();
                    // Indicates that the build runner has reported compile errors
                    // and this parent process does not need to report any further
                    // diagnostics.
                    if (code == 2) process.exit(2);

                    if (code == 3) {
                        if (!dev.env.supports(.fetch_command)) process.exit(3);
                        // Indicates the configure phase failed due to missing lazy
                        // dependencies and stdout contains the hashes of the ones
                        // that are missing.
                        const s = fs.path.sep_str;
                        const tmp_sub_path = "tmp" ++ s ++ results_tmp_file_nonce;
                        const stdout = local_cache_directory.handle.readFileAlloc(arena, tmp_sub_path, 50 * 1024 * 1024) catch |err| {
                            fatal("unable to read results of configure phase from '{}{s}': {s}", .{
                                local_cache_directory, tmp_sub_path, @errorName(err),
                            });
                        };
                        local_cache_directory.handle.deleteFile(tmp_sub_path) catch {};

                        var it = mem.splitScalar(u8, stdout, '\n');
                        var any_errors = false;
                        while (it.next()) |hash| {
                            if (hash.len == 0) continue;
                            if (hash.len > Package.Hash.max_len) {
                                std.log.err("invalid digest (length {d} exceeds maximum): '{s}'", .{
                                    hash.len, hash,
                                });
                                any_errors = true;
                                continue;
                            }
                            try unlazy_set.put(arena, .fromSlice(hash), {});
                        }
                        if (any_errors) process.exit(3);
                        if (system_pkg_dir_path) |p| {
                            // In this mode, the system needs to provide these packages; they
                            // cannot be fetched by Zig.
                            for (unlazy_set.keys()) |*hash| {
                                std.log.err("lazy dependency package not found: {s}" ++ s ++ "{s}", .{
                                    p, hash.toSlice(),
                                });
                            }
                            std.log.info("remote package fetching disabled due to --system mode", .{});
                            std.log.info("dependencies might be avoidable depending on build configuration", .{});
                            process.exit(3);
                        }
                        continue;
                    }

                    const cmd = try std.mem.join(arena, " ", child_argv.items);
                    fatal("the following build command failed with exit code {d}:\n{s}", .{ code, cmd });
                },
                else => {
                    const cmd = try std.mem.join(arena, " ", child_argv.items);
                    fatal("the following build command crashed:\n{s}", .{cmd});
                },
            }
        } else {
            const cmd = try std.mem.join(arena, " ", child_argv.items);
            fatal("the following command cannot be executed ({s} does not support spawning a child process):\n{s}", .{ @tagName(native_os), cmd });
        }
    }
}
```
嗯，我现在要总结这段Zig代码的思维模型。首先，我需要理解这段代码的作用。看起来这是一个处理构建命令的函数，可能属于Zig编译器或构建系统的一部分。函数名是cmdBuild，接受分配器gpa和arena，以及命令行参数args。

首先，我注意到代码中有很多变量初始化，比如build_file、override_lib_dir等，这些可能都是配置选项。然后，child_argv被初始化为一个ArrayList，用于构建子进程的命令行参数。接着，通过introspect.findZigExePath获取当前Zig可执行文件的路径，并将其添加到child_argv中。这表明这个函数可能是在准备调用另一个Zig进程来执行构建任务。

接下来，有很多处理命令行参数的逻辑，比如处理--build-file、--zig-lib-dir等选项。这里有很多条件判断，根据不同的参数设置不同的变量，比如reference_trace、debug_compile_errors等。这些选项会影响构建过程的详细输出或其他行为。

然后，代码处理构建目标（target），包括解析目标三元组（triple），这可能用于交叉编译。之后，生成构建脚本的可执行文件名，并设置输出位置。这里涉及到了Zig的缓存目录，包括全局缓存和本地缓存，这可能用于管理构建过程中的临时文件和缓存数据。

在处理依赖项的部分，代码使用了线程池（thread_pool）来处理并发任务，比如获取HTTP客户端（http_client）来获取远程依赖。这部分可能涉及到Zig的包管理功能，从远程仓库获取依赖项。如果支持fetch_command，则会进行依赖项的获取和解析，否则创建一个空的依赖模块。

接着，创建编译（Compilation）实例，配置各种编译选项，如verbose_cc、verbose_link等。如果编译成功，会生成可执行文件，并将其路径设置到child_argv中，以便后续执行。

最后，通过子进程执行构建脚本。如果子进程退出代码为0，表示成功；如果是2或3，可能有特定的错误处理，比如编译错误或缺少依赖项。如果无法生成子进程（如在不支持spawn的平台上），则会报错。

整个流程大致是：解析参数，配置构建环境，处理依赖项，编译构建脚本，然后执行该脚本。过程中涉及大量的错误处理和资源管理，比如目录的打开和关闭，缓存的配置，线程池的初始化和销毁等。

不过，有些部分我还不太确定。例如，处理依赖项时的unlazy_set和Package.Fetch的逻辑，可能需要更深入理解Zig的包管理机制。此外，如何处理不同的构建目标和缓存目录的具体细节可能需要进一步研究。不过，整体来看，这个函数负责协调整个构建过程，处理用户输入，配置环境，并最终调用子进程执行构建任务。
================================================
这段代码是 Zig 构建系统的核心逻辑，用于处理 `zig build` 命令的执行流程。以下是其思维模型的总结：

---

### **核心目标**
通过解析用户输入和配置环境，生成并执行一个构建脚本（`build_runner`），管理依赖、缓存、并发编译，最终完成项目的构建。

---

### **主要流程**
1. **初始化配置**  
   - 解析命令行参数（`args`），设置构建选项（如 `--build-file`、`--cache-dir`）。
   - 确定 Zig 可执行文件路径（`self_exe_path`）和库目录（`zig_lib_directory`）。
   - 初始化子进程参数列表（`child_argv`），准备调用构建脚本。

2. **处理环境变量和路径**  
   - 解析全局和本地缓存目录（`global_cache_directory`、`local_cache_directory`）。
   - 确定构建根目录（`build_root`）和构建脚本路径（`build_zig_basename`）。

3. **依赖管理**  
   - 使用线程池和 HTTP 客户端（若支持）获取远程依赖。
   - 通过 `Package.Fetch` 处理包依赖，生成依赖模块（`deps_mod`）。
   - 处理“惰性依赖”（`unlazy_set`），若依赖缺失则重新触发配置阶段。

4. **编译构建脚本**  
   - 创建 `Compilation` 实例，配置编译选项（如调试信息、详细输出）。
   - 生成构建脚本的可执行文件（`exe_basename`），并写入缓存目录。

5. **执行构建脚本**  
   - 通过子进程执行生成的构建脚本。
   - 根据子进程退出码处理不同场景：
     - **0**：成功退出。
     - **2**：构建脚本编译错误（直接退出）。
     - **3**：缺少惰性依赖（重新解析并继续循环）。

---

### **关键机制**
- **缓存管理**  
  全局缓存（跨项目共享）和本地缓存（项目特定）分离，支持通过环境变量覆盖路径。
- **并发与资源管理**  
  使用线程池（`thread_pool`）优化编译任务，限制文件描述符数量（`raiseFileDescriptorLimit`）。
- **错误处理**  
  通过 `fatal` 函数终止进程并输出错误，支持颜色化日志（`color` 选项）。
- **跨平台兼容**  
  处理不同操作系统的限制（如 WASI 不支持子进程时直接报错）。

---

### **依赖解析逻辑**
- **包获取（Fetch）**  
  递归解析 `build.zig` 中的依赖，生成哈希标识唯一包，通过 HTTP 客户端下载（若需）。
- **模块化构建**  
  每个依赖的 `build.zig` 被编译为独立模块（`Package.Module`），通过命名空间（如 `@dependencies`）隔离。
- **惰性依赖（Lazy Dependencies）**  
  若首次配置阶段发现缺失依赖，通过临时文件传递哈希，触发二次解析。

---

### **设计特点**
- **分层抽象**  
  分离配置（`configure`）和执行（`make`）阶段，未来可能进一步解耦。
- **灵活扩展**  
  支持通过 `--debug-target` 调试交叉编译，`--verbose-*` 选项提供详细日志。
- **环境隔离**  
  通过 `arena` 分配器管理临时内存，避免资源泄漏。

---

### **总结**
这段代码是 Zig 构建系统的“协调者”，负责整合配置、依赖、编译和执行，最终通过子进程驱动实际构建逻辑。其核心在于高效管理资源、处理复杂依赖关系，并提供灵活的错误反馈机制。