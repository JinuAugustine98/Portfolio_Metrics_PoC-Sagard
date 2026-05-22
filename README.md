# Portfolio Metrics Extraction PoC

Notebook: `portfolio_metrics_extraction_poc.ipynb`

This repo is a small proof of concept for extracting portfolio-company metrics from PDF reporting packages. The output is a long table with evidence, confidence, and review flags, plus a SQLite copy for later analysis.

## Runtime

- Python 3.12.5

## Dependencies

- Python packages: `pdfplumber`, `pypdf`, `pdf2image`, `pytesseract`, `langchain`, `langgraph`, `pandas`, `matplotlib`, `pydantic`, `scikit-learn`
- System packages: `tesseract`, `poppler`
- Ollama CLI: required for the `glm-ocr` fallback
- SQLite: built into Python through `sqlite3`; no extra pip install is required

## Stack

- Direct PDF text extraction: `pdfplumber` with `pypdf` as a fallback
- OCR fallback: `pdf2image` + `pytesseract`
- Vision fallback: `glm-ocr` through Ollama, using the local API with the CLI path available for manual checks
- Workflow: LangChain tools and LangGraph routing
- Forecasting baseline: scikit-learn Ridge regression on report order
- Storage: pandas and built-in SQLite (`sqlite3`)
- Charts: matplotlib

## Processing Order

1. Parse the embedded PDF text.
2. Run OCR if the text layer is weak or missing.
3. Use Ollama `glm-ocr` on page images when direct parsing and OCR still leave the page empty. The notebook keeps it enabled by default, but the graph only routes into it for those empty or near-empty cases.
4. Validate against the prior period for the same company.
5. Persist the long table to SQLite, then run the simple forecast baseline once there is enough history.

The notebook keeps the fallback order explicit instead of hiding it inside a generic agent. The downstream validation and forecast stay separate so the extraction path is easy to inspect.

## Ollama GLM-OCR

The vision branch uses Ollama with `glm-ocr`. On Linux, install Ollama with:

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama serve
```

On macOS, install from the official download page or the desktop app.

Pull the model once and smoke-test it with:

```bash
ollama pull glm-ocr
ollama run glm-ocr Text Recognition: ./image.png
```

The notebook calls the local Ollama API, and the same model can be checked manually with the `ollama run` command above.

## Tesseract install

macOS:

```bash
brew install tesseract poppler
```

Ubuntu / Debian:

```bash
sudo apt-get update
sudo apt-get install -y tesseract-ocr poppler-utils
```

## Run

Open the notebook in Jupyter Lab or Notebook and run all cells. Place the challenge PDFs in `Tech Challenge Package/` at the repo root.

The PDF package is not included in this public repo because it contains confidential reporting files, and the folder is ignored by Git. Notebook outputs are also not committed so the repository does not expose derived data from the challenge package. You can also point the notebook at a local copy by setting `TECH_CHALLENGE_PACKAGE_DIR`.

If that folder is not present, the notebook raises an error.

On the first run, Ollama may need to download `glm-ocr` automatically. A manual check looks like this:

```bash
ollama pull glm-ocr
ollama run glm-ocr Text Recognition: ./image.png
```

The notebook calls the local API behind the scenes, so the same model is reused once it is pulled.

## Assumptions

- One PDF usually maps to one company and one reporting period.
- Currency-like metrics without a currency symbol are treated as USD.
- Percentages are stored as displayed, for example `5.8%` becomes `5.8` with unit `%`.
- Portfolio summary PDFs are not split into sub-documents in this version.

## Limits

- Label-based extraction will miss some layouts.
- OCR quality depends on local Tesseract and Poppler.
- Ollama GLM-OCR is a fallback, not a replacement for clean text extraction. It is only invoked when direct parsing and OCR do not provide enough signal.
- The validation rules are intentionally simple: missing-after-history and >50% quarter-over-quarter change.

## What’s in the notebook

- metric schema and alias definitions
- regex and normalization helpers
- direct PDF extraction
- OCR fallback
- Ollama GLM-OCR fallback
- LangGraph routing
- batch folder processing
- SQLite persistence
- historical checks
- review table and basic charts
- SQLite-backed forecasting baseline and forecast charts
