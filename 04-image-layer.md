# Docker Module 3: Image Layers, Storage Accounting & Build Cache

Comprehensive engineering notes covering image identity, layered filesystems, runtime writable diffs, daemon storage accounting, and build cache optimization.

---

## Episode 1: Image Identity, Layers, Inspection & History

### The Core Problem: "Both Hosts Say `latest`"
When `payment-service` behaves differently on `server-01` and `server-02` despite both deployment notes referencing `payment-service:latest`, the assumption that both hosts run the exact same artifact is unsupported. `latest` is merely a movable tag, and tags can point to different image content at different times or across different Docker daemons.

To investigate an image discrepancy, always separate three distinct questions:
1. **What packaged content exists?** (Layers and filesystem contents)
2. **Which local image object is present?** (Local configuration digest / Image ID)
3. **Which reference was used to find it?** (Tag vs. immutable registry digest)

---

### 1. An Image Is a Layered Starting Point
An image is packaged user-space content and configuration assembled from **read-only layers**.

* **Layer:** An immutable filesystem change set that can be reused by multiple images. Docker merges the selected layers into a single unified visible image filesystem.
* **Base Image:** The starting image on which another image is built. It can supply an OS user space, a language runtime, or a minimal application foundation. "Base" describes its role in the build relationship—it does **not** mean every image contains a full operating system or a private kernel.
* **Content-Addressed Reuse:** Because layers are content-addressed and read-only after creation, multiple images can share identical underlying layers locally.

```text
shared base layers
       ├── payment-service layer
       └── checkout-service layer
```

> **Storage Driver Rule:** The storage driver controls how layers are represented on the host. Never manually edit Docker's internal storage directory or build automation that depends on its internal paths.
>
> **Disk Accounting Note:** Because unchanged base layers are shared locally on the same daemon, image counts and displayed image sizes **cannot** simply be summed to determine unique disk usage.

---

### 2. Separate Tags, Image IDs, and Digests
Different image identifiers answer fundamentally different operational questions:

| Identifier Type | Example | Mutability | What It Identifies |
| :--- | :--- | :--- | :--- |
| **Tag** | `payment-service:1.4`<br>`payment-service:latest` | **Movable** | A human-friendly pointer inside a repository namespace. Multiple tags can point to one image, and a tag can be reassigned later. |
| **Image ID** | `8a2f31c4d1ab` (short)<br>`sha256:...` (full) | **Immutable** (for that config) | A content digest of the **local image configuration object**. Identifies a specific local image on a daemon. |
| **Registry Digest** | `registry.example.com/infrathrone/payment-service@sha256:...` | **Immutable** | Exact published content in a registry (specific manifest or multi-platform image index). |

#### Key Nuances:
* **The `latest` Tag:** If no tag is written in a standard image reference, Docker defaults to `latest`. This does **not** make it the newest, most stable, tested, or special build:
  * `latest` = ordinary tag named `"latest"`
  * `latest` $\neq$ automatic proof of newest content
* **Matching Local IDs:** If `docker image ls` shows two rows with different tags (`1.4` and `stable`) but the exact same `IMAGE ID` (`8a2f31c4d1ab`), both tags point to **one** single local image configuration.
* **Image ID vs. Registry Digest:** A local Image ID and a registry manifest digest identify different objects (local config vs. registry manifest/index) and will legitimately have different hash values even for the exact same release. When a deployment requires exact published content, pin by **registry digest**.

---

### 3. Inspect Structured Image Evidence
Use `docker image inspect IMAGE` to examine the image object itself (unlike container inspection, which inspects a runtime instance).

#### Key Inspection Fields:
* `.Id` $\rightarrow$ Full local image ID (config digest)
* `.RepoTags` $\rightarrow$ Local `repository:tag` references pointing to this image
* `.RepoDigests` $\rightarrow$ Known repository digest references (`repo@sha256:...`)
* `.Created` $\rightarrow$ Image creation timestamp
* `.Architecture` $\rightarrow$ Target CPU architecture (e.g., `amd64`, `arm64`)
* `.Os` $\rightarrow$ Target operating system (e.g., `linux`)
* `.Config` $\rightarrow$ Runtime defaults stored in image metadata (entrypoint, cmd, env, user, ports)
* `.RootFS.Layers` $\rightarrow$ Content digests for the underlying filesystem layers
* `.Size` $\rightarrow$ Total image size reported by the daemon

#### Focused Evidence Extraction:
```bash
docker image inspect --format 'id={{.Id}} tags={{json .RepoTags}} digests={{json .RepoDigests}}' payment-service:1.4
```

