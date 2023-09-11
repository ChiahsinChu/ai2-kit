# CLL Machine Learning Potential Training Workflow

```bash
ai2-kit workflow cll-mlp-training
```

## Introduction

CLL workflow is an improved version of DPGEN workflow, which is designed to meet more complex potential training requirements and sustainable code integration. CLL workflow adopts a closed-loop learning mode to automatically train MLP potential. In each iteration, the workflow uses labeled structures generated by first-principles methods to train multiple MLP models. Then, these models are used to explore new structures as training data for the next iteration. The iteration will continue until the quality of the MLP model meets the predetermined standard. The configuration of each iteration can be updated according to the training needs to further improve the training efficiency.

![cll-mlp-diagram](../res/cll-mlp-diagram.svg)

The main improvements of CLL workflow include:
* More semantic configuration system to support the selection of different software, and set different software configurations according to different systems.
* More robust checkpoint mechanism to reduce the impact of execution interruption.
* Support remote Python method execution to avoid unnecessary data transfer and improve the efficiency and stability of execution on HPC clusters.

Currently, CLL workflow supports the following tools for potential training:
* Label: CP2K, VASP
* Train: DeepMD
* Explore: LAMMPS, LASP
* Select: Model deviation, Distinctive structure selection

Currently, CLL workflow submits jobs through the HPC executor provided by `ai2-kit`, which supports completing calculations in a single HPC cluster. In the future, multi-cluster scheduling and support for different workflow engines including `DFlow` will be considered according to needs.

## Environment Requirements
* The Python version of the workflow execution environment and the HPC execution environment need to be consistent, otherwise there will be problems with remote execution.
* The HPC execution environment needs to install `ai2-kit`. Generally speaking, the `ai2-kit` on the HPC does not need to be strictly the same as the local `ai2-kit` version, but if the difference is too large, there may still be problems, so it is recommended to use the same version of `ai2-kit` when conditions permit.


## Installation

```bash
pip install -U ai2-kit
```

## Usage

The usage of CLL workflow is demonstrated through an example.

### Data Preparation
The data required for the execution of the workflow needs to be placed on the HPC cluster node in advance. Before starting the execution of the workflow, you need to prepare the following data:
* Initial structure or initial data set for potential training
* Initial structure for structure search

Assuming that you already have a trajectory generated by AIMD `h2o_64.aimd.xyz`, then you can use the [`ai2-kit tool ase`](./ase.md) command line tool to prepare these data.

```bash
mkdir -p data/explore

# Extract training set from 0-900 frames, extract every 5 frames
ai2-kit tool ase read h2o_64.aimd.xyz --index ':900:5' - set_cell "[12.42,12.42,12.42,90,90,90]" - write data/training.xyz

# Extract validation set from 900- frames, extract every 5 frames
ai2-kit tool ase read h2o_64.aimd.xyz --index '900::5' - set_cell "[12.42,12.42,12.42,90,90,90]" - write data/validation.xyz

# Extract data for initial structure search, extract every 100 frames
ai2-kit tool ase read h2o_64.aimd.xyz --index '::100' - set_cell "[12.42,12.42,12.42,90,90,90]" - write_each_frame "./data/explore/POSCAR-{i:04d}" --format vasp
```

### Configuration File Preparation

The configuration file of CLL workflow is in YAML format and supports splitting in any dimension. `ai2-kit` will automatically merge them during execution. Moderate splitting is conducive to the maintenance and reuse of configuration files. In general, we can split the configuration file into the following parts:

* artifact.yml: the data required by the workflow
* executor.yml: the parameters of the HPC executor
* workflow.yml: the parameters of the workflow software

Another way to build configuration is to use the configuration file provided in [example](../../example/config/cll-mlp-training/) as a reference to create your own workflow. For details, please refer to the document in the example directory.

We start with `artifact.yml`, which is used to configure the data required by the workflow. In this example, we need to configure three datasets, which are the training dataset, the validation dataset, and the dataset for structure search. The configuration of these three datasets is as follows:


```yaml
.base_dir: &base_dir /home/user01/data/

artifacts:
  h2o_64-train:
    url: !join [*base_dir, training.xyz]
    
  h2o_64-validation:
    url: !join [*base_dir, validation.xyz]
    attrs:
      deepmd:
        validation_data: true  # Specify this dataset as the validation set, Optional

  h2o_64-explore:
    url: !join [*base_dir, explore]
    includes: POSCAR*
    attrs:  
    # If necessary, you can specify specific software configurations for specific systems here. This example does not require this configuration, so it is empty
      # lammps:
      #   plumed_config:  !load_text plumed.in
      # cp2k:
      #   input_template: !load_text cp2k.inp
```

Here we use the custom tag `!join` provided by `ai2-kit` to simplify the data configuration. For related functions, please refer to the [TIPS](./tips.md) document.

Next, we configure the `executor.yml` file, which is used to configure parameters related to the HPC link and the use template of the software.

```yaml
executor:
  hpc-cluster01:
    ssh:
      host: user01@login-01  # Login node
      gateway:
        host: user01@jump-host  # Jump host (optional)
    queue_system:
      slurm: {}  # Use slurm as the job scheduling system

    work_dir: /home/user01/ai2-kit/workdir  # Working directory
    python_cmd: /home/user01/libs/conda/env/py39/bin/python  # Remote Python interpreter

    context:
      train:
        deepmd:  # Configure the deepmd job submission template
          script_template:
            header: |
              #SBATCH -N 1
              #SBATCH --ntasks-per-node=4
              #SBATCH --job-name=deepmd
              #SBATCH --partition=gpu3
              #SBATCH --gres=gpu:1
              #SBATCH --mem=8G
            setup: |
              set -e
              module load deepmd/2.2
              set +e

      explore:
        lammps:  # Configure the lammps job submission template
          lammps_cmd: lmp_mpi
          concurrency: 5
          script_template:
            header: |
              #SBATCH -N 1
              #SBATCH --ntasks-per-node=4
              #SBATCH --job-name=lammps
              #SBATCH --partition=gpu3
              #SBATCH --gres=gpu:1
              #SBATCH --mem=24G
            setup: |
              set -e
              module load deepmd/2.2
              export OMP_NUM_THREADS=1
              set +e

      label:
        cp2k:  # Configure the cp2k job submission template
          cp2k_cmd: mpiexec.hydra cp2k.popt
          concurrency: 5
          script_template:
            header: |
              #SBATCH -N 1
              #SBATCH --ntasks-per-node=16
              #SBATCH -t 12:00:00
              #SBATCH --job-name=cp2k
              #SBATCH --partition=c52-medium
            setup: |
              set -e
              module load intel/17.5.239 mpi/intel/2017.5.239
              module load gcc/5.5.0
              module load cp2k/7.1
              set +e
```

Finally, the configuration of the `workflow.yml` file is the configuration of the parameters of the workflow. This file is used to configure the parameters of the workflow.

