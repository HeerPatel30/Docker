# 03 - Docker Container Lifecycle and Management

## Introduction

Starting a Docker container is easy. Managing it correctly is what matters in real-world environments.

A container may start and exit immediately, remain running while the application is unhealthy, or stop unexpectedly because its main process has failed.

To manage containers properly, you need to understand:

- How a container moves through its lifecycle.
- Why PID 1 controls the container's running lifetime.
- How to inspect container configuration and state.
- How to retrieve container logs.
- How to execute commands inside a running container.
- How to start, stop, restart, kill, and remove containers.
- How Docker restart policies behave.

This module covers these concepts through three episodes.

---

# Episode 1: Understanding the Container Lifecycle

## 1. The Container Started, Then Vanished

Imagine an engineer starts a new `payment-service` container on `server-01`.

Docker prints a container ID, but when the engineer runs `docker ps`, the container does not appear.

Did the container disappear?

Not necessarily. It may have started successfully and then exited.

Remember the fundamental rule:

> A container remains running as long as its main process, PID 1, is running.

To understand what happened, you must distinguish between a container's existence and its running state.

## 2. Think in States, Not Commands

A Docker container moves through a lifecycle:

```text
created → running → exited → removed
```

Each state represents something different.

| State | Meaning |
|---|---|
| `created` | Docker has created the container, but its main process has not started. |
| `running` | The container's main process is running. |
| `exited` | The main process has ended, but the container still exists. |
| `removed` | The container has been deleted. Its container identity and writable layer are gone. |

The `removed` state is terminal for that particular container identity.

You can create another container from the same image, but it will be a new container.

### What happens when you run docker run?

Consider:

```bash
docker run nginx
```

Docker performs two operations:

1. Creates a new container.
2. Starts the container's main process.

Conceptually:

```text
BEFORE:
No container

COMMAND:
docker run nginx

AFTER:
Container created → Container started
```

`docker run` does not guarantee that the container will remain running forever.

The main process might complete its work and exit immediately.

### Quick Example

```bash
docker run alpine echo "Hello Docker"
```

The container prints:

```text
Hello Docker
```

Then it exits.

This is expected because `echo` is a short-lived process. Once it finishes, there is no main process left to keep the container running.

## 3. PID 1 Defines the Container's Running Lifetime

Inside a container's PID namespace, the configured main process normally becomes PID 1.

Docker considers the container running while this main process is running.

For example:

```bash
docker run nginx
```

The Nginx process runs in the foreground and keeps the container alive.

But consider an application that starts its service in the background and then exits.

```text
Container starts
      |
      v
Application starts in background
      |
      v
Startup script finishes
      |
      v
PID 1 exits
      |
      v
Container exits
```

The background process might not behave as the application author expected. The container's lifecycle is tied to its main process, not simply to whether some process was started earlier.

**Important:** Keep the actual application in the foreground whenever possible so that it remains the container's main process.

### A Stand-in Process Is Not the Application

You might see commands such as:

```bash
sleep infinity
```

This keeps the `sleep` process running.

However:

```text
sleep is running
        ≠
payment-service is running
```

A container can have a running status without providing the intended application service.

Use `sleep infinity` for temporary learning or diagnostic environments, not as a replacement for a properly configured application process.

## 4. Understanding docker run Options

The general syntax is:

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

- `OPTIONS` configure container behavior.
- `IMAGE` identifies the image to use.
- `COMMAND` overrides the image's configured command.
- `ARG...` provides arguments to that command.

### Give Containers a Name

```bash
docker run --name payment-service nginx
```

`--name` assigns a human-readable name to the container.

Without it, Docker generates a name automatically.

Names make containers easier to identify and manage:

```bash
docker start payment-service
docker stop payment-service
docker logs payment-service
```

A container name must be unique on the Docker daemon.

Even an exited container continues to own its name until it is removed.

### Detached Mode: -d

```bash
docker run -d --name payment-service nginx
```

The `-d` option runs the container in detached mode.

