# Behind the Data Packet — Architecture

A browser programming game in the spirit of *The Farmer Was Replaced*: you write code
that drives an agent around a grid, and every action costs time. Here the language is a
**subset of C++**, it is unlocked chapter by chapter following the C++ course notes
(Ch 03–19), and the interface feels like **monkeytype**: minimal, keyboard-first,
themeable, with a results screen after every run.

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
| Curriculum | Vault course `02-Resources/CPP-Course`, Ch 03–19 | The 19 lessons |
| UI feel | Monkeytype: centered, borderless, CSS-variable themes, command line, results screen | Requested |

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
 │   │      │  events (moved, delivered, error, finished)               │
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
    stdlib/               natively implemented library, see §6
    diagnostics.ts        compile and runtime errors with source spans
  game/
    world.ts              grid, tiles, entities; pure data
    api.ts                the `network.h` natives and their tick costs
    clock.ts              cycles and ticks
    economy.ts            currency, upgrade tree, unlocks
    levels/               level definitions (goal, seed, starting unlocks)
    rng.ts                seeded PRNG so a run is reproducible
    replay.ts             seed + source = identical run
  ui/
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
  conformance/            C++ programs + g++ expected output (see §9)
  game/
tools/
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
- Every game action (`move`, `read`, `deliver`, …) costs a fixed number of cycles
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

## 5. Memory model

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
  the conformance suite marks such cases (§9).
- **`new` failure** (Ch 08, "When new Fails"). The heap has a per-level size. `new`
  beyond it ends the program with `terminate called after throwing an instance of
  'std::bad_alloc'`, as real C++ does without a `try`. `new (std::nothrow)` returns
  `nullptr`.

---

## 6. Standard library surface

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

## 7. The game world

> **Proposal — rename freely.** The mechanics are the *Farmer* loop mapped onto
> networking: sources "grow" packets, you collect, process and deliver them, and the
> currency buys upgrades.

- **Grid** of tiles. Tiles hold nodes: **Source** (produces a packet every N ticks,
  like a crop growing), **Router**, **Sink** (accepts packets and pays out bandwidth),
  **Corrupt** (must be repaired before it produces).
- **The courier** is the agent the player's code controls, like the drone.
- **Packets** have a header (destination, type, checksum) and a string payload, which
  gives Ch 10 (strings) and Ch 04 (arithmetic, checksums) real work.
- **Currency: bandwidth.** Spent in an upgrade tree: chapter unlocks, bigger grid, more
  heap, cheaper actions, more couriers later.

The player's API is one header, `#include "network.h"`:

```cpp
enum class Dir { North, East, South, West };
enum class Tile { Empty, Source, Router, Sink, Corrupt };

void   move(Dir d);         // 1 tick
Tile   scan();              // 1 tick
bool   has_packet();        // free
Packet read();              // 1 tick, takes the packet at this tile
void   deliver(Packet p);   // 1 tick, only on a Sink
void   repair();            // 3 ticks
int    get_x(); int get_y(); int world_size();
```

From Ch 17 onward the API itself becomes object-oriented: `Packet` becomes a class you
can extend, and from Ch 18–19 the engine calls **your** overrides. For example you
derive from an abstract `Handler` with `virtual void on_packet(Packet&) = 0;` and the
world dispatches through your vtable.

---

## 8. Lesson map

The notes cover Ch 03–19. Ch 01–02 of the video course have no notes in the vault;
they are assumed to be setup and a first program, and become the tutorial level.

| Ch | Language unlocked | Game unlock | The trap, as a level |
|---|---|---|---|
| 01–02 | `int main()`, calls, `;`, `std::cout <<` | `move`, the grid | — (tutorial) |
| 03 | variables, integer/float/bool/char types, `auto`, brace init | counters, `get_x/get_y` | Countdown with `unsigned` never ends: wraparound |
| 04 | arithmetic, precedence, `++`/`--`, compound ops, relational, logical, `<iomanip>`, `<limits>`, `<cmath>` | checksums on packet headers | `char + char` prints a number, not a letter |
| 05 | `if` / `else if` / `switch` / `?:` | `scan()` and tile types | `=` vs `==`; missing `break` falls through |
| 06 | `for`, `while`, `do while`, `break`, `continue` | full-grid sweeps; Source growth timers | `continue` in a `while` skips the increment → cycle cap |
| 07 | arrays, `sizeof`, char arrays, bounds | a routing table of the grid | Out of bounds; `sizeof` of an array inside a function |
| 08 | pointers, `new` / `delete`, `nullptr`, dynamic arrays | heap budget as a resource; leaks cost bandwidth | Dangling pointer; `delete` vs `delete[]` |
| 09 | references, `const&` | modify packets in place | `r = b` assigns, does not rebind |
| 10 | C-strings, `<cstring>`, `<cctype>`, `std::string`, `getline` | packet payloads, input tape | `>>` then `getline` reads an empty line |
| 11 | functions, declarations vs definitions, multiple files, pass by value/pointer/reference | editor tabs (`.h` / `.cpp`) | Missing definition → link error |
| 12 | output parameters, return by value | — | Returning a reference to a local |
| 13 | overloading | `deliver` overloads for packet kinds | Ambiguous call from equal-rank conversions |
| 14 | lambdas, captures, `std::function` | callbacks: `on_tick([&]{...})` | Reference capture outliving its scope |
| 15 | function templates, deduction, explicit args, specialization | generic helpers over packet kinds | Template definition not visible where used |
| 16 | concepts, `requires` | constrained helpers | `requires` clause vs `requires` expression |
| 17 | classes, ctors, dtors, `this`, `struct`, sizeof objects | your own `Packet` / `Route` classes | Members initialise in declaration order |
| 18 | inheritance, access, base ctors, copy ctors | extend engine node types | Derived copy ctor forgets the base |
| 19 | `virtual`, `override`, `final`, abstract classes, `dynamic_cast`, virtual dtor | engine calls your `Handler` overrides | Missing virtual dtor; slicing on pass-by-value |

