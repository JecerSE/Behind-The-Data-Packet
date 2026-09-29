# Behind the Data Packet — Architecture

A browser programming game in the spirit of *The Farmer Was Replaced*. The concept is the
same: you write code that drives a machine around a grid, and every action costs time.
But instead of fields of plants, you are **assembling chips**. The language is a
**subset of C++**, taught from the core outward: from the first level you work with
**instances** and write your **own functions**, and the game makes one difference from
Python impossible to miss: **C++ does not run top to bottom. The whole program is
compiled first, and nothing runs unless all of it compiles** (§5).

The language unlocks through a skill tree built from the C++ course notes (Ch 03–19).
Monkeytype is the reference for the web app's look and behaviour only: minimal,
keyboard-first, themeable, with a results screen after every run. There is no typing
test mechanic.

This file maps every part of the system, and every place we expect trouble, before
any code is written. Update it whenever a decision changes.

Author: Pursion (JecerSE).

---

## 1. Decisions already made

| Decision | Choice | Why |
|---|---|---|
| Platform | Web, static site, no backend | Shareable by link; GitHub Pages hosting |
| Language of the game code | TypeScript (strict), Vite, vanilla DOM | Monkeytype's core is vanilla TS; no framework to learn or fight |
| Player's language | A defined subset of C++20, interpreted in the browser | Follows the course; real compilation cannot pause per action |
| Editor | CodeMirror 6 + `@codemirror/lang-cpp` | Highlighting, undo, brackets and diagnostics for free |
| Execution | Compile to bytecode, run on our own VM, time-sliced on the main thread | Pause/step/speed control; an infinite loop can never freeze the tab |
| Curriculum | Vault course `02-Resources/CPP-Course`, Ch 03–19 (Ch 01–02 skipped) | The lessons; every chapter and section is shown in game as the skill tree (§9) |
| World | Chip assembly on a grid, replacing the farm (§8) | Requested |
| Core lesson | Compile first, then run; contrasted with Python's top-to-bottom execution (§5) | Requested |
| UI feel | Monkeytype as a reference only: centered, borderless, CSS-variable themes, command line, results screen. No typing mechanic | Requested |

Rejected: compiling real C++ to WebAssembly (clang-in-wasm). It is tens of megabytes,
it cannot yield after each `move()`, and its undefined behaviour is silent, which is the
opposite of what a teaching game wants.

---

## 2. System overview

```
 ┌──────────────────────────── browser tab ────────────────────────────┐
 │                                                                     │
 │  ui/  ── editor (CodeMirror) ──► source files (main.cpp, *.h)        │
 │   │                                   │                              │
 │   │                                   ▼                              │
 │   │    lang/  preprocess ─► lex ─► parse ─► sema ─► codegen          │
 │   │                                     (types, overloads,           │
 │   │                                      templates, classes)        │
 │   │                                   │ Program (bytecode)           │
 │   │                                   ▼                              │
 │   │    lang/vm  ◄── memory model (stack, heap, addresses, poison)    │
 │   │      │  calls native functions                                   │
 │   │      ▼                                                           │
 │   │    game/  world state, tick clock, economy, level goals          │
 │   │      │  events (moved, harvested, error, finished)               │
 │   ▼      ▼                                                           │
 │  ui/  board renderer · console · results screen · command line       │
 │                                                                      │
 │  storage: localStorage (code per level, settings, personal bests)    │
 └──────────────────────────────────────────────────────────────────────┘
```

**Dependency rule.** `lang/` imports nothing from `game/` or `ui/`. `game/` imports
`lang/` types only to register native functions. `ui/` may import everything. This keeps
the interpreter testable in plain Node with no DOM, and makes it reusable.

---

## 3. Repository layout

