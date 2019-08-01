## TPTPU-Sim: A Toy-Purpose TPU Simulator

TPU (tensor-processing unit) is Google's deep learning architecture for inference (all versions) and training (all versions except 1) focusing mainly on its systolic array-based computation method and throughput.\
Here's a link to Google's first paper on TPU, from arXiv and later published in 2017 ISCA.\
https://arxiv.org/ftp/arxiv/papers/1704/1704.04760.pdf

### Purpose:

TPTPU-Sim is a cycle-level simulator built to simulate with different bandwidth/frequency configurations than the original TPU to see what changes, and gain more insight into how architecture for deep learning should focus on.


### How to run TPTPU:
1. clone both TPTPU(https://github.com/gobblygobble/tptpu-sim) and Ramulator(https://github.com/CMU-SAFARI/ramulator) into the same directory
2. build ramulator with `make -j` in ramulator directory
3. build tptpu-sim with `make sim` in tptpu-sim directory
4. run simulator with proper options\
(-d D -c C -r R -x X -y Y -z Z -l L for X-by-Y matrix times Y-by-Z matrix multiplication with DRAM type D, with C channels and R ranks, and dimension layout L - nchw or nhwc)\
Different types of DRAMs supported can be easily spotted in the function `double DRAM::GetFrequencyByName(std::string name)` of `tptpu-sim/src/dram.cpp`.

For example:
```
cd ~/path/to/some/directory/
git clone https://github.com/gobblygobble/tptpu-sim
git clone https://github.com/CMU-SAFARI/ramulator
cd ramulator/
make -j
cd ..
cd tptpu-sim/
make testtptpu
./testtptpu -d DDR3_1600K -c 1 -r 1 -x 640 -y 640 -z 1080 -l nchw
```

### Architecture:

1. ***Overall:***

**DRAM** is connected to **Weight Fetcher** via **Interconnect**, **CPU** (interface) is connected to **Unified Buffer** via **Interconnect**, and **Weight Fetcher** and **Unified Buffer** are connected to **Matrix Multiply Unit** via **Interconnects**, providing it with weights and activations for neural network *inference*. It consits of the 5 **bold face** units along with a **Controller**.

2. ***Interconnect:***

Interconnect module consists of a sender-side sender queue and three receiver-side queues, served queue, waiting queue, and request queue, all of which are shared with their corresponding owners (receiver and sender). Any requests coming in are pushed into request queue, which are moved to waiting queue in *Interconnect::Cycle()*. The sender starts to prepare (send request to another unit or bring in data) the requested data when it *notices that the receiver's waiting queue is not empty*. When the *data is ready*, the request information is pushed into sender queue, which will be sent over the interconnect. When *transfer is complete*, the request is popped from both waiting queue and sender queue, and pushed into served queue, for use by the receiver.\
Note that interconnect bandwidth can be adjusted, so that not all interconnects are forced to have the same bandwidth. When the interconnect is transferring information, it is considered 'busy' and when it is not, it is considered 'idle'.

3. ***DRAM:***

Unlike the CPU module which does basically nothing, DRAM is a quite busy module in that it is responsible for calculating how many cycles it will take to fetch the data when it receives a request. To do so, **Ramulator: A DRAM Simulator** (https://github.com/CMU-SAFARI/ramulator) is used. In its construction, DRAM requires what type of DRAM it is. It then runs simulation with the given DRAM information (configuration file of the DRAM type is generated at the construction of DRAM module) and DRAM trace generated by DRAM module. The result is parsed by DRAM module to check for how many DRAM cycles is required. With the acquired DRAM cycle and DRAM frequency of the type of DRAM, the total time required for DRAM is calculated, which the total TPU cycles needed can be derived from. The DRAM module 'stalls' for that number of TPU cycles, and when that number of stall cycles have passed, it pushes the request into its sender queue so the interconnect can start delivering the data to its receiver, weight fetcher.\
Although TPU version 1 was said to use DDR3 DRAM, TPTPU-Sim also supports using DDR4 DRAM.\
If for some reason the config file is deleted, just copy *DDR3-config.cfg* and *DDR4-config.cfg* from **ramulator/configs** as "dram-config.cfg".
```
cd ~/path/to/some/directory/
cd tptpu-sim/
cp ../ramulator/configs/DDR3-config.cfg ./dram-config.cfg
```
or
```
cd ~/path/to/some/directory/
cd tptpu-sim/
cp ../ramulator/configs/DDR4-config.cfg ./dram-config.cfg
```
