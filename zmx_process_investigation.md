# zmx Process Investigation

## Using `zmx list`

The command `zmx list` provides an overview of all active `zmx` sessions. When executed (without the `--short` flag), it queries each session's daemon via IPC and outputs tab-separated information. This includes:

- **`name`**: The name of the session.
- **`pid`**: The Process ID (PID) of the shell or command running inside the pseudo-terminal (PTY) for that session.
- **`clients`**: The number of clients currently attached.
- **`created`**: The Unix timestamp of when the session was created.
- **`start_dir`** (optional): The working directory when the session was created.
- **`cmd`** (optional): The command being run, if specified (e.g., `zmx run <name> <cmd>`).

**Example output:**
```
name=dev   pid=12345   clients=1   created=1713000000   start_dir=/home/user/project
```

### Querying Process Information with OS Tools

Since `zmx list` directly exposes the PID of the wrapped process (the child process created by `forkpty` in the daemon), you can use standard OS tools to query detailed information about it:

1. **`ps`**:
   Query the status, CPU/memory usage, and arguments of the wrapped process:
   ```bash
   ps -p <pid> -o user,pid,ppid,state,cmd
   ```

2. **`pstree`**:
   View the process hierarchy. This is very useful if you are running a script or a build command inside the `zmx` session that spawns other child processes:
   ```bash
   pstree -p <pid>
   ```

3. **`/proc/<pid>` (Linux)**:
   You can inspect the process's live state directly from the `/proc` filesystem:
   - **Environment variables**: `cat /proc/<pid>/environ | tr '\0' '\n'`
   - **Current working directory**: `readlink /proc/<pid>/cwd`
   - **Open file descriptors**: `ls -l /proc/<pid>/fd`
   - **Process status**: `cat /proc/<pid>/status`

---

## Exploring the zmx State Directory

By default, `zmx` stores its state in a directory determined by environment variables (in priority order: `ZMX_DIR`, `$XDG_RUNTIME_DIR/zmx`, or `/tmp/zmx-<uid>`). The state directory contains Unix domain sockets and log files for each session.

### 1. Unix Domain Sockets

- For each active session, there is a socket file named after the session (e.g., `/tmp/zmx-1000/dev`).
- The daemon for each session listens on this socket, and `zmx` clients communicate with the daemon by connecting to it.
- Using tools like `lsof` or `fuser`, you can identify which processes have the socket open. This can help you locate the `zmx` **daemon process** (the parent of the PTY shell process exposed by `zmx list`), as well as any connected clients.
  ```bash
  fuser /tmp/zmx-1000/dev
  ```
  or
  ```bash
  lsof /tmp/zmx-1000/dev
  ```

### 2. Logs Subdirectory (`logs/`)

- `zmx` creates a `logs/` directory inside the state directory.
- **`zmx.log`**: Contains global logs for CLI commands.
- **`<session_name>.log`** (e.g., `dev.log`): Contains debug logs specifically for that session's daemon. You can tail this file to see internal daemon activity, such as client connections, PTY inputs/outputs, resize events, and potential errors.
  ```bash
  tail -f /tmp/zmx-1000/logs/dev.log
  ```
  *(Note: Logging is currently enabled by default in the codebase to help with debugging.)*

## Summary

- **`zmx list`** provides the direct PID of the shell/process running inside the session.
- You can feed this PID to tools like **`ps`**, **`pstree`**, or read **`/proc/<pid>`** to monitor the process and its children.
- The **socket files** in the state directory can be inspected with `fuser` or `lsof` to find the `zmx` daemon process.
- The **`logs/`** subdirectory contains global and per-session logs useful for debugging the `zmx` daemon's behavior.