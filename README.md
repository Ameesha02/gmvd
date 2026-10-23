# GMVD Reproduction

An end-to-end, from-scratch reproduction of **"GMVD: Smart Contract Vulnerability Detection Based on GAT-Mamba Framework"** (Liu & Zhang, Research Square preprint, DOI: 10.21203/rs.3.rs-8968309/v1 — posted March 2026, not peer-reviewed).

This project detects three smart-contract vulnerabilities — **reentrancy**, **timestamp dependence**, and **infinite loop** — using a from-scratch PyTorch implementation of the paper's Graph Attention Network (GAT) + Mamba (selective state-space model) + cross-attention fusion architecture, trained and evaluated on the real, official labeled datasets this line of research uses (traced back through the paper's own citation lineage — see the notebook's "Paper Analysis and Reproduction Plan" section for the full provenance).

## 1. Project overview

- **Main deliverable**: `notebooks/GMVD_reproduction.ipynb` — one notebook covering paper analysis, dataset acquisition/verification, EDA, preprocessing, model implementation, training, evaluation, baselines, ablations, a full reproduction audit, and a paper-vs-reproduction comparison.
- **No fabricated numbers**: every metric in the notebook comes from an actual run of the code in it.
- **Honest scope**: some baselines (5 static-analysis tools, DR-GCN/TMP/AME's *published* numbers) are cited from the paper rather than re-run — this is declared explicitly in the notebook, with reasons.
- **Headline finding (read Section 4b and the final "FINAL REPRODUCTION AUDIT" section of the notebook before citing any number from this project)**: the officially released train/valid split for this benchmark is contaminated — 99-100% of "validation" rows are exact duplicates of training rows. After fixing this with a leakage-free split, our faithful reimplementation of GMVD does **not** outperform a plain GCN baseline, and this holds up across 5 random seeds. This is reported as a genuine, evidence-based negative result, not attributed to a bug in this implementation. See the notebook for full details.

## 2. Requirements

- Windows, macOS, or Linux; **CPU is enough** (this was built and run on a CPU-only Windows machine — no GPU required, since every graph in this dataset has 3-10 nodes).
- Python 3.10+ (tested on 3.13).
- ~500 MB disk space for the virtual environment + data (excluding the cloned reference repos, which are optional).

## 3. Environment setup

From the project root (`gmvd/` — adjust the drive/path to wherever you placed this project):

```bash
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate

pip install --upgrade pip
pip install -r requirements.txt

python -m ipykernel install --user --name=gmvd --display-name="GMVD"
```

> **Note on disk space**: if your `C:` drive (or primary drive) is low on space, create the venv and project on a different drive — pip caches and PyTorch wheels can total 1-2 GB. Set `PIP_CACHE_DIR` to a directory on that drive if needed:
> ```bash
> set PIP_CACHE_DIR=D:\gmvd\.pipcache   (Windows cmd)
> $env:PIP_CACHE_DIR = "D:\gmvd\.pipcache"   (PowerShell)
> ```

## 4. Dataset setup

The labeled datasets are already included under `data/raw/` (small — a few MB total) — no download needed to run the notebook as-is:

| Vulnerability | Location | Contracts | Source |
|---|---|---|---|
| Reentrancy | `data/raw/reentrancy/` | 1,671 | `github.com/Messi-Q/AMEVulDetector` |
| Timestamp dependence | `data/raw/timestamp/` | 1,671 | `github.com/Messi-Q/AMEVulDetector` |
| Infinite loop | `data/raw/loop/` | 1,317 | `github.com/Messi-Q/GraphDeeSmartContract` |

If you want to re-derive these from scratch, or want the larger ~40,932-contract raw ESC corpus (mostly unlabeled `.sol` source used for pattern mining, not required for the numbers in this notebook), see `github.com/Messi-Q/Smart-Contract-Dataset` — it is hosted on Google Drive and may require a manual click-through download (Drive throttles large automated downloads).

No license click-through, API key, or authentication is required for the data actually used by this notebook.

## 5. How to open the notebook

```bash
jupyter lab notebooks/GMVD_reproduction.ipynb
# or
code notebooks/GMVD_reproduction.ipynb   # VS Code, with the Jupyter extension
```

Select the **"GMVD"** kernel (registered in step 3 above) when prompted.

## 6. How to execute it

Run all cells top to bottom (Jupyter Lab: "Run All Cells"; VS Code: "Run All"). The notebook is also fully executable non-interactively:

```bash
jupyter nbconvert --to notebook --execute --inplace notebooks/GMVD_reproduction.ipynb
```

## 7. Expected runtime

- Everything up to and including dataset loading/EDA: a few seconds.
- The full training sweep (9 models x 3 vulnerabilities = 27 runs, up to 120 epochs each with early stopping on validation F1, patience 15): **a few minutes on CPU** — these graphs are tiny (mean ~3-4 nodes) and datasets are small (~1,300-1,700 samples each), so convergence is typically reached well before 120 epochs.
- Total notebook runtime: well under 15 minutes on a modern CPU.

## 8. Expected output

- Printed per-model metrics (accuracy/precision/recall/F1/ROC-AUC) for all 27 training runs.
- Figures saved to `figures/`: class distributions, graph-size histograms, paper-vs-reproduction F1 bar charts, ablation study plot, confusion matrices, ROC curves, and an interpretability-weight plot reproducing the paper's Figure 2.
- CSVs saved to `results/`: `dataset_verification.csv`, `our_reproduction_results.csv`, `comparison_with_paper.csv`, `reproducibility_info.json`, plus a full pickle of all results/histories/predictions (`all_results.pkl`).

## 9. Results

**Read this before trusting any number.** An initial run of this pipeline against the *officially released* train/valid split scored ~100% accuracy almost everywhere — including a plain RNN using no graph information at all. That was investigated (notebook Section 4b) and traced to severe train/test contamination in the released data itself (99-100% of "validation" rows are exact duplicates of training rows). After fixing this with a leakage-free, content-deduplicated split, honest results are dramatically lower, and **the full GMVD model does not outperform a plain GCN baseline** on our reproduction — confirmed stable across 5 random seeds (Section 18b). See the notebook's "FINAL REPRODUCTION AUDIT" section (very end) for the complete numbers, the full evidence trail, and the reproduction-status verdict.

## 10. Reproduction limitations (read before citing this work)

- **Data leakage in the officially released split (found and fixed)**: `contract_name_train.txt`/`contract_name_valid.txt` in `AMEVulDetector` contain only ~185 unique contracts behind 1,671 rows each for reentrancy/timestamp, with the same contracts appearing in both files. We replaced this with `data_utils.load_dataset_clean`, a content-signature-grouped split verified to have zero train/valid overlap. **All final numbers in the notebook use the corrected split**; the notebook also demonstrates the original contamination for transparency.
- **A second, independent upstream data defect**: 51.6% of reentrancy edges and 27.8% of timestamp edges in the officially released JSON reference node indices that don't exist in that graph (verified against the raw file, not a parsing bug on our end) — these are safely dropped rather than crashing, but this measurably degrades the graph signal available to GAT-based models.
- **The paper's central comparative claim is not confirmed by our reproduction**: once leakage is fixed, `GMVD (full)` ranks 5th of 9, 5th of 9, and last of 9 (by F1) among the models we trained on reentrancy, timestamp, and loop respectively — a plain GCN baseline wins on two of the three. This is reported as a genuine finding, not an implementation bug (see the code-audit section in the notebook for the checks that rule out common bugs).
- The **5 static-analysis-tool baselines** (Smartcheck, Oyente, Mythril, Securify, Slither) and **infinite-loop tool baselines** (Jolt, PAD, SMT, Looper) are **cited from the paper's Table 2**, not independently re-run — this requires per-contract solc version matching across a decade-old corpus and several unmaintained tools, an infrastructure project orthogonal to the paper's ML contribution.
- **DR-GCN and TMP** numbers are similarly cited from the paper; we implement a plain GCN baseline and our own AME-lite reimplementation instead of byte-for-byte reproducing their exact normalization tricks.
- **Infinite-loop local-pattern embeddings** (P1/P2/P3) have no public release (unlike reentrancy/timestamp) — we derive a documented proxy from the graph's own node attributes rather than the paper's actual expert-pattern extractor.
- **RNN/LSTM/GRU baselines** use the `[P1,P2,P3]` pattern-embedding sequence rather than raw tokenized source code (which is not available for the full labeled sets locally).
- This is a **preprint that has not undergone peer review**; its own Table 2 baseline numbers were traced back to an earlier paper (AME, IJCAI 2021). If that paper's evaluation used the same publicly released split we found to be contaminated, its reported numbers (and possibly GMVD's own) may be similarly inflated — we cannot confirm this without the authors' code, but we can confirm the only public split for this data leaks.

## Project structure

Everything (dataset loading, leakage audit, GAT/Mamba/cross-attention model code, training harness, baselines, ablations, and the final audit) lives in **one self-contained notebook** — there are no side `.py` modules to import.

```
gmvd/
├── notebooks/GMVD_reproduction.ipynb   <- the entire project: dataset loading, model, training, audit
├── data/raw/{reentrancy,timestamp,loop}/  <- real labeled datasets
├── results/                             <- CSVs, JSON, pickled results (generated)
├── figures/                             <- saved plots (generated)
├── checkpoints/                         <- (unused by default; model states kept in-memory)
├── logs/                                <- execution logs
├── repos/                               <- cloned reference implementations (for provenance, not imported)
└── requirements.txt
```
