# Unison for ns-3

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.10077300.svg)](https://doi.org/10.5281/zenodo.10077300)
[![CI](https://github.com/NASA-NJU/UNISON-for-ns-3/actions/workflows/per_commit.yml/badge.svg)](https://github.com/NASA-NJU/UNISON-for-ns-3/actions/workflows/per_commit.yml)

A fast and user-transparent parallel simulator implementation for ns-3.

With fine-grained partition and load-adaptive scheduling, Unison allows users to easily simulate models with multithreaded parallelization without further configurations.
Meanwhile, cache misses are reduced by fine-grained partition, and the mutual waiting time among threads is minimized by load-adaptive scheduling, resulting in efficient parallelization.
More information about Unison can be found in our [EuroSys '24 paper](https://dl.acm.org/doi/10.1145/3627703.3629574).

Supported ns-3 version: >= 3.36.1.
We are trying to keep Unison updated with the latest version of ns-3.
You can find each unison-enabled ns-3 version via `unison-*` tags.

## Getting Started

The quickest way to get started is to type the command

```shell
./ns3 configure --enable-mtp --enable-examples
```

> The build profile is set to default (which uses `-O2 -g` compiler flags) in this case.
> If you want to get `-O3` optimized build and discard all log outputs, please add `-d optimized` arguments.

The `--enable-mtp` option will enable multi-threaded parallelization.
You can verify Unison is enabled by checking whether `Multithreaded Simulation : ON` appears in the optional feature list.

Now, let's build and run a DCTCP example with default sequential simulation and parallel simulation (using 4 threads) respectively:

```shell
./ns3 build dctcp-example dctcp-example-mtp
time ./ns3 run dctcp-example
time ./ns3 run dctcp-example-mtp
```

The simulation should finish in 4-5 minutes for `dctcp-example` and 1-2 minutes for `dctcp-example-mtp`, depending on your hardware and your build profile.
The output in `*.dat` should be in accordance with the comments in the source file.

The speedup of Unison is more significant for larger topologies and traffic volumes.
If you are interested in using it to simulate topologies like fat-tree, BCube and 2D-torus, please refer to [Running Evaluations](#running-evaluations).

## Speedup Your Existing Code

To understand how Unison affects your model code, let's find the differences between two versions of the source files of the above example:

```shell
diff examples/tcp/dctcp-example.cc examples/mtp/dctcp-example-mtp.cc
```

It turns out that to bring Unison to the existing model code, all you need to do is to include the `ns3/mtp-interface.h` header file and add the following line at the beginning of the `main` function:

```c++
MtpInterface::Enable(numberOfThreads);
```

The parameter `numberOfThreads` is optional.
If it is omitted, the number of threads is automatically chosen and will not exceed the maximum number of available hardware threads on your system.
If you want to enable Unison for distributed simulation on existing MPI programs for further speedup, place the above line before MPI initialization and do not explicitly specify the simulator implementation in your code.
For such hybrid simulation with MPI, the `--enable-mpi` option is also required when configuring ns-3.

Unison resolved a lot of thread-safety issues with ns-3's architecture.
You don't need to consider these issues on your own for most of the time, except if you have custom global statistics other than the built-in flow-monitor.
In the latter case, if multiple nodes can access your global statistics, you can replace them with atomic variables via `std::atomic<>`.
When collecting tracing data such as Pcap, it is strongly recommended to create separate output files for each node instead of a single trace file.
For complex custom data structures, you can create critical sections by adding

```c++
MtpInterface::CriticalSection cs;
```

at the beginning of your methods.

## Examples

In addition to the DCTCP example, you can find other adapted examples in `examples/mtp`.
Meanwhile, Unison also supports manual partition, and you can find a minimal example in `src/mtp/examples/simple-mtp.cc`
For hybrid simulation with MPI, you can find a minimal example in `src/mpi/examples/simple-hybrid.cc`.

We also provide three detailed fat-tree examples for Unison, traditional MPI parallel simulation and hybrid simulation:

| Name | Location | Required configuration flags | Running commands |
| - | - | - | - |
| fat-tree-mtp | src/mtp/examples/fat-tree-mtp.cc | `--enable-mtp --enable-exaples` without `--enable-mpi` | `./ns3 run "fat-tree-mtp --thread=4"` |
| fat-tree-mpi | src/mpi/examples/fat-tree-mpi.cc | `--enable-mpi --enable-exaples` without `--enable-mtp` | `./ns3 run fat-tree-mpi --command-template "mpirun -np 4 %s"` |
| fat-tree-hybrid | src/mpi/examples/fat-tree-hybrid.cc | `--enable-mtp --enable-mpi --enable-exaples` | `./ns3 run fat-tree-hybrid --command-template "mpirun -np 2 %s --thread=2"` |

Feel free to explore these examples, compare code changes and adjust the `-np` and `--thread` arguments.

## Running Evaluations

To evaluate Unison, please switch to [unison-evaluations](https://github.com/NASA-NJU/Unison-for-ns-3/tree/unison-evaluations) branch, which is based on ns-3.36.1.
In this branch, you can find various topology models in the `scratch` folder.
There are a lot of parameters you can set for each topology.
We provided a utility script `exp.py` to compare these simulators and parameters.
We also provided `process.py` to convert these raw experiment data to CSV files suitable for plotting.
Please see the [README in that branch](https://github.com/NASA-NJU/Unison-for-ns-3/tree/unison-evaluations) for more details.

The evaluated artifact (based on ns-3.36.1) is persistently indexed by DOI [10.5281/zenodo.10077300](https://doi.org/10.5281/zenodo.10077300).

## Module Documentation

### 1. Overview

Unison for ns-3 is mainly implemented in the `mtp` module (located at `src/mtp/*`), which stands for multi-threaded parallelization.
This module contains three parts: A parallel simulator implementation `multithreaded-simulator-impl`, an interface to users `mtp-interface`, and `logical-process` to represent LPs in terms of parallel simulation.

All LPs and threads are stored in the `mtp-interface`.
It controls the simulation progress, schedules LPs to threads and manages the lifecycles of LPs and threads.
The interface also provides some methods and options for users to tweak the simulation.

Each LP's logic is implemented in `logical-process`. It contains most of the methods of the default sequential simulator plus some auxiliary methods for parallel simulation.

The simulator implementation `multithreaded-simulator-impl` is a derived class from the base simulator.
It converts calls to the base simulator into calls to logical processes based on the context of the current thread.
It also provides a partition method for automatic fine-grained topology partition.

For distributed simulation with MPI, we added `hybrid-simulator-impl` in the `mpi` module (located at `src/mpi/model/hybrid-simulator-impl*`).
This simulator uses both `mtp-interface` and `mpi-interface` to coordinate local LPs and global MPI communications.
We also modified the module to make it locally thread-safe.

### 2. Modifications to ns-3 Architecture

In addition to the `mtp` and `mpi` modules, we also modified the following part of the ns-3 architecture to make it thread-safe, also with some bug fixing for ns-3.
You can find the modifications to each unison-enabled ns-3 version via `git diff unison-* ns-*`.

Modifications to the build system to provide `--enable-mtp` option to enable/disable Unison:

```
ns3                                                |    2 +
CMakeLists.txt                                     |    1 +
build-support/custom-modules/ns3-configtable.cmake |    3 +
build-support/macros-and-definitions.cmake         |   10 +
```

Modifications to the `core` module to make reference counting thread-safe:

```
src/core/CMakeLists.txt                            |    1 +
src/core/model/atomic-counter.h                    |   50 +
src/core/model/hash.h                              |   16 +
src/core/model/object.cc                           |    2 +
src/core/model/simple-ref-count.h                  |   11 +-
```

Modifications to the `network` module to make packets thread-safe:

```
src/network/model/buffer.cc                        |   15 +-
src/network/model/buffer.h                         |    7 +
src/network/model/byte-tag-list.cc                 |   14 +-
src/network/model/node.cc                          |    7 +
src/network/model/node.h                           |    7 +
src/network/model/packet-metadata.cc               |   26 +-
src/network/model/packet-metadata.h                |   14 +-
src/network/model/packet-tag-list.h                |   11 +-
src/network/model/socket.cc                        |    6 +
```

Modifications to the `internet` module to make it thread-safe and add per-flow ECMP routing:

```
src/internet/model/global-route-manager-impl.cc    |    2 +
src/internet/model/ipv4-global-routing.cc          |   32 +-
src/internet/model/ipv4-global-routing.h           |    8 +-
src/internet/model/ipv4-packet-info-tag.cc         |    2 +
src/internet/model/ipv6-packet-info-tag.cc         |    2 +
src/internet/model/tcp-option.cc                   |    2 +-
```

Modifications to the `flow-monitor` module to make it thread-safe:

```
src/flow-monitor/model/flow-monitor.cc             |   48 +
src/flow-monitor/model/flow-monitor.h              |    4 +
src/flow-monitor/model/ipv4-flow-classifier.cc     |   12 +
src/flow-monitor/model/ipv4-flow-classifier.h      |    5 +
src/flow-monitor/model/ipv4-flow-probe.cc          |    2 +
src/flow-monitor/model/ipv6-flow-classifier.cc     |   12 +
src/flow-monitor/model/ipv6-flow-classifier.h      |    5 +
src/flow-monitor/model/ipv6-flow-probe.cc          |    2 +
```

Modifications to the `nix-vector-routing` module to make it thread-safe:

```
src/nix-vector-routing/model/nix-vector-routing.cc |   92 ++
src/nix-vector-routing/model/nix-vector-routing.h  |    8 +
```

Modifications to the `mpi` module to make it thread-safe with the hybrid simulator:

```
src/mpi/model/granted-time-window-mpi-interface.cc |   25 +
src/mpi/model/granted-time-window-mpi-interface.h  |    7 +
src/mpi/model/mpi-interface.cc                     |    3 +-
```

### 3. Logging

The reason behind Unison's fast speed is that it divides the network into multiple logical processes (LPs) with fine granularity and schedules them dynamically.
To get to know more details of such workflow, you can enable the following log component:

```c++
LogComponentEnable("LogicalProcess", LOG_LEVEL_INFO);
LogComponentEnable("MultithreadedSimulatorImpl", LOG_LEVEL_INFO);
```

### 4. Advanced Options

These options can be modified at the beginning of the `main` function using the native config syntax of ns-3.

You can also change the default maximum number of threads by setting

```c++
Config::SetDefault("ns3::MultithreadedSimulatorImpl::MaxThreads", UintegerValue(8));
Config::SetDefault("ns3::HybridSimulatorImpl::MaxThreads", UintegerValue(8));
```

The automatic partition will cut off stateless links whose delay is above the threshold.
The threshold is automatically calculated based on the delay of every link.
If you are not satisfied with the partition results, you can set a custom threshold by setting

```c++
Config::SetDefault("ns3::MultithreadedSimulatorImpl::MinLookahead", TimeValue(NanoSeconds(500));
Config::SetDefault("ns3::HybridSimulatorImpl::MinLookahead", TimeValue(NanoSeconds(500));
```

The scheduling method determines the priority (estimated completion time of the next round) of each logical process.
There are five available options:

- `ByExecutionTime`: LPs with a higher execution time of the last round will have higher priority.
- `ByPendingEventCount`: LPs with more pending events of this round will have higher priority.
- `ByEventCount`: LPs with more pending events of this round will have higher priority.
- `BySimulationTime`: LPs with larger current clock time will have higher priority.
- `None`: Do not schedule. The partition's priority is based on their ID.

Many experiments show that the first one usually leads to better performance.
However, you can still choose one according to your taste by setting

```c++
GlobalValue::Bind("PartitionSchedulingMethod", StringValue("ByExecutionTime"));
```

By default, the scheduling period is 2 when the number of partitions is less than 16, 3 when it is less than 256, 4 when it is less than 4096, etc.
Since more partitions lead to more scheduling costs.
You can also set how frequently scheduling occurs by setting

```c++
GlobalValue::Bind("PartitionSchedulingPeriod", UintegerValue(4));
```

## Links

If you find the code useful, please consider citing [our paper](https://dl.acm.org/doi/10.1145/3627703.3629574).

```bibtex
@inproceedings{10.1145/3627703.3629574,
author = {Bai, Songyuan and Zheng, Hao and Tian, Chen and Wang, Xiaoliang and Liu, Chang and Jin, Xin and Xiao, Fu and Xiang, Qiao and Dou, Wanchun and Chen, Guihai},
title = {Unison: A Parallel-Efficient and User-Transparent Network Simulation Kernel},
year = {2024},
isbn = {9798400704376},
publisher = {Association for Computing Machinery},
address = {New York, NY, USA},
url = {https://doi.org/10.1145/3627703.3629574},
doi = {10.1145/3627703.3629574},
abstract = {Discrete-event simulation (DES) is a prevalent tool for evaluating network designs. Although DES offers full fidelity and generality, its slow performance limits its application. To speed up DES, many network simulators employ parallel discrete-event simulation (PDES). However, adapting existing network simulation models to PDES requires complex reconfigurations and often yields limited performance improvement. In this paper, we address this gap by proposing a parallel-efficient and user-transparent network simulation kernel, Unison, that adopts fine-grained partition and load-adaptive scheduling optimized for network scenarios. We prototype Unison based on ns-3. Existing network simulation models of ns-3 can be seamlessly transitioned to Unison. Testbed experiments on commodity servers demonstrate that Unison can achieve a 40\texttimes{} speedup over DES using 24 CPU cores, and a 10\texttimes{} speedup compared with existing PDES algorithms under the same CPU cores.},
booktitle = {Proceedings of the Nineteenth European Conference on Computer Systems},
pages = {115–131},
numpages = {17},
keywords = {Data center networks, Network simulation, Parallel discrete-event simulation},
location = {<conf-loc>, <city>Athens</city>, <country>Greece</country>, </conf-loc>},
series = {EuroSys '24}
}
```

Below are some links that may also be helpful to you:

- [ns-3 Tutorial](https://www.nsnam.org/docs/tutorial/html/index.html)
- [ns-3 Model Library](https://www.nsnam.org/docs/models/html/index.html)
- [ns-3 Manual](https://www.nsnam.org/docs/manual/html/index.html)
