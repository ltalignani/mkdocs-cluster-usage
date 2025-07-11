# Running R scripts on a Cluster (clean & reproducible)

<h2 class="no-toc">Table of Content</h2>

[TOC]

When launching R scripts on a computing cluster, it is highly recommended to run them in a clean environment to ensure reproducibility and avoid unexpected side effects caused by user-specific configurations. By creating a R projects, you will isolate your work from other projects. Not only can you have project specific files and libraries, but also project specific settings. To do so, you just have to:

1. Create `renv` environment, 
2. Use `--vanilla` option when running a script,

## Organize your code:
Up next: organize your code.  
Choose a structure that works well for you and stick to it as much as possible. Below is a suggested way of organising the code, with some files and directories created automatically:

```
my_project/
├── Cluster_logs/  
├── code/          		        # R script directory
│   └── myscript.R
│   └── setup_env.R             # First script to run
├── data/          		        # Raw data directory
│   └── data.csv
├── results/     		        # Analysis results directory
├── renv/          		        # renv directory
│   └── activate.R
│   └── library
│   └── local
├── renv.lock      		        # Frozen dependencies
├── tests/          		    # Temporary test code
│   └── my_temporary_script.R
├── tmp/    
├── .Renviron     		        # Local environment variables
├── .Rprofile                   # Starting session instructions
├── .gitignore 
└── README.md
```

**Mandatory**:  
- `README.md` file. This file is arguably **the most important** file you’ll create in your project. It should provide a clear overview of the code’s purpose, including:
    - The goal of the project
    - Set-up and execution instructions
    - If no separate documents exist (e.g., architecture diagram or system maintenance guide), a high-level explanation of how the code is structured and works

!!! warning "The `README.md` is not the place to describe individual functions in detail"  
     Keep it high-level and accessible.  
     The `README.md` is a living document. As your code evolves, make sure to keep the documentation up to date.  
     Don’t postpone writing it until the end of the project — by then, you’ll likely be racing against deadlines and already thinking about your next project.  
     Start documenting early. It will save you time and effort later. 

**File / directories to be created:**  
- `Cluster_logs` directory. It contains logs files redirected from standard output (`.out`) and standard error (`.err`).
- `code/` directory. It contains scripts, in the form of `.R` files.  
- `data/` directory. It contains data to be analyzed (.csv, .vcf...).  
- `results/` directory. It contains outputs. It is recommended to add subdirectories in this directory.  
- `tests/` folder. Code tests should go in here.  
- `tmp/`directory. It is used to store temporary files generated by the programs or workflows within this project. It serves as a local alternative to the system’s default $TMPDIR, which is often saturated or unreliable on shared cluster environments.  
- `.gitignore`. In this file, you add everything you do not want ending up in your remote repository. As a minimum, add `.RProj` file in there.  


**Automatically created if you followed step 1:**  
- `renv.lock` file, `renv/` folder and `.Rprofile` file. This is the virtual environment set up. They will be automatically created by renv when you activate it.  
- `.Rproj` file. This is an RStudio file containing the project specific settings. Created automatically when you create an R project with RStudio ON Demand (see further).  


## Use renv::init() to create clean environment and install necessary packages

`renv::init()` does several important things:   

1.	Creates the renv/ infrastructure in your project:  
      1. `renv/`directory with configuration files  
      2. `Rprofile` file that activates `renv` at start-up  
      3. `renv.lock` file (empty at start)  
2.	Isolates the environment: creates a private package library for this project only.  
3.	Scans the existing code to automatically detect the packages used (via library(), require() ...)  
4.	Installs the detected packages in the project's private library (NOT all, unfortunatelly...)  

## How to
1. If you haven't already done so, go into the project's directory 
```bash
cd /path/to/my_project/
```

