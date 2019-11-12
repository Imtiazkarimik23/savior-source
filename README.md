# savior
SAVIOR Fuzzer

## How to build savior

### Build with Docker (tested on Ubuntu 16.04)
```
$ curl -fsSL https://get.docker.com/ | sudo sh

$ sudo usermod -aG docker [user_id]

$ docker run ubuntu:16.04

Unable to find image 'ubuntu:16.04' locally
16.04: Pulling from library/ubuntu
Digest: sha256:e348fbbea0e0a0e73ab0370de151e7800684445c509d46195aef73e090a49bd6
Status: Downloaded newer image for ubuntu:16.04

$ docker build -t savior .

$ docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
savior             latest            687322eff8f3        29 minutes ago        62.8GB
ubuntu              16.04            657d80a6401d         10 days ago          112MB

```
Once the build has been successful, lunch the Docker image 
to test out SAVIOR.


```
$ docker run -it savior:latest /bin/bash
```

### Build savior natively
TODO: add auto installation script

For now plese refer to the build script in Docker/build_savior.sh to setup the
whole environment.

## Savior components
* Fuzzer: a modified AFL based on version 2.50b
* KLEE: concolic execution engine, more details in [KLEE/features.md](KLEE/features.md) 
* DMA: static analyzer based on SVF
* Coordinator: python based framework, more details in [coordinator/README.md](coordinator/README.md)
* Clang: patched LLVM Clang front end to instrument meta data node for UBSAN 


## Fuzzing environment preparation 
It is a little convoluted to prepare the fuzzing environment and we prepared two examples
in the [tests](./tests) directory.
In general a user can follow these steps in order to fuzz a target program. We use the example
of building libjpeg to demonstrate the process. You can find the build script at 
[tests/build_djpeg.sh](tests/build_djpeg.sh).

1. Prepare the whole-program bitcode instrumented with UBSAN and with our metadata node.

```
cd jpeg-src

CC=wllvm LLVM_COMPILER=clang CFLAGS="-fsanitize=integer,bounds,shift -g" ./configure  --enable-shared=no --enable-static=yes
LLVM_COMPILER=clang make -j$(nproc)

extract-bc djpeg
```

2. Compile the aggregated bc with our modified AFL/afl-clang-fast.

```
savior/AFL/afl-clang-fast djpeg.bc -o savior-djpeg -lubsan -lm
```

at this step, the compiler generate 2 outputs: 
one executable with AFL instrumentation and savior instrumentations; and a bitcode file
with the same basic block IDs at each basic block. 

3. Use DMA to analyze the generated bitcode.
```
dma -fspta savior-djpeg.bc -savior-label-only -o djpeg.reach.bug -edge djpeg.edge
```

at this step, the analyzer generates 2 **important** outputs: 
1) the *.reach.bug file records the basic block ID and how many basic blocks 
(or sanitizer instrumentations, depends on if the flag 'savior-label-only' is set) 
can be reached from this BBL.

2) the *.edge file records each basic block ID and its the outgoing edge IDs

4. generate the final bitcode that should be run with the KLEE component.
```
opt -load savior/svf/InsertBugPotential/build/insertpass/libInsertBugPass.so -InsertBug -i djpeg.reach.bug savior-djpeg.bc -o savior-djpeg.dma.bc
```

The difference between this dma.bc and the input.bc is that the final bc has the bug 
reachability infomation from *.reach.bug file. Why don't we just generate the final bc in 
the previous step? Because DMA is built with llvm-4.0 while our KLEE is built with llvm-3.6.
A bitcode generated by a later llvm version is not backward compatible, so we have to use
an additional pass (compiled with llvm-3.6) to inject the the bug/code reachability infomation.

5. The last step, prepare the necessary inputs and edit the config file to choose the configuration
that works the best for your fuzzing campaign. You can find an example of the config file for
fuzzing libjpeg at [tests/config_samples/fuzz_jpeg.cfg](tests/config_samples/fuzz_jpeg.cfg).


## Citation
If your research find one or several components of savior useful, please cite the following paper:
```
@INPROCEEDINGS {savior,
author = {Y. Chen and P. Li and J. Xu and S. Guo and R. Zhou and Y. Zhang and T. Wei and L. Lu},
booktitle = {2020 IEEE Symposium on Security and Privacy (SP)},
title = {SAVIOR: Towards Bug-Driven Hybrid Testing},
year = {2020},
volume = {},
issn = {2375-1207},
pages = {2-2},
keywords = {fuzzing;bug-finding;hybrid-testing;sanitizers},
doi = {10.1109/SP.2020.00002},
url = {https://doi.ieeecomputersociety.org/10.1109/SP.2020.00002},
publisher = {IEEE Computer Society},
address = {Los Alamitos, CA, USA},
month = {may}
}

```

## Credits 

- [Yaohui Chen](https://github.com/evanmak)
- [Peng Li](https://github.com/lipeng28)
- [Jun Xu](https://github.com/junxzm1990)
- [Yueqi Chen](https://github.com/chenyueqi)
- [Rundong Zhou](https://github.com/rdzhou)
- [Shengjian Guo](https://github.com/DanielGuoVT)

## Q&A

1. KLEE concolic executor is not open sourced?

We are working on upstreaming our changes to the official KLEE repository, stay tuned.

2. Does savior support fuzzing with other sanitizers (e.g., ASAN, TSAN)?

Savior's bug-driven prioritization design is in general applicable to other sanitizers. All you need
to do is to instrument the meta data node at the sanitizer instrumentation sites. However,
for bug-guided verification to work, the concolic executor should be able to model the bug-triggering
conditions into SMT constraints, which is not very straightforward with sanitizers other than UBSAN, since
ASAN uses constructs that are not encodable into constraints such as the allocation bitmap, redzones etc.

3. Fuzzing c++ programs linked with GNU libstdc++?

Please stay tuned, our modified libstdc++.bca will be released soon.