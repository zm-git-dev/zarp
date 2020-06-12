# ZARP pipeline

[Snakemake][snakemake] workflow for general purpose RNA-Seq library annotation
developed by the [Zavolan lab][zavolan-lab].

Reads are processed, aligned, quantified and analyzed with state-of-the-art
tools to give meaningful initial insights into various aspects of an RNA-Seq
library while cutting down on hands-on time for bioinformaticians.

Below is a schematic representation of the individual workflow steps ("pe"
refers to "paired-end"):

> ![rule_graph][rule-graph]

For a more detailed description of each step, please refer to the [pipeline
documentation][pipeline-documentation].

## Requirements

Currently the workflow is only available for Linux distributions. It was tested
on the following distributions:

- CentOS 7.5
- Debian 10
- Ubuntu 16.04, 18.04

## Installation

### Cloning the repository

Traverse to the desired path on your file system, then clone the repository and
move into it with:

```bash
git clone ssh://git@git.scicore.unibas.ch:2222/zavolan_group/pipelines/zarp.git
cd zarp
```

### Installing Conda

Workflow dependencies can be conveniently installed with the [Conda][conda]
package manager. We recommend that you install
[Miniconda][miniconda-installation] for your system (Linux). Be sure to select
Python 3 option. The workflow was built and tested with `miniconda 4.7.12`.
Other versions are not guaranteed to work as expected.

### Installing dependencies

For improved reproducibility and reusability of the workflow,
each individual step of the workflow runs in its own [Singularity][singularity]
container. As a consequence, running this workflow has very few
individual dependencies. It does, however, require Singularity to be installed
on the system running the workflow. As the functional installation of
Singularity requires root privileges, and Conda currently only provides
Singularity for Linux architectures, the installation instructions are
slightly different depending on your system/setup:

#### For most users

If you do *not* have root privileges on the machine you want to run the
workflow on *or* if you do not have a Linux machine, please [install
Singularity][singularity-install] separately and in privileged mode, depending
on your system. You may have to ask an authorized person (e.g., a systems
administrator) to do that. This will almost certainly be required if you want
to run the workflow on a high-performance computing (HPC) cluster. We have
successfully tested the workflow with the following Singularity versions:

- `v2.4.5`
- `v2.6.2`
- `v3.5.2`

After installing Singularity, install the remaining dependencies with:

```bash
conda env create -f install/environment.yml
```

#### As root user on Linux

If you have a Linux machine, as well as root privileges, (e.g., if you plan to
run the workflow on your own computer), you can execute the following command
to include Singularity in the Conda environment:

```bash
conda env create -f install/environment.root.yml
```

### Activate environment

Activate the Conda environment with:

```bash
conda activate zarp
```

### Installing non-essential dependencies

Most tests have additional dependencies. If you are planning to run tests, you
will need to install these by executing the following command _in your active
Conda environment_:

```bash
conda env update -f install/environment.dev.yml
```

## Testing the installation

