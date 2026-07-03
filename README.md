# 📊 AI Financial Analyst

A local AI application that reads annual reports, balance sheets, P&L and
cash flow statements (PDFs), extracts the numbers, calculates financial
ratios, and lets you ask natural-language questions like:

> "Why did the company's net profit decrease in FY2024?"

It runs **entirely on your own PC** — no API keys, no cloud costs, no data
leaving your machine.

---

## 1. How it works (architecture)

```
                 ┌──────────────────────┐
   PDF Upload →  │   FastAPI Backend     │
                 │  ───────────────────  │
                 │ 1. OCR Extractor      │  (pdfplumber + Tesseract OCR fallback)
                 │ 2. Text Chunking      │
                 │ 3. Embeddings (bge)   │  → stored in ChromaDB (vector DB)
                 │ 4. Financial Calc     │  (regex-based figure extraction + ratios)
                 │ 5. LLM (Ollama)       │  (RAG answers + narrative summaries)
                 └──────────────────────┘
                            ▲
                            │ REST API (HTTP/JSON)
                            ▼
                 ┌──────────────────────┐
                 │ Streamlit Frontend    │
                 │  - Upload tab         │
                 │  - Ask Questions tab  │
                 │  - Financial Summary  │
                 │  - Manage Documents   │
                 └──────────────────────┘
```

**Note on the LLM model:** the project brief mentioned Qwen3-VL / Gemma 4 /
GLM-OCR. Those are not yet publicly distributable as easy local packages, so
this build uses widely-available, equivalent free/open alternatives that
deliver the same skills (OCR, RAG, financial analysis, LLM, vector DB) and
run reliably on a normal PC:
- **OCR:** Tesseract OCR (industry-standard, free) instead of GLM-OCR
- **LLM:** Qwen2.5 / Llama 3.1 / Gemma 2 served via **Ollama** (you can pick
  any of these — see step 4 below) instead of Qwen3-VL/Gemma4
- **Embeddings:** `BAAI/bge-small-en-v1.5` exactly as requested
- **Vector DB:** ChromaDB exactly as requested

If you later get access to Qwen3-VL or GLM-OCR specifically, you can swap
`backend/llm_handler.py` and `backend/ocr_extractor.py` to call them instead
— the rest of the app (API, frontend, ratio engine) doesn't need to change.

---

## 2. Folder structure

```
financial-analyst-ai/
├── backend/
│   ├── main.py                 # FastAPI app & all API endpoints
│   ├── ocr_extractor.py        # PDF text extraction + OCR fallback
│   ├── rag_engine.py           # Embeddings + ChromaDB vector search
│   ├── llm_handler.py          # Talks to the local LLM via Ollama
│   ├── financial_calculator.py # Figure extraction + ratio formulas
│   └── models.py                # Request/response data schemas
├── frontend/
│   └── app.py                  # Streamlit user interface
├── data/
│   ├── uploads/                # Uploaded PDFs are stored here
│   └── chroma_db/               # Vector database files (auto-created)
├── requirements.txt
├── .env.example
└── README.md   ← you are here
```

---

## 3. Prerequisites

You need three things installed on your PC **before** installing the Python
packages:

| Requirement | Why | Windows | macOS | Linux |
|---|---|---|---|---|
| Python 3.10+ | Runs the app | [python.org](https://www.python.org/downloads/) | `brew install python` | `sudo apt install python3 python3-pip` |
| Tesseract OCR | Reads scanned/image PDF pages | [UB-Mannheim installer](https://github.com/UB-Mannheim/tesseract/wiki) | `brew install tesseract` | `sudo apt install tesseract-ocr` |
| Poppler | Converts PDF pages to images for OCR | [poppler for Windows](https://github.com/oschwartz10612/poppler-windows/releases) (add `bin` folder to PATH) | `brew install poppler` | `sudo apt install poppler-utils` |

> ⚠️ **Windows users:** after installing Tesseract, note the install path
> (usually `C:\Program Files\Tesseract-OCR\tesseract.exe`). If pytesseract
> can't find it automatically, add this line at the top of
> `backend/ocr_extractor.py`:
> ```python
> import pytesseract
> pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"
> ```

---

## 4. Install Ollama (the local LLM engine)

1. Download and install Ollama from **https://ollama.com/download** (Windows, Mac, Linux all supported).
2. Open a terminal and pull a model. Pick ONE based on your PC's RAM:

   | Your RAM | Recommended model | Command |
   |---|---|---|
   | 8 GB | `qwen2.5:3b` (fastest, lighter quality) | `ollama pull qwen2.5:3b` |
   | 16 GB | `qwen2.5:7b` (recommended default) | `ollama pull qwen2.5:7b` |
   | 32 GB+ | `qwen2.5:14b` or `llama3.1:8b` | `ollama pull qwen2.5:14b` |

3. Update the model name in your `.env` file (step 6 below) to match what you pulled.
4. Ollama runs automatically as a background service after install — you can verify it's running by visiting `http://localhost:11434` in your browser (it should show "Ollama is running").

---

## 5. Set up the Python project

Open a terminal in the `financial-analyst-ai` folder and run:

```bash
# 1. Create a virtual environment (keeps dependencies isolated)
python -m venv venv

# 2. Activate it
# Windows:
venv\Scripts\activate
# macOS / Linux:
source venv/bin/activate

# 3. Install all required packages
pip install -r requirements.txt
```

This will download the `bge-small-en-v1.5` embedding model automatically the
first time you run the backend (it's small, ~130MB, and only happens once).

---

## 6. Configure environment variables

```bash
# Copy the example file
# Windows:
copy .env.example .env
# macOS / Linux:
cp .env.example .env
```

Open `.env` in any text editor and set `OLLAMA_MODEL` to match the model you
pulled in step 4 (e.g. `qwen2.5:7b`). Leave everything else as default unless
you have a specific reason to change it.

---

## 7. Run the application

You need **two terminals open at the same time** (both with the virtual
environment activated):

**Terminal 1 — start the backend (FastAPI):**
```bash
uvicorn backend.main:app --reload --port 8000
```
You should see `Application startup complete.` Leave this running.
You can check it's working by visiting `http://localhost:8000` in a browser
— it should show `{"status":"ok", ...}`.

**Terminal 2 — start the frontend (Streamlit):**
```bash
streamlit run frontend/app.py
```
This automatically opens your browser at `http://localhost:8501` with the
app's user interface.

---

## 8. Using the app (for operators / end users)

1. **Upload Documents tab** — choose a PDF (annual report, balance sheet,
   P&L, or cash flow statement) and click "Process & Index Document". Wait
   for the success message — large or scanned PDFs take longer because of OCR.
2. **Ask Questions tab** — type any question in plain English, optionally
   restrict it to one document, and click "Ask". The answer always shows
   which document/page it came from underneath, in "View source excerpts".
3. **Financial Summary tab** — pick an uploaded document and click "Generate
   Summary" to see auto-extracted figures, calculated ratios (with formulas
   and plain-English interpretations), a bar chart, and an AI-written
   narrative summary of company performance.
4. **Manage Documents tab** — see everything currently indexed and delete
   any document you no longer need (this also removes its data from the
   vector database).

---

## 9. Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| "Could not connect to the backend API" | FastAPI server (Terminal 1) isn't running | Start it as shown in step 7 |
| "Could not reach the local LLM (Ollama)" | Ollama isn't running, or the model name in `.env` doesn't match a pulled model | Run `ollama list` to see pulled models; update `OLLAMA_MODEL` in `.env` to match exactly |
| OCR returns empty text on scanned PDFs | Tesseract or Poppler not installed / not on PATH | Re-check step 3, restart your terminal after installing |
| "No standard figures could be auto-detected" | The report uses an unusual layout/labels the regex patterns don't recognize | Use the Ask Questions tab instead and ask directly, e.g. "What was the net profit in FY2024?" |
| First upload is slow | The embedding model is being downloaded the first time | Only happens once; subsequent uploads are fast |
| Answers seem generic / not grounded | Document wasn't indexed properly | Check the "Manage Documents" tab to confirm it appears in the list |

---

## 10. Extending the project (optional next steps)

- Swap `qwen2.5` for a vision-capable model (e.g. `qwen2.5vl` once available
  in Ollama) to let the LLM directly read charts/tables as images instead of
  only OCR'd text.
- Add a login/auth layer if multiple people will share one deployment.
- Replace the in-memory `_document_text_cache` in `main.py` with a small
  SQLite database for persistence across backend restarts.
- Add support for `.xlsx` balance sheets directly (skip OCR, read via
  `pandas`/`openpyxl`) for companies that share spreadsheets instead of PDFs.

---

## 11. Skills demonstrated (for your resume)

- **OCR**: Tesseract + pdfplumber hybrid extraction pipeline
- **RAG**: chunking, embeddings (bge-small), ChromaDB retrieval, grounded LLM answers
- **Financial analysis**: regex-based figure extraction, ratio formulas (profitability, liquidity, leverage)
- **LLMs**: local inference via Ollama, prompt design for grounded Q&A and summarization
- **Vector databases**: ChromaDB persistence, metadata filtering, similarity search
- **Full-stack app**: FastAPI REST backend + Streamlit frontend, end-to-end local deployment
