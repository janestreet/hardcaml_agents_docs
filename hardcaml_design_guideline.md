# Hardcaml Module Conventions

For make configurable parameters in verilog, that standard pattern to first define a
Config module type.

```ocaml
module type Config = sig
  val foo_bits : int

  (** A brief comment describing what [bar_latency] is *)
  val bar_latency : int

  module Bar : Hardcaml.Interface.S
end
```

You can refer to the Config fields in the I and O interfaces to define certain
types.

```ocaml
open! Hardcaml

module Make(Config : Config) = struct
  module I = struct
    type 'a t =
      { clock : 'a
      ; foo : 'a [@bits Config.foo_bits]
      }
    [@@deriving hardcaml]
  end

  module O = struct
    type 'a t =
      { bar : 'a Bar.t
      ; num_events : 'a [@bits 32]
      }
    [@@deriving hardcaml]
  end

  let create (scope : Scope.t) (i : Signal.t I.t) =
    (* Implementation here *)
    ...
  ;;
end
```

# ppx_hardcaml

Add ppx_hardcaml to the list of preprocessors in the jbuild or dune file of a hardcaml
library.

```
(library (
  (name my_library_name)
  (libraries (hardcaml))
  (preprocess (pps (ppx_hardcaml ppx_jane)))))
```

deriving hardcaml implicitly includes sexp_of.
Do not write `[@@deriving sexp_of, hardcaml]` It is a very bad pattern.

# Signal and Always API

In Hardware design code, have the following import at the top of your file:

```
open! Hardcaml
open! Signal
```

The `~enable` argument in `reg` / `reg_fb` is optional and defaults to vdd.

Using `~enable:vdd` is BAD PRACTICE. DO NOT do the following:

```
let foo = reg spec ~enable:vdd bar in
```

Include it only when you need to override it to something else.

Use let%hw to name Signal.t. Use let%hw_var to name Always.Variable.t. The following
is how you do this.

```
let%hw foo = a +: b in

let%hw_var foo_reg = Always.Variable.reg ~width:8 spec in

let%hw.Always.State_machine sm = Always.State_machine.create (module State) spec in
```

DO NOT USE the `--` operator. IT IS DANGEOURS. The following is BAD PRACTICE

```
let foo = a +: b -- "foo" in

let foo = a +: b in
ignore (foo -- "foo" : Signal.t)

let foo_reg = Always.Variable.reg ~width:8 spec in
ignore (foo_reg.value -- "foo_reg")

let sm = Always.State_machine.create (module State) spec in
ignore (sm.current -- "sm" : Signal.t);
```

To convert an OCaml integer to Signal.t , use `of_unsigned_int` or `of_signed_int`

```
let a = of_unsigned_int ~width:8 10 in
let b = of_unsigned_int ~width:8 10 in
let c = of_signed_int ~width:8 (-128) in
```

DO NOT USE `of_int_trunc` IT IS BROKEN AND DANGEROUS. The following is BAD PRACTICE.

```
let d = of_int_trunc ~width:8 255
```

When considering what should go into the Always API vs Signal API:

- Prefer writing inline combinational logic like `let%hw x = mux2 cond foo bar` over
  writing if_ assignments inside. This is true for most combinational logic.
- Only exception to the rule above is is there is a combinational logic that is driven in
  a particular state in a state machine in the always DSL. It is appropriate to use
  Always.Variable.wire in such cases.
- A register that is driven by a next signal should use Signal.reg rather than
  Always.Variable.reg

Follow the following good practice:

```ocaml
let sm = Always.State_machine.create (module State) in
let is_active_wire = Always.Variable.wire ~default:gnd in
let cycles_since_inception =
  reg_fb spec ~width:8 ~enable:stream.valid ~f:(fun fb ->
    fb +:. 1)
in
let foo_next = Alway.Variable.wire ~default:gnd in
let foo = reg spec foo_next in
Always.(compile [
  sm.switch [
    Idle, [
      when_ (cycles_since_inception >:. 10) [
        sm.set_next Active;
      ]
    ]
  ; Active, [
      is_active_wire <--. (cycles_since_inception >:. 1);
      foo_next <-- (cycles_since_inception ==:. 255);
    ]
  ]
])
```

The following contains bits of bad practice with the Always DSL you should avoid.

