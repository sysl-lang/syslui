# syslUI

A declarative retained user interface for sysl, for a machine with a heap.

It began as a probe — card `0203` asks six questions about whether the architecture decided in
`0168` survives contact with the compiler — and all six are answered. What is here now is a styling
layer built on top of those answers: chainable modifiers, a view whose appearance is decided while it
is drawn, and animation. The API is still moving.

`imui` is the sibling for a panel on a microcontroller and is not replaced by this. It refuses a
retained tree on purpose, and the split between the two is by application size rather than by
device.

**Desktop first**, mobile once it works well there, embedded only maybe.

## What the probe has established

A hundred and ninety-one tests, `sysl test .` green — and all six of card `0203`'s questions answered.

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
- **A rebuild costs about 22 ns and one allocation a node**, which is nothing, and **the drawing
  costs four orders of magnitude more**. Measured through `syslui-demo --bench`, a real frame at
  900×660 with a counting allocator, at three list lengths:

  | rows | boxed nodes | rebuild | + rasterize | upload | allocations a rebuild |
  |---:|---:|---:|---:|---:|---:|
  | 30 | 92 | 2.8 µs | 4.71 ms | 65 µs | 124 |
  | 300 | 632 | 14.6 µs | 4.65 ms | 66 µs | 664 |
  | 3000 | 6032 | 126.9 µs | 5.21 ms | 64 µs | 6064 |

  Three things worth reading off that table. **The rebuild is linear in the node count and free** —
  0.76% of a 60 Hz frame at three thousand rows. **The frame is nearly flat in the list length**,
  because a row outside the clip is measured and not painted; before culling, three hundred rows
  cost 7.74 ms against twenty rows' 4.98 ms, and now the two are the same to within noise. And
  **the upload is not the problem anybody expects it to be**: 2.3 MiB across to the GPU every frame
  is 0.4% of the budget.

  What is left is the rasterizer, and it is almost all text — the same window with no list at all
  draws in 0.51 ms. The demo holds a vsync-locked 120 fps.

  **So the per-frame arena the mobile survey argued for is refused, and by a wider margin than
  before**: allocation is now three parts in a thousand of a frame.

## What a program looks like

```
column(spacing = 10):
    text(s"count: ${count.read()}").padding(6)

    row([
        button("increment", () -> count.set(count.read() + 1)),
        button("decrement", () -> count.set(count.read() - 1))
    ], 10)

    scroll(column(rows.map(r -> text(r).padding(3)), 0), offset, 420)
```

The block fills the first parameter no written argument took, so naming `spacing` ahead of it still
leaves the block to `children`. A `for` cannot go inside one — a block is a list of expressions —
so a list whose length is known only while running is named with `sysl.seq`'s `map` and handed to
the container whole, rather than accumulated into a `Buf` beside the call.

## The architecture, in one paragraph

`View` is an ordinary object-safe trait and children are `[]&View`, so the tree is erased and kept
alive by ARC. There is **no reconciler**: diffing an erased tree would have to ask a node what type
it is, which object safety deliberately refuses, so a write to a `Signal[T]` notifies exactly what
read it instead. That is where SolidJS, Svelte 5, Angular and Swift Observation all landed. SwiftUI
encodes the whole tree in the static type of `body` and pays for it with associated types and an
opaque result type; that buys no box per node, which is worth having on a microcontroller and worth
very little here.

## Styling, and why a hover needs a view of its own

A tree is built before anything knows where the pointer is — that is what a rebuild *is*, a function
of the application state alone — so a node cannot be handed the colour it will need. Immediate mode
does not have this problem: `imui` draws and tests the pointer in the same call, so a hover effect
there is an `if`.

So a node carries every appearance it might take and chooses while **painting**, which is the only
moment a view knows its rectangle and can ask the canvas about it. That is SwiftUI's `ButtonStyle`
exactly, arrived at from the other direction:

```
reactive(i -> text(label)
    .foreground(WHITE)
    .padding(8)
    .background(if i.pressed() then sunk else lifted, 6)).on_tap(f)
```

`measure` uses the resting state and never another, or a style a pixel wider when hovered would
relayout the row under the cursor and shove its neighbours out from under the pointer.

The modifiers are `padding`, `background(color, radius)`, `border(color, width, radius)`,
`foreground(color)`, `frame` and `on_tap`. A border is drawn **inside** the rectangle layout already
settled on, so adding one moves nothing. Text colour is canvas state rather than a parameter on
`text`, which is what lets a styled view set the ink around whatever it wraps.

## The controls

`switch`, `checkbox`, `radio`, `slider`, `progress` and `divider`, in `widgets.sysl`. Every one of
them is the same four things: a size it asks for, some filled rounded rectangles, a question put to
the canvas about the pointer, and a phase easing toward the answer. There is nothing else, because
`Canvas` offers nothing else — which is what lets a switch drawn as two rounded rectangles be a
switch on a phone as well as on a desktop.

**A control owns no state.** A `&Signal[bool]` is passed in, read while painting and written when
tapped, so a rebuild between those two moments changes nothing. That is `scroll.sysl`'s finding
applied everywhere, and it is what makes a control safe to throw away sixty times a second.

Two of them are worth singling out:

- **A checkbox is a `Reactive` wrapping a `Row` wrapping a state-dependent leaf**, so pointing at
  the *label* lights the *box*. No widget had to be told about another; it is the styling layer
  composing somewhere other than a button.
- **A radio group is a `column` of options over one `int`, and there is no `RadioGroup` view.** The
  mutual exclusion it exists for lives entirely in the signal its options share, so there is no state
  in which two are on and nothing anywhere has to clear the previous one — which is also why an
  option is callable on its own and can sit in a different half of a form from its siblings.
- **A slider is the one thing that reads the pointer's position**, because a hit region carries no
  coordinate and a control whose value *is* a coordinate cannot be served by one. So `Canvas` gained
  `pointer()`, the drag happens during `paint`, and the write is guarded on the value actually
  changing — an unguarded one would mark every frame dirty for as long as a pointer rested on a
  slider nobody was moving.

## Theming

A `Theme` is nineteen numbers: six role colours, a text colour for labels on filled controls, ink,
inert, ground, panel, edge, alt, a corner radius, a padding, three lifts and a soft mix. Every
control reads all of its appearance off it and computes none of it, which is the whole test of
whether a theme is finished.

**It lives on the canvas, for the same reason the pointer and the phases do: the tree has nowhere to
keep it.** A node is built before anything knows what it will be drawn into, so a colour cannot be
handed to it at construction. And it is a *stack* rather than a field, so a subtree can be restyled
— the discipline `ink`/`unink` and `clip`/`unclip` already keep.

```
switch(on)                                  // the theme's primary
switch(on, .Success)                         // a role: green here, green in every theme
button("Delete", drop, .Error)
panel.restyle(t -> light())                  // this subtree, and everything under it
button("Brand", f).tinted(0xE2007A)          // a colour the palette has no name for
```

`Themed` holds a **function** `Theme -> Theme` rather than a theme, which is what makes restyles
nest: a wrapper storing a whole theme could only replace one, and could not have known what the
theme above it did anyway. `measure` pushes as well as `paint`, and that is load-bearing — `pad` and
`radius` are in the theme, so a restyled button is a different *size*.

**A caller names a role and never a colour, because a caller cannot see the theme.** A hex literal
at a call site is a colour that will still be there when the light theme is not. Six roles is what
every design system converges on and they are not interchangeable: `Error` and `Warning` differ in
whether the action can be undone.

### Button variants

Four shapes × six roles = twenty-four appearances out of ten numbers, and every one of them moves
when the theme does.

