---
name: market-research
description: "Use when user wants market research, opportunity analysis, or competitive landscape study. /market-research [topic] to run."
---

# Market Research Agent Team

## 상수

- MR_BASE: 아래 순서로 출력 디렉토리 결정:
  1. `$MR_OUTPUT_DIR` 환경변수 (설정된 경우)
  2. `/Users/jason/Library/Mobile Documents/com~apple~CloudDocs/project/Market_Research` (기본값)
- PROMPTS: `~/.claude/skills/market-research/prompts.md`

## 에이전트 구성

| Agent | Model (depth별) | 역할 |
|-------|-----------------|------|
| Team Leader | 메인 | 오케스트레이션, 압축, 파일 저장 |
| Trend Scanner | haiku | PH, HN, X 트렌드 수집 |
| Community Analyst | haiku | Reddit, 블라인드 불만 분석 |
| Deep Researcher | sonnet | 경쟁 검증 + TAM/SAM/SOM + (consulting: market dynamics, value chain) |
| Local Specialist | sonnet | 로컬 시장 특수성 분석 |
| Strategist | sonnet/opus | 점수카드 + 액션플랜 + (consulting: risk matrix, segmentation) |

## 시작

**Announce:** "시장조사 에이전트 팀을 시작합니다."

Read PROMPTS 파일로 에이전트 프롬프트 템플릿 로드.

## 1단계: 입력 수집

### 1-1. 주제 확인

`$ARGUMENTS`가 있으면 주제로 사용. 없으면 AskUserQuestion:
"어떤 주제/영역을 조사할까요? (예: 개인 생산성, 수면 관리, 재테크)"

### 1-2. 변수 생성

주제를 받으면 리더가 직접 생성:
- `{TOPIC}`: 한국어 주제 (사용자 입력 그대로)
- `{TOPIC_EN}`: 영어 번역
- `{TOPIC_KR}`: 한국어 검색 키워드 3-5개
- `{SEARCH_KEYWORDS}`: 영어+한국어 혼합 키워드 목록
- `{SLUG}`: 영문 하이픈 슬러그 (예: personal-productivity)
- `{TODAY}`: 오늘 날짜

### 1-3. 캐시 확인

Bash로 `ls {MR_BASE}/{SLUG}/meta.md` 확인.

존재하면 Read로 meta.md 읽고 AskUserQuestion:
- **전체 재조사** — Phase 1부터 새로 시작
- **업데이트 조사** (Recommended) — 이전 Phase 1 결과 재사용, Phase 2부터 실행
- **이전 결과 보기** — 기존 report.md 출력만

"업데이트 조사" 선택 시 → `{CACHED_PHASE1}` = Read `raw/trend-scan.md` + `raw/community.md`, Phase 1 스킵.

존재하지 않으면 → 다음 단계 진행.

### 1-4. 로케일 선택

AskUserQuestion:
- **글로벌 전용** — 영어 중심 리서치
- **한국 시장** (Recommended) — +Local Specialist, 블라인드/네이버 분석 포함
- **커스텀 로케일** — 사용자가 타겟 시장 지정

`{LOCALE}` = `none` / `kr` / `{custom}` 설정.

### 1-5. 조사 깊이 선택

AskUserQuestion:

- **Quick Scan** — "빠른 탐색: 이 주제가 조사할 가치가 있는지 확인. ~3-5분, ~30-50K 토큰. Phase 1만 실행, 니즈 목록 + 기본 우선순위."
- **Standard** (Recommended) — "표준 조사: 경쟁 검증 + TAM/SAM/SOM + 점수카드 + 액션플랜. ~10-15분, ~100-150K 토큰. 사이드 프로젝트 / 아이디어 검증용."
- **Consulting Grade** — "컨설팅 레벨: 모든 주장에 출처 각주 + 시장 동인/리스크 + 세그먼테이션 + Value Chain + Risk Matrix + Methodology. ~20-30분, ~250-350K 토큰. 투자자 피칭 / 정식 사업 검토용."

`{DEPTH}` = `quick` / `standard` / `consulting` 설정.

**깊이에 따른 파생 변수:**

| 변수 | quick | standard | consulting |
|------|-------|----------|-----------|
| `{STRATEGIST_MODEL}` | - | sonnet | opus |
| `{PHASES}` | P1 | P1+P2+P3 | P1+P2(확장)+P3(확장) |
| `{DR_SEARCHES}` | - | 7-10회 | 12-15회 |
| `{SOURCE_MODE}` | url_collect | url_per_need | inline_citation |
| `{EXTRA_SECTIONS}` | - | - | market_dynamics, segmentation, value_chain, risk_matrix, methodology |

