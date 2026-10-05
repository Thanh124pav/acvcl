# REPRODUCTION_STATUS

Live status of the ACVCL experimental testbed. Updated as components come up.

## 1. Hardware / software

| Resource | Value | Note |
|---|---|---|
| GPU | 1× **NVIDIA H100 80 GB** carved into a **MIG 3g.40gb slice** (~42 GB usable) | not the full card; MIG mode disables NVIDIA EGL for CUDA-less GL rendering → MuJoCo must use `MUJOCO_GL=osmesa` |
| CPU | 4 cores (cgroup-pinned on a 128-core Xeon 8462Y+ host) | preprocessing bottleneck; cap `num_workers<=2` |
| RAM | 2 TiB visible (host-level) | effectively unbounded |
| Disk | 10 TB mount, **876 GB free (92 % used)** | budget-sensitive; skip 200+ GB datasets |
| Driver / CUDA | 570.133.20 / runtime 12.8, toolkit 12.2 on-host | PyTorch cu121 wheel used |
| Python | 3.10.12 | no conda; use venvs |
| ffmpeg | not system-installed | fallback: `imageio_ffmpeg` bundled binary |
| GL / Vulkan | `libEGL`, `libGL`, Vulkan icd.d present | software rendering fine; MIG blocks NV EGL |

## 2. Network policy (self-probed)

Allow-list behavior. Reachable hosts: `github.com`, `*.githubusercontent.com`,
`api.github.com`, `codeload.github.com`, `huggingface.co`, `pypi.org`,
`zenodo.org`, `carla.org`, `storage.googleapis.com` (public GCS). **Blocked**:
`*.ens.fr`, `*.inria.fr`, `*.eecs.yorku.ca`, `youcook2.eecs.umich.edu`,
`*.github.io`, `web.archive.org`, `drive.google.com` (DNS-only IPv6 unroutable),
`media.withyoutube.com`, `nuscenes.org`, `tau.ac.il`, `sjtu.edu.cn`.

Implication: **anything hosted only on university or Google-Drive mirrors is
unreachable from here**. HuggingFace Hub and public GitHub are the viable paths.

## 3. Python environments

| venv | Purpose | Status |
|---|---|---|
| `.venv/` (python 3.10) | main — torch cu121, transformers, decord, HF, hydra, gdown | ✅ |
| `.venv-mujoco/` | mujoco + dm_control + robosuite + metaworld | ✅ (robosuite 1.5.2 has a joint-type assertion with mujoco 3.14 — see sim table) |
| `.venv-maniskill/` | ManiSkill 3.0.1 / SAPIEN | ⚠ installed, but SAPIEN **cannot init Vulkan** in this container (`ErrorIncompatibleDriver`); even CPU-sim path forces Vulkan via `_setup_scene`. Hard-blocked. |
| `.venv-carla/` | CARLA | **not created** — CARLA requires its own server + Vulkan drivers; infeasible in this container |

### Key package versions

main venv: torch 2.4.1+cu121, torchvision 0.19.1+cu121, transformers 4.57.6,
decord 0.6, av 17.1, imageio 2.38, datasets 5.0.1, huggingface_hub 1.33.0,
timm 1.0.30, hydra-core 1.3.7, numpy 2.2.6.

mujoco venv: mujoco 3.14.0, dm_control (latest), gymnasium 1.4.0,
robosuite 1.5.2, metaworld 3.0.0, numpy<2.

## 4. Simulators

| Simulator | Status | Notes |
|---|---|---|
| **DMC — walker/stand**           | ✅ pass | 84×84 RGB via osmesa, 0.9 s |
| **DMC — reacher/easy**           | ✅ pass | 0.3 s |
| **DMC — cartpole/swingup**       | ✅ pass | 0.3 s |
| **MetaWorld — reach-v3**         | ✅ pass | MT1 API with `set_task` + rgb_array |
| **MetaWorld — pick-place-v3**    | ✅ pass | |
| **robosuite — Lift (Panda)**     | ✗ fail  | `AssertionError` in `get_joint_qpos_addr`: robosuite 1.5.2 adds composite robot models with free/ball joints; `binding_utils.py:525` restricts to hinge/slide. `robosuite_models` already installed but didn't help. Fix would need downgrading robosuite to 1.4 or patching the assert. |
| **RoboCasa — kitchen task**      | ⛔ deferred | downstream of robosuite; same blocker. |
| **ManiSkill — PushCube/PickCube**| ✗ fail  | SAPIEN `vk::createInstanceUnique: ErrorIncompatibleDriver` — Vulkan driver absent/unsupported in container. Even `sim_backend='cpu'` triggers Vulkan via `_setup_scene`. |
| **CARLA — any map**              | ⛔ deferred | needs 15 GB server binary + port 2000 + Vulkan driver; container-hostile. |

