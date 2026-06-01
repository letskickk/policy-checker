# policy-checker — AI Pledge Drafting & Verification

**English** · **[한국어](README.md)**

An open-source Korean-language AI pipeline that **drafts and verifies** election pledges. Give it a topic and it gathers evidence and **generates** a policy draft; give it a finished pledge and it instantly **scores and reports** it against the party platform — checking alignment, overlap with other candidates, and gaps.

---

## 🗳️ Real-world use

This is not a demo. The tool was used **in production by Korea's Reform Party (개혁신당)** to draft and vet pledges from its **2026 local-election candidates** — checking each pledge against the party platform and past winners' pledges for feasibility and consistency, and suggesting points of differentiation.

Open-sourcing it makes data-driven pledge drafting and verification — previously locked inside a party — available to anyone: citizens, parties, and candidates. It aims to be civic-tech infrastructure for evaluating the quality of political pledges with data.

---

## ✨ Features

| Feature | Description |
|---|---|
| **Pledge / policy drafting** | From a topic or keywords, a research assistant gathers evidence and GPT streams a draft (policy position, regional pledge, legislative rationale, commentary, message) via `/api/tools/generate/stream`, `/api/policy/draft` |
| **Party-alignment check** | Scores a pledge against the platform and past pledges on 5 axes (feasibility, consistency, deliverability, etc.) with a fix-up checklist |
| **Evidence-cited RAG search** | Retrieves supporting snippets via OpenAI embeddings + FAISS / OpenAI Vector Store and cites them **with sources** (`POST /api/pledge/verify`) |
| **Overlap / similarity detection** | Compares against other candidates' and past winners' pledges and suggests differentiation |
| **Access control & quotas** | Sign-up, admin approval, daily/monthly quotas, rate limiting, and result caching to manage API cost |

## 🏗️ Architecture

```
[ Generate ]  topic ─▶ research assistant (evidence) + RAG context ─▶ GPT draft (streaming)
                                                                       └─▶ policy position / regional pledge / legislative rationale / commentary / message

[ Verify ]    pledge
                │
                ├─▶ chunking + embeddings (text-embedding-3-large)
                │        └─ FAISS  or  OpenAI Vector Store (File Search)
                │
                ├─▶ RAG search: platform · own-party pledges · other-region/past pledges
                │
                └─▶ LLM grading (GPT): 5-axis score + cited evidence + conflict analysis + suggestions
                           │
                           └─▶ JSON report (summary / platform / pledges / conflicts / improvements)
```

- **Backend**: FastAPI (`backend/main.py`)
- **Generation**: `backend/policy_drafter.py`, `backend/research_assistant.py`, `backend/tools_routes.py`
- **Retrieval**: `backend/embeddings.py`, `backend/vector_index.py`, `backend/openai_vector_store.py`
- **Grading**: `backend/check_service.py`, `backend/prompts.py`
- **Public data**: Korea NEC winners/pledge APIs (`backend/national_assembly_api.py`, `backend/public_data_api.py`)

## 🚀 Quick start

```bash
# 1. venv + deps
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# 2. env: copy .env.example → .env and set OPENAI_API_KEY
cp .env.example .env

# 3. place documents under data/pdf/ (정강정책/ and 공약/ folders)
#    Platform/pledge PDFs are NOT included in the repo (licensing/copyright)

# 4. init DB (first run only)
python scripts/init_db.py

# 5. run
uvicorn backend.main:app --reload --workers 1
#    → http://127.0.0.1:8000/docs
```

Verify a pledge:

```bash
curl -X POST http://127.0.0.1:8000/api/pledge/verify \
  -H "Content-Type: application/json" \
  -d '{"text": "Create 1,000 local youth jobs", "top_k_platform": 6, "top_k_pledge": 6}'
```

> **Data note**: this repository ships **code only**. Real data (pledge/platform PDFs, index cache, runtime DB) is excluded (`data/pdf/`, `data/index_cache/` are gitignored). Set your own `OPENAI_API_KEY` and Korea public-data key (`DATA_GO_KR_API_KEY`) in `.env`.

## ⚙️ Key environment variables

```env
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-large
USE_OPENAI_VECTOR_STORE=0          # 1 → OpenAI File Search instead of FAISS
DATA_GO_KR_API_KEY=                # Korea public-data portal (winners / pledge APIs)
ADMIN_EMAILS=admin@example.com     # auto-approved admins
```

See [`docs/`](docs/) for full configuration, operations, and deployment (Korean).

## 🤝 Contributing

Issues and PRs welcome. The Korean public-document RAG + grading pipeline is built to be reusable for other domains (legal/administrative documents, etc.).

## 📄 License

[MIT](LICENSE) © 2026 개혁신당 정책국 (Reform Party Policy Bureau)
