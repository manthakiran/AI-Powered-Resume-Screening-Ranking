# ATS Project — AI-Powered Resume Screening & Ranking

An Applicant Tracking System (ATS) pipeline that ingests a batch of candidate resumes, indexes them in a vector database, retrieves the most relevant candidates for a given job description, and uses a Large Language Model (LLM) to score and rank them.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-RAG-1C3C3C)
![FAISS](https://img.shields.io/badge/Vector%20Store-FAISS-0467DF)
![HuggingFace](https://img.shields.io/badge/Embeddings-HuggingFace-FFD21E?logo=huggingface&logoColor=black)

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Sample Output](#sample-output)
- [Dependencies](#dependencies)
- [Limitations and Future Improvements](#limitations-and-future-improvements)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

Recruiters often receive dozens or hundreds of resumes for a single opening. Reading each one is slow, and keyword search misses candidates who describe the same skills in different words.

This project tackles that with **Retrieval-Augmented Generation (RAG)**:

1. Resumes (PDF and DOCX, including scanned ones) are converted to clean text.
2. The text is split into chunks and embedded into a **FAISS** vector index using a Sentence-Transformers model.
3. A job description is used as the query to retrieve the most semantically relevant resume chunks.
4. Retrieved chunks are grouped by candidate, and an **LLM scores each candidate from 0 to 100** against the job description.
5. Candidates are printed as a ranked shortlist with their contact email.

The whole workflow lives in a single notebook, `ATS_Project.ipynb`.

## How It Works

```
Tracker.xlsx ─┐
              ├─► Match CV files to candidates (by Excel row number)
downloaded_cvs/ ┘            │
                             ▼
              Extract text (PyMuPDF → OCR fallback with Tesseract; python-docx for DOCX)
                             │
                             ▼
              Clean text, drop empty files, de-duplicate by email
                             │
                             ▼
              Split into chunks (800 characters, 100 overlap)
                             │
                             ▼
              Embed with all-mpnet-base-v2  ──►  FAISS index (faiss_cv_index/)
                             │
Job description ─────────────┤
                             ▼
              MMR retrieval (k=60 of 120 fetched) ─► group chunks by candidate
                             │
                             ▼
              Top 15 candidates ─► LLM scores each 0–100 (JSON output, with retries)
                             │
                             ▼
                      Ranked shortlist
```

The notebook cells map to these stages:

| Stage | What the notebook does |
| --- | --- |
| Imports | Loads all libraries (and an optional Windows Tesseract path) |
| Configuration | Sets `EXCEL_FILE` and `CV_FOLDER` |
| Candidate matching | Reads the tracker and links each CV file to a name and email |
| Text extraction | `extract_text()` reads PDF (with OCR fallback) and DOCX files |
| Text cleaning | `clean_extracted_text()` repairs encoding artifacts and broken spacing |
| Resume assembly | Builds the `resumes` list; skips duplicate emails and empty files |
| Chunking | `RecursiveCharacterTextSplitter` creates `all_documents` with name/email metadata |
| Validation | Sanity checks on counts, metadata and random chunk samples |
| Indexing | Embeds chunks and saves the FAISS index to disk |
| LLM client | Connects to an OpenAI-compatible endpoint (Gemini by default) |
| Retrieval | MMR search using the job description as the query |
| Scoring | Scores each of the top 15 candidates and handles rate limits |
| Ranking | Sorts and prints the final ranking |

## Features

- **Multi-format ingestion:** reads PDF and DOCX resumes
- **OCR fallback:** pages with little or garbled text are rasterized and read with Tesseract, so scanned resumes still work
- **Text repair:** fixes common encoding artifacts and character-per-line extraction problems
- **Data hygiene:** skips duplicate emails and empty documents
- **Metadata-aware chunks:** every chunk carries the candidate's name and email, so results always trace back to a person
- **Semantic search:** local embeddings (`all-mpnet-base-v2`) find relevant candidates even when the wording differs from the job description
- **Diverse retrieval:** Maximal Marginal Relevance (MMR) reduces near-duplicate chunks in the results
- **LLM scoring:** each candidate receives a 0–100 fit score as strict JSON
- **Resilient API calls:** up to 5 retries per candidate, with automatic back-off on rate-limit (HTTP 429) errors
- **Reusable index:** the FAISS index is saved to disk, so new job descriptions can be matched without re-processing resumes

## Tech Stack

| Component | Technology |
| --- | --- |
| Language | Python 3 (Jupyter Notebook) |
| Data handling | pandas, openpyxl |
| PDF parsing | PyMuPDF (`fitz`) |
| DOCX parsing | python-docx |
| OCR | Tesseract via pytesseract, Pillow |
| Chunking | LangChain `RecursiveCharacterTextSplitter` |
| Embeddings | `sentence-transformers/all-mpnet-base-v2` via `langchain-huggingface` (CPU) |
| Vector store | FAISS via `langchain-community` |
| LLM | Gemini, accessed through the OpenAI-compatible API using the `openai` SDK |

## Project Structure

```
ATS_Project/
├── ATS_Project.ipynb        # Main notebook containing the entire pipeline
├── Tracker.xlsx             # Candidate tracker (input; you provide this)
├── downloaded_cvs/          # Resume files, PDF / DOCX (input; you provide these)
│   ├── jane doe_2.pdf
│   ├── john smith_3.docx
│   └── ...
├── faiss_cv_index/          # Generated vector index (created by the notebook)
│   ├── index.faiss
│   └── index.pkl
└── README.md
```

### Input conventions

The notebook links resumes to candidates using the **Excel row number**, so the file names and tracker must follow these rules:

- **`Tracker.xlsx`** has a header row. Column 1 is the candidate **name** and column 2 is the **email**. Columns are read by position, not by header name.
- Each resume file is named **`<name>_<excel_row>.<pdf|docx>`**, where `<excel_row>` is the candidate's row in the spreadsheet (the first data row is row 2). For example, the candidate in row 5 of the tracker has a file like `jane doe_5.pdf`.

> The notebook expects the resumes to already be downloaded. The script that originally downloaded them is not part of this notebook, so you'll need to place the files in `downloaded_cvs/` yourself, following the naming above.

## Installation

### Prerequisites

- **Python 3.10 or newer** (the notebook was developed on Python 3.12 / 3.13)
- **Jupyter Notebook** or JupyterLab (or VS Code with the Jupyter extension)
- **Tesseract OCR** installed on your system (needed for scanned PDFs)
- An **API key** for an OpenAI-compatible LLM endpoint (the notebook uses Google's Gemini endpoint)

### 1. Get the code

```bash
git clone https://github.com/manthakiran/ATS_Project.git
cd ATS_Project
```

### 2. Create and activate a virtual environment

```bash
python -m venv .venv

# macOS / Linux
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1
```

### 3. Install the Python packages

```bash
pip install pandas openpyxl pypdf python-docx pillow pytesseract pymupdf \
            langchain-text-splitters langchain-huggingface langchain-community \
            langchain-core sentence-transformers faiss-cpu openai requests \
            jupyter
```

The first run downloads the embedding model (roughly 400 MB) from the Hugging Face Hub.

### 4. Install Tesseract OCR

```bash
# Debian / Ubuntu
sudo apt install tesseract-ocr

# macOS (Homebrew)
brew install tesseract
```

**Windows:** install Tesseract from the [UB Mannheim builds](https://github.com/UB-Mannheim/tesseract/wiki), then uncomment and adjust this line in the first notebook cell:

```python
pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"
```

> If Tesseract is missing, the notebook does not stop. Pages that need OCR will print an error and fall back to whatever text could be extracted directly.

### 5. Add your data

Place `Tracker.xlsx` next to the notebook and put the resumes in `downloaded_cvs/`, following the [input conventions](#input-conventions).

## Configuration

| Setting | Where | Default | Description |
| --- | --- | --- | --- |
| `EXCEL_FILE` | Configuration cell | `Tracker.xlsx` | Path to the candidate tracker |
| `CV_FOLDER` | Configuration cell | `downloaded_cvs` | Folder containing resumes |
| `chunk_size` / `chunk_overlap` | Chunking cell | `800` / `100` | Characters per chunk and overlap |
| Embedding model | Indexing cell | `sentence-transformers/all-mpnet-base-v2` | Runs on CPU with normalized embeddings |
| `k` / `fetch_k` | Retrieval cell | `60` / `120` | Chunks returned / candidates considered by MMR |
| Top candidates | Grouping cell | `15` | Number of candidates sent to the LLM |
| LLM model | Scoring cell | as set in the notebook | Any model your endpoint supports |
| Request pacing | Scoring cell | `time.sleep(5)` | Pause between candidates to respect rate limits |

### API key

The notebook creates the LLM client in a single cell. **Do not paste a real API key into the notebook.** Load it from an environment variable instead:

```bash
# macOS / Linux
export GEMINI_API_KEY="your-key-here"

# Windows (PowerShell)
$env:GEMINI_API_KEY = "your-key-here"
```

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["GEMINI_API_KEY"],
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/"
)
```

To use a different provider, change `base_url`, the API key variable and the `model` name in the scoring cell. Any OpenAI-compatible endpoint works.

## Usage

1. **Start Jupyter** from the project folder:
   ```bash
   jupyter notebook ATS_Project.ipynb
   ```
2. **Run the setup cells** (imports through FAISS indexing) top to bottom. This builds `faiss_cv_index/` and prints a summary, for example the number of resumes matched and chunks created.
3. **Check the validation cell.** It should show zero chunks with missing metadata and one unique email per resume.
4. **Run the LLM client cell** after setting your API key.
5. **Set the job description.** Replace the text in the `jd = """..."""` cell with your own role. A good job description lists required skills, experience level, and anything that disqualifies a candidate:
   ```python
   jd = """
   Company: Example Corp
   Job Role: Data Engineer (2 openings)
   Location: Bangalore

   What we care about:
   - Strong SQL skills
   - Experience with ETL pipelines
   - 0-2 years experience. Strong freshers encouraged.

   NOT suitable for:
   - Pure dashboarding roles
   """
   ```
6. **Run the retrieval, grouping, scoring and ranking cells.** Scoring takes about 5 to 10 seconds per candidate because of the built-in pacing.

> **Note:** the notebook contains more than one `jd = ...` cell (a sample resume and a sample job posting). Whichever cell ran last is the one used, so make sure your job description is the last one you run before retrieval.

### Re-using the saved index

After the first run you don't need to rebuild the index. In a new session, load it instead:

```python
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_community.vectorstores import FAISS

model = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-mpnet-base-v2",
    model_kwargs={"device": "cpu"},
    encode_kwargs={"normalize_embeddings": True},
)
vectorstore = FAISS.load_local(
    "faiss_cv_index", model, allow_dangerous_deserialization=True
)
```

Only load indexes you created yourself, since `allow_dangerous_deserialization` uses pickle.

## Sample Output

```
✅ Total matched: 84
✅ Total resumes: 84
✅ Total chunks: 352
✅ FAISS index: 352 vectors

[1/15] Candidate A
   ✅ 85
[2/15] Candidate B
   ✅ 75
...

============================================================
🏆 FINAL ATS RANKING
============================================================
#1  Candidate A  —  85/100  (candidate.a@example.com)
#2  Candidate B  —  85/100  (candidate.b@example.com)
#3  Candidate C  —  75/100  (candidate.c@example.com)
```

## Dependencies

### Python packages

| Package | Purpose |
| --- | --- |
| `pandas`, `openpyxl` | Read the Excel tracker |
| `pymupdf` (imported as `fitz`) | Extract text from PDFs and render pages for OCR |
| `python-docx` | Extract text from DOCX files |
| `pytesseract`, `pillow` | OCR for scanned pages |
| `langchain-text-splitters` | Chunk resume text |
| `langchain-huggingface`, `sentence-transformers` | Generate embeddings locally |
| `langchain-community`, `faiss-cpu` | FAISS vector store |
| `langchain-core` | `Document` objects with metadata |
| `openai` | Client for the OpenAI-compatible LLM endpoint |
| `pypdf`, `requests` | Imported by the notebook but not used by the current pipeline |
| `jupyter` | Run the notebook |

### System requirements

- Tesseract OCR binary
- Internet access for the first-time model download and for LLM API calls
- No GPU required (embeddings run on CPU)

## Limitations and Future Improvements

Things to be aware of, and good places to start improving the project:

- **Partial resume context:** the LLM scores only the chunks retrieved for each candidate, not the full resume, so relevant details can be missed. Passing the full resume text for the shortlisted candidates would improve accuracy.
- **Score only:** the model returns a single number with no explanation. Adding a short justification, skill match and gaps would make results easier to trust and audit.
- **Coarse, non-deterministic scores:** ties are common and scores can vary between runs. Setting a low temperature and using a defined rubric would help.
- **Top-15 cutoff:** candidates outside the first 15 retrieved are never scored.
- **Manual inputs:** the job description is edited directly in the notebook and file matching depends on a strict naming convention.
- **Resume parsing:** layouts with multiple columns or heavy graphics may extract poorly, and OCR quality depends on the scan.
- **Human oversight:** AI-generated scores can reflect bias or misread resumes. Use the ranking to prioritize human review, not to automatically reject candidates.

Ideas for future work:

- [ ] Move the pipeline into a Python package or script with a command-line interface
- [ ] Add a `requirements.txt` and `.gitignore`
- [ ] Include the CV downloader step in the repository
- [ ] Return structured results (score, reasoning, matched skills) and export them to CSV/Excel
- [ ] Add a web UI (for example Streamlit) for uploading resumes and pasting a job description
- [ ] Add a reranking step with a cross-encoder
- [ ] Add evaluation against human-labelled rankings

## Contributing

Contributions are welcome, including bug fixes, new features, documentation and ideas.

### Getting started

1. **Fork** the repository and clone your fork.
2. Follow the [Installation](#installation) steps.
3. Create a branch for your work:
   ```bash
   git checkout -b feature/short-description
   ```
   Use prefixes such as `feature/`, `fix/`, `docs/` or `refactor/`.

### Guidelines

- **Never commit secrets.** API keys, tokens and `.env` files must stay out of the repository. Load credentials from environment variables.
- **Never commit real candidate data.** Resumes, the tracker, names, emails and phone numbers are personal information. Use synthetic or anonymized samples in examples and tests.
- **Clear notebook outputs before committing**, since they often contain personal data and large logs. [`nbstripout`](https://github.com/kynan/nbstripout) can do this automatically:
  ```bash
  pip install nbstripout
  nbstripout --install
  ```
- **Keep the notebook runnable top to bottom** (`Kernel → Restart & Run All`) before opening a pull request.
- **Style:** follow [PEP 8](https://peps.python.org/pep-0008/), use descriptive names, and add short docstrings or comments to new functions.
- **Keep changes focused:** one feature or fix per pull request.
- **Document behaviour changes** in this README, especially new settings or dependencies.

### Commit messages

Use short, imperative messages, ideally following [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add score justification to LLM output
fix: handle CVs with missing email in tracker
docs: clarify Tesseract setup on Windows
```

### Submitting a pull request

1. Push your branch and open a pull request against `main`.
2. Describe **what** changed and **why**, and how you tested it.
3. Include before/after output or screenshots where relevant (with any personal data removed).
4. Respond to review feedback; a maintainer will merge once approved.

### Reporting bugs and requesting features

Open a GitHub issue with the steps to reproduce, the full error message, your Python version and operating system, and what you expected to happen. Please remove any personal data from logs and screenshots.

## License

No license has been specified yet. Add a `LICENSE` file (for example MIT or Apache-2.0) to let others know how they may use and contribute to this project, then update this section.

---

<p align="center">Built with LangChain, FAISS and Sentence-Transformers</p>