Rendering: `MUJOCO_GL=osmesa` + `PYOPENGL_PLATFORM=osmesa` works on this MIG
slice. GPU EGL is blocked by MIG mode (expected per NVIDIA MIG docs).

## 5. Datasets

| Dataset | Role | State | Path / Source | Issue |
|---|---|---|---|---|
| COIN annotations           | COIN task metadata                | ✅ full                 | `data/coin/COIN.json` (11 827 videos, 180 tasks) | — |
| YouCook2 splits + captions | YouCook2 cross-dataset validation | ✅ metadata-only        | `data/youcook2/merve__YouCook2` + `lmms-eval__YouCook2/data/*.parquet` | annotations json at UMich blocked |
| HowTo100M loader           | ref metadata                      | ✅                      | `data/howto100m/HuggingFaceM4__howto100m/` | raw videos stay on YouTube |
| CrossTask                  | TCC/CARL/LAV eval                 | ⛔ not downloadable      | — | upstream hosts (ENS, INRIA, YorkU) all blocked; no HF/Zenodo mirror |
| Pouring (Sermanet)         | TCC/CARL/LAV Exp 1 primary        | ✅ full 475 MB          | `data/pouring/sermanet_pouring/` (21 MOVs + 798 frame jpgs) | second attempt succeeded after disabling xet |
| YouCook2 InternVideo feats | Exp 7 real-data pair comparison   | ✅ 436 videos / 89 tasks | `data/youcook2/internvideo_features/internvideo_MM_L14_features/` | InternVideo-MM-L14 768-D @ 8 fps |
| Penn Action (subset)       | CARL eval                         | ⚠ not downloaded        | `urzsi/Penn-Action-Lossless-MKV-Test-Subset` (4.45 GB) | pending; disk budget OK |
| FineGym (skeleton)         | CARL eval                         | ⚠ not downloaded        | `Lozumi/FineGym-skeleton` (9.75 GB) | skeleton features; RGB subactions subset only |
| IKEA ASM                   | ref dataset                       | ⛔ skipped               | — | 206 GB on HF — exceeds our disk budget |
| SYNCR (metadata)           | SYNCR Exp 5                       | ✅ meta + SFT json      | `data/syncr/` | large tars (habitat 10 GB, kubric 38 GB) deferred |
| MDS-VQA (YouTube-UGC/SFV)  | MDS-VQA Exp 4                     | ⛔ unreachable           | — | hosted at `media.withyoutube.com` (blocked) |
| VideoSetDiff               | VideoSetDiff Exp 6                | ⛔ no release            | — | official repo is a website scaffold only; no code, no data |
| ActiveAD (nuScenes/CARLA)  | ActiveAD Exp (deferred)           | ⛔ out of scope          | — | repo is README-only; nuScenes requires auth + 400 GB |

## 6. Baseline repositories (`third_party/`)

| Baseline | Repo | Code? | Data? | Trainable here? |
|---|---|---|---|---|
| TCC            | `google-research/tcc` (sparse) + `June01/tcc_Temporal_Cycle_Consistency_Loss.pytorch` | ✅ | ⚠ needs Pouring/PennAction (CrossTask blocked) | ✅ once Pouring finishes |
| LAV            | `hadjisma/VideoAlignment` | ✅ | needs Penn Action etc. | ✅ when data ready |
| CARL           | `minghchen/CARL_code` | ✅ (+ data_prep scripts) | needs Pouring/PennAction/FineGym | ✅ when data ready |
| MDS-VQA        | `Multimedia-Analytics-Laboratory/MDS-VQA` | ✅ | ⛔ datasets unreachable | ⛔ |
| ActiveAD       | `Thinklab-SJTU/ActiveAD` | **README-only** | — | ⛔ |
| VideoSetDiff   | `qiuyue1993/videosetdiff` | website scaffold, no model code | — | ⛔ |
| SYNCR          | `SaraGhazanfari/SYNCR` | data-generation scripts | ✅ (benchmark on HF) | 🟡 eval possible, generation needs CLEVRER/Kubric/Habitat (deferred) |

## 7. Experiments

| ID | Baseline | Dataset | Simulator | venv | Setup | Train | Eval | Result | Issue |
|---|---|---|---|---|---|---|---|---|---|
| 1 | TCC (minimal port) | Pouring (sermanet, 11 train / 10 val MOVs) | — | main | ✅ | ✅ 100 iters | ✅ cycle-cons | **val_cycle 0.125** (vs 0.153 at init) | TCC regression σ² term is noisy on tiny dataset; see §9 |
| 2 | LAV              | Pouring                              | — | main   | code only | — | — | — | not run; the Soft-DTW variant can be run against the TCC pipeline next |
| 3 | CARL (ref only)  | Pouring                              | — | main   | ⚠ legacy deps (torch 1.6 + tf-gpu 2.3 incompatible with H100 / CUDA 12) | — | — | — | would need a port; we use their dataset prep + the ported TCC head instead |
| 4 | MDS-VQA          | YouTube-UGC / SFV                    | — | main   | ⛔ blocked | — | — | — | datasets unreachable |
| 5 | SYNCR eval       | HF SYNCR test metadata               | — | main   | ✅ metadata only | — | — | — | 10 GB habitat tar needed for actual eval; LoRA SFT deferred |
| 6 | VideoSetDiff     | —                                    | — | main   | ⛔ no code | — | — | — | official repo is website scaffold |
| 7 | **Pair-strategy** controlled (YouCook2 InternVideo) | YouCook2 InternVideo-MM-L14 (436 videos / 89 tasks) | — | main | ✅ | ✅ 300 iters/selector (4s on H100) | ✅ held-out same-task cycle-cons | **sim > hard > random**: 0.131 / 0.091 / 0.030 | ran on real precomputed features |

