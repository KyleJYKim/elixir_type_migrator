# elixir_type_migrator

Suggests set-theoretic type signatures for Elixir functions with a fine-tuned code model,
then verifies each suggestion with the Elixir type checker. Built on my M.Sc. thesis with [type_migrator](https://github.com/KyleJYKim/type_migrator) and [type_migrator_with_llm](https://github.com/KyleJYKim/type_migrator_with_llm) repos.

Status: **pre-alpha** — see [docs/SPEC.md](docs/SPEC.md) for the v1 plan.

## Layout (planned)

| Path | What |
| --- | --- |
| `service/` | Python FastAPI inference service (`/suggest`) |
| `client/` | Elixir mix task: extract, suggest, type-check, accept/reject |
| `eval/` | Evaluation harness on a frozen test set |
| `baseline/` | Prompted API models with a generate → check → repair loop |
| `deploy/` | Docker, compose, CI and cloud deployment |
| `docs/devlog.md` | Daily devlog |
