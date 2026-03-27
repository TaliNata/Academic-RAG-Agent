# 📚 Academic Knowledge Base Bot

**Telegram bot with a RAG pipeline for scientific articles**

A medium-complexity agent: ingests documents into a vector store, performs semantic search, and generates answers with source attribution.

---

## Problem

Enable researchers to upload a set of scientific papers and ask questions about their content — without having to re-read each paper manually.

## Features

| Feature | Description |
|---------|-------------|
| 📎 Paper upload | Send a PDF or TXT — the bot extracts text, splits it into chunks, and stores embeddings in a vector database |
| 💬 Q&A over documents | Ask a question — the bot retrieves relevant fragments and generates an answer with source references |
| 📝 Literature review | Specify a topic — the bot compiles a review based on uploaded papers |
| 📊 Knowledge base management | Paper list, statistics, full reset |
| 🤖 Model selection | Switch between LLMs directly in the chat |

## Architecture (RAG Pipeline)

```
                        ┌─────────────────┐
                        │  Telegram User   │
                        └────────┬─────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │   Telegram Bot Layer    │
                    │      (aiogram 3)       │
                    │                        │
                    │  Commands, file        │
                    │  handling, state       │
                    │  management            │
                    └───────────┬────────────┘
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
                 ▼                             ▼
    ┌─────────────────────┐       ┌─────────────────────┐
    │   PAPER INGESTION   │       │   QUERY ANSWERING   │
    │                     │       │                     │
    │  PDF/TXT            │       │  Query text         │
    │       │             │       │       │             │
    │       ▼             │       │       ▼             │
    │  Text extraction    │       │  Query embedding    │
    │  (pdfplumber)       │       │  creation           │
    │       │             │       │       │             │
    │       ▼             │       │       ▼             │
    │  Text cleaning      │       │  Semantic search    │
    │       │             │       │  in ChromaDB        │
    │       ▼             │       │       │             │
    │  Chunking           │       │       ▼             │
    │  (800 chars,        │       │  Top-K fragments    │
    │   200 overlap)      │       │       │             │
    │       │             │       │       ▼             │
    │       ▼             │       │  Context assembly   │
    │  Embedding          │       │  (≤6000 chars)      │
    │  (all-MiniLM-L6-v2) │       │       │             │
    │       │             │       │       ▼             │
    │       ▼             │       │  LLM generation     │
    │  Store in           │       │  with source        │
    │  ChromaDB           │       │  attribution        │
    └─────────────────────┘       └─────────────────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │    OpenRouter API      │
                              │                       │
                              │  Gemini · Claude      │
                              │  GPT-4o · DeepSeek    │
                              └───────────────────────┘
```

## Project Structure

```
academic-rag-bot/
├── bot.py              # Telegram bot: commands, file uploads, question handling
├── rag_engine.py       # RAG pipeline: search → context → generation
├── vector_store.py     # ChromaDB wrapper: collections, CRUD, search
├── text_processing.py  # PDF/TXT parsing, text cleaning, chunking
├── prompts.py          # System prompts for answers and reviews
├── config.py           # Configuration via Pydantic Settings
├── requirements.txt    # Dependencies
├── .env.example        # Environment variables template
└── .gitignore
```

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Language | Python 3.11+ | Core |
| Telegram API | aiogram 3 | Async bot framework |
| Vector DB | ChromaDB | Local embedding storage with persistence |
| Embeddings | all-MiniLM-L6-v2 (via ChromaDB) | Semantic search over document fragments |
| PDF parsing | pdfplumber | Text extraction preserving structure |
| LLM API | OpenRouter + OpenAI SDK | Multi-model access via single endpoint |
| Validation | Pydantic + pydantic-settings | Configuration and typing |

## Key Technical Decisions

**Paragraph-aware chunking.** Text is split along paragraph boundaries, not mechanically by character count. Long paragraphs are further split by sentences. A 200-character overlap between chunks preserves context at boundaries.

**Isolated collections.** Each Telegram user gets their own ChromaDB collection (`user_{id}`). Knowledge bases do not overlap.

**Context window control.** When building the LLM prompt, fragments are added by relevance until hitting the limit (6000 chars). This ensures the prompt never exceeds the model's context window.

**Startup warmup.** On first launch ChromaDB downloads the embedding model (~80 MB). Warmup runs during initialization so the first user request doesn't time out.

**Non-blocking operations.** PDF parsing, embedding creation, and vector search are CPU-bound. They are wrapped in `asyncio.to_thread()` to avoid blocking the bot's event loop and prevent Telegram API timeouts.

**Auto title detection.** When uploading a paper, the bot attempts to detect its title from the first lines using heuristics (length, period presence, position).

## Example Usage

**Uploading a paper:**
```
👤 [sends a PDF file]

🤖 ✅ Paper uploaded!
   📄 Influence of Machine Learning Methods on Time Series
      Forecasting Accuracy in Financial Markets
   🧩 Fragments: 12
   📊 Total in base: 1 papers, 12 fragments
```

**Asking a question:**
```
👤 Which model showed the best results?

🤖 According to the study, the hybrid model combining LSTM
   and XGBoost through a meta-classifier showed the best
   results. It achieved MAE = 0.0119 and RMSE = 0.0167,
   which is 12–18% better than each method individually
   [Influence of Machine Learning Methods...].

   📎 Sources used:
     • Influence of Machine Learning Methods on Time Series
       Forecasting Accuracy in Financial Markets
```

## Comparison with Simple Agent

| Parameter | Simple Agent | RAG Agent |
|-----------|-------------|-----------|
| Data storage | None | ChromaDB (persistent) |
| File processing | None | PDF + TXT parsing |
| Pipeline | Prompt → LLM → response | Parse → chunk → embed → search → context → LLM |
| Context | Message text only | Relevant fragments from knowledge base |
| Sources | None | Cited in response |
| Scalability | 1 text at a time | Multiple documents |

## How to Run

1. Create a bot via [@BotFather](https://t.me/BotFather)
2. Get an API key at [openrouter.ai](https://openrouter.ai)
3. Fill in `.env` (using `.env.example` as template)
4. `pip install -r requirements.txt`
5. `python bot.py` (first launch includes ~80 MB embedding model download)

## Status

✅ Working prototype — bot is deployed and tested on scientific papers.

---

*Built as an example of a medium-complexity RAG agent with vector search, document processing, and source-attributed answer generation.*