### Exp 1 (TCC-port on Pouring) — detail

- Model: frozen `facebook/dinov2-small` → 2-layer Conv1d temporal head (384 → 256 → 128).
- Data: 11 train MOVs (Sermanet Pouring), 10 val MOVs (sampled 24 frames each at 224×224).
- Loss: TCC cycle-back regression (my port in `src/losses/tcc.py`).
- 100 iters @ 1 pair/iter, Adam lr 3e-4, bf16 autograd on MIG 3g.40gb slice.
- Runtime: ~500 s (~5 s/iter — bottlenecked by DINOv2 forward on 48 frames + 4 CPU video decode).
- Val cycle consistency: 0.153 → 0.125 (small dataset, high variance; σ² denominator in the regression loss amplifies instability — reproducible but noisy).

### Exp 7 — detail

Shared across runs: 2-layer Transformer temporal head (256 hidden, 4 heads), lr 1e-3, Adam, 48 frames, 2 pairs/iter, 300 iters, bf16. Held-out set = 16 same-task pairs chosen with a fixed seed never used in training.

| selector    | loss@start | loss@end | **final cycle** | runtime |
|-------------|-----------:|---------:|----------------:|--------:|
| random      | ~1e7 (σ² term blow-up) | 6.25 | **0.030** | 4 s |
| similarity  | ~2.2e7 | 6.24 | **0.131** | 4 s |
| hard_pair   | ~2.8e6 | 41.80 | **0.091** | 4 s |

Interpretation (preliminary): selecting *similar* videos within a task yields a cleaner training signal than random pairing on this feature-space; hard pairs are informative but noisier at this scale. Matches the hypothesis motivating ACVCL.

## 8. Scripts added

| path | purpose |
|---|---|
| `scripts/env_check.py`              | torch/CUDA/decord/pyav/DINOv2 health |
| `scripts/smoke_test_sims.py`        | DMC / robosuite / MetaWorld smoke tests (osmesa) |
| `scripts/inspect_dataset.py`        | per-dataset sanity report |
| `scripts/smoke_test_pairs.py`       | COIN surrogate pair-selection pipeline |
| `scripts/prepare_pouring.py`        | sermanet/pouring MOV → CARL train/val/test.pkl |
| `scripts/tcc_train_pouring.py`      | **Exp 1** — TCC-port training on Pouring |
| `scripts/exp7_pair_strategy_youcook2.py` | **Exp 7** — real controlled pair-strategy on InternVideo features |

## 9. Open issues / deferrals

1. **CrossTask** — only hosted on blocked ENS / INRIA / YorkU URLs. No HF or
   Zenodo mirror found. Options: user-side upload, or re-extract DINOv2
   features from the CrossTask YouTube IDs (needs YouTube reach, not tested).
2. **CARL original repo** is Python 3.7 + torch 1.6 + tf-gpu 2.3 (CUDA 10.1).
   None of that works on an H100 CUDA 12 stack. Would need a full port —
   we instead reuse CARL's `dataset_preparation` scripts and ported only
   the TCC loss into `src/losses/tcc.py`.
3. **TCC regression loss** on tiny datasets (Pouring 11 videos) is numerically
   noisy — the `log σ²` term dominates whenever the softmax collapses on
   a single frame. For the next pass we should switch to the TCC
   *classification* variant or add σ clamping. Not critical for pipeline
   validation.
4. **robosuite 1.5.2 + mujoco 3.14** `get_joint_qpos_addr` assert. Fix:
   downgrade robosuite to 1.4 (next time we need it).
5. **ManiSkill / SAPIEN** needs Vulkan which this container cannot provide
   (`ErrorIncompatibleDriver`). No fix without changing container image.
6. **CARLA / ActiveAD / VideoSetDiff / MDS-VQA** data or server infeasible,
   or no public code — treated as references only.
7. **SYNCR eval**: downloading `habitat_data.tar` (10 GB) + `kubric_data.tar`
   (38 GB) would use 5 % of our disk budget. Deferred until we actually need
   to run multimodal eval; the SFT JSON (`sft_all8_answer_only.json`,
   20 MB) is already in `data/syncr/train/`.