---

## 2단계: Phase 1 — 트렌드 + 커뮤니티 수집

**디렉토리 생성 (최초 1회):**
```bash
mkdir -p "{MR_BASE}/{SLUG}/raw"
```

**캐시된 Phase 1이 있으면 이 단계 스킵.**

### 2-1. 병렬 dispatch

단일 메시지에 두 Agent tool을 동시 호출:

1. **Agent(market-trend-scanner)**
   - subagent_type: `general-purpose`
   - model: `haiku`
   - prompt: PROMPTS의 `## TREND_SCANNER` 템플릿에 변수 대입. `{SOURCE_MODE}` 전달.

2. **Agent(market-community-analyst)**
   - subagent_type: `general-purpose`
   - model: `haiku`
   - prompt: PROMPTS의 `## COMMUNITY_ANALYST` 템플릿에 변수 대입. `{SOURCE_MODE}` 전달.

두 결과를 수신할 때까지 대기.

### 2-2. 중간 저장

각 에이전트 결과를 Write:
- `{MR_BASE}/{SLUG}/raw/trend-scan.md`
- `{MR_BASE}/{SLUG}/raw/community.md`

### 2-3. Phase 1 압축

리더가 두 결과를 합산하여 **중복 제거 + 압축** 수행.
결과: `{PHASE1_SUMMARY}` (≤800단어)

압축 규칙:
- 동일한 니즈가 양쪽에 있으면 하나로 병합, 출처 URL 모두 유지
- 빈도 판단: 양쪽 모두 발견 → "높음", 한쪽만 → "중간"
- 인용은 가장 강력한 것 1개만 유지
- 트렌드 키워드와 지불 의사 증거는 별도 섹션으로 보존
- **모든 사실 주장에 [Source: URL] 태그 유지** (consulting이 아니어도 URL은 보존)

### 2-4. Quick Scan 분기

**`{DEPTH}` = `quick`이면:**
- 리더가 Phase 1 압축본으로 **간이 우선순위 테이블** 생성
- 점수카드 없이 빈도 × 감정 강도로 순위만 매김
- → 5단계로 건너뜀 (quick용 축소 출력)

**`{DEPTH}` = `standard` 또는 `consulting`이면:**
- → 3단계로 진행

---

## 3단계: Phase 2 — 심층 분석

**`{DEPTH}` = `quick`이면 이 단계 전체 스킵.**

### 3-1. 병렬 dispatch (로케일 + 깊이 조건부)

**`{LOCALE}` = `none`인 경우:** Deep Researcher만 단독 dispatch.

**`{LOCALE}` = `kr` 또는 custom인 경우:** 두 Agent tool을 동시 호출.

1. **Agent(market-deep-researcher)**
   - subagent_type: `general-purpose`
   - model: `sonnet`
   - prompt: PROMPTS `## DEEP_RESEARCHER` 템플릿에 `{PHASE1_SUMMARY}`, `{DEPTH}`, `{SOURCE_MODE}` 대입

2. **로케일 에이전트** (로케일 선택 시)
   - `kr` → **Agent(market-korea-specialist)**, prompt: PROMPTS `## KOREA_SPECIALIST`
   - custom → **Agent(market-local-specialist)**, prompt: PROMPTS `## LOCAL_SPECIALIST`
   - subagent_type: `general-purpose`, model: `sonnet`

결과 수신까지 대기.

### 3-2. 중간 저장

에이전트 결과를 Write:
- `{MR_BASE}/{SLUG}/raw/deep-research.md`
- `{MR_BASE}/{SLUG}/raw/local-market.md` (로케일 선택 시)

### 3-3. 정규화 테이블 병합

리더가 Phase 1 압축본 + Phase 2 결과를 정규화 테이블로 병합.
결과: `{NORMALIZED_TABLE}` (standard ≤1200단어, consulting ≤1800단어)

정규화 형식:

```
## 정규화 분석 테이블

### 니즈 목록

| # | 니즈 | 페르소나 | 빈도 | 현재 workaround | 기존 솔루션 (전수) | 실패 원인 | Gap Reality | 로컬 특수성 | WTP 증거 |
|---|------|----------|------|----------------|-------------------|----------|------------|------------|----------|
| 1 | ...  | ...      | ...  | ...            | ...               | ...      | 🔴/🟡/🟢   | ...        | ...      |

> Gap Reality: 🔴 No Gap / 🟡 Partial Gap / 🟢 Real Gap

### 경쟁 환경 요약
| 니즈 | Gap Reality | 경쟁 강도 | 공백 정의 | 핵심 근거 |
|------|------------|----------|----------|----------|

### 경쟁 환경 맵
| 카테고리 | 주요 플레이어 | 포지셔닝 | 가격대 | 강점 | 약점 |
|---------|-------------|---------|--------|------|------|
경쟁 강도: [포화/중간/초기]

### 시장 규모 (TAM/SAM/SOM)
| 구분 | 규모 | 산출 근거 | 출처 |
|------|------|----------|------|
| TAM | $[X]B | ... | [Source: URL] |
| SAM | $[X]M | ... | [Source: URL] |
| SOM | $[X]M | ... | ... |
성장률: CAGR [X]%
한국 시장: [별도 기재]

### 출처 레지스트리 (consulting 전용)
| # | URL | 제목/설명 | 유형 | Tier |
|---|-----|---------|------|------|
| [1] | https://... | 시장 리포트명 | report/article/review/forum | T1/T2/T3 |
```

**consulting 추가 섹션 (정규화 테이블에 추가):**

```
### 시장 동인 & 리스크
| 유형 | 요인 | 영향도 | 근거 [Source: #] |
|------|------|--------|----------------|
| Growth Driver | ... | 높음/중간 | ... |
| Headwind | ... | 높음/중간 | ... |
| Regulatory | ... | ... | ... |

### 고객 세그먼테이션
| 세그먼트 | 규모 추정 | 특징 | WTP | 현재 솔루션 |
|----------|----------|------|-----|-----------|

### Value Chain
[산업의 가치 사슬에서 어디에 기회가 있는지]
```

---

## 4단계: Phase 3 — 전략 수립

**`{DEPTH}` = `quick`이면 이 단계 전체 스킵.**

### 4-1. Strategist dispatch

**Agent(market-strategist)**
- subagent_type: `general-purpose`
- model: `{STRATEGIST_MODEL}` (standard=sonnet, consulting=opus)
- prompt: PROMPTS의 `## STRATEGIST` 템플릿에 `{NORMALIZED_TABLE}`, `{DEPTH}` 대입

### 4-2. 중간 저장

결과를 Write:
- `{MR_BASE}/{SLUG}/raw/strategy.md`

---

## 5단계: 마무리

### 5-1. 인터랙티브 HTML 리포트 생성

리더가 전체 결과를 **JSON 객체**로 구조화한 뒤 HTML 템플릿에 삽입.

#### 5-2-1. JSON 데이터 구성

