# Hedge Automata for JSON Validation

This notebook explores **hedge automata** (tree automata that accept unranked trees, i.e. nodes with a variable number of children) as a mechanism for validating JSON documents against a structural grammar. It includes synthetic dataset generation, a from-RAM tree acceptor, and two streaming acceptors (regex-based and DFA-based) designed to validate very large JSON files with low memory overhead.

## Contents

### 1. JSON dataset generation
- **`RecursiveJsonGenerator`** — recursively builds nested JSON documents (objects/arrays/values) with configurable `depth`, `members_per_object`, `array_size`, and `recursive_ratio`.
- **`generate_file`** — generates a single JSON file from the generator and reports its size.
- **`generate_experiments`** — sweeps one parameter (`depth`, `members`, or `arrays`) over a range while holding the others fixed, writing a batch of JSON files to `json_dataset/`. Two example sweeps are included (varying `members` and `arrays` from 5 to 50).
- The generated dataset is zipped for download (`json_dataset3.zip` or any name chosen by the user).




# Hedge Automata for JSON Validation

This project explores **hedge automata** (tree automata that accept unranked trees, i.e. nodes with a variable number of children) as a mechanism for validating JSON documents against a structural grammar, and benchmarks two acceptor implementations against each other on large synthetic datasets.

It's split across two notebooks that form a pipeline:

1. **`hedge-automata.ipynb`** — defines the hedge automaton model, generates synthetic JSON datasets, implements the acceptors, and runs the benchmarks that produce raw timing CSVs.
2. **`All_hedge_Expreiment.ipynb`** — loads those CSVs and produces the comparison plots and summary statistics used for analysis/write-up.

---

## Notebook 1: `hedge-automata.ipynb` — implementation & benchmarking

### 1. JSON dataset generation
- **`RecursiveJsonGenerator`** — recursively builds nested JSON documents (objects/arrays/values) with configurable `depth`, `members_per_object`, `array_size`, and `recursive_ratio`.
- **`generate_file`** — generates a single JSON file from the generator and reports its size.
- **`generate_experiments`** — sweeps one parameter (`depth`, `members`, or `arrays`) over a range while holding the others fixed, writing a batch of JSON files to `json_dataset/`. Example sweeps are included (varying `members` and `arrays` from 5 to 50) — these correspond to the "by_depth", "by_members", and "by_arrays" datasets consumed later in Notebook 2.
- The generated dataset is zipped for download (`json_dataset3.zip`).

### 2. Dataset Download
The dataset used for the experiment can be downloaded using
```python
import kagglehub

# Download latest version
path = kagglehub.dataset_download("yanos843/json-true")

print("Path to dataset files:", path)
```
Please make sure to edit the correct path directories when using the dataset.

