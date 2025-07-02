# Submitting a Job with sbatch

The `sbatch` command is used to submit batch jobs to the SLURM scheduler. SLURM (*Simple Linux Utility for Resource Management*) is a job scheduler widely used in high-performance computing (HPC) clusters. It manages and allocates computing resources, queues user-submitted jobs, and efficiently schedules them across cluster nodes to optimize workload distribution and resource utilization.

Jobs submitted via sbatch are defined in a batch script (usually with a .sh or .slurm extension). This script contains a header section specifying the requested resources (such as number of CPUs, memory, runtime), followed by the commands to execute. The scheduler reads these directives and runs the job accordingly.

The most common way to launch a job on a SLURM cluster is with a script written in Bash scripting language. 

In addition to the bash commands it must execute, Bash script must include a **header** containing the **SLURM directives**, specifying the resources requested (number of CPUs, memory, maximum time, partition, etc.). 

## SLURM CONFIGURATION HEADER
The SLURM scheduler will read this header to determine the priority of the script in the queue. 

SLURM does not use a priority based on the order of directives, but calculates a numerical priority for each job, according to several factors (partition requested, time in the queue, size of the job (=number of nodes, memory and cpus requested)). The number of directives entered in the header also has a positive influence on the priority.

These directives begin with `#SBATCH`:


```bash linenums="1"
#!/bin/bash
###################configuration slurm##############################
#SBATCH -A invalbo
#SBATCH --time=2-23:00:00
#SBATCH --job-name=fastqc
#SBATCH -p long
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
#SBATCH --array 1-22
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
####################################################################
```

### SLURM Account (-A / --account=)
```bash
#SBATCH -A invalbo
or 
#SBATCH --account=invalbo
```
An account name must be entered. In most cases, this corresponds to the name of the project submitted and therefore the name of the directory in which the scripts and data to be processed are stored on the cluster.

### Time (-t / --time=)
```bash
#SBATCH --time=2-23:00:00
```
This directive corresponds to the maximum time that must be allocated to the job. The number of days appears first (max. 30), followed by a dash, then the hours:minutes:seconds. If the job exceeds this time, it is automatically cancelled by the scheduler. **Note that some partitions, such as fast, do not accept jobs lasting more than 24hrs.**

### Job name (-J / --job-name=)
```bash
#SBATCH --job-name=fastqc
```
Name of the job that appears in SLURM information, available with `sinfo` or `squeue`

### Partition (-p / --partition=)
```bash
#SBATCH -p long
```
Name of the partition to be used (fastc, long, gpu...).


### Nodes (-N / --nodes)
```bash
#SBATCH -N 1
```
Number of nodes required for the job. A node is a physical machine in the cluster. Using more than one node (-N > 1) in a SLURM job is rare in standard workflows, but essential in certain large-scale distributed computing contexts. 

You use several nodes when...

1. You need more cores or memory than a single node can provide
2. You run distributed computing (MPI)

This is the classic case for using several nodes. For example, in numerical simulation, molecular modelling, deep learning on supercomputers, etc.

`-N > 1` is rarely used.

### Tasks (-n / --ntasks)
```bash
#SBATCH -n 1
```
Total number of tasks (parallel processes) to run. Each task can run on a separate thread or core, or even a separate node.
If -n > 1, tells SLURM to run several parallel tasks (often for MPI, or sample processing). 

`-n > 1` is rarely used.

### CPUs & Memory (--cpus-per-task | -m / --mem)
```bash
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
```
`--cpus-per-task` is a long option only. It is used to specify the number of cores (CPU threads) to allocate per task.  
`--mem=` is used to specify the **maximum memory** allocated to the task.


### SLURM Job Array (--array)
```bash
#SBATCH --array=1-22
```
means that SLURM will **submit 22 identical jobs**, each with a unique index accessible via the environment variable `SLURM_ARRAY_TASK_ID`. This is commonly used to iterate over samples or files and apply the same script automatically. (see `SLURM job on a simple bash script` for more info). The slurm array range used must correspond to the number of samples to be processed.

