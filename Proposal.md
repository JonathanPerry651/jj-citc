# Architecture Design: Unified Cloud-Native Monorepo & Build Cache (CitC-Clone)

## 1. Executive Summary
This system provides a globally distributed, virtualized development environment. It eliminates the "clone/download" bottleneck for massive monorepos and the "upload" bottleneck for remote builds. By utilizing a **REAPI-compliant** unified storage backend, the system treats human-authored source code and machine-generated build artifacts as first-class citizens of a single, content-addressed universe.

---

## 2. Infrastructure & Compute Layer

### 2.1 Segregated Kubernetes Environments
* **Developer Pods:** Persistent developer environments accessed via secure SSH tunnels.
    * **Privileges:** Requires `SYS_ADMIN` capabilities for FUSE mounting.
    * **Local SSD:** Uses node-local NVMe for the Metadata Cache (SQLite) and a Blob LRU Cache (`.jj/vfs-cache`).
* **Buildbarn (BB) Worker Pods:** Ephemeral pods dedicated to remote execution.
    * **Identity:** Distinct mTLS Workload Certificates (`bb-cert`) to isolate build-system writes from version-control history.

### 2.2 The Workspace Management Service (Control Plane)
* **Role:** Orchestrates pod lifecycles and "Sparse Profiles" (user-defined working sets).
* **Cold-Boot Hydration:** Pre-emptively pushes metadata snapshots to a new Pod’s SQLite cache before the developer logs in, preventing IDE "scanning" freezes.
* **Stale Archival:** If a pod is dormant >30 days, it force-pushes unpushed Spanner drafts to hidden "backup refs" in the Tier 1 Git repo before GC can delete them.

---

## 3. Storage Strategy: The "Bifurcated REAPI" Model

### 3.1 Content-Addressing & Hashing
* **The Hashing Contract:** All files are stored by their **raw byte SHA-256 hash** (REAPI Digest).
* **VCS Header Stripping:** The FUSE daemon explicitly strips Git-style headers (`blob <size>\0`) before hashing to ensure that a file saved by a human results in the exact same Digest that Bazel/Buildbarn expects.

### 3.2 The Write Path (Workspace Write Gateway)
* **Small Files (<1MB):** Inlined directly into the gRPC `CommitTree` request. The Gateway writes metadata + raw bytes into a single Spanner row.
* **Large Files (>=1MB):** FUSE requests a GCS Signed URL. The file streams directly to GCS.
* **Synchronous Validation:** The Gateway opens a lazy stream to the new GCS blob, computes the hash inline, and only commits to Spanner if the hash matches. This prevents "CAS poisoning" without the complexity of async workers.

### 3.3 The Read Path (Asymmetric Access)
* **Optimization:** FUSE reads directly from Spanner/GCS via Workload Identity to minimize latency, bypassing the Gateway.
* **Schema Migrations:** Since reads are client-side, migrations use a "Fat Client" rollout (N/N-1 read support) followed by a Gateway write-flip and background Spanner backfill.

---

## 4. The FUSE Layer: Namespace & Conflict Logic

### 4.1 Specialized Daemons
* **Developer FUSE:** Integrated with `jj-lib`. It performs **Eager Evaluation**, intercepting every `fsync`/`release` to update the implicit `jj` workspace commit in Spanner.
* **Buildbarn FUSE:** A high-throughput "Execution Root" daemon. It projects Bazel Merkle Trees into a virtual directory for the compiler. It has no knowledge of version control.

### 4.2 POSIX Emulation & IDE Support
* **`inotify` Synthesis:** Since background updates (like a `jj fetch`) happen in Spanner, the FUSE daemon must manually inject `inotify` events into the kernel so IDEs (VS Code) refresh their file buffers.
* **Conflict Materialization:** `jj` stores conflicts as graph objects. On `read()`, FUSE forces `jj-lib` to render Git-style markers (`<<<<<<<`) in memory. On `write()`, FUSE parses the buffer to confirm markers are gone before resolving the conflict in Spanner.

---

## 5. Build System Integration: "Zero-Upload" Builds

1.  **Developer Save:** Human saves `main.cc`. FUSE writes the REAPI-compliant blob to Spanner/GCS.
2.  **Bazel Request:** Bazel sends an Action Digest to Buildbarn.
3.  **Execution:** Because the blob already exists in the unified backend (from the save), the BB Worker instantly "sees" the file in its virtual Execution Root. **Network upload is zero.**
4.  **BwoB (Build without the Bytes):** Bazel leaves symlinks to CAS hashes in the developer's `bazel-bin`. FUSE resolves these hashes on-demand from the global CAS when the dev executes the binary.

---

## 6. Unified Garbage Collection (GC)

### 6.1 Shared-Fate Reference Counting
We no longer distinguish between "Source" and "Build" GC. A blob is preserved if it is referenced by **any** active root.
* **VCS Roots:** Current workspace commits, Upstream Git Refs, and `jj` Operation Logs (30-day TTL).
* **Build Roots:** All blobs referenced by unexpired Action Cache (AC) entries (14-day TTL).

### 6.2 The Safety Protocol
* **Referential Integrity:** A CAS blob is never deleted if an Action Cache record still points to it (preventing "AWOL jars").
* **The Grace Period:** Objects are only eligible for deletion if they are unreachable from *all* roots AND are older than 30 days.

---

## 7. Security & Access Matrix

| Role | VCS Metadata (`/src`) | Build Metadata (AC) | Blobs (CAS) |
| :--- | :--- | :--- | :--- |
| **Developer** | Read / Write | Read-Only | Read / Write (via Gateway) |
| **BB Worker** | Read-Only | Read / Write | Read / Write (via Gateway) |

* **Logic:** Developers cannot poison the global Build Cache (AC). BB Workers cannot corrupt the human version history (`jj`).
