# Porting *Code the Classics* from Python to Go — lessons learned

This document distils the **main lessons** from porting the *Code the Classics*
games (Raspberry Pi Press) from their original **Pygame Zero / Python** sources to
**Go** on [go-sdl3](https://github.com/Zyko0/go-sdl3). It is a cross-cutting
summary of the eight per-game write-ups that live in the individual port repos:

- **Volume 1**: Boing!, Cavern, Myriapod, Substitute Soccer
- **Volume 2**: Avenger, Eggzy, Kinetix, Leading Edge

Each of those repos contains its own `Python_and_Go_implementation_comparison.md`
with the game-specific details. This file collects the patterns that recurred
across all of them, with a pointer to the game that best illustrates each one.

The guiding principle in every port was **behavioural fidelity**: the Go code is a
faithful, often line-by-line translation of the game logic. It deviates only where
a language or library difference *forces* a different expression of the same idea,
or where a host-platform feature (game controllers, save folders) is out of scope.
As a rule of thumb, each single Python module (≈500–1,600 lines) became several
small files in one `package main`. The Go is larger, but **less so than it once
was**: these ports run on the [pgzgo](https://github.com/chrplr/pgzgo) harness
(over go-sdl3), which — like Pygame Zero on the Python side — hides the repetitive
framework plumbing: SDL init/teardown, the fixed-step game loop with an FPS cap,
window/renderer creation, the image cache and its blit/text/polygon/clip helpers,
the mixer wrapper, and the keyboard/gamepad snapshot. So that plumbing is **no
longer duplicated per game** — Boing, Cavern, Myriapod and Kinetix carry no
`assets.go`/`audio.go`/`text.go` at all, only a ~20–40 line `harness.go` of
`//go:embed` directives and type aliases (`type Assets = pgzgo.Screen`). The extra
volume that *does* remain is almost entirely **language-level** boilerplate Python
gets for free: explicit struct/interface declarations, `None`→pointer conversions,
enum blocks, per-type slice handling, and small integer-math/vector helpers. (A few
games still keep genuinely game-specific plumbing — a bespoke `audio.go` for the
looping crowd/engine/skid tracks pgzgo doesn't cover, or a PNG terrain-mask
decoder — but that is necessary logic, not boilerplate.)

---

## 1. Language paradigm: classes and inheritance

Pygame Zero games lean on classical inheritance (an `Actor` base with subclasses).
Go has neither classes nor inheritance, so three related patterns recur:

### 1.1 Inheritance → struct embedding + explicit forwarding

The actor tree (`Actor → CollideActor → GravityActor → Player/Enemy…`) becomes a
chain of **embedded structs**. Python's implicit `super().update(...)` becomes an
**explicit call to the embedded method** — e.g. `p.gravUpdate(g, !p.hurt)` stands
in for `super().update(not self.hurt)`. Seen in every port; clearest in the
platformers (Cavern, Eggzy).

### 1.2 The `self` back-reference (when the base must call an override)

Embedding is *not* inheritance: a method on an embedded base doesn't know which
concrete struct wraps it, so it can't dynamically dispatch to an overridden method.
Where Python relied on that (`CollideActor.get_rect` calling a subclass
`get_collidable_width`, or `track_piece.cars.remove(self)`), the Go port stores the
concrete value in a `self` **interface field**, wired at construction (`p.self = p`):

```go
type CollideActor struct {
    Actor
    self collidable   // back-reference to the concrete Player/Enemy
}
```

Used in Eggzy and Leading Edge. Notably **Cavern did *not* need it** — its base
`move` only touches its own position — so plain embedding sufficed. The lesson:
add the `self` interface only where a virtual call actually crosses the boundary.

### 1.3 Duck typing → small interfaces + type assertions

Where Python uses different objects interchangeably by just accessing common
attributes, Go needs an **interface capturing exactly the methods that role uses**,
and `isinstance(x, T)` becomes a **type assertion** `x.(*T)`:

| Python | Go |
|---|---|
| a pass target is a `Player` *or* a `Goal` | `type posTeam interface { Pos() Vec2; TeamID() int }` (Soccer) |
| `game.cars` holds `PlayerCar` and `CPUCar` | `type Car interface { … }` (Leading Edge) |
| `isinstance(target, Player)` | `if pl, ok := target.(*Player); ok { … }` |

Substitute Soccer is the showcase: its `posTeam`/`Marker` interfaces let a Player
and a Goal be treated uniformly — the static-typing counterpart to duck typing.

### 1.4 Polymorphic heterogeneous lists → typed slices in the same order

Python iterates one mixed list of everything (`for obj in self.fruits +
self.bolts + … : obj.update()`). Go can't hold "anything with `.Update()`" without
an interface, and each slice is already a concrete type, so the port **unrolls the
concatenation into typed loops, preserving the original order** — because update
*and* draw order is load-bearing (it decides what mutates first and what draws on
top). A genuine mixed collection only gets a `Drawable` interface when it must be
**sorted together** (e.g. Myriapod's depth sort, where `isinstance(obj, Explosion)`
becomes a type-assertion sort key).

---

## 2. Dynamic typing: `None`, enums, and optionals

### 2.1 `None` → nil pointers, sentinels, or value + companion boolean

- Object references (`self.owner = None / Player`) → **nil pointers** (`owner *Player`).
- An `int`/`float` field that is *either* a number *or* `None` can't hold nil, so
  it becomes a **value plus a `has*` boolean**: `lead float64` + `hasLead bool`
  (Soccer), `fastestLap float64` + `hasFastest bool` (Leading Edge).
- For grid/array cells, a **numeric sentinel** replaces `None`: `-1` for an empty
  brick (Kinetix), an empty myriapod cell (Myriapod), or "no trapped enemy" in an
  orb (Cavern).
- One Python attribute that held *either* a Player or an Enemy (`human.carrier`)
  had to be **split into two typed fields** in Go (Avenger).

### 2.2 Enums → `const … iota`

`class State(Enum)` / `IntEnum` → a typed `const … iota` block. A Python `list`
indexed by an enum becomes a fixed-size Go **array** indexed by the same constants
(Avenger's `Timers [4]int`). Every game with a state machine does this.

---

## 3. Vectors and math

### 3.1 Operator overloading → value-type structs with methods

`pygame.math.Vector2`/`Vector3` use overloaded `+ - *`; Go has none, so a small
`Vec2`/`Vec3` **value type** provides `Add`, `Sub`, `Mul/Scale`, `Dot`, `Length`,
etc. The AI-heavy games are the test: in Soccer, `*` between two vectors means
**dot product**, translated as `v0.Dot(v1)`.

Some ports skip a vector type entirely when motion is simple axis-aligned — Boing
and Avenger just keep **scalar `dx/dy` (or `VelocityX/Y`) pairs**, matching how the
Python already stored them.

### 3.2 Value semantics gives Python's explicit copies for free

`pygame`'s `Vector2` is a **reference type**, so Python must write `Vector2(self.home)`
to copy. Go structs are **value types**, so a plain assignment (`carOffset := offset`,
`b.dir = dir`) already copies. Several ports call this out explicitly — it's a place
where Go is quietly *safer* than the original.

### 3.3 Floor vs. truncating integer maths — the subtle one

Python's `//` and `%` **floor** (round toward −∞); Go's `/` and `%` **truncate**
(toward zero). They agree for non-negative operands but differ for negatives. Games
with negative coordinates or offsets provide explicit helpers and use them wherever
the original relied on floor behaviour:

```go
func floorDiv(a, b int) int { q := a / b; if a%b != 0 && (a<0) != (b<0) { q-- }; return q }
func pmod(a, b int) int      { m := a % b; if m < 0 { m += b }; return m }
```

Critical in Myriapod (segments at negative cells; `out_edge - in_edge` can be
negative — getting this wrong corrupts cell tracking), Soccer (facing rotation and
animation-frame index with `anim_frame == -1`), and Avenger (radar/terrain wrap).
Conversely, `int(float)` **truncates toward zero in both languages**, so it is
matched deliberately where Python used `int()` (and distinguished from the places
the original deliberately used `math.floor`, as in Leading Edge's two track lookups).

---

## 4. Idiom translations

| Python idiom | Go equivalent | Notes |
|---|---|---|
| `[x for x in xs if cond]` | in-place filter `out := xs[:0]; for … { if keep { out = append(out, v) } }` | per-type helper, or one **generic `filter[T]`** (Myriapod) |
| reverse `del` loop to remove items | same in-place slice filter | Boing |
| `min(xs, key=fn)` | explicit loop tracking the best candidate | Soccer |
| `sorted(xs, key=fn)` | `sort.SliceStable(xs, less)` | Soccer, Leading Edge, Myriapod |
| `getattr(images, name)` | build the string, look it up in a **texture cache** | universal |
| f-strings / `str(n)` | `fmt.Sprintf` / `strconv.Itoa` | universal |
| `hex(v)[2:]` / `int(s,16)` | tiny `hexDigit`/`parseHexDigit` helpers | Kinetix brick IDs |
| `try/except` around audio | nil checks / best-effort no-ops | universal |

Two idiom translations were genuinely clever, both in the games with the hardest
AI:

- **Myriapod's `rank`**: Python returns a **tuple of seven booleans** and lets
  `min(range(4), key=…)` compare them lexicographically. Go can't order tuples, but
  lexicographic comparison of a bool-tuple *is* integer comparison of those bits
  packed **most-significant-first** — so the port packs them into one `int` and does
  a manual `min` (strict `<` to keep the earliest on ties, matching Python's `min`).
- **Myriapod's `occupied` set** mixed 2- and 3-tuples in one Python `set`; Go needs
  a single key type, so both shapes unify into a `map[[3]int]bool` with a `-1`
  sentinel in the third slot. (A comparable array *is* a valid Go map key.)

And one the other way: Soccer's Python `cost()` returned a `(result, pos)` **tuple
purely to dodge a crash** (so `min` never compares two `Vector2`s on a tie). In Go
the cost is a plain `float64` and the min loop compares floats fine, so the tuple
**disappears** — a workaround the target language simply doesn't need.

---

## 5. Removing the module global

Python reaches a module-level `game` global from inside every actor method
(`game.player`, `game.play_sound(...)`, `game.orbs.append(...)`). The Go ports have
**no such global**; instead `g *Game` is **threaded as a parameter** through every
method that needs it. This is mechanical but pervasive (nearly every method
signature gains a `g *Game`), and it pays off twice: the data flow becomes explicit,
and the **initialization-order hazard disappears** — Python needs `if game is not
None` guards because actors are constructed *during* `Game` construction before the
global is assigned; in Go those actors simply receive the finished `g`, so the
nil-checks vanish (Eggzy, Cavern).

The **one deliberate exception** is a demo/attract-mode AI that outlives individual
`Game` objects: Kinetix keeps a single package-level `game` for `AIControls` to
read, mirroring the original rather than inventing a back-reference.

---

## 6. Framework: Pygame Zero → pgzgo (on go-sdl3)

Pygame Zero supplies a lot of implicit machinery. On the Go side that machinery now
lives in the **pgzgo harness**, which plays the same role Pygame Zero does: it owns
SDL init/teardown, the fixed-step loop, the window/renderer, an image cache with
drawing helpers, a sprite-font, a mixer wrapper, and a keyboard/gamepad snapshot. So
the mapping below is mostly **Pygame Zero ↔ pgzgo at parity** — not something each
game re-implements. (The earlier per-game write-ups describe these as hand-written
`assets.go`/`audio.go`/`input.go`/`text.go` against raw go-sdl3, which is how the
ports started; most of that code has since been deleted in favour of the harness.)

| Pygame Zero feature | pgzgo equivalent |
|---|---|
| global `screen` | `pgzgo.Screen` (aliased as the game's `Assets`) |
| `Actor("name")` auto-loads `images/name.png` | `Screen.Texture`/`Blit` — a lazily-populated texture cache |
| `screen.blit` centred / scaled / sub-tile | `Screen.BlitCentred` / `BlitScaled` / `BlitTile` |
| `screen.draw.text` sprite font | `Screen.DrawText` / `TextWidth` with a `pgzgo.Font` |
| `screen.draw.polygon`, `set_clip` | `Screen.FillPolygon`, `SetClip`/`ClearClip` |
| `keyboard.left`, `keyboard.space` | `app.Keyboard.Held(scancode)` snapshot (see §6.2) |
| `sounds.foo.play()` via `getattr` | `Audio.PlaySound(name, count)` picking a random variant |
| `music.play` / `set_volume` | `Audio.PlayMusic` / `StopMusic` |
| the `update()`/`draw()` loop | `app.Loop(update, draw)` — fixed-step, FPS-capped (see §6.3) |

What each game still writes for itself is either a thin type-alias `harness.go`
(`type Assets = pgzgo.Screen`, the `//go:embed` directives) plus **genuine game
logic**: the anchor model (§6.1), the just-pressed edge detection over pgzgo's
snapshot (§6.2), and, for three games, a bespoke `audio.go` (§6.5).

### 6.1 Anchors

Pygame Zero anchors are heterogeneous tuples (`("center","center")`,
`("center","bottom")`, `("center", 60)`). Go has no such union, so a tiny `Anchor`
struct tags each axis as *centre*, *bottom*, or an *absolute pixel offset*, and
`Anchor.offset(w,h)` reproduces Pygame's resolution rules — so the drawing code and
the collision code agree on where each sprite sits (e.g. a character's *feet*).

### 6.2 Input edge detection — reproduce it exactly

The trickiest input detail is "was this key *just* pressed?". Pygame Zero exposes
only the current state, so the originals keep a module-level latch. Two lessons:

- pgzgo supplies the held-key snapshot (`app.Keyboard.Held(sc)`); each game derives
  the rising edge over it — either a `keys`/`prevKeys` `keyJustPressed`, or a simple
  per-frame latch (`pressed := down && !wasDown; wasDown = down`).
- **Cavern is a cautionary tale**: its `space_pressed()` latch mutates on *every
  call*, so it must be called **at most once per frame**, and there's a real quirk
  that depends on it (during the recoil frames the not-hurt branch is skipped, so
  the latch isn't updated). Using the port's own generic edge-detector here would
  have been *wrong* — the port reproduces the single-call-per-frame latch verbatim.

### 6.3 Game loop and timing — match the original's model

pgzgo's `app.Loop(update, draw)` runs a **fixed-step, FPS-capped** loop for every
game (the v0.4.0 fixed-timestep loop also keeps WASM builds from running too fast on
high-refresh displays). Most games are **frame-count driven** (`timer` increments
once per update; animation frames are `timer // interval`), so they just supply the
two callbacks. **Leading Edge is different**: its physics need a fixed `dt`
independent of frame cadence, so on top of pgzgo's callback it keeps **its own
accumulator** (`accumulatedTime += dt; for accumulatedTime >= FixedTimestep {
game.Update(FixedTimestep) }`), including the subtlety that a freshly created `Game`
gets its first update on the same frame it's created.

### 6.4 Rendering specifics worth their own note

- **Pre-rendered surfaces → direct per-frame drawing.** Kinetix's Python caches
  bricks onto persistent `Surface`s and updates them incrementally; the Go port
  drops the cache and **draws each brick from the grid every frame** (the grid is
  the authoritative state anyway) — behaviourally identical and it sidesteps SDL
  render targets.
- **Filled polygons → triangulation.** Pygame fills arbitrary polygons; SDL draws
  only triangles. That triangulation (each convex quad as a vertex-coloured fan) now
  lives inside pgzgo's `Screen.FillPolygon`, so Leading Edge's pseudo-3D renderer
  just calls it — clipping likewise goes through `Screen.SetClip`.
- **Sprite scaling** → `Screen.BlitScaled` lets the renderer scale during blit via a
  destination rect, using nearest-neighbour to match pygame's transform.
- **Bitmask collision → alpha test.** Avenger's terrain uses `pygame.mask`; the Go
  port decodes `terrain.png` with the standard `image/png` and tests pixel alpha.
- **XML data → `encoding/xml` structs.** Eggzy's Tiled `.tmx`/`.tsx` maps move from
  ElementTree XPath walking to typed structs unmarshalled once, with helper
  functions replacing the XPath attribute predicates Go can't express.

### 6.5 Audio

For most games, pgzgo's `Audio` covers the sound model directly — `PlaySound(name,
count)` picks a random numbered variant, `PlayMusic` loops a track, and every call
is **best-effort** (a headless mixer just does nothing), mirroring the Python
`try/except`. Boing, Cavern, Myriapod, Eggzy and Kinetix carry no `audio.go` at all.

**Three games keep a bespoke `audio.go`** (Soccer, Avenger, Leading Edge) because
their sound goes beyond what the harness offers: a looping crowd ambience with a
start whistle (Soccer), a fading thrust loop (Avenger), and a speed-indexed engine
loop plus a grip-tracking skid loop (Leading Edge) — persistent `mixer.Track`s you
re-point at different audio. This is necessary game-specific logic, not the
plumbing pgzgo replaced. A couple of these accept a small documented
**simplification** — per-instance volume scaling of a few effects isn't reproduced
(loudness only, never gameplay).

---

## 7. Faithfulness discipline

The ports treat "faithful" strictly, which produced a few notable habits:

- **Preserve original quirks and even bugs, with a comment — don't "fix" them.**
  Eggzy reproduces a timer that the original never decrements and a `self.flame_image`
  typo (an assignment to the wrong attribute that means a sprite is never cleared),
  because changing either would change behaviour.
- **Watch the one porting-bug class: coordinate/anchor conventions.** Pygame's "add
  a camera offset to a world position, then blit at the anchor" is *not* the same as
  SDL's "blit at position minus the camera". Avenger shipped with the offset sign
  inverted — invisible at the start position, broken after the first respawn at a
  random X — and it was caught in review. This is the mistake most likely to recur
  when translating the drawing convention, alongside **negative-modulo** wrap maths.
- **Add a headless `-selftest` where practical.** Several ports (Cavern, Eggzy,
  Kinetix, Leading Edge) added a Go-only flag that steps the logic without a display
  — loading every level, running orbs/physics/AI, printing per-level counts — so the
  port can be verified in CI. On-screen visuals and audio still need a real display
  to confirm.

---

## 8. Intentional out-of-scope differences

A few Python features are deliberately dropped because they concern the host, not
the game: **game controllers / joystick** classes (the ports are keyboard-only;
where the AI drives a demo, that AI *is* fully ported); Python/Pygame-Zero
**version checks**; and **save-folder discovery** (Eggzy writes its ghost-replay
file to a `-replays` path, but keeps the on-disk **format identical** so files are
interchangeable). Debug switches (`DEBUG_SLOWMO`, etc.) are dropped as well.

---

## 9. The recurring differences at a glance

| Category | Python | Go | Reason |
|---|---|---|---|
| Inheritance | `Actor` subclasses | embedded structs + explicit forwarding | no classes |
| Virtual call from base | dynamic dispatch | `self` **interface** back-reference | embedding ≠ inheritance |
| Duck typing | shared attribute access | small **interfaces** + type assertions | static typing |
| Heterogeneous lists | one mixed list | typed slices in original order; `Drawable` only when sorted together | static typing |
| Optionals | `None` | nil pointers / value + `has*` bool / `-1` sentinel | no `None` |
| Enums | `Enum`/`IntEnum` | `const … iota`, arrays indexed by them | static typing |
| Vectors | `Vector2/3` + operators (`*` = dot) | value-type structs with methods | no operator overloading |
| Copies | explicit `Vector2(x)` | plain assignment (value semantics) | Go copies by default |
| Integer maths | floor `//`, `%` | `floorDiv`, `pmod`; `int()` truncation matched | Go truncates toward zero |
| Comprehensions | `[x for x…]` | in-place filters / generic `filter[T]` | idiom |
| `min/sorted(key=)` | closures | explicit loops / `sort.SliceStable`; bit-packed rank | no `key=`/tuple ordering |
| Dynamic names | `getattr(images, …)` | string keys into a texture cache | static typing |
| Module global | `game` | explicit `*Game` parameter (except demo AI) | avoids global coupling & init-order hazards |
| Framework | Pygame Zero | the **pgzgo** harness (assets, font, input snapshot, mixer, loop) — a harness↔harness swap, not per-game boilerplate | library swap |
| Rendering | surfaces / mask / polygons / XPath | direct draw / `image/png` alpha test / `encoding/xml` (polygon/clip now via pgzgo) | SDL & stdlib capabilities |

---

## 10. Bottom line

Across all eight games, the control flow, constants, physics, AI, and game rules are
**line-by-line equivalent** to the Python originals. Essentially every difference is
a *mechanical consequence* of three facts about Go — no classes/inheritance, no
operator overloading, and static typing (no `None`, no duck typing). The framework
plumbing, by contrast, is **not** a source of difference any more: the pgzgo harness
hides it on the Go side just as Pygame Zero does on the Python side, so what each
game writes is game logic, not glue. The handful of
genuinely interesting translations (Soccer's `posTeam`/`Marker` interfaces,
Myriapod's bit-packed `rank` and `[3]int` occupied-set, Kinetix's grid-direct
brick drawing) are each a case of finding the Go idiom that expresses a dynamically
typed Python construct *exactly*, rather than approximately.
