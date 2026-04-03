# Task: Port Terminal State Serialization Improvements

Your goal is to port the terminal state serialization improvements from the `zmosh` fork back into the `zmx` project. 

The `zmx` codebase has been refactored recently, so you will need to find where the terminal state serialization logic resides (likely in `src/main.zig`, `src/socket.zig`, or `src/util.zig`) and apply the following changes:

1. **Introduce `ScrollPreservingHandler`:**
   Create a custom VT stream handler that detects `ESC[2J` (erase display complete) on the primary screen. This sets a flag so the daemon can prepend a scroll-preserving sequence (`ESC[22J`) to client output.
   
   Example implementation from the fork:
   ```zig
   const ScrollPreservingHandler = struct {
       terminal: *ghostty_vt.Terminal,
       clear_detected: bool = false,

       pub fn init(terminal: *ghostty_vt.Terminal) ScrollPreservingHandler {
           return .{ .terminal = terminal };
       }

       pub fn deinit(_: *ScrollPreservingHandler) void {}

       pub fn vt(
           self: *ScrollPreservingHandler,
           comptime action: ghostty_vt.StreamAction.Tag,
           value: ghostty_vt.StreamAction.Value(action),
       ) !void {
           if (comptime action == .erase_display_complete) {
               if (self.terminal.screens.active_key == .primary) {
                   self.terminal.screens.active.scrollClear() catch {};
                   self.clear_detected = true;
               }
           }
           var handler = self.terminal.vtHandler();
           return handler.vt(action, value);
       }
   };
   ```

2. **Update VT Stream Initialization:**
   Update the instantiation of `ghostty_vt.Stream` to use the new `ScrollPreservingHandler` instead of the default handler.

3. **Update `serializeTerminalState`:**
   Modify the signature of the function that serializes terminal state to accept `client_rows: u16`.
   Implement the two-phase serialization logic (history first, then active screen) to ensure proper scrollback preservation on re-attach.

   Reference the `zmosh` fork at `/code/github.com/mmonad/zmosh` or your knowledge of the diff to implement the two-phase logic:
   - Phase 1: Serialize scrollback history as content only. Push exactly `min(history_rows, client_rows)` newlines to scroll rendered content off screen.
   - Phase 2: Serialize the active screen with cursor position and terminal modes.

Ensure your changes compile and pass any existing tests.