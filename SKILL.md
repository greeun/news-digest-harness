---
name: news-digest-harness
version: 1.0.0
description: |
  사용자가 입력한 주제의 최신 뉴스 다이제스트를 Planner → Generator → Evaluator 하네스 패턴으로 생성.
  Anthropic "Harness Design for Long-Running Application Development" (Prithvi Rajasekaran, 2026) 원칙 적용.
  주제는 자유(예: 암호화폐, 반도체, 한국 정치, 기후, K-콘텐츠, 부동산, 테슬라, 바이오).
  웹에서 해당 주제의 최신 뉴스를 수집하고, 한국어로 요약하며, 정갈한 디자인의 self-contained HTML 페이지로 제공.
  트리거 — EN: "news digest about X", "news summary on X", "daily news page for X",
  "curate news on X", "generate news digest for X", "latest X news page".
  KO: "{주제} 뉴스 정리", "{주제} 뉴스 요약", "{주제} 뉴스 모아줘",
  "{주제} 뉴스 다이제스트", "{주제} 소식 정리", "{주제} 관련 뉴스 페이지",
  "오늘의 {주제} 뉴스", "{주제} 트렌드 정리", "최신 {주제} 뉴스",
  "뉴스 다이제스트 만들어줘", "뉴스 요약 페이지 만들어줘".
---

# News Digest Harness (Topic-Agnostic)

사용자가 입력한 주제(예: 암호화폐, 반도체, K-콘텐츠, 부동산, 기후, 바이오, 테슬라, 한국 정치 등)의 최신 뉴스를 웹에서 수집하고, 읽기 쉬운 한국어로 요약하여, 모던하고 정갈한 디자인의 단일 HTML 페이지로 제공하는 하네스 기반 스킬.

Planner가 주제를 확정하고 적절한 카테고리를 도출하여 실제 뉴스를 리서치하고, Generator가 HTML 페이지를 제작하며, Evaluator가 콘텐츠 완성도와 디자인 품질을 독립적으로 검증한다.

## 이 하네스가 방지하는 두 가지 실패

1. **자기평가 편향 (Self-evaluation bias).** 뉴스를 수집하고 HTML을 만드는 에이전트가 스스로 "잘 만들었다"고 판단하면, 누락된 뉴스, 어색한 한국어, 조잡한 디자인, 주제와 무관한 기사 혼입을 놓친다. 별도의 Evaluator가 skeptical하게 검증해야 품질이 보장된다.
2. **컨텍스트 불안 (Context anxiety).** 뉴스 리서치 + HTML 생성 + 디자인 작업이 길어지면 에이전트가 조급하게 마무리하려 한다. 파일 기반 핸드오프와 컨텍스트 리셋으로 이를 방지한다.

## 핵심 설계 철학

> "Find the simplest solution possible, and only increase complexity when needed."

하네스의 모든 컴포넌트는 "모델이 혼자서는 이것을 못 한다"는 가정을 인코딩한다. 그 가정이 틀렸거나 모델 업그레이드로 무효화되면 해당 컴포넌트는 불필요한 복잡도일 뿐이다.

**점진적 단순화 원칙**: 컴포넌트를 제거할 때 한 번에 전부 걷어내지 말 것. 급진적 단순화는 원문에서도 실패했다. **한 번에 하나씩** 제거하고 결과물 품질에 미치는 영향을 측정한 뒤 다음 컴포넌트를 판단한다.

## 비타협 원칙 (Non-negotiable Principles)

1. **구조적 역할 분리**: Planner(주제 해석 + 뉴스 리서치), Generator(HTML 제작), Evaluator(품질 검증)는 반드시 별도 서브에이전트로 실행. 하나의 세션에서 역할극 금지.
2. **파일 기반 핸드오프**: 에이전트 간 통신은 오직 파일(`spec.md`, `sprint_contract.md`, `generator_report.md`, `critique.md`)을 통해서만.
3. **구현 전 스프린트 계약**: Generator는 코딩 전에 무엇을 만들고 어떻게 검증할지 `sprint_contract.md`로 합의.
4. **압축(Compaction) 대신 컨텍스트 리셋**: 압축은 이전 대화를 요약하여 연속성은 유지하지만, **컨텍스트 불안(조급하게 마무리하려는 행동)은 해소하지 못한다.** 컨텍스트 리셋은 완전히 새 세션 + `handoff.md`로 불안 자체를 제거한다.
5. **증거 기반 루브릭 평가**: Evaluator는 "괜찮아 보인다"가 아니라 실제 파일 내용을 인용하며 1-5점으로 채점.
6. **모든 하네스 컴포넌트는 가정을 인코딩**: 모델 업그레이드 시 각 컴포넌트를 **한 번에 하나씩** 제거하며 필요성을 재검증. 한꺼번에 전부 제거하면 어떤 컴포넌트가 실제로 필요했는지 알 수 없다.

