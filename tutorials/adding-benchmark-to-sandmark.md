Adding a benchmark to Sandmark
---

## Quickstart
1. Add your dependencies to the `packages` folder
2. Add your benchmark code in `benchmark` folder and create a `dune` file with the alias `dune_bench`
3. Add an entry in the `run_config.json`

## Introduction

Sandmark is a repository that consists of various programs for benchmarking OCaml code. In this short tutorial, we will learn how to add a new benchmark to Sandmark.

### Concepts
Before we start, we need to make sure we are clear on a few concepts - 
1. `OPAM` - OPAM is the package manager for OCaml, which takes care of the compilers and installed packages.
2. `OPAMROOT` - `OPAMROOT` is an environment variable, which points to a directory where all the code managed by OPAM is stored. It holds the source code for the compilers and packages, the binaries and other metadata.
3. `Packages` - By packages in this context, we mean OCaml libraries. These libraries can export modules and binaries. For example, the `domainslib` package exports the `chan` module and the `frama-c` package exports the `frama-c` binary, among others. 
4. `Dune` - Dune is a build system for OCaml. It takes care of handling dependencies, versioning and invoking the OCaml compiler to build the code.

### Structure of Sandmark
Before we dive into the tutorial, it would be instructive to understand the structure of Sandmark itself.

```
.
├── benchmarks
|   ...
│   ├── multicore-numerical
|   ...
├── dependencies
│   ├── packages
│   ├── repo
│   └── template
├── dune
├── dune-project
├── Makefile
├── multicore_parallel_run_config.json
├── ocaml-versions
|   ├── ...
├── orun
│   ├── ...
├── README.md
├── run_config.json
└── rungen
    ├── ...
```

Let's go over the folders one-by-one,
 
1. `benchmarks` - This directory stores the code that is to be benchmarked, i.e. the code that should be run and then performance characteristics collected. Benchmarks are divided into folders, with each folder containing some source code and a `dune` file. The `multicore-numerical` has some `.ml` files and a `dune` file. We will look into how these files should be defined later on.

```
.
├── dune
├── LICENSE
├── mandelbrot_multicore.ml
├── mandelbrot_vanilla.ml
└── spectralnorm2_multicore.ml
```

2. `dependencies` - Remember back to the concept of `OPAMROOT`, which allows us to specify a folder for OPAM to store all the information about the environment. In the case of Sandmark, a new `OPAMROOT` is created in the Sandmark folder, so that it does not interfere with the system dependencies. The definitions for the packages that OPAM should refer to are put in this folder.

3. `dune` and `dune-project` - These top-level files add the information required by dune to build the benchmarks and run them. 

4. `Makefile` - A Makefile used for setting up the environment, launching the jobs and collecting results.

5. `multicore_parallel_run_config.json` and `run_config.json` - These files are used to denote the commands for running the benchmarks along with the parameters to be passed.

6. `orun` and `rungen` - These are OCaml applications defined for running the benchmarks, collecting metrics and so on.

## Adding a benchmark
This process can essentially be broken down into the following steps - 
1. Add your dependencies to the packages folder
2. Add the code to be benchmarked
3. Add the commands to run your application
4. Build and run your application

### Setup
Do the following steps to get a local copy of Sandmark that you can start working on - 

    git clone https://github.com/ocaml-bench/sandmark
    cd sandmark

All code from now on assumes you are in the root directory of Sandmark.

### Add your dependencies to the packages folder
In this step, we will add the `stringext` dependency to the Sandmark repo, which is a library with string functions. In case your package depends on other packages, you will have to repeat the process for all of them. Start by removing the package if it exists,

    rm -r ./dependencies/packages/stringext

Every dependency in Sandmark should live in its own folder. Moreover, the folders are also tagged by the version of the package. In this tutorial, we will install `stringext.1.6.0`. 

    mkdir -p ./dependencies/packages/stringext/stringext.1.6.0