```ocaml
let sm = Always.State_machine.create (module State) in
let cycles_since_inception = Always.Variable.reg ~width:8 spec in
let foo_next = Alway.Variable.wire ~default:gnd in
let foo = Always.Variable.reg spec ~width:32 in
(* BAD PATTERN because [is_active] is driven by the current state, so it should live in the Always DSL. *)
let is_active = sm.is Active &: (cycles_since_inception >:. 1) in
Always.(compile [
  (* BAD PATTERN because foo is just driven by a next signal *)
  foo <-- foo_next;

  (* BAD PATTERN because cycles_since_inception is not affected by the state of the state machine. *)
  when_ stream.valid [
    cycles_since_inception <-- (cycles_since_inception.value +:. 1);
  ]

  sm.switch [
    Idle, [
      when_ (cycles_since_inception >:. 10) [
        sm.set_next Active;
      ]
    ]
  ; Active, [
      foo_next <-- (cycles_since_inception ==:. 255);
    ]
  ]
])
```

When writing a multiplexer in the event that you have a small number of concreate cases and everything
else uses a default cases, make use of the `val Signal.case : default: t -> (t * t) list -> t` function.
This is akin a match statement in Ocaml with a default case.

To index a specific bit in a Signal.t instance, use the `val .:() : t -> int -> t` operator.

To index range of bits in a Signal.t instance, use the `val .:[] : t -> int -> int -> t` operator.
Eg: foo.:[10, 0] will return a 11-bit wide signal, where foo.:(0) is the lsb and foo.:(10)
is the msb.

Use unary operators in operator form. Insert parens around the argument. For example:

```
let not_foo = ~:(foo) in
let not_bar_baz = ~:(bar.baz) in
when_ (~:(bar.baz)) [
  ...
]
```

Using unary operators in prefix notation is BAD PRACTICE. Do not do this:

```
let not_foo = (~:) foo in
let not_bar_baz = (~:) bar.baz in
when_ ((~:) bar.baz)) [
  ...
]
```

When writing a chain of mux2 conditionals with 2, 3 or 4 cases, you should use the @@
operator to prevent growing the indentation. For example:

```
let%hw foo =
  mux2
    cond_a
    a_very_long_statement_that_is_result_a
  @@ mux2
       cond_b
       a_very_long_statement_that_is_result_b
  @@ mux2
      cond_c
      a_very_long_statement_that_is_result_c
  @@ default_resut
in
```

If you have more than 5 cases, use the `priority_select_with_default` function.

Letting the mux2 chain indentation grow is BAD PRACTICE. Do not do the following:

```
let%hw foo =
  mux2
    cond_a
    a_very_long_statement_that_is_result_a
    (mux2
       cond_b
       a_very_long_statement_that_is_result_b
       (mux2
          cond_c
          a_very_long_statement_that_is_result_c
          default_resut)
in
```

Usine @@ on mux2 that doesn't form a chain is BAD PRACTICE. Do not write this

```
let%hw foo = mux2 cond result1 @@ result2 in
```

Write this instead:

```
let%hw foo = mux2 cond result1 result2 in
```

Leaving unused values in your code is BAD PRACTICE. Do not do this:

```
let special_code_0 = 0
let _special_code_1 = 1 (* UNUSED - BAD PRACTICE *)
```

The `Signal.t` and `Always.Variable.t` are different types. To use the Signal.t APIs with
a Always.Variable.t, access the underlying `Signal.t` via the the `.value` field. For
example:

```
let foo_reg = Always.Variable.reg spec ~width:8 in
let foo = a +: foo_reg.value in
```

The following WILL NOT COMPILE. Do not do this

```
let foo_reg = Always.Variable.reg spec ~width:8 in
let foo = a +: foo_reg in
```

Always use parentheses around Hardcaml operator usage to make your intent explicit.
Hardcaml has unintuitive operator precedence — for example, `&:` and `|:` have equal
precedence and are left-associative, so `a |: b &: c` parses as `(a |: b) &: c` (unlike
Verilog/C where `&` binds tighter than `|`). Other operators like `+:`, `*:`, `^:`, and
`@:` have their own precedence levels and associativities that also differ from Verilog
conventions in subtle ways. Do not rely on any of this — always parenthesize. The
auto-formatter may remove some redundant parentheses, but this is fine since it preserves
the semantics. Examples:

```
let foo = (a +: b) -: d in
let foo = (a <: b) &: (e >: f) in
```

DO NOT rely on operator precedence. This is BAD PRACTICE and IS VERY DANGEROUS. DO NOT do
the following

```
let foo = a +: b -: d in
let foo = a <: b &: e >: f in
```

The `*:`, `<:`, `>:`  operators treats its argument as unsigned.

The `*+`, `<+`, `>+`  operators treats its argument as signed.

# Naming Signals with ppx_hardcaml

Naming signals is important in Hardcaml. Named signals show up in waveforms and in the
generated RTL, making debugging much easier. Unnamed signals get anonymous names like
`_T_123`. Always prefer to name as many variables as possible.

The `let%hw` family of extensions requires a `Scope.t` variable called `scope` to be in
context. They name the signal with the OCaml variable name automatically.

```ocaml
(* For Signal.t *)
let%hw my_signal = some_expression in
let%hw_list my_signal_list = some_list_expression in
let%hw_array my_signal_array = some_array_expression in

(* For Always.Variable.t *)
let%hw_var my_always_var = Always.Variable.reg spec ~width:8 in
let%hw_var_list my_always_var_list = some_list_of_always_vars in
let%hw_var_array my_always_var_array = some_array_of_always_vars in

(* For Signal.t Foo.t (interface records) *)
let%hw.Foo.Of_signal my_foo = some_interface_expression in
let%hw_array.Foo.Of_signal my_foo_array = some_array_of_interfaces in

(* For Always.Variable.t Foo.t (interface records of always variables) *)
let%hw.Foo.Of_always my_foo_always = some_always_interface_expression in
let%hw_array.Foo.Of_always my_foo_always_array = some_array_of_always_interfaces in

(* For Always.State_machine *)
let%hw.Always.State_machine sm = Always.State_machine.create (module State) spec in

(* Tuples (all components must have the same type) *)
let%hw a, b = some_tuple_expression in
```

## Subscope creation

When writing helper functions that produce named signals, use `let%subscope` to
automatically create a sub-scope. This prevents name collisions across instantiations:

```ocaml
(* Without subscope: manual and repetitive *)
let my_helper scope inputs =
  let scope = Scope.sub_scope scope "my_helper" in
  let%hw result = ... in
  result
;;

(* With subscope: automatic *)
let%subscope my_helper scope inputs =
  let%hw result = ... in
  result
;;
```

## RTL naming in interface definitions

By default, Hardcaml uses the OCaml field name as the RTL signal name for interface ports.
This is usually what you want — manual naming is discouraged unless:

- There is a RTL name collision between the I and O interfaces (e.g., both have a
  field called `valid`).
- You are interfacing with external IP that expects specific port names.

When you do need to override names, use these field attributes:

```ocaml
type 'a t =
  { (* Rename a field in RTL *)
    foo : 'a [@rtlname "bar"]            (* "foo" in OCaml, "bar" in RTL *)

  ; (* Add a prefix or suffix *)
    baz : 'a [@rtlprefix "tx_"]          (* becomes "tx_baz" in RTL *)
  ; qux : 'a [@rtlsuffix "_out"]         (* becomes "qux_out" in RTL *)

  ; (* For nested interfaces: prefix all sub-fields *)
    sub : 'a Sub.t [@rtlprefix "sub_"]   (* all fields in Sub.t get "sub_" prefix *)
  ; sub2 : 'a Sub.t [@rtlmangle "_"]     (* auto-prefix with field name: "sub2_..." *)
  }
[@@deriving hardcaml]
```

You can also apply a prefix or suffix globally to all fields:

```ocaml
type 'a t = { ... } [@@deriving hardcaml ~rtlprefix:"tx_"]
```

# Reg_spec creation

`Reg_spec.t` specifies the clock, reset, and clear signals for registers. Create one at
the top of your `create` function and reuse it:

```ocaml
let create scope (i : Signal.t I.t) =
  let spec = Reg_spec.create ~clock:i.clock ~clear:i.clear () in
  ...
;;
```

`Reg_spec.create` requires `~clock` and accepts optional `~reset`, `~clear`,
`~clock_edge`, and `~reset_level` arguments.

