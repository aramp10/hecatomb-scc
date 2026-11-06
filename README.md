# Hecatomb on the BU Shared Computing Cluster (SCC)

Steps used to install [Hecatomb](https://github.com/shandley/hecatomb) (viral metagenomics pipeline) on BU's Shared Computing Cluster (SCC) and run its included test dataset. Tested November 2026 with Hecatomb 1.3.4.

## Summary

1. Install Hecatomb in a conda environment: a few minutes
2. Download the databases (~52 GB): about 15 minutes
3. Prepare the software Hecatomb uses during a run: about 15 minutes
4. Run the test dataset as a batch job: about 1.5–2 hours

Steps 2 and 3 download files from the internet, so run them in an SCC OnDemand Desktop session, not in a batch job.

## 1. Install Hecatomb

```bash
module load miniconda
conda create -n hecatomb python=3.11
conda activate hecatomb
pip install hecatomb "snakemake==9.16.2"
hecatomb --version
# hecatomb, version v1.3.4
```

Two differences from the Hecatomb README:

- **Python 3.11**, not 3.10. With Python 3.10, `pip` installs the older Hecatomb 1.3.3 without a warning.
- **`snakemake==9.16.2`**. Newer Snakemake versions stop Hecatomb with: `The following resources are equivalent and cannot be simultaneously provided to a rule: ['mem', 'mem_mb']`.

## 2. Download the databases (once per project)

Save the databases in one shared folder so all project members use the same copy. Hecatomb finds them through the `HECATOMB_DB` environment variable (Hecatomb 1.3.4 has no `--databases` option).

```bash
module load miniconda
conda activate hecatomb
export HECATOMB_DB=/projectnb/vervmb/hecatomb_databases
hecatomb install --threads 4
```

Took about 14 minutes; 52 GB.

**Set `HECATOMB_DB` in every session and job script.** Without it, Hecatomb looks for the databases inside your conda environment and stops with `ERROR: missing database file`.

## 3. Prepare the software Hecatomb uses (once per working folder)

Hecatomb installs its tools (MEGAHIT, MMseqs2, etc.) into a `.snakemake/conda` folder inside the folder you run it from. Do this once from your working folder:

```bash
module load miniconda
conda activate hecatomb
export HECATOMB_DB=/projectnb/vervmb/hecatomb_databases
cd /projectnb/vervmb/your-folder
hecatomb test build_envs
```

Took about 15 minutes; 6.6 GB. Always run Hecatomb from this same folder afterwards.

## 4. Run the test dataset (batch job)

Edit the `cd` line in [`examples/hecatomb_test.qsub`](examples/hecatomb_test.qsub) to your working folder, then:

```bash
cd /projectnb/vervmb/your-folder
qsub hecatomb_test.qsub
```

The script requests:

- `-pe omp 28`: 28 cores. `hecatomb test --threads $NSLOTS` uses the number of cores the job was given.
- `-l avx2`: a node with AVX2 CPU instructions. Hecatomb's MEGAHIT stops with an "illegal instruction" error (exit code -4) on older nodes without them.

Test results on the SCC (28 cores):

| | |
|---|---|
| Run time | 1h50m |
| Peak memory | 39 GB |
| Output | `hecatomb.out/`, 8.9 GB |

The test ends with `Hecatomb finished successfully!` in the job log (`hecatomb_test.<jobid>.log`).

## Disk space

| What | Size | Where |
|---|---|---|
| Databases (shared) | 52 GB | `/projectnb/vervmb/hecatomb_databases` |
| Conda environment | < 1 GB | each user |
| Hecatomb's tools | 6.6 GB | `.snakemake/conda` in the working folder |
| Test output | 8.9 GB | `hecatomb.out/` in the working folder |
| **Total** | **about 68 GB** | |

Before starting, make sure your SCC project has enough free space (quota minus usage):

```bash
pquota vervmb
```