Docker returns control of the terminal while the container continues running independently.

This is useful for services that should run without occupying your current terminal.

Remember:

```text
-d = Run without attaching the current terminal
```

Detached mode does not guarantee that the application will stay alive. PID 1 can still exit immediately.

### Interactive Terminal Mode: -it

```bash
docker run -it alpine sh
```

Two options are combined here:

| Option | Meaning |
|---|---|
| `-i` | Keeps standard input open. |
| `-t` | Allocates a terminal-like interface. |

Together, `-it` is useful when you need to interact with a process through your terminal.

For example:

```bash
docker run -it ubuntu bash
```

You can now interact with the shell inside the container.

When you type:

```bash
exit
```

The shell exits.

If that shell is PID 1, the container also exits.

Remember:

```text
-d  → Detached operation

-it → Interactive terminal
```

These options change how you interact with the container. They do not change the PID 1 lifecycle rule.

## 5. See Running and Exited Containers

Docker provides two important commands for viewing containers.

### Show Running Containers

```bash
docker ps
```

By default, this command shows only running containers.

If the output is empty, it means there are no containers currently running on the selected Docker daemon.

It does not mean that no containers were created.

### Show All Containers

```bash
docker ps -a
```

The `-a` option includes containers that have exited or are otherwise retained.

Example:

```text
CONTAINER ID   IMAGE     COMMAND       STATUS
a1b2c3d4       alpine    "echo Hello"  Exited (0) 10 seconds ago
```

This tells us:

- The container still exists.
- Its main process has finished.
- Docker recorded an exit status of `0`.

Use the following approach before changing a container:

```text
LIST
  ↓
IDENTIFY
  ↓
CONFIRM STATE
  ↓
CHANGE
  ↓
VERIFY
```

## 6. Understanding --rm

Normally, when a container exits, Docker retains the container object.

You can inspect it, read its logs, and start it again.

But sometimes you only need a container for a temporary task.

For that, Docker provides `--rm`.

```bash
docker run --rm alpine echo "Temporary task"
```

Docker automatically removes the container after it exits.

Compare the behavior:

**Without `--rm`:**

```text
running → exited → retained
```

You can inspect or restart the same container.

**With `--rm`:**

```text
running → exited → removed
```

The container is automatically deleted.

### When Should You Use --rm?

Use it for:

- Temporary testing.
- One-time commands.
- Disposable troubleshooting containers.
- Short-lived tasks that do not need later investigation.

Avoid it when you need to:

- Inspect the container after it exits.
- Read its retained logs.
- Restart the same container.
- Preserve its writable layer for investigation.

Docker also does not allow combining automatic removal with a restart policy. One requests removal after exit; the other requests automatic restarting.

---

## Episode 1: Important Takeaways

- A container moves through `created → running → exited → removed`.
- `docker run` creates and starts a container.
- PID 1 controls the container's running lifetime.
- `-d` runs a container in detached mode.
- `-it` provides an interactive terminal.
- `docker ps` shows running containers.
- `docker ps -a` includes retained exited containers.
- `--rm` automatically removes a container after it exits.

---

# Episode 2: Inspect, Logs, and Exec

## 1. Running Is Not the Same as Working

Imagine the `checkout-service` container is listed as running, but customers cannot complete their payments.

One engineer immediately wants to restart it.

Another wants to open a shell and investigate.

But before changing anything, we need to understand what is happening.

A container being `running` only tells us that its main process is running. It does not guarantee that the application is healthy or responding correctly.

Docker provides three important tools for investigation:

```text
inspect → Structured container configuration and state

logs    → Captured standard output and standard error

exec    → Start a new process inside a running container
```

Each tool answers a different question.

## 2. Understanding docker inspect

The `docker inspect` command returns detailed information about a Docker object.

For example:

```bash
docker inspect payment-service
```

It returns JSON containing information about the selected container.

You can use it to understand:

- Container configuration.
- Current lifecycle state.
- Exit code.
- Startup command.
- Entrypoint.
- Restart policy.
- Network attachments.
- Mount configuration.