> **Operational Notes:**
> * A locally built image that has never been pushed to or pulled from a registry by digest will have an empty `.RepoDigests` array (`[]`). This is normal and does not mean the image lacks an ID or layers.
> * **Security Hygiene:** Raw `.Config` output often contains default environment variables or metadata. Never paste raw inspect dumps into public incident channels without reviewing for sensitive data first.

---

### 4. Read the Construction Trail with `docker history`
Use `docker history IMAGE` (or `docker history --no-trunc IMAGE` to prevent command truncation) to view the image's construction records ordered **newest first**, showing the command description and attributed byte size for each entry.

#### Questions `docker history` Answers:
* Which build step introduced a large filesystem change?
* Which entries contribute zero filesystem bytes (metadata-only instructions)?
* Where should an image-size investigation begin?

#### Important Caveats:
* **Not Perfect Source Reconstruction:** Build metadata may abbreviate or normalize commands, and BuildKit can produce metadata-only entries.
* **Never Rely on History to Hide Secrets:** Even if a history line looks brief or a secret was deleted in a later step, secrets baked into build steps are never safely hidden.
* **History Size vs. Daemon Storage:**
  * *History entry size* = filesystem change attributed to a single image build entry.
  * *Local daemon storage* = actual disk footprint, where layers may be shared across multiple images.

---

### 5. Understand Untagged and Dangling Images
When a tag (like `latest`) is reassigned to a newly built image, the previous image remains on the local daemon without a tag. An image with **no tag** and **no child image** depending on it is called a **dangling image**.

* **Docker Engine 29 Behavior Change:**
  * `docker image ls` **hides dangling images by default** starting in Docker Engine 29.
  * Pre-Engine 29 versions displayed dangling images (`<none>:<none>`) by default. Always record the Docker Engine version during investigations.
  * To list all local images including dangling ones:
    ```bash
    docker image ls --all
    # or
    docker image ls -a
    ```
* **Safety Rule:** An untagged image is not automatically safe to delete—it may still hold valuable incident evidence or be referenced by ID/digest.

---

### Episode 1 Summary: Traps, Habits & Interview Prep

#### Common Traps
1. **Treating `latest` as a Release Policy:** It provides zero ordering, testing, or immutability guarantees.
2. **Calling Every Hash "the Digest":** Always label whether a hash is a local Image ID, layer digest, or registry manifest digest.
3. **Adding Image Sizes to Estimate Disk Use:** Summing displayed sizes double-counts shared base layers.
4. **Assuming Default `docker image ls` Shows Everything:** On Engine 29+, dangling images are hidden unless `--all` (`-a`) is passed.

#### Production Habit: Standardized Evidence Label Block
```text
REFERENCE          -> repository, tag, or digest requested
LOCAL ID           -> full image ID on the selected daemon
PUBLISHED IDENTITY -> repository digest when available
PLATFORM           -> operating system and architecture
HISTORY LEAD       -> entry associated with unexpected bytes
```

#### Quick Interview & Review Checks
* **Q: What is the difference between an image ID, tag, and digest?**
  * *A:* A **tag** is a movable repository pointer; an **image ID** is the immutable local content digest of the image's configuration object; a **registry digest** is an immutable reference to published content (manifest/index) in a registry.
* **Q: Why can two images occupy less disk space than the sum of their displayed sizes?**
  * *A:* Because image layers are immutable and content-addressed, allowing multiple images on the same daemon to share identical underlying layers.
* **Q: Is `latest` always the newest image?**
  * *A:* No. It is simply an ordinary tag whose target depends entirely on whichever image was last tagged or pushed as `latest`.

---
---

## Episode 2: Writable Container Layers, Whiteouts & Storage Accounting

### The Core Problem: "The File Is Gone, but the Image Is Still Huge"
A `checkout-service` build downloads a large archive in one step and deletes it in a subsequent step. Inside a running container, the file is completely invisible—yet the image size remains bloated.

This happens because the team confused the **merged runtime filesystem view** with **physical layer history**. In a layered filesystem, *"not visible now"* does not mean *"never stored in an earlier layer."*

Always separate three distinct storage surfaces:
1. **Image layers** (immutable build artifacts)
2. **Container writable changes** (ephemeral runtime state)
3. **Local daemon storage accounting** (shared vs. unique disk usage)

---

### 1. Add One Writable Layer at Runtime
All image layers are strictly read-only. When Docker starts a container, it mounts a thin, container-specific **writable layer** on top of the read-only image stack:

