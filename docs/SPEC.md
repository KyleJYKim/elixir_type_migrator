# elixir_type_migrator — v1 spec

*Status: draft, 10-11-2026 · Target: v1.0 tagged by Sun 06 Dec 2026, deployed by Sun 20 Dec 2026*

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

| Path | Lang | Responsibility |
| --- | --- | --- |
| `lib/translator/` | Elixir | TypeSpec AST → Elixir Types, ported from `type_migrator/lib/migrator/` |
| `lib/checker/` | Elixir | Run the fork's checker with contracts in place; map warnings back to contracts |
| `lib/service/` | Elixir, Plug + Bandit | `/migrate` and `/health`, input validation, latency logging |
| `lib/mix/tasks/type.migrate.ex` | Elixir | The pipeline above on one project; table output, optional `--write` |
| `lib/mix/tasks/type.eval.ex` | Elixir | Run the pipeline over the corpus, write `report.json` + `report.md` |
| `test/fixtures/` | Elixir | ~100 TypeSpec → Elixir Types cases ported from `type_migrator/example/` |
| `deploy/` | Docker, GH Actions | Image with the pinned fork, CI, deploy on tag |

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

## 3. Evaluation

The question v1 answers: **how many real contract violations does the new type checker find in
existing code, and how many of its warnings can be trusted?**

- **Corpus:** the open-source projects behind the thesis dataset that compile with the fork
  (the thesis's 23,073 specs are the starting list). No training, so nothing is held out.
- **Primary — findings:** warnings per project and per 1,000 specs. A labelled sample of
  ~200 warnings, each sorted into *code bug* · *spec bug* (contract wrong or out of date) ·
  *translator artefact* · *checker limitation*. Precision = (code bugs + spec bugs) / sample.
- **Compared with Dialyzer** on the same specs: found by both, by the checker only, by Dialyzer only.
- **Preconditions:** fixture pass rate (target 100%); share of types left as `dynamic()`
  (the baseline v3 has to beat).
- **Report:** `mix type.eval` writes `report.json` and `report.md`; CI publishes the summary.

## 4. Done criteria for v1.0

1. `mix type.migrate` in a project lists every contract violation: function, contract, offending value.
2. `mix test` passes all ported fixtures on the pinned Elixir/OTP versions.
3. `mix type.eval` produces the findings report on the corpus; the labelled sample and precision table are in the README.
4. `docker compose up` runs the service from a clean clone; `curl /migrate` returns the contract above.
5. CI runs tests and the evaluation on every push; README shows the badge.
6. *(Goal 4, by 20 Dec)* Public demo: paste a TypeSpec, see the translation.

## 5. Starting assets

| Asset | Status | If missing |
| --- | --- | --- |
| Translator code (`type_migrator/lib/migrator/`) | Exists, ~4.8k lines with mix tasks | — |
| Fixtures (`type_migrator/example/`) | Exist, ~100 cases | — |
| Dataset (`results/dataset.jsonl`, 23,073 rows) | Exists in `type_migrator` | — |
| Corpus project list (repos + commits) | Repos recoverable from `dataset.jsonl` `file` paths (`owner/repo/…`); commits not recorded | Pin each repo to a commit when cloning; record them in a manifest |
| `gldubc/elixir` fork, branch `typespec-translation` | **?** builds on OTP 28.1? | Pin a commit; v1 cannot report findings without it |
| *v2:* CodeT5+ / Qwen checkpoints, GPU access | **?** | Not needed for v1 |

## 6. Risks

- **Fork drift:** the fork moves or falls behind upstream → pin a commit SHA in CI and Docker.
- **`Descr` internals change** between Elixir releases → depend on the printed form, not internal structs, where possible.
- **Corpus projects fail to compile** with the fork → report the compile rate; findings only on projects that build.
- **Too many `dynamic()` types** hide violations → report the share per project; it motivates v3.
- **Labelling is manual** (~200 warnings) → budget about two days; write the labelling guide first.
- **Docker size/build time** from compiling the fork → build it once into a base image and cache it.
- **Scope creep:** new ideas go to `BACKLOG.md` until v1.0 is deployed.

## 7. Later versions

- **v2 — LLM type inference** for functions without specs (CodeT5+ 770M, Qwen2.5-Coder 7B,
  prompted API models with a generate → check → repair loop), behind `POST /suggest`.
- **v3 — analysis of compiled code.** Read specs and types from `.beam` debug info
  (`Code.Typespec.fetch_specs/1`, `fetch_types/1`, `spec_to_quoted/2`) instead of source, so
  types from other modules, dependencies and Erlang/OTP resolve, and macro-generated specs are
  included. Success = fewer `dynamic()` types than v1 and more findings on the same corpus.
  Cost: every analysed project must compile, and deps need `debug_info`.