### Important Fields

| Field | Meaning |
|---|---|
| `.Name` | Container name. |
| `.State.Status` | Current container state. |
| `.State.Running` | Whether the main process is running. |
| `.State.ExitCode` | Recorded exit code. |
| `.State.StartedAt` | Last recorded start time. |
| `.State.FinishedAt` | Last recorded finish time. |
| `.State.Error` | Recorded error information, if available. |
| `.Config.Image` | Image reference used by the container. |
| `.Config.Entrypoint` | Configured entrypoint. |
| `.Config.Cmd` | Configured command and arguments. |
| `.RestartCount` | Number of recorded restarts. |
| `.HostConfig.RestartPolicy` | Configured restart policy. |
| `.NetworkSettings.Networks` | Network attachment information. |

Field availability and behavior can depend on the Docker object and version.

### Extract Specific Information with --format

The complete JSON output can be large.

You can use `--format` to retrieve only the information you need.

Check the container status:

```bash
docker inspect --format '{{.State.Status}}' payment-service
```

Example:

```text
running
```

Check the exit code:

```bash
docker inspect --format '{{.State.ExitCode}}' payment-service
```

Retrieve multiple values:

```bash
docker inspect --format 'status={{.State.Status}} exit={{.State.ExitCode}} restarts={{.RestartCount}}' payment-service
```

Example:

```text
status=exited exit=1 restarts=0
```

Docker uses Go template syntax for these formatting expressions.

Field names and capitalization matter.

### Understanding JSON Output

Some configuration fields are arrays rather than simple strings.

For example, a command might be represented as:

```json
["serve", "--config", "/app/config.yaml"]
```

Use the `json` template function to preserve the structure:

```bash
docker inspect --format '{{json .Config.Cmd}}' payment-service
```

You can also inspect network attachments:

```bash
docker inspect --format '{{json .NetworkSettings.Networks}}' payment-service
```

A container can be attached to multiple networks.

Its IP address belongs to a particular network, so always consider the network name when interpreting an IP address.

**Why use JSON?**

JSON preserves argument boundaries and nested structures. Joining values into one string can make it difficult to understand which text belongs to which argument.

## 3. Understanding docker logs

Applications running inside containers commonly write messages to:

- Standard output (`stdout`).
- Standard error (`stderr`).

Docker can capture these streams through the configured logging mechanism.

To retrieve the output:

```bash
docker logs payment-service
```

This is useful when:

- An application fails to start.
- A container exits unexpectedly.
- A service reports an error.
- You need to understand what happened before a failure.

### Useful Log Options

Show the last 50 lines:

```bash
docker logs --tail 50 payment-service
```

Show logs from the last 10 minutes:

```bash
docker logs --since 10m payment-service
```

Follow new log output:

```bash
docker logs -f payment-service
```

The `-f` option continuously follows new log messages.

Pressing `Ctrl+C` stops following the logs. It does not stop the container.

### Logs Are Not Every Application Log

An important distinction:

`docker logs` does not automatically search every file inside the container.

For example, suppose an application writes logs to:

```text
/var/log/app.log
```

Those messages will not automatically appear through `docker logs`.

The configured logging driver also matters. Some logging drivers do not support local log retrieval in the same way, and log rotation or retention can remove older output.

Remember:

```text
No output from docker logs
          ≠
No application error
```

It only means that the selected logging retrieval path returned no output for that query.

### Can You Read Logs from an Exited Container?

Yes.

If the container still exists and its logging configuration retains accessible output, you can read its logs even after the main process exits.

This is one reason not to remove a failed container before investigating it.

## 4. Understanding docker exec

Sometimes logs and inspection are not enough.

You may need to run a command inside the container to investigate the environment.

Docker provides:

```bash
docker exec CONTAINER COMMAND [ARG...]
```

For example:

```bash
docker exec payment-service printenv APP_ENV
```

This starts a new process inside the running container.

### Open an Interactive Shell

If the image contains a shell, you can use:

