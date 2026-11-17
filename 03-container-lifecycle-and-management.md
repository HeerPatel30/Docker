# 03 — Container Lifecycle and Management

A container can start successfully and still stop a few seconds later. It can also be running without the application inside it working correctly.

To manage containers properly, understand three things:

1. **Lifecycle** — what makes a container start, run, stop, and disappear.
2. **Inspection** — how to find out what happened and collect evidence.
3. **Control** — how to start, stop, restart, remove, and automatically recover containers.

This guide combines the three episodes into one practical reference.

---

## 1. The Container Started, Then Vanished

Imagine starting a payment service:

```bash
docker run nginx
```

Docker returns a container ID, but when you run:

```bash
docker ps
```

the container is not listed.

That does not necessarily mean Docker failed to create it. The container may have started and then exited.

### 1.1 Understand the container lifecycle

A container moves through a few main states:

```text
Created  ── start ──>  Running
                         │
                         │ main process exits
                         ▼
                       Exited
                         │
                         │ docker rm
                         ▼
                       Removed
```

- **Created:** The container object exists, but its main process has not started.
- **Running:** Its main process is running.
- **Exited:** Its main process has stopped. The container object still exists.
- **Removed:** The container object has been deleted.

A removed container cannot be started again. You can create another container from the same image, but that is a new container with a new identity.

### 1.2 A container's life depends on its main process

A container is not a small virtual machine that stays alive simply because it was created.

Its running lifetime is tied to its main process, commonly called **PID 1** inside the container.

For example:

```bash
docker run alpine
```

If the image's default command finishes immediately, the container exits.

Try this:

```bash
docker run alpine echo "Hello from Docker"
```

The command prints the message and finishes. Once the main process exits, the container exits too.

This is expected behavior.

> **Important:** A container can exit successfully. An exit code of `0` means the process reported success; it does not mean the container should remain running.

### 1.3 Understand the `docker run` command

General syntax:

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

For example:

```bash
docker run --name web-server -d nginx
```

- `docker run` creates and starts a container.
- `--name web-server` assigns it a readable name.
- `-d` runs it in detached mode.
- `nginx` is the image.

Options go before the image name. A command written after the image overrides the image's configured default command.

### 1.4 Container names

Without a name, Docker assigns a random name.

```bash
docker run -d nginx
```

You can assign your own name:

```bash
docker run -d --name payment-service nginx
```

Names make it easier to inspect and manage containers:

```bash
docker logs payment-service
docker inspect payment-service
docker stop payment-service
```

A container name must be unique on that Docker daemon. If a stopped container still owns the name, you cannot use that name for another container until the old one is removed or renamed.

### 1.5 Detached and interactive modes

**Detached mode (`-d`)**

```bash
docker run -d --name web nginx
```

Docker starts the container and returns control to your terminal.

Detached mode does not guarantee that the container will keep running. If its main process exits, the container exits too.

**Interactive mode (`-i` and `-t`)**

```bash
docker run -it --name alpine-shell alpine sh
```

- `-i` keeps standard input open.
- `-t` allocates a terminal (TTY).
- `-it` is commonly used when you want to interact with a shell.

If the shell is the container's main process and you exit the shell, PID 1 exits and the container stops.

For a temporary learning container, you may see commands such as:

```bash
docker run -d --name test-container alpine sleep infinity
```

This keeps the `sleep` process running. It is useful for experiments, but it does not prove that a real application or service is healthy.

### 1.6 `docker ps` versus `docker ps -a`

```bash
docker ps
```

Shows running containers only.

```bash
docker ps -a
```

Shows all retained containers, including created and exited containers.

If a container is missing from `docker ps`, check:

```bash
docker ps -a
```

The difference matters:

| Command | What it shows |
|---|---|
| `docker ps` | Running containers |
| `docker ps -a` | All containers that have not been removed |

If a container does not appear in either list, it may have been removed, or you may be looking at a different Docker daemon/context.

### 1.7 What does `--rm` do?

```bash
docker run --rm alpine echo "Temporary task"
```

The `--rm` option tells Docker to automatically remove the container after its main process exits.

This is useful for disposable tasks, such as one-off commands and temporary tests.

However, once the container is automatically removed:

- You cannot restart that same container.
- Its container object and writable layer are gone.
- You lose the opportunity to inspect that retained container later.
- Logs or other evidence may no longer be available through that container.

Use `--rm` when you are sure you do not need the stopped container for troubleshooting.

**Remember:** `--rm` and restart policies are not meant to be used together. A container configured for automatic removal is not a container that Docker can keep around and restart.

