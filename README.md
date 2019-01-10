# Davis In-Order (DINO) CPU models

![Cute Dino](dino-128.png)

This repository contains chisel implementations of the CPU models from Patterson and Hennessy's Computer Organization and Design primarily for use in UC Davis's Computer Architecture course (ECS 154B).

The repository was originally cloned from https://github.com/ucb-bar/chisel-template.git.

## How to run

First, you need to install Chisel.
For now, I'm using a docker image: `powerjg/chisel`.

### STEP 1: Run tests

Once Chisel is installed, or you're in the docker image, you an run the following to test the CPU:

```
sbt test
```

### STEP 2: Build the verilog for the CPU

To build the CPU:

```
sbt run
```

This creates the file `Top.v` *in the current directory*.

### STEP 3: Build the emulator

Now, you can copy this `Top.v` to emulator.
Then, you can build the emulator by running `make` in the `emulator/` directory.

```
make
```

### STEP 4: Run the emulator

Now, you can run the emulator.
The important options are:

- `+verbose`: makes the emulator verbose.
- `+max-cycles=`: Sets the max cycles to simulate.
- `+loadmem=`: The file to load into the memory for initialization. This is passed on to the dtm (see fesvr)

For `+max-cycles`, it takes some time to load the binary, so you may need to increase `max-cycles` to more than 1000 before you start to see debug output.
Note: The debug output comes from `printf` in the Chisel code.

Note: You may have to build riscv-fesvr and copy in `libfesvr.so`.


I think the only instructions I've implemented so far are:
`lw, sw, beq, add, sub, and, or`

As an example:

```
LD_LIBRARY_PATH=. obj_dir/emulator +verbose +max-cycles=50 +loadmem=test
```


## Getting baremetal programs working

Here's an example baremetal program that loads -1.
We have to have the `_start` symbol declared or the linker gets mad at us.

```
  .text
  .align 2       # Make sure we're aligned to 4 bytes
  .globl _start
_start:
    lw t0, 0x100(zero)   # t0 (x5) <= mem[0x100]

  .data
.byte 0xFF,0xFF,0xFF,0xFF
```

### STEP 1: Assemble

Now, to compile, first we have to assemble it.

```
riscv32-unknown-elf-as -march=rv32i src/main/risc-v/test.riscv -o test.o
```

### STEP 2: Link

Then, we have to link it.
When linking, we have to specify that the text should be located at `0x8000000` and that the data should be located at `0x8000000`.
The emulator/simulator and/or `SimDTM` automatically takes the ELF object and loads it at `0x8000000`.
However, that address appears at `0x0` in the scratchpad memory used for testing.

```
riscv32-unknown-elf-ld -Ttext 0x8000000 -Tdata 0x80000100 test.o -o test
```

### STEP 3: Check output

Finally, we can disassemble it to make sure it looks right.

```
riscv32-unknown-elf-objdump -Dx test
```

We get the following:

```
test:     file format elf32-littleriscv
test
architecture: riscv:rv32, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x08000000

Program Header:
    LOAD off    0x00001000 vaddr 0x08000000 paddr 0x08000000 align 2**12
         filesz 0x00000004 memsz 0x00000004 flags r-x
    LOAD off    0x00001100 vaddr 0x80000100 paddr 0x80000100 align 2**12
         filesz 0x00000004 memsz 0x00000004 flags rw-

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000004  08000000  08000000  00001000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  80000100  80000100  00001100  2**0
                  CONTENTS, ALLOC, LOAD, DATA
SYMBOL TABLE:
08000000 l    d  .text  00000000 .text
80000100 l    d  .data  00000000 .data
80000904 g       .data  00000000 __global_pointer$
08000000 g       .text  00000000 _start
80000104 g       .data  00000000 __bss_start
80000104 g       .data  00000000 _edata
80000104 g       .data  00000000 _end



Disassembly of section .text:

08000000 <_start>:
 8000000:       10002283                lw      t0,256(zero) # 100 <_start-0x7ffff00>

Disassembly of section .data:

80000100 <__bss_start-0x4>:
80000100:       ffff                    0xffff
```

### STEP 4: Run in emulator

We can simply run the binary that we generated by using the emulator.

```
LD_LIBRARY_PATH=. obj_dir/emulator +verbose +max-cycles=50 +loadmem=../test
```

We get the following output:

```
Instantiated DTM.
warning: tohost and fromhost symbols not in ELF; can't communicate with target
DASM(10002283)
Cycle=         0 pc=0x80000000, r1= 0, r2= 0, rw= 5, daddr=00000100, npc=0x80000004
                 r1=00000000, r2=00000000, imm=00000100, alu=00000100, data=ffffffff, write=ffffffff
DASM(00000000)
Cycle=         1 pc=0x80000004, r1= 0, r2= 0, rw= 0, daddr=00000000, npc=0x80000008
                 r1=00000000, r2=00000000, imm=00000000, alu=00000000, data=10002283, write=00000000
```