```bash
docker exec -it payment-service sh
```

Or, if Bash is available:

```bash
docker exec -it payment-service bash
```

The important point is that `docker exec` creates a new process.

Conceptually:

```text
Container is already running
          |
          v
docker exec -it payment-service sh
          |
          v
New shell process starts
          |
          v
You interact with the shell
          |
          v
You type exit
          |
          v
Only the exec shell exits
```

The main application continues running as long as PID 1 remains alive.

### docker exec Is Not SSH

This is a common misunderstanding.

SSH normally requires:

- An SSH server.
- A network connection.
- Authentication.
- A remote login session.

Docker exec works differently.

```text
docker exec
     |
     v
Docker daemon
     |
     v
Starts a new process inside the existing container
```

It does not require an SSH server or a network login.

Access to the Docker daemon is powerful because it allows an operator to start processes inside containers.

### A Container Might Not Have a Shell

Minimal container images often do not include:

- Bash.
- Sh.
- Curl.
- Ps.
- Other diagnostic utilities.

For example:

```bash
docker exec -it payment-service bash
```

may fail because Bash is not installed.

That does not automatically mean the container is broken.

Prefer running a focused command when possible:

```bash
docker exec payment-service printenv APP_ENV
```

Avoid installing debugging tools into production containers unnecessarily. Changing the running container can make it different from the image and weaken reproducibility.

### Important Limitation

`docker exec` requires the container to be running.

You cannot use it to start a process inside an exited container.

For an exited container, investigate its retained state and logs instead.

---

## 5. Which Tool Should You Use?

| Question | Command |
|---|---|
| What state did Docker record? | `docker inspect` |
| What exit code was recorded? | `docker inspect` |
| What command is configured? | `docker inspect` |
| What did the application write to stdout or stderr? | `docker logs` |
| How do I follow new log messages? | `docker logs -f` |
| How do I run a command inside a running container? | `docker exec` |

A useful troubleshooting approach is:

```text
OBSERVE
   |
   v
Inspect state and configuration
   |
   v
CORRELATE
   |
   v
Compare logs with the reported problem
   |
   v
INTERACT IF NEEDED
   |
   v
Run a focused command using exec
   |
   v
CHANGE
   |
   v
VERIFY
```

Reading evidence first helps avoid unnecessary restarts and changes.

## 6. Common Mistakes

### Mistake 1: Treating inspect as a Health Check

A `running` status means the main process is running.

It does not prove that:

- The application is ready.
- Dependencies are available.
- Requests are succeeding.
- Customers can reach the service.

### Mistake 2: Assuming docker logs Shows Every Log File

Docker logs retrieves captured process output through the configured logging mechanism.

It does not automatically read every log file inside the container.

### Mistake 3: Treating exec as SSH

`docker exec` starts a new process through the Docker daemon. It is not a remote SSH login.

### Mistake 4: Trying exec on an Exited Container

`docker exec` requires a running container.

Use `inspect` and `logs` to investigate an exited container.

---

## Episode 2: Important Takeaways

- `docker inspect` retrieves structured configuration and state.
- `--format` extracts specific values.
- `{{json ...}}` preserves arrays and nested structures.
- `docker logs` retrieves captured stdout and stderr.
- `docker logs -f` follows new log output.
- Logs can remain available after a container exits.
- `docker exec` starts a new process inside a running container.
- Exiting an exec shell does not normally stop the main application.
- `docker exec` is not SSH and does not require an SSH server.

---

# Episode 3: Container Recovery, Termination, and Restart Policies

## 1. Recovery or Evidence Destruction?

Imagine the `auth-service` container becomes unhealthy during a production incident.

An operator wants to immediately kill and remove it, then create a fresh container.

But doing so can destroy useful evidence.

Before taking action, ask:

- What state is the container in?
- What evidence should be preserved?
- Does the application support graceful shutdown?
- Should the existing container be restarted or replaced?
- How will the final service outcome be verified?

Docker provides different lifecycle commands for different purposes.

