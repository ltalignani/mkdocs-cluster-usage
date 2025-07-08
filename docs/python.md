# Running Python Scripts on a Cluster (Clean and Reproducible)

<h2 class="no-toc">Table of Content</h2>

[TOC]

Python is widely used for data science, bioinformatics, and HPC workflows. On a cluster, it’s essential to manage environments carefully to avoid dependency conflicts and ensure reproducibility.

## Organize your code
Read this first: [Organizing data in the project directory](Logging.md#organizing-data-in-the-project-directory)

Below is a suggested way of organizing the code, with some files and directories created automatically:
```
my_project/
├── Cluster_logs/  
├── code/          		        # Script directory
│   └── myscript.py
├── data/          		        # Raw data directory
│   └── data.csv
├── results/     		        # Analysis results directory
├── tests/          		    # Temporary test code
│   └── my_temporary_script.py
├── tmp/    
├── venv/          		        # Python virtual env.
│   └── bin/
│         └── activate
│   └── include/
│   └── lib/
│   └── pyenv.cfg
└── README.md
```


## Best Practices for Running Python Jobs

Before launching a job, make sure your script runs in a controlled Python environment.

### With venv (standard Python)

```python
python -m venv myenv
source myenv/bin/activate
pip install -r requirements.txt
```

the file requirements.txt contains, for example:
```python
numpy==1.26.4
pandas>=2.2.0
scipy<1.12
matplotlib
seaborn==0.12.2
scikit-learn>=1.3
jupyterlab
requests
biopython
```

### With conda:
```conda
conda create -n myenv python=3.11
conda activate myenv
conda install numpy pandas scipy  # etc.
```

Once set up, activate your environment inside your SLURM job script before running your script.

## SLURM Job Script for Python

```bash
#!/bin/bash
#SBATCH --job-name=python_analysis
#SBATCH --output=out_%j.log
#SBATCH --error=err_%j.log
#SBATCH --time=02:00:00
#SBATCH --partition=short
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --account=my_account_name

# Load the Python module (if available)
module load python/3.11

# Optional: Activate your virtual or conda environment
# For venv
source ~/myenv/bin/activate
# Or for conda
# source ~/.bashrc
# conda activate myenv

# Run the script
python my_script.py
```

## Run a job interactively with srun
If you want to test your Python code interactively on a compute node:

```bash
srun -c 4 -m 8G -t 01:00:00 -p fast -A my_account_name --pty bash -i
```

Then inside the shell:

```bash
module load python/3.11
source ~/myenv/bin/activate
python
```
Or
```bash
module load conda
conda activate myenv
python
```
