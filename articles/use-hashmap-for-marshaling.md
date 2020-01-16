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

The steps involved are as follows:

1. Run marshaling tests for OCaml Multicore code.
2. Run marshaling tests for Ocaml code.
3. Port use of addrmap.h in runtime/extern.c for OCaml.
4. Run marshaling tests for OCaml with use of addrmap.h.
5. Compare OCaml marshaling test results between linked data structure and hashmap.
6. Add branch to sandmark for regression testing.
