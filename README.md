# syslUI

A declarative retained user interface for sysl, for a machine with a heap.

**This is a probe, not yet a framework.** Card `0203` asks six questions about whether the
architecture decided in `0168` survives contact with the compiler, and everything here exists to
answer them. Nothing in it should be read as a settled API.

`imui` is the sibling for a panel on a microcontroller and is not replaced by this. It refuses a
retained tree on purpose, and the split between the two is by application size rather than by
device.

**Desktop first**, mobile once it works well there, embedded only maybe.

## What the probe has established

Nine tests, `sysl test .` green.

- **A modifier chain works.** `text("hi").padding(8).background(red)` — the modifiers are trait
  defaults on `View` returning `&View`, with `&self` receivers so a wrapper stores the child's box
  rather than a copy. Object safety admits them because `Self` appears nowhere but the receiver.
- **The erasure survives a wrapper.** A leaf, a modifier chain and a fixed frame meet in one
  `Buf[&View]`. The Rust failure mode — a default returning a box needs `Self: Sized`, so it is
  unavailable on the trait object — does not occur.
- **`weak Fn() -> unit` is a working dependent list, and the drop is the unsubscribe.** A dependent
  whose last strong reference has gone is swept on the next notify, so no teardown path in the
  framework has to remember to unhook.
- **State that outlives a rebuild lives in the signal, and the view stays stateless.** A scroll
  offset in a `&Signal[int]` survives the entire tree being thrown away and built again — verified
  by building a *different* tree that shares only the signal. This was the question most likely to
  move the architecture, and it did not.
- **One layout pass works** — constraints down, sizes up, the parent places.

Not yet answered: what a rebuild costs under ARC. That needs a real frame loop against SDL3.

## The architecture, in one paragraph

`View` is an ordinary object-safe trait and children are `[]&View`, so the tree is erased and kept
alive by ARC. There is **no reconciler**: diffing an erased tree would have to ask a node what type
it is, which object safety deliberately refuses, so a write to a `Signal[T]` notifies exactly what
read it instead. That is where SolidJS, Svelte 5, Angular and Swift Observation all landed. SwiftUI
encodes the whole tree in the static type of `body` and pays for it with associated types and an
opaque result type; that buys no box per node, which is worth having on a microcontroller and worth
very little here.

## Layout

- `sh/sysl/ui/view.sysl` — the `View` trait, the leaves, and the modifiers
- `sh/sysl/ui/signal.sysl` — `Signal[T]` and its weak dependent list
- `sh/sysl/ui/layout.sysl` — `Column` and `Row`
- `sh/sysl/ui/scroll.sysl` — where long-lived state lives
- `sh/sysl/ui/canvas.sysl` — a backend that draws nothing and remembers everything, so the tree can
  be asserted with no window
- `sh/sysl/ui/tests.sysl` — the findings, kept as tests rather than as prose