```text
CONTAINER FILESYSTEM VIEW
┌──────────────────────────────┐
│ writable container layer     │  <- runtime adds, changes, deletes
├──────────────────────────────┤
│ read-only application layer  │
├──────────────────────────────┤
│ read-only dependency layer   │
├──────────────────────────────┤
│ read-only base image layers  │
└──────────────────────────────┘
```

The storage driver merges these layers into a single unified directory tree for processes inside the container:
* **Creating a file:** Written directly into the container's writable layer.
* **Modifying an image file (Copy-on-Write / CoW):** Docker copies the file up into the writable layer and modifies that copy; the underlying read-only image layer remains untouched.
* **Deleting an image file:** Docker records a deletion marker in the writable layer to hide the lower path; the underlying image layer is not rewritten.

> **Isolation Principle:** If Container A modifies an image-provided config file and Container B starts later from the same image (with no shared volume), Container B sees the **original unmodified image file**. Container A's changes exist strictly within A's writable layer.

---

### 2. Read Writable-Layer Evidence with `docker diff`
Use `docker diff CONTAINER` to inspect path-level changes in a container's writable layer compared to its starting image:

```text
C /var
C /var/log
A /var/log/payment-service
A /var/log/payment-service/runtime.log
D /app/default-banner.txt
```