| variant | what it is |
|---|---|
| `Filled` | the colour is the ground, label in `on_fill`. The loud one; one to a screen |
| `Outline` | an outline and a label in the colour, and **no ground at all** |
| `Dashed` | an outline that is dashed — says "not yet" where a solid one says "no" |
| `Soft` | a muted ground mixed from `ground` toward the colour. The one that scales |

**`Outline` fills nothing, ever, and that is a constraint rather than a taste.** `Canvas` has no
alpha, so a "transparent" ground would have to be painted in whatever is behind it — and a view does
not know what is behind it. Its hover is carried by the outline and the label brightening and the
outline thickening, which needs no such knowledge and is why it is the one variant that can sit on
an unknown surface.

`Canvas.stroke` gained a `dash` for the dashed one: **one number, not a pattern**, meaning that many
pixels on and that many off. A general dash array would be a slice a backend has to keep alive past
the call, and what a dashed border is *for* is saying "provisional" at a glance.

## Positioning: the grid

`Row` and `Column` answer *what comes after what*. A grid answers *where*.

A row gives each child the width that child measured to, so a form laid out as rows has its fields
starting wherever their labels' text happened to end — which looks like an accident, because it is
one. A grid divides the width into equal columns and a child says how many it takes, so two children
in different lines line up because they were given the same fractions rather than because their
contents were the same length.

```
grid():
    text("volume").padding(4).cell(4)
    slider(volume, 0, 100).cell(16)
    text(s"${volume.read()}%").padding(4).cell(4)

    text("progress").padding(4).cell(4)
    progress(done, 380, .Success).cell(16)
    text("").cell(4)
```

**Two counts, chosen to be complementary rather than coarse and fine.** Both give halves, thirds and
sixths; each gives one thing the other cannot.

| | halves | thirds | quarters | fifths | sixths | eighths | tenths |
|---|---|---|---|---|---|---|---|
| **24** | 12 | 8 | 6 | — | 4 | 3 | — |
| **30** | 15 | 10 | — | 6 | 5 | — | 3 |

`COLUMNS_24` is the default — quarters and eighths are what an ordinary layout is made of.
`COLUMNS_30` is for a layout with fifths in it, which 24 simply cannot express.

**60 and 120 are the obvious alternatives and are deliberately absent.** Each exists only to hold
both families at once, and the price is a count whose spans stop meaning anything at a glance: a span
of 40 in 120 reads as a number, where 6 in 30 reads as a fifth. Two small grids you can do arithmetic
on in your head beat one large one you cannot.

**The count lives in the `Theme`**, for the same reason the colours do: what makes a grid worth
having is that things in different parts of an interface line up, and they can only do that if
nobody chose separately. A section wanting fifths writes
`.restyle(t -> t with { columns = COLUMNS_30 })`.

`.cell(span, offset)` skips `offset` columns first, a line wraps when it runs out, and an unmarked
child takes the whole width. **`.cell` must be the last link in a chain** — a wrapper outside it
would answer the grid's question for itself, the same rule that makes `.frame()` stop a `Spacer`
being flexible.

The edges are computed as `col * (w + gutter) / cols` rather than from a column width, so the
rounding falls between columns instead of accumulating at the right-hand end, and the last edge lands
exactly on the width.

## The table

A scrollable, sortable-width, striped, tappable table — and **it adds no drawing at all**. It is a
`Column` of `Row`s inside a `Scroll` under a header that does not scroll, and every one of those
already existed. If a table had needed a new `Canvas` operation it would have meant the surface was
wrong, rather than that tables are special.

```
table([col("#", true), flex_col("name"), col("status"), col("value", true)],
      rows, offset, 340, Some(i -> select(i)))
```

**What it adds is the one thing containers cannot do for themselves: columns that line up.** A `Row`
gives each child the width that child measured to, so two rows of the same shape produce two
different sets of column edges the moment one cell's text is longer. So the widths are decided once,
by the table, from the header and every cell, and handed down as fixed frames. Three kinds:
`col` fits its widest cell, `fixed_col` takes exactly what it is given, `flex_col` shares out what is
left over.

