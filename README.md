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

## Logic and functions

All logic lives in a single Streamlit script. The functions divide into four groups: the custom
model activation, asset loaders, the two core computation functions, and the interface layer.

### Custom activation

```python
def custom_activation(x):
    return K.sigmoid(x) * 12

get_custom_objects().update({'custom_activation': Activation(custom_activation)})
```

The network's output layer uses a scaled sigmoid — a standard sigmoid squashed into `(0, 1)` and
then multiplied by 12, giving an output range of `(0, 12)` that matches the dynamic range of the
normalized expression values the model was trained on.

Because this activation is not part of Keras, it must be **registered in the custom-objects registry
before `load_model()` is called**. This registration happens at import time, at module level, so it
is guaranteed to run before any model loading. Removing or renaming it will cause deserialization of
the `.h5` file to fail with an unknown-object error.


```

| Condition | `status` | `submatrix` | Meaning |
|---|---|---|---|
| `reference_genes == user_genes` | `"equal"` | `user_matrix[reference_genes_pred]` | Exact match |
| `reference_genes ⊆ user_genes` | `"extra"` | `user_matrix[reference_genes_pred]` | Superset — extra columns dropped |
| otherwise | `"missing"` | `None` | Rejected; `missing_genes` holds the sorted difference |

The key line in both accepting branches is the indexing expression `user_matrix[reference_genes_pred]`.
Selecting with a **list** rather than a set is what performs the reordering: pandas returns columns
in the order of the list you pass, so the output is guaranteed to be in reference order regardless
of how the input file was arranged. This is why `reference_genes_pred` must remain an ordered list
throughout and must never be converted to a set outside the comparison.

On the `"missing"` branch, the set difference `reference_genes - user_genes` is sorted before being
returned so the report of absent genes is presented alphabetically rather than in hash order.

### `predict_and_merge(submatrix, reference_genes, model)`

This runs inference and reassembles the result.

```python
input_matrix    = submatrix.to_numpy()
predicted       = model.predict(input_matrix, batch_size=1, verbose=0)
predicted_genes = reference_genes[len(submatrix.columns):]
predicted_df    = pd.DataFrame(predicted, columns=predicted_genes, index=submatrix.index)
return pd.concat([submatrix, predicted_df], axis=1)
```

Four things happen, in order:

1. **`to_numpy()`** strips the DataFrame down to a bare numeric array. The model expects positions,
   not labels — column names are deliberately discarded here and reattached later.
2. **`model.predict(..., batch_size=1)`** runs one sample at a time. This keeps peak memory low,
   at the cost of runtime scaling roughly linearly with sample count.
3. **`reference_genes[len(submatrix.columns):]`** derives the output gene labels by slicing the full
   reference list from the end of the input block onwards. Note that the offset comes from the
   *submatrix width*, not a hard-coded 12712 — so the labelling stays correct if the reference list
   or input size is ever changed. This slice is the only thing mapping model output columns back to
   gene identities, and it relies entirely on the reference list ordering being stable.
4. **`pd.concat(..., axis=1)`** joins measured and predicted blocks side by side. Both frames carry
   the same `submatrix.index`, so sample alignment is preserved and no row can be mismatched.

### State and interface layer

Results are passed between tabs through `st.session_state`, since each Streamlit rerun re-executes
the whole script:

| Key | Set in | Consumed by |
|---|---|---|
| `submatrix` | Upload & Validate, on a successful check | Predict |
| `reference_genes` | Upload & Validate | Predict |
| `merged_df` | Predict | Download, Query |
| `show_contact` | Contact button | Contact popup |

Each tab guards on the presence of its key and shows a warning instead of failing if the previous
step has not been completed. On a failed validation, `submatrix` is explicitly deleted from session
state, which prevents a stale, previously valid matrix from being predicted on after a bad upload.

The remaining functions — `get_img_as_base64()`, `load_css_and_background()` and `contact_popup()` —
handle appearance only. The background image is base64-encoded and inlined into a CSS rule, and the
whole stylesheet is injected with `unsafe_allow_html=True`. If the background file is absent, the
app warns and falls back to a solid dark background rather than failing.



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

## Contact and support

- Email: [shandar@sciwhylab.org](mailto:shandar@sciwhylab.org)
- Report a problem: [open an issue](https://github.com/SougataJana/MPGEM_PREDICTION_TOOL/issues/new)
- Source code: [SougataJana/MPGEM_PREDICTION_TOOL](https://github.com/SougataJana/MPGEM_PREDICTION_TOOL)
- Lab website: [shandarslab.org](http://shandarslab.org)

A contact panel with these links is also available from the button in the corner of the app.