2. Create a setup_env.R file 
```bash
vim code/setup_env.R
```
```R
#!/usr/bin/env Rscript
# setup_env.R

cat("Setting up R environment...\n")

# 1. Install renv first (required before)
if (!requireNamespace("renv", quietly = TRUE)) {
    cat("Installing renv package...\n")
    install.packages("renv", repos = "https://cran.r-project.org/")
}

# 2. Initialize renv for this projet
cat("Initializing renv environment...\n")
renv::init()

# 3. List all necessary packages
required_packages <- c(
    "data.table",
    "ggplot2",
    "dplyr",
    "parallel",
    "BiocManager"			# For Bioconductor packages if necessary
)

# 4. CRAN packages management
cat("Installing CRAN packages...\n")
install.packages(required_packages)

# 5. Bioconductor packages management
if ("BiocManager" %in% required_packages) {
    cat("Installing Bioconductor packages...\n")
    BiocManager::install(c("DESeq2", "edgeR"))		# examples
}

# 6. Final snapshot - Freeze all
cat("Creating final snapshot...\n")
renv::snapshot()

# 7. Final checkup
cat("Setup completed. Session info:\n")
sessionInfo()
```
Save the file and quit vim by **pressing escape** and entering the following command :
```bash
:wq
```
At this step, you should already have made a list of the packages needed for the analyses. If you need to add packages during the analysis, simply edit this file and run it again.  

3. Use interactive mode to setup the environment

```bash
srun -p fast -c 2 --pty bash -i
module load R/4.3.3
Rscript code/setup_env.R
```
4. The next move: analyzing data
   
When you create or open a R script in the project, either via a terminal or in RStudio ONDemand(if available), simply start it with your isolated environment:
```R
renv::activate()
```
The previously versions of installed packages will then be available.
Don't forget to add to the end of your script: 
```R
renv::deactivate()
```
In a bash script, you can also use
```R
renv::restore()
```
if you already have created the `renv`. `renv::restore()` compares packages recorded in the lockfile to the packages installed in the project library. Where there are differences it resolves them by installing the lockfile-recorded package into the project library.

## Use --vanilla to launch R scripts: avoid side-effects

The `--vanilla` option disables all user and system configuration files and avoids automatic workspace saving/loading. It is equivalent to:

```
--no-save: 		    don't save workspace 
--no-restore: 		don't restore workspace
--no-site-file: 	ignore system configuration files
--no-init-file: 	ignore .Rprofile
--no-environ: 		ignore .Renviron
```

This ensures that your script runs exactly the same way regardless of your personal `.Rprofile`, `.Renviron`, or any `.RData` file.
`--vanilla` does NOT manage versions of packages. That's why `renv` is necessary.

## Example in sbatch mode:
In your project directory, create a file in charge of cleaning the environment, restoring the created environment and running the R script: 

```bash
cd /path/to/your/directory/code/
vim my_script.R
```

```bash
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

#Setup cleanup trap
cleanup() {
    echo "Cleaning up temporary files..."
    rm -rf $TMPDIR/R_session_*
    rm -rf /tmp/Rtmp*
}

trap cleanup EXIT

# Load module
module purge
module load R/4.3.3

# Set R environment variables
export TMPDIR=$SLURM_TMPDIR

# Navigate to working directory
cd $SLURM_SUBMIT_DIR

# Restore Renv THEN run R script
Rscript -e "renv::restore()"
Rscript my_script.R

# Cleanup R temporary directories
find /tmp -name "Rtmp*" -user $USER -exec -rm -rf {} +
```

Save the file and quit the editor (esc + :wq). 

To run the script, just type the following command:

```bash
sbatch --vanilla my_script.R
```

if `--vanilla` is already in the bash script (ie: Rscript --vanilla my_script.R), just type:

```bash
sbatch my_script.R
```
This opens a clean R session in the compute node with the specified resources.



## Example in srun mode:
1. Follow the procedure described previously. 
2. Ask for resources in interactive mode
```bash   
srun --pty --time=2:00:00 --mem=8G bash
```

3. Create a complete R script
   
4. Run the analysis file

```bash
srun --time=1:00:00 --mem=4G Rscript --vanilla my_script.R 
```