---

## 2. Running Is Not the Same as Working

Suppose a production container is running, but users report that the application is unavailable.

Should you restart it immediately?

Not necessarily.

A running container only tells you that its main process is still running. It does not prove that the application is healthy, ready, or serving requests correctly.

Before changing anything, collect evidence.

A useful troubleshooting sequence is:

```text
INSPECT  →  CHECK LOGS  →  EXECUTE A TARGETED CHECK
```

### 2.1 Inspect a container

```bash
docker inspect payment-service
```

`docker inspect` returns detailed information about a Docker object in JSON format.

It can show:

- Container configuration.
- Current state.
- Exit code and timestamps.
- Image and startup command.
- Restart policy and restart count.
- Network configuration.

The output is large, so use `--format` to extract the fields you need.

**Check the current state:**

```bash
docker inspect payment-service --format '{{.State.Status}}'
```

Possible output:

```text
running
```

**Check whether the container is running:**

```bash
docker inspect payment-service --format '{{.State.Running}}'
```

**Check the exit code:**

```bash
docker inspect payment-service --format '{{.State.ExitCode}}'
```

**Check when it started and finished:**

```bash
docker inspect payment-service --format '{{.State.StartedAt}}'
docker inspect payment-service --format '{{.State.FinishedAt}}'
```

**Check the configured image and command:**

```bash
docker inspect payment-service --format '{{.Config.Image}}'
docker inspect payment-service --format '{{.Config.Entrypoint}}'
docker inspect payment-service --format '{{json .Config.Cmd}}'
```

**Check restart information:**

```bash
docker inspect payment-service --format '{{.RestartCount}}'
docker inspect payment-service --format '{{json .HostConfig.RestartPolicy}}'
```

The exact values depend on the container's configuration and Docker version.

### 2.2 Why use `{{json ...}}`?

Docker's `--format` option uses Go templates.

For a simple value, you can print the field directly:

```bash
docker inspect payment-service --format '{{.State.Status}}'
```

For an array or map, use the `json` template function:

```bash
docker inspect payment-service --format '{{json .Config.Cmd}}'
```

You can also inspect network configuration:

```bash
docker inspect payment-service --format '{{json .NetworkSettings.Networks}}'
```

This is useful because fields such as commands and network settings can contain structured data.

Template field names are case-sensitive. Also, do not assume every container has the same network name or IP address.

### 2.3 Read container logs

```bash
docker logs payment-service
```

Docker logs can help answer questions such as:

- Did the application start?
- Did it report a configuration error?
- Did it fail to connect to a database?
- Did it receive a shutdown signal?
- Did it report an exception?

Useful options:

**Show only the latest 50 lines:**

```bash
docker logs --tail 50 payment-service
```

**Show logs from the last 10 minutes:**

```bash
docker logs --since 10m payment-service
```

**Follow new log output:**

```bash
docker logs -f payment-service
```

You can combine options:

```bash
docker logs --since 10m --tail 50 -f payment-service
```

Pressing `Ctrl+C` stops following the log output in your terminal. It does not stop the container.

### 2.4 Where do Docker logs come from?

`docker logs` displays the container process's captured standard output (`stdout`) and standard error (`stderr`), subject to the configured logging driver and its settings.

For example, an application that writes startup messages to the console can make those messages available through:

```bash
docker logs payment-service
```

But `docker logs` is not a command for reading every log file inside the container.

Keep these points in mind:

- Applications should generally write container logs to `stdout` and `stderr`.
- The logging driver controls how Docker handles log output.
- Log rotation and retention settings affect how much history is available.
- A lack of output does not prove that no error occurred.
- Logs may not be available in the same way with every logging-driver configuration.

You can often read logs from an exited container as long as the container remains and its logs are still available.

### 2.5 Execute a command inside a running container

Use `docker exec` to start an additional process inside an already-running container.

General syntax:

```bash
docker exec CONTAINER COMMAND [ARG...]
```

For example:

```bash
docker exec payment-service printenv APP_ENV
```

To open a shell, if the image contains one:

```bash
docker exec -it payment-service sh
```

Some images include Bash:

```bash
docker exec -it payment-service bash
```

Minimal images may not contain `bash`, `sh`, or common troubleshooting tools. In that case, use an available executable or another debugging approach.

### 2.6 `docker exec` does not replace PID 1

This distinction is important.

When you run:

```bash
docker exec -it payment-service sh
```

Docker starts a new process inside the running container. That shell is not the container's main process.