We can ignore the warning since we aren't trying to communicate with the target right now.
Also, after the 0th cycle, nothing is valid since there are no instructions after the load.
The `DASM()` statement can be fed into other RISC-V tool to get a trace of the instructions executed.
Remember, these verbose statements came from the `printf`s in `cpu.scala`.

## Running instruction tests

The tests for instructions are in `src/test/emulator` since they are using the emulator.

There are tiny RISC-V programs in `src/test/emulator/risc-v`.
These programs can be built with **the 32-bit GCC tools** by running `make` in that directory.

There are also scripts to be used with the `tester` program in `src/tests/emulator/tests`.
These scripts have a special format as specified in [the readme in that directory](src/test/emulator/tests/README.md).
These scripts specify the binary to run, the initial register file state, how many cycles to run, and the ending register file state.
Right now, the test only checks to make sure the register file is in the right state.

Note: If you want to set an initial memory state, you have to include the `.data` section in the binary.

Finally, there is the `tester` code which is a verilator wrapper.
Right now, you can build the `tester` by running `make` in `src/tests/emulator`.

To run a test you simply need to use the `tester` binary and specify one parameter: the test script.
Note: Right now, the test script uses a relative path to the binary.
This means that *you must run the `tester` from the base directory*.
You can also override all of the script details by specifying the same parameters as what the emulator in `emulator/` uses.

# Helpful hints

## Getting DASM to work

The `spike-dsm` program will convert the `DASM(<instruction>)` strings into RISC-V instructions.
You can use this by adding the `+verbose`option to the emulator (or the tester) and then storing the output into a file (e.g., `2> trace`).
Then, you can send that trace to `spike-dsm`.

```
$RISCV/bin/spike-dsm < trace
```

Or, you can use pipe to automatically convert the `DASM` statements as the emulator is running.

```
./emulator +verbose 2>&1 | $RISV/bin/spike-dsm
```

## sbt hints

If you want to pass an option to `sbt run`, you need to enclose `run` + your options in `"`.
For instance the below will show you the help for chisel and firrtl.

```
sbt "run --help"
```

Another hint: If you're having strange java problems and you've been mucking with the build.sbt or installing different version of java, you can try to blow away the target directory and start over.


# Testing for grading

You can run the test and have it output a junit compatible xml file by appending the following after `sbt test` or `sbt testOnly <test>`

```
-- -u <directory>
```

However, we now have Gradescope support built in, so there's no need to do the above.

## How to do it

If you run the test under the "Grader" config, you can run just the grading scripts.
This assumes that you are running inside the gradescope docker container.

```
sbt "Grader / test"
```

### Updating the docker image

```
docker build -f Dockerfile.gradescope -t jlpteaching/codcpu-grading .
```


# Getting started

- I suggest install intellj with the scala plug in. On Ubuntu, you can install this as a snap so you don't have to fight java versions.
  - To set this up you have to point it to a jvm. Mine was /usr/lib/jvm/<jvm version>

# Using this with singularity

