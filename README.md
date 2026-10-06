# ManuVerity

**ManuVerity** is a local, rule-based pre-submission checker for scientific manuscripts. Before you send a paper to a journal, it helps you catch the mechanical mistakes that slip through spell-check: figures and tables that are cited but not captioned (or captioned but never cited), reference list mismatches, broken Word cross-references, tracked changes, placeholders, and other “should never reach a reviewer” issues.

Everything runs on your machine. **No AI model reads your text**, uploads are **not stored**, and nothing is sent to external services unless you choose to host the web app on a network interface yourself.

The Python package that implements ManuVerity is named `manuscript_checker` (this folder). The web UI may still say “Submission check” in places; functionally it is the same tool.

<p align="center">
  <img src="logo.svg" alt="ManuVerity logo" width="320">
</p>

---

## What ManuVerity does

ManuVerity ingests one or more document files, splits them into paragraphs (“blocks”), understands document structure (abstract, main text, reference list, headings), and runs four classes of checks:

| Area | What you get |
|------|----------------|
| **Figures** | Caption ↔ citation consistency, duplicates, numbering gaps, out-of-order first citations, supplementary/extended-data labels |
| **Tables** | Same logic as figures, kept separate (Figure 1 and Table 1 never collide) |
| **References** | Numeric or author–year style; uncited entries, orphan citations, ordering, duplicates, mixed styles |
| **Proofing** | Broken cross-refs, tracked changes/comments, placeholders, encoding glitches, optional journal limits, abbreviations, repeated words |

Results are available through a **web dashboard** (readiness score, tabs, citation-flow views, copyable issue lists) or a **CLI** suitable for scripts and CI (exit code 1 when error-level issues exist).

Supported inputs: **`.docx`** (recommended), **`.pdf`**, **`.txt`**, **`.md`**.

---

## Requirements

- **Python 3.10+** (developed and tested on 3.11 and 3.13)
- Dependencies listed in `requirements.txt`:
  - **FastAPI** + **Uvicorn** — web server
  - **python-multipart** — file uploads
  - **python-docx** — Word structure, superscripts, tables, tracked changes
  - **pdfplumber** — PDF text extraction and layout heuristics

---

## Installation

### 1. Get the code

Clone or copy the `manuscript_checker` directory. For command-line use, Python must be able to import the package as `manuscript_checker`. The usual layout is a parent folder that **contains** this directory:

```text
your-project/
└── manuscript_checker/    ← this folder (package root)
    ├── run.py
    ├── cli.py
    ├── server.py
    ├── requirements.txt
    └── ...
```

If you only have the inner folder, place it inside any parent directory (for example `~/tools/manuscript_checker/`) and run commands from that parent, as shown below.

### 2. Create a virtual environment (recommended)

```bash
cd /path/to/parent-of-manuscript_checker
python3 -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install -r manuscript_checker/requirements.txt
```

### 4. Verify installation

From the parent directory:

```bash
python -m pytest manuscript_checker -q
```

You should see all tests pass (with a small number of skips for optional fixtures).

---

## Quick start

### Web UI (recommended for interactive review)

From **any** working directory:

```bash
python manuscript_checker/run.py
```

Then open **http://127.0.0.1:8000** in your browser.

Optional flags:

```bash
python manuscript_checker/run.py --host 127.0.0.1 --port 8501
```

Use `--host 0.0.0.0` only if you intentionally want other machines on your network to reach the server (files would still be processed locally on that host).

**What to do in the UI**

1. Drop or browse for files: main manuscript, optional figure/table legends file, optional supplementary material.
2. Assign each file a **role** (Manuscript, Legends, Supplementary). Filename heuristics suggest roles automatically (e.g. names containing `supp`, `SI`, `MOESM`, `legend`).
3. Choose **reference style**: Auto-detect, Numeric, or Author–year.
4. Optionally set **journal limits** (abstract words, main-text words, figure/table/reference counts).
5. Click **Run check** and review the readiness summary and issue tabs.

Upload size limit: **50 MB per file** (enforced by the server).