If you exit the shell, the application can continue running.

Compare:

| Situation | Result |
|---|---|
| Start a container whose main command is `sh`, then exit the shell | The main process exits, so the container stops |
| Run `docker exec -it container sh`, then exit the shell | Only the additional shell exits; the main process can continue |

`docker exec` is not SSH. It does not create an SSH connection or require an SSH server inside the container. Docker asks the Docker daemon to start another process in the container.

Also remember that access to the Docker daemon is powerful. Someone who can control Docker may be able to gain extensive access to the host.

### 2.7 A practical troubleshooting order

When a service is reported as unavailable, avoid immediately restarting or removing its container.

Start with:

```bash
docker ps -a
docker inspect payment-service
docker logs --tail 100 payment-service
```

If the container is running and you need to verify something from inside it:

```bash
docker exec payment-service printenv APP_ENV
```

Then check the application itself, its dependencies, network connectivity, and health/readiness indicators as appropriate.

**Key distinction:**

- **Running:** The container's main process is running.
- **Healthy:** A configured health check reports that the application meets its health condition.
- **Ready:** The application is prepared to accept the work or traffic expected of it.

These are not interchangeable.

---

## 3. Recovery or Evidence Destruction?

Once you understand the container's state and have collected useful evidence, you can decide what action to take.

Docker provides several lifecycle commands. They do different jobs.

### 3.1 Start an existing container

```bash
docker start payment-service
```

Starts an existing created or stopped container.

It reuses the same container object and its retained configuration and writable layer.

Use `docker start` when you want to start that existing container again.

### 3.2 Restart a container

```bash
docker restart payment-service
```

A restart stops the container and starts the same container again.

It does not create a new container.

A restart can be useful when an application needs to be restarted, but it changes the running process and lifecycle timestamps. If you restart before collecting evidence, you may lose useful information about the original failure.

### 3.3 Stop a container gracefully

```bash
docker stop payment-service
```

By default, Docker sends the container's configured stop signal. This is normally `SIGTERM`.

Docker then waits for a grace period (10 seconds by default). If the main process has not exited by the end of that period, Docker sends `SIGKILL`.

You can change the timeout:

```bash
docker stop --time 30 payment-service
```

This gives the process up to 30 seconds to stop before Docker forcefully terminates it.

A well-behaved application should respond to a graceful shutdown by doing things such as:

1. Stop accepting new work.
2. Finish or safely cancel work already in progress.
3. Flush important data and logs.
4. Close database connections and other resources.
5. Exit cleanly.

The application must handle the stop signal correctly. Process wrappers and shell scripts can affect whether signals reach the intended application process.

### 3.4 What is `SIGTERM`?

`SIGTERM` is a termination signal asking a process to stop.

It gives the application an opportunity to perform cleanup before exiting.

For example, a web application may stop accepting new requests, finish current requests, close connections, and then exit.

### 3.5 What is `SIGKILL`?

`SIGKILL` forces a process to terminate. The process cannot catch or ignore it and cannot perform its own graceful cleanup.

Docker's `kill` command sends `SIGKILL` by default:

```bash
docker kill payment-service
```

You can specify another signal:

```bash
docker kill --signal SIGHUP payment-service
```

The effect of a signal other than `SIGKILL` depends on how the application handles it.

Use `docker kill` when forceful termination is necessary, not as the normal first choice for stopping an application.

### 3.6 `docker stop` versus `docker kill`

| Command | Default behavior | Cleanup opportunity |
|---|---|---|
| `docker stop` | Sends the configured stop signal, waits, then sends `SIGKILL` if needed | Yes, during the grace period |
| `docker kill` | Sends `SIGKILL` by default | No graceful cleanup |

A simple way to remember:

- `stop` gives the application time to shut down.
- `kill` forcefully terminates it by default.

### 3.7 Understand container exit codes

An exit code is a value returned by the process when it exits.

You can inspect it with:

```bash
docker inspect payment-service --format '{{.State.ExitCode}}'
```

Common values and conventions:

| Exit code | Common meaning |
|---|---|
| `0` | Process completed successfully |
| `1` | General application failure (a common convention) |
| `125` | Docker failed before the container command could run |
| `126` | Command was found but could not be invoked |
| `127` | Command was not found |
| `137` | Often `128 + 9`, associated with `SIGKILL` |
| `143` | Often `128 + 15`, associated with `SIGTERM` |

For example:

- `137` commonly indicates that the process was killed with `SIGKILL`.
- `143` commonly indicates that the process was terminated with `SIGTERM`.

