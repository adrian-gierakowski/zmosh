# Task: Port IPC Tags Improvements

Your goal is to port a minor IPC protocol addition from the `zmosh` fork back into the `zmx` project.

1. **Update `src/ipc.zig`:**
   Find the `Tag` enum in `src/ipc.zig`.
   Add `SessionEnd = 11,` to the enum. 

This change ensures that the IPC protocol remains aligned between the core project and any future plugins/forks like `zmosh` that need to signal the end of a session gracefully.

Ensure the code compiles successfully after making this change.