# Bits API

Use `to_unsigned_int` and `to_signed_int` to convert a `Bits.t` instance to an OCaml integer.

DO NOT USE `to_int` or `to_int_trunc`. It is BAD PRACTICE.

Use the `(<--.)` operator to assign an integer to a `Bits.t ref` instance. This operator
is available if you include `open Bits` in the top of the module.

## Printing Bits values

Use `Bits.Hex.t` to sexp/print a `Bits.t` as a hex value. Similarly:
- `Bits.Unsigned_int.t` — prints as unsigned decimal
- `Bits.Signed_int.t` — prints as signed decimal
- `Bits.Binary.t` — prints as binary string

These are useful in `[%message]` / `[%sexp]` expressions:

```ocaml
print_s [%message (my_bits : Bits.Hex.t)];
(* prints something like (my_bits 0xDEAD) *)
```

# Before\_and\_after\_edge

`Before_and_after_edge.t` is a record used throughout the simulation frameworks:

```ocaml
type 'a t =
  { before_edge : 'a
  ; after_edge : 'a
  }
```

This captures output values at two points in the clock cycle:

- `before_edge` — sampled **before** the active clock edge. This reflects combinational /
  transparent outputs. If a register's input changes this cycle, `before_edge` shows the
  **old** registered value (or the combinational feed-through if the output is not
  registered). An example where is this useful is when checking
  [before_edge.valid &: before_edge.tready] to check that a handshake has happened.
- `after_edge` — sampled **after** the active clock edge. This reflects the **new**
  registered values after the flip-flops have latched.

# `Hardcaml_lws`

`Hardcaml_lws` (also known as Lws) is a library that allows us to write simulations with
concurrent (but not parallel) tasks that interact with an underlying simulation object.

New tests should opt for Lws by default with the cyclesim backend. DO NOT use the event
sim backend unless the existing code uses it, or the user explicitly asks for it.


## Execution model

Lws uses OCaml 5 effects for cooperative multitasking. Each task runs until it calls
`Lws.step h`, at which point it yields. The cyclesim backend ordering
per simulation step is:

1. All tasks run until they yield at their next `Lws.step` call.
2. `Cyclesim.cycle` advances the simulation by one clock cycle.

This means: you set inputs, call `Lws.step h`, and when your task resumes, the outputs
reflect the result of that clock cycle.

## Lws_context.t — interacting with ports

The simulation context gives you access to input and output ports:

```ocaml
type ('i, 'o) Lws_context.t =
  { inputs : 'i                                      (* Bits.t ref I.t *)
  ; outputs : 'o Before_and_after_edge.t              (* Bits.t ref O.t Before_and_after_edge.t *)
  ; ...
  }
```

- **Inputs** are `Bits.t ref` — assign with `:=` or `<--.` (from `open Bits`).
- **Outputs** are `Bits.t ref` — read with `!`.
- Use `ctx.outputs.after_edge` for registered outputs.
- Use `ctx.outputs.before_edge` for combinational / transparent outputs.

A convenient type alias: `type sim_context = Lws_context.M(Dut.I)(Dut.O).t`

## Core API

```ocaml
(* Step the simulation by one clock cycle (or n cycles) *)
Lws.step h          (* step 1 cycle *)
Lws.step ~n:5 h     (* step 5 cycles *)

(* Spawn a concurrent task. Returns a Task.t handle. *)
let task = Lws.spawn h (fun h -> ...) in

(* Block the current task until another task finishes *)
let result = Lws.wait h task in

(* Step until a condition is met *)
let value = Lws.step_until_some h (fun () -> if cond then Some x else None) in

(* Kill a spawned task *)
Lws.Task.kill task;
```

## Spawning and concurrency

- `Lws.spawn h f` spawns a concurrent task. The child inherits the parent's trigger
  (clock domain) unless overridden with `~trigger`.
- Tasks spawned during a step are enqueued and **run in that same step** — they do not
  wait for the next clock cycle.
- For multi-clock designs, use `Lws_context.spawn_in_clock_domain` to spawn a task in a
  specific clock domain.
- `Lws.spawn_unit` is like `spawn` but discards the `Task.t` handle.

## Lws_harness — the preferred way to write Lws tests

