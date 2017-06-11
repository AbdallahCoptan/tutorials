-*- mode: markdown; mode: visual-line; fill-column: 80 -*-

Copyright (c) 2013-2017 UL HPC Team  <hpc-sysadmins@uni.lu>

        Time-stamp: <Sun 2017-06-11 19:12 svarrette>

---------------------------------------------------------------
# UL HPC MPI Tutorial: OSU Micro-Benchmarks

[![By ULHPC](https://img.shields.io/badge/by-ULHPC-blue.svg)](https://hpc.uni.lu) [![Licence](https://img.shields.io/badge/license-GPL--3.0-blue.svg)](http://www.gnu.org/licenses/gpl-3.0.html) [![GitHub issues](https://img.shields.io/github/issues/ULHPC/tutorials.svg)](https://github.com/ULHPC/tutorials/issues/) [![](https://img.shields.io/badge/slides-PDF-red.svg)](slides.pdf) [![Github](https://img.shields.io/badge/sources-github-green.svg)](https://github.com/ULHPC/tutorials/tree/devel/advanced/OSU_MicroBenchmarks) [![Documentation Status](http://readthedocs.org/projects/ulhpc-tutorials/badge/?version=latest)](http://ulhpc-tutorials.readthedocs.io/en/latest/advanced/OSU_MicroBenchmarks/) [![GitHub forks](https://img.shields.io/github/stars/ULHPC/tutorials.svg?style=social&label=Star)](https://github.com/ULHPC/tutorials)

<!-- [![](cover_slides.png)](slides.pdf) -->

The objective of this tutorial is to compile and run on of the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/) which permit to measure the performance of an MPI implementation.

You can work in groups for this training, yet individual work is encouraged to
ensure you understand and practice the usage of MPI programs on an HPC platform.

In all cases, ensure you are able to [connect to the UL HPC  clusters](https://hpc.uni.lu/users/docs/access.html).


```bash
# /!\ FOR ALL YOUR COMPILING BUSINESS, ENSURE YOU WORK ON A COMPUTING NODE
# Have an interactive job
(access)$> si -N 2 --ntasks-per-node=1                    # iris
(access)$> srun -p interactive --qos qos-iteractive -N 2 --ntasks-per-node=1 --pty bash  # iris (long version)
(access)$> oarsub -I -l enclosure=1/nodes=2,walltime=4   # chaos / gaia
```

**Advanced users only**: rely on `screen` (see  [tutorial](http://support.suso.com/supki/Screen_tutorial) or the [UL HPC tutorial](https://hpc.uni.lu/users/docs/ScreenSessions.html) on the  frontend prior to running any `oarsub` or `srun/sbatch` command to be more resilient to disconnection.

The latest version of this tutorial is available on [Github](https://github.com/ULHPC/tutorials/tree/devel/advanced/OSU_MicroBenchmarks)

## Objectives

The [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/) feature a series of MPI benchmarks that measure the performances of various MPI operations:

* __Point-to-Point MPI Benchmarks__: Latency, multi-threaded latency, multi-pair latency, multiple bandwidth / message rate test bandwidth, bidirectional bandwidth
* __Collective MPI Benchmarks__: Collective latency tests for various MPI collective operations such as MPI_Allgather, MPI_Alltoall, MPI_Allreduce, MPI_Barrier, MPI_Bcast, MPI_Gather, MPI_Reduce, MPI_Reduce_Scatter, MPI_Scatter and vector collectives.
* __One-sided MPI Benchmarks__: one-sided put latency (active/passive), one-sided put bandwidth (active/passive), one-sided put bidirectional bandwidth, one-sided get latency (active/passive), one-sided get bandwidth (active/passive), one-sided accumulate latency (active/passive), compare and swap latency (passive), and fetch and operate (passive) for MVAPICH2 (MPI-2 and MPI-3).
* Since the 4.3 version, the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/) also features OpenSHMEM benchmarks, a 1-sided communications library.

In this tutorial, we will build **version 5.3.2 of the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/)** (the latest at the time of writing), and focus on two of the available tests:

* `osu_get_latency` - Latency Test
* `osu_get_bw` - Bandwidth Test

> The latency tests are carried out in a ping-pong fashion. The sender sends a message with a certain data size to the receiver and waits for a reply from the receiver. The receiver receives the message from the sender and sends back a reply with the same data size. Many iterations of this ping-pong test are carried out and average one-way latency numbers are obtained. Blocking version of MPI functions (MPI_Send and MPI_Recv) are used in the tests.

> The bandwidth tests were carried out by having the sender sending out a fixed number (equal to the window size) of back-to-back messages to the receiver and then waiting for a reply from the receiver. The receiver sends the reply only after receiving all these messages. This process is repeated for several iterations and the bandwidth is calculated based on the elapsed time (from the time sender sends the first message until the time it receives the reply back from the receiver) and the number of bytes sent by the sender. The objective of this bandwidth test is to determine the maximum sustained date rate that can be achieved at the network level. Thus, non-blocking version of MPI functions (MPI_Isend and MPI_Irecv) were used in the test.

The idea is to compare the different MPI implementations available on the [UL HPC platform](http://hpc.uni.lu).:

* [Intel MPI](http://software.intel.com/en-us/intel-mpi-library/)
* [OpenMPI](http://www.open-mpi.org/)
* [MVAPICH2](http://mvapich.cse.ohio-state.edu/overview/) (MPI-3 over OpenFabrics-IB, Omni-Path, OpenFabrics-iWARP, PSM, and TCP/IP)

For the sake of time and simplicity, we will focus on the first two suits. Eventually, the benchmarking campain will typically involves for each MPI suit:

* two nodes, belonging to the _same_ enclosure
* two nodes, belonging to _different_ enclosures

## Pre-requisites

On the **access** node of the cluster you're working on, clone the [ULHPC/tutorials](https://github.com/ULHPC/tutorials)  and [ULHPC/launcher-scripts](https://github.com/ULHPC/launcher-scripts) repositories

```bash
$> cd
$> mkdir -p git/ULHPC && cd  git/ULHPC
$> git clone https://github.com/ULHPC/launcher-scripts.git
$> git clone https://github.com/ULHPC/tutorials.git         # If not yet done
```

Prepare your working directory

```bash
$> mkdir -p ~/tutorials/OSU-MicroBenchmarks
$> cd ~/tutorials/OSU-MicroBenchmarks
$> ln -s ~/git/ULHPC/tutorials/advanced/OSU_MicroBenchmarks ref.ulhpc.d   # Keep a symlink to the reference tutorial
$> ln -s ref.ulhpc.d/Makefile .     # symlink to the root Makefile
```

Fetch and uncompress the latest version of the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/)

```bash
$> cd ~/tutorials/OSU-MicroBenchmarks
$> mkdir src
$> cd src
# Download the latest version
$>
```


    $> mkdir ~/TP
    $> cd ~/TP
    $> wget --no-check-certificate https://scm.mvapich.cse.ohio-state.edu/benchmarks/osu-micro-benchmarks-4.3.tar.gz
    $> tar xvzf osu-micro-benchmarks-4.3.tar.gz
    $> cd osu-micro-benchmarks-4.3


## OSU Micro-benchmarks with Intel MPI

We are first going to use the
[Intel Cluster Toolkit Compiler Edition](http://software.intel.com/en-us/intel-cluster-toolkit-compiler/),
which provides Intel C/C++ and Fortran compilers, Intel MPI.

We will compile the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/) in a specific directory (that a good habbit)

    $> cd ~/TP/osu-micro-benchmarks-4.3
    $> module avail MPI
    $> module load toolchain/ictce/7.3.5
    $> module list
    Currently Loaded Modules:
      1) compiler/icc/2015.3.187     3) toolchain/iccifort/2015.3.187            5) toolchain/iimpi/7.3.5                7) toolchain/ictce/7.3.5
      2) compiler/ifort/2015.3.187   4) mpi/impi/5.0.3.048-iccifort-2015.3.187   6) numlib/imkl/11.2.3.187-iimpi-7.3.5

    $> mkdir build.impi && cd build.impi
    $> ../configure CC=mpiicc --prefix=`pwd`/install
    $> make && make install

