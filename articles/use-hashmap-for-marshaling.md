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

In OCaml, the visited set is maintained by mutating the header and the
first fields of the objects being marshaled in place and then rolling
back the changes after marshaling. This is incompatible with
concurrent mutators and GC threads. Hence, Multicore ocaml uses an
external data structure i.e, the hashmap (addrmap in
byterun/caml/addrmap.h), and does not modify the objects which are
being marshaled.

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

Test system: Parabola GNU/Linux-libre on Intel(R) Core(TM) i5-6200 CPU @ 2.30GHz

The bench2.ocamllabs.io:8083 tests do not use a lot of marshaling,
but, here is the value of `maxrss_kB` from the test suite run for both
trunk and multicore/use-addrmap-hash-table-for-marshaling branch:

| Metric | trunk     | hash-table-for-marshaling |
|--------|-----------|---------------------------|
| count  | 192       | 192                       |
| mean   | 24702.625 | 24701.5                   |
| std    | 72617.01  | 72612.89                  |
| min    | 2476      | 2476                      |
| 25%    | 4524      | 4524                      |
| 50%    | 5212      | 5212                      |
| 75%    | 9531      | 9531                      |
| max    | 730868    | 730868                    |
|--------|-----------|---------------------------|

Source: [Comparison of multicore/use-addrmap-hash-table-for-marshaling with vanilla OCaml trunk](http://bench2.ocamllabs.io:8083/comparison/?exe=6%2BL%2Btrunk%2C26%2BL%2Bmulticore%2Fuse-addrmap-hash-table-for-marshaling&ben=1%2C2%2C161%2C162%2C3%2C4%2C5%2C6%2C188%2C189%2C163%2C7%2C8%2C9%2C10%2C11%2C12%2C13%2C14%2C15%2C16%2C17%2C18%2C164%2C165%2C166%2C167%2C168%2C169%2C170%2C171%2C172%2C19%2C20%2C21%2C22%2C23%2C24%2C25%2C26%2C27%2C28%2C173%2C29%2C190%2C191%2C30%2C192%2C31%2C193%2C32%2C194%2C33%2C195%2C34%2C196%2C35%2C197%2C36%2C198%2C37%2C199%2C38%2C200%2C39%2C201%2C40%2C202%2C41%2C203%2C174%2C42%2C43%2C175%2C44%2C45%2C46%2C47%2C48%2C49%2C50%2C51%2C52%2C53%2C54%2C55%2C56%2C57%2C58%2C59%2C60%2C61%2C62%2C63%2C64%2C65%2C66%2C67%2C68%2C69%2C70%2C71%2C72%2C73%2C74%2C176%2C75%2C76%2C77%2C78%2C79%2C80%2C81%2C82%2C83%2C84%2C85%2C86%2C177%2C178%2C87%2C88%2C89%2C90%2C91%2C92%2C93%2C94%2C95%2C96%2C97%2C98%2C99%2C100%2C179%2C180%2C181%2C182%2C101%2C102%2C103%2C104%2C183%2C105%2C106%2C107%2C108%2C109%2C110%2C111%2C112%2C113%2C114%2C115%2C116%2C117%2C118%2C119%2C120%2C121%2C122%2C123%2C124%2C125%2C126%2C127%2C128%2C129%2C130%2C131%2C132%2C204%2C133%2C134%2C205%2C135%2C136%2C137%2C138%2C206%2C139%2C140%2C141%2C184%2C142%2C143%2C144%2C145%2C146%2C147%2C185%2C186%2C187%2C148%2C149%2C150%2C151%2C152%2C153%2C154%2C155%2C156%2C157%2C158%2C159%2C160%2C207%2C208%2C209%2C210&env=3&hor=true&bas=none&chart=normal+bars)

You can use
[sandmark-analyze](https://github.com/ocaml-bench/sandmark-analyze) to
analyze the .bench files from bench2.ocamllabs.io:8083.

Example commit values for using sandmark-compareAB.ipynb:

```
commit_a = {
    'host': 'bench2.ocamllabs.io:8083',
    'environment': 'bench2.ocamllabs.io',
    'repo_branch_name': 'ocaml_trunk__trunk',
    'commitid': '87296ee8e0ffa5e29d21fc54b6a8bce3db6e1889',
    'variant': 'vanilla',
    'timestamp': '20200206_111510',
    'ocaml_version': '4.11.0',
    }

commit_b = {
    'host': 'bench2.ocamllabs.io:8083',
    'environment': 'bench2.ocamllabs.io',
    'repo_branch_name': 'hash_table_marshaling__multicore/use-addrmap-hash-table-for-marshaling',
    'commitid': '777a4915bbac6d3f6637b0370a51a99690f76537',
    'variant': 'vanilla',
    'timestamp': '20200206_103615',
    'ocaml_version': '4.11.0',
    }
```

Note: The sandmark-analyze.ipynb requires the input JSON file to be
comma separated and enclosed within [].
