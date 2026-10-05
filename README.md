# ACVCL — Active Cross-Video Consensus Learning (testbed)

Experimental testbed for cross-video representation learning. The current
goal is **not** to implement the full ACVCL method but to stand up the data
pipeline, baseline selectors (Random / Similarity / Hard Pair), and a
minimal TCC-style alignment loss so that future ACVCL selectors can be
dropped in behind the same API.

Research spec: see [`SETUP.md`](SETUP.md). Current status: see
[`SETUP_STATUS.md`](SETUP_STATUS.md).

## Environment

```bash
# venv already exists at .venv/ (Python 3.10.12)
source .venv/bin/activate
pip install -r requirements.txt            # already installed in this env
python scripts/env_check.py                # torch / CUDA / ffmpeg / HF
```

Compute: 1× NVIDIA H100 **MIG 3g.40gb slice** (≈ 42 GB usable), 4 CPU cores,
CUDA 12.1 PyTorch build. bf16 is supported.

## Datasets

| dataset     | what is here                                     | path                                    |
|-------------|--------------------------------------------------|-----------------------------------------|
| COIN        | full annotations JSON (11 827 videos, 180 tasks) | `data/coin/COIN.json`                   |
| YouCook2    | split lists + duration CSVs (train/val/test)     | `data/youcook2/merve__YouCook2/splits/` |
| HowTo100M   | HF loader + extensions metadata                  | `data/howto100m/HuggingFaceM4__howto100m/` |
| CrossTask   | **not downloaded** — upstream hosts blocked      | see `SETUP_STATUS.md`                   |

Inspect a dataset:

```bash
python scripts/inspect_dataset.py --dataset coin
python scripts/inspect_dataset.py --dataset youcook2
python scripts/inspect_dataset.py --dataset howto100m
```

## Baselines

Pulled into `third_party/` for reference only (do not merge into main):

- `third_party/CrossTask/`  — DmZhukov/CrossTask (code only).
- `third_party/lav/`        — hadjisma/VideoAlignment (LAV).
- `third_party/tcc_ref/tcc/`— google-research/tcc (sparse checkout).
- VideoSetDiff / SYNCR — **no public code found yet**.

## Smoke tests

```bash
# A + C combined: pair sampling -> temporal module -> TCC loss -> backward
python scripts/smoke_test_pairs.py
```

Produces a table comparing `random`, `similarity`, `hard_pair` selectors on
loss convergence. See `SETUP_STATUS.md` for latest run numbers.
