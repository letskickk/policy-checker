# policy-checker — AI 공약 작성·검증

**한국어** · **[English](README.en.md)**

후보의 **선거 공약을 작성하고 검증하는** 한국어 AI 파이프라인입니다. 주제만 입력하면 근거를 모아 정책 초안을 **생성**하고, 작성된 공약을 당 정강·정책과의 **부합 여부**·다른 후보와의 **중복·유사도**·**보완점** 기준으로 즉시 **채점·리포트**합니다.

---

## 🗳️ 실제 사용 사례 (Real-world use)

이 도구는 데모가 아니라 **실제 정치 현장에서 쓰였습니다.** 개혁신당(Reform Party)이 **2026년 지방선거 출마자들의 공약을 검증·보정**하는 데 직접 활용했습니다. 후보가 작성한 공약을 당 정강·과거 당선인 공약과 대조해 실현가능성·정합성을 점검하고, 차별화 포인트를 제안하는 내부 도구로 운영됐습니다.

그동안 정당 내부에서 비공개로만 돌던 '공약 작성·검증'을 오픈소스로 공개합니다. 누구나 — 시민·정당·후보 — 공약의 품질을 **데이터로 검증**할 수 있게 하는 시빅테크 인프라를 지향합니다.

---

## ✨ 핵심 기능

| 기능 | 설명 |
|------|------|
| **① 공약·정책 초안 생성** | 주제·지역만 입력하면 리서치 어시스턴트가 국회·지방의회 의정자료·통계·여론조사에서 근거를 모아 브리핑하고, RAG 컨텍스트 + GPT가 **정책포지션·지역공약·입법취지서·논평·메시지** 초안을 출처와 함께 스트리밍 생성 (`/api/tools/generate/stream`, `/api/policy/draft`) |
| **② 5축 정량 채점 (100점 만점)** | 공약을 **정강정책 정합성(20) · 정책 설계 완성도(30) · 실현 가능성(20) · 구체성(15) · 전달력(15)** 5개 축으로 채점하고, 강점·보완점 체크리스트와 종합 등급을 산출 |
| **③ 근거 인용 RAG 검증** | 정강·우리당 공약·타지역/과거 당선인 공약을 벡터 검색해 **근거 스니펫을 출처와 함께 인용**하고 정합성·상충점을 리포트 (`POST /api/pledge/verify`) |
| **④ 중복·유사 탐지 & 후보 랭킹** | 다른 후보·과거 당선인 공약과 비교해 유사 공약·차별화 포인트를 제시하고, 후보별 공약 점수 리더보드를 제공 |
| **⑤ 이슈 레이더 (핫이슈 발굴)** | 여론조사·논평·법안·지역 현안에서 주목 이슈를 추출해 공약 주제를 추천 |
| **⑥ 자동 데이터 수집 (SSOT)** | 국회 법안·NESDC 여론조사·논평·공약 PDF를 정기 인제스트해 정책 단일출처(SSOT)로 관리하고 Vector Store를 자동 동기화 |
| **⑦ 운영·접근제어** | 회원가입·관리자 승인·신청자 검증·일/월 쿼터·레이트리밋·결과 캐싱으로 API 비용과 접근을 관리 |

## 🏗️ 아키텍처

```
[ 생성 ]  주제 입력 ─▶ 리서치 어시스턴트(근거 수집) + RAG 컨텍스트 ─▶ GPT 초안 생성(스트리밍)
                                                                       └─▶ 정책포지션·지역공약·입법취지서·논평·메시지

[ 검증 ]  공약 입력
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
- **생성**: `backend/policy_drafter.py`, `backend/research_assistant.py`, `backend/tools_routes.py`
- **검색**: `backend/embeddings.py`, `backend/vector_index.py`, `backend/openai_vector_store.py`
- **채점**: `backend/check_service.py`, `backend/prompts.py`
- **데이터 연동**: 공공데이터포털(NEC) 당선인·선거공약 API (`backend/national_assembly_api.py`, `backend/public_data_api.py`)

## 🔌 데이터 출처 & 연동 API

공약 생성은 단순 GPT 호출이 아니라 **실제 공공데이터·의정자료에 근거**합니다. 리서치 어시스턴트가 주제·지역에 맞춰 아래 공개 API를 호출해 근거를 수집한 뒤, OpenAI로 초안을 생성하고 인용까지 붙입니다.

| 출처 | 용도 | 엔드포인트 · 키 |
|------|------|-----------------|
| **OpenAI API** | 임베딩 · Vector Store(File Search) · GPT 초안 생성·채점 | `OPENAI_API_KEY` |
| **국회 지방의회 의정포털 (CLIK)** | 지방·광역의회 의정자료·안건 | `clik.nanet.go.kr/openapi` · `ASSEMBLY_API_KEY` |
| **국회 발언 빅데이터 (NANET)** | 의원 발언 검색 | `dataset.nanet.go.kr/api` · `SPEECH_API_KEY` |
| **중앙선관위 (NEC)** | 당선인 정보 · 선거공약 정보 | `apis.data.go.kr/9760000` · `DATA_GO_KR_API_KEY` |
| **소상공인 상권정보 (SEMAS)** | 지역 상권 통계 | `apis.data.go.kr/B553077` (data.go.kr) |
| **도로교통공단 TAAS** | 교통사고 통계 | `apis.data.go.kr/B552061` · `opendata.koroad.or.kr` |
| **통계청 KOSIS** | 국가통계 지표 | `kosis.kr/openapi` |
| **중앙여론조사심의위 (NESDC)** | 여론조사 등록자료 | `nesdc.go.kr` |

> 키가 없거나 호출에 실패한 출처는 자동으로 건너뜁니다(graceful degradation) — 일부 공공 API 키 없이도 동작합니다. 생성 품질을 높이려면 `.env`에 해당 키를 설정하세요.

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

[MIT](LICENSE) © 2026 개혁신당 정책국 (Reform Party Policy Bureau)
