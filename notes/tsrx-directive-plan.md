# Plan: `"civet tsrx"` directive

Incrementally add a Civet prologue directive that routes JSX through tsrx.

**Decisions locked:**
- Integration shape: **pre-parse JSX with tsrx** (Civet grammar untouched; JSX source spans handed to tsrx `parseModule`).
- Output target: **decide later** (tsrx core is AST-only, no codegen/runtime).

## How Civet directives work (background)

- A directive is a prologue string: `"civet <opts>"` at top of file.
- **Defaults** live in the config object in `source/parser.hera` (~L9696–9735), e.g. `react: false`, `solid: false`.
- **Parse**: `CivetOption` rule (`source/parser.hera` L9429–9451) turns `"civet foo"` into `[optionName, value]`, camelCased.
- **Apply**: `Init` rule (L9838–9846) does `Object.assign(config, directive.config)`.
- **Compound flags** (e.g. `coffeeCompat` fanning out to many sub-flags) use `Object.defineProperty` setters (~L9747).
- Behavior is read downstream from `config.<flag>` in `source/parser/lib.civet`.

---

## Phase 0 — plumb the flag (no behavior)

Goal: `"civet tsrx"` parses, flows into config, changes nothing yet. Safe, mergeable.

1. Add default `tsrx: false,` to config object near `solid: false` (`source/parser.hera` ~L9726).
2. Add `tsrx: boolean` to `types/types.d.ts` near `solid` (L39).
3. Mirror in `source/main.civet` ParseOptions (~L93) if the public option needs declaring.
4. Test: add a case in `test/prologues.civet` asserting `"civet tsrx"` is consumed with no output change.

**Done =** flag exists and is ignored.

## Phase 1 — reach the transform layer

Goal: prove the flag is visible where JSX is emitted.

5. Locate JSX emit in `source/parser/lib.civet` (`typeOfJSX`, `convertObjectToJSXAttributes`, JSX codegen).
6. Behind `if config.tsrx`, add a temporary marker (debug comment in output). Test it appears only with the directive.

**Done =** control flow forks on the flag.

## Phase 2 — pre-parse JSX through tsrx (no codegen yet)

Goal: smallest real round-trip.

7. Add dep: `pnpm add @tsrx/core` (published on npm). NOT the in-repo `tsrx/` copy — it's absent from `pnpm-workspace.yaml` and its deps use Ripple's `catalog:default`, so it won't resolve here. In-repo `tsrx/` = source reference only.
8. When `config.tsrx`, extract the JSX source span (use AST node `loc` + exact source slice) and feed it to tsrx `parseModule(src, filename)`.
9. tsrx returns an ESTree AST only — so emit the JSX unchanged for now; just prove parse succeeds (assert no throw / debug marker).

**Done =** tsrx parses Civet's JSX spans without error.

## Phase 3 — codegen (after target chosen)

Goal: actually emit tsrx runtime output.

10. Add a framework package (`@tsrx/react`, `@tsrx/ripple`, etc.) — core ships no emitter.
11. Walk tsrx AST → emit runtime; replace the JSX span in output.
12. Wire source maps via tsrx `convertSourceMapToMappings`.

**Blocker:** no target = no emitter. Phases 0–2 are fully doable without picking one.

---

## Risks to track

- **Source maps**: splicing tsrx output breaks Civet's offset mapping. Defer to Phase 3 with `convertSourceMapToMappings`.
- **Span extraction**: need exact JSX source slice + offset; verify Civet AST nodes carry usable `loc` before feeding tsrx.
- **Dep wiring**: use npm `@tsrx/core`. In-repo `tsrx/` is not a workspace pkg and uses Ripple's pnpm catalog — don't link it.

## Workflow

- One phase = one PR. Tests green before moving on.
- Start: Phase 0, steps 1–4.

## Key file refs

- `source/parser.hera` — directive grammar + config defaults
- `source/parser/lib.civet` — JSX emit / transform
- `source/main.civet` — public ParseOptions
- `types/types.d.ts` — option type decls
- `test/prologues.civet`, `test/jsx/` — tests
- `@tsrx/core` (npm) — tsrx core (`parseModule`, scope analysis, source-map utils). In-repo `tsrx/` mirrors it for reference.