```
index.html
src/
  main.ts                 boot, routing between screens
  lang/
    source.ts             virtual files, #include resolution, spans for errors
    lexer.ts              tokens; splits `>>` when closing templates
    parser.ts             recursive descent; consults the symbol table (see T1)
    ast.ts
    sema/
      types.ts            type representation, sizeof/alignof (x86-64 layout)
      scope.ts            names, lookup, namespaces (std only)
      conversions.ts      promotions, conversions, narrowing checks
      overload.ts         candidate ranking, ambiguity errors
      templates.ts        on-demand instantiation, deduction
      concepts.ts         requires-expressions, satisfaction
      classes.ts          layout, vtables, special members, ctor/dtor order
      lambdas.ts          closure types, captures
    features.ts           chapter gates: which constructs are unlocked
    codegen.ts            AST → bytecode, scope-exit cleanup (destructors)
    vm/
      vm.ts               instruction loop, frames, budget, stepping
      memory.ts           addressable memory, stack, heap, poison tracking
      values.ts           int8..int64, float, double, pointers
    stdlib/               natively implemented library, see §7
    diagnostics.ts        compile and runtime errors with source spans
  game/
    world.ts              grid, tiles, entities; pure data
    api.ts                the `fab.h` natives and their tick costs
    clock.ts              cycles and ticks
    economy.ts            stock, part costs, skill-tree prices
    levels/               level definitions (goal, seed, starting unlocks)
    rng.ts                seeded PRNG so a run is reproducible
    replay.ts             seed + source = identical run
  skilltree/
    notes.json            generated from the vault notes (§9.1), not hand-edited
    tree.ts               nodes, edges, unlock state
  ui/
    compile.ts            the compile strip and error list (§5.1)
    notes.ts              notes / skill-tree view
    editor.ts             CodeMirror setup, lint bridge, optional vim mode
    board.ts              canvas renderer
    console.ts            std::cout output panel
    results.ts            post-run stats and graph
    commandline.ts        Esc palette: settings, themes, levels
    themes/               one CSS file of variables per theme
    settings.ts
    storage.ts
tests/
  lang/                   unit tests per stage
  conformance/            C++ programs + g++ expected output (see §10)
  game/
tools/
  build-notes.ts              vault notes → src/skilltree/notes.json
  extract-vault-examples.ts   pulls ```cpp blocks from the course notes
  gen-expected.sh             compiles each case with g++, saves stdout
ARCHITECTURE.md
```

---

## 4. The language pipeline

### 4.1 Stages

1. **Preprocess (lite).** `#include <iostream>` and friends map to built-in modules.
   `#include "thing.h"` pastes a file from the player's own tabs (needed from Ch 11,
   "Multiple Files", and Ch 17, "Class Across Multiple Files"). `#pragma once` and simple
   include guards are supported. Macros with arguments are **not**.
2. **Lex.** Standard C++ tokens, raw integer suffixes (`u`, `ll`, `'` digit separators),
   char and string literals with escapes.
3. **Parse.** Recursive descent to an AST. C++ cannot be parsed without knowing which
   names are types and templates, so the parser keeps a live symbol table (T1).
4. **Sema.** Resolve names, compute types, check conversions, pick overloads,
   instantiate templates, check concepts, lay out classes, build vtables, check chapter
   gates. Everything the real compiler would reject is reported here, **before** the run
   starts, as a compile error.
5. **Codegen.** Emit bytecode for a stack VM. Every scope exit (normal, `return`,
   `break`, `continue`) runs the destructors of live objects in reverse order.
6. **VM.** Execute with a cycle budget per animation frame.

### 4.2 Why bytecode, not a tree-walking interpreter

A tree walker using JS generators (`yield` on each action) is the quickest to write,
but it is slow on hot loops and makes "step one statement" awkward. A flat bytecode VM
with explicit frames can stop after **any** instruction, can be snapshotted for a
replay scrubber later, and runs about 10× faster. The cost is writing codegen. We accept
that cost from the start rather than rewriting later.

### 4.3 Time model

- Every VM instruction costs **1 cycle**.
- Every game action (`move`, `place`, `harvest`, …) costs a fixed number of cycles
  listed in `game/api.ts`, and advances the world by one **tick**.
- The world only changes on ticks, so plain computation is cheap and actions are
  expensive, as in *The Farmer Was Replaced*.
- A run ends when `main` returns, the level goal is met, a runtime error happens, or the
  cycle cap for the level is hit.
- The score is ticks and cycles to the goal. Lower is better.

### 4.4 Chapter gates

`features.ts` lists the language constructs each chapter unlocks. Using a locked construct
is a compile error that names the chapter:

```
error: pointers are locked — unlock Ch 08 "Pointers" to use `int*`
```

This is the game's version of the unlock tree: the upgrades are parts of the language
itself.

---

## 5. The core lesson: compiled, not read top to bottom

