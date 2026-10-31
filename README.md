# Offline Scholar

**An offline study assistant for students.** Upload your lecture notes, past exam papers and syllabus as PDFs.
Offline Scholar then answers your questions *from your own documents* (with the file name and page number),
shows which topics the exams ask about most, finds gaps between the syllabus and your notes, makes practice
quizzes, spots your weak topics and builds a revision plan.

Everything runs **on your own computer**. No cloud, no paid API, no internet needed after the one-time model
download. MIT licensed.

---

## Contents

1. [What problem does it solve?](#1-what-problem-does-it-solve)
2. [What can it do?](#2-what-can-it-do)
3. [How it works, in plain words](#3-how-it-works-in-plain-words)
4. [The team and who built what](#4-the-team-and-who-built-what)
5. [Tools and technologies](#5-tools-and-technologies)
6. [Project structure](#6-project-structure)
7. [Getting started](#7-getting-started)
8. [Using it (API overview)](#8-using-it-api-overview)
9. [The exam engine in detail](#9-the-exam-engine-in-detail)
10. [Testing](#10-testing)
11. [Honest limitations](#11-honest-limitations)
12. [What could come next](#12-what-could-come-next)
13. [License](#13-license)

---

## 1. What problem does it solve?

Before an exam, students have piles of slides, notes and old papers, and a few simple questions:

- *"Where in my notes does this topic appear?"*
- *"What do examiners keep asking?"*
- *"Did my lectures even cover everything on the syllabus?"*
- *"What should I revise first, and where am I weakest?"*

Chatbots can answer such questions, but they usually run in the cloud, can make things up, and don't know your
course. Offline Scholar is built differently: it answers **only from your documents**, shows where each answer
came from, says "I couldn't find that" when it can't, and never sends your files anywhere.

## 2. What can it do?

| Feature | What you get |
|---|---|
| **Ask your documents** | Type a question, get an answer with the source file and page number. If the notes don't contain it, it says so instead of guessing. |
| **Past-paper analysis** | Finds the questions in old papers, spots questions that repeat across years, and groups similar questions. |
| **Important topics** | Ranks topics by how often and how heavily they are examined, with a one-line reason for each rank. |
| **Syllabus coverage** | For every syllabus topic: *Covered*, *Partially covered* or *Not covered*, with the document and pages that prove it. |
| **Practice quizzes** | Multiple-choice questions made only from your notes, with a correct answer, an explanation and the source page. |
| **Weak-topic finder** | Reads your quiz results and tells you which topics need work, in plain language. |
| **Revision plan** | A day-by-day plan that combines exam importance, coverage gaps, your quiz results and your weak topics. |
| **Works offline** | After the one-time download of the language models, nothing needs the internet. |

## 3. How it works, in plain words

```
  Your PDFs (lectures, past papers, syllabus)
          |
          v
  1. READ        PyMuPDF reads each page. Scanned pages and slide images go through OCR (Tesseract).
          |
          v
  2. CUT         The text is cut into small pieces that never cross a page, so every piece
          |      has a known page number.
          v
  3. UNDERSTAND  A small embedding model turns each piece into numbers that capture its meaning.
          |      These go into a FAISS index (one per subject) for fast search.
          v
  +-------+-------------------------------+
  |                                       |
  v                                       v
  CHAT                                  EXAM ENGINE
  Your question is turned into          Reads the past papers, syllabus, lectures
  numbers too, the closest pieces       and your quiz results, and produces
  are found, and the local AI           topic rankings, coverage, quizzes,
  model (Ollama) writes an answer       weak topics and a revision plan.
  using ONLY those pieces and
  cites them.
          |                                       |
          +------------------+--------------------+
                             v
               FastAPI server + SQLite database  -->  the website (frontend)
```

Two safety rules run through the whole project:

1. **No source, no answer.** If nothing relevant is found, the AI model is not even asked.
2. **Everything is explainable.** Every ranking, status and recommendation carries a short sentence saying why.

## 4. The team and who built what

The project has three parts, one per team member.

### Member 1: Malaika Adnan, backend and AI engine

Built the foundation that everything else stands on:

- **Document processing:** PDF text extraction, OCR for scanned pages and slide images, and sentence-aware
  chunking (`ai_engine/`).
- **Search:** embeddings with a MiniLM model and a FAISS vector index per subject.
- **Question answering (RAG):** retrieval, a strict prompt that forces the local Ollama model to cite numbered
  passages or answer `NOT_FOUND`, and a support check against made-up answers.
- **Backend:** FastAPI server, SQLite database, upload / chat / quiz / subject / health routes, error handling
  (`backend/`).
- **Reliability:** offline mode with clear error messages, Windows fixes (Tesseract path, locked files,
  FAISS saves), model download script, automatic tests on GitHub Actions.
- **Evaluation:** tested with the real models on a real lecture PDF (`docs/EVALUATION.md`, `scripts/evaluate.py`).
- **Integration contract:** a clear interface (`docs/INTEGRATION.md`) so the other members could plug in.

### Member 2: Malaika Khan, exam intelligence engine

Built the `exam_engine/`, the part that turns documents into exam-preparation insight. Six modules:

1. **Past-paper analysis:** finds questions (numbering, sub-parts, marks, sections), detects repeated questions
   across different papers, groups similar questions.
2. **Topic analysis:** ranks topics by recurrence, marks and recency, and links each question to a topic.
3. **Syllabus coverage:** Covered / Partially covered / Not covered with document, pages, an evidence score and
   a short explanation. It checks meaning, not just keywords.
4. **Quiz generation:** four-option multiple-choice questions that are always backed by a sentence in the notes.
   With the local model, every question must quote its evidence or it is thrown away.
5. **Weak-topic finder:** accuracy per topic from quiz results, with "revise the basics first" or
   "practise application" advice.
6. **Adaptive revision plan:** combines the other five into a day-by-day schedule.

Also contributed: connecting the engine to the backend (`exam_engine/api.py`, `exam_engine/bridge.py`), the JSON
contracts, example outputs, a test suite, and scripts to try the engine on real PDFs. It was tested on a real
course (MIT OpenCourseWare 6.867 Machine Learning: four exams, six lectures and a syllabus), which led to many
fixes. Details in [section 9](#9-the-exam-engine-in-detail).

### Member 3: Essa Afridi, frontend

Responsible for the user interface (`frontend/`) that students use: uploading documents, chatting, viewing the
exam analysis, taking quizzes and following the revision plan. It talks to the backend through the JSON
described in [`docs/API.md`](docs/API.md) and [`docs/EXAM_ENGINE_CONTRACTS.md`](docs/EXAM_ENGINE_CONTRACTS.md).

> **Essa: please complete this section** with the pages and components you built, the framework and libraries
> you used, and a screenshot or two.

## 5. Tools and technologies

Everything below is free and runs locally.

| Tool | What it is | What we use it for |
|---|---|---|
| **Python 3** | Programming language | The backend, the AI engine and the exam engine. |
| **FastAPI** | Web framework | The API server that the frontend talks to. It also generates interactive docs at `/docs`. |
| **SQLite** + **SQLAlchemy** | A database in one file, and the Python tool to use it | Stores subjects, documents, text chunks, quiz attempts, topic performance and revision recommendations. |
| **PyMuPDF** | PDF library | Reads the text of every page. |
| **Tesseract (OCR)** | Text recognition for images | Reads scanned pages and slides whose content is in pictures. |
| **sentence-transformers** (`all-MiniLM-L6-v2`) | A small neural model, about 90 MB | Turns text into numbers (embeddings) so the computer can compare meaning. |
| **FAISS** | Fast similarity search library | Finds the pieces of text closest in meaning to a question. |
| **Ollama** + **qwen2.5** | A tool to run AI language models locally, and the model | Writes answers from the retrieved pieces, and writes quiz questions. |
| **scikit-learn** + **NumPy** | Math and machine-learning libraries | TF-IDF fallback matching, clustering of similar questions, and fast vector math. |
| **Pydantic** | Data validation | Checks every request and response shape. |
| **pytest** | Testing tool | Automatic tests for every part. |
| **GitHub Actions** | Automatic checks on GitHub | Runs the tests on every change. |

Two ideas worth knowing:

- **Embeddings** are lists of numbers that represent the *meaning* of a text. Texts with similar meaning get
  similar numbers, so "support vector machine" and "SVM" or "overfitting" and "memorising noise" can be matched
  even when the words differ.
- **RAG (retrieval-augmented generation)** means the AI model first gets the relevant pieces of *your* notes and
  must answer from them. This is why answers can be checked against a page.

## 6. Project structure

```
offline-scholar/
  frontend/        The user interface (Member 3)
  backend/         FastAPI app, SQLite models, routes (Member 1)
  ai_engine/       PDF reading, chunking, embeddings, FAISS, RAG, Ollama client (Member 1)
  exam_engine/     Exam analysis, quizzes, weak topics, revision plan (Member 2)
    question_extraction.py   finds questions in past papers
    similarity.py            repeated / similar question groups
    topic_analysis.py        topic ranking
    syllabus_coverage.py     Covered / Partial / Not covered
    quiz_generator.py        grounded multiple-choice questions
    performance_analysis.py  weak topics from quiz results
    revision_engine.py       the revision plan
    api.py, bridge.py        how the backend calls the engine
  docs/            API.md, INTEGRATION.md, EXAM_ENGINE_CONTRACTS.md, examples
  scripts/         download_models.py, analyze_folder.py, compare_embedders.py, test_llm_quiz.py, ...
  tests/           automatic tests
  data/            uploads, indexes and the database (not stored in git)
```

## 7. Getting started

You need Python 3.10 or newer. Ollama is only needed for the AI answers and the better quiz mode.

```bash
# 1. get the code
git clone https://github.com/malaikamalik23/OfflineScholar.git
cd OfflineScholar

# 2. create a virtual environment and install the packages
python -m venv .venv
.venv\Scripts\activate            # Windows (on Linux / macOS: source .venv/bin/activate)
pip install -r requirements.txt

# 3. one-time download of the models (needs internet)
#    install Ollama first: https://ollama.com
python scripts/download_models.py    # MiniLM embeddings (~90 MB) and the qwen2.5 model (~2 GB)

# 4. optional OCR for scanned PDFs and slide images: install Tesseract

# 5. start the server
copy .env.example .env               # cp on Linux / macOS; set OS_OFFLINE=1 to forbid network use
uvicorn backend.main:app --port 8000
```

Open <http://localhost:8000/docs> for the interactive API page, and
<http://localhost:8000/api/health> to check that everything is running (`status` should say `ok`).

## 8. Using it (API overview)

Full details and examples are in [`docs/API.md`](docs/API.md).

| Method and path | What it does |
|---|---|
| `GET /api/health` | Checks the database, the language model and the embeddings. |
| `POST /api/subjects` | Creates a subject (a course). |
| `POST /api/documents/upload` | Uploads a PDF and marks it as `lecture`, `past_paper` or `syllabus`. |
| `GET /api/documents` | Lists the uploaded documents. |
| `POST /api/chat/ask` | Asks a question; returns an answer with sources, or says it couldn't find one. |
| `POST /api/quiz/submit` | Stores quiz answers and updates per-topic performance. |
| `GET /api/analysis/{subject_id}` | Past-paper analysis, topic ranking, syllabus coverage and weak topics. |
| `GET /api/revision/{subject_id}` | The revision plan (also saved in the database). |

A typical flow: create a subject, upload the syllabus, lecture notes and a few past papers (choosing the right
type for each), then call `analysis` and `revision`.

## 9. The exam engine in detail

### How it decides (all explainable)

- **Questions** are found with rules for numbering styles, sub-parts, marks and sections, and headers
  (name, time allowed, instructions) are ignored. Page headers and footers that repeat on every page are removed.
- **Repeated questions** are questions with a very similar meaning that appear in at least two *different*
  papers. Each group lists the shared key terms.
- **Topic ranking** = how many papers ask about it (45 %) + how many marks it carries (35 %) + how recently (20 %).
  With only one paper, the engine says recurrence can't be measured and marks its confidence as low.
- **Syllabus coverage** mixes meaning-based matching, word coverage, acronyms (SVM = support vector machine)
  and evidence from several pages. A topic that is not covered names the closest material it did find.
- **Quizzes** are built only from sentences that exist in the notes. If the material is too thin or messy,
  the engine returns fewer questions instead of unsupported ones.
- **Weak topics** need at least three answers before a topic is called weak, and use smoothed accuracy so that
  one lucky or unlucky answer doesn't decide.
- **Revision plan** mixes four signals (exam importance, weakness, coverage gap, and their interaction) and
  only uses the signals that are actually available.

### What it found on a real course

Tested on MIT OpenCourseWare 6.867 Machine Learning (four exams, six lectures, a syllabus):

- The question finder read 77 real exam questions after cleaning out page footers and header lines.
- The syllabus parser went from 80 messy lines (menus, ISBNs, dates) to 37 real topics.
- With lectures 1 to 6 only, topics taught there (perceptron, margin, regularization, logistic regression,
  active learning) came out *Covered* with page numbers, and topics from later lectures (boosting, hidden Markov
  models, clustering, Bayesian networks) came out *Not covered*, which is correct.
- The most examined topics were linear classification, logistic regression, maximum margin and regularization.

Testing on real files caught several problems that the demo data had hidden (page footers read as questions,
a course number read as a question number, special characters breaking word matching, topics wrongly marked as
covered). All were fixed and covered by new tests.

### Try it on your own PDFs

```bash
python scripts/analyze_folder.py path\to\your\folder     # file names decide roles: *Midterm / *Final / *Paper
                                                         # = past papers, *Syllabus = syllabus, others = lectures
python scripts/compare_embedders.py                      # simple fallback vs the real model
python scripts/test_llm_quiz.py path\to\folder "topic"   # quizzes with the local Ollama model
```

More in [`exam_engine/README.md`](exam_engine/README.md) and
[`docs/EXAM_ENGINE_CONTRACTS.md`](docs/EXAM_ENGINE_CONTRACTS.md).

## 10. Testing

```bash
pip install -r requirements-dev.txt
pytest -q
```

The tests use a simple stand-in embedder and a fake language model, so they run without any downloads. They
exercise real PDFs, OCR, chunking, the vector index, the database and every endpoint, plus the whole exam engine
through the real upload API. On Windows without Tesseract installed, one OCR test is skipped
(`66 passed, 1 skipped` at the time of writing).

Answer quality with the real models was checked separately: see [`docs/EVALUATION.md`](docs/EVALUATION.md).

## 11. Honest limitations

- The exam engine has been checked on **one real course**. Its thresholds (for example, when a topic counts as
  covered) are provisional, and generic topic names can still be marked *Covered* too easily.
- Without Ollama, quizzes are simple definition and fill-in-the-blank questions. The Ollama mode writes more
  natural questions and checks them against the notes, but it has **not yet been tested on real lectures**.
- Small local models can misread numbers in tables and formulas, so citations (page numbers) always matter.
- PDF only. Scanned pages depend on OCR quality. Two-column layouts and tables can confuse extraction.
- Single user, no login, and no database migrations yet.
- The backend currently has no quiz-generation route. The engine function `generate_quiz_for_subject` is ready.
- Quiz answers don't store difficulty yet, so weak-topic advice is more generic than it could be.

## 12. What could come next

- A quiz-generation endpoint and difficulty tracking, so weak-topic advice can say "revise the basics" or
  "practise application".
- Calibrating the engine on more courses and fixing the generic-topic case.
- Support for Word and PowerPoint files.
- User accounts and a more polished study dashboard.

## 13. License

MIT. Free to use, change and share.
