# Relational Semantic Machine (RSM)

A sparse, discrete, compositional reasoning engine. The working hypothesis
(PROJECT.md): meaning can be represented as compact compositional structures and
computation can be restricted to the relevant active structure, so intelligence
can scale far more efficiently than dense Transformer architectures.

Everything here is CPU-only, deterministic, pure-stdlib Python (>= 3.11). There
is no neural network, no embeddings, no vector database, and no LLM call in the
engine.

## Architecture (PROJECT.md §1)

```text
Natural language ──> Semantic Compiler ──> Thought IR ──> Sparse Semantic
                                                          Memory (concepts,
                                                          entities, relations,
                                                          rules, procedures)
                                                                │
                                                                v
                                                    Reasoning VM (lookup,
                                                    binding, unification,
                                                    rules, proofs) ──> result
```

Five components, one shared world:

1. **Semantic compiler** (`rsm/compiler/`) — controlled parsers and the
   procedure layer that maps verb *meanings* to executable IR templates
   (`semantics.py`), lexical aliases (`lexical.py`), and explicit competing-
   interpretation handling (`ambiguity.py`). Dataset grammars (SCAN, COGS,
   CFQ/SPARQL subsets, ...) live only in `rsm/adapters/`.
2. **Thought IR** (`rsm/ir/`) — a small typed, hashable AST (`Atom`,
   `Variable`, `Constant`, `Call`, `Sequence`, `Conjunction`, `Negation`,
   `Conditional`) with JSON / S-expression serialization and a validator.
3. **Sparse semantic memory** (`rsm/core/`) — stable symbol table with ID bands
   and loud collision detection, typed graph, facts/rules, states.
4. **Reasoning VM** (`rsm/reasoning/`) — dispatch executor over general
   operators (`SEQUENCE`, `REPEAT`, `NOT`, `CAUSE`, `END`, `RESOLVE`, ...),
   forward/backward chaining with explicit proof trees, unification with
   occurs check, three-valued TRUE/FALSE/UNKNOWN semantics.
5. **Instrumentation** — every operation is counted (`rsm/core/metrics.py`);
   every benchmark reports accuracy AND operations/query AND latency
   percentiles.

## Status: implemented vs deferred

Implemented milestones:

- **RSM-0 Semantic VM** — SCAN 2018: 1.000000 on all 16 generalization splits
  (41,001 examples), ops/query ≈ 12.7.
- **RSM-1 Structured English semantics** — COGS: executor gate 1.000000 on
  dev/test/gen; parser structural match 1.000000 end-to-end.
- **RSM-2 Reasoning VM** — CLUTRR multi-hop kinship (1.000000, L2–L6, one rule
  base), RuleTaker T/F/U (1.000000 over 1200 questions with explicit proof
  re-checking), ProofWriter CWA depth-5 (1.000000, exact gold-proof-graph
  match 1.0).
- **RSM-3 part 1 Hard composition** — CFQ MCD1/2/3: executor gate + gold
  round-trip 100%, end-to-end MCD mean 34.82% (parser coverage is the
  bottleneck; failures are attributed per PROJECT.md §14 layers).
- **Scaling evidence** — flat-query experiment (§28): with an argument-keyed
  index, query cost stays constant (4 edges/query) from 100 to 1,000,000
  irrelevant facts while the naive predicate-bucket index grows linearly.
- **§13 semantic torture seed** — the twelve canonical `kill` sentences as one
  shared TERMINATE procedure (`experiments/semantic_torture/seed_kill.py`).

Deferred (do not assume they exist): RSM-4 lexical ontology mining at scale,
**RSM-5+ learned/neural semantic compiler** (the current English coverage is
deliberately narrow and controlled), RSM-6 self-ingestion, RSM-7 automatic
abstraction discovery, any GPU/packed-layout work, frontier-model comparisons.

## Quick start

```bash
pip install -e .            # runtime needs nothing beyond Python >= 3.11

# Run one input (SCAN grammar or controlled English §3.2 sentences)
python -m rsm.cli.run --text "jump around right twice"
python -m rsm.cli.run --text "John killed the snake."
python -m rsm.cli.run --text "The committee killed the proposal."

# Inspect the machine
python -m rsm.cli.inspect concept KILL
python -m rsm.cli.inspect concept PROPOSAL
python -m rsm.cli.inspect symbols --namespace OPERATOR
```

Benchmarks — one unified entry point for every implemented dataset
(regenerates machine-readable metrics under `results/metrics/`, failures under
`results/failures/`, and human reports under `results/reports/`):

```bash
python -m rsm.cli.benchmark --dataset scan                 # RSM-0, full 41k
python -m rsm.cli.benchmark --dataset cogs --limit 2000    # RSM-1
python -m rsm.cli.benchmark --dataset clutrr               # RSM-2 (200/length)
python -m rsm.cli.benchmark --dataset ruletaker            # RSM-2 (1200 questions)
python -m rsm.cli.benchmark --dataset proofwriter --limit 200   # RSM-2, theories/bucket cap
python -m rsm.cli.benchmark --dataset cfq                  # RSM-3 (2000 rows/split)

# Scaling experiment (PROJECT.md §28): flat query cost vs knowledge size
python -m experiments.scaling.run_scaling

# Unknown dataset exits 2; missing data prints an explicit skip reason.
```

Quality gates:

```bash
python -m pytest -q -p no:cacheprovider
python -m ruff check .
python -m mypy rsm                # strict
```

## Repository layout

```
rsm/
|-- core/       ids, symbols, types, graph, state, memory, metrics
|-- ir/         nodes, operations, expressions, serializer, validator
|-- reasoning/  executor, matcher, unification, forward, backward, search,
|               proof, relational, uncertainty, semantic_ops
|-- compiler/   controlled_parser (COGS grammar), semantics (procedures),
|               lexical (aliases), ambiguity (interpretations)
|-- learning/   abstraction, compression, induction, pattern_mining, validator
|-- adapters/   scan, cogs, clutrr, ruletaker, proofwriter, cfq
`-- cli/        run, benchmark (+ bench_* runners), inspect
experiments/
|-- scaling/    §22 indexed store + §28 flat-query experiment
`-- semantic_torture/   §13 kill seed dataset generator
scripts/        dataset analysis / semantics verification helpers
tests/          unit / integration / generalization
results/        metrics/*.json, failures/*.json, reports/*.md
data/raw/       vendored benchmark datasets (see data/raw/DATA_STATUS.md)
```

## Reports

Human-readable evidence lives in `results/reports/`: `rsm0_scan.md`,
`rsm1_cogs.md`, `rsm2_clutrr.md`, `rsm2_ruletaker.md`, `rsm2_proofwriter.md`,
`rsm3_cfq.md`, `experiment_scaling_flat_query.md`, `status_round2.md`
(round summary table), and `architecture_log.md` (per-§39 rule 7 change log).
Every report states accuracy, operations/query, latency percentiles, and the
exact reproduction command. The full specification is `PROJECT.md`.

## Ground rules (from PROJECT.md)

- Deterministic: same input, same output, same trace, same metrics.
- Instrumented: no expensive work hidden in unmeasured helpers.
- No example-specific lookup tables, no one-rule-per-phrase patches, no dense
  vector computation anywhere in the core.
- Dataset formats stay in adapters; the core never sees SCAN/COGS/SPARQL syntax.
