# CIAI-Wrangler

CIAI-Wrangler is a simple Python CLI tool to orchestrate and monitor the submission of batch job scripts to an HPC Slurm cluster. It ensures that a user-defined number of jobs run concurrently and logs the status of each job (queued, running, completed, failed).

> **Disclaimer:**
> This is an experimental tool for orchestrating SLURM jobs. Use at your own risk—no guarantees! If your job pipeline fails hours before your paper deadline, please don't @ me.

## Features
- Submits jobs from a list, limiting the maximum number of concurrent jobs (`-q`)
- Automatically starts new jobs as others complete
- Polls job status every 5 seconds
- Logs status (to stdout and optionally a file): job script, job ID, and real-time state

## Requirements
- Python 3.7+
- Access to Slurm CLI tools (`sbatch`, `squeue`, `sacct`)

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
   ciai_wrangler JOB_LIST_FILE [-q MAX_QUEUE] [-l LOG_FILE]
   ```

Or, to run locally (without moving):
```sh
./ciai_wrangler JOB_LIST_FILE [-q MAX_QUEUE] [-l LOG_FILE]
```


- `JOB_LIST_FILE`: Path to a text file containing **absolute paths** to your job scripts (one per line).
- `-q MAX_QUEUE`: Number of jobs to have running/queued at once. **Default:** 3
- `-l LOG_FILE`: (optional) Log file path; if omitted, logs print only to stdout.

**Example:**

Suppose you have a file `jobs.txt`:
```
    /absolute/path/to/mysim1.sh
    /absolute/path/to/mysim2.sh
    /absolute/path/to/mysim3.sh
    /absolute/path/to/mysim4.sh
```
And you want at most 2 jobs running at a time, while logging to both stdout and `runlog.txt`:



```sh
ciai_wrangler jobs.txt -q 2 -l runlog.txt
```

If you omit `-q`, the default is 3 concurrent jobs:

```sh
ciai_wrangler jobs.txt
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
- Requires working `sbatch`, `squeue`, and `sacct` commands in your environment.
- Only Slurm job scripts (`.sh` files, typically) should be listed in the input list.
- **Important:** The job list file should contain absolute paths to scripts. This avoids issues with changing working directories or running the tool from different locations.
- If a job fails to submit, this will also be logged.