If everything goes fine, you shall have the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/) installed in the directory `install/libexec/osu-micro-benchmarks/mpi/`.


Once compiled, ensure you are able to run it:

	$> cd install/libexec/osu-micro-benchmarks/mpi/one-sided/
	$> mpirun -hostfile $OAR_NODEFILE -perhost 1 ./osu_get_latency
	$> mpirun -hostfile $OAR_NODEFILE -perhost 1 ./osu_get_bw

Now you can use the [MPI generic launcher](https://github.com/ULHPC/launcher-scripts/blob/devel/bash/MPI/mpi_launcher.sh) to run the code:

	$> cd ~/TP/osu-micro-benchmarks-4.3/
	$> mkdir runs  && cd runs
	$> ln -s ~/git/ULHPC/launcher-scripts/bash/MPI/mpi_launcher.sh launcher_osu_impi
	$> ./launcher_osu_impi --basedir $HOME/TP/osu-micro-benchmarks-4.3/build.impi/install/libexec/osu-micro-benchmarks/mpi/one-sided --npernode 1 --module toolchain/ictce/7.3.5 --exe osu_get_latency,osu_get_bw

If you want to avoid this long list of arguments, just create a file `launcher_osu_impi.default.conf` to contain:

	$> cat launcher_osu_impi.default.conf # this command will fail if you have not already created the file !
	# Defaults settings for running the OSU Micro benchmarks compiled with Intel MPI
	NAME=impi

	MODULE_TO_LOADstr=toolchain/ictce/7.3.5
	MPI_PROG_BASEDIR=$HOME/TP/osu-micro-benchmarks-4.3/build.impi/install/libexec/osu-micro-benchmarks/mpi/one-sided/

	MPI_PROGstr=osu_get_latency,osu_get_bw
	MPI_NPERNODE=1

Now you can run the launcher script interactively.

	$> ./launcher_osu_impi

You might want also to host the output files in the local directory (under the date)

	$> ./launcher_osu_impi --datadir data/`date +%Y-%m-%d`

## OSU Micro-benchmarks with OpenMPI

We will repeat the procedure, this time using OpenMPI.

	$> cd ~/TP/osu-micro-benchmarks-4.3/
	$> module purge
	$> module load mpi/OpenMPI/1.8.4-GCC-4.9.2
	$> mkdir build.openmpi && cd build.openmpi
	$> ../configure CC=mpicc --prefix=`pwd`/install
	$> make && make install

If everything goes fine, you shall have the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/) installed in the directory `install/libexec/osu-micro-benchmarks/mpi/`.

