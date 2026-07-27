# MPGEM — Molecular Prediction of Gene Expression Matrix

MPGEM is a web tool that expands a partial gene expression matrix into a much larger one. You supply
expression values for a defined set of genes; a pre-trained deep learning model predicts the values
for the remaining genes and returns both sets merged into a single matrix.

Developed and maintained by **SciWhy Lab**.

---

## Access

**https://mpgempre.streamlit.app**

Freely available in any web browser — nothing to install, no account, no login. Uploaded data is
held only for the duration of your session; results are downloaded by you and nothing is retained
afterwards.

---



## How the tool works

The model is built around a **fixed reference gene list** with a fixed ordering. The first
**12,712** genes in that list are the model's input layer; every gene after position 12,712 is an
output the model predicts.

The end-to-end flow is:

```
Upload CSV
    │
    ▼
Read into DataFrame (first column → index)
    │
    ▼
Validate gene set against required 12,712 input genes
    │
    ├── missing genes ──► reject, report exactly which genes are absent
    │
    └── complete ──► reorder columns to reference order  →  submatrix
                          │
                          ▼
                    Model inference (batch_size = 1)
                          │
                          ▼
              Predicted block, labelled with reference genes 12,713 → end
                          │
                          ▼
              Concatenate [ submatrix | predicted ] → final matrix
                          │
                          ▼
                    Preview → Download → Query
```

Two properties matter here. First, **column order in your file is irrelevant** — the tool reorders
your columns into the reference order before inference, because the model interprets input by
position, not by name. Second, **extra genes are harmless** — anything beyond the required set is
simply not selected.

---



## Using the interface

The app is organised as five tabs, worked through in order.

**1 · Upload & Validate** — upload your CSV, preview it, see sample and gene counts, and see how
many of your samples already appear in the MPGEM reference sample list versus how many are new.
The required reference gene list can be browsed here, and a correctly formatted example file is
available for download. The compatibility check reports success, or lists every missing gene.

**2 · Predict** — runs the model on the validated matrix and previews the merged result.

**3 · Download** — exports the complete matrix as CSV, with the file size shown.

**4 · Query** — filters results by gene, by sample, or both, with the subset downloadable separately.

**5 · Tutorial** — walkthrough video and written instructions.

---

## Input format

A CSV in which the first column holds sample IDs and every following column is a gene:

```
sample_id,A1BG,A1CF,A2M,...
sample_1,4.21,2.98,7.55,...
sample_2,3.87,3.10,7.02,...
```

Values should be normalized gene expression values. Gene names must use official HGNC symbols —
check yours with the
[HGNC Multi-Symbol Checker](https://www.genenames.org/tools/multi-symbol-checker/), since alias
symbols are the most common cause of a matrix being rejected.

---

## Output

A CSV indexed by your sample IDs, containing your original input genes followed by the predicted
genes, in reference-list order. Available in full, or as a filtered subset from the Query tab.

---

