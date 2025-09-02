# CIAI-Wrangler

CIAI-Wrangler is a simple Python CLI tool to orchestrate and monitor the submission of batch job scripts to an HPC Slurm cluster. It ensures that a user-defined number of jobs run concurrently and logs the status of each job (queued, running, completed, failed).

> **Disclaimer:**
> This is an experimental tool for orchestrating SLURM jobs. Use at your own risk—no guarantees! If your job pipeline fails hours before your paper deadline, please don't @ me.

## Features
- Submits jobs from a YAML config list
- Enforces two limits: `max_queue` (our in‑flight limit) and `cluster_capacity` (total capacity respected by subtracting external RUNNING+PENDING jobs via `squeue --me`)
- Automatically starts new jobs as others complete and as external jobs finish to fully utilize capacity
- Polls job status every 5 seconds
- Logs status (to stdout and optionally a file): job script, job ID, and real-time state

## Requirements
- Python 3.7+
- PyYAML (`pip install pyyaml`)
- Access to Slurm CLI tools (`sbatch`, `squeue`)

## Installation
No installation needed. Copy `ciai_wrangler` to your cluster login node.

## Usage

### To install for global use:

1. Move the `ciai_wrangler` script (no `.py` extension) to a directory in your `PATH`, e.g.:
   ```sh
   mv ciai_wrangler ~/bin/
   chmod +x ~/bin/ciai_wrangler
   export PATH="$HOME/bin:$PATH"
   ```
2. Now you can invoke it from anywhere as:
   ```sh
   ciai_wrangler CONFIG_YAML
   ```

Or, to run locally (without moving):
```sh
./ciai_wrangler CONFIG_YAML
```


- `CONFIG_YAML`: Path to a YAML configuration file.

YAML schema:

```
jobs:
  - /absolute/path/to/mysim1.sh
  - /absolute/path/to/mysim2.sh
cluster_capacity: 3  # optional; default = max_queue
max_queue: 2         # optional; default 3
log_file: runlog.txt  # optional, default: none (stdout only)
```

**Example:**

Suppose you have a file `config.yaml`:
```
jobs:
  - /absolute/path/to/mysim1.sh
  - /absolute/path/to/mysim2.sh
  - /absolute/path/to/mysim3.sh
  - /absolute/path/to/mysim4.sh
cluster_capacity: 3
max_queue: 2
log_file: runlog.txt
```
And you want at most 2 jobs running at a time, while logging to both stdout and `runlog.txt`:



```sh
ciai_wrangler config.yaml
```

If you omit `max_queue` in the YAML, the default is 3 concurrent jobs:

```sh
ciai_wrangler config.yaml
```

Typical output:
```
[[36m2024-07-16 19:06:42[0m] Job /absolute/path/to/mysim1.sh started with ID 12345 (queued)
[[36m2024-07-16 19:06:42[0m] Job /absolute/path/to/mysim2.sh started with ID 12346 (queued)
[[36m2024-07-16 19:06:46[0m] Job /absolute/path/to/mysim1.sh with ID 12345 running
[[36m2024-07-16 19:10:51[0m] Job /absolute/path/to/mysim1.sh with ID 12345 completed
[[36m2024-07-16 19:10:51[0m] Job /absolute/path/to/mysim3.sh started with ID 12347 (queued)
...
```

**IMPORTANT: Due to current cluster issues, jobs that disappear from the queue are always marked as "completed" in the log, whether they succeeded or failed. You will not see a "failed" log entry. You must verify success/failure through other means if needed.**

The log shows job script, job ID, and status, each time it changes, with a timestamp.

## Job Statuses
- **queued:** The job is in the Slurm queue (waiting to run)
- **running:** The job is running on compute resources
- **completed:** The job left the queue; due to current cluster conditions, this status is used for any job removed from the queue, regardless of whether it succeeded or failed.

## Notes
- Requires working `sbatch` and `squeue` commands in your environment.
- Uses `squeue --me` to identify your currently RUNNING and PENDING jobs and adjusts submissions accordingly.
- If `cluster_capacity` is omitted, it defaults to `max_queue` (backwards-compatible behavior).
- Only Slurm job scripts (`.sh` files, typically) should be listed in `jobs`.
- **Important:** Prefer absolute paths to scripts. This avoids issues with changing working directories or running the tool from different locations.
- If a job fails to submit, this will also be logged.