# cut_through_reg

`Signal.cut_through_reg` creates a register that forwards its input directly to the output
on cycles where `~enable` is high. When enable is low, the stored register value is
output. Implemented as `mux2 enable d (reg spec ~enable d)`.

This is useful for pipelines where you want zero-latency forwarding when the pipeline is
advancing:

```ocaml
(* Signal-level *)
let%hw q = cut_through_reg spec ~enable d in

(* Always-level: creates an Always.Variable.t with cut-through behaviour *)
let%hw_var q = Always.Variable.cut_through_reg spec ~width:8 in

(* Via Clocking *)
let%hw q = Clocking.cut_through_reg clocking ~enable d in
let%hw_var q = Clocking.Var.cut_through_reg clocking ~width:8 in
```

# Clocking module

`Clocking` is a convenience module that bundles `clock` and `clear` together. It is
possible to use a `Clocking.t` instead of a `Reg_spec.t`:

```ocaml
let clocking = { Clocking.clock = i.clock; clear = i.clear } in

(* Use it for registers *)
let%hw q = Clocking.reg clocking ~enable d in

(* Convert to Reg_spec.t when needed *)
let spec = Clocking.to_spec clocking in
```

# Hierarchical design with Hierarchy.In_scope

To instantiate a sub-module hierarchically (so it appears as a separate module in RTL
rather than being inlined), use `Hierarchy.In_scope`. The standard pattern is to define
`module I`, `module O`, `let create`, and `let hierarchical` within each module:

```ocaml
module Make (Config : Config) = struct
  module I = struct
    type 'a t =
      { clocking : 'a Clocking.t
      ; data_in : 'a [@bits Config.data_bits]
      }
    [@@deriving hardcaml]
  end

  module O = struct
    type 'a t =
      { data_out : 'a [@bits Config.data_bits]
      }
    [@@deriving hardcaml]
  end

  let create scope { I.clocking; data_in } =
    let spec = Clocking.to_spec clocking in
    let%hw data_out = reg spec ~enable:vdd data_in in
    { O.data_out }
  ;;

  let hierarchical ?instance scope i =
    let module H = Hierarchy.In_scope (I) (O) in
    H.hierarchical ?instance ~scope ~name:"my_module" create i
  ;;
end
```

Callers use the `hierarchical` function instead of `create` to get a separate Verilog
module in the hierarchy:

```ocaml
let create scope (i : Signal.t I.t) =
  let sub_o = Sub.hierarchical ~scope { clocking = i.clocking; data_in = i.foo } in
  { O.result = sub_o.data_out }
;;
```

The `~scope` argument is threaded through so that signal names and circuit databases are
properly maintained. By default, the instantiation will be hierarchical when
`flatten_design` is false and inlined otherwise.

# with_valid and priority selection

The `with_valid` type is used throughout Hardcaml for handshake-style signals:

```ocaml
type 'a with_valid = { valid : 'a; value : 'a }
```

Use `priority_select_with_default` for priority-encoded multiplexing:

```ocaml
let%hw result =
  priority_select_with_default
    [ { valid = cond_a; value = value_a }
    ; { valid = cond_b; value = value_b }
    ; { valid = cond_c; value = value_c }
    ]
    ~default:default_value
in
```

Use `onehot_select` when exactly one condition is guaranteed to be high.

# Concatenation and splitting

```ocaml
(* Concatenate: msb of head becomes msb of result *)
let%hw combined = concat_msb [ upper; lower ] in

(* Infix concatenation: a @: b == concat_msb [ a; b ] *)
let%hw combined = upper @: lower in

(* Split into parts of given width, lsb-first *)
let parts = split_lsb ~part_width:8 wide_signal in
```

# Interface Of_signal and Of_always operations

When working with interface records (`'a I.t`, `'a O.t`), use the `Of_signal` and
`Of_always` sub-modules:

```ocaml
(* Create a zero-valued interface *)
let zeros = I.Of_signal.const 0 in

(* Create always-variable registers for an interface *)
let%hw.O.Of_always regs = O.Of_always.reg spec in

(* Read the Signal.t values from always variables *)
let output = O.Of_always.value regs in

(* Assign an interface of signals to an interface of always variables *)
O.Of_always.assign regs signals;
```