## 실행 흐름 (Activation)

사용자가 주제를 포함한 뉴스 다이제스트 요청을 보내면:

### Step 0: 주제 확정 (Topic Clarification)

사용자 요청에서 주제를 추출한다.

- 주제가 명확한 경우(예: "암호화폐 뉴스 정리해줘", "반도체 뉴스 모아줘") → 그대로 사용.
- 주제가 모호한 경우(예: "뉴스 다이제스트 만들어줘", "뉴스 정리해줘") → **사용자에게 한 번만 질문**:
  - "어떤 주제의 뉴스를 정리할까요? (예: 암호화폐, 반도체, 한국 정치, K-콘텐츠, 기후, 바이오, 부동산 등)"
  - 추가로 선택 가능한 옵션을 한 번에 제시: 지역 범위(글로벌/국내/모두), 수량(기본 15-25개), 기간(기본 최근 7일).

사용자 입력을 정규화:
- `topic_raw`: 사용자가 입력한 원본 주제 문자열 (예: "K-콘텐츠와 OTT")
- `topic_slug`: 파일명용 슬러그 — 한글은 로마자 음역 또는 영문 키워드 매핑, 공백은 하이픈, 소문자 (예: `k-content-ott`, `crypto`, `semiconductor`)
- `topic_display`: HTML 제목용 표기 (예: "K-콘텐츠 & OTT", "암호화폐")

### Step 1: 작업 디렉토리 설정

```
working_dir = ./news-digest-output/{topic_slug}/
filename = news-digest-{topic_slug}-{YYYYMMDD-HHmmss}.html
# 예: news-digest-crypto-20260417-143025.html
```

작업 디렉토리를 생성하고 모든 파일을 여기에 저장. HTML 파일명에 주제 슬러그와 생성 시점의 datetime을 포함.

### Step 2: Planner 서브에이전트 디스패치

```
Agent({
  description: "News Digest Planner",
  prompt: [references/planner-prompt.md 내용]
          + "\n\n사용자 요청: {user_input}"
          + "\n작업 디렉토리: {working_dir}"
          + "\n주제(원본): {topic_raw}"
          + "\n주제(표기): {topic_display}"
          + "\n주제(슬러그): {topic_slug}"
          + "\n오늘 날짜: {today}"
          + "\n지역 범위: {global|korea|both}"
          + "\n목표 수량: {N}"
          + "\n기간: 최근 {days}일"
          + "\n추가 제약: {사용자가 지정한 내용이 있으면 전달, 없으면 '없음'}",
  mode: "auto"
})
```

- Planner는 주제에 적절한 **카테고리 4-8개를 스스로 도출** (예: 암호화폐 → [시장/시세, 규제, 프로젝트, 보안, 기업/투자, 기술, 한국])
- WebSearch/WebFetch로 실제 최신 뉴스를 검색하고 수집
- `spec.md` 작성 완료까지 대기
- **사용자에게 spec.md 내용(특히 도출된 카테고리와 뉴스 목록)을 보여주고 확인 받기** (필수 게이트)

### Step 3: Generator 서브에이전트 디스패치

```
Agent({
  description: "News Digest HTML Generator",
  prompt: [references/generator-prompt.md 내용]
          + "\n\n작업 디렉토리: {working_dir}"
          + "\n주제(표기): {topic_display}"
          + "\n주제(슬러그): {topic_slug}",
  mode: "auto"
})
```

- Generator는 `spec.md`를 읽고 `sprint_contract.md` 작성
- 승인 후 `news-digest-{topic_slug}-{datetime}.html` 생성
- `generator_report.md` 작성

### Step 4: Evaluator 서브에이전트 디스패치

```
Agent({
  description: "News Digest Evaluator",
  prompt: [references/evaluator-prompt.md 내용]
          + "\n\n작업 디렉토리: {working_dir}"
          + "\n주제(표기): {topic_display}",
  mode: "auto"
})
```

- Evaluator는 `spec.md`, `sprint_contract.md`, `generator_report.md`, 생성된 HTML을 읽고 검증
- **주제 정합성**을 필수로 확인 (주제와 무관한 기사 혼입 금지)
- `critique.md`에 루브릭 점수와 판정 작성