`Hardcaml_test_harness.Lws_harness` wraps the Lws setup boilerplate and handles waveform
configuration, timeouts, random initial state, and wave file output. It is the preferred
way to write new Lws tests.

```ocaml
open Core
open Hardcaml
open Hardcaml_lws

module Dut = My_module
module Harness = Hardcaml_test_harness.Lws_harness.Make(Dut.I)(Dut.O)

let%expect_test "my test" =
  Harness.run ~create:Dut.create (fun (h @ local) ~inputs ~outputs ->
    let open Bits in
    inputs.data <--. 42;
    Lws.step h;
    printf "result = %d\n" (to_unsigned_int !(outputs.after_edge.result)));
  [%expect {| result = ... |}]
;;
```

Key features of the harness:
- `~waves_config` — control waveform output (e.g., `Waves_config.to_test_directory ()` to
  save to disk, `Waves_config.no_waves` to disable).
- `~print_waves_after_test` — pass a function like `Waveform.print ~wave_width:2` to
  print waveforms inline in expect tests.
- `~random_initial_state` — randomize register and/or memory state before the test (`` `All ``, `` `Regs ``, `` `Mems ``, `` `None ``).
- `~timeout` — stop the simulation after a given number of cycles.
- `~trace` — control which signals are traced (`` `All_named ``, `` `Everything ``, `` `Ports_only ``).
- `Harness.run_advanced` — provides the full `Lws_context.t` instead of separate
  `~inputs`/`~outputs` arguments, for when you need access to the underlying simulator.

## Waveform access without the harness

If you are not using the harness, enable waveforms via `Backend_specific_config`:

```ocaml
let lws =
  L.create
    ~backend_specific_config:{ Lws_cyclesim.Backend_specific_config.default with waves = true }
    Dut.create
in
```

Print waveforms using:

```ocaml
Lws_context.Waveform.print ctx ~wave_width:2 ~display_rules;
Lws_context.Waveform.expect ctx ~wave_width:2 ~display_rules;
```

## Canonical Lws test template

```ocaml
open Core
open Hardcaml
open Hardcaml_lws

module Dut = My_module
module Lws_backend = Lws_cyclesim
module L = Lws_backend.With_interface(Dut.I)(Dut.O)

type sim_context = Lws_context.M(Dut.I)(Dut.O).t

let%expect_test "my test" =
  let testbench (h @ local) ({ inputs; outputs; _ } : sim_context) =
    let open Bits in
    inputs.enable := vdd;
    inputs.data <--. 42;
    Lws.step h;
    printf "count = %d\n" (to_unsigned_int !(outputs.after_edge.count));
    Lws.step ~n:5 h;
    printf "count = %d\n" (to_unsigned_int !(outputs.after_edge.count))
  in
  Lws_backend.run (L.create Dut.create) testbench;
  [%expect {|
    count = ...
    count = ...
    |}]
;;
```

## Using Standard Library Functions like List.iter and List.map

You can use `[@nontail]` annotations when you are calling local functions at a tail call,
but DO NOT annotate the tail-call of tail-call recursive functions. The annotation above
allows you to use `List.iter` / `List.map` / `Array.map` etc. functions.

This is an example of iterating through a list of items and stepping the simulation by one
clock cycle as you traverse through all the items:

```
open Hardcaml_lws

let foo (h @ local) =
  let items = [ 1; 2; 3 ] in
  List.map items ~f:(fun i ->
    Lws.step h;
    i)
    [@nontail]
;;
```

You should always prefer `List.map` / `List.iter` functions over writing custom
`for` loops or recursive functions, where possible.

## Conventions for `h` Argument

The convention is for the `h @ local` to have the name `h` and just a mode annotation, but
without the type annotation. The `[@nontail]` annotation on the tail call is necessary, as
the `f` argument captures `h @ local`. Tail calls that uses a local object in the current
call frame needs to be annotated with nontail.

## Using the lower-level Lws_runtime API

For advanced use cases (e.g., integrating Lws into a larger test harness), you can use the
`Lws_runtime` API directly:

```ocaml
let lws = Lws_cyclesim.to_polymorphic (L.create Dut.create) in
let trigger = Lws_runtime.resolve_trigger_by_clock_name lws "clock" |> Option.value_exn in
let task =
  Lws_runtime.schedule_task ~trigger lws (fun h ctx ->
    (* testbench code *)
    ...)
in
Lws_runtime.poll_task lws task;
```

# Cyclesim Simulations

Various tests in the tree uses the `Hardcaml_test_harness.Cyclesim_harness` for cyclesim
simulations that interacts directly with input and output ports of the simulator.


This following is a common template for a simulation using this template.

```
open Hardcaml_waveterm
open Hardcaml_test_harness

module Dut = ...
module Harness = Cyclesim_harness.Make(Dut.I)(Dut.O)
module Sim = Harness.Sim

let%expect_test "" =
  let debug = false in
  let testbench (sim : Sim.t) =
    let inputs = Cyclesim.inputs sim in
    let outputs_before = Cyclesim.outputs ~clock_edge:Before sim in
    let outputs_after = Cyclesim.outputs ~clock_edge:After sim in
    ...
  in
  let waves_config, trace =
    if debug then
      Waves_config.to_test_directory () , `Everything
    else
      Waves_config.no_waves, `All_named
  in
  (* Optional - include a waveform print out if it is appropriate *)
  let print_waves_after_test waves =
    Waveform.print waves
  in
  Harness.run_advanced ~trace ~waves_config ~create:Dut.create ~print_waves_after_test testbench
;;
```

`Waveform.print` takes plenty of arguments to configure the waveform printed.

In particular, the `~display_rules` argument controls which and how waves are
rendered. You should always use display_rules to render only relevant signals.


# Step Testbench (legacy)

The Step testbench is widely used in existing tests but **new tests should use Lws
instead**. Understanding the step testbench is still useful for reading and modifying
existing code. Similar to Lws, the step testbench uses an effect-based API with
cooperative scheduling.

There are two flavors: functional and imperative.


## Imperative Step Testbench

The imperative step testbench is the simpler flavor. The testbench pokes `Bits.t ref`
input ports directly (like Lws) and reads `Bits.t ref` output ports.

```ocaml
module Step = Hardcaml_step_testbench.Imperative.Cyclesim

let testbench (h @ local) =
  let inputs = Cyclesim.inputs sim in
  let outputs = Cyclesim.outputs ~clock_edge:After sim in
  inputs.enable := Bits.vdd;
  Step.cycle h;
  printf "%d\n" (Bits.to_unsigned_int !(outputs.count))
;;

Step.run_until_finished () ~simulator:sim ~testbench;
```

Key points:
- `Step.cycle h` advances the simulation by one clock cycle.
- `Step.spawn h f` spawns a concurrent task, returns a `finished_event`.
- `Step.wait_for h event` blocks until a spawned task completes.
- Input and output port access is through `Bits.t ref` from the simulator object.

## Functional Step Testbench

The functional step testbench passes input/output data explicitly through the `cycle`
function instead of poking refs directly.

```ocaml
module Step = Hardcaml_step_testbench.Functional.Cyclesim.Make(Dut.I)(Dut.O)

let testbench (h @ local) (initial_output : Step.O_data.t) =
  let input_data = { I. enable = Bits.vdd; data = Bits.of_unsigned_int ~width:8 42 } in
  let output = Step.cycle h input_data in
  let count = (Step.O_data.after_edge output).count in
  printf "%d\n" (Bits.to_unsigned_int count);
  ...
;;

Step.run_until_finished () ~simulator:sim ~testbench;
```

Key points:
- `Step.cycle h (input_data : Bits.t I.t)` applies the input data and returns
  `Bits.t O.t Before_and_after_edge.t` — output sampled before and after the clock edge.
- `Step.O_data.after_edge output` and `Step.O_data.before_edge output` extract the
  respective halves.
- `Step.spawn h f` spawns a child task. The child's `cycle` call also takes input data;
  child inputs are merged with the parent's using `merge_inputs`.
- `Step.spawn_io` allows spawning tasks that interact with a subset of the I/O ports,
  bridged via `~inputs` and `~outputs` mapping functions.
- `input_hold` (default) keeps unset inputs at their previous value; `input_zero` zeros
  them.

# Instantiations Expect Tests

Hardcaml simulations are sometimes functorized over a config. You need to instantiate the
tests over specific configurations, as follows:

```
module Make(Config : Config) = struct
  ... test body here ...
end

module%test Test_foo = Make(Foo_config)
```