### Command line (automation / CI)

Run from the directory that contains the `manuscript_checker` package:

```bash
python -m manuscript_checker.cli main_paper.docx
python -m manuscript_checker.cli main_paper.docx --figures legends.docx --supp supplementary.docx
python -m manuscript_checker.cli paper.docx --ref-style author-year
python -m manuscript_checker.cli paper.docx --json > report.json
```

- **Exit code `0`**: no error-level issues  
- **Exit code `1`**: at least one error-level issue (warnings alone do not fail the run)

---

## How to prepare your files

ManuVerity works best when you mirror how journals expect materials to be split:

| Role | Typical content | Effect on checks |
|------|-----------------|------------------|
| **Main manuscript** | Body text; captions may live here or in a separate legends file | Body citations are matched against captions everywhere you uploaded |
| **Figures / legends** | Figure and table legends only | Text here is treated as legend context, not as body citations for figures/tables |
| **Supplementary** | SI PDF/DOCX | Unprefixed “Figure 1” in SI is treated as supplementary Figure 1; `Figure S1` and `Supplementary Figure 1` align |

If supplementary material is **cited in the main text** but you **do not** upload an SI file, those items appear as **unchecked** (one warning), not as missing captions.

---

## Figures and tables (detailed)

### Item status

| Status | Meaning |
|--------|---------|
| `OK` | Caption found and cited in body text |
| `UNCITED` | Caption found, never cited in body (mentions inside other captions do not count) |
| `MISSING` | Cited in the text, no caption found in any uploaded file |
| `UNCHECKED` | Supplementary item cited, but no supplementary file was uploaded |

### Additional figure/table checks

- Duplicate captions  
- Numbering gaps  
- Items first cited out of numerical order  
- Supplementary items cited only in the SI  
- Separate namespaces for figures vs tables  
- A figure mentioned inside a **table** caption counts as legend text, not a body citation  

### Citation patterns (examples)

- `Fig. 2`, `Figs. 1 and 4`, `Figures 2–4`, `Fig. 2a–d`, `Figures 1 and S2`  
- `Supplementary` / `Suppl.` / `SI Fig. 4`, `Extended Data Fig. 1`  
- `Table 3`, `Tab. 3`, `Tables S1–S3`  
- Thesis-style `Figure 3.2`  

### Caption patterns (examples)

- `Figure 1.`, `Fig. 2 |`, `Table 1:`, `Figure S6a.`, `[Figure S6] Title`  
- Word **Caption** style paragraphs  
- `Figure 2 shows…` is treated as a **citation**, not a caption  

Captions after manual line breaks, inside Word tables, and in text boxes are detected. “List of Figures/Tables” entries are ignored. If something is cited but no caption is recognized, paragraphs that **start** with the label may be reported as a possible unrecognized caption.

---

## References (detailed)

Style is **auto-detected** unless you set `--ref-style` or pick a style in the UI.

### Numeric styles

- Brackets: `[3]`, `[1–4, 7]`  
- Parentheses `(3)` only when parentheses are the **dominant** citation style (to avoid confusing `(1)` lists)  
- Superscripts: Word formatting, PDF size/position, Unicode ¹²  
- `ref. 5` / `refs 3–5`  

**Filtered out** (not treated as citations): affiliation superscripts on the title page, powers of ten, isotopes, chemistry notation (`sp³`, `R²`, `Å³`), issue numbers like `295(2)`, and many gene/name superscript patterns.

If no reference list is found, that is reported **once**, not as one error per in-text citation. Lists may be plain numbered text or Word list numbering.

Reports include: uncited entries, citations beyond the list length, references not numbered in order of first citation (skipped when the list is clearly alphabetical), duplicate entries (DOI or text), numbering gaps, mixed citation styles.

### Author–year styles

- `(Smith et al., 2019)`, `Smith and Jones (2019)`  
- Compressed forms: `(e.g., Smith 2018, 2019a, b; Lee, 2020)`  
- Particles and accents normalized for matching (`Müller` / `Muller`)  
- Organisation acronyms when the list supports them  

