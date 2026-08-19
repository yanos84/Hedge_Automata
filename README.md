# Hedge Automata for JSON Validation

This notebook explores **hedge automata** (tree automata that accept unranked trees, i.e. nodes with a variable number of children) as a mechanism for validating JSON documents against a structural grammar. It includes synthetic dataset generation, a from-RAM tree acceptor, and two streaming acceptors (regex-based and DFA-based) designed to validate very large JSON files with low memory overhead.

## Contents

### 1. JSON dataset generation
- **`RecursiveJsonGenerator`** — recursively builds nested JSON documents (objects/arrays/values) with configurable `depth`, `members_per_object`, `array_size`, and `recursive_ratio`.
- **`generate_file`** — generates a single JSON file from the generator and reports its size.
- **`generate_experiments`** — sweeps one parameter (`depth`, `members`, or `arrays`) over a range while holding the others fixed, writing a batch of JSON files to `json_dataset/`. Two example sweeps are included (varying `members` and `arrays` from 5 to 50).
- The generated dataset is zipped for download (`json_dataset3.zip` or any name chosen by the user).

### Dataset Download
The dataset used for the experiment can be downloaded using
```python
import kagglehub

# Download latest version
path = kagglehub.dataset_download("yanos843/json-true")

print("Path to dataset files:", path)
```
Please make sure to edit the correct path directories when using the dataset.

### 2. Hedge automaton core classes
A minimal object model for finite tree automata (FTA) over unranked trees:
- **`State`** — named state with `is_Final` / `is_Initial` flags.
- **`Symbol`** / **`Ranked_Symbol`** — alphabet symbols, with an optional fixed rank.
- **`AbstractTree`** — base tree node (symbol + children + parent) with a `is_well_formed` hook for subclasses.
- **`Fta`** (abstract) — base class holding an automaton's name and state list.
- **`HedgeTransition`** — a rule of the form `symbol(horizontal_language) -> target_state`, where `horizontal_language` is a regular language over states (i.e. it constrains the *sequence* of children's states).
- **`HedgeAutomaton`** — concrete hedge automaton: alphabet + list of `HedgeTransition` rules, with printing/debugging helpers.

### 3. Acceptors (RAM-based)
- **`HedgeAcceptor`** — bottom-up recognizer. For each tree node it recursively computes the states reachable by its children, forms the "horizontal word" of those states, and checks it against each transition's horizontal language regex. A tree is accepted if the root can reach a final state.
- **`json_to_tree` / `_convert_value` / `load_json_as_tree`** — converts a parsed JSON document (of the shape `{"document": ...}`) into an `UnrankedTree`, mapping JSON objects → `object` nodes, arrays → `array` nodes, and scalars → leaf `value` nodes.

### 4. The JSON grammar as a hedge automaton
`build_json_hedge_automaton()` defines the grammar used throughout the experiments:

| Symbol | Horizontal language | Target state | Meaning |
|---|---|---|---|
| `value` | `""` (empty) | `qV` | leaf value |
| `object` | `(qV\|qO\|qA)*` | `qO` | object = any mix of values/objects/arrays |
| `array` | `(qO\|qV)*` | `qA` | array = any mix of objects/values |
| `document` | `qO` | `qDoc` (final) | a valid document is a single top-level object |

### 5. Streaming acceptors
Built to validate JSON files far larger than available RAM, using [`ijson`](https://pypi.org/project/ijson/) for event-driven parsing instead of loading a full tree:
- **`StreamingHedgeAcceptor`** — accumulates the "horizontal word" of child state names per open container and matches it against each rule's compiled regex when the container closes.
- **`StreamingHedgeDfaAcceptorV2`** — converts each rule's regex into a DFA (via [`automata-lib`](https://pypi.org/project/automata-lib/)) and steps it incrementally as each child completes, keeping only the current DFA state per open frame. This bounds memory to `O(depth)` rather than `O(width of widest container)`, and aborts early (`HedgeRejected`) as soon as a file is known to be invalid — no need to read the rest of a multi-GB file.

### 6. Experiments & benchmarking
- Small correctness checks running each acceptor against sample files (`vsmall.json`, `bhuge.json`, etc.).
- A benchmark loop that runs `StreamingHedgeDfaAcceptorV2` over every file in a dataset directory, timing each with `time.thread_time()` and collecting results (file name, size, accepted/rejected, elapsed ms) into a `pandas.DataFrame`, exported to `hedge_dfa_acceptor_benchmark.csv`.

## Requirements

- Python 3
- `pandas`
- `ijson` (`pip install ijson`)
- `automata-lib` (`pip install automata-lib`)

## Notes

- The notebook was originally run on Kaggle; several paths reference `/kaggle/working/` and `/kaggle/input/datasets/...`. Update these paths to match your own environment before rerunning.
- Some cells are exploratory/superseded (e.g. an earlier `array(O*)` grammar comment is corrected later to `array((O|V)*)`, and a couple of early cells re-define `json_to_tree`/`load_json_as_tree` before the final versions are used). Run cells top-to-bottom in order for a consistent state.
- The two streaming acceptors assume the automaton is **unambiguous** (at most one matching transition per symbol).