Do not assume that exit code `137` always means an out-of-memory (OOM) kill. It can also result from another source of `SIGKILL`. Check container state, daemon events, and host-level evidence before deciding what happened.

Signal-related exit-code conventions can also be affected by wrappers and the way the application exits.

### 3.8 Remove a stopped container

```bash
docker rm payment-service
```

This removes the container object.

It does not remove the image used to create the container.

Docker normally refuses to remove a running container. Stop it first:

```bash
docker stop payment-service
docker rm payment-service
```

You can also remove an exited container directly:

```bash
docker rm payment-service
```

Be careful with removal:

- The container object is deleted.
- Its writable container layer is deleted.
- You cannot start that same container again.
- Volumes managed separately are not removed by default.

Avoid force removal unless you understand the impact:

```bash
docker rm -f payment-service
```

Force removal can stop and remove a running container, so it may interrupt service and destroy useful troubleshooting evidence.

**Important:** Removing a container is not the same as removing its image or deleting its separately managed volumes.

### 3.9 Understand Docker restart policies

What happens if the main process exits unexpectedly?

You can configure Docker to apply a restart policy.

Example:

```bash
docker run -d \
  --name payment-service \
  --restart unless-stopped \
  nginx
```

The restart policy is configured when the container is created. The common policies are:

| Policy | Behavior |
|---|---|
| `no` | Do not automatically restart the container (default) |
| `on-failure` | Restart when the container exits with a non-zero exit code |
| `on-failure:5` | Retry on failure, up to the specified retry limit |
| `always` | Always restart the container when it exits, subject to Docker's restart behavior |
| `unless-stopped` | Restart after exits unless it was explicitly stopped by the user or Docker |

**`no`**

```bash
docker run -d --restart no nginx
```

Docker does not automatically restart the container when its process exits.

**`on-failure`**

```bash
docker run -d --restart on-failure nginx
```

Docker restarts the container when its main process exits with a non-zero exit code.

You can set a retry limit:

```bash
docker run -d --restart on-failure:5 nginx
```

**`always`**

```bash
docker run -d --restart always nginx
```

Docker attempts to restart the container whenever it exits. This policy can also cause the container to start again when the Docker daemon restarts.

**`unless-stopped`**

```bash
docker run -d --restart unless-stopped nginx
```

This behaves similarly to `always`, but preserves an intentional manual stop across daemon restarts.

### 3.10 Restart policies are not health checks

A restart policy reacts to the container's main process exiting. It does not automatically repair an application that is still running but malfunctioning.

For example:

- The web server process is running.
- The application cannot connect to its database.
- The container remains in the running state.
- The restart policy may do nothing because the main process has not exited.

A health check can help report application health, but a restart policy and a health check are different mechanisms.

A bad configuration can also cause repeated restarts. If the application exits immediately every time it starts, Docker may keep trying to restart it according to the configured policy.

### 3.11 Manual stops and restart policies

Restart policies have behavior designed to respect an intentional manual stop.

For example:

- If you manually stop a container, Docker does not immediately restart it against that stop request.
- `always` can restart a container after the Docker daemon restarts.
- `unless-stopped` keeps a manually stopped container stopped when the daemon restarts.
- `on-failure` is based on a non-zero process exit; a daemon restart by itself is not a process failure.

Do not treat restart policies as a replacement for monitoring, application health checks, or fixing the root cause.

---

## 4. A Practical Container Troubleshooting Workflow

When a container behaves unexpectedly, follow this order:

### Step 1 — Find the container

```bash
docker ps -a
```

Check whether it is running, exited, or still created.

### Step 2 — Inspect its state

```bash
docker inspect payment-service
```

Focus on:

- `.State.Status`
- `.State.ExitCode`
- `.State.StartedAt`
- `.State.FinishedAt`
- `.RestartCount`
- `.HostConfig.RestartPolicy`

### Step 3 — Read its logs

```bash
docker logs --tail 100 payment-service
```

Look for startup errors, dependency failures, exceptions, and shutdown messages.

### Step 4 — Run a targeted check if it is running

```bash
docker exec payment-service printenv APP_ENV
```

Use a shell only if the image contains one:

```bash
docker exec -it payment-service sh
```

### Step 5 — Choose the right action

| Situation | Possible action |
|---|---|
| Existing container is stopped and should run again | `docker start` |
| Application needs a controlled restart | `docker restart` |
| Application should shut down cleanly | `docker stop` |
| Process must be terminated immediately | `docker kill` |
| Stopped container is no longer needed | `docker rm` |
| Container should recover after process failure | Configure an appropriate restart policy |

