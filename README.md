# Trireme

# Overview

The Trireme© framework is a tool for automatically exploring hierarchical multi-level parallelism for domain specific hardware acceleration. Trireme identifies and selects HW accelerators directly from the application source files, while suggesting parallelism-related optimizations (Task Level, Loop Level and Pipeline Parallelism). It is built within LLVM9 compiler infrastructure and consists of Analysis Passes that estimate Software (SW) latency, Hardware (HW) latency, Area and I/O requirements. Subsequently, an exact 
selection algorithm selects the subset of HW accelerators that maximizes performance (speedup) under a user
defined area (HW resources) budget.

If you use Trireme in your research, we would appreciate a citation to:
```shell
@article{10.1145/3580394,
	author = {Zacharopoulos, Georgios and Ejjeh, Adel and Jing, Ying and Yang, En-Yu and Jia, Tianyu and Brumar, Iulian and Intan, Jeremy and Huzaifa, Muhammad and Adve, Sarita and Adve, Vikram and Wei, Gu-Yeon and Brooks, David},
	title = {Trireme: Exploration of Hierarchical Multi-Level Parallelism for Hardware Acceleration},
	year = {2023},
	publisher = {Association for Computing Machinery},
	address = {New York, NY, USA},
	issn = {1539-9087},
	url = {https://doi.org/10.1145/3580394},
	doi = {10.1145/3580394},
	note = {Just Accepted},
	journal = {ACM Trans. Embed. Comput. Syst.},
	month = {jan},
	keywords = {compiler techniques and optimizations, design tools, heterogeneous systems parallelism, accelerators, ASICs}
}

```
# Prerequisite

__We recommend you to install HPVM first.__ Please follow the [instructions](https://hpvm.readthedocs.io/en/latest/build-hpvm.html) to install HPVM:

    git clone https://gitlab.engr.illinois.edu/llvm/hpvm-release.git -b v1.0 --recursive

Please also note that HPVM v1.0 requires `CMake>=3.18`. Ubuntu 20.04 locks CMake at 3.16, so you may want to install your own CMake and add it to the environment path. HPVM also requires CUDNN 7. One way to circumvent the CUDNN version problem is to use [Anaconda](https://www.anaconda.com/). For example, you can run the following commands to generate a conda environment for building and running Trireme:
```shell
conda create --name trireme python=3.6 cudatoolkit=10.1 cudnn=7.6.5
conda activate trireme
```
Inside the [`set_paths.sh`](https://gitlab.engr.illinois.edu/llvm/hpvm-release/-/blob/v1.0/hpvm/set_paths.sh) file, please set [`CUDA_TOOLKIT_PATH`](https://gitlab.engr.illinois.edu/llvm/hpvm-release/-/blob/v1.0/hpvm/set_paths.sh#L4) to your Anaconda environment path (E.g. `$PATH_TO_YOUR_ANACONDA/env/trireme`.). Please also change [`CUDA_LIB_PATH`](https://gitlab.engr.illinois.edu/llvm/hpvm-release/-/blob/v1.0/hpvm/set_paths.sh#L10) to `CUDA_LIB_PATH=$CUDA_TOOLKIT_PATH/lib/`. Please then do a source of `set_paths.sh` to configure the environment variables:

    source set_paths.sh

 When building HPVM inside the HPVM folder (e.g. `hpvm-release/hpvm`), you can run the following command to avoid building the python-related parts of HPVM:

    bash ./install.sh -j $N -t X86 --no-pypkg --no-params

`N` is the desired number of threads to build HPVM.

When [testing](https://hpvm.readthedocs.io/en/latest/build-hpvm.html#tests) your built HPVM by `check_hpvm_pass` under `hpvm-release/hpvm/build`, you may find that the tests related to the python packages fail. This will not affect the functionality of Trireme, so please feel free to ignore them.

# Installation

You can download the Trireme repository using the following code:    

    git clone --branch hpvm-integration https://github.com/harvard-acc/Trireme.git

Tireme comes with an automated installer that installs the HPVM compiler then sets up Trireme as a sub-project in HPVM. The installer also has an option to use an existing HPVM installation.

To install Trireme:
```shell
bash install.sh [flags]
[flags]:
  -c  Clone HPVM automatically and run its install script.
  -d  Location of HPVM repo (if available) or where HPVM 
      repo will be cloned (if -c provided). REQUIRED.
  -j  Specifies how many threads to use when running make.
  -p  Add environment variable exports to bashrc.
  -h  Prints the help message.
```

The install script will generate a `set_paths.sh` script which needs to be sourced to set required environment variables before running any Trireme components. This can be done using:
```shell
source set_paths.sh
```
Note that it is necessary to source the script in order for the environment variables to get updated in the main shell.

## Suggested Installation Options
### If HPVM is not already installed
```shell
bash install.sh -c -d ./hpvm -j N
source set_paths.sh
```
This will clone and install HPVM under `Trireme/hpvm`. Then it will copy `hpvm-trireme` into `hpvm/hpvm/projects` and build HPVM with Trireme. You can replace `./hpvm` with any alternative relative or absolute path that will be used as the root of the HPVM repo. `N` has to be set to the desired number of threads.

### If HPVM is already installed
```shell
bash install.sh -d path/to/your/hpvm -j N
source set_paths.sh
```
This will set up the Trireme components in your existing HPVM repo. The path provided to the script has to be the root of the HPVM repository (i.e. the folder that contains `hpvm` directory). As above, set `N` to the desired number of threads to be used for building HPVM.

# Usage

For testing, audio decoder https://github.com/ILLIXR/audio_pipeline from the [ILLIXR](https://github.com/ILLIXR/ILLIXR), the Illinois Extended Reality testbed, of University of Illinois at Urbana-Champaign is used. The audio pipeline is responsible for both recording and playing back spatialized audio for XR. The audio decoder here is used to play back the encoded XR audio.

    cd audioDecoding

When working on other applications, you can generate the HPVM IR (e.g. `main.hpvm.ll` in audioDecoding) first and follow the following steps for using Trireme.

<!-- ### 1) Collect dynamic profiling information and generate the annotated  Intermediate Representation (IR) files.

    cd sim

We make sure that the LLVM lines in "Makefile_AccelSeeker" point to the path of the LLVM9 build and lib directory:    

    BIN_DIR_LLVM=path/to/llvm/build/bin
    LIB_DIR_LLVM=path/to/llvm/build/lib

Then we run the instrumented binary with the appropriate input parameters and generate the annotated IR files using
the profiling information.    

    make profile -->

### 1) Identification of candidates for acceleration and estimation of Latency, Area and I/O requirements.   

We make sure that the LLVM_BUILD line in "run_sys_aw.sh" points to the path of the LLVM9 build directory:

    LLVM_BUILD=path/to/hpvm/build

The following script invokes the AccelSeeker Analysis passes and generates the files needed to construct the final Merit/Cost estimation.
The files generated are: FCI.txt  IO.txt  LA.txt 
    
    ./run_sys_aw.sh


### 2) Merit, Cost Estimation of candidates for acceleration and application of the Overlapping Rule.

The following script generates the Merit/Cost (MC) file along with the implementation of the Overlapping rule in the final Merit/Cost/Indexes (MCI) file.
The files generated are: MCI.txt  MC.txt

    ./generate_accelcands_list.sh

The MCI.txt format is as follows:

BENCHMARK-NAME ACCELERATOR-NAME MERIT(CYCLES SAVED) COST(LUTS) FUNCTION_INDEXES

** Modifications are needed to comply for every benchmark. **

# Author

Georgios Zacharopoulos georgios@seas.harvard.edu Date: July, 2021