**Cells are strings, not views, and that is a deliberate limit.** A cell that could be any `&View`
would mean measuring a tree per cell to decide a column width, twice a frame, for every row including
the ones nobody can see. Strings go through the canvas's measurement cache, so a column's width costs
one shaping pass per distinct string for the life of the canvas.

That limit buys the thing culling could not: **a table builds only the rows that can be seen.**
Culling stops an invisible row being *painted*, but the row still has to exist to be culled — and a
table's body is built during paint, so 300 rows meant 12,605 boxes made and thrown away every frame.
Every row being the same height turns "which rows are visible" into arithmetic, and the rows above
and below become two framed spacers so the list keeps its full height for the scrollbar to be a
fraction of.

| | allocations a frame | frame |
|---|---:|---:|
| all rows built | 12,605 | 1.69 ms |
| only the window | **933** | **1.21 ms** |
| …at 3000 rows | **933** | **1.33 ms** |

### Controlling how it looks

A table has more surface than everything else here put together, so its appearance is a
**`TableStyle` block inside the `Theme`** rather than ten more fields on `Theme` — header ground and
ink, the ordinary row, the stripe, the hovered row, the rule, whether rows are striped, ruled or
highlighted, and the cell inset that decides the row height.

```
table(cols, rows, offset, 340)
    .restyle(t -> t with { table = t.table with { striped = false, lined = true } })
```

**Striping is the theme's, not the call's.** A flag on `table(...)` would be one table's appearance
decided at one call site, which is the thing a theme exists to prevent — and striping and rules are
alternatives rather than companions, since both say "this row ends here".

Until this block existed the header's text was `mix(ink, panel, 35)` computed inside `table.sysl` — a
colour a theme could not name and therefore could not change, which is exactly what a theme is for.

**Every row paints its own ground**, because `Canvas` has no alpha: a row that wanted to be "the
surface underneath" would have to know what that was. Naming it makes the stripe, the hover and the
ordinary row one piece of arithmetic rather than three special cases.

**The hovered row is a `Reactive`**, so it eases in — the same `Interaction` that drives a button
drives a table row, and nothing new was needed for it.

## Overlays — the one thing that does not compose downward

Everything else here composes downward: a view is given a rectangle and must not draw outside it, a
container clips its children, and culling skips what the clip cannot reach. That is exactly wrong for
a dropdown, which is anchored to something small and has to spill out over what comes after it.

**The answer is a second pass, not a second tree.** A view that wants to be above everything doesn't
paint itself — it hands the canvas a view and a rectangle, and the frame paints those after the tree.
By then every clip has been unwound, so an overlay is unclipped without asking; and because the hit
list is last-wins, it is on top for input without asking either. Neither needed a special case.

```
button("Options", () -> open.set(!open.read()))
    .popover(open, menu)              // anchored below the trigger, escapes any clip

body.modal(confirming, confirm_panel, Some(cancel))   // centred, everything under it untappable
```

Use **`render(tree, c, rect)`** rather than `measure` and `paint` by hand — an application that
painted the tree itself would silently never draw a dialog, which is the same class of failure as
forgetting to subscribe a signal, designed out the same way.

`raise_above` is the only member on `Canvas` with **no default body**. Elsewhere the rule is that not
implementing something costs the feature and never the frame — a backend ignoring `visible` is merely
slower. A backend ignoring this would draw a frame with the dialog *missing*, which is a wrong
picture rather than a plainer one.

**What is missing, for a stated reason:** a modal has no dimmed backdrop. Dimming needs alpha and
`Canvas` has none — a solid rectangle would hide the interface rather than subdue it. What a modal
must do regardless is stop what's behind being clicked, and a full-size region that swallows taps
does that on its own.

## Tabs

