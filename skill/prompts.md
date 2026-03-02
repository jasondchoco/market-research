# Market Research Agent — 에이전트 프롬프트 템플릿

각 섹션의 `{변수}`를 실제 값으로 대입하여 Agent tool의 prompt에 전달.
모든 에이전트의 subagent_type은 `general-purpose`.

---

## TREND_SCANNER

**model:** haiku

**prompt:**

주제: {TOPIC}
출처 모드: {SOURCE_MODE}

아래 키워드를 중심으로 Product Hunt, Hacker News, Twitter/X에서 검색하세요.

검색 키워드:
{SEARCH_KEYWORDS}

WebSearch를 3-5회 수행하세요:
1. "site:producthunt.com {TOPIC_EN} frustration OR problem OR wish 2025 2026"
2. "site:news.ycombinator.com {TOPIC_EN} pain point OR annoying OR broken"
3. "{TOPIC_EN} Twitter complaint OR rant OR wish there was"
4. (선택) "Product Hunt {TOPIC_EN} alternatives disappointment"
5. (선택) "{TOPIC_EN} tool fatigue OR app overload"

**출처 규칙:**
- 모든 불만/니즈에 발견 URL을 기록하세요.
- {SOURCE_MODE}=inline_citation이면: 모든 인용에 [Source: URL] 태그 필수.

에이전트 정의 파일의 출력 형식을 엄수하세요.

---

## COMMUNITY_ANALYST

**model:** haiku

**prompt:**

주제: {TOPIC}
출처 모드: {SOURCE_MODE}

아래 키워드를 중심으로 Reddit, 블라인드에서 검색하세요.

검색 키워드:
{SEARCH_KEYWORDS}

WebSearch를 3-5회 수행하세요:
1. "site:reddit.com r/productivity OR r/WorkReform {TOPIC_EN} frustrated OR hate OR wish"
2. "site:reddit.com {TOPIC_EN} existing tools suck OR doesn't work OR gave up"
3. "블라인드 {TOPIC_KR} 불만 OR 스트레스 OR 비효율"
4. (선택) "site:reddit.com {TOPIC_EN} would pay for OR shut up and take my money"
5. (선택) "{TOPIC_KR} 직장인 고충 OR 업무 도구 불만 2025 2026"

**출처 규칙:**
- 모든 불만/니즈에 발견 URL을 기록하세요.
- {SOURCE_MODE}=inline_citation이면: 모든 인용에 [Source: URL] 태그 필수.

에이전트 정의 파일의 출력 형식을 엄수하세요.

---

## DEEP_RESEARCHER

**model:** sonnet

**prompt:**

주제: {TOPIC}
조사 깊이: {DEPTH}
출처 모드: {SOURCE_MODE}

아래는 Phase 1에서 수집된 트렌드와 커뮤니티 분석의 압축본입니다:

--- Phase 1 압축본 ---
{PHASE1_SUMMARY}
---

당신의 임무는 위 니즈들이 **진짜 미충족인지 반증**하는 것입니다.

### A. 니즈별 경쟁사 검증 (필수, 최소 5회):

1. "G2 {니즈 카테고리} software"
2. "Capterra {니즈 키워드} tools best"
3. "site:producthunt.com {니즈 키워드} 2025 2026"
4. "{니즈 키워드} app OR tool OR platform pricing"
5. "{TOPIC_KR} 관련 앱 OR 서비스 OR 솔루션"
6. (선택) "[발견된 제품명] review negative OR limitation"
7. (선택) "AlternativeTo [주요 제품명]"

### B. 시장 규모 + 경쟁 환경 (필수 2-3회):

8. "{TOPIC_EN} market size 2025 2026 forecast report"
9. "{TOPIC_EN} TAM SAM total addressable market"
10. "{TOPIC_KR} 시장 규모 OR 시장 전망 2025 2026"

### C. Consulting 전용 (DEPTH=consulting일 때만, 추가 3-5회):

11. "{TOPIC_EN} growth drivers trends 2025 2026"
12. "{TOPIC_EN} regulation risk challenge"
13. "{TOPIC_EN} customer segments market breakdown"
14. "{TOPIC_EN} value chain industry structure"
15. (선택) "[주요 경쟁사] revenue users funding"

**출처 규칙:**
- {SOURCE_MODE}=url_per_need: 각 니즈별 URL 1-3개 + TAM 출처 명시
- {SOURCE_MODE}=inline_citation: **모든 숫자/통계에 [Source: URL] 필수.** 출처 없으면 "추정:" 표시.
  - 출처 Tier 분류 필수: `[Source: URL] (T1)` 형식
  - T1 (Authoritative): 정부 통계, 시장 리포트, 학술 논문, 기업 공시
  - T2 (Industry): G2, Capterra, Product Hunt, 뉴스, 업계 블로그
  - T3 (Community): Reddit, HN, Blind, Twitter/X, 커뮤니티

