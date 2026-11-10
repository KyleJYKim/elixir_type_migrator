# elixir_type_migrator — v1 spec

*Status: draft, 09-11-2026 · Target: v1.0 tagged by Sun 06 Dec 2026, deployed by Sun 20 Dec 2026*

## 1. Goal

Given an Elixir module, suggest set-theoretic type signatures for its functions,
**verify every suggestion with the Elixir type checker**, and report which ones it accepts.
The project turns my M.Sc. thesis (fine-tuned models on a 15,499-example dataset, filtered
from 23,073) into a running, tested and deployed tool, and compares fine-tuning with
prompting frontier API models.

**Non-goals for v1:** editor/LSP integration, training new models, serving the 7B model
in production, multi-module or whole-project inference, a polished UI.

## 2. Architecture

```
 mix type.suggest MyMod            FastAPI service                     Backends
┌──────────────────────┐  HTTP   ┌───────────────────────┐   ┌──────────────────────────┐
│ Elixir client        │ ──────▶ │ POST /suggest         │──▶│ codet5p  (CodeT5+ 770M,   │
│ 1. read specs/source │  JSON   │ GET  /health          │   │           CPU, default)  │
│ 2. call /suggest     │ ◀────── │ backend registry      │──▶│ qwen7b   (offline eval,  │
│ 3. type-check each   │         └───────────────────────┘   │           GPU only)      │
│    (type_migrator)   │                                     │ llm-api  (Claude/OpenAI/ │
│ 4. accept / reject   │                                     │  Gemini, generate→check→ │
└──────────────────────┘                                     │  repair, max N rounds)   │
                                                             └──────────────────────────┘
 Evaluation harness (Python): frozen test set ─▶ /suggest per backend ─▶ checker ─▶ report.json + report.md
```

| Component | Lang | Responsibility |
| --- | --- | --- |
| `service/` | Python 3.12, FastAPI | Load a backend, expose `/suggest` and `/health`, validate input, log latency |
| `client/` | Elixir, mix task | Extract functions + existing specs, call service, verify with the checker via `type_migrator`, print a table |
| `eval/` | Python | Run a backend over the frozen test set, score, write JSON + Markdown |
| `baseline/` | Python | Provider-agnostic LLM backend with the generate→type-check→repair loop |
| `deploy/` | Docker, GH Actions | Image, compose (service + client), CI, deploy on tag |

### `/suggest` contract (v1)

```json
// request
{ "backend": "codet5p", "items": [ { "id": "Mod.fun/2", "source": "def fun(a, b) ...", "spec": "@spec fun(integer, term) :: term" } ] }
// response
{ "backend": "codet5p", "model_version": "…", "items": [ { "id": "Mod.fun/2", "suggestions": ["…"], "latency_ms": 41 } ] }
```

Type-checking happens on the Elixir side (the service never needs an Elixir toolchain),
except in the repair loop, which calls the same checker through a small CLI wrapper.

## 3. Evaluation

- **Test set:** ~500 examples held out from the 15,499, frozen on Wed 18 Nov with a
  committed manifest (ids + SHA-256). Never used for prompt tuning.
- **Metrics:** exact match (normalised type AST); subtype compatibility (suggestion is a
  subtype of, or equal to, the reference, per the checker); checker-accept rate;
  plus cost (USD) and p50/p95 latency for API backends.
- **Comparison table:** fine-tuned CodeT5+ · Qwen 7B (if GPU) · prompted API model ·
  prompted + repair (1–3 rounds).
- **Sanity check:** reproduce the thesis numbers on this test set; explain any gap.

## 4. Done criteria for v1.0

1. `mix type.suggest MyMod` prints suggestions with accepted/rejected status against a running service.
2. `docker compose up` runs service + client from a clean clone.
3. CI runs Python and Elixir tests and the evaluation on every push; README shows the badge and latest report.
4. Report covers at least CodeT5+ and one API model with and without repair.
5. README has the problem, architecture, results table and how to run.
6. *(Goal 4, by 20 Dec)* Public demo URL on a cloud provider, auto-deployed on tag, API keys as secrets.

## 5. Starting assets — to confirm today

| Asset | Status | If missing |
| --- | --- | --- |
| CodeT5+ 770M fine-tuned checkpoint | **?** | Re-fine-tune (~1–2 GPU days) or ship API-only and move fine-tuning to v1.1 |
| Qwen2.5-Coder 7B checkpoint / adapter | **?** | Drop from table; note as thesis result only |
| GPU access (and until when) | **?** | CodeT5+ inference on CPU; run Qwen eval while access lasts |
| Dataset (15,499 + filtering scripts) | **?** | Rebuild from the build script; blocks the test set on 18 Nov |
| `type_migrator` (Elixir version it targets, does it build?) | **?** | Pin the Elixir version it needs; budget a day to port |

## 6. Risks

- **Checker drift:** Elixir's type system changes between releases → pin Elixir/OTP versions in CI and Docker.
- **Data licensing:** redistribution may be blocked → publish build script instead (decided 24 Nov).
- **API cost:** hard spending cap per provider; cache responses keyed by (prompt, model).
- **Scope creep:** new ideas go to `BACKLOG.md` until v1.0 is deployed.