## 2. Understanding Container Lifecycle Commands

| Command | Purpose |
|---|---|
| `docker start` | Starts an existing stopped container. |
| `docker stop` | Requests graceful termination. |
| `docker restart` | Stops and starts the same container. |
| `docker kill` | Sends a signal to the main process; defaults to SIGKILL. |
| `docker rm` | Removes a stopped container. |

### Start an Existing Container

Suppose a container has exited:

```bash
docker start payment-service
```

This starts the existing container again.

It reuses:

- The same container identity.
- The existing configuration.
- The existing writable layer.

It does not create a new container from the image.

Conceptually:

```text
BEFORE:
payment-service → exited

COMMAND:
docker start payment-service

AFTER:
payment-service → running
```

If the original problem was caused by incorrect configuration, starting the same container may reproduce the same failure.

### Restart an Existing Container

```bash
docker restart payment-service
```

For a running container, Docker performs:

```text
running → exited → running
```

It stops and starts the same container.

A restart can help recover from a temporary problem, but it also changes process state and timestamps.

Before restarting during an incident, preserve any evidence you may need.

**Important:** `docker restart` does not create a new container from the image.

## 3. Understanding Graceful Shutdown: docker stop

When you stop a container, Docker normally gives the main process an opportunity to shut down gracefully.

```bash
docker stop payment-service
```

By default, Docker:

1. Sends SIGTERM to the container's main process.
2. Waits for the configured stop timeout.
3. Sends SIGKILL if the process has not exited.

The default timeout is 10 seconds.

Conceptually:

```text
docker stop
     |
     v
Send SIGTERM
     |
     v
Wait for graceful shutdown
     |
     +---- Process exits ----> Container exits
     |
     +---- Timeout expires
                 |
                 v
             Send SIGKILL
                 |
                 v
             Container exits
```

### What Is SIGTERM?

SIGTERM is a termination signal.

It gives the application an opportunity to handle the shutdown request.

A well-designed application can use this time to:

- Stop accepting new requests.
- Finish or reject ongoing requests safely.
- Flush buffered output.
- Close network connections.
- Release resources.

### Configure the Stop Timeout

Some applications need more than 10 seconds to shut down safely.

You can configure the timeout:

```bash
docker stop --time 30 payment-service
```

Docker now waits up to 30 seconds before sending SIGKILL.

The correct timeout depends on the application's actual shutdown behavior.

### The Stop Signal Can Be Configured

SIGTERM is the normal default, but Docker also supports a configured stop signal.

Image `STOPSIGNAL` metadata or container settings can change which signal is used.

Therefore, do not assume every container always receives SIGTERM.

### PID 1 Must Handle the Signal Correctly

A process may not shut down properly if the entrypoint or shell wrapper does not forward signals to the actual application.

For example:

```text
Docker sends signal
       |
       v
Container PID 1
       |
       v
Application process
```

If the main process ignores the signal or does not forward it correctly, Docker may eventually use SIGKILL.

This is why proper process management and foreground execution matter.

## 4. Understanding docker kill

Sometimes a process does not terminate, or you intentionally need to test abrupt termination.

Docker provides:

```bash
docker kill payment-service
```

By default, Docker sends SIGKILL to the container's main process.

Unlike SIGTERM, SIGKILL cannot be caught or handled by the process.

The application does not receive a graceful cleanup opportunity.

### Send a Different Signal

You can also specify another signal:

```bash
docker kill --signal SIGHUP payment-service
```

The effect depends on how the application handles that signal.

Do not send signals randomly in production.

### Stop vs Kill

| Feature | `docker stop` | `docker kill` |
|---|---|---|
| Default signal | SIGTERM | SIGKILL |
| Graceful shutdown opportunity | Yes | No |
| Timeout | 10 seconds by default | No graceful timeout by default |
| Can force termination | Yes, after timeout | Yes, immediately by default |
| Typical purpose | Normal shutdown | Abrupt termination or signal testing |

Use `docker stop` for normal operational shutdown.

