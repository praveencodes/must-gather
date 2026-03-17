# Profiling `oc` commands in must-gather

This directory provides a way to **profile all `oc` invocations** used during gathering: each call is timed and logged so you can see which commands are slow and contribute most to must-gather runtime.

## Quick start

1. Put the profiling wrapper first in `PATH` and set the log file. Then run the main gather script (from the repo root, with a cluster available):

   ```bash
   export PATH="$(pwd)/hack/profile-bin:$PATH"
   export OC_PROFILE_LOG="$(pwd)/oc-profile.log"
   rm -f "$OC_PROFILE_LOG"
   bash collection-scripts/gather
   ```

2. After the run, inspect the log (header line, then one line per `oc` call: start | end | duration | exit | command):

   ```bash
   cat oc-profile.log
   ```

3. Sort by duration to find the slowest calls (skip header with `tail -n +2`):

   ```bash
   tail -n +2 oc-profile.log | sort -t'|' -k3 -rn | head -20
   ```

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| **`OC_PROFILE_LOG`** | `./oc-profile.log` | File to append profile lines to (one line per `oc` invocation). |
| **`OC_REAL_OC`** | `/usr/bin/oc` | Path to the real `oc` binary used when the wrapper is invoked as `oc`. |

## Log format

The first line is a header. Each following line is one `oc` invocation: the first five fields are pipe-separated (start, end, duration, exit code, failure reason), then a tab, then the full command (so the command can contain ` | `). On success (exit 0), the reason field is empty. On failure, the wrapper captures the first line of stderr as the reason (sanitized, up to 200 chars).

```
START | END | DURATION(s) | EXIT | REASON | COMMAND
START_RFC3339 | END_RFC3339 | duration | exit_code | reason	oc subcommand ...
```

Example:

```
START | END | DURATION(s) | EXIT | REASON | COMMAND
2025-02-25T10:00:01Z | 2025-02-25T10:00:15Z |    14.23 | 0 | 	oc adm inspect --since=24h --dest-dir must-gather --rotated-pod-logs clusteroperators
2025-02-25T10:00:20Z | 2025-02-25T10:00:21Z |     1.00 | 1 | Error: the server doesn't have a resource type "badresource"	oc get badresource -A
```

## Coverage

- **Local (PATH wrapper):** Scripts that run `oc` without a path use the wrapper when `hack/profile-bin` is first in `PATH`. Scripts that call `/usr/bin/oc` explicitly are not profiled.
- **Profiling image (operator or in-cluster):** The image built from `Dockerfile.profiling` replaces `/usr/bin/oc` with the wrapper, so **all** `oc` invocations (including from `gather_windows_node_logs`, `gather_vsphere`, etc.) are profiled.

## Running inside the must-gather image (e.g. `oc adm must-gather`)

Build an image that replaces `/usr/bin/oc` with the wrapper (real `oc` at `/usr/bin/oc.real`), set `OC_PROFILE_LOG=/must-gather/oc-profile.log`, run must-gather with that image, then inspect `must-gather/oc-profile.log` in the gathered archive.

---

## Profiling with the must-gather operator

When must-gather is triggered by the **must-gather operator** (not `oc adm must-gather`), the operator creates a pod that runs the must-gather image. To profile all `oc` commands in that run:

### 1. Build the profiling image

From the repo root, build the profiling variant (same as the default image but with the `oc` wrapper installed and the real binary at `/usr/bin/oc.real`):

```bash
podman build -f Dockerfile.profiling -t must-gather-profiling:latest .
# or
docker build -f Dockerfile.profiling -t must-gather-profiling:latest .
```

Push the image to a registry the cluster can pull from:

```bash
podman push must-gather-profiling:latest <your-registry>/must-gather-profiling:latest
```

### 2. Point the operator at the profiling image

Configure the must-gather operator to use this image for the next run. How you do this depends on your operator version and install:

- **Operator configuration / custom resource:** Set the must-gather image to your profiling image (e.g. `spec.image` or the equivalent field in the MustGather / MustGatherRun CR, if your operator supports it).
- **Image stream / global pull spec:** If the operator reads the must-gather image from a ConfigMap, image stream, or cluster-wide setting, update that to `<your-registry>/must-gather-profiling:latest`.

Refer to your operator’s documentation for the exact field or resource name.

### 3. Run must-gather via the operator

Trigger a must-gather run as you normally do (e.g. create a MustGather CR or use the UI/console). Wait for the run to finish and for the operator to produce the archive.

### 4. Read the profile from the archive

After the run, open the must-gather archive (downloaded or attached by the operator). The profile log is at:

```
must-gather/oc-profile.log
```

(or inside the same top-level directory the operator uses for the rest of the gathered data). Each line is one `oc` invocation (tab-separated: start time, end time, duration in seconds, exit code, full command).

**Find the slowest commands:**

```bash
tail -n +2 must-gather/oc-profile.log | sort -t'|' -k3 -rn | head -20
```

### Optional: override the profile log path

The profiling image sets `OC_PROFILE_LOG=/must-gather/oc-profile.log` by default. If your operator collects a different directory, set `OC_PROFILE_LOG` in the pod template to a path under that directory (e.g. `OC_PROFILE_LOG=/path/to/collected/root/oc-profile.log`), if the operator allows passing environment variables into the must-gather pod.

---

## Example: top 10 slowest commands

```bash
tail -n +2 oc-profile.log | sort -t'|' -k3 -rn | head -10
```

With the header preserved:

```bash
head -1 oc-profile.log
tail -n +2 oc-profile.log | sort -t'|' -k3 -rn | head -10
```