Each trap level is built so the trap shows up as a visible failure in the world, with
the chapter section linked from the error.

---

## 9. Testing

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

## 10. Interface (monkeytype-inspired)

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
- **Results screen.** Ticks, cycles, actions, code size, heap peak, leaks, and a graph of
  bandwidth over ticks (the counterpart of monkeytype's wpm graph). Personal best per
  level, stored locally.
- **Optional vim keys** through `@replit/codemirror-vim`.

---

## 11. Persistence

localStorage only, versioned (`dp.v1.*`), every read wrapped in try/catch:
code per level, unlocks, currency, settings, theme, personal bests. Export/import of a
save as JSON from the command line. No accounts.

---

## 12. Troubles register

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
| T8 | **Memory and UB** | Silent in real C++ | Shadow metadata + sanitizer checks (§5). Documented as intended divergence |
| T9 | **Integer semantics in JS** | JS numbers are doubles | `Math.imul`, `\|0`, `>>>0` for 32-bit; `BigInt` for 64-bit; tests for every wraparound case in Ch 03–04 |
| T10 | **Infinite loops** | Player code may never yield | VM runs a cycle budget per frame; level cycle cap ends the run with a clear message |
| T11 | **Scope creep** | Nineteen chapters of C++ is most of the language | The subset is fenced in §13. Anything not listed is a compile error "not supported", never undefined behaviour in our interpreter |
| T12 | **Error messages** | g++ messages are hostile | Short messages with the source span and one sentence of advice; the chapter link for gated or trap errors |
| T13 | **Library fidelity** | `std::vector` copying and growth must match | Native implementations follow the standard semantics (copies call copy ctors, `push_back` may reallocate and invalidate pointers — reported as UB) |
| T14 | **Performance** | Interpreted C++ in JS | Bytecode VM, typed arrays for memory, no allocation in the hot loop; target 20M cycles/s |
| T15 | **Lesson accuracy** | The game could teach wrong C++ | Conformance suite against g++ (§9) gates every milestone |

---

## 13. Supported subset (the fence)

**In:** everything in the lesson map (§8); namespaces `std` only; `using namespace std;`
and `using std::x;`; `enum` and `enum class`; `const`, `constexpr` (evaluated as `const`);
`static` locals and members; `struct` / `class`; operator overloading for `<<` on
`ostream` and comparison operators (needed by examples in Ch 17–19).

**Out, until decided otherwise:** exceptions (`try`/`catch`/`throw`) beyond the
`bad_alloc` termination in §5; user-defined class templates; macros with arguments;
multiple and virtual inheritance; unions and bit fields; threads; `goto`; user-defined
literals; coroutines; modules.

---

## 14. Milestones

| M | Scope | Done when |
|---|---|---|
| M0 | Vite + TS skeleton, CodeMirror editor, canvas board, run/stop, VM running `move()` in a loop | A hard-coded level is playable end to end |
| M1 | Ch 01–06: types, operators, control flow, loops, `cout`, tick model, results screen, one theme | First playable; Ch 03–06 conformance cases pass |
| M2 | Ch 07–10: memory model, arrays, pointers, heap, references, strings, sanitizer | Ch 07–10 cases pass; every UB trap level reports correctly |
| M3 | Ch 11–14: functions, multiple files, overloading, lambdas, `std::function` | Ch 11–14 cases pass |
| M4 | Ch 15–16: function templates, concepts | Ch 15–16 cases pass |
| M5 | Ch 17–19: classes, inheritance, polymorphism, `vector`, `unique_ptr`, the OO API | Ch 17–19 cases pass; `Handler` levels playable |
| M6 | Themes, command line, personal bests, save export, GitHub Pages deploy | Public build |

M4 and M5 are the largest by far. M1 should arrive early so the game is fun before the
language is complete.

---

## 15. Open questions

1. **Ch 01–02.** No notes exist in the vault; the map assumes setup plus a first program.
2. **Theme.** Is the packet/network world in §7 what "Behind the Data Packet" means?
3. **Lesson text in game.** Should the game show chapter notes, or only link section
   names? The notes follow a video course and carry its timestamps.
4. **Typing.** Does "like monkeytype" also mean a typing-speed element (e.g. scoring
   how fast you write a solution), or only the look and feel? This file assumes look and
   feel.
5. **Exceptions.** Out for now; the course notes use `<stdexcept>` in two examples.

---

## 16. Conventions

- Commits are authored by Pursion only, with no co-author trailers.
- TypeScript `strict`, no `any` in `lang/`.
- Every language feature lands with its conformance cases in the same commit.