### Step 6 — Verify the result

```bash
docker ps -a
docker inspect payment-service --format '{{.State.Status}}'
docker logs --tail 50 payment-service
```

Do not stop at seeing `running`. Verify that the application is actually responding and that its dependencies are available.

---

## 5. Common Mistakes to Avoid

1. **Assuming `docker run -d` means the container will stay running.**  
   Detached mode only separates the container from the terminal. The main process still controls its lifetime.

2. **Using `docker ps` to look for exited containers.**  
   Use `docker ps -a` to see retained stopped containers.

3. **Using `--rm` when you need to troubleshoot later.**  
   The container is automatically removed after it exits.

4. **Assuming a running container means a healthy application.**  
   Check application logs, health checks, readiness, and dependencies.

5. **Restarting before collecting evidence.**  
   Inspect the state and logs first whenever possible.

6. **Using `docker exec` on a stopped container.**  
   `docker exec` requires a running container.

7. **Assuming every image contains Bash or common Linux tools.**  
   Minimal images may not include them.

8. **Using `docker kill` as the normal shutdown command.**  
   Prefer graceful shutdown with `docker stop` when possible.

9. **Assuming exit code `137` proves an OOM event.**  
   It commonly indicates `SIGKILL`, but more evidence is needed to identify why.

10. **Assuming restart policies fix application failures.**  
    They restart containers after qualifying process exits; they do not repair broken application configuration or dependencies.

11. **Removing a container before collecting useful evidence.**  
    Once removed, the same container object and its writable layer are gone.

---

## 6. Quick Command Reference

| Command | Purpose |
|---|---|
| `docker run IMAGE` | Create and start a container |
| `docker run -d IMAGE` | Start in detached mode |
| `docker run -it IMAGE sh` | Start an interactive shell as the main process |
| `docker ps` | List running containers |
| `docker ps -a` | List all retained containers |
| `docker inspect CONTAINER` | View detailed container information |
| `docker logs CONTAINER` | Read captured container logs |
| `docker logs -f CONTAINER` | Follow container logs |
| `docker exec -it CONTAINER sh` | Start an interactive shell in a running container |
| `docker start CONTAINER` | Start an existing stopped container |
| `docker restart CONTAINER` | Restart the same container |
| `docker stop CONTAINER` | Gracefully stop a container, then force it if necessary |
| `docker kill CONTAINER` | Send `SIGKILL` by default |
| `docker rm CONTAINER` | Remove a stopped container |
| `docker run --rm IMAGE` | Automatically remove the container after it exits |
| `docker run --restart unless-stopped IMAGE` | Configure an automatic restart policy |

---

## 7. Check Your Understanding

Try answering these before looking back at the notes.

1. Why can a container exit even when `docker run` returns a container ID?
2. What is the difference between `docker ps` and `docker ps -a`?
3. Does `-d` guarantee that a container will keep running?
4. What happens when the main process (PID 1) exits?
5. What is the purpose of `--rm`?
6. Why might `docker inspect` be more useful than checking only `docker ps`?
7. What is the difference between `docker logs` and `docker exec`?
8. Why does exiting a shell started with `docker exec` usually not stop the container?
9. What is the difference between `docker stop` and `docker kill`?
10. Does exit code `137` always mean the container ran out of memory?
11. What is the difference between `always` and `unless-stopped`?
12. Will a restart policy necessarily restart a container whose application is unhealthy but whose main process is still running?

### Interview perspective

Be ready to explain these ideas in your own words:

- A container is a process with isolation, not a miniature virtual machine.
- The main process determines whether the container is running.
- Detached mode changes how the process is attached to your terminal; it does not change the process lifecycle.
- `inspect`, `logs`, and `exec` answer different troubleshooting questions.
- Graceful shutdown gives an application time to clean up.
- Restart policies react to process exits; they are not application repair mechanisms.
- Removing a container deletes that container object, not the image it came from.

---

## 8. Final Takeaway

Think of container management as three stages:

```text
LIFECYCLE
  Create → Start → Run → Exit → Remove

INVESTIGATION
  Inspect → Read logs → Run targeted checks

CONTROL
  Start → Stop → Restart → Kill → Remove
                 │
                 └── Restart policies can automate recovery
```

The most important habit is simple:

**Understand the state before taking action. Collect evidence before restarting or removing a container. Verify the application after making a change.**

That is the difference between merely running Docker commands and managing containers reliably.
