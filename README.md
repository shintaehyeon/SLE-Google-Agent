# 🧠 SLE Agent: AI 기반 사내 규정 탐색 시스템

> **Google AI Agent Challenge 프로젝트**
> 사내 규정(System/Service Level Agreement, HR 규정 등)을 검색하고 답변하는 RAG(Retrieval-Augmented Generation) 기반의 AI 에이전트 시스템입니다.

---

## 🛠️ 기술 스택 및 구조
- **Frontend**: Next.js (TypeScript, React)
- **Backend**: FastAPI (Python)
- **Database/Auth**: Supabase

---

# 📌 차기 개발자를 위한 인수인계 가이드
## 🚀 AI 데이터 레이블링(마킹) 및 성능 고도화 전략

본 시스템은 사내 규정 문서를 정확하게 검색하고 답변하는 데 초점이 맞춰져 있습니다. 현재 버전 이후, **답변의 신뢰도와 정확도를 비약적으로 상승시키기 위한 "데이터 레이블링(마킹) 도입 가이드"**입니다. 다음 작업자는 본 가이드를 참고하여 시스템을 고도화해 주시기 바랍니다.

---

## 📂 Phase 1. 문서 레이아웃 마킹 및 구조화 (Document Layout Marking)
사내 규정 문서는 표(Table), 항목 기호, 그리고 다단계 조항 구조(제N조 제N항)로 되어 있어 일반적인 텍스트 파싱으로는 검색 정확도에 한계가 있습니다.

### 1. 수행 작업
- PDF 규정 문서의 영역별 레이아웃 마킹 작업을 수행합니다.
- **마킹 종류:** `Title` (제목), `Header` (소제목), `Paragraph` (본문), `Table` (표), `List` (항목 리스트)

### 2. 기술 제안
- **LayoutParser** 또는 **Surya**, **Unstructured** 라이브러리를 도입하여 문서의 시각적 요소를 인식합니다.
- 표(Table) 영역은 마킹 후 `Markdown`이나 `HTML` 형식으로 변환하여 벡터 DB에 임베딩해야 AI가 표 데이터를 정확하게 매핑할 수 있습니다.

> [!TIP]
> **청크 분할 전략:** 문서를 고정된 글자 수(예: 500자)로 쪼개지 말고, 레이블링된 **조항(제N조) 단위로 구조적 청크 분할**을 진행하는 것이 좋습니다.

---

## 🎯 Phase 2. 평가용 골드 데이터셋(Gold Standard) 구축 및 자동 평가
에이전트의 답변 품질을 지속적으로 측정하고 개선하기 위해서는 기준 정답 세트가 필요합니다.

### 1. 수행 작업
1. **질문 생성:** 규정 파일별로 자주 묻는 질문(FAQ) 100~200개를 생성합니다.
2. **골드 레이블링:** 인사/노무/총무 등 사내 도메인 전문가(또는 담당자)가 각 질문에 대해 **"가장 완벽한 정답 답변"**과 **"정답이 명시된 정확한 출처 파일 및 조항"**을 매칭하여 수동으로 레이블링합니다.

### 2. 성능 평가 자동화 (RAG Evaluation)
- 구축된 골드 데이터셋을 바탕으로 **Ragas**나 **LLM-as-a-judge** 프레임워크를 도입합니다.
- 매번 프롬프트를 수정하거나 모델을 교체할 때마다 아래 지표를 자동 평가합니다.
  - **Faithfulness (충실도):** 답변이 문서 내용에만 기반하고 있는지 (환각 방지)
  - **Answer Relevance (답변 유사도):** 사용자의 질문 의도에 맞는 답변을 했는지
  - **Context Recall (검색 재현율):** 정답이 있는 청크를 벡터 DB에서 잘 찾아왔는지

---

## 🔁 Phase 3. 사용자 피드백 루프 및 RLHF 시스템 구축
사용자가 에이전트와 대화하면서 생성되는 실시간 피드백을 수집하여 데이터를 고도화하는 휴먼인더루프(Human-in-the-Loop) 아키텍처를 구현합니다.

### 1. UI/UX 확장 제안 (`app/chat/page.tsx`)
- AI 답변 말풍선 하단에 `👍 (도움이 됨)` / `👎 (도움이 안 됨)` 버튼을 추가합니다.
- **싫어요(👎) 클릭 시:** 피드백 팝업창을 띄워 사용자가 정정 사유를 입력할 수 있도록 합니다.

```
+-------------------------------------------------------------+
| [AI 답변] 육아휴직은 최대 1년까지 신청 가능합니다. ...        |
|                                            [ 👍 ]  [ 👎 ]   |
+-------------------------------------------------------------+
```

### 2. Supabase 피드백 데이터베이스 스키마 설계
수집된 피드백 데이터는 Supabase에 적재하여 향후 AI 개선용 데이터셋으로 활용합니다.

```sql
CREATE TABLE agent_feedbacks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  question TEXT NOT NULL,
  ai_answer TEXT NOT NULL,
  feedback_type VARCHAR(10) CHECK (feedback_type IN ('good', 'bad')),
  comment TEXT, -- 사용자가 작성한 상세 피드백 또는 정정 요청 내용
  corrected_answer TEXT, -- [관리자 전용] 관리자가 수정한 최종 정답 답변
  is_resolved BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT TIMEZONE('utc'::text, NOW())
);
```

### 3. 관리자 페이지 고도화 (`app/admin`)
- 관리자 페이지에서 `feedback_type = 'bad'`이고 `is_resolved = false`인 항목들을 필터링하여 보여줍니다.
- 인사담당자가 해당 질문에 대해 `corrected_answer`를 작성하고 저장하면, 이 데이터는 즉시 **임베딩 검색의 우선순위 캐시(Cache)** 또는 **차기 Fine-tuning 학습 데이터**로 활용됩니다.

> [!IMPORTANT]
> 이 피드백 데이터가 누적되면, 향후 OpenAI나 Anthropic 등의 LLM 모델을 사내 데이터에 완벽히 정렬(Alignment)시키는 **RLHF(실시간 인간 피드백 기반 강화학습)** 학습 자료로 직결되는 엄청난 자산이 됩니다.

---

## 🏃‍♂️ 시작하기 (Development Setup)

### Frontend (Next.js)
```bash
cd sle-agent
npm install
npm run dev
```

### Backend (FastAPI)
```bash
cd sle-agent/backend
pip install -r requirements.txt
uvicorn main:app --reload
```
