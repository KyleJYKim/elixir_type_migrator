# Backlog

Ideas parked until v1.0 is deployed (scope stays fixed until then).

- **Evaluation** (cut from v1 on 11-11-2026): labelled sample of ~200 checker warnings
  (code bug / spec bug / translator artefact / checker limitation), precision, comparison with
  Dialyzer, a `mix type.eval` corpus report.
- **Translating a hand-written `:: term`**: `term()` (exact) or `dynamic()`? Exact makes every
  `GenServer.call` wrapper with a narrower spec warn (e.g. `GenStage.estimate_buffered_count/2`).
  Ask Guillaume Duboc.
- **Upstream the arity fix** in `Module.Types.Descr.map_update/5` (fork commit 6c6ab0e).