| Marker | Meaning |
| :--- | :--- |
| `A` | A filesystem path was **Added** in the writable layer |
| `C` | A filesystem path **Changed** (file content or directory/file metadata) |
| `D` | A filesystem path was **Deleted** (hidden from the container's visible view) |

#### What `docker diff` Does NOT Do:
* Does **not** identify which Dockerfile instruction created an original file.
* Does **not** map `A`, `C`, or `D` markers to separate read-only image layers.
* Does **not** prove which user or process made the change.
* Does **not** report the byte size of each path change.
* Does **not** provide a chronological audit log of intermediate runtime edits (only current state vs. image start).

> *Note:* Parent directories (like `C /var`) frequently appear as `C` simply because creating or removing a child file updates the parent directory's metadata.

---

### 3. Understand Deletion Across Immutable Layers (Whiteouts)
Because committed image layers are immutable, deleting a file in a subsequent build layer cannot remove bytes from an earlier layer:

```text
Layer 1: add /tmp/large-artifact       +800 MB  (committed & immutable)
Layer 2: hide /tmp/large-artifact      small deletion marker (whiteout)
```

Storage drivers implement this hiding mechanism using a **whiteout**—a special marker file interpreted when merging layers so the upper layer obscures the lower path.

```text
DELETE IN A LATER LAYER
  -> path hidden in the final merged view
  -> earlier layer remains completely unchanged
  -> earlier bytes still ship with and bloat the image
```

* **Build-Time Solution:** Temporary files or package caches must be created and deleted **within the exact same layer/step** before that layer is committed (or isolated in a multi-stage build) so the bytes are never persisted in a retained layer.
* **Runtime Deletion:** Deleting an image file inside a running container only creates a whiteout in that container's writable layer. Removing the container deletes that writable layer without altering the underlying image.

---

### 4. Account for Local Storage with `docker system df`
Use `docker system df` to query the selected Docker daemon for local disk accounting across resource categories:

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          ...       ...       ...       ...
Containers      ...       ...       ...       ...
Local Volumes   ...       ...       ...       ...
Build Cache     ...       ...       ...       ...
```

> **Scope Warning:** `docker system df` reflects **only the selected local Docker daemon**. It is not registry usage, cluster-wide usage, or overall host filesystem usage.

#### Verbose Breakdown (`docker system df --verbose`):
Separates image sizes into **Shared** and **Unique** bytes:
* **SHARED SIZE:** Bytes in layers also referenced by at least one other local image.
* **UNIQUE SIZE:** Bytes attributed exclusively to that specific local image.

```text
Example:
Image A visible size: 600 MB
Image B visible size: 650 MB
Shared between them:  500 MB
-> Total local storage consumed = 600 + (650 - 500) = 750 MB (NOT 1,250 MB)
```

#### Interpreting `RECLAIMABLE` Cautiously:
`RECLAIMABLE` is the daemon's estimate of space used by unreferenced/inactive objects. It is **not** a guarantee that:
1. Every reported byte is safe to delete operationally (e.g., cached base images or forensic containers).
2. Deleting one image will immediately free its full displayed size (shared layers remain until all referencing images are removed).
3. Host OS filesystem free space (`df -h`) will increase by the exact reported byte count (due to compression, storage driver overhead, and content stores).

---

### 5. Six-Step Evidence Sequence for Image Size Incidents
1. **IDENTIFY:** Confirm target daemon, image reference, full local Image ID, and platform (`OS/Arch`).
2. **LOCATE:** Run `docker history --no-trunc` to pinpoint which build step introduced unexpected bytes.
3. **EXPLAIN:** Check if a later step deleted the path, leaving orphaned bytes hidden behind a whiteout in an earlier immutable layer.
4. **ACCOUNT:** Run `docker system df --verbose` to separate unique bytes from shared base layers.
5. **CHANGE:** Fix the reproducible build definition (do **not** patch a running container or delete forensic evidence first).
6. **VERIFY:** Rebuild, test application runtime behavior, and compare new Image ID, `docker history`, and `docker system df` metrics against the baseline.

---

### Episode 2 Summary: Traps, Habits & Interview Prep

#### Common Traps
1. **Treating a Deleted Path as Removed Bytes:** Hiding a file in Layer $N+1$ leaves its bytes intact in Layer $N$.
2. **Reading `docker diff` as Image History:** `A`/`C`/`D` markers only reflect runtime writable-layer state, never build layers.
3. **Summing Every Image's Displayed Size:** Fails to account for shared layers.
4. **Calling Reclaimable Space Guaranteed Free Space:** Treat it as an estimate requiring dependency and operational review.

#### Quick Interview & Review Checks
* **Q: Why does deleting a file in a later image layer not reduce image size?**
  * *A:* Image layers are immutable. Deleting a file in a later layer creates a whiteout marker that hides the path in the merged view, while the earlier layer retains the original file's bytes.
* **Q: What do `A`, `C`, and `D` mean in `docker diff`?**
  * *A:* Added, Changed, and Deleted paths in the container's runtime writable layer relative to its starting image. They do not map to build layers.
* **Q: Why must `docker system df` values be interpreted carefully?**
  * *A:* It is scoped strictly to one local daemon, relies on shared vs. unique layer accounting, and `RECLAIMABLE` is an estimate affected by layer sharing and storage driver internals.

---
---

## Episode 3: Build Cache Boundaries & Reproducible Build Reasoning

### The Core Problem: "One-Line Change Reinstalls Everything"
A developer edits a single line of application source code in `payment-service`, and CI downloads and reinstalls the entire dependency tree from scratch.

Even though dependency manifests didn't change, the Dockerfile placed volatile source code *before* dependency installation. Because a build is a sequential dependency chain, changing an early input invalidates the cache for every downstream step.

**Core Engineering Question:**
> *Which inputs does this step actually depend on, and how often do those inputs change?*

---

### 1. Treat Cache as Proven Reusable Work
An image build is an ordered dependency chain where each step's output becomes the parent state for the next step:

```text
base state
  ↓
step 1 result
  ↓
step 2 result
  ↓
step 3 result
```

* **Cache Hit (`CACHED` in BuildKit):** The builder verifies it has a valid cached result matching the exact instruction, relevant input checksums/metadata, and parent state.
* **Cache Miss (Re-execution):** Any change in the instruction, relevant inputs, or parent state forces the step to execute anew.

> *Note:* BuildKit displays `CACHED` in build logs when a step reuses a cached result. Always look for `CACHED` in build output rather than guessing based on how fast a build felt.

---

### 2. Understand Downstream Invalidation
When any step changes, it produces a new parent state. Every subsequent step receives that altered parent state, invalidating its cache even if its own instruction text and files are completely unchanged:

```text
STEP 1  stable setup        CACHED
STEP 2  application input   CHANGED  <-- invalidation point
STEP 3  dependency work     REBUILT  <-- invalidated due to changed parent state
STEP 4  final preparation   REBUILT  <-- invalidated due to changed parent state
```

**Additional Cache Key Factors:** Beyond visible file contents, cache keys can be affected by file metadata/permissions, `ARG` values, secret/cache mounts, builder configuration, target platform, and base-image digest resolution.

---

### 3. Order Work by Change Frequency and Dependency Boundary
Classify build inputs by their **rate of change** and **true logical dependencies**:

| Change Frequency | Input Category | Examples |
| :--- | :--- | :--- |
| **RARELY** | Base & System Setup | Base image (`FROM`), OS packages, runtime tools |
| **SOMETIMES** | Dependency Declarations | `package.json`, `requirements.txt`, `go.mod`, `pom.xml` |
| **OFTEN** | Application Source Code | `src/`, business logic, templates, unit tests |

#### Broad Ordering Principle:
```text
1. Stable, expensive system/runtime setup   (Changes RARELY)
      ↓
2. Copy dependency manifests & install deps (Changes SOMETIMES)
      ↓
3. Copy application source & build/assemble (Changes OFTEN)
```

* **Correctness Comes First:** Never place an instruction before an input it genuinely needs just to chase cache hits.
* **The Target Invalidation Boundary:** When dependency manifests are unchanged and only application source changes, dependency installation must remain `CACHED` while only source-dependent steps rebuild.

---

### 4. Prove Cache Behavior with Controlled Evidence
Never rely on anecdotal timing (*"the second build felt faster"*). Follow a 4-step controlled proof:

1. **BASELINE BUILD:** Run an initial build and record which steps execute.
2. **CONTROLLED CHANGE:** Modify **only** one input category (e.g., a single source code file).
3. **SECOND BUILD:** Run the build again and record:
   * Which stable steps report `CACHED`.
   * The exact first step that re-executes.
4. **CORRECTNESS CHECK:** Inspect and run the resulting image to prove the source change actually made it into the image and the application behaves properly.

> **Why Correctness Verification Matters:** If *every* step reports `CACHED` after you modified application source code, the build did **not** succeed—it means your changed file was ignored (e.g., via `.dockerignore`), not copied, or excluded from the build context.

---

### 5. Keep Cache Efficiency Separate from Image Size
Build cache efficiency and final image size are independent properties:
* **Cache Question:** *Which build steps can the builder reuse across builds?*
* **Image Size Question:** *Which bytes remain inside the final image's committed layers?*

A cached layer that contains an 800 MB mistake is still an 800 MB layer. Conversely, a completely uncached build (`--no-cache`) can produce a tiny, optimized image.

Use the right tool for each question:
* **Build output (`CACHED`)** $\rightarrow$ Cache reuse
* **`docker history`** $\rightarrow$ Per-step layer size leads
* **`docker image inspect`** $\rightarrow$ Image ID, tags, digests, metadata
* **`docker system df`** $\rightarrow$ Local daemon shared/unique disk accounting
* **Runtime test** $\rightarrow$ Application correctness

---

### 6. Reusable Build Reasoning Model
Evaluate every expensive build step using five prompts:

1. **INPUTS:** What files, flags, and parent state does this step genuinely require?
2. **CHANGE RATE:** Do those inputs change rarely, sometimes, or often?
3. **BOUNDARY:** Can stable inputs be processed before volatile inputs without violating correctness?
4. **EVIDENCE:** Which exact line in the build output proves cache reuse (`CACHED`) or invalidation?
5. **OUTCOME:** Does the final artifact contain the updated code and pass functional verification?

---

### Episode 3 Summary: Traps, Habits & Interview Prep

#### Common Traps
1. **Copying Volatile Input Too Early:** Copying the entire repo (`COPY . .`) before installing dependencies invalidates the dependency cache on every code edit.
2. **Treating Cache Hits as Correctness:** A `CACHED` step proves work was skipped, not that your latest code was included.
3. **Treating Cache Misses as Random:** Misses always stem from an altered input, parent state, base digest, build argument, platform, or missing cache backend.
4. **Confusing Fast Builds with Small Images:** Cache speed only optimizes build execution time; layer design controls final artifact size.

#### Production Habit: Auditable Build Claim Format
```text
Given:
  - unchanged dependency inputs
  - one controlled source-only change
  - the same intended builder context

Observed:
  - stable dependency work returned CACHED
  - source-dependent work rebuilt
  - the resulting service passed its behavior check
```

#### Quick Interview & Review Checks
* **Q: Why does Dockerfile instruction order affect build time?**
  * *A:* Docker builds are sequential dependency chains. Changing an early instruction or file alters the parent state for all downstream steps, invalidating their cache keys. Ordering stable, expensive steps (like dependency installation) before frequently changing source code isolates invalidation.
* **Q: How would you prove that a source-only change did not reinstall dependencies?**
  * *A:* Make a controlled edit strictly to a source file, run the build, verify in the BuildKit output that the dependency installation step shows `CACHED` while the source copy/compile step rebuilds, and run the container to verify the code change is active.
* **Q: Does a fully cached build prove the image is correct?**
  * *A:* No. If you edited code and every step was `CACHED`, the changed file likely wasn't included in the build context or `COPY` instruction. You must verify artifact contents and runtime behavior separately.