Use `docker kill` when an immediate signal is required and you understand the consequences.

## 5. Understanding Container Exit Codes

When a container exits, Docker records an exit code.

An exit code helps identify how the main process ended, but it does not always explain the complete reason.

| Exit code | Common interpretation |
|---|---|
| `0` | Process reported successful completion. |
| `1` | Generic application failure by convention. |
| `125` | Docker failed before it could run the requested container command. |
| `126` | Command was found but could not be invoked. |
| `127` | Command was not found. |
| `137` | Commonly associated with SIGKILL (`128 + 9`). |
| `143` | Commonly associated with SIGTERM (`128 + 15`). |

### Understanding 125, 126, and 127

These codes help distinguish Docker or command-start problems.

```text
125 → Docker could not perform the run request

126 → Command exists but cannot be invoked correctly

127 → Command could not be found
```

### Understanding 137 and 143

Linux conventionally represents a process terminated by a signal using:

```text
128 + signal number
```

For example:

```text
SIGKILL = 9

128 + 9 = 137
```

And:

```text
SIGTERM = 15

128 + 15 = 143
```

Therefore:

- Exit code `137` commonly suggests SIGKILL.
- Exit code `143` commonly suggests SIGTERM.

But do not treat these codes as complete proof of the cause.

For example, exit code 137 does not automatically prove that the container was killed because of an out-of-memory condition.

Possible causes include:

- An explicit kill operation.
- A stop timeout expiring.
- Memory pressure.
- Another external termination action.

Similarly, an application may handle SIGTERM and exit with its own code.

### Investigate Exit Codes with Other Evidence

Use:

```bash
docker inspect --format '{{.State.ExitCode}}' payment-service
```

Then correlate the result with:

- Container state.
- Start and finish timestamps.
- Application logs.
- Restart count.
- Docker daemon events.
- Host-level evidence.
- Application shutdown behavior.

Remember:

> An exit code gives you a clue. Correlated evidence helps establish the cause.

## 6. Understanding docker rm

When you no longer need a stopped container, you can remove it.

```bash
docker rm payment-service
```

Removing a container deletes:

- The container object.
- Its container identity.
- Its writable layer.

It does not remove the image.

It also does not normally delete separately managed volumes.

### Why Should You Be Careful?

Consider:

```text
payment-service → exited
```

At this point, you can still:

- Inspect its configuration.
- Read available logs.
- Check its exit code.
- Start the same container again.

But after:

```bash
docker rm payment-service
```

That container identity and its writable layer are gone.

Use a deliberate sequence:

```text
INSPECT
   |
   v
Confirm identity, state, logs, and mounts
   |
   v
CONFIRM
   |
   v
Make sure the container is no longer required
   |
   v
REMOVE
   |
   v
VERIFY
```

Docker normally refuses to remove a running container.

Force-removal options exist, but they combine termination and removal. Avoid using them blindly during an incident because they can increase the impact and destroy useful evidence.

## 7. Understanding Docker Restart Policies

What happens if a container exits unexpectedly?

Do you need to manually start it every time?

Docker provides restart policies that allow the Docker daemon to attempt automatic recovery after a container exits.

The general syntax is:

```bash
docker run --restart POLICY IMAGE
```

Restart policies include:

| Policy | Behavior |
|---|---|
| `no` | Do not automatically restart. This is the default. |
| `on-failure` | Restart when the container exits with a non-zero exit code. |
| `always` | Restart after an exit, including after a daemon restart, subject to manual-stop behavior. |
| `unless-stopped` | Similar to `always`, but a manually stopped container remains stopped across daemon restarts. |

### Policy 1: no

```bash
docker run --restart no nginx
```

Docker does not automatically restart the container after it exits.

This is the default behavior.

### Policy 2: on-failure

```bash
docker run --restart on-failure nginx
```

Docker attempts to restart the container when its main process exits with a non-zero exit code.

You can also specify a retry limit:

```bash
docker run --restart on-failure:5 nginx
```