*The Farmer Was Replaced* uses Python, which executes a file **as it reads it**, top to
bottom. A typo on line 40 is only found after lines 1–39 have already moved the drone.
C++ works differently: the **whole program is compiled first**, every file and every
line, and nothing runs unless all of it compiles. Then execution starts at `main()`,
wherever `main` sits in the file. The game makes this difference visible and plays it
for puzzles.

| Situation | Python (*Farmer*) | C++ (this game) |
|---|---|---|
| Error on the last line | Lines before it run, then it crashes | **Nothing runs.** The board stays frozen and the compile panel lists every error |
| Code outside any function (`arm.move(...)` at file scope) | Runs, in order | Compile error: only declarations live there |
| Where execution starts | Line 1 | `main()` |
| Calling a function defined further down | Fine, if it is defined by the time the call runs | Compile error unless it is declared above (a prototype) |
| Wrong argument type | Error when that line is reached | Compile error before anything runs |
| Function declared but never defined | Error when called | Link error, before anything runs |

### 5.1 How the game shows it

- **Run is two visible phases.** *Compile* (preprocess → parse → check → link, a thin strip
  where each stage lights up) and then *Run*. A failed compile shows the assembly line
  stopped: the Arm never moves.
- **All errors at once.** The compiler reports every error it finds (capped at 20),
  not only the first, to reinforce that the whole program was read.
- **Python ghost** (tutorial levels only). A faded replay shows what top-to-bottom
  execution would have done with the same broken code, next to C++ refusing to start.
  The ghost is authored per tutorial level, not computed, because "run broken C++ top to
  bottom" has no general meaning.

### 5.2 Instances and your own functions from the core

The player works with **instances** and writes **their own functions** from the first
level, before any chapter is unlocked. The always-unlocked **core kit** is:

- `#include "fab.h"`, `int main()`, `return`
- declaring an instance of an engine class (`Arm arm;`) and calling its member
  functions with `.`
- `enum class` values from `fab.h` (`Dir::East`, `Part::Wafer`)
- defining your own `void` functions, and passing the arm to them as `Arm&`

```cpp
#include "fab.h"

void lay_row(Arm& arm) {        // your own function
    arm.place(Part::Silicon);
    arm.move(Dir::East);
    arm.place(Part::Silicon);
}

int main() {                    // execution starts here, not at line 1
    Arm arm;                    // an instance
    lay_row(arm);
    arm.move(Dir::North);
    lay_row(arm);
}
```

`Arm&` is the one reference form allowed before Ch 09. `Arm` deletes its copy
constructor (there is one physical arm), so `void lay_row(Arm arm)` is a compile error
whose message says to use `Arm&`. Ch 09 later explains what that `&` meant all along.
Defining your **own** classes stays locked until Ch 17.

---

## 6. Memory model

The course spends real time on memory (Ch 07, 08, 09, 12, 14, 17, 19). If the game
fakes memory, those lessons become untrue. So the VM simulates it.

- **One flat address space** in a `DataView` over an `ArrayBuffer`. Stack grows down from
  the top, heap grows up, globals sit at the bottom. Addresses print like real ones
  (`0x7ffd…` for stack, `0x55…` for heap).
- **Exact integer semantics.** `int` is 32-bit two's complement, `unsigned` wraps,
  `long long` uses `BigInt`, `char` is signed 8-bit and promotes to `int` in arithmetic.
  This is required for Ch 03 (unsigned countdown wraparound) and Ch 04 (`char + char`).
  Signed overflow is undefined behaviour in C++; the game reports it (see below).
- **Layout matches x86-64 g++.** `sizeof`, alignment and padding follow the real ABI;
  a class with virtual functions gains an 8-byte vptr. Ch 17 "Size of objects" and
  Ch 19 "Size of polymorphic objects" print real numbers and must match g++.
- **Sanitizer mode, always on.** Real C++ is silent about undefined behaviour. The game
  is not. The VM keeps shadow metadata for every allocation and reports:
  - out-of-bounds array access (Ch 07)
  - use after `delete`, double `delete`, `delete` vs `delete[]` mismatch (Ch 08)
  - leaks at the end of the run, with the allocating line (Ch 08)
  - reading an uninitialized variable (poison bits)
  - returning a reference or pointer to a local (Ch 12), dangling lambda captures (Ch 14)
  - signed overflow, division by zero, null dereference

  Each error stops the run with the line, a short explanation, and a link to the
  chapter section that covers it. This is a **deliberate** difference from real C++ and
  the conformance suite marks such cases (§10).