Now, we need an `opam` file to add to these folders, so that OPAM knows how to build and install this library. Most OCaml packages are distributed with the a corresponding OCaml file. For our case you can find the corresponding files [here](https://github.com/rgrinberg/stringext/blob/master/stringext.opam). The file is also below, go ahead and open the following files - 

    <text-editor> ./dependencies/packages/stringext/stringext.1.6.0/opam

and paste the following, 

```
opam-version: "2.0"
maintainer: "rudi.grinberg@gmail.com"
authors: "Rudi Grinberg"
license: "MIT"
homepage: "https://github.com/rgrinberg/stringext"
bug-reports: "https://github.com/rgrinberg/stringext/issues"
depends: [
  "ocaml" {>= "4.02.3"}
  "dune" {build & >= "1.0"}
  "ounit" {with-test}
  "qtest" {with-test & >= "2.2"}
  "base-bytes"
]
build: [
  ["dune" "subst"] {pinned}
  ["dune" "build" "-p" name "-j" jobs]
  ["dune" "runtest" "-p" name "-j" jobs] {with-test}
]
dev-repo: "git+https://github.com/rgrinberg/stringext.git"
synopsis: "Extra string functions for OCaml"
description: """
Extra string functions for OCaml. Mainly splitting. All functions are in the
Stringext module.
"""

url {
  src: "https://github.com/rgrinberg/stringext/archive/1.6.0.tar.gz"
  checksum: "md5=3e3d68f9593cfdba26fc70e8f22ed31e"
}
```

Note that the `url` clause at the end is extra, as it tells OPAM where to download a release version of the source code. This can typically be found in the Releases section of the Github repository. The checksum is computer by downloading the file and running `md5sum` on it.

For the final step, you need to make the following change in the Makefile,

    <text-editor> ./Makefile

Change this,

    PACKAGES = ... sexplib0

To,

    PACKAGES = ... sexplib0 stringext

## Adding the code to be benchmarked
With this, the dependency has been added. Now we need to add the code that we actually want to benchmark. Go ahead and create the following folder,

    mkdir ./benchmarks/stringext

We will create some code to be benchmarked, 

    <text-editor> ./benchmarks/stringext/stringext_ex1.ml

Paste the following code in it - 

```
let _ = Stringext.split "test:one:two" ~on:':'
```
    
This folder has a requirement to be understood by the build system, it should have a `dune` file. Let's create one and fill it with the following content, 

    <text-editor> ./benchmarks/stringext/dune

```
(executable
 (name stringext_ex1)
 (modules stringext_ex1)
 (libraries stringext)

(alias (name mybench) (deps stringext1_ex1.exe))
```

Without going into too much detail into the `dune` syntax, what we can take away here is that all the executables that should be produced by your benchmark should be listed with this syntax. Finally, you need to define a `alias`, which is used to build all the executable files. The default is `dune_bench`. For multicore, this is `multibench_parallel`. Here we define our own alias and we will see how to build and run that.

### Add the commands to run your application
Now, we finally need to add a config file, which will run the application that we have added. Start by creating this file and putting the following content, 

    <text-editor> myrun_config.json

```
{
  "wrappers": [
    {
      "name": "orun",
      "command": "orun -o %{output} -- %{command}"
    },
    {
      "name": "perfstat",
      "command": "perf stat -o %{output} -- %{command}"
    }
  ],
  "benchmarks": [
    {
      "executable": "benchmarks/stringext/stringext_ex1.exe",
      "name": "test_stringext_ex1",
      "runs": [
        {
          "params": ""
        }
      ]
    }
   ]
}
```
As you can see, this is a JSON file, which has two wrappers to run tests, namely `orun` and `perf`. Moreover, we define all the benchmarks that we want to run, in the `benchmarks` section. The `runs` list contains all different variations of the runs that you want. You can pass command line args in the `params` string.

Typically, we always add these runs to either `run_config.json` or `multicore_parallel_run_config.json`. In this case, for an illustrative example, we have added it to a new file.

### Build and run your application

Finally, we can now build the application with the following command - 

    BUILD_BENCH_TARGET=mybench RUN_CONFIG_JSON=myrun_config.json RUN_BENCH_TARGET=run_orun make ocaml-versions/4.08.0.bench

To break down the args,
1. `BUILD_BENCH_TARGET` - The `alias` in the dune file that we want to build.
2. `RUN_CONFIG_JSON` - Relative path to the config file we want to use for running jobs
3. `RUN_BENCH_TARGET` - The run wrapper we want to use 
4. `ocaml-versions/4.08.0.bench` - The version of the compiler we want to run. The available versions can be checked in the folder `ocaml-versions`

Once the run completes, you can check the results in `_results` folder.