Matching is driven by **first-author surnames and years**; prose like “In (2019)” is not flagged. Second-author names in “Smith and Jones” are not verified.

### Finding the bibliography

Headings such as References, Bibliography, Literature cited, Methods references, Supplementary references; also Word bibliography styles (Zotero, EndNote, Mendeley). A citation in the SI resolves against the SI list first, then the main list.

---

## Proofing (detailed)

| Check | Examples |
|-------|----------|
| Broken cross-references | `Error! Reference source not found.`, `Error! Bookmark not defined.`, LaTeX `Figure ??`, `[?]` |
| Tracked changes & comments | Unaccepted insertions/deletions, comments, highlights (DOCX) |
| Placeholders | `TODO`, `TBD`, `XX`, `[ref]`, `[citation needed]`, `???`, lorem ipsum |
| Garbled characters | Wrong encoding: `Î²` vs β, `â€™` vs ’ |
| Required statements | Data availability, author contributions, competing interests, funding (plus acknowledgements, code, ethics) |
| Journal limits | Optional caps on abstract words, main-text words, figures, tables, references |
| Abbreviations | Used before definition, defined twice, never used again, defined only in abstract |
| Repeated words | “the the” (with sensible exceptions) |

**Main-text word count** excludes the title page, abstract, headings, captions, table cells, and the reference list.

---

## Architecture (for developers)

```text
manuscript_checker/
├── run.py           # Starts Uvicorn; adds parent dir to sys.path
├── server.py        # FastAPI app: GET /, POST /api/check
├── cli.py           # argparse CLI; JSON or human-readable issues
├── extract.py       # DOCX/PDF/TXT/MD → Block list; roles; Word metadata
├── structure.py     # Abstract, sections, reference-list boundaries
├── labels.py        # Figure/table caption ↔ citation logic
├── references.py    # Numeric and author–year reference checking
├── proofing.py      # Hygiene checks and journal limits
├── analyze.py       # Orchestrates all checks → Analysis
├── report.py        # Serializes Analysis for UI/CLI (--json)
├── static/index.html # Single-page web client
└── tests/           # pytest suite
```

**Data flow:** uploaded bytes → `load_document()` → `analyze()` → `to_dict()` → JSON consumed by the browser or printed by the CLI.

**Privacy model:** the web handler reads each upload into memory (or rejects if too large), runs analysis in-process, returns JSON, and does not write manuscripts to disk. The UI may store **UI preferences** (theme, limits) in browser `localStorage` only.

---

## Running tests

From the parent of `manuscript_checker/`:

```bash
python -m pytest manuscript_checker
python -m pytest manuscript_checker/tests/test_references.py -v
```

Tests use synthetic fixtures and selected real-world-style documents where present.

---

## Known limitations

- **No LaTeX source** (`.tex`) yet — `\label`, `\ref`, and `\cite` need a dedicated parser. Use compiled PDF or DOCX export for now.  
- **PDF is reconstructed**, not semantic: columns, scans, and unusual layouts can mis-segment paragraphs or reference lists. **DOCX is always more reliable.**  
- **Author–year** matching uses first author + year; it does not validate full author lists.  
- **Abbreviation** checks only see `Long form (ABBR)` definitions and skip common acronyms (DNA, PCR, …).  
- Checks are **label- and pattern-based**. ManuVerity does not judge whether a caption matches the right panel or whether a reference supports a claim.

---

## Logo assets

- `logo.svg` — vector logo (Manu**Verity** wordmark)  
- `logo.eps` — print/export variant  

---

## Summary

| Goal | Command |
|------|---------|
| Interactive review | `python manuscript_checker/run.py` → http://127.0.0.1:8000 |
| Batch / CI gate | `python -m manuscript_checker.cli paper.docx` |
| Machine-readable output | Add `--json` to the CLI |
| Install deps | `pip install -r manuscript_checker/requirements.txt` |

ManuVerity is built for authors who want a fast, private sanity pass before submission — deterministic rules, full control on your own hardware, and a clear list of what to fix before a human editor or reviewer sees the draft.