- **`new` failure** (Ch 08, "When new Fails"). The heap has a per-level size. `new`
  beyond it ends the program with `terminate called after throwing an instance of
  'std::bad_alloc'`, as real C++ does without a `try`. `new (std::nothrow)` returns
  `nullptr`.

---

## 7. Standard library surface

Implemented natively in TypeScript, not written in C++. Counted from the `#include`s in
the course notes:

| Tier | Headers | Needed by |
|---|---|---|
| Core | `<iostream>` (cout, cin, endl, `>>` / `getline`), `<string>`, `<cstring>`, `<cctype>`, `<cmath>`, `<limits>`, `<iomanip>` | Ch 03–10 |
| Containers | `<vector>`, `<array>` | Used in 24 examples, heavily in Ch 17–19 |
| Functional | `<functional>` (`std::function`), `<utility>` | Ch 14 |
| Types | `<concepts>`, `<type_traits>` | Ch 15–16 |
| Ownership | `<memory>` (`unique_ptr`, `make_unique`) | Ch 19 collections |
| Later / maybe | `<map>`, `<algorithm>`, `<optional>`, `<sstream>`, `<tuple>`, `<bitset>`, `<typeinfo>` | 1–3 examples each |

`<vector>` and `<memory>` are the expensive ones: they are templates whose behaviour
(copying, destructors running on elements) must match the language rules. They get
dedicated tests.

Player input: `std::cin` reads from a per-level input tape shown in the UI, so Ch 10's
`>>`-then-`getline` trap reproduces exactly.

---

## 8. The game world: a chip fab

The concept is *The Farmer Was Replaced* with the farm replaced by a **fab floor**:
instead of planting and harvesting crops, you place parts in sockets, let them cure, and
harvest finished components. The name reads as "the hardware behind every data packet":
by the end, packets flow through chips you designed yourself.

| *Farmer* | Behind the Data Packet |
|---|---|
| Farm grid | Fab floor: a grid of sockets |
| Drone | The **Arm**, a pick-and-place head |
| Plant a seed | `arm.place(Part::…)` |
| Crop grows over time | Part cures over ticks |
| Water / fertilizer | `arm.anneal()`: faster curing, costs heat |
| Harvest | `arm.harvest()` into stock |
| Crop tiers (grass → carrots…) | Part tiers: Silicon → Wafer → Resistor / Capacitor → Transistor → Gate → Die |
| Pumpkins merge when adjacent | Adjacent finished Dies fuse into one larger multi-core die |
| Sunflowers (power) | Power cells: harvesting the strongest one speeds the Arm |
| Cactus (sort to harvest) | **Binning**: chips come out with speed grades and must be sorted in the grid |
| Dinosaur (snake) | **Bus routing**: the trace grows behind the Arm and cannot cross itself |
| Maze | **Trace routing** through a PCB maze to a target pad |
| Multiple drones | Extra Arms, spawned with a lambda (Ch 14) |

Higher tiers cost lower tiers to place, as crops cost other crops in *Farmer*. Stock is
also the currency for the skill tree (§9).

### 8.1 The engine header

```cpp
// fab.h — what the player sees
enum class Dir  { North, East, South, West };
enum class Part { None, Silicon, Wafer, Resistor, Capacitor, Transistor, Gate, Die };

class Arm {
public:
    Arm();
    Arm(const Arm&) = delete;       // one physical arm
    void move(Dir d);               // 1 tick
    Part scan() const;              // 1 tick, what is in this socket
    bool can_harvest() const;       // free
    void place(Part p);             // 1 tick, spends stock
    void harvest();                 // 1 tick
    void anneal();                  // 1 tick, spends heat
    int  x() const;
    int  y() const;
};

int       floor_size();
long long stock(Part p);
```

### 8.2 Your own chips (Ch 17–19)

From Ch 17 the player designs chips as classes. From Ch 18–19 they derive from the
engine's abstract `Chip`, and the engine clocks **their** overrides through a real
vtable. Data packets then flow across the floor through the player's designs:

```cpp
class Adder : public Chip {
public:
    void on_clock(Bus& in, Bus& out) override;
};
```

---

## 9. Skill tree and lesson map

