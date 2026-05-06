# Ansys OnDemand: The Permissions Trap (and a Lookalike Session-Teardown Bug)

## What this file is and why it matters

This note documents the **first** class of Ansys OnDemand failures we hit on May 6 — failures that *looked* like noVNC / Slurm / app-config bugs but were actually caused by **filesystem permissions on the Ansys install tree**. It also documents a second behavior that *looked* like a reconnect bug but was actually expected session teardown.

Read this if you see:
- `runwb2: command not found` in `output.log` even after `module load ansys/<version>` succeeds
- `Failed to connect to server` from noVNC right after the job starts
- A session that worked, then refused to reconnect after you closed Workbench

The common trap in all three cases is the same: the visible symptom is downstream of the real cause. `PATH` can be correct and still point at directories your account cannot traverse. noVNC can fail because the backend session has already cleaned itself up by design.

For the *separate* problem of Workbench failing inside a CPU-only Slurm allocation (the SceneGraphChart addin error), see `MAY6_ANSYS_DEBUG_SUMMARY.md` — that one led to the GPU-only refactor and is unrelated to permissions.

## Summary
This debugging session uncovered two separate Ansys Open OnDemand behaviors:

1. An initial hard failure caused by permissions on the Ansys installation tree.
2. A later session that appeared to "fail to reconnect" but had actually ended cleanly because closing Ansys ended the entire VNC-backed session.

The main lesson is to separate:
- app startup failures
- software access and module issues
- expected session teardown behavior after the main GUI process exits

## Problem 1: Job failed immediately after launch

### Symptoms
- Open OnDemand showed the session as created.
- noVNC opened but often showed `Failed to connect to server`.
- Slurm email reported the job as `FAILED` after about 12 seconds.
- The session looked contradictory because OOD briefly showed it as running while Slurm had already marked the backend job failed.

### What I inspected
I inspected the app source in the original app repo:
- `template/script.sh.erb`
- `template/before.sh.erb`
- `submit.yml.erb`
- `form.yml.erb`
- generated session files under:
  - `~/ondemand/data/sys/dashboard/batch_connect/dev/ansys/output/<session-id>/`

I also inspected:
- `output.log`
- `vnc.log`
- `websockify.log`
- generated `job_script_content.sh`
- Slurm accounting via `sacct`
- module behavior via `module show ansys/2025R2`

### Root cause
The failure was not caused by noVNC first.

The actual failure was:
- `module load ansys/2025R2` succeeded
- the launch script then ran `runwb2 -oglmesa`
- but `runwb2` was not reachable for my account

The session log showed:
- `runwb2: command not found`

The key discovery was that the module added Ansys directories to `PATH`, but my user could not traverse the underlying installation tree:

- `/cluster/tufts/apps/manual/9/x86_64/ansys/2025R2`

and `namei -l` showed permission denial at:
- `ansys_inc - Permission denied`

So `PATH` contained the right Ansys directories, but they were not actually accessible to my user.

### Proof commands
These commands confirmed the permissions issue:

```bash
source /etc/profile.d/modules.sh
module purge
module load ansys/2025R2
echo "$PATH" | tr ':' '\n' | grep ansys
command -v runwb2
namei -l /cluster/tufts/apps/manual/9/x86_64/ansys/2025R2/ansys_inc/v252/Framework/bin/Linux64/runwb2
```

What they proved:
- Ansys paths were added to `PATH`
- `runwb2` was still not found
- traversal into the Ansys install tree failed for my user

### Fix / resolution
This was not fixed in the app code.
The operational fix was to get the account/access situation corrected so the Ansys install became reachable to my user.

Important conclusion:
- this was a permissions / group / ACL access issue on the Ansys installation
- not primarily a VNC problem
- not primarily an `LD_LIBRARY_PATH` issue

## Problem 2: Session connected, but reconnect failed after closing something

### Symptoms
A later Ansys session connected successfully. After closing the app/window, reconnecting from noVNC failed with:
- `Failed to connect to server`

At first this looked like a reconnect bug.

### What I inspected
For session:
- Job ID: `730580`
- Session ID: `5893555f-910e-4e88-a1b1-3d4246814559`

