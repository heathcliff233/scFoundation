# AGENTS Guide for scFoundation

## 1. Scope and Priorities
This guide is focused on:
- model architecture and inference mechanics
- data/preprocessing pipeline and data contracts
- how data and model stages connect end to end

This guide is intentionally not benchmark-focused. Downstream task folders are noted for navigation only.

## 2. High-Level Repository Layout
Core paths for architecture/data work:
- `model/`: checkpoint loading, embedding inference entrypoint, model internals, finetune example.
- `preprocessing/`: single-cell preprocessing helpers and demo download workflow.
- `OS_scRNA_gene_index.19264.tsv`: canonical gene index/order contract.

Other task folders (inventory only):
- `enhancement/`, `DeepCDR/`, `SCAD/`, `GEARS/`, `annotation/`, `mapping/`, `genemodule/`, `ablation/`, `apiexample/`.

## 3. Architecture Pipeline (Model Side)
Primary entrypoint: `model/get_embedding.py`.

Inference flow:
1. Parse CLI args (`input_type`, `output_type`, `pre_normalized`, `tgthighres`, `pool_type`).
2. Load expression input from `csv`/`npy`/`npz`/`h5ad`.
3. Align gene features to the 19264-gene canonical list (`main_gene_selection`).
4. Build token sequence:
- base sequence is 19264 genes
- append 2 scalar tokens (resolution and total-count), resulting in length 19266
5. Convert sparse/nonzero structure via `gatherData(...)` from `model/load.py` and build padding masks.
6. Load pretrained checkpoint with `load_model_frommmf(...)` (`model/load.py`), which:
- converts MMF-style checkpoint config (`convertconfig`)
- instantiates model via `select_model(...)` (`model/pretrainmodels/select_model.py`)
7. Run architecture:
- token value embedding: `AutoDiscretizationEmbedding2` (`model/pretrainmodels/mae_autobin.py`)
- positional embedding: `nn.Embedding(max_seq_len+1, dim)`
- encoder/decoder modules selected as performer or transformer blocks
8. Emit outputs by mode:
- `cell`: pooled encoder embeddings
- `gene`: per-cell gene-context embeddings via encoder-decoder path
- `gene_batch`: batch-mode gene-context embeddings
- `gene_expression`: decoder scalar output mode
9. Save `.npy` artifact to `save_path` with task/version/resolution metadata in filename.

## 4. Data Pipeline (Preprocessing Side)
Primary helpers: `preprocessing/scRNA_workflow.py`.

Data-prep flow:
1. Load raw matrices with Scanpy tooling (see `preprocessing/README.md` demo usage).
2. Harmonize gene symbols to `OS_scRNA_gene_index.19264.tsv` via `main_gene_selection(...)`.
3. Run QC/filtering:
- `BasicFilter(...)` for min genes/cells filtering
- `QC_Metrics_info(...)` for QC metric generation/inspection
4. Save processed objects as `.h5ad` using `save_adata_h5ad(...)`.
5. Feed processed matrix into `model/get_embedding.py` for embedding generation.

Download/bootstrap script: `preprocessing/demo.sh` (uses `down.sh`).

## 5. Model-Data Interface Contract
Required assumptions for stable inference:
- Gene axis must match the canonical 19264 gene order (`OS_scRNA_gene_index.19264.tsv`).
- For `singlecell`, preprocessing semantics depend on `pre_normalized` (`F`, `T`, `A`):
- `F`: script applies normalize_total + log1p style transform per sample.
- `T`: input already normalized/log-transformed.
- `A`: input already normalized/log-transformed and includes appended total-count.
- `tgthighres` controls the added resolution token (`t*`, `f*`, `a*` modes).
- For `bulk`, total-count token handling differs based on `pre_normalized`.
- `output_type` controls output tensor shape and downstream compatibility.

## 6. Key Files for Quick Navigation
- `model/get_embedding.py`: end-to-end inference CLI and tensor construction.
- `model/load.py`: checkpoint conversion/loading + gather utilities.
- `model/pretrainmodels/select_model.py`: model/encoder/decoder module selection.
- `model/pretrainmodels/mae_autobin.py`: tokenization embedding and encoder-decoder forward path.
- `model/finetune_model.py`: example for integrating/freezing parts of scFoundation.
- `preprocessing/scRNA_workflow.py`: preprocessing and `.h5ad` persistence helpers.
- `model/demo.sh`: sample commands across multiple data/task modes.

## 7. Other Task Scripts (Not Primary Scope)
These are useful for locating runnable workflows but are not the architecture/data core:
- `enhancement/run.sh`: read-depth enhancement task scripts.
- `DeepCDR/prog/run.sh`: drug response prediction training/eval scripts.
- `SCAD/run.sh`: SCAD drug sensitivity scripts with and without embeddings.
- `GEARS/run_sh/*.sh`: perturbation modeling runs using baseline or scFoundation features.
- `annotation/README.md`: cell type annotation usage notes.
- `mapping/README.md`: cell mapping workflow notes.
- `genemodule/README.md`: gene module analysis notes.
- `ablation/README.md`: ablation notebook map.
- `apiexample/README.md`: API usage references.

## 8. Practical Caveats
- Checkpoint binaries are not fully contained in repo; `model/models/download.txt` points to external retrieval.
- Runtime paths in `model/get_embedding.py` assume working directory relative paths (for example `./models/models.ckpt`).
- Code paths are GPU-oriented in multiple places (`.cuda()` calls in inference path).

## 9. Suggested Agent Workflow
When modifying architecture/data behavior:
1. Start at `model/get_embedding.py` to confirm active mode/shape path.
2. Trace into `model/load.py` and `model/pretrainmodels/*` for model semantics.
3. Validate gene-order and normalization assumptions against `preprocessing/scRNA_workflow.py`.
4. Use `model/demo.sh` commands for quick smoke checks after changes.