### Output & Error management (-o / --output= | -e / --error)
```bash
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
```
are used to redirect standard and error output to files. This is very useful for debugging and tracing jobs.  
`-o` or `--output=` defines the file in which the job's standard output (stdout) will be saved, i.e. what would normally be printed on the screen.

`-e` or `--error=` defines the file in which the error output (stderr) will be saved, i.e. all error messages (system errors, execution errors, etc.).

#### About SLURM variables:
In an SBATCH directive, certain SLURM variables correspond to :  
- %x: job name (defined by --job-name or -J)  
- %j: job ID (unique number assigned by SLURM)  
- %N: name of the compute node on which the job is running  


#### Other common SBATCH variables :
* Job information :  
- `%A`: main job ID (for job arrays)  
- `%a`: job array index  
- `%u`: user name  
- `%x`: job name  
  
* System information :  
- `%N`: node name (short)  
- `%n`: relative node name in allocation  
- `%t`: job ID  


### Mail (--mail)
```bash
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
```
You can use this directive to enter an email address. If this is the case, SLURM will send an email to warn you if any problems arise during the job.

`--mail-type` Main options:

- NONE: no mail (default)
- BEGIN: at the start of the job
- END: at the end of the job (success or failure)
- FAIL: only in the event of failure
- REQUEUE: when the job is queued again
- ALL: equivalent to BEGIN,END,FAIL,REQUEUE


## Summary of the main SLURM directives
Short option (single hyphen -):  
  - No equal sign (=)  
  - Arguments are separated by a space  
  


| Long Option        | Short Option | Description               |
| :----------------- | :----------: | :------------------------ |
| `--mem=`           |     `-m`     | Total memory per node     |
| `--time=`          |     `-t`     | Job time limit            |
| `--partition=`     |     `-p`     | Partition (queue) name    |
| `--cpus-per-task=` |      —       | Number of CPUs per task   |
| `--account=`       |     `-A`     | Computing project/account |
| `--job-name=`      |     `-J`     | Job name                  |
| `--output=`        |     `-o`     | Standard output file      |
| `--error=`         |     `-e`     | Error log file            |




## Submitting a SLURM job on a simple bash script

```bash linenums="1"
#!/bin/bash
###################configuration slurm##############################
#SBATCH -A invalbo
#SBATCH --time=2-23:00:00
#SBATCH --job-name=fastqc
#SBATCH -p long
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
#SBATCH --array 1-22
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
####################################################################

module load fastqc/0.12.1
module load bcftools/1.14

# Recover of fastq file name
SAMPLELANE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" info_files/fastq_files)

# Locate INPUT directory 
INPUT_FOLDER="raw"


fastqc --threads 8 -o qc/fastqc/ "$INPUT_FOLDER/${SAMPLELANE}_R1.fastq.gz"
fastqc --threads 8 -o qc/fastqc/ "$INPUT_FOLDER/${SAMPLELANE}_R2.fastq.gz"

bcftools merge --threads 12 -r 3L $(cat vcf_files.txt) -Oz -o UG.ag3.3L.vcf.gz
```

After the SLURM directives, the rest of the script contains regular Bash commands. These commands can use SLURM environment variables or call external programs.

### Loading Environment Modules

Most HPC systems use environment modules to manage software. To ensure the required tools are available (and versions are controlled), modules must be loaded explicitly:

```bash linenums="18-19"
module load fastqc/0.12.1
module load bcftools/1.14
```

These commands load specific versions of fastqc and bcftools that are pre-installed on the cluster. To find out which programs and program versions are available on the cluster, use the following command:

```bash
module avail
```
You can scroll through the long list of available modules using the space bar or the vertical scroll arrows.


### Using SLURM Variables and Automating Input

```bash linenums="21-22"
# Recover fastq filename from a list based on the SLURM_ARRAY_TASK_ID
SAMPLELANE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" info_files/fastq_files)
```
This command retrieves the appropriate sample name from a text file (info_files/fastq_files), where each line corresponds to a sample:

```bash title="Content of the info_files/fastq_files file"
Sample_10_EKDN230051732-1A_H2F5FDSXC_L4
Sample_11_EKDN230051733-1A_H2WC5DSXC_L1
Sample_13_EKDN230051735-1A_HFTFWDSX7_L4
Sample_14_EKDN230051736-1A_HFTFWDSX7_L4
Sample_15_EKDN230051737-1A_H2F5FDSXC_L4
```
The script dynamically selects the correct line based on the job array index. `Sample_10_EKDN230051732-1A_H2F5FDSXC_L4` will have the index `${SLURM_ARRAY_TASK_ID}` number 1 and will be ran in the first array, `Sample_11_EKDN230051733-1A_H2WC5DSXC_L1` will take the index `${SLURM_ARRAY_TASK_ID}` number 2 and will be ran in the second array... 


### Defining Input Directory and Running the Commands
```bash linenums="24-25"
# Locate INPUT directory 
INPUT_FOLDER="raw"
```

In this case, the program FastQC is then run on the paired-end reads:
```bash linenums="28-29"
fastqc --threads 8 -o qc/fastqc/ "$INPUT_FOLDER/${SAMPLELANE}_R1.fastq.gz"
fastqc --threads 8 -o qc/fastqc/ "$INPUT_FOLDER/${SAMPLELANE}_R2.fastq.gz"
```
These commands perform quality control on the forward and reverse FASTQ files of the current sample.


## Submitting a SLURM job on a python script

You can submit a python script directly to SLURM:
```bash
sbatch --wrap="python my_script.py" \
       -A invalbo \
       --time=2-23:00:00 \
       --job-name=python_wrap \
       -p long \
       --cpus-per-task=8 \
       --mem=8G
```
!!! warning "However, this will have major limitations"
    - No module loading: impossible to make module load python/3.11 in the Python script  
    - Environment variables: more difficult to configure  
    - Error handling: less control over execution  
    - Debugging: more complicated to debug  



### Running a simple python script
In addition to your python script file, you need to prepare in the same directory a Bash script containing:

```bash title="A simple job" linenums="1" hl_lines="21-22"
#!/bin/bash
###################configuration slurm##############################
#SBATCH -A invalbo
#SBATCH --time=23:00:00
#SBATCH --job-name=python_script
#SBATCH -p fast
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
#SBATCH --array 1-22
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
####################################################################

# Load python 3.11 module
module load python/3.11

# Run python script
python my_script.py
```


The last command will run the python script. All log messages from this script will be saved in the files defined by `-o` and `-e` options.



### Running a complex python script

It is possible to retrieve the values of SLURM environment variables and pass them to the script: 

```bash title="A complex job script for a complex python script" linenums="1" hl_lines="21-22"
#!/bin/bash
###################configuration slurm##############################
#SBATCH -A invalbo
#SBATCH --time=2-23:00:00
#SBATCH --job-name=data_processing
#SBATCH -p long
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
#SBATCH --array 1-22
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
####################################################################

module load python/3.11

# environment variables
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export SAMPLE_ID=$SLURM_ARRAY_TASK_ID

# Define pathways
INPUT_DIR="/data/raw/samples"
OUTPUT_DIR="/data/processed/results"
CONFIG_FILE="/configs/analysis_config.yaml"

# Create output directory
mkdir -p ${OUTPUT_DIR}

# Runnng script with multiple arguments
python analysis_pipeline.py \
    --input ${INPUT_DIR}/sample_${SAMPLE_ID}.fastq \
    --output ${OUTPUT_DIR}/sample_${SAMPLE_ID}_results \
    --config ${CONFIG_FILE} \
    --threads ${SLURM_CPUS_PER_TASK} \
    --memory ${SLURM_MEM_PER_NODE} \
    --job-id ${SLURM_JOB_ID} \
    --array-id ${SLURM_ARRAY_TASK_ID} \
    --verbose \
    --force-overwrite

# Check output code
if [ $? -eq 0 ]; then
    echo "Job ${SLURM_ARRAY_TASK_ID} completed successfully"
else
    echo "Job ${SLURM_ARRAY_TASK_ID} failed with exit code $?"
    exit 1
fi
```