This tells Docker to attempt a limited number of restarts after failures.

It is useful when you want to avoid unlimited restart attempts.

### Policy 3: always

```bash
docker run --restart always nginx
```

Docker attempts to restart the container whenever it exits.

It also starts the container again when the Docker daemon restarts.

However, there is an important exception.

If an operator manually stops the container, Docker does not immediately restart it against that manual stop request.

When the Docker daemon restarts, the `always` policy can start the container again.

### Policy 4: unless-stopped

```bash
docker run --restart unless-stopped nginx
```

This behaves similarly to `always`, with one important difference.

If an operator manually stops the container, it remains stopped even after the Docker daemon restarts.

### always vs unless-stopped

This is one of the most important differences to understand.

| Situation | `always` | `unless-stopped` |
|---|---|---|
| Container exits unexpectedly | Restart attempted | Restart attempted |
| Docker daemon restarts | Restart attempted | Restart attempted unless manually stopped |
| Operator manually stops container | Remains stopped for now | Remains stopped |
| Docker daemon restarts after manual stop | Starts again | Remains stopped |

Example:

An operator stops a maintenance container:

```bash
docker stop maintenance-service
```

The Docker daemon is then restarted.

- With `always`, Docker can start the container again.
- With `unless-stopped`, Docker preserves the operator's stopped intent.

### Restart Policy Is Not Health Management

A restart policy responds to container process exits observed by the Docker daemon.

It does not automatically understand whether:

- The application is responding correctly.
- A database is available.
- An API is healthy.
- Customers can access the service.
- The application is stuck but still running.

For example:

```text
Bad configuration
       |
       v
Application starts
       |
       v
Application exits
       |
       v
Restart policy starts it again
       |
       v
Same bad configuration
       |
       v
Application exits again
```

This can create a restart loop.

You can check the restart count:

```bash
docker inspect --format '{{.RestartCount}}' payment-service
```

A high restart count indicates repeated restarts. It is evidence of instability, not proof of resilience.

### Restart Policies and --rm

These two options have conflicting purposes.

```text
--rm
  → Remove the container after it exits

Restart policy
  → Keep the container available for automatic restarting
```

Docker does not allow combining automatic removal with a restart policy.

---

## 8. A Practical Production Workflow

Consider a production container that is not responding.

Instead of immediately restarting or removing it, follow a controlled process.

### Step 1: Identify the Container

```bash
docker ps -a
```

Confirm the correct container name and current state.

### Step 2: Inspect Its State

```bash
docker inspect payment-service
```

Check:

- Status.
- Exit code.
- Start and finish timestamps.
- Restart count.
- Image.
- Startup configuration.
- Restart policy.

### Step 3: Read Its Logs

```bash
docker logs --tail 100 payment-service
```

Look for application errors around the time the problem started.

### Step 4: Decide What Action Is Required

Choose based on the evidence:

| Situation | Possible action |
|---|---|
| Container is stopped and should run again | `docker start` |
| Running container needs a controlled restart | `docker restart` |
| Container needs graceful shutdown | `docker stop` |
| Immediate termination is required | `docker kill` |
| Stopped container is no longer needed | `docker rm` |

### Step 5: Verify the Result

Check the container state:

```bash
docker ps -a
```

Inspect it again:

```bash
docker inspect --format 'status={{.State.Status}} exit={{.State.ExitCode}} restarts={{.RestartCount}}' payment-service
```

Finally, verify that the actual application responds.

A running container alone does not prove that the service is working.

---

## Episode 3: Important Takeaways

- `docker start` starts an existing stopped container.
- `docker restart` stops and starts the same container.
- `docker stop` normally sends SIGTERM, waits 10 seconds, then sends SIGKILL if required.
- The stop timeout can be configured.
- `docker kill` sends SIGKILL by default.
- Exit codes help investigate failures but do not always prove their cause.
- `docker rm` removes the container object and writable layer.
- Restart policies control automatic restart attempts after container exits.
- `always` and `unless-stopped` differ in how they handle manually stopped containers after a daemon restart.
- Restart policies do not guarantee application health.