```yaml
workflow:
  general:
    type_map: [ H, O ]
    mass_map: [ 1.008, 15.999 ]
    max_iters: 2  # Specify the maximum number of iterations

  train:
    deepmd:  # deepmd parameter configuration
      model_num: 4
      # The data used in this example has not been labeled, so it needs to be configured here. 
      # If there is an existing labeled deepmd/npy dataset, you can specify it here.
      init_dataset: [ ]  
      input_template:
        model:
          descriptor:
            type: se_a  
            sel:
            - 100
            - 100
            rcut_smth: 0.5
            rcut: 5.0
            neuron:
            - 25
            - 50
            - 100
            resnet_dt: false
            axis_neuron: 16
            seed: 1
          fitting_net:
            neuron:
            - 240
            - 240
            - 240
            resnet_dt: true
            seed: 1
        learning_rate:
          type: exp
          start_lr: 0.001
          decay_steps: 2000
        loss:
          start_pref_e: 0.02
          limit_pref_e: 2
          start_pref_f: 1000
          limit_pref_f: 1
          start_pref_v: 0
          limit_pref_v: 0
        training:
          #numb_steps: 400000
          numb_steps: 5000
          seed: 1
          disp_file: lcurve.out
          disp_freq: 1000
          save_freq: 1000
          save_ckpt: model.ckpt
          disp_training: true
          time_training: true
          profiling: false
          profiling_file: timeline.json

  label:
    cp2k:  # Specify the cp2k parameter configuration
      limit: 10
      # The data used in this example has not been labeled, so it needs to be configured here. 
      # If there is an existing labeled dataset, this should be empty
      # When this configuration is empty, the workflow will automatically skip the label phase of the first iteration 
      # and start execution from the train phase.
      init_system_files: [ h2o_64-train, h2o_64-validation ]
      input_template: |
        &GLOBAL
           PROJECT  DPGEN
        &END
        &FORCE_EVAL
           &DFT
              BASIS_SET_FILE_NAME  /home/user01/data/cp2k/BASIS/BASIS_MOLOPT
              POTENTIAL_FILE_NAME  /home/user01/data/cp2k/POTENTIAL/GTH_POTENTIALS
              CHARGE  0
              UKS  F
              &MGRID
                 CUTOFF  600
                 REL_CUTOFF  60
                 NGRIDS  4
              &END
              &QS
                 EPS_DEFAULT  1.0E-12
              &END
              &SCF
                 SCF_GUESS  RESTART
                 EPS_SCF  3.0E-7
                 MAX_SCF  50
                 &OUTER_SCF
                    EPS_SCF  3.0E-7
                    MAX_SCF  10
                 &END
                 &OT
                    MINIMIZER  DIIS
                    PRECONDITIONER  FULL_SINGLE_INVERSE
                    ENERGY_GAP  0.1
                 &END
              &END
              &LOCALIZE
                 METHOD  CRAZY
                 MAX_ITER  2000
                 &PRINT
                    &WANNIER_CENTERS
                       IONS+CENTERS
                       FILENAME  =64water_wannier.xyz
                    &END
                 &END
              &END
              &XC
                 &XC_FUNCTIONAL PBE
                 &END
                 &vdW_POTENTIAL
                    DISPERSION_FUNCTIONAL  PAIR_POTENTIAL
                    &PAIR_POTENTIAL
                       TYPE  DFTD3
                       PARAMETER_FILE_NAME  dftd3.dat
                       REFERENCE_FUNCTIONAL  PBE
                    &END
                 &END
              &END
           &END
           &SUBSYS
              @include coord_n_cell.inc
              &KIND O
                 BASIS_SET  DZVP-MOLOPT-SR-GTH
                 POTENTIAL  GTH-PBE-q6
              &END
              &KIND H
                 BASIS_SET  DZVP-MOLOPT-SR-GTH
                 POTENTIAL  GTH-PBE-q1
              &END
           &END
           &PRINT
              &FORCES ON
              &END
            &END
        &END

  explore:
    lammps:
      timestep: 0.0005
      sample_freq: 100
      nsteps: 2000
      ensemble: nvt

      template_vars:
        POST_INIT: |
          neighbor 1.0 bin
          box      tilt large

        POST_READ_DATA: |
          change_box all triclinic

      system_files: [ h2o-64-explore ]

      explore_vars:
        TEMP: [ 330, 430, 530]
        PRES: [1]
        TAU_T: 0.1  # Optional
        TAU_P: 0.5  # Optional

  select:
    model_devi:
        f_trust_lo: 0.4
        f_trust_hi: 0.6

  update:
    walkthrough:
      # You can specify the parameter configuration to be used from the second iteration and beyond here
      # The parameters configured here will override any configuration in the workflow section, 
      # and can be adjusted according to the training strategy
      table:
        - train:  # The training steps are 10000 in the second iteration
            deepmd:
              input_template:
                training:
                  numb_steps: 10000
        - train:  # The training steps are 20000 in the third iteration
            deepmd:
              input_template:
                training:
                  numb_steps: 20000
```

### Execute Workflow

After completing the configuration, you can start the execution of the workflow

```bash
ai2-kit workflow cll-mlp-training *.yml --executor hpc-cluster01 --path-prefix h2o_64-run-01 --checkpoint run-01.ckpt
```

In the above parameters,
* `*.yml` is used to specify the configuration file, you can specify multiple configuration files, `ai2-kit` will automatically merge them, `*` wildcard is used here;
* `--executor hpc-cluster01` is used to specify the HPC executor to use. Here, the `hpc-cluster01` executor configured in the previous section is used;
* `--path-prefix h2o_64-run-01` specifies the remote working directory, which will create a `h2o_64-run-01` directory under `work_dir` to store the execution results of the workflow;
* `--checkpoint run-01.cpkt` will generate a checkpoint file locally to save the execution status of the workflow, so as to resume execution after the execution is interrupted.