I inspected:
- `output.log`
- `vnc.log`
- `websockify.log`
- `connection.yml`
- the generated `job_script_content.sh`
- Slurm state with `sacct`
- OOD batch_connect DB record for the session

### Root cause
This session did not fail.
It completed successfully.

Evidence:
- Slurm reported job `730580` as `COMPLETED`
- OOD session DB had `cache_completed: true`
- `vnc.log` and `websockify.log` showed successful startup and client connection
- the wrapper script later ran `clean_up`, killed VNC, and exited `0`

The reason reconnect failed is that the app is designed so that:
- `runwb2 -oglmesa` is the main foreground application process
- the wrapper waits for that process to end
- when it ends, the wrapper shuts down VNC and websockify

So closing Ansys/Workbench effectively ends the whole interactive session.

This means:
- reconnect failure after closing the main GUI is expected behavior for this app as currently written
- it is not a permissions issue
- it is not a noVNC bug
- it is not a Slurm failure

## Important implementation detail
The generated job wrapper does roughly this:
- start TurboVNC
- start websockify
- launch the app script
- `wait` on the app process
- when that process exits, clean up VNC and end the job

That means session lifetime is tied directly to the Ansys Workbench process.

## Relevant logs and what they mean

### `output.log`
Best first log for overall session behavior.
Useful for seeing:
- whether VNC started
- whether websockify started
- whether `runwb2` launched
- whether cleanup ran
- whether the session ended with failure or clean exit

### `vnc.log`
Useful for confirming:
- TurboVNC started successfully
- the VNC display and port
- whether a client actually connected
- when authentication occurred

### `websockify.log`
Useful for confirming:
- noVNC websocket proxy started
- browser connected to websocket
- backend proxy target was correct

### `job_script_content.sh`
Useful for understanding the actual Open OnDemand wrapper logic, especially:
- when the main script is launched
- what process is being waited on
- what cleanup does

## Commands that were useful for debugging

### Check job outcome
```bash
sacct -j <jobid> --format=JobID,JobName,Partition,State,ExitCode,Elapsed,AllocTRES%40,NodeList%30 -P
```

### Inspect session logs
```bash
SESSION="$HOME/ondemand/data/sys/dashboard/batch_connect/dev/ansys/output/<session-id>"
ls "$SESSION"
sed -n '1,220p' "$SESSION/output.log"
sed -n '1,220p' "$SESSION/vnc.log"
sed -n '1,220p' "$SESSION/websockify.log"
```

### Check generated wrapper behavior
```bash
sed -n '1,260p' "$SESSION/job_script_content.sh"
```

### Check module and path behavior
```bash
source /etc/profile.d/modules.sh
module purge
module load ansys/2025R2
module show ansys/2025R2
command -v runwb2
echo "$PATH" | tr ':' '\n' | grep ansys
```

### Prove directory traversal/permission problem
```bash
namei -l /cluster/tufts/apps/manual/9/x86_64/ansys/2025R2/ansys_inc/v252/Framework/bin/Linux64/runwb2
```

## App-specific observations
There were also some copy/paste leftovers in the app source that were not the root cause, but are worth cleaning up later:
- `submit.yml.erb` still referenced `OnDemand/Paraview`
- a comment in `script.sh.erb` still said `Launch fastqc`
- partition handling looked partially hardcoded in one version of the app

These were not the main cause of the two behaviors above, but they are good cleanup targets.

## Future troubleshooting checklist
If Ansys OnDemand breaks again, check in this order:

1. Did Slurm mark the job `FAILED` or `COMPLETED`?
2. Does `output.log` show `runwb2: command not found` or another direct launch error?
3. Did VNC and websockify start successfully?
4. Did the user close the main Ansys window, causing expected session cleanup?
5. Does `command -v runwb2` work after `module load ansys/2025R2`?
6. If not, is the Ansys install tree traversable with `namei -l`?

## Key takeaways
- A correct-looking `PATH` is not enough if the underlying directories are permission-blocked.
- `runwb2: command not found` can be caused by inaccessible directories, not just a missing module.
- `Failed to connect to server` in noVNC can be a downstream symptom after the backend VNC session is already gone.
- If the app is written to tie the whole session to `runwb2`, closing Ansys ends the session by design.
- `output.log` is usually the best first place to look.