### Step 5: 반복 또는 완료

```
if critique.md verdict == "FAIL":
    # spec.md + critique.md만 전달하여 새 Generator 디스패치
    # critique.md의 blocking issues를 반드시 해결하도록 지시
    # Generator에게 전략적 판단 권한 부여:
    #   - 점수가 개선 추세이면 현재 방향을 개선 (refine)
    #   - 점수가 정체/하락이면 접근 방식 자체를 피봇 (pivot)
    → Step 3으로 (최대 3회 반복)

if critique.md verdict == "PASS":
    # 완료! 사용자에게 결과 보고
    # news-digest-{topic_slug}-{datetime}.html을 브라우저에서 열기 제안

if iteration_cap(3) 도달 && still FAIL:
    # 사용자에게 현재 상태 보고 및 수동 개입 요청
```

**비선형 품질 향상 주의**: 원문 경험에 따르면, 마지막 이터레이션이 항상 최선은 아니다. 중간 이터레이션이 더 나은 경우가 있으므로, 각 이터레이션의 HTML을 `news-digest-{topic_slug}-{datetime}-v{N}.html`로 보존하고, Evaluator 점수가 가장 높은 버전을 최종 산출물로 선택한다.

### Step 6: 결과 전달

- 완성된 HTML 경로를 사용자에게 안내
- `open {working_dir}/news-digest-{topic_slug}-{datetime}.html`로 브라우저에서 확인 제안

## 역할 프롬프트 (Role Prompts)

- **Planner**: See [references/planner-prompt.md](references/planner-prompt.md)
- **Generator**: See [references/generator-prompt.md](references/generator-prompt.md)
- **Evaluator**: See [references/evaluator-prompt.md](references/evaluator-prompt.md)

## 평가 루브릭 (Evaluation Rubric)

See [references/rubric.md](references/rubric.md)

| Criterion | Weight | Rationale |
|-----------|--------|-----------|
| Topic Relevance | **2x** | 주제 정합성이 이 스킬의 1순위 — 주제와 무관한 기사 혼입은 즉시 FAIL |
| Content Completeness | 1x | Binary — Claude handles this well |
| Summary Quality | **2x** | Korean NLP quality varies; needs extra pressure |
| Visual Design Quality | **2x** | Claude defaults to generic HTML; design needs forcing |
| Technical Correctness | 1x | Claude reliably produces valid HTML |
| Interactivity | 1x | Concrete functional checks |
| Accessibility & Polish | 1x | Important but not primary differentiator |

**통과 기준**: 모든 항목 >= 4, 2x 가중 항목(주제 정합성, 요약 품질, 디자인 품질) 셋 다 >= 4.

## Evaluator 운영 지침

### Evaluator 튜닝은 반복적 과정이다

원문 경험: "Out of the box, Claude is a poor QA agent. In early runs, I watched it identify legitimate issues, then talk itself into deciding they weren't a big deal and approve the work anyway."

Evaluator가 관대해지는 패턴을 발견하면:
1. Evaluator 로그(critique.md)를 직접 읽고, 판단이 자신의 기대와 어디서 갈라지는지 식별
2. Evaluator 프롬프트의 해당 부분을 구체화 (예: "hover effect가 없으면 무조건 Design 3점 이하")
3. 다시 실행하여 개선 확인
4. 이 루프를 여러 라운드 반복 — 한 번에 완벽한 Evaluator 프롬프트는 없다

### Evaluator 경계(Boundary) 개념

원문 핵심: "The evaluator is not a fixed yes-or-no decision. It is worth the cost when the task sits beyond what the current model does reliably solo."

- Opus 4.6에서 경계가 바깥으로 이동하여, **단순한 작업에는 Evaluator가 불필요**할 수 있다
- 이 스킬의 도메인(단일 HTML 뉴스 페이지)은 Opus 4.6 기준으로 경계 근처에 있다
- **Evaluator를 생략할 수 있는 조건**: 뉴스 아이템이 5개 이하이고 사용자가 빠른 결과를 원할 때
- **Evaluator가 반드시 필요한 조건**: 주제 정합성 검증이 중요하거나(좁은 니치 주제), 디자인 품질이 중요하거나, 뉴스 아이템이 10개 이상이거나, 한국어 요약의 정확성이 중요할 때

## 파일 핸드오프 계약 (File Handoff Contract)