에이전트 정의 파일의 출력 형식을 엄수하세요.

---

## KOREA_SPECIALIST

**model:** sonnet

**prompt:**

주제: {TOPIC}
출처 모드: {SOURCE_MODE}

아래는 Phase 1에서 수집된 트렌드와 커뮤니티 분석의 압축본입니다:

--- Phase 1 압축본 ---
{PHASE1_SUMMARY}
---

위 니즈들을 한국 시장 맥락에서 분석하세요.

WebSearch를 2-3회 수행하세요:
1. "{TOPIC_KR} 한국 시장 트렌드 2025 2026"
2. "블라인드 OR 네이버카페 {TOPIC_KR} 불편 OR 개선"
3. (선택) "{TOPIC_KR} 한국 스타트업 OR 서비스 B2C"

**출처 규칙:** {SOURCE_MODE}=inline_citation이면 모든 인용에 [Source: URL] 필수.

에이전트 정의 파일의 출력 형식을 엄수하세요.

---

## LOCAL_SPECIALIST

**model:** sonnet

**prompt:**

주제: {TOPIC}
타겟 로케일/시장: {LOCALE}
출처 모드: {SOURCE_MODE}

아래는 Phase 1에서 수집된 트렌드와 커뮤니티 분석의 압축본입니다:

--- Phase 1 압축본 ---
{PHASE1_SUMMARY}
---

위 니즈들을 {LOCALE} 시장 맥락에서 분석하세요. 고려 사항:
- 문화적 차이, 로컬 규제/인프라
- 영어 검색에서 미발견 로컬 경쟁자
- 결제/가격 규범, 고유 로컬 니즈

{LOCALE} 시장의 로컬 언어 키워드로 WebSearch 2-3회 수행하세요.

에이전트 정의 파일의 출력 형식을 엄수하세요.

---

## STRATEGIST

**model:** {STRATEGIST_MODEL} (standard=sonnet, consulting=opus)

**prompt:**

주제: {TOPIC}
조사 깊이: {DEPTH}
검색 횟수: 약 {SEARCH_COUNT}회
평가 소스 수: 약 {SOURCES_COUNT}개

아래는 Phase 1+2의 전체 분석 결과를 정규화한 테이블입니다:

--- 정규화 분석 테이블 ---
{NORMALIZED_TABLE}
---

위 데이터를 기반으로 작성하세요:

**standard:** 4개 섹션 — 점수카드, 시장 규모 분석, 액션플랜(상위 3개), PRD 시드(1위)
**consulting:** 7개 섹션 — 위 4개 + Risk Matrix, 세그먼테이션 정리, Methodology

에이전트 정의 파일의 출력 형식을 엄수하세요.

**절대 규칙:**
- WebSearch 하지 마세요. 제공된 데이터만 사용.
- 점수는 반드시 근거와 함께.
- 리서치에 없는 기능/시장 정보를 상상하지 마세요.

---

## 변수 목록

| 변수 | 설명 | 설정 시점 |
|------|------|----------|
| `{TOPIC}` | 조사 주제 (한국어) | 사용자 입력 |
| `{TOPIC_EN}` | 조사 주제 (영어) | 리더 번역 |
| `{TOPIC_KR}` | 한국어 검색 키워드 | 리더 생성 |
| `{SEARCH_KEYWORDS}` | 혼합 키워드 목록 | 리더 생성 |
| `{PHASE1_SUMMARY}` | Phase 1 압축본 (≤800단어) | Phase 1 후 |
| `{NORMALIZED_TABLE}` | 정규화 테이블 (≤1200/1800단어) | Phase 2 후 |
| `{DEPTH}` | quick/standard/consulting | 깊이 선택 |
| `{SOURCE_MODE}` | url_collect/url_per_need/inline_citation | 깊이에서 파생 |
| `{STRATEGIST_MODEL}` | sonnet 또는 opus | 깊이에서 파생 |
| `{SEARCH_COUNT}` | 수행된 WebSearch 횟수 합산 | Phase 2 후 집계 |
| `{SOURCES_COUNT}` | 평가된 소스 수 합산 | Phase 2 후 집계 |
| `{LOCALE}` | none/kr/커스텀 | 로케일 선택 |
| `{TODAY}` | 오늘 날짜 | 자동 |
| `{SLUG}` | 주제 슬러그 | 리더 생성 |