We have prepared several tests to check the integrity of the workflow, its
components and non-essential processing scripts. These can be found in
subdirectories of the `tests/` directory. The most critical of these tests
lets you execute the entire workflow on a small set of example input files.
Note that for this and other tests to complete without issues,
[additional dependencies](#installing-non-essential-dependencies) need to be
installed.

### Run workflow on local machine

Execute the following command to run the test workflow on your local machine:

```bash
bash tests/test_integration_workflow/test.local.sh
```

### Run workflow via Slurm

Execute the following command to run the test workflow on a
[Slurm][slurm]-managed high-performance computing (HPC) cluster:

```bash
bash tests/test_integration_workflow/test.slurm.sh
```

> **NOTE:** Depending on the configuration of your Slurm installation or if
> using a different workflow manager, you may need to adapt file `cluster.json`
> and the arguments to options `--config` and `--cores` in file
> `test.slurm.sh`, both located in directory `tests/test_integration_workflow`.
> Consult the manual of your workload manager as well as the section of the
> Snakemake manual dealing with [cluster execution].

## Running the workflow on your own samples

1. Assuming that you are currently inside the repository's root directory,
create a directory for your workflow run and traverse inside it with:

    ```bash
    mkdir config/my_run
    cd config/my_run
    ```

2. Create empty sample table, workflow configuration and, if necessary, cluster
configuration files:

    ```bash
    touch samples.tsv
    touch config.yaml
    touch cluster.json
    ```

3. Use your editor of choice to manually populate these files with appropriate
values. Have a look at the examples in the `tests/` directory to see what the
files should look like, specifically:

    - [samples.tsv](tests/input_files/samples.tsv)
    - [config.yaml](tests/input_files/config.yaml)
    - [cluster.json](tests/input_files/cluster.json)

4. Create a runner script. Pick one of the following choices for either local
or cluster execution. Before execution of the respective command, you must
replace the data directory placeholders in the argument to the
`--singularity-args` option with a comma-separated list of _all_ directories
containing input data files (samples and any annoation files etc) required for
your run.

    Runner script for _local execution_:

    ```bash
    cat << "EOF" > run.sh
    #!/bin/bash
    snakemake \
        --snakefile="../../snakemake/Snakefile" \
        --configfile="config.yaml" \
        --cores=4 \
        --printshellcmds \
        --rerun-incomplete \
        --use-singularity \
        --singularity-args="--bind <data_dir_1>,<data_dir_2>,<data_dir_n>"
    EOF
    ```

    **OR**

    Runner script for _Slurm cluster exection_ (note that you may need
    to modify the arguments to `--cluster` and `--cores` depending on your HPC
    and workload manager configuration):

    ```bash
    cat << "EOF" > run.sh
    #!/bin/bash
    mkdir -p logs/cluster_log
    snakemake \
        --snakefile="../../snakemake/Snakefile" \
        --configfile="config.yaml" \
        --cluster-config="cluster.json" \
        --cluster="sbatch --cpus-per-task={cluster.threads} --mem={cluster.mem} --qos={cluster.queue} --time={cluster.time} --job-name={cluster.name} -o {cluster.out} -p scicore" \
        --cores=256 \
        --printshellcmds \
        --rerun-incomplete \
        --use-singularity \
        --singularity-args="--bind <data_dir_1>,<data_dir_2>,<data_dir_n>"
    EOF
    ```

5. Start your workflow run:

    ```bash
    bash run.sh
    ```

### Configuring workflow runs via LabKey tables

Our lab stores metadata for sequencing samples in a locally deployed
[LabKey][labkey] instance. This repository provides two scripts that give
programmatic access to the LabKey data table and convert it to the
corresponding workflow inputs (`samples.tsv` and `config.yaml`), respectively.
As such, these scripts largely automate step 3. of the above instructions.
However, as these scripts were specifically for the needs of our lab, they are
likely not portable or, at least, will require considerable modification for
other setups (e.g., different LabKey table structure).

> **NOTE:** All of the below steps assume that your current working directory
> is the repository's root directory.

1. The scripts have additional dependencies that can be installed with:

    ```bash
    pip install -r scripts/requirements.txt
    ```

2. In order to gain programmatic access to LabKey via its API, a credential
file is required. Create it with the following command after replacing the
placeholder values with your real credentials (talk to your LabKey manager if
you do not have these):

    ```bash
    cat << EOF | ( umask 0377; cat >> ${HOME}/.netrc; )
    machine <remote-instance-of-labkey-server>
    login <user-email>
    password <user-password>
    EOF
    ```

3. Generate the workflow configuration with the following command, after
replacing the placeholders with the appropriate values (check out the
help screen with option '--help' for further options and information):

    ```bash
    python scripts/prepare_inputs.py \
        --labkey-domain="my.labkey.service.io"
        --labkey-domain="/my/project/path"
        --input-to-output-mapping="scripts/prepare_inputs.dict.tsv" \
        --resources-dir="/path/to/my/genome/resources" \
        --output-table="config/my_run/samples.tsv" \
        --config_file="config/my_run/config.yaml" \
        <table_name>
    ```

#### Additional information

The metadata field names in the LabKey instance and those in the parameters
in the Snakemake workflow have different names. A mapping between LabKey
field identifiers and Snakemake parameters is listed below:

Labkey | Snakemake
--- | ---
Entry date | entry_date
Path to FASTQ file(s) | fastq_path
Condition name | condition
Replicate name | replicate_name
End type (PAIRED or SINGLE) | seqmode
Name of Mate1 FASTQ file | fq1
Name of Mate2 FASTQ file | fq2
Direction of Mate1 (SENSE, ANTISENSE or RANDOM) | mate1_direction
Direction of Mate2 (SENSE, ANTISENSE or RANDOM) | mate2_direction
5' adapter of Mate1 | fq1_5p
3' adapter of Mate1 | fq1_3p
5' adapter of Mate2 | fq2_5p
3' adapter of Mate2 | fq2_3p
Fragment length mean | mean
Fragment length SD | sd
Quality control flag (PASSED or FAILED) | quality_control_flag
Checksum of raw Mate1 FASTQ file | mate1_checksum
Checksum of raw Mate2 FASTQ file | mate2_checksum
Name of metadata file | metadata
Name of quality control file for Mate1 | mate1_quality
Name of quality control file for Mate2 | mate2_quality
Organism | organism
Taxon ID | taxon_id
Name of Strain / Isolate / Breed / Ecotype | strain_name
Strain / Isolate / Breed / Ecotype ID | strain_id
Biomaterial provider | biomaterial_provider
Source / tissue name | source_name
Tissue code | tissue_code
Additional tissue description | tissue_description
Genotype short name | genotype_name
Genotype description | genotype_description
Disease short name | disease_name
Disease description | disease_description
Abbreviation for treatment | treatment
Treatment description | treatment_description
Gender | gender
Age | age
Developmental stage | development_stage
Passage number | passage_number
Sample preparation date (YYYY-MM-DD) | sample_prep_date
Prepared by | prepared_by
Documentation | documentation
Name of protocol file | protocol_file
Sequencing date (YYYY-MM-DD) | seq_date
Sequencing instrument | seq_instrument
Library preparation kit | library_kit
Cycles | cycles
Molecule | molecule
Contaminant sequences | contaminant_seqs

[conda]: <https://docs.conda.io/projects/conda/en/latest/index.html>
[cluster execution]: <https://snakemake.readthedocs.io/en/stable/executing/cluster-cloud.html#cluster-execution>
[labkey]: <https://www.labkey.com/>
[miniconda-installation]: <https://docs.conda.io/en/latest/miniconda.html>
[rule-graph]: images/rule_graph.svg
[snakemake]: <https://snakemake.readthedocs.io/en/stable/>
[singularity]: <https://sylabs.io/singularity/>
[singularity-install]: <https://sylabs.io/guides/3.5/admin-guide/installation.html>
[slurm]: <https://slurm.schedmd.com/documentation.html>
[zavolan-lab]: <https://www.biozentrum.unibas.ch/research/researchgroups/overview/unit/zavolan/research-group-mihaela-zavolan/>
[pipeline-documentation]: pipeline_documentation.md
