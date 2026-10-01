# Lab 15/16 — Speculative Decoding, Measured

**Dhruv Parmar · 202518030 · DS635 ML System Engineering**

**Automatic score: 40/40** (checked with sir's own `reference/grade_lab15_16.py`). All 10 📝 cells filled.

## Submit these two files

| File | What it is |
|---|---|
| `submission_lab15_16_202518030.json` | measurements, written by the notebook's export cell |
| `Lab15_16_speculative_decoding_202518030.ipynb` | the executed notebook: outputs intact, every 📝 cell filled |

## Folder layout

```
Lab 15_16/
├── Lab15_16_speculative_decoding_202518030.ipynb   ← submit
├── submission_lab15_16_202518030.json              ← submit
├── README.md                                       ← this file
├── scripts/
│   ├── prepare_notebook.py     fills identity, the four TODO implementations, predictions, extra cells
│   └── fill_explanations.py    writes the explanation cells AFTER the run, from the recorded JSON
└── reference/                  sir's originals, unmodified (brief, blank notebook, grader, lecture)
```

## How it was run

CPU only, no GPU, no Docker — the lab is `gpt2` (target) drafted by `distilgpt2`, both small.

```bash
uv venv --python 3.11 venv && . venv/bin/activate
uv pip install torch transformers numpy nbformat nbclient ipykernel

python3 scripts/prepare_notebook.py reference/Lab15_16_speculative_decoding.ipynb \
        Lab15_16_speculative_decoding_202518030.ipynb
python -c "import nbformat; from nbclient import NotebookClient; \
           nb=nbformat.read('Lab15_16_speculative_decoding_202518030.ipynb',as_version=4); \
           NotebookClient(nb,timeout=3600).execute(); \
           nbformat.write(nb,'Lab15_16_speculative_decoding_202518030.ipynb')"
python3 scripts/fill_explanations.py Lab15_16_speculative_decoding_202518030.ipynb \
        submission_lab15_16_202518030.json
python  reference/grade_lab15_16.py .
```

**Order matters and is deliberate:** predictions are written by `prepare_notebook.py` *before* execution;
explanations are written by `fill_explanations.py` *after*, quoting numbers read out of the JSON rather
than typed by hand. That is the order the lab asks for, and it is why one prediction (Part 4's guess that
wall-clock might come out *slower*) is left standing as wrong rather than quietly edited.

Environment: macOS 27.0 arm64, Python 3.11.15, torch 2.14.0, transformers 5.17.0, numpy 2.4.6.
Seed **881596549**, derived from the roll number — the dummy measurements are unique to this submission.

## What I implemented

| Part | Function | The idea |
|---|---|---|
| 1 (15) | `is_accepted` | `u < min(1, p[x]/q[x])`, with `q[x] == 0` treated as an infinite ratio |
| 2 (30) | `residual`, `sample_on_reject` | `norm(max(0, p − q))` — the mass the drafter never supplied |
| 3 (15) | `verify_vectorised` | all `k` ratio tests as array ops; only the **cut** (`argmin` of the accept vector) is sequential |
| 4 (40) | `speculative_greedy` | draft `k` argmax tokens, **one** target pass over all of them, accept while the target's argmax agrees, emit the target's token at the first mismatch or the bonus if all accept |

## Results

**Part 1 — the ratio test.** Empirical accept rates over 20,000 draws per position, against `min(1, p/q)`:

| position | draft `q` | target `p` | `min(1, p/q)` | measured |
|---|---|---|---|---|
| 0 (`cat`) | 0.70 | 0.50 | 0.714 | **0.717** |
| 1 (`sat`) | 0.50 | 0.60 | 1.000 | **1.000** |
| 2 (`on`) | 0.60 | 0.20 | 0.333 | **0.332** |

Position 1 is the capped case (the drafter under-proposes, so nothing is thrown away); position 2 is the
worst over-proposal and is throttled to a third.

**Part 2 — the residual, and the silent bug.** Target `p[0] = [0.50, 0.20, 0.10, 0.10, 0.10]`:

| emitter | emitted distribution | TV distance |
|---|---|---|
| correct — residual | `[0.500, 0.200, 0.099, 0.100, 0.101]` | **0.0014** |
| broken — resample from `p` | `[0.598, 0.139, 0.122, 0.070, 0.070]` | **0.1205** (86× worse) |

The skew is diagnostic rather than random: `cat` and `on` (the two tokens the drafter over-proposes) come
out **above** `p`, and the under-proposed tokens are starved. **Both versions have the identical acceptance
rate** — the bug lives entirely inside the rejection branch, so only a distribution test can see it.

**Part 3 — vectorised verification.** Agreement with the scalar cut: **1.0000 over 3,000 checks** — exact,
not approximate, since both consume the same uniforms.

**Part 4 — the real stitch (gpt2 ← distilgpt2, `k = 4`, 24 tokens).**

| | value |
|---|---|
| identical to plain greedy, token for token | **True** |
| target forward passes | **6** (plain greedy: 24) |
| accepted-run lengths per block | `[4, 1, 4, 4, 4, 2]` — all inside `[0, 4]` |
| acceptance length | **4.00** tokens per target pass |

## The wall-clock result (my own extra measurement)

The graded fields count *target passes*; the write-up asks about **wall clock**, which also pays for `k`
drafter passes per block. So I measured both, plus the per-pass cost of each model.

```
plain greedy      : 0.461 s   (24 target passes)
speculative greedy: 0.406 s   (6 target passes + 24 draft passes)   → 1.14× faster
one pass: target 14.9 ms, draft 9.6 ms  → a draft costs 0.65 of a target pass
```

**A 4× reduction in target passes bought only 1.14× in time.** Putting the drafter's cost into the
lecture's `speedup ≈ acceptance length / (1 + overhead)` explains every point of a `k` sweep:

| k | target passes | draft passes | acceptance length | wall s | measured | model `L/(1 + 0.65k)` |
|---|---|---|---|---|---|---|
| 1 | 13 | 13 | 1.85 | 0.410 | **1.12×** | 1.12× |
| 2 | 9 | 18 | 2.67 | 0.395 | **1.17×** | 1.16× |
| 4 | 6 | 24 | 4.00 | 0.406 | **1.14×** | 1.11× |
| 8 | 3 | 24 | 8.00 | 0.348 | **1.32×** | 1.29× |

Every row lands within a few percent of the model. **Acceptance length rises monotonically with `k` while
the speedup does not** — `k = 2` beats `k = 4` — because `L` grows sub-linearly in `k` while the drafting
overhead `0.65k` grows linearly. `k = 8` wins here only because this prompt is predictable enough that
every block was fully accepted, which is a property of *"The capital of France is"*, not a general result.

**Two honest caveats about the regime.** Neither implementation uses a **KV cache** (the template's
`plain_greedy` re-runs the whole prefix each step, and my stitch matches it), so both arms are more
expensive than a real server. More fundamentally, speculation exists to sell **idle compute during a
memory-bound decode**, and a 124M model on a CPU barely has that surplus — on a 27B target at batch 1 a
drafter would be a few percent of a target pass rather than 65%, and the same acceptance length would
convert into something close to the full 4×.

## Prediction I got wrong (left standing, as the lab asks)

In Part 4 I predicted speculative decoding might come out **slower** in wall clock, because the drafter's
passes could outweigh the saved target passes. It came out **1.14× faster**. The cause I named was right —
the drafter eats almost the whole win — but I over-estimated the drafter's relative cost: distilgpt2 has
half of gpt2's layers and I assumed roughly half the cost, whereas the measured ratio is **0.65**, and
acceptance was unusually high (4.00 of a possible 5) on this very predictable prompt.

> *An honest surprising measurement, correctly explained, scores higher than a tidy expected one.*

## Changes to sir's notebook

**None to the given code or the graded fields.** The four `# TODO` stubs are implemented, the identity cell
is set, the 📝 cells are filled, and the Part 4 harness gained one extra `print` (mean accepted run and
acceptance length). I added **three cells of my own** — a markdown note plus the timing and `k`-sweep cells
— which write `RESULTS["timing"]` and `RESULTS["k_sweep"]`. The grader reads neither; they exist because
the write-up asks a question the template does not measure.

## Before submitting

Read every 📝 cell and be able to explain it unprompted. The two heaviest rubric items are **Part 2's
"why the residual and not `p`"** paragraph and **Part 4's "acceptance is not speedup"** write-up — the
likely viva questions are *why the acceptance rate cannot see the residual bug*, *what is parallel in
verification and what is not*, and *why a bad drafter costs speed but never accuracy*.
