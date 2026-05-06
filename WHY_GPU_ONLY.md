# Why This Is a GPU-Only App: The CPU/GPU Workbench Problem

## What this file is and why it matters

This note explains **why the Ansys Open OnDemand app is restricted to GPU partitions only**, and the chain of debugging that led to that decision. Read this if you are wondering:

- Why does `form.yml.erb` set `@partition_filter = :gpu`?
- Why can't users pick `batch` (or any other CPU-only partition) from this app?
- What happens if you try to launch Workbench on a CPU-only allocation?

Short version: the original app was *accidentally* GPU-only because it hardcoded `--partition=gpu` and `--gres=gpu:1`. When we refactored it to be Slurm-driven and dynamic, users could suddenly select CPU-only partitions like `batch` — and Ansys Workbench failed at startup with an internal addin error (`Ans.SceneGraphChart.SceneGraphAddin`). Workbench needs an actual GPU at runtime, not just the `-oglmesa` flag. The fix is to constrain the dynamic form to GPU-capable partitions only.

---

## The conclusion (what was actually changed)

A single line in `form.yml.erb`:

```erb
<%
  @partition_filter = :gpu       # <-- added
  _partial_path = File.expand_path('partials/gpu_discovery.erb', __dir__)
  ERB.new(File.read(_partial_path)).result(binding)
  ...
%>
```

That filter is read by every form generator in `partials/gpu_discovery.erb` (`generate_partition_field`, `generate_num_cores_field`, `generate_num_memory_field`, `generate_num_hours_field`). With it set to `:gpu`:

- The **Partition** dropdown only lists user-accessible partitions that have GPU nodes. `batch`, `largemem`, `rc2`, and the lab CPU-only partitions disappear.
- Cores / memory / hours caps are scoped to GPU partitions only.
- The GPU type select (already partition-aware) can never collapse to its `'none'` fallback.
- Default `gpu_type=any` continues to work — `submit.yml.erb` emits `--gres=gpu:1`.

**No other files were changed.** The shared `partials/gpu_discovery.erb` is left intact because other apps (`javi_jupyter`, `igv`) depend on its `:cpu` and `:all` filter modes. The dead `skip_gres` branch in `submit.yml.erb` was deliberately left in place as a defensive guard — if anything ever bypasses the form filter, it still suppresses a bad `--gres`.

This was a one-line, easily revertable fix. To restore CPU-or-GPU selection, just remove the `@partition_filter = :gpu` line.

---

## How we got here (the debugging chain)

### 1. Early failures were caused by Ansys install permissions for my user

For the earliest failing sessions, the app launched VNC correctly, but the batch job died when it tried to run:

```bash
runwb2 -oglmesa
```

The session log showed:

```text
runwb2: command not found
```

This was traced to a permissions problem on the Ansys install tree for my user at:

`/cluster/tufts/apps/manual/9/x86_64/ansys/2025R2`

The module added Ansys directories to `PATH`, but traversal into the tree failed. This was proven with:

```bash
source /etc/profile.d/modules.sh
module purge
module load ansys/2025R2
echo "$PATH" | tr ':' '\n' | grep ansys
command -v runwb2
namei -l /cluster/tufts/apps/manual/9/x86_64/ansys/2025R2/ansys_inc/v252/Framework/bin/Linux64/runwb2
```

At that time:
- Ansys paths appeared in `PATH`
- `command -v runwb2` returned nothing
- `namei` showed `ansys_inc - Permission denied`

So the initial issue was not noVNC. It was user access to the Ansys install. (See `May6AnsysLearning.md` for the full permissions writeup.)

### 2. Reconnect failures after closing Ansys were expected behavior

For session:

- Job ID: `730580`
- Session ID: `5893555f-910e-4e88-a1b1-3d4246814559`

The session connected successfully. When Ansys/Workbench was closed, reconnect later failed with noVNC saying:

```text
Failed to connect to server
```

This was expected for the current app design. The wrapper launches `runwb2 -oglmesa` as the main process, waits on it, and tears down VNC/websockify as soon as it exits.

So:
- VNC was fine
- websockify was fine
- Slurm was fine
- reconnect failed only because the backend session had already been intentionally cleaned up

### 3. Dynamic app exposed an internal Workbench addin failure on CPU-only partitions

For the dynamic app the relevant failing session was:

- Job ID: `730603`
- Session ID: `44bcd80d-7ca8-4d1c-94df-2f27ec4804b5`
- Partition: `batch` (CPU-only — this is the key fact)

The popup shown inside Workbench was:

```text
Unexpected error: The following required addins could not be loaded:
Ans.SceneGraphChart.SceneGraphAddin. The software will exit.
(SceneGraphChart.Components.dll assembly: <unknown assembly> type:<unknown type> member:(null))
```