`tabs(labels, chosen, i -> panel(i))` — **and there is no `Tabs` view.** A tab bar is a `Row` of
`Reactive` headers, each a `Column` of a label over a restyled two-pixel `divider`, above a rule and
the chosen panel. All of that already existed; if tabs had needed a new `View` or a new `Canvas`
operation it would have meant something below was missing.

The panel is a **function of the index**, so only the chosen one is built — a list of views would
build every panel on every rebuild, including the ones nobody is looking at, to show one.

**Whether a header is chosen is captured at build time, not read while painting**, because the panel
below it is also built once. A header reading the signal live would move its indicator on the frame a
tab was tapped while the panel waited for the rebuild — one frame of the bar highlighting the wrong
thing.

## Spacing

```
text("title").pad(bottom = 12)
panel.pad(left = 8, right = 8)
text("hi").padded()                 // however much room the theme gives things
```

`padding(n)` insets every side; `pad(top =, right =, bottom =, left =)` insets the ones you name.
**Named arguments are what makes that readable**, which is why it's four defaulted parameters rather
than an `Insets` struct — `.pad(bottom = 12)` says what it does where `.inset(Insets(0, 0, 12, 0))`
makes a reader count commas. `padded()` takes its inset from the theme, the same argument as `panel`
against `background`.

**There is no `margin`, and there doesn't need to be one.** The difference between a padding and a
margin is only *which side of the background it falls on*, and the chain decides that:

```
text("hi").padding(8).background(red)    // padding — the ground covers the inset
text("hi").background(red).padding(8)    // margin  — the ground stops at the text
```

A second name for the same wrapper would be two ways to write one thing.

## Selecting text

**The toolkit selects; the application copies.** A selection is geometry and state, which is what a
view knows about; the clipboard belongs to the window system, which a view shouldn't reach. `sdl3`
has `clipboard_text` / `set_clipboard_text`, and a program binding ⌘C reads the selection signal and
hands the substring over.

```
selectable(line, sel)                    // drag to select, highlight painted behind the run
selected_text(line, sel.read())          // what to put on the clipboard
```

Two `Canvas` members do the work, **both with default bodies so no backend implements them**:
`text_at(s, x)` gives the byte offset nearest a pixel and `text_x(s, at)` the reverse. The default
walks characters and measures prefixes — which is why an offset is **never inside a multi-byte
character**. A hit test that divided a width by a byte count would split an `é`; there's a test on
exactly that.

**The selection is stateless in the view.** A `Selectable` holds no anchor: both ends are recomputed
every frame from where the press *began* and where the pointer is now. The canvas already knew the
first, because capturing the pointer for the slider meant recording it — so there is no first frame
of a drag to notice and no half-set state to leave behind. `press_origin()` is what made that
possible.

## Culling

A container asks `c.visible(child_rect)` before it paints a child and skips it if the answer is no.
The measure still happens — a column needs each child's height to place the one after it — and the
drawing, which is nearly all of the cost, does not.

This is what makes a long list affordable: a scrolling list of three hundred rows shows about ten,
and the other two hundred and ninety used to be painted, clipped away by the backend, and thrown
out after all the work of producing them. That was 2.8 ms a frame at 900×660, seventeen per cent of
a 60 Hz budget, spent entirely on discarded pixels.

**A culled child registers no hit region either**, which is a correctness claim rather than a
performance one: a row scrolled out of sight must not answer a click where it would have been.

`visible` defaults to `true`, so a backend that tracks no clip is correct and merely slower — the
right way round, since a backend that answered `false` by mistake would draw nothing at all.

## Animation

`Interaction` carries two `real`s rather than two flags, so a style is told how far *in* a hover is
and the arithmetic that picked a colour picks every colour on the way to it. Animation therefore
costs a style nothing and is not something it opts into.

The phase itself lives on the **canvas**, keyed by the rectangle — the same trick the hit list plays,
and it works for the same reason. `phases.sysl` is the store: a linear step toward a target at a rate
per second, interruptible by construction, sweeping whatever nothing asked about during a frame.

