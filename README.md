# 📝 MemoDB

LangChain + ChromaDB + OpenAI를 활용한 자연어 기반 개인 메모 관리 챗봇입니다.  
Streamlit UI로 채팅하듯 메모를 저장·조회·수정·삭제할 수 있습니다.

---

## ⚡ 빠른 시작 (uv)

### 1. uv 설치 (없다면)
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. 프로젝트 세팅
```bash
# 저장소 클론 후 프로젝트 폴더로 이동
cd memodb

# 가상환경 생성 + 패키지 설치 (한 번에)
uv sync

# 가상환경 활성화
source .venv/bin/activate   # macOS / Linux
# .venv\Scripts\activate    # Windows
```

### 3. API 키 설정
.env `OPENAI_API_KEY`에 본인 키를 입력합니다.


### 4. 실행
```bash
uv run streamlit run MemoDB.py
```

---

## 📁 디렉토리 구조

```
memodb/
├── MemoDB.py           # 메인 애플리케이션
├── pyproject.toml      # uv 패키지 설정
├── db/                 # 일반 메모 ChromaDB 저장소 (자동 생성)
└── db_link/            # 링크 메모 ChromaDB 저장소 (자동 생성)
```

---

## 💬 사용법

채팅창에 **`<인텐트> 내용`** 형식으로 입력합니다.

| 인텐트 | 예시 | 설명 |
|---|---|---|
| `<save>` | `<save> 내일 오전 10시 회의` | 텍스트 메모 저장 |
| `<save>` | `<save> https://example.com` | URL 페이지 저장 |
| `<show>` | `<show> ` | 저장된 메모 전체 출력 |
| `<qa>` | `<qa> 내일 회의 몇 시야?` | 메모에서 질문에 답변 |
| `<del>` | `<del> 내일 회의` | 관련 메모 삭제 |
| `<update>` | `<update> 내일 회의 시간이 오후 2시로 변경됨` | 관련 메모 수정 |
| `<summarize>` | `<summarize> https://example.com` | URL 페이지 요약 후 저장 |

---

## 🔧 함수(모듈)별 설명

### `split_intent_passage(query)`
사용자 입력에서 **인텐트 태그**(`<save>`, `<qa>` 등)와 **내용**을 분리합니다.
- `<` `>` 기호를 파싱해 인텐트와 본문을 튜플로 반환
- 태그가 없으면 `(None, query)` 반환

---

### `save_input(query, db)`
**일반 텍스트 메모**를 ChromaDB에 저장합니다.
- `RecursiveCharacterTextSplitter`로 긴 텍스트를 청크로 분할
- 분할된 텍스트를 벡터 임베딩 후 `db`에 저장

---

### `save_link(query, db)`
**URL 페이지**를 크롤링해 ChromaDB에 저장합니다.
- `SeleniumURLLoader`로 웹 페이지 내용 로드 (JS 렌더링 지원)
- 텍스트 분할 후 `db_link`에 저장

---

### `summarize_link(query, db)`
**URL 페이지를 요약**하고 ChromaDB에도 저장합니다.
- `SeleniumURLLoader`로 페이지 로드 및 저장
- `map_reduce` 체인으로 요약 생성
  - **Map**: 각 청크를 개별 요약
  - **Combine**: 전체를 합쳐 최종 요약 (제목·링크·핵심내용·불릿포인트 형식)

---

### `retriv_one(query)`
쿼리와 **가장 유사한 메모 1개**를 검색해 반환합니다.
- ChromaDB similarity search로 관련 문서 1개 조회
- 상위 200자를 리스트로 반환
- `<del>`, `<update>` 전처리 단계에서 대상 문서 특정 용도로 사용

---

### `retrieval_answer(query, db)`
저장된 메모를 근거로 **질문에 답변**합니다.
- `RetrievalQA` 체인 사용 (RAG 패턴)
- 가장 관련성 높은 문서 1개를 컨텍스트로 활용
- 답변(answer)과 출처 문서(source) 함께 반환

---

### `update(memo, information)`
기존 메모를 새 정보로 **업데이트**합니다.
- GPT-3.5-turbo에 원본 메모와 변경 정보를 전달
- Few-shot 프롬프팅으로 자연스럽게 수정된 메모 생성
- 내부적으로 기존 메모 삭제 → 수정된 내용 재저장

---

### `generate_response(intent, passage)`
인텐트에 따라 **적절한 함수를 라우팅**하고 응답 리스트를 반환합니다.
- 6개 인텐트(`<save>`, `<del>`, `<qa>`, `<show>`, `<update>`, `<summarize>`) 분기 처리
- `<save>` 시 https 여부로 URL / 일반 텍스트를 자동 구분

---

### `split_session_state(messages)`
Streamlit 세션의 메시지 리스트를 **동일 role 기준으로 그룹핑**합니다.
- 연속된 같은 role의 메시지를 하나의 말풍선으로 묶어 UI 렌더링에 활용

---

## 🔄 전체 작동 플로우

```
사용자 입력 (채팅창)
      │
      ▼
split_intent_passage()
  ├─ 인텐트 없음 → "No intent detected." 반환
  └─ (인텐트, 내용) 분리
          │
          ▼
    generate_response()
          │
    ┌─────┼─────────────────────────────┐
    │     │                             │
  <save> <qa>                  <del>/<update>
    │     │                             │
    │  retrieval_answer()        retriv_one()
    │  (RAG 답변)                (유사 메모 검색)
    │                                   │
  https?                           similarity_search()
  ├─ Yes → save_link()          → db.delete()
  └─ No  → save_input()         └─ update() → save_input()
    │
  <show> → db.get() 전체 조회
    │
  <summarize> → summarize_link() (크롤링 + 요약 + 저장)
          │
          ▼
    응답 리스트 반환
          │
          ▼
  Streamlit 채팅 UI 렌더링
  (st.session_state에 메시지 추가)
```

---

## ⚠️ 주의사항

- `SeleniumURLLoader`는 Chrome WebDriver가 필요합니다. ChromeDriver를 별도 설치하거나 `webdriver-manager` 패키지를 추가하세요.
- ChromaDB는 실행 디렉토리에 `db/`, `db_link/` 폴더를 자동 생성합니다.
- OpenAI API 사용 비용이 발생합니다.

---

## 🛠️ 주요 변경사항 (2023 → 2024+ 버전)

| 항목 | 이전 | 변경 후 |
|---|---|---|
| LangChain import | `langchain.llms` | `langchain_openai` |
| OpenAI Python SDK | `openai.ChatCompletion.create()` | `client.chat.completions.create()` |
| Retriever 호출 | `get_relevant_documents()` | `retriever.invoke()` |
| QA 체인 호출 | `qa(query)` | `qa.invoke({"query": ...})` |
| VectorStore import | `langchain.vectorstores` | `langchain_community.vectorstores` |
| Document loaders | `langchain.document_loaders` | `langchain_community.document_loaders` |