```json
{
  "topic": "{TOPIC}",
  "topic_en": "{TOPIC_EN}",
  "date": "{TODAY}",
  "depth": "quick|standard|consulting",
  "locale": "{LOCALE}",
  "agents_used": 5,
  "executive_summary": "전체 결과 3-5문장 요약",

  "market_sizing": {
    "tam": { "value": "$XB", "basis": "산출 근거", "source": "출처", "ref": 1 },
    "sam": { "value": "$XM", "basis": "좁힌 기준", "source": "출처", "ref": 2 },
    "som": { "value": "$XM", "basis": "점유율 가정", "source": "출처" },
    "cagr": "X%",
    "cagr_period": "20XX-20XX",
    "korea_market": "한국 시장 규모",
    "data_quality": "high|medium|low"
  },

  "competitive_landscape": [
    {
      "category": "직접 경쟁|간접 대체재|로컬",
      "players": ["제품A", "제품B"],
      "positioning": "포지셔닝",
      "price_range": "$X-Y/월",
      "strengths": "강점",
      "weaknesses": "약점"
    }
  ],
  "competition_intensity": "포화|중간|초기",
  "market_gap_summary": "시장 공백 요약",

  "needs": [
    {
      "id": 1,
      "title": "니즈 제목",
      "persona": "페르소나",
      "frequency": "높음/중간/낮음",
      "workaround": "현재 대처",
      "existing_solutions": [
        { "name": "제품명", "pricing": "가격대", "target": "개인/SMB/Enterprise", "limitation": "한계" }
      ],
      "failure_reason": "실패 원인",
      "gap_reality": "real|partial|none",
      "gap_definition": "Gap 판정 근거",
      "local_specifics": "로컬 특수성",
      "wtp_evidence": "지불 의사 증거",
      "sources": ["URL1", "URL2"]
    }
  ],

  "scorecard": [
    {
      "rank": 1,
      "problem": "문제명",
      "market": 9, "urgency": 8, "competition": 3, "barrier": 4, "wtp": 8,
      "total": 38,
      "rationale": { "시장성 9/10": "근거 [1]", ... }
    }
  ],

  "action_plan": [
    {
      "problem": "문제명",
      "persona": "구체적 인물상",
      "validation": "검증 방법",
      "mvp_scope": "핵심 기능",
      "timeline": "타임라인",
      "risks": "리스크",
      "revenue_model": "수익 모델"
    }
  ],

  "prd_seed": {
    "problem": "핵심 문제",
    "target_user": "타겟 사용자",
    "frequency_severity": "빈도와 심각도",
    "market_size": "TAM/SAM/SOM 요약",
    "community_evidence": "커뮤니티 불만 요약",
    "solution_failures": "기존 솔루션 실패 원인",
    "wtp_evidence": "지불 의사 증거",
    "value_prop": "핵심 가치 제안",
    "must_have": ["기능1", "기능2"],
    "nice_to_have": ["기능3"],
    "anti_scope": ["하지 않을 것"],
    "kpis": ["KPI 1", "KPI 2"],
    "next_steps": ["사용자 인터뷰 N명", "경쟁사 심층 분석", "정식 PRD 작성"]
  },

  "sources_registry": [
    { "id": 1, "url": "https://...", "title": "제목", "type": "report|article|review|forum", "tier": "T1|T2|T3", "accessed": "{TODAY}" }
  ],

  "methodology": {
    "approach": "5-agent parallel research pipeline",
    "search_count": 25,
    "sources_evaluated": 40,
    "date_range": "2024-2026",
    "limitations": ["AI 검색 기반 — 유료 리포트 미포함", "정량 데이터는 공개 출처 한정"]
  },

  "market_dynamics": {
    "growth_drivers": [
      { "factor": "동인", "impact": "높음|중간", "evidence": "근거 [ref]" }
    ],
    "headwinds": [
      { "factor": "역풍", "impact": "높음|중간", "evidence": "근거 [ref]" }
    ],
    "regulatory": [
      { "factor": "규제", "region": "지역", "impact": "영향", "evidence": "근거" }
    ]
  },

  "segmentation": [
    { "segment": "세그먼트명", "size": "규모", "characteristics": "특징", "wtp": "지불 의사", "current_solution": "현재 솔루션" }
  ],

  "value_chain": {
    "stages": ["단계1", "단계2", "단계3"],
    "opportunity_stage": "기회가 있는 단계",
    "rationale": "이유"
  },

  "risk_matrix": [
    { "risk": "리스크", "probability": "높음|중간|낮음", "impact": "높음|중간|낮음", "mitigation": "완화 방안" }
  ]
}
```

**깊이별 JSON 필드 포함 규칙:**

| 필드 | quick | standard | consulting |
|------|-------|----------|-----------|
| executive_summary | ✅ (간략) | ✅ | ✅ |
| needs | ✅ (sources 없음) | ✅ | ✅ |
| market_sizing | ❌ | ✅ | ✅ (ref 포함) |
| competitive_landscape | ❌ | ✅ | ✅ |
| scorecard | ❌ | ✅ | ✅ |
| action_plan | ❌ | ✅ | ✅ |
| prd_seed | ❌ | ✅ | ✅ |
| sources_registry | ❌ | ❌ | ✅ |
| methodology | ❌ | ❌ | ✅ |
| market_dynamics | ❌ | ❌ | ✅ |
| segmentation | ❌ | ❌ | ✅ |
| value_chain | ❌ | ❌ | ✅ |
| risk_matrix | ❌ | ❌ | ✅ |

**데이터 매핑 규칙:**
- `needs`: Phase 1 압축본 + Phase 2 결과에서 통합
- `market_sizing`, `competitive_landscape`: Deep Researcher 결과에서 추출
- `scorecard`, `action_plan`, `prd_seed`: Strategist 결과에서 직접 추출
- `sources_registry`: consulting 전용 — 모든 에이전트 결과에서 [Source: URL] 태그를 파싱하여 번호 부여 + Tier 분류
  - **T1 (Authoritative)**: 정부 통계, 시장 리포트 (Gartner, Grand View Research 등), 학술 논문, 기업 공시
  - **T2 (Industry)**: G2/Capterra, Product Hunt, TechCrunch, 업계 분석 블로그, 뉴스
  - **T3 (Community)**: Reddit, HN, Blind, Twitter/X, 네이버 카페, 개인 블로그