### Running a script requiring a Python environment

```bash title="A complex job script with Python environment" linenums="1" hl_lines="20-21"
#!/bin/bash
###################configuration slurm##############################
#SBATCH -A invalbo
#SBATCH --time=2-23:00:00
#SBATCH --job-name=ml_training
#SBATCH -p long
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
#SBATCH --array 1-22
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
####################################################################

module load python/3.11

# Activating virtual environment
source /path/to/venv/bin/activate

# Install dependancies if necessary
# pip install -r requirements.txt

# Job array variable
PARAM_FILE="parameters_${SLURM_ARRAY_TASK_ID}.txt"

python train_model.py \
    --param-file ${PARAM_FILE} \
    --output-dir results/run_${SLURM_ARRAY_TASK_ID} \
    --n-jobs ${SLURM_CPUS_PER_TASK}
```

## Submitting a SLURM job on a R script: 

### Running a simple Rscript:
```bash title="A simple Rscript" linenums="1" hl_lines="20-21"
#!/bin/bash
###################configuration slurm##############################
#SBATCH -A invalbo
#SBATCH --time=2-23:00:00
#SBATCH --job-name=r_analysis
#SBATCH -p long
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
#SBATCH --array 1-22
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
####################################################################

module load R/4.3.0

# Lancement simple du script R
Rscript my_script.R
```

### Running a complex Rscript:
```bash title="A complex Bash script for a complex Rscript" linenums="1" 
#!/bin/bash
###################configuration slurm##############################
#SBATCH -A invalbo
#SBATCH --time=2-23:00:00
#SBATCH --job-name=phylogenetic_tree
#SBATCH -p long
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task 8
#SBATCH --mem=8G
#SBATCH --array 1-22
#SBATCH -o Cluster_logs/%x-%j-%N.out
#SBATCH -e Cluster_logs/%x-%j-%N.err
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
####################################################################

module load R/4.3.0

# Configuration de l'environnement R
export R_LIBS_USER="/home/user/R/library"
export TMPDIR="/tmp/${SLURM_JOB_ID}"
mkdir -p $TMPDIR

# Installation conditionnelle de packages (si nécessaire)
Rscript -e "
if (!require('BiocManager', quietly = TRUE)) {
    install.packages('BiocManager', lib = Sys.getenv('R_LIBS_USER'))
}
if (!require('phyloseq', quietly = TRUE)) {
    BiocManager::install('phyloseq', lib = Sys.getenv('R_LIBS_USER'))
}
"

# Variables pour le script
DATASET="dataset_${SLURM_ARRAY_TASK_ID}"
CONFIG_FILE="/configs/phylo_config.yaml"

# Lancement avec lecture depuis stdin et redirection
Rscript phylogenetic_analysis.R \
    --dataset "${DATASET}" \
    --config "${CONFIG_FILE}" \
    --output-prefix "results/phylo_${SLURM_ARRAY_TASK_ID}" \
    --bootstrap 1000 \
    --method "neighbor-joining" \
    --distance "bray-curtis" \
    --cores ${SLURM_CPUS_PER_TASK} \
    --seed 12345 \
    2>&1 | tee "logs/phylo_${SLURM_ARRAY_TASK_ID}.log"

# Nettoyage
rm -rf $TMPDIR
```


## Summary
	•	The SLURM header sets up the job environment and resource requirements.  
	•	Modules must be explicitly loaded to ensure required tools are available.  
	•	SLURM_ARRAY_TASK_ID allows you to automate jobs over multiple input samples.  
	•	Output and error logs are stored in the specified Cluster_logs/ directory, with informative file names based on job name and ID.  
    •	To run a python or a R script, use a bash script starting with a SLURM header.