### 3. Hedge automaton core classes
A minimal object model for finite tree automata (FTA) over unranked trees:
- **`State`** — named state with `is_Final` / `is_Initial` flags.
- **`Symbol`** / **`Ranked_Symbol`** — alphabet symbols, with an optional fixed rank.
- **`AbstractTree`** — base tree node (symbol + children + parent) with a `is_well_formed` hook for subclasses.
- **`Fta`** (abstract) — base class holding an automaton's name and state list.
- **`HedgeTransition`** — a rule of the form `symbol(horizontal_language) -> target_state`, where `horizontal_language` is a regular language over states (i.e. it constrains the *sequence* of children's states).
- **`HedgeAutomaton`** — concrete hedge automaton: alphabet + list of `HedgeTransition` rules, with printing/debugging helpers.

### 4. Acceptors (RAM-based)
- **`HedgeAcceptor`** — bottom-up recognizer. For each tree node it recursively computes the states reachable by its children, forms the "horizontal word" of those states, and checks it against each transition's horizontal language regex. A tree is accepted if the root can reach a final state.
- **`json_to_tree` / `_convert_value` / `load_json_as_tree`** — converts a parsed JSON document (of the shape `{"document": ...}`) into an `UnrankedTree`, mapping JSON objects → `object` nodes, arrays → `array` nodes, and scalars → leaf `value` nodes.

### 5. The JSON grammar as a hedge automaton
`build_json_hedge_automaton()` defines the grammar used throughout the experiments:

| Symbol | Horizontal language | Target state | Meaning |
|---|---|---|---|
| `value` | `""` (empty) | `qV` | leaf value |
| `object` | `(qV\|qO\|qA)*` | `qO` | object = any mix of values/objects/arrays |
| `array` | `(qO\|qV)*` | `qA` | array = any mix of objects/values |
| `document` | `qO` | `qDoc` (final) | a valid document is a single top-level object |

### 6. Streaming acceptors
Built to validate JSON files far larger than available RAM, using [`ijson`](https://pypi.org/project/ijson/) for event-driven parsing instead of loading a full tree:
- **`StreamingHedgeAcceptor`** (regex-based) — accumulates the "horizontal word" of child state names per open container and matches it against each rule's compiled regex when the container closes.
- **`StreamingHedgeDfaAcceptorV2`** (DFA-based) — converts each rule's regex into a DFA (via [`automata-lib`](https://pypi.org/project/automata-lib/)) and steps it incrementally as each child completes, keeping only the current DFA state per open frame. This bounds memory to `O(depth)` rather than `O(width of widest container)`, and aborts early (`HedgeRejected`) as soon as a file is known to be invalid — no need to read the rest of a multi-GB file.

These two acceptors are the "DFA-based" and "Regex-based" methods compared throughout Notebook 2's plots.

### 7. Experiments & benchmarking
- Small correctness checks running each acceptor against sample files (`vsmall.json`, `bhuge.json`, etc.).
- A benchmark loop that runs an acceptor over every file in a dataset directory, timing each with `time.thread_time()` and collecting results (file name, size, accepted/rejected, elapsed ms) into a `pandas.DataFrame`, exported to CSV (e.g. `hedge_dfa_acceptor_benchmark.csv`, `hedge_acceptor_benchmark.csv`). Re-running this loop against the depth/members/arrays/rejection datasets from section 1 (with the output CSV renamed accordingly) is what produces the various input files Notebook 2 expects, such as `dfa_by_depth.csv`, `regex_by_members.csv`, `dfa_reject.csv`, etc.

#### Requirements
- Python 3, `pandas`
- `ijson` (`pip install ijson`)
- `automata-lib` (`pip install automata-lib`)

#### Notes
- The notebook was originally run on Kaggle; several paths reference `/kaggle/working/` and `/kaggle/input/datasets/...`. Update these paths to match your own environment before rerunning.
- Some cells are exploratory/superseded (e.g. an earlier `array(O*)` grammar comment is corrected later to `array((O|V)*)`, and a couple of early cells re-define `json_to_tree`/`load_json_as_tree` before the final versions are used). Run cells top-to-bottom in order for a consistent state.
- The two streaming acceptors assume the automaton is **unambiguous** (at most one matching transition per symbol).

---

## Notebook 2: `All_hedge_Expreiment.ipynb` — analysis & plotting

This notebook does **not** run any automata — it's a pure analysis layer over the `time_ms` benchmark CSVs exported by Notebook 1. Each section loads one or more CSVs, merges the DFA-based and regex-based results, and produces publication-style comparison plots (`matplotlib`/`seaborn`, serif fonts, log-log axes where relevant, saved as both `.png` and `.pdf`). It was written for Google Colab (paths under `/content/...`, includes a Google Drive mount cell).

| Section | Input CSV(s) | What it shows |
|---|---|---|
| **Overall experiment** | `hedge_dfa_acceptor_benchmark.csv`, `hedge_acceptor_benchmark.csv` | DFA vs. regex runtime vs. file size on **accepted** files, plotted for all files, small files (<100 MB), and large files (≥100 MB) separately |
| **Rejection experiment** | `experiments/rejection/dfa_reject.csv`, `regex_reject.csv` | DFA vs. regex runtime vs. file size on **rejected** (invalid) files |
| **By-depth experiment** | `experiments/by_depth/dfa_by_depth.csv`, `regex_by_depth.csv` | Runtime vs. file size when sweeping JSON nesting **depth** |
| **By-members experiment** | `experiments/by_members/dfa_by_members.csv`, `regex_by_members.csv` | Runtime vs. file size when sweeping object **branching factor** (`members_per_object`) |
| **By-arrays experiment** | `experiments/by_arrays/dfa_by_array.csv`, `regex_by_array.csv` | Runtime vs. file size when sweeping array **branching factor** (`array_size`) |
| **Combined rejection plot** | `reject.csv` (pre-aggregated) | Three-way comparison: DFA reject vs. regex reject vs. DFA accept on similar-size valid files — highlights the DFA acceptor's early-abort speed advantage |
| **Throughput** | benchmark CSVs + `json_symbol_counts.csv` | Normalizes runtime by `total_symbols` processed to get symbols/ms throughput, so acceptor speed can be compared independent of raw file size |
| **Drive mount** | — | Mounts Google Drive at `/content/drive` (Colab-only) so CSVs stored there are accessible |
| **Peak / mean / median / std throughput** | — (uses `long_throughput_df` from the Throughput section) | Summary statistics on the computed throughput values, for reporting headline numbers |

#### Requirements
- Python 3, `pandas`, `numpy`, `matplotlib`, `seaborn`
- Google Colab (for the Drive-mount cell) — optional if you're running locally and already have the CSVs on disk

#### Notes
- This notebook must be run **after** Notebook 1 has produced the required CSVs; several cells `raise` with a clear error message if an expected file (e.g. `merged_benchmark.csv`) is missing.
- File paths are Colab-style (`/content/...`); update them if running elsewhere.
- The **Throughput** section requires `symbols_file_path` (pointing to `json_symbol_counts.csv`) to be updated to your actual location before running.
- An annotated copy of this notebook (`All_hedge_Experiment_annotated.ipynb`) is available with a markdown explanation added above every section/cell, useful as a standalone walkthrough of the analysis without needing this README alongside it.

---

## End-to-end workflow

1. Run **Notebook 1** to generate JSON datasets (overall size sweep, plus depth/members/arrays/rejection sweeps) and benchmark both acceptors against each dataset, exporting the resulting CSVs.
2. Place/upload those CSVs where **Notebook 2** expects them (`/content/...` paths, or update the paths to match your setup).
3. Run **Notebook 2** to produce the comparison plots and throughput/summary statistics.