```
Files (single source of truth):
  spec.md                                             Planner → Generator, Evaluator
  sprint_contract.md                                  Generator ↔ Evaluator
  generator_report.md                                 Generator → Evaluator
  critique.md                                         Evaluator → Generator
  handoff.md                                          outgoing → fresh session (context reset)
  news-digest-{topic_slug}-{YYYYMMDD-HHmmss}.html    Generator output (최종 산출물)
  news-digest-{topic_slug}-{datetime}-v{N}.html       이터레이션별 버전 보존 (비선형 품질 대응)
```

## Simplified vs Full Harness 가이드

이 스킬은 **Simplified Harness**를 기본으로 사용:
- 산출물이 단일 HTML 페이지 (단일 아티팩트)
- Opus 4.6 기준 한 세션에서 충분히 완성 가능
- 스프린트 분할 불필요 — Generator가 한 번에 전체 구현

**Full Harness로 전환이 필요한 경우**:
- 뉴스 아이템이 30개 이상으로 복잡해질 때
- 다국어 버전을 동시에 만들어야 할 때
- 인터랙티브 차트/시각화(가격 추이, 투표율 그래프 등)가 추가될 때
- 주제가 여러 하위 주제로 분할되어 각각의 디자인이 다른 경우

## Red Flags — 이런 증상이 보이면 STOP

| 증상 | 원인 | 해결 |
|------|------|------|
| 주제와 무관한 기사가 섞임 | Planner가 너무 넓게 검색 | Planner에게 주제 범위를 엄격하게 정의하도록 지시; Evaluator Topic Relevance 2x 가중 |
| Generator가 "잘 만들었습니다!"로 끝냄 | 자기평가 편향 | Evaluator를 반드시 실행; Generator 프롬프트에서 자화자찬 금지 |
| 뉴스가 오래되었거나 가짜 | Planner가 검색을 건너뜀 | Planner에게 반드시 WebSearch 사용 강제 |
| HTML이 깨져서 열림 | Generator가 서둘러 마무리 | 컨텍스트 리셋 (handoff.md) 후 새 세션 |
| Evaluator가 모두 5점 | 관대한 기본 LLM 평가 | "기본값은 회의적" 프롬프트 강화 |
| 같은 이슈가 반복 | Generator가 critique.md를 제대로 안 읽음 | critique.md의 blocking issues를 명시적으로 인용하도록 강제 |
| 한국어가 번역투 | 영어 소스 직역 | 루브릭에서 Summary Quality 2x 가중 + 구체적 한국어 품질 체크 |
| 디자인이 밋밋함 | CSS 최소화 경향 | 루브릭에서 Visual Design 2x 가중 + 구체적 디자인 체크리스트 |
| 카테고리가 주제에 안 맞음 | Planner의 도메인 이해 부족 | Planner 프롬프트에 카테고리 도출 예시 강화; 사용자에게 카테고리 확인 게이트 |

## 사용 예시

```
사용자: "암호화폐 뉴스 정리해줘"
  → topic_raw = "암호화폐", topic_slug = "crypto", topic_display = "암호화폐"
  → 카테고리 자동 도출: [시장/시세, 규제, 프로젝트, 보안/해킹, 기업/투자, 기술, 한국]

사용자: "반도체 업계 최근 소식 모아줘"
  → topic_raw = "반도체", topic_slug = "semiconductor", topic_display = "반도체"
  → 카테고리 자동 도출: [제조/공정, 설계/IP, 장비/소재, 시장/수요, M&A/투자, 정책/규제, 한국]

사용자: "K-콘텐츠 뉴스 페이지 만들어줘"
  → topic_raw = "K-콘텐츠", topic_slug = "k-content", topic_display = "K-콘텐츠"
  → 카테고리 자동 도출: [드라마, 영화, K-POP, 예능/OTT, 해외 진출, 산업/정책, 인물]

사용자: "기후변화 뉴스 다이제스트"
  → topic_raw = "기후변화", topic_slug = "climate", topic_display = "기후변화"
  → 카테고리 자동 도출: [과학/관측, 정책/COP, 재생에너지, 탄소시장, 기후재난, 기업 ESG, 한국]
```

## Changelog

### v1.0.0 (2026-04-17)
- 초기 릴리스: ai-news-digest-harness v1.1.0을 주제 불문으로 일반화
- 사용자가 입력한 주제에서 카테고리 자동 도출
- 주제 슬러그 기반 파일명 네이밍
- 루브릭에 **Topic Relevance (2x 가중)** 추가 — 주제 정합성을 1순위 품질 축으로 강제
- Planner/Generator/Evaluator 프롬프트를 주제 불문 템플릿으로 리라이트
- 지역 범위·수량·기간 옵션 런타임 지정 지원