[Singularity](https://www.sylabs.io/singularity/) is yet another container (LXC) wrapper.
It's somewhat like docker, but it's built for scientific computing instead of microservices.
The biggest benefit is that you can "lock" an image instead of depending on docker's layers which can constantly change.
This allows you to have much more reproducible containers.

However, this reproducibility comes at a cost of disk space.
Each image is stored in full in the current working directory.
This is compared to docker which only stores unique layers and stores them in /var instead of the local directory.

The other benefit of singularity is that it's "safer" than docker.
Instead of having to run everything with root privileges like in docker, with singularity you still run with your user permissions.
Also, singularity does a better job at automatically binding local directories than docker.
[By default](https://singularity.lbl.gov/docs-mount#system-defined-bind-points) it binds `$HOME`, the current working directory, and a few others (e.g., `/tmp`, etc.)

## Building a singularity image

To build the singularity image

```
sudo singularity build chisel.sif chisel.def
```

## To run chisel with singularity

The `chisel.def` file specifies `sbt` as the runscript (entrypoint in docker parlance).
Thus, you can simply `run` the image and you'll be dropped into the `sbt` shell.

Currently, the image is stored in the [singularity cloud](https://cloud.sylabs.io/library) under `jlowepower/default/dinocpu`.
This might change in the future.

To run this image directly from the cloud, you can use the following command.

```
singularity run library://jlowepower/default/dinocpu
```

This will drop you directly into `sbt` and you will be able to run the tests, simulator, compile, etc.

Note: This stores the image in `~/.singularity/cache/library/`.

If, instead, you use `singularity pull library://jlowepower/default/dinocpu`, then the image is downloaded to the current working directory.

**Important:** We should discourage students from using `singularity pull` in case we need to update the image!

# Example debugging

First, I tried to run the test:

```
testOnly dinocpu.ImmediateSimpleCPUTester
```

When this ran, I received the output:

```
[info] ImmediateSimpleCPUTester:
[info] Simple CPU
[info] - should run auipc0 *** FAILED ***
[info]   false was not true (ImmediateTest.scala:33)
[info] Simple CPU
[info] - should run auipc1 *** FAILED ***
[info]   false was not true (ImmediateTest.scala:33)
[info] Simple CPU
[info] - should run auipc2 *** FAILED ***
[info]   false was not true (ImmediateTest.scala:33)
[info] Simple CPU
[info] - should run auipc3 *** FAILED ***
[info]   false was not true (ImmediateTest.scala:33)
[info] Simple CPU
[info] - should run lui0
[info] Simple CPU
[info] - should run lui1 *** FAILED ***
[info]   false was not true (ImmediateTest.scala:33)
[info] Simple CPU
[info] - should run addi1 *** FAILED ***
[info]   false was not true (ImmediateTest.scala:33)
[info] Simple CPU
[info] - should run addi2 *** FAILED ***
[info]   false was not true (ImmediateTest.scala:33)
[info] ScalaTest
[info] Run completed in 5 seconds, 392 milliseconds.
[info] Total number of tests run: 8
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 1, failed 7, canceled 0, ignored 0, pending 0
[info] *** 7 TESTS FAILED ***
[error] Failed: Total 8, Failed 7, Errors 0, Passed 1
[error] Failed tests:
[error]         dinocpu.ImmediateSimpleCPUTester
[error] (Test / testOnly) sbt.TestsFailedException: Tests unsuccessful
[error] Total time: 7 s, completed Jan 1, 2019 11:38:09 PM
```

Now, I am going to dive into the `auipc` instruction.

So, I need to run the simulator.
The simulator takes two parameters, the RISC binary and the CPU type.
So, to run with the `auipc` workload on the single cycle CPU I would use the following:

```
runMain dinocpu.simulate src/test/resources/risc-v/auipc0 single-cycle --max-cycles 5
```

Then, I get the following output, which I can step through to find the problem.

```
[info] [0.000] Elaborating design...
[info] [0.017] Done elaborating.
Total FIRRTL Compile Time: 78.6 ms
Total FIRRTL Compile Time: 119.4 ms
file loaded in 0.153465121 seconds, 458 symbols, 393 statements
DASM(537)
CYCLE=1
pc: 4
control: Bundle(opcode -> 55, branch -> 0, memread -> 0, memtoreg -> 0, memop -> 0, memwrite -> 0, regwrite -> 1, alusrc2 -> 1, alusrc1 -> 1, jump -> 0)
registers: Bundle(readreg1 -> 0, readreg2 -> 0, writereg -> 10, writedata -> 0, wen -> 1, readdata1 -> 0, readdata2 -> 0)
aluControl: Bundle(memop -> 0, funct7 -> 0, funct3 -> 0, operation -> 2)
alu: Bundle(operation -> 2, inputx -> 0, inputy -> 0, result -> 0)
immGen: Bundle(instruction -> 1335, sextImm -> 0)
branchCtrl: Bundle(branch -> 0, funct3 -> 0, inputx -> 0, inputy -> 0, taken -> 0)
pcPlusFour: Bundle(inputx -> 0, inputy -> 4, result -> 4)
branchAdd: Bundle(inputx -> 0, inputy -> 0, result -> 0)
```

Also, I could have just run one of the tests that were failing by using `-z` when running `testOnly` like the following:

```
testOnly dinocpu.ImmediateSimpleCPUTester -- -z auipc0
```

# How to use specific versions of chisel, firrtl, etc

Clone the repos:

```
git clone https://github.com/freechipsproject/firrtl.git
git clone https://github.com/freechipsproject/firrtl-interpreter.git
git clone https://github.com/freechipsproject/chisel3.git
git clone https://github.com/freechipsproject/chisel-testers.git
git clone https://github.com/freechipsproject/treadle.git
```

Compile each by running `sbt compile` in each directory and then publish it locally.

```
cd firrtl && \
sbt compile && sbt publishLocal && \
cd ../firrtl-interpreter && \
sbt compile && sbt publishLocal && \
cd ../chisel3 && \
sbt compile && sbt publishLocal && \
cd ../chisel-testers && \
sbt compile && sbt publishLocal && \
cd ../treadle && \
sbt compile && sbt publishLocal && \
cd ..
```

By default, this installs all of these to `~/.ivy2/local`.
You can change this path by specifying `-ivy` on the sbt command line.

```
`sbt -ivy /opt/ivy2`
```

However, you only want to do that while building installing.
Once installed, now you have an ivy repository at /opt/ivy2.
We want to use that as one of the resolvers in the `build.sbt` file.
It's important not to use `-ivy /opt/ivy2` in the singularity file as it writes that location when in use.