Setting up a development OCaml compiler with OPAM
---

This guide is meant to get you a setup with the multicore compiler and its packages.

#### 1. Install OPAM
OPAM is the definitive package manager for OCaml which manages both the various compiler versions and packages. Let's start by installing that,

    apt install make m4 build-essential patch unzip bubblewrap
    sh <(curl -sL https://raw.githubusercontent.com/ocaml/opam/master/shell/install.sh)

Once this is done, enter the following command and answer `N` to every question. This initializes the files OPAM needs and also installs a default OCaml compiler for us.

    opam init

After this, enter the following command, to check if everything went well,

    opam sw
   
The expected output is (the version might be different),

    #  switch  compiler                    description
    -> 4.09.0  ocaml-base-compiler.4.09.0  4.09.0

Let's install dune and jbuilder, which are two tools that OCaml packages require to be built and installed.

    opam install dune.1.11.4
    opam install jbuilder

The following command,

    opam list

Should give the following output,

    # Packages matching: installed
    # Name              # Installed # Synopsis
    base-bigarray       base
    base-threads        base
    base-unix           base
    dune                1.11.4      Fast, portable and opinionated build system
    jbuilder            transition  This is a transition package, jbuilder is now named dune. Use the dune
    ocaml               4.09.0      The OCaml compiler (virtual package)
    ocaml-base-compiler 4.09.0      Official release 4.09.0
    ocaml-config        1           OCaml Switch Configuration

#### 2. Create a multicore switch and install OCaml multicore from Git repo
Whether you are developing OCaml multicore applications or hacking on the compiler, it is always nifty to have an OPAM switch with the multicore compiler. In this case, we will build it from a git repo, so you can quickly make local changes and test it out.

Start by cloning the multicore repo into a suitable location,

    git clone https://github.com/ocaml-multicore/ocaml-multicore
    cd ocaml-multicore

Now we can create a new switch and install the compiler to it,

    opam sw create --empty 4.06.1+multicore
    opam pin add --inplace-build ocaml-variants.4.06.1+multicore file://$(pwd)

This will create a new switch with the multicore compiler. Verify it thus,

    opam sw

Which will output (among other things),

    #   switch            compiler                    description
    ->  4.06.1+multicore                              4.06.1+multicore

Let's check if we have the compiler up,

    opam sw 4.06.1+multicore
    eval $(opam env)
    ocaml

The output should be,
   
    OCaml version 4.06.1+multicore-dev0
    #
    
#### 3. The Dune trick
Dune is a build tool used for OCaml packages. To build and install multicore packages, we need to have Dune installed on our switch. But as of now, Dune does not build natively, so we have to apply a small trick.

Remember installing Dune with the 4.09.0 compiler? We are going to re-use it now. Enter the following commands,

    cp -r ~/.opam/4.09.0/lib/dune ~/.opam/4.06.1+multicore/lib
    cp -r ~/.opam/4.09.0/bin/dune ~/.opam/4.06.1+multicore/bin

Now we need to edit an OPAM file to convince it that Dune is installed. 

    <text-editor> ~/.opam/4.06.1+multicore/.opam-switch/switch-state

Change the following lines, 

    roots: ["ocaml-base-compiler.4.09.0"]
    installed: [
        "base-bigarray.base"
        "base-threads.base"
        "base-unix.base"
        "ocaml.4.09.0"
        "ocaml-base-compiler.4.09.0"
        "ocaml-config.1"
    ]

to,

    roots: ["dune.1.11.4" "ocaml-base-compiler.4.09.0"]
    installed: [
        "base-bigarray.base"
        "base-threads.base"
        "base-unix.base"
        "dune.1.11.4"
        "ocaml.4.09.0"
        "ocaml-base-compiler.4.09.0"
        "ocaml-config.1"
    ]

Now, the following command,

    opam list

Should report `dune` as one of the installed packages. Go ahead and install jbuilder,

    opam install jbuilder

And check if `opam list` shows it up or not (among other things),

    # Packages matching: installed
    # Name         # Installed      # Synopsis
    dune           1.11.4           Fast, portable and opinionated build system
    jbuilder       transition       This is a transition package, jbuilder is now named dune. Use the dune

#### 4. Install essential multicore libraries
Now, we can install multicore libraries as easy as normal libraries. Let us first add the multicore repository package. First ensure you are on the multicore switch,

    opam sw 4.06.1+multicore

Now add the OPAM multicore repository,
 
    opam repository add multicore https://github.com/ocamllabs/multicore-opam.git

Now for the smoke test, lets install `lockfree`, a multicore library. 

    opam install lockfree

Check with `opam list`, whether you could install it or not,

    # Packages matching: installed
    # Name         # Installed      # Synopsis
    lockfree       0.1.3            Lock-free data structures for multicore OCaml
