# Variant Interfaces (`[@@deriving hardcaml_variants]`)

## What this is NOT

This is **not** a hardware-level enum or mux type. It does not synthesize a runtime
selector or tag bits. The variant is resolved **at compile time** (technically at module
instantiation time) — you pick exactly one case when you apply the `Make` functor, and
that choice is fixed for the entire module. The resulting `Interface.S` only works with
values of that one case. Passing a value of the wrong variant case will raise at
simulation time and will **prevent hardware from synthesizing**.

If you need a hardware-level enum or multiplexed type, use `Enum` or `Signal.mux` /
`Signal.case` instead.

## When to use this

Use `hardcaml_variants` when you have a hardware module that can be configured with one of
several different interface shapes, and the choice is made once at build time.

## How it works

Given a variant type where each case wraps a Hardcaml `Interface.S`:

```ocaml
module Narrow = struct
  type 'a t =
    { addr : 'a [@bits 16]
    ; data : 'a [@bits 8]
    }
  [@@deriving hardcaml]
end

module Wide = struct
  type 'a t =
    { addr : 'a [@bits 32]
    ; data : 'a [@bits 64]
    }
  [@@deriving hardcaml]
end

module Bus = struct
  type 'a t =
    | Narrow of 'a Narrow.t
    | Wide of 'a Wide.t
  [@@deriving hardcaml_variants]
end
```

The deriver generates the following signature (shown for the `Bus` example above):

```ocaml
(* Enum of variant cases *)
module Kind : sig
  type t =
    | Narrow
    | Wide
  [@@deriving sexp]
end

(* The full module type produced by Make *)
module type S = sig
  val kind : Kind.t

  include Interface.S with type 'a t = 'a t

  val narrow_exn : 'a t -> 'a Narrow.t
  val wide_exn : 'a t -> 'a Wide.t

  val map_variants
    :  'a t
    -> narrow:('a Narrow.t -> 'b Narrow.t)
    -> wide:('a Wide.t -> 'b Wide.t)
    -> 'b t
end

(* Functor to instantiate for a specific kind *)
module Make (_ : sig 
    val kind : Kind.t
end) : S
```

In addition, `compare`, `equal`, and `sexp_of` derivations are included at the top level.

## Usage

### Instantiating for a specific kind

```ocaml
(* Each of these is a full Interface.S, specialized to one case. *)
module Narrow_bus = Bus.Make (struct let kind = Bus.Kind.Narrow end)
module Wide_bus = Bus.Make (struct let kind = Bus.Kind.Wide end)
```

`Narrow_bus` is now a valid `Hardcaml.Interface.S`. You can use it anywhere an interface is
expected — in `Hardcaml.Circuit.With_interface`, in testbenches, etc.

### Correct usage — all values match the configured kind

```ocaml
(* This works: creating and operating on Narrow values through Narrow_bus *)
let x : int Bus.t = Narrow { addr = 100; data = 50 }
let doubled = Narrow_bus.map x ~f:(fun v -> v * 2)
(* doubled = Narrow { addr = 200; data = 100 } *)
```

### What happens with mismatched kinds

**This will raise at runtime and prevent hardware from synthesizing:**

```ocaml
(* BAD: Narrow_bus expects Narrow values, but we pass a Wide *)
let wrong : int Bus.t = Wide { addr = 42; data = 0 }
let _ = Narrow_bus.map wrong ~f:(fun v -> v + 1)
(* Raises: ("mismatched tag" (kind Narrow) (t (Wide ((addr _) (data _))))) *)
```

This also applies to binary operations:

```ocaml
let a : int Bus.t = Narrow { addr = 1; data = 2 }
let b : int Bus.t = Wide { addr = 3; data = 4 }

(* BAD: both arguments must be the configured kind *)
let _ = Narrow_bus.map2 a b ~f:( + )
(* Raises: ("mismatched tag" (kind Narrow) (t (Wide ((addr _) (data _))))) *)
```

The key point: in a hardware design context, `port_names_and_widths` is determined at
functor application time by the chosen kind, so attempting to use mismatched values will
either raise during circuit construction or produce nonsensical wiring. **There is no
hardware mux selecting between variants at runtime.**

### Using map_variants (kind-agnostic operations)

If you need to operate on a `Bus.t` without knowing which kind it is, use
`map_variants` which takes a labeled function for each case:

```ocaml
let result =
  Bus.map_variants some_bus
    ~narrow:(fun (n : int Narrow.t) -> Narrow.map n ~f:(( * ) 2))
    ~wide:(fun (w : int Wide.t) -> Wide.map w ~f:(( + ) 1))
```

### Extractor functions

The deriver generates `_exn` functions to extract the inner value:

```ocaml
let bus = Bus.Narrow { addr = 10; data = 5 }
let inner : int Narrow.t = Bus.narrow_exn bus   (* OK *)
let _ = Bus.wide_exn bus                         (* Raises *)
```

## Constraints

- Every variant case must carry exactly one payload of the form `'a Some_module.t`, where
  `Some_module` satisfies `Interface.S`.
