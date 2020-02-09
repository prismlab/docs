Abstract
========

The OCaml runtime uses a linked data structure implementation
(trail_block in trunk/runtime/extern.c) for marshaling.

```
/* Trail mechanism to undo forwarding pointers put inside objects */

struct trail_entry {
  value obj;    /* address of object + initial color in low 2 bits */
  value field0; /* initial contents of field 0 */
};

struct trail_block {
  struct trail_block * previous;
  struct trail_entry entries[ENTRIES_PER_TRAIL_BLOCK];
};
```

Source: https://github.com/ocaml/ocaml/blob/trunk/runtime/extern.c

On the other hand, the Multicore OCaml runtime uses a hashmap (addrmap
in byterun/caml/addrmap.h).

```
struct addrmap_entry { value key, value; };

struct addrmap {
  struct addrmap_entry* entries;
  uintnat size;
};
```
Source: https://github.com/ocaml-multicore/ocaml-multicore/blob/master/byterun/caml/addrmap.h

The goal is to port the use of the hashmap from Multicore OCaml to
OCaml runtime, and to measure the performance improvements. The
marshalling tests for both OCaml and Multicore OCaml are present in
testsuite/tests/lib-marshal directory.

Plan
====

The steps involved are as follows:

1. Run marshaling tests for OCaml Multicore code.
2. Run marshaling tests for Ocaml code.
3. Port use of addrmap.h in runtime/extern.c for OCaml.
4. Run marshaling tests for OCaml with use of addrmap.h.
5. Compare OCaml marshaling test results between linked data structure and hashmap.
6. Add branch to sandmark for regression testing.

Changes
=======

- PR: Use hashmap for marshaling
  (Stephen Dolan, KC Sivaramakrishnan)

  Copied runtime/addrmap.c and runtime/caml/addrmap.h.

  Add addrmap.h and addrmap.c to runtime/.depend by including them in
  runtime/Makefile.

  Defined Is_power_of_2() function in runtime/caml/misc.h.

  Updated runtime/extern.c to include addrmap.h and to use addrmap.c
  functions.

Build
=====

The build steps in OCaml trunk are as follows:

~~~~{.sh}
$ ./configure
$ make -j3 world.opt
~~~~

By default, the --enable-debugger option is enabled.

The following command is used to run all the tests:

~~~~{.sh}
$ USE_RUNTIME=d OCAMLRUNPARAM=v=0,V=1 make tests
~~~~

If you would like to run a test from a specific folder, you can use:

~~~~{.sh}
$ make ONE DIR=tests/lib-marshal
~~~~

Results
=======

After porting the changes from Multicore OCaml to Ocaml trunk, there
are no errors or failing tests:

~~~~{.sh}
Summary:
  2642 tests passed
  40 tests skipped
   0 tests failed
  100 tests not started (parent test skipped or failed)
   0 unexpected errors
2782 tests considered
~~~~

The compilation time for `make world` and `make world.opt` for both
trunk and multicore/use-addrmap-hash-table-for-marshaling branches are
provided below for reference:

| Branch         | trunk             | hash-table-for-marshaling |
|----------------|-------------------|---------------------------|
| make world     | real    2m54.526s | real    2m57.806s         |
|                | user    2m41.816s | user    2m45.053s         |
|                | sys     0m11.904s | sys     0m12.345s         |
|----------------|-------------------|---------------------------|
| make world.opt | real    5m26.222s | real    5m31.342s         |
|                | user     5m0.913s | user     5m5.496s         |
|                | sys     0m24.386s | sys      0m24.972s        |
|----------------|-------------------|---------------------------|

System: Parabola GNU/Linux-libre on Intel(R) Core(TM) i5-6200 CPU @ 2.30GHz