Once compiled, ensure you are able to run it:

	$> cd install/libexec/osu-micro-benchmarks/mpi/one-sided/
	$> mpirun -x LD_LIBRARY_PATH -hostfile $OAR_NODEFILE -npernode 1 ./osu_get_latency
	$> mpirun -x LD_LIBRARY_PATH -hostfile $OAR_NODEFILE -npernode 1 ./osu_get_bw

Again, we will rely on the [MPI generic launcher](https://github.com/ULHPC/launcher-scripts/blob/devel/bash/MPI/mpi_launcher.sh) to run the code:

	$> cd ~/TP/osu-micro-benchmarks-4.3/runs
	$> ln -s ~/git/ULHPC/launcher-scripts/bash/MPI/mpi_launcher.sh launcher_osu_openmpi
	$> cat launcher_osu_openmpi.default.conf  # this command will fail if you have not already created the file !
	# Defaults settings for running the OSU Micro benchmarks wompiled with OpenMPI
	NAME=openmpi

	MODULE_TO_LOADstr=mpi/OpenMPI/1.8.4-GCC-4.9.2
	MPI_PROG_BASEDIR=$HOME/TP/osu-micro-benchmarks-4.3/build.openmpi/install/libexec/osu-micro-benchmarks/mpi/one-sided/

	MPI_PROGstr=osu_get_latency,osu_get_bw
	MPI_NPERNODE=1

Now you can run the launcher script interactively.

	$> ./launcher_osu_openmpi

You might want also to host the output files in the local directory (under the date)

	$> ./launcher_osu_openmpi --datadir data/`date +%Y-%m-%d`


## Benchmarking on two nodes

Operate the benchmarking campain (in the two cases) in the following context:

* 2 nodes belonging to the same enclosure. Use for that:

		$> oarsub -l enclosure=1/nodes=2,walltime=8 […]

* 2 nodes belonging to the different enclosures:

		$> oarsub -l enclosure=2/core=1,walltime=8 […]

## Now for Lazy / frustrated persons

You will find in the [UL HPC tutorial](https://github.com/ULHPC/tutorials)
repository, under the `advanced/OSU_MicroBenchmarks` directory, a set of tools / script that
facilitate the running and analysis of this tutorial that you can use/adapt to
suit your needs.

In particular, once in the `advanced/OSU_MicroBenchmarks` directory:

* running `make fetch` will automatically download the archives for the [OSU micro-benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/) in the `src/` directory
* you will find the patch file to apply to the version 4.3 in `src/osu-micro-benchmarks-4.3/mpi/one-sided/Makefile.am.patch`
* The different configuration files for the [MPI generic launcher](https://github.com/ULHPC/launcher-scripts/blob/devel/bash/MPI/mpi_launcher.sh) in `runs/`
* Some sample output data in `runs/data/`
* run `make build` to build the different versions of the OSU Micro-benchmarks
* run `make run_interactive` to run the benchmarks, assuming you are in an interactive job
* run `make run` to run a passive job executing the benchmarks
* run `make plot` to invoke the [Gnuplot](http://www.gnuplot.info/) script
  `plots/benchmark_OSU.gnuplot` and generate various plots from the sample
  runs.

In particular, you'll probably want to see the comparison figure extracted from
the sample run in `plots/benchmark_OSU_2H_latency.pdf` and `plots/benchmark_OSU_2H_bandwidth.pdf`

A PNG version of these plots is available on Github:
[OSU latency](https://raw.github.com/ULHPC/tutorials/devel/advanced/OSU_MicroBenchmarks/plots/benchmark_OSU_2H_latency.png) -- [OSU Bandwidth](https://raw.github.com/ULHPC/tutorials/devel/advanced/OSU_MicroBenchmarks/plots/benchmark_OSU_2H_bandwidth.png)

![OSU latency](https://raw.github.com/ULHPC/tutorials/devel/advanced/OSU_MicroBenchmarks/plots/benchmark_OSU_2H_latency.png)
![OSU Bandwidth](https://raw.github.com/ULHPC/tutorials/devel/advanced/OSU_MicroBenchmarks/plots/benchmark_OSU_2H_bandwidth.png)