---

# Module 2: Final Revision

## 1. Container Lifecycle

```text
created → running → exited → removed
```

- `docker run` creates and starts a container.
- PID 1 determines the container's running lifetime.
- `docker ps` shows running containers.
- `docker ps -a` shows retained containers, including exited ones.
- `--rm` automatically removes a container after it exits.

## 2. Container Investigation

| Command | Purpose |
|---|---|
| `docker inspect CONTAINER` | Read structured configuration and state. |
| `docker inspect --format` | Retrieve specific values. |
| `docker logs CONTAINER` | Read captured stdout and stderr. |
| `docker logs -f CONTAINER` | Follow new log output. |
| `docker exec CONTAINER COMMAND` | Start a new process inside a running container. |

## 3. Container Management

| Command | Purpose |
|---|---|
| `docker start CONTAINER` | Start an existing stopped container. |
| `docker stop CONTAINER` | Request graceful termination. |
| `docker restart CONTAINER` | Stop and start the same container. |
| `docker kill CONTAINER` | Send a signal, SIGKILL by default. |
| `docker rm CONTAINER` | Remove a stopped container. |

## 4. Restart Policies

| Policy | Meaning |
|---|---|
| `no` | No automatic restart. |
| `on-failure` | Restart after a non-zero exit. |
| `always` | Restart after exit and after daemon restart, subject to manual-stop behavior. |
| `unless-stopped` | Like `always`, except a manually stopped container remains stopped across daemon restarts. |

## 5. Important Concepts to Remember

1. A container is not the same thing as an image.
2. A container can exist without running.
3. PID 1 controls the container's running lifetime.
4. Detached mode does not guarantee that a service stays alive.
5. `docker inspect`, `docker logs`, and `docker exec` serve different purposes.
6. `docker exec` creates a new process; it does not replace PID 1.
7. `docker stop` provides a graceful shutdown opportunity.
8. `docker kill` does not provide a graceful shutdown window by default.
9. Removing a container is different from stopping it.
10. Restart policies do not replace proper application health monitoring.

---

# Practical Learning Challenge

Now apply all three episodes in a practical environment.

## Challenge 1: Container Lifecycle

1. Start an Alpine container that prints a message and exits.
2. Check whether it appears in `docker ps`.
3. Use `docker ps -a` to find it.
4. Inspect its exit code.
5. Start the same container again.
6. Run another disposable container using `--rm`.

## Challenge 2: Inspect, Logs, and Exec

1. Start an Nginx container in detached mode.
2. Give it the name `web-service`.
3. Read its configuration using `docker inspect`.
4. Retrieve its status using `--format`.
5. Read its logs.
6. Follow its logs using `docker logs -f`.
7. Execute a command inside the running container.
8. Open an interactive shell if one is available.
9. Exit the shell and verify that Nginx is still running.

## Challenge 3: Stop and Restart

1. Stop the Nginx container.
2. Verify its state using `docker ps -a`.
3. Start the same container again.
4. Restart it.
5. Inspect its restart count.
6. Stop it and remove it deliberately.

## Challenge 4: Restart Policy

Create separate test containers using:

```bash
--restart no
```

```bash
--restart on-failure:5
```

```bash
--restart always
```

```bash
--restart unless-stopped
```

Observe their behavior when:

- The main process exits.
- The container is manually stopped.
- The Docker daemon restarts.

Record what actually happens rather than assuming every policy behaves identically.

---

# Final Thought

Managing containers is not just about memorizing Docker commands.

It is about understanding what state a container is in, what its main process is doing, what evidence Docker has retained, and what impact your next action will have.

A good operator follows this habit:

```text
OBSERVE
   ↓
UNDERSTAND
   ↓
PRESERVE EVIDENCE
   ↓
ACT
   ↓
VERIFY
```

**Do not assume a container is healthy because it is running. Do not assume it is gone because it is missing from `docker ps`. Understand its lifecycle before changing it.
