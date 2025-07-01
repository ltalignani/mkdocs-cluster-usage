# Running Interactive Jobs with srun

srun can be used to run jobs interactively or within a job script. This is useful for quick tests or debugging.


## Example: Interactive shell with 4 CPUs and 4 GB RAM for 30 minutes
```bash
srun --partition=fast --cpus-per-task=4 --mem=4G --time=00:30:00 --pty bash -i
```
Or
```bash
srun -p fast -c 4 -m 4G -t 00:30:00 --pty bash -i
```

This will open a shell on the compute node `fast` with the requested resources.


Most clusters have different partitions (also called “queues”) that are optimized for different job durations or resource types. For example:  
    - fast/short → for jobs < 24 hours  
    - long → for jobs between 24 hours and 30 days  
    - Other examples might include gpu, highmem, or interactive  
