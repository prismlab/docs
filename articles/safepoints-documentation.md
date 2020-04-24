## Safepoints

### Quick links
1. Issue on Multicore OCaml - [link](https://github.com/ocaml-multicore/ocaml-multicore/issues/187)
2. ref_table grows without bound issue - [link](https://github.com/ocaml-multicore/ocaml-multicore/issues/187)
3. catch-break does not always catch break - [link](https://github.com/ocaml/ocaml/issues/3747)
4. Polling efficiently on stock hardware (Marc Feeley) - [link](https://www.iro.umontreal.ca/~feeley/papers/FeeleyFPCA93.pdf)
5. Cmm-to-Cmm pass for polling - [link](https://github.com/dc-mak/ocaml-1)
6. Polling insertion at Mach - [link](https://github.com/anmolsahoo25/ocaml/tree/safepoints-debug)

### Background
Safepoints are locations in the code, where a designated set of actions are executed. Typically, some examples of such actions would be calling the garbage collector or checking pending signals. The OCaml compiler implicitly treats allocations as safepoints. But this has some issues, wherein code that does not allocate can go for a very long time without checking signals. Moreover, in the case of multicore OCaml, it becomes even more important, as periodic checking of inter-process signals is necessary to prevent livelock. The document uses the words polling and safepoints interchangeably.

Thus this work aims to create a better system for handling signals in the OCaml compiler, herein referred to as safepoint insertion. The main moving pieces with safepoint insertion are as follows - 
1. **Polling primitive** - This refers to a construct which will be inserted in the compiler representation of the program and will emit the necessary assembly instructions to carry out the actual polling
2. **Safety analysis** - Not every point in the generated code is actually valid for a successful poll to be carried out. This part determines where it is actually safe to insert a polling operation
3. **Poll insertion** - Once the safe regions to insert polls have been identified, we need to actually insert the polls according to strategy, considering the overhead v/s polling latency tradeoffs

### Past solutions
The first attempt at introducing safepoints in the OCaml native backend was made by Dhruv Makhwana. It is a Cmm-to-Cmm translation, which recurses over the Cmm structure of the program and inserts the safepoints. The code can be found here - [link](https://github.com/ocaml/ocaml/compare/trunk...dc-mak:trunk)

The salient features of this proposal were - 
1. Polling primitive - A zero byte float allocation was taken to be the polling construct. This simulates all the code paths required for testing of pending signals. The code as in the original proposal - 

```ocaml
let add_poll e =
  let dbg, _args = Debuginfo.none, [Cconst_int 1] in
  (* Stolen from cmmgen.ml for 0-sized float-arrays *)
  let block_header tag sz = Nativeint.(add (shift_left (of_int sz) 10) (of_int tag)) in
  Csequence (Cop(Calloc, Cblockheader(block_header 0 0, dbg) :: [], dbg), e)
```

2. Polling insertion - The safepoint insertion was carried out according to the balanced polling algorithm by Feeley. A global counter is maintained which is updated after every Cmm instruction. Once the counter hits the constraints as defined by the algorithm, the poll is inserted

While this proposal managed to fix the open issues, there were some key concerns which required further analysis - 

1. Safety analysis - There are values of type `Addr` which are stored in the registers during intermediate computations for generating addresses. If these values are live, and a safepoint is triggered, it will lead to undefined behaviour.
2. The zero-sized allocation was still allocating a header object on the heap, thus creating some garbage objects.
3. The OCaml testsuite was not passing

### Current proposal
As discussed in the previous section, the safety analysis for poll point insertion requires knowing the liveness information. Thus when we have values of type `Addr` live, we do not insert poll points there. Thus, the salient features of the current proposal are - 
1. Polling construct - A new polling construct called `Ipoll` is added to the Mach language
2. Safety analysis - A new compiler pass called `polling` is added to `asmgen`. This pass is inserted after the liveness pass has been carried out. It recurses over the Mach representation of the program and performs the checks required for proper safepoints insertion
3. Polling insertion - A simple counter is maintained which is incremented as the instructions are encountered, along with the necessary checks for inserting safepoints and resetting the counter.

The first point is to look at the polling pass itself. It is a Mach-to-Mach translation and it operates on the Mach `fundecl` produced after register availability analysis (line 6).

```ocaml
(* asmcomp/asmgen.ml *)
let compile_fundecl ~ppf_dump fd_cmm =
  ...
  ++ Profile.record ~accumulate:true "available_regs" Available_regs.fundecl
  ++ Profile.record ~accumulate:true "liveness" liveness
  ++ Profile.record ~accumulate:true "polling" Polling.fundecl
  ++ pass_dump_if ppf_dump dump_live "After polling"
  ...
```

Once this is done, we add the `Polling.fundecl` pass itself,
```ocaml
(* asmcomp/polling.ml *)
let fundecl f =
    let new_body =
      if !Clflags.no_poll then
        f.fun_body
      else
        insert_poll f.fun_body
    in
      { f with fun_body = new_body }
```

The `insert_poll` function is necessary for keeping track of the various constraints as it recurses over the Mach representation. (cases elided for brevity)

```ocaml
(* asmcomp/polling.ml *)
let rec insert_poll_aux delta instr =
    (* address live at this point, skip insertion*)
    if (is_addr_live instr.live) then begin
        { instr with next = (insert_poll_aux delta instr.next) }
    end else begin
        match instr.desc with
        (* terminating condition *)
        | Iend -> instr

        (* reset counter *)
        | Iop (Ialloc _) ->
            { instr with next = (insert_poll_aux 0 instr.next) }
        ...

let insert_poll fun_body =
    insert_poll_aux (lmax-e) fun_body
```

Finally, the `Ipoll` primitive itself is emitted to assembly in the emit stage. We compare the `Domain_young_limit` to the value of the `Young_ptr` cached in `r15`. If it is below that value, then it means that either we have filled up the minor heap or a pending signal is present and has set the `Domain_young_limit` to the `Domain_young_start`. Thus, we execute a conditional jump and call the GC if the comparison succeeds.

```ocaml
  | Lop (Ipoll) ->
      I.cmp (domain_field Domainstate.Domain_young_limit) r15;
      let gc_call_label = new_label () in
      let label_after_gc = new_label () in
      let lbl_frame =
        record_frame_label ?label:None i.live (Dbg_other (Debuginfo.none))
      in
      I.jb (label gc_call_label);
      call_gc_sites :=
        { gc_lbl = gc_call_label;
          gc_return_lbl = label_after_gc;
          gc_frame = lbl_frame;
          gc_spacetime = None } :: !call_gc_sites;
      def_label label_after_gc;
      ()
  ```
  
  ### Status
  Herein, we discuss the current status of the proposal. 
  1. The patch has been created on trunk OCaml and the compiler successfully compiles
  2. These are the testsuite tests which fail - 
  ```
  List of failed tests:
    tests/lib-threads/'torture.ml' with 1.2 (native)
    tests/callback/'signals_alloc.ml' with 1.2 (native)
    tests/lib-bigarray/'bigarrays.ml' with 1 (native)
    tests/asmgen/'catch-rec-deadhandler.cmm' with 1.1.1 (check-program-output)
    tests/statmemprof/'minor_no_postpone.ml' with 1 (native)
    tests/c-api/'alloc_async.ml' with 1 (native)

List of unexpected errors:
    tests/lib-unix/common/'channel_of.ml' with 1.2 (native)
    tests/ast-invariants/'test.ml' with 1.1 (native)
    tests/lib-bigarray/'fftba.ml' with 1 (native)
    tests/lib-buffer/'test.ml' with 1 (native)
    tests/lib-threads/'fileio.ml' with 1.2 (native)
  ```
  3. The safety analysis is incomplete. That is, there are places in the Mach tree, where we do not know if it is safe to insert a poll or not. 
  
  Here are some key issues that need to be targetted going ahead,
  1. The safety analysis is incomplete. An `Istore` after an `Ialloc` is actually initializing the header of the newly created object on the heap. The fields still have not been initialized. Thus, inserting a safepoint between these two instructions causes this invalid object to be collected and causes the GC to segfault as it tries to follow invalid objects present in the objects fields. There could be more cases like this which need to be handled.
  2. Signal handling - As it stands, there are test cases, where signal handling is tested and also user provided signal handlers are executed when signals are raised. Due to safepoint insertion, signals are actually before the expected point, thus causing these tests to fail.
  3. Poll-free blocks - There needs to be an annotation to signal the compiler to not insert polls, to prevent existing code from breaking
  
  ### Future proposal
  To address some of the issues with the safety analysis, one proposal is to create a new field in the Mach operation called `head_of_cmm`. 
  
  ```ocaml
  type instruction =
  { desc: instruction_desc;
    next: instruction;
    arg: Reg.t array;
    res: Reg.t array;
    dbg: Debuginfo.t;
    mutable live: Reg.Set.t;
    mutable available_before: Reg_availability_set.t;
    mutable available_across: Reg_availability_set.t option;
    head_of_cmm: bool
  }
  ```
  
  Then at the time of instruction selection, where the Mach instruction is generated, we add if this instruction is at the head of a CMM operation. This is in a way, necessary to delimit the fact that we do not insert polls in-between CMM operations, emitted to multiple Mach ops. This information is then used in the polling insertion pass to rule out further places, where inserting polls can cause undefined behavior.