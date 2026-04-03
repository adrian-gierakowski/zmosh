Integrate the zmosh remote functionality into zmx using a modular, conditionally compiled approach. This respects the zmx author's desire to keep the core tool simple while supporting the remote features as an opt-in addition.

# Phase 1: Upstream Generic Improvements
Before porting the network layer, there are several modifications made in zmosh's main.zig that are strictly local improvements and bug fixes. These should be merged into zmx directly to reduce the delta.
 1. Terminal State Serialization: Port the ScrollPreservingHandler and the updated serializeTerminalState (the two-phase history and active screen serialization) back to zmx. This improves how local scrollback history behaves when re-attaching, benefiting zmx users directly.
 2. IPC Tags: Add the SessionEnd = 11 tag to src/ipc.zig in zmx. Even if zmx doesn't use it heavily yet, it keeps the IPC protocol aligned.
 3. Pty Output Logic: Upstream the logic in daemonLoop that drops stale output buffered before Init and prepends \x1b[22J (scroll complete) when clear_detected is true.

# Phase 2: Establish the contrib/zmosh Workspace
Move the zmosh-specific networking and cryptography features into an isolated directory so they don't pollute the zmx core codebase.
 1. Create a contrib/zmosh/ directory at the root of the zmx repository.
 2. Move the following files from the fork into contrib/zmosh/src/:
    - remote.zig
    - serve.zig
    - transport.zig
    - udp.zig
    - crypto.zig
    - lib.zig
 3. Move include/zmosh/zmosh.h to contrib/zmosh/include/zmosh.h.
 4. Move docs/udp-auto-reconnect-research.md to contrib/zmosh/docs/.

# Phase 3: Conditional Compilation via build.zig
We will use Zig's build system and comptime to ensure zmx has zero overhead and zero knowledge of zmosh when compiled normally.
 1. In zmx/build.zig, introduce a new build option:

 1    const enable_zmosh = b.option(bool, "enable-zmosh", "Enable zmosh remote UDP session support") orelse false;
 2    options.addOption(bool, "enable_zmosh", enable_zmosh);
 2. If enable_zmosh is true:
    - Create a zmosh_mod using the source files in contrib/zmosh/src/.
    - Add zmosh_mod as an import to the main executable: exe_mod.addImport("zmosh", zmosh_mod);
    - (Optional) Conditionally add the library build steps (lib, macos-lib, ios-lib, xcframework) to build.zig by calling a helper function from contrib/zmosh/build.zig.

# Phase 4: comptime Hooks in src/main.zig
Refactor the CLI routing in zmx/src/main.zig to conditionally tap into the zmosh module using Zig's comptime evaluation. If enable_zmosh is false, the compiler will completely optimize out the zmosh branches.

 1. Import the module conditionally:

 1    const build_options = @import("build_options");
 2    const zmosh = if (build_options.enable_zmosh) @import("zmosh") else struct {};

 2. Hook into attach:
   Modify the attach command parsing to recognize -r / --remote only if zmosh is enabled.

  1    // Inside the attach command parser
  2    if (build_options.enable_zmosh and (std.mem.eql(u8, arg, "--remote") or std.mem.eql(u8, arg, "-r"))) {
  3        remote_host = args.next();
  4    }
  5    ...
  6    if (remote_host) |host| {
  7        if (build_options.enable_zmosh) {
  8            const session = try zmosh.connectRemote(alloc, host, sesh);
  9            return zmosh.remoteAttach(alloc, session);
 10        } else {
 11            // Fallback/safety check, though parsing shouldn't allow it
 12            @panic("zmx was not built with zmosh support");
 13        }
 14    }

 3. Hook into serve:
   Add a conditional block to the main CLI router to handle the new serve command.

  1    if (build_options.enable_zmosh and (std.mem.eql(u8, cmd, "serve") or std.mem.eql(u8, cmd, "s"))) {
  2        const session_name = args.next() orelse "";
  3        const sesh = try util.getSeshName(alloc, session_name);
  4        defer alloc.free(sesh);
  5
  6        // Bootstrap the daemon similarly to local attach, then hand over
  7        var daemon = try setupDaemon(cfg, alloc, sesh);
  8        const result = try ensureSession(&daemon);
  9        if (result.is_daemon) return;
 10
 11        return zmosh.serveMain(alloc, sesh);
 12    }

 4. Conditionally update the Help menu:
   Use an if (build_options.enable_zmosh) block inside help() to print the serve and attach -r usage instructions only when built with the feature.

# Phase 5: Expose Required Core Functions
The code inside contrib/zmosh/src/ (like serve.zig) relies on calling into core zmx functions (like ensuring a session is running, finding the socket path, etc.).
 - Ensure that functions like getSocketPath, ensureSession, and the client communication primitives are marked pub in zmx/src/util.zig or zmx/src/socket.zig.
 - The zmosh module will import the root zmx module or specific utility files so it can interface with the local Unix sockets seamlessly as a "gateway".

Summary of Benefits
 - Strict Separation: zmx remains a simple local session tool by default.
 - Zero-Cost: For users installing standard zmx, the resulting binary has zero bloat, as the zmosh code is pruned at compile time.
 - Easy Maintenance: Patches and updates to zmosh are isolated to contrib/zmosh/, meaning zmx's author doesn't have to navigate network code to fix local PTY issues.