### 9.1 The notes are the skill tree

Every chapter note (Ch 03–19) and every section inside it is shown in the game. Ch 01–02
are skipped; the core kit and tutorial (§5) cover the first program.

- **Now:** a *Notes* view shows all chapters and all sections, everything readable, no
  locks.
- **Later:** the same content becomes the skill tree. Each chapter is a node, its
  sections are sub-steps inside it, and the edges come from the index's reading order:

```
03 ─► 04 ─► 05, 06, 07 (any order)
08 Pointers ──► 09 References ──► 11 Functions ──► 12 Out of functions
     │                                  ├──► 13 Overloading ──► 15 Templates ──► 16 Concepts
     │                                  └──► 14 Lambdas
     └──────────────────────► 17 Classes ──► 18 Inheritance ──► 19 Polymorphism
```

  Unlocking a chapter costs stock and turns on its language features in the compiler
  (`features.ts`). Reading a section and passing its self-check gives a small reward.

**Build step.** `tools/build-notes.ts` converts the vault markdown to
`src/skilltree/notes.json` at build time, so the game never reads the vault at runtime:

| Obsidian feature | Becomes |
|---|---|
| Frontmatter (`chapter`, `timestamp`) | Node id, "video at h:mm:ss" label |
| `##` sections | Sub-nodes |
| ```` ```cpp ```` blocks | Highlighted code, with an "open in editor" button |
| ASCII diagrams | Preformatted blocks, kept as is |
| `[[Ch 08 — Pointers#Dangling Pointers\|…]]` | In-game link to that node and section |
| `Question :: Answer` spaced-repetition lines | Self-check cards on that section |
| `### Self-check` | The chapter's quiz |

Compile and runtime errors link into the same nodes, e.g. a dangling pointer error opens
Ch 08 › Dangling Pointers.

### 9.2 Lesson map

| Ch | Language unlocked | Game unlock | The trap, as a level |
|---|---|---|---|
| Core | The core kit (§5.2) | Arm, `move`, `place`, `harvest`, Silicon | Compile-first tutorials: an error on the last line stops everything; a statement outside `main`; a function called before it is declared; `Arm` passed by value |
| 03 | variables, integer/float/bool/char types, `auto`, brace init | `stock()`, `x()` / `y()` | Countdown with `unsigned` never ends: wraparound |
| 04 | arithmetic, precedence, `++`/`--`, compound ops, relational, logical, `<iomanip>`, `<limits>`, `<cmath>` | yield math, `anneal` timing | `char + char` prints a number, not a letter |
| 05 | `if` / `else if` / `switch` / `?:` | `scan()`, mixed part tiers | `=` vs `==`; missing `break` falls through |
| 06 | `for`, `while`, `do while`, `break`, `continue` | full-floor sweeps, curing timers, trace maze | `continue` in a `while` skips the increment → cycle cap |
| 07 | arrays, `sizeof`, char arrays, bounds | binning (sort chips by grade), die fusion | Out of bounds; `sizeof` of an array inside a function |
| 08 | pointers, `new` / `delete`, `nullptr`, dynamic arrays | heap budget; leaks cost stock; bus routing | Dangling pointer; `delete` vs `delete[]` |
| 09 | references, `const&` | explains the core kit's `Arm&` | `r = b` assigns, does not rebind |
| 10 | C-strings, `<cstring>`, `<cctype>`, `std::string`, `getline` | serial numbers and part labels; an input tape of orders | `>>` then `getline` reads an empty line |
| 11 | declarations vs definitions, multiple files, pass by value / pointer / reference | editor tabs (`.h` / `.cpp`) | Declared but not defined → link error (compile-first again) |
| 12 | output parameters, return by value | — | Returning a reference to a local |
| 13 | overloading | `place` overloads for part kinds | Ambiguous call from equal-rank conversions |
| 14 | lambdas, captures, `std::function` | extra Arms: `spawn_arm(lambda)` | `[&]` capture outliving its scope |
| 15 | function templates, deduction, explicit args, specialization | generic helpers over part types | Template definition not visible where used |
| 16 | concepts, `requires` | constrained helpers (`Placeable`) | `requires` clause vs `requires` expression |
| 17 | classes, ctors, dtors, `this`, `struct`, sizeof objects | your own chip classes | Members initialise in declaration order |
| 18 | inheritance, access, base ctors, copy ctors | derive from the engine's `Chip` | Derived copy ctor forgets the base |
| 19 | `virtual`, `override`, `final`, abstract classes, `dynamic_cast`, virtual dtor | the engine clocks your chips; packets flow through them | Missing virtual dtor; slicing on pass-by-value |

Each trap level is built so the trap shows up as a visible failure on the floor, with the
chapter section linked from the error.

---

## 10. Testing

1. **Unit tests** (Vitest) per pipeline stage.
2. **Conformance against g++.** The course notes contain 247 ```` ```cpp ```` blocks, about 100 of
   them complete programs with `main()`. `tools/extract-vault-examples.ts` copies them into
   `tests/conformance/cases/`. `tools/gen-expected.sh` compiles each with
   `g++ -std=c++20` and saves stdout to `expected/`. The test runs each case through our
   interpreter and diffs the output. Cases that print addresses, or that exist to show
   undefined behaviour, are tagged `// dp:expect-runtime-error <kind>` or
   `// dp:skip-output` instead.

   This is the main defence against the interpreter quietly teaching the wrong C++.
   Coverage of this suite is the progress bar for each milestone.
3. **Game tests.** Levels run headless from a seed plus a reference solution and must
   reach the goal within a fixed tick count.

The vault is only read by the extraction tool; the game never depends on it at runtime.

---

## 11. Interface (monkeytype as the reference)

- **One centered column.** Board on top, editor below, a thin console line under that.
  No sidebars. Hairline separators and soft depth; no hard borders or offset shadows.
- **Themes** as CSS variables with monkeytype's names, so their palettes port directly:
  `--bg-color --main-color --caret-color --sub-color --sub-alt-color --text-color
  --error-color`. Default is a soft dark theme.
- **Keyboard first.**
  - `Ctrl+Enter` run · `Tab` then `Enter` restart run (monkeytype's restart) ·
    `Ctrl+.` step one statement · `Esc` command line
  - The command line (`Esc`, fuzzy search) is where every setting, theme and level lives,
    as in monkeytype.
- **Focus mode.** While code runs, everything except the board and the current line fades
  out.
- **Notes and skill tree** open from the top bar or the command line, in the same single
  column.
- **Results screen.** Ticks, cycles, actions, code size, heap peak, leaks, and a graph of
  stock over ticks (the counterpart of monkeytype's wpm graph). Personal best per
  level, stored locally.
- **Optional vim keys** through `@replit/codemirror-vim`.

---

## 12. Persistence

localStorage only, versioned (`dp.v1.*`), every read wrapped in try/catch:
code per level, unlocks, currency, settings, theme, personal bests. Export/import of a
save as JSON from the command line. No accounts.

---

## 13. Troubles register

The places we expect this project to hurt, and what we do about each.

| # | Trouble | Why it is hard | Mitigation |
|---|---|---|---|
| T1 | **Parsing C++ needs semantic information** | `a < b > c`, `a * b;`, `T(x);` parse differently depending on whether names are types/templates | Parser consults the live symbol table (the C "typedef-name" technique). The subset requires declaration before use, which C++ already does |
| T2 | **`>>` closing two templates** | Lexed as shift | Lexer emits `>` `>` inside template argument lists |
| T3 | **Most vexing parse** | `Widget w();` declares a function | Parse it as C++ does and add a warning explaining it — a free lesson |
| T4 | **Overload resolution** | Real ranking rules are pages long | Implement the three ranks (exact, promotion, conversion) plus user-defined conversion via constructors; ambiguity when best ranks tie. Enough for Ch 13's trap; tested against g++ |
| T5 | **Templates and concepts** | Deduction, instantiation, specialization, satisfaction | Instantiate on demand during sema; memoize by argument list; errors show the instantiation chain. Concepts evaluate `requires` expressions by trying substitution. Class templates are only supported for `std::vector` / `std::array` / `unique_ptr` (built-in) until later |
| T6 | **Object lifetime** | Ctor/dtor order, temporaries, copies, destructors on every exit path | Codegen emits cleanup blocks per scope; temporaries destroyed at end of full-expression; a lifetime test suite from Ch 17–18 examples |
| T7 | **Virtual dispatch and slicing** | vptr, override, `final`, `dynamic_cast`, virtual dtors | Real vtables in simulated memory; RTTI record per class for `dynamic_cast`; slicing happens naturally because copies copy only the base layout |
| T8 | **Memory and UB** | Silent in real C++ | Shadow metadata + sanitizer checks (§6). Documented as intended divergence |
| T9 | **Integer semantics in JS** | JS numbers are doubles | `Math.imul`, `\|0`, `>>>0` for 32-bit; `BigInt` for 64-bit; tests for every wraparound case in Ch 03–04 |
| T10 | **Infinite loops** | Player code may never yield | VM runs a cycle budget per frame; level cycle cap ends the run with a clear message |
| T11 | **Scope creep** | Seventeen chapters of C++ are most of the language | The subset is fenced in §14. Anything not listed is a compile error "not supported", never undefined behaviour in our interpreter |
| T12 | **Error messages** | g++ messages are hostile | Short messages with the source span and one sentence of advice; the chapter link for gated or trap errors |
| T13 | **Library fidelity** | `std::vector` copying and growth must match | Native implementations follow the standard semantics (copies call copy ctors, `push_back` may reallocate and invalidate pointers — reported as UB) |
| T14 | **Performance** | Interpreted C++ in JS | Bytecode VM, typed arrays for memory, no allocation in the hot loop; target 20M cycles/s |
| T15 | **Lesson accuracy** | The game could teach wrong C++ | Conformance suite against g++ (§10) gates every milestone |
| T16 | **Converting Obsidian notes** | Wikilinks with anchors, spaced-repetition lines, ASCII diagrams, frontmatter | One build tool with a test per construct (§9.1); unknown syntax fails the build instead of rendering wrong |
| T17 | **`Arm&` before Ch 09** | The core kit uses a reference before references are taught | Allowed only as a parameter of type `Arm&`; any other `&` stays locked. Ch 09 is written to pay this off |
| T18 | **Teaching compile-first without frustration** | "Nothing ran" can feel like the game is broken | Compile strip shows which stage failed; every error is listed with its line; the tutorial's Python ghost shows the contrast |

---

## 14. Supported subset (the fence)

**In:** everything in the lesson map (§9); namespaces `std` only; `using namespace std;`
and `using std::x;`; `enum` and `enum class`; `const`, `constexpr` (evaluated as `const`);
`static` locals and members; `struct` / `class`; operator overloading for `<<` on
`ostream` and comparison operators (needed by examples in Ch 17–19).

**Out, until decided otherwise:** exceptions (`try`/`catch`/`throw`) beyond the
`bad_alloc` termination in §6; user-defined class templates; macros with arguments;
multiple and virtual inheritance; unions and bit fields; threads; `goto`; user-defined
literals; coroutines; modules.

---

## 15. Milestones

| M | Scope | Done when |
|---|---|---|
| M0 | Vite + TS skeleton, CodeMirror editor, canvas board, run/stop, VM running `arm.move()` in a loop | A hard-coded level is playable end to end |
| M1 | Core kit and compile-first tutorials, Ch 03–06, `cout`, tick model, compile strip, results screen, Notes view with every chapter, one theme | First playable; Ch 03–06 conformance cases pass |
| M2 | Ch 07–10: memory model, arrays, pointers, heap, references, strings, sanitizer. Notes become the skill tree with unlock costs | Ch 07–10 cases pass; every UB trap level reports correctly |
| M3 | Ch 11–14: functions, multiple files, overloading, lambdas, `std::function` | Ch 11–14 cases pass |
| M4 | Ch 15–16: function templates, concepts | Ch 15–16 cases pass |
| M5 | Ch 17–19: classes, inheritance, polymorphism, `vector`, `unique_ptr`, the OO API | Ch 17–19 cases pass; `Chip` levels playable |
| M6 | Themes, command line, personal bests, save export, GitHub Pages deploy | Public build |

M4 and M5 are the largest by far. M1 should arrive early so the game is fun before the
language is complete.

---

## 16. Open questions

1. **Publishing the notes.** The vault repo is private; bundling the notes into this
   public repo and site makes them public, including the course video timestamps.
2. **Exceptions.** Out for now; the course notes use `<stdexcept>` in two examples.

---

## 17. Conventions

- Commits are authored by Pursion only, with no co-author trailers.
- TypeScript `strict`, no `any` in `lang/`.
- Every language feature lands with its conformance cases in the same commit.