After clicking `OK`, Workbench exited, the wrapper ran cleanup, killed VNC, and Slurm marked the job as `FAILED` with exit code `1`.

**This was not an OOD/VNC startup failure.** For session `730603`:
- VNC started successfully
- websockify started successfully
- browser connected successfully
- Slurm job was running normally until Workbench itself exited

Evidence came from `sacct -j 730603`, `output.log`, `vnc.log`, `websockify.log`, and the OOD batch_connect DB record.

### 4. The launcher script was not the cause — runtime allocation was

I checked the live app. The dynamic changes are mainly in:
- `form.yml.erb`
- `partials/gpu_discovery.erb`
- `submit.yml.erb`

These affect:
- version discovery
- partition / GPU / CPU / memory choices
- Slurm submit arguments

They do **not** modify:
- Workbench addins
- DLL search configuration
- `LD_LIBRARY_PATH` for Workbench-specific component trees
- .NET / Mono addin loading logic

The launch path is still fundamentally:

```bash
module load openjdk
module load ansys/<version>
runwb2 -oglmesa
```

The original (working) app and the dynamic (sometimes-failing) app launch Workbench identically. What differs is **what kind of Slurm allocation Workbench finds itself running inside**. The original app forced `--partition=gpu --gres=gpu:1`; the dynamic app let the user pick `batch` with no GPU. Same launcher, different runtime environment — and Workbench's SceneGraphChart addin needs a GPU device to initialize.

### 5. Confirming the pattern

Subsequent testing showed:

- Dynamic app on `batch` with no GPU: SceneGraphChart addin error, Workbench exits.
- Dynamic app on `gpu` with `gpu_type=any`: works.
- Original app (always `gpu` + `--gres=gpu:1`): works.

That's what made it a partition/allocation problem rather than an install problem. The conclusion: keep the dynamic discovery (it's useful), but only let users pick GPU-capable choices.

---

## App source observations (carried forward, unchanged)

### Original app
`/cluster/home/jlavea01/ondemand/prod/ansys_initial_commit`

Noted copy/paste leftovers:
- `submit.yml.erb` used to say `OnDemand/Paraview`
- `template/script.sh.erb` has comment `Launch fastqc`

### Dynamic app
`/cluster/home/jlavea01/ondemand/prod/ansys`

Currently:
- dynamic version discovery is enabled
- dynamic GPU/partition form behavior is enabled (now constrained to `:gpu`)
- submit options are cleaner than the original

---

## Useful logs and locations

Generated session output lives under:

`~/ondemand/data/sys/dashboard/batch_connect/dev/ansys/output/<session-id>/`

Most useful files:
- `output.log`
- `vnc.log`
- `websockify.log`
- `job_script_content.sh`
- `connection.yml`

OOD batch_connect DB records:

`~/ondemand/data/sys/dashboard/batch_connect/db/<session-id>`

---

## Useful commands

Check Slurm state:

```bash
sacct -j <jobid> --format=JobID,JobName,Partition,State,ExitCode,Elapsed,AllocTRES%40,NodeList%30 -P
```

Inspect session logs:

```bash
SESSION="$HOME/ondemand/data/sys/dashboard/batch_connect/dev/ansys/output/<session-id>"
sed -n '1,260p' "$SESSION/output.log"
sed -n '1,220p' "$SESSION/vnc.log"
sed -n '1,220p' "$SESSION/websockify.log"
sed -n '1,260p' "$SESSION/job_script_content.sh"
```

Check module / command availability:

```bash
source /etc/profile.d/modules.sh
module purge
module load ansys/2025R2
module show ansys/2025R2
command -v runwb2
env | sort | rg '^(PATH|LD_LIBRARY_PATH|LIBGL|DISPLAY|ANSYS|AWP|WB|MONO|DOTNET|JAVA|JRE)='
```

Prove Ansys install traversal issues:

```bash
namei -l /cluster/tufts/apps/manual/9/x86_64/ansys/2025R2/ansys_inc/v252/Framework/bin/Linux64/runwb2
```

---

## If you ever want to re-allow CPU partitions

Remove the `@partition_filter = :gpu` line from `form.yml.erb`. The `:cpu` and `:all` filter modes still exist in the shared `partials/gpu_discovery.erb`, and `submit.yml.erb` still has the `skip_gres` branch that handles CPU-only partitions. Reverting is one line.

But before doing that: confirm whether the SceneGraphChart / GPU-required-at-runtime constraint has changed in the Ansys version you're running. If it hasn't, CPU-only sessions will still fail the same way.
