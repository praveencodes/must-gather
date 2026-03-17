# `oc adm inspect` options and must-gather (time/size)

This document lists the options supported by `oc adm inspect` and how they can be used to reduce must-gather time and size.

## Options supported by `oc adm inspect`

From `oc adm inspect --help`:

| Option | Description | Relevance to time/size |
|--------|-------------|------------------------|
| **`--since=DURATION`** | Only return logs newer than a relative duration (e.g. `5s`, `2m`, `3h`). Default: all logs. | **Reduces size and time** by limiting log volume. |
| **`--since-time=TIMESTAMP`** | Only return logs after a specific date (RFC3339). Default: all logs. | **Reduces size and time** by limiting log volume. |
| **`--dest-dir=DIR`** | Root directory for gathered data. Default: `$(PWD)/inspect.local.<rand>`. | Must-gather sets this to `must-gather`; no tuning. |
| **`-A` / `--all-namespaces`** | List requested object(s) across all namespaces. | Used where needed; scoping *fewer* resources reduces size/time. |
| **`-n` / `--namespace=NS`** | Namespace scope for the request. | Used by many gather_* scripts; scoping to one NS reduces size/time. |
| **`--request-timeout=DURATION`** | Time to wait before giving up on a single server request (e.g. `1s`, `2m`, `3h`). `0` = no timeout. | Can **reduce hang time** on stuck requests; does not reduce total data size. |
| **`--disable-compression`** | Opt-out of response compression. | **Increases** size/bandwidth; avoid for size reduction. |
| **`--events-file=PATH`** | Path to an events.json file to create an HTML page from. | Output/display only; not used for size reduction. |
| **`--cache-dir`**, **`--kubeconfig`**, **`--server`**, **`--token`**, **`--certificate-authority`**, **`--client-certificate`**, **`--client-key`**, **`--insecure-skip-tls-verify`**, **`--tls-server-name`**, **`--as`**, **`--as-group`**, **`--as-uid`**, **`--cluster`**, **`--context`**, **`--user`** | Standard auth/connection options. | Not used to reduce time/size. |

**Note:** The main gather script and some collection scripts (e.g. `gather_vsphere`) use **`--rotated-pod-logs`**. This flag may not appear in `oc adm inspect --help` on all versions; it limits pod log collection to rotated logs and helps keep size smaller.

---

## What must-gather already uses

- **`--since` / `--since-time`**  
  The image uses `common.sh` → `get_log_collection_args()`, which sets `log_collection_args` from:
  - **`MUST_GATHER_SINCE`** → `--since=<value>` (e.g. `8h`, `2h30m`)
  - **`MUST_GATHER_SINCE_TIME`** → `--since-time=<RFC3339>`  

  These are passed through to every `oc adm inspect` call that uses `${log_collection_args}` (see `collection-scripts/gather`, `gather_metallb`, `gather_network_logs_basics`, `gather_olm_v1`, etc.). So **time-based log limiting is already wired for inspect**.

- **`--rotated-pod-logs`**  
  Used in the main `gather` script and in `gather_vsphere` to limit pod log data.

- **`--dest-dir`**  
  Set to `must-gather` (or `BASE_COLLECTION_PATH` in helper scripts).

- **`--all-namespaces`** / **`-n`**  
  Used where the design requires cluster-wide or namespace-scoped collection.

---

## Leveraging options to reduce time and size

### 1. **Use `--since` / `--since-time` (recommended)**

- **From the client (OpenShift 4.16+):**
  ```bash
  oc adm must-gather --since=24h
  # or
  oc adm must-gather --since-time=$(date -u -d '-24 hours' +%Y-%m-%dT%H:%M:%SZ)
  ```
  The client passes these into the must-gather pod as `MUST_GATHER_SINCE` / `MUST_GATHER_SINCE_TIME`; the image’s `get_log_collection_args()` then adds `--since` or `--since-time` to `oc adm inspect` (and `node_log_collection_args` for `oc adm node-logs`). This **reduces log volume and often runtime**.

- **Older versions (e.g. 4.15):**  
  If the client does not set these env vars, you can force time filtering inside the image, e.g.:
  ```bash
  oc adm must-gather -- "sed -i 's#oc adm inspect#oc adm inspect --since=24h#g' /usr/bin/*gather* ; /usr/bin/gather"
  ```
  (Use only when you understand the risk of modifying scripts on the fly.)

### 2. **`--request-timeout`**

- Not currently used in this repo’s scripts. You could add it to `oc adm inspect` (e.g. `--request-timeout=5m`) so that a single stuck request does not block the whole gather. This **reduces wall-clock time** in bad conditions but does not shrink output size.

### 3. **Scoping resources**

- **Inspect only what you need:**  
  Running `oc adm inspect` for a subset of resources (e.g. one cluster operator or one namespace) reduces both time and size. The default must-gather image intentionally collects a broad set; for targeted debugging, run inspect (or a custom gather) only for the relevant types/namespaces.

- **Optional gather scripts:**  
  The main `gather` script launches many optional `gather_*` scripts (OLM, etcd, network, metallb, sriov, etc.). Using a custom image or entrypoint that runs only the scripts you need would reduce time and size.

### 4. **Avoid `--disable-compression`**

- Leaving compression on (default) keeps network and disk usage lower; do not set `--disable-compression` if the goal is to reduce time/size.

### 5. **Storage limit (client-side)**

- `oc adm must-gather` supports **`--volume-percentage`** (e.g. to raise the 30% default). That does not change `oc adm inspect` behavior but can prevent the gather from stopping early when the pod volume fills; combined with `--since`/scoping, it can make a large gather finish instead of failing.

---

## Summary

| Goal | Leverage |
|------|----------|
| **Reduce size** | Use **`--since`** or **`--since-time`** (via `oc adm must-gather --since=...` on 4.16+). Keep **`--rotated-pod-logs`** where already used. Scope to fewer resources when possible. |
| **Reduce time** | Same **`--since`**/`--since-time` (less data to pull). Optionally add **`--request-timeout`** to avoid indefinite hangs. Run only needed gather scripts or inspect only needed resources. |
| **Avoid increasing size** | Do not use **`--disable-compression`** for size reduction. |

The main lever already supported by the must-gather image is **time-bounded log collection** via **`MUST_GATHER_SINCE`** / **`MUST_GATHER_SINCE_TIME`**, which map to **`oc adm inspect`’s `--since` and `--since-time`**.
