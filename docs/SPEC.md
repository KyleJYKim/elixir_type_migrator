# elixir_type_migrator — v1 spec

*Status: draft, 11-11-2026 · Target: v1.0 tagged by Sun 06 Dec 2026, deployed by Sun 20 Dec 2026*

## 1. Goal

Given an Elixir project, **translate its hand-written TypeSpecs (`@spec`, `@type`, `@typep`,
`@opaque`) into Elixir set-theoretic types**, a.k.a. Elixir Types, keeping their exact meaning,
and **let the new type checker find where the code breaks its own contract.** Developers wrote
the code against these contracts, so a violation is a real finding: either the code or the spec
is wrong. No inference is involved, so every warning traces back to something the developer wrote.

v1 turns the translator from my M.Sc. thesis ([type_migrator](https://github.com/KyleJYKim/type_migrator))
into a tested, documented, deployable tool, keeping its source-based analysis as it is.

**Translation rule:** follow the translation rules already implemented in `type_migrator`.
A type that cannot be resolved (e.g. defined in another module or a dependency) becomes
`dynamic()`, so the checker treats that part of the contract as unchecked rather than guessing.

**Non-goals for v1:** type suggestion/inference (v2), analysis of compiled code (v3),
editor/LSP integration, training new models, rewriting source files without review, a polished UI.

## 2. Architecture

```
 mix type.migrate (in the user's project)
┌──────────────────────────────────────────────────────────────────────────┐
│ 1. read @spec / @type / @typep / @opaque from the project's source files │
│ 2. build the type table; resolve what the source allows, else dynamic()  │
│ 3. translate with the type_migrator rules → @assert_type_form …          │
│ 4. re-check the project with the gldubc/elixir fork, contracts in place  │
│ 5. report each warning: function, contract, value that breaks it         │
└──────────────────────────────────────────────────────────────────────────┘
        │ same translator, as a library
        ▼
 Migrate service (Elixir, Plug + Bandit): POST /migrate, GET /health — for the demo.
 v2 plugs LLM backends in behind the same service (POST /suggest).
```

| Path | Layer | Responsibility | Ported from (`type_migrator`) |
| --- | --- | --- | --- |
| `lib/source/` | Source | Find the project's `.ex` files and read their spec/type ASTs | `SpecTranslator.process`, `Utils.safe_string_to_quoted`, `TranslationRunner.discover_files` |
| `lib/type_table/` | TypeTable | Resolve user-defined types; anything unresolved becomes `dynamic()` | `TypeTableHandler` |
| `lib/translator/` | Translator | TypeSpec AST → Elixir Types | `Utils.parse/translate`, `Approximator`, `TypeTranslator` |
| `lib/emitter/` | Emitter | Render each translation as `@assert_type_form …` | `ElixirTypeConstructor` |
| `lib/checker/` | Checker | Compile the project with the fork (`typespec-translation-fixes`), collect the warnings | new, small |
| `lib/mix/tasks/type.migrate.ex` | Interface | The pipeline above on one project; table output, optional `--write` | `Migrator.main` |
| `lib/service/` | Interface | Plug + Bandit: `POST /migrate`, `GET /health`, input validation, latency logging | new |
| `test/fixtures/` | — | ~100 TypeSpec → Elixir Types cases | `type_migrator/example/` |
| `deploy/` | — | Docker image with the pinned fork, GitHub Actions CI, deploy on tag | new |

**Rules:**
- **Source is the only layer v3 replaces.** It hands the next layer spec ASTs, not file paths,
  so a `.beam` reader can take its place without changing anything downstream.
- **Translator does no I/O.** It takes an AST and a type table and returns a type; no files,
  processes or network. The mix task, the service and v2 all call it the same way.

Not ported: `lib/experiments/` (dataset, Dialyzer, evaluators), `lib/ex1–4.ex`, the thesis TeX.

### `/migrate` contract (v1)

```json
// request
{ "items": [ { "id": "Mod.fun/2", "spec": "@spec fun(integer, Keyword.t()) :: term" } ] }
// response
{ "translator_version": "…", "items": [ { "id": "Mod.fun/2",
    "translation": "@assert_type_form (integer(), dynamic() -> term())",
    "unresolved": ["Keyword.t()"], "latency_ms": 4 } ] }
```

`unresolved` lists the types that became `dynamic()`.

## 3. Done criteria for v1.0

1. `mix test` passes all ported fixtures on the pinned Elixir/OTP versions.
2. `mix type.migrate` runs on a real project (e.g. `gen_stage`) and lists each warning: function, contract, offending value.
3. `docker compose up` runs the service from a clean clone; `curl /migrate` returns the contract above.
4. CI runs the tests on every push; README shows the badge.
5. *(by 20 Dec)* Public demo: paste a TypeSpec, see the translation.

## 4. Starting assets

| Asset | Status | If missing |
| --- | --- | --- |
| Translator code (`type_migrator/lib/migrator/`) | Exists, ~4.8k lines with mix tasks | — |
| Fixtures (`type_migrator/example/`) | Exist, ~100 cases | — |
| Dataset (`results/dataset.jsonl`, 23,073 rows) | Exists in `type_migrator` | — |
| Corpus project list (repos + commits) | Repos recoverable from `dataset.jsonl` `file` paths (`owner/repo/…`); commits not recorded | Pin each repo to a commit when cloning; record them in a manifest |
| `gldubc/elixir` fork, local branch `typespec-translation-fixes` (`typespec-translation` + arity fix) | **6c6ab0e6e244d1c83bb599d6d2b6c186113be613** builds on OTP 28.1 with Elixir 1.20.5 (`.tool-versions`) | Pin a commit; v1 cannot report findings without it |
| *v2:* CodeT5+ / Qwen checkpoints, GPU access | **?** | Not needed for v1 |

## 5. Risks

- **Fork drift:** the fork moves or falls behind upstream → pin a commit SHA in CI and Docker.
- **`Descr` internals change** between Elixir releases → depend on the printed form, not internal structs, where possible.
- **Projects fail to compile** with the fork → `mix type.migrate` says so and shows the compiler output; no silent failures.
- **Too many `dynamic()` types** hide violations → report the share per project; it motivates v3.
- **Docker size/build time** from compiling the fork → build it once into a base image and cache it.
- **Scope creep:** new ideas go to `BACKLOG.md` until v1.0 is deployed.

## 6. Later versions

- **v2 — LLM type inference** for functions without specs (CodeT5+ 770M, Qwen2.5-Coder 7B,
  prompted API models with a generate → check → repair loop), behind `POST /suggest`.
- **v3 — analysis of compiled code.** Read specs and types from `.beam` debug info
  (`Code.Typespec.fetch_specs/1`, `fetch_types/1`, `spec_to_quoted/2`) instead of source, so
  types from other modules, dependencies and Erlang/OTP resolve, and macro-generated specs are
  included. Success = fewer `dynamic()` types than v1 and more findings on the same corpus.
  Cost: every analysed project must compile, and deps need `debug_info`.
