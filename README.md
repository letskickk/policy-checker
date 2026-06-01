# policy-checker — AI 공약검증 / AI Pledge Checker

후보의 **선거 공약**을 입력하면, 당 정강·정책과의 **부합 여부**, 다른 후보 공약과의 **중복·유사도**, 그리고 **보완점**을 AI가 즉시 채점·리포트로 돌려주는 한국어 정책검증 파이프라인입니다.

> An open-source Korean-language pipeline that verifies election pledges with RAG search and LLM scoring — checking each pledge against a party platform, detecting overlap with other candidates, and returning a graded report with cited evidence.

---

## 🗳️ 실제 사용 사례 (Real-world use)

이 도구는 데모가 아니라 **실제 정치 현장에서 쓰였습니다.** 개혁신당(Reform Party)이 **2026년 지방선거 출마자들의 공약을 검증·보정**하는 데 직접 활용했습니다. 후보가 작성한 공약을 당 정강·과거 당선인 공약과 대조해 실현가능성·정합성을 점검하고, 차별화 포인트를 제안하는 내부 도구로 운영됐습니다.

그동안 정당 내부에서 비공개로만 돌던 '공약 검증'을 오픈소스로 공개합니다. 누구나 — 시민·정당·후보 — 공약의 품질을 **데이터로 검증**할 수 있게 하는 시빅테크 인프라를 지향합니다.

> Not a demo: this tool was used in production by Korea's Reform Party to vet pledges from its 2026 local-election candidates. Open-sourcing it makes data-driven pledge verification — previously locked inside a party — available to anyone.

---

## ✨ 핵심 기능

| 기능 | 설명 |
|------|------|
| **당 방향 부합 점검** | 정강·정책·과거 공약을 기준으로 공약의 부합 여부를 5개 축(실현가능성·정합성·전달력 등)으로 채점하고 보완 체크리스트 제공 |
| **벡터 검색 기반 근거 인용** | OpenAI 임베딩 + FAISS / OpenAI Vector Store로 관련 근거 스니펫을 검색해 **출처와 함께** 인용 (`POST /api/pledge/verify`) |
| **중복·유사 탐지** | 다른 후보·과거 당선인 공약 DB와 비교해 유사 공약을 제시하고 차별화 포인트 제안 |
| **접근제어·쿼터** | 회원가입·관리자 승인·일/월 쿼터·레이트리밋·결과 캐싱으로 API 비용 관리 |

## 🏗️ 아키텍처

```
공약 입력
   │
   ├─▶ 청킹·임베딩 (text-embedding-3-large)
   │        └─ FAISS  또는  OpenAI Vector Store (File Search)
   │
   ├─▶ RAG 검색: 정강정책 · 우리당 공약 · 타지역/과거 공약
   │
   └─▶ LLM 채점 (GPT): 5축 점수 + 근거 인용 + 상충 분석 + 개선 제안
              │
              └─▶ JSON 리포트 (summary / platform / pledges / conflicts / improvements)
```

- **백엔드**: FastAPI (`backend/main.py`)
- **검색**: `backend/embeddings.py`, `backend/vector_index.py`, `backend/openai_vector_store.py`
- **채점**: `backend/check_service.py`, `backend/prompts.py`
- **데이터 연동**: 공공데이터포털(NEC) 당선인·선거공약 API (`backend/national_assembly_api.py`, `backend/public_data_api.py`)

## 🚀 빠른 시작 (Quick start)

```bash
# 1. 가상환경 + 의존성
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# 2. 환경변수: .env.example → .env 복사 후 OPENAI_API_KEY 입력
cp .env.example .env

# 3. 문서 배치: data/pdf/ 아래에 정강정책/ · 공약/ 폴더로 PDF 배치
#    (정강·공약 PDF는 라이선스/저작권 사유로 저장소에 포함되지 않습니다)

# 4. DB 초기화 (최초 1회)
python scripts/init_db.py

# 5. 서버 실행
uvicorn backend.main:app --reload --workers 1
#    → http://127.0.0.1:8000/docs
```

검증 API 호출 예시:

```bash
curl -X POST http://127.0.0.1:8000/api/pledge/verify \
  -H "Content-Type: application/json" \
  -d '{"text": "지역 청년 일자리 1000개 창출", "top_k_platform": 6, "top_k_pledge": 6}'
```

> **데이터 안내**: 이 저장소는 **코드만** 공개합니다. 정강·공약 PDF, 인덱스 캐시, 운영 DB 등 실데이터는 포함되지 않습니다(`data/pdf/`, `data/index_cache/`는 gitignore). 공공데이터포털 공약·당선인 API 키(`DATA_GO_KR_API_KEY`)와 `OPENAI_API_KEY`는 각자 `.env`에 설정하세요.

## ⚙️ 주요 환경변수

```env
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-large
USE_OPENAI_VECTOR_STORE=0          # 1이면 FAISS 대신 OpenAI File Search 사용
DATA_GO_KR_API_KEY=               # 공공데이터포털 (당선인/선거공약 API)
ADMIN_EMAILS=admin@example.com    # 자동 승인 관리자
```

전체 설정·운영·배포는 [`docs/`](docs/)를 참고하세요.

## 📚 문서

- [요구사항 명세서](docs/요구사항_명세서.md)
- [기술 명세 및 구현 방향](docs/기술_명세_구현방향.md)
- [OpenAI Vector Store 사용법](docs/OpenAI_Vector_Store_사용법.md)
- [AWS 배포 가이드](docs/AWS_배포_가이드.md)

## 🤝 기여 (Contributing)

이슈·PR 환영합니다. 한국어 공공문서 RAG·채점 파이프라인은 정책 외 도메인(법률·행정 문서 등)에도 재사용 가능한 구조로 설계했습니다.

## 📄 라이선스

[MIT](LICENSE) © 2026 Kwon Sol ([@letskickk](https://github.com/letskickk))
