# Task: Port Pty Output Logic Improvements

Your goal is to port improvements to the PTY output broadcasting logic from the `zmosh` fork back into the `zmx` project. 

The `zmx` codebase has been refactored, so you will need to locate the `Client` struct and the daemon loop (likely in `src/main.zig`, `src/socket.zig`, or `src/util.zig`) and apply the following changes:

1. **Update the `Client` struct:**
   Add a new field `initialized: bool = false` to the `Client` struct.

2. **Update Client Initialization:**
   When handling a client's initialization/resize event (the point at which terminal state is serialized and sent to the client):
   - Set `client.initialized = true;`.
   - If the client is being re-initialized, clear its `write_buf` (e.g., `client.write_buf.clearRetainingCapacity();`) before appending the terminal snapshot. This drops any stale output buffered before the init so the snapshot is the first payload rendered.

3. **Update PTY Output Broadcasting:**
   In the daemon loop where PTY output is read and broadcasted to clients:
   - Reset the `clear_detected` flag on the VT stream handler before processing the new chunk of output (e.g., `vt_stream.handler.clear_detected = false;`). Note: This relies on the `ScrollPreservingHandler` being implemented (another task). If it's not present yet, you can add a stub for `clear_detected` or coordinate with that task.
   - Only broadcast PTY output to clients where `client.initialized == true`. Utility clients (run/history/probe) never send Init and should only receive explicit replies.
   - If `vt_stream.handler.clear_detected` is true, prepend `\x1b[22J` (scroll complete) to the output buffer before forwarding the raw bytes to the client.

4. **Update Pending Output Flushing:**
   In the poll loop, ensure that pending output is flushed immediately if a client queues replies while handling `POLLIN`. Check `if ((revents & posix.POLL.OUT != 0) or client.has_pending_output)` to trigger the write flush.

Ensure your changes compile and pass any existing tests.