Linear rather than eased is a decision rather than a shortcut. An ease-in-out is a curve over a
*duration*, and a phase interrupted half way — which is exactly what a pointer crossing a button
twice produces — has no such duration any more. Moving at a rate is interruptible: wherever the value
has got to is a legitimate place to head back from.

**What it costs:** a view that *moves* loses its phase, because its key changed. For a button in a
static chrome that is never; for a row in a list being scrolled it is every frame, so a hover effect
on a scrolling row snaps rather than eases. That is the honest boundary of keying by geometry, and
the fix — were it ever worth one — is an identity the application supplies, which is the retained
element card `0168` spent its length refusing.

A backend with no clock answers the target outright, so a tree drawn through one is drawn settled.
That is what makes animation something a backend opts into rather than something every backend owes.

## The backend

`PlutoCanvas` draws a frame into a **PlutoVG** surface, and PlutoVG is here because of corners: SDL3
has no shape or path API at all, the triangle fan that would draw a rounded rectangle is not
antialiased, and a rounded *border* is an annulus, which is not convex and cannot be drawn that way
regardless. PlutoVG vendors FreeType's `ftgrays` rasterizer, carries its own C99, and needs nothing
installed — which is why the same backend serves a desktop and a phone, where cairo does not.

**It draws; it does not present.** What comes out is a surface of premultiplied ARGB32 pixels, and
getting those onto a screen is the application's: a streaming texture on the desktop and the same on
Android, since SDL3 is the presenter on both. That split is what keeps one backend serving two
platforms, and it is why this package asks for no window system of its own.

The arithmetic that can be *wrong* — the half-width inset a centred stroke needs to land inside the
rectangle layout settled on, the smaller radius that inset corner takes, the channel conversion, and
a line height read from a `descent` whose sign is a matter of convention — is lifted out of the
delegation and tested as ordinary functions, so none of it needs a font to be checked.

## Layout

- `sh/sysl/ui/view.sysl` — the `View` trait, the leaves, and the modifiers
- `sh/sysl/ui/theme.sysl` — `Theme`, `Role`, `Variant`, and the `Themed`/`Panel` views
- `sh/sysl/ui/style.sysl` — `Interaction`, `Reactive`, and `button`'s four variants built on them
- `sh/sysl/ui/widgets.sysl` — `switch`, `checkbox`, `radio`, `slider`, `progress` and `divider`
- `sh/sysl/ui/color.sysl` — `mix`, `lighten` and `darken`
- `sh/sysl/ui/phases.sysl` — where an animation lives, in a framework with nowhere to put it
- `sh/sysl/ui/signal.sysl` — `Signal[T]` and its weak dependent list
- `sh/sysl/ui/layout.sysl` — `Column` and `Row`, and the leftover a `Spacer` takes
- `sh/sysl/ui/grid.sysl` — positioning by fractions of the width
- `sh/sysl/ui/table.sysl` — a scrollable table, built entirely out of the files above it
- `sh/sysl/ui/tabs.sysl` — a tab bar, built entirely out of the files above it
- `sh/sysl/ui/overlay.sysl` — `render`, popovers and modals: the second paint pass
- `sh/sysl/ui/atom.sysl` — application state a module declares, and values derived from it
- `sh/sysl/ui/select.sysl` — selecting a run of text with the pointer
- `sh/sysl/ui/scroll.sysl` — where long-lived state lives, and the scrollbar
- `sh/sysl/ui/canvas.sysl` — the `Canvas` trait a frame is drawn through and measured against; a
  recorder that draws nothing and remembers everything, so the tree can be asserted with no window;
  and `Blind`, the smallest thing that satisfies the trait, which is there to say what a backend
  actually has to write
- `sh/sysl/ui/pluto.sysl` — the real backend: a frame drawn into a PlutoVG surface
- `sh/sysl/ui/input.sysl` — where a tap goes: the regions the paint pass collected
- `sh/sysl/ui/tests.sysl` — the findings, kept as tests rather than as prose