- `methodology`, `market_dynamics`, `segmentation`, `value_chain`, `risk_matrix`: consulting 전용 — Deep Researcher + Strategist 결과에서 추출
- `gap_reality` 값: 🟢→`"real"`, 🟡→`"partial"`, 🔴→`"none"`
- `data_quality`: 시장 리포트 직접 인용→`"high"`, 인접 시장 추론→`"medium"`, 데이터 부족→`"low"`

#### 5-2-2. HTML 생성

1. Read `~/.claude/skills/market-research/report-template.html` 로 템플릿 로드
2. 템플릿의 `{{DATA_JSON}}` 을 위에서 만든 JSON 객체로 교체
3. 템플릿의 `{{TOPIC}}` 을 `{TOPIC}` 으로 교체
4. Write `{MR_BASE}/{SLUG}/report.html`

#### 5-1-3. Markdown 파일도 병행 저장 (아카이브용)

리더가 md 파일 Write (깊이에 따라 파일 수 다름):

**모든 깊이:**
1. **report.md** — 메인 보고서 (Executive Summary + 문제별 상세)

**standard + consulting:**
2. **scorecard.md** — 점수카드
3. **action-plan.md** — 액션플랜
4. **prd-seed.md** — PRD 시드 (하단: `> 다음 단계: product-manager 에이전트로 정식 PRD 작성`)

**consulting 전용:**
5. **references.md** — 전체 출처 목록 (Tier별 그룹, 번호, URL, 유형)
6. **methodology.md** — 조사 방법론

### 5-2. 메타데이터 저장

Write `{MR_BASE}/{SLUG}/meta.md`:

```
---
topic: {TOPIC}
slug: {SLUG}
created: {TODAY}
updated: {TODAY}
depth: {DEPTH}
locale: {LOCALE}
status: completed
tags: [{자동 생성 태그}]
agents_used: [{사용된 에이전트 목록}]
---
```

### 5-3. 결과 안내

"조사 완료! 결과 저장: `{MR_BASE}/{SLUG}/`"

Bash로 HTML 리포트를 브라우저에서 열기:
```bash
open "{MR_BASE}/{SLUG}/report.html"
```

AskUserQuestion:
- **보고서 열기** — 브라우저에서 report.html 다시 열기
- **터미널에서 요약 보기** — Executive Summary 텍스트 출력
- **PRD 시드 보기** — prd-seed.md 출력
- **종료** — 세션 종료

---

## 출처 수집 규칙 (SOURCE_MODE)

### url_collect (quick)
- WebSearch 결과의 URL만 수집
- 출력에 포함하지 않음

### url_per_need (standard)
- 각 니즈에 관련 URL 1-3개 첨부
- needs[].sources 배열에 저장
- TAM/SAM/SOM에 source 필드 포함

### inline_citation (consulting)
- **모든 사실 주장에 `[Source: URL]` 인라인 각주 필수**
- 에이전트 프롬프트에 명시: "숫자, 통계, 시장 규모, 사용자 수 등 모든 사실적 주장에 [Source: URL] 태그 필수. 출처 없는 주장은 '추정:' 표시."
- 리더가 모든 [Source: URL]을 파싱하여 `sources_registry`에 번호 부여 + Tier 자동 분류
  - T1: 정부(.go.kr, .gov), 시장 리포트(grandviewresearch, mordorintelligence, gartner, statista), 학술, 기업 공시
  - T2: 리뷰 플랫폼(g2, capterra, producthunt), 뉴스(techcrunch, bloomberg), 업계 블로그
  - T3: 커뮤니티(reddit, news.ycombinator, blind, twitter, naver)
- HTML에서 인라인 각주 [1], [2] 클릭 → References 탭으로 이동 (Tier별 색상 구분)

---

## 에러 처리

- WebSearch 실패 시: 해당 에이전트 결과에 "[검색 실패: {쿼리}]" 표시, 나머지 결과로 계속 진행
- 에이전트 결과가 형식 미준수 시: 리더가 최선으로 파싱하여 진행
- Phase 1 결과가 모두 빈약할 경우: 사용자에게 "검색 결과가 부족합니다. 키워드를 조정할까요?" AskUserQuestion
- TAM/SAM/SOM 데이터를 찾지 못한 경우: `data_quality: "low"` 설정, 인접 시장 추론으로 대체
