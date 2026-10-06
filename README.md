# 🇮🇳 SchemeSaathi — Agentic RAG Assistant for Government Scheme Eligibility

Describe yourself in English, Hindi or Marathi. SchemeSaathi extracts your profile, asks for missing details,
checks eligibility against official scheme documents, and returns a ranked list with **reasons, citations,
a personalised document checklist, application steps and a downloadable PDF**.

## Why it is agentic (not a basic chatbot)
| Step | Tool | Uses LLM? |
|---|---|---|
| 1 Understand | `extract_profile` — multilingual text → JSON | yes |
| 2 Clarify | asks follow-ups when core fields are missing (never guesses) | no |
| 3 Retrieve | `retriever.search` — ChromaDB + multilingual-e5, filtered per scheme, returns file + page | no |
| 4 Reason | `rules.evaluate` — deterministic rule table → ELIGIBLE / NOT_ELIGIBLE / NEED_INFO | no |
| 5 Verify | `verify_with_llm` — checks retrieved evidence for exclusions the rules missed; flags conflicts | yes |
| 6 Act | `generate_checklist`, `find_links`, `build_pdf` | no |
| Q&A | `rag_answer` — grounded answers with `[n]` citations, "not found" fallback | yes |

**Hybrid design:** rules give reproducible decisions, RAG supplies evidence and catches exclusions,
and the LLM handles language. This is what reduces hallucination.

## Setup
```bash
python -m venv venv && source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                  # add GOOGLE_API_KEY or GROQ_API_KEY (or use ollama)
python ingest.py                                      # builds the vector DB (first run downloads the embedding model)
streamlit run app.py
```
Optional: put `NotoSansDevanagari-Regular.ttf` (Google Fonts) in `assets/fonts/` for Hindi/Marathi text in the PDF.

## Replace sample docs with official sources
`data/schemes/*.md` are **student-written summaries** of public scheme information so the project runs out of the box.
For the final submission, download official guideline PDFs (pmkisan.gov.in, pmjay.gov.in, scholarships.gov.in,
myscheme.gov.in, Maharashtra GRs) into `data/schemes/` named `<scheme_id>.pdf` (e.g. `pm_kisan.pdf`) and re-run
`python ingest.py`. Page numbers then appear in citations. **Verify amounts, limits and links**; schemes change.

Add a scheme: add a doc + an entry in `build_schemes.py` (rules, documents, steps) → `python build_schemes.py` → `python ingest.py`.

## Evaluation
```bash
pytest -q                                   # rule-engine unit tests (13 hand-labelled profiles)
python -m eval.run_eval --no-llm            # rule engine only, no API key
python -m eval.run_eval --baseline --retrieval   # agent vs plain-LLM baseline + retrieval hit-rate@4
```
Metrics: precision, recall, F1, exact match, false-positive rate (= wrongly recommended schemes, the hallucination proxy)
and retrieval hit-rate. **Expand `eval/cases.py` to 30–40 profiles** before reporting numbers.

## Project layout
```
app.py          Streamlit UI            agent.py     orchestration
tools.py        agent tools             rules.py     deterministic rule engine
retriever.py    Chroma search           ingest.py    PDF/MD → chunks → embeddings
models.py       profile schema          report.py    PDF generator
llm.py          Gemini/Groq/Ollama      eval/        test cases + evaluation
data/schemes/   knowledge base          data/schemes.json  rules, checklists, steps
```

## Viva cheat-sheet
1. **Why RAG, not fine-tuning?** Documents change often, citations are needed, and it is cheaper.
2. **Why rules + LLM?** Eligibility numbers (age, income) must be exact; LLMs are unreliable at that. Rules decide, the LLM verifies against evidence.
3. **How is hallucination reduced?** Grounded prompts, "not found" fallback, citations, NEED_INFO instead of guessing, measured against a plain-LLM baseline.
4. **Why multilingual-e5?** One embedding space for English, Hindi and Marathi; uses `query:`/`passage:` prefixes.
5. **Chunk size?** 800 / overlap 120; tune by comparing hit-rate@k.
6. **Outdated documents?** Metadata carries `last_verified`; re-ingest on update.

## Limitations & future scope
Rule tables and sample docs are simplified (e.g., SECC-based criteria can't be fully verified from self-reported data).
Future: voice I/O, WhatsApp bot, form auto-fill, expiry alerts, more states.

## Resume bullet
> Built **SchemeSaathi**, a multilingual agentic RAG system (LangChain, ChromaDB, multilingual-e5, Streamlit) that matches
> citizens to 11 government schemes with cited eligibility reasoning, document checklists and PDF reports; reduced
> wrongly-recommended schemes by **X%** vs a plain-LLM baseline on a **N**-profile test set.
