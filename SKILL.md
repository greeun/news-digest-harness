---
name: news-digest-harness
version: 1.1.0
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
7. **게이트 주체는 오케스트레이터다**: 모든 사용자 확인 게이트(주제 확정, spec 확인, 비주얼 사인오프)는 이 SKILL.md를 실행하는 **메인 스레드(오케스트레이터)**가 수행한다. Planner/Generator/Evaluator 서브에이전트는 사용자와 직접 대화할 수 없으며 **오직 파일만 입출력**한다. 오케스트레이터가 서브에이전트의 산출 파일을 읽어 사용자에게 제시하고, 사용자 응답을 받아 다음 서브에이전트에 파일로 전달한다. "서브에이전트가 사용자에게 물어본다"는 흐름은 불가능 — 게이트는 항상 오케스트레이터를 경유한다.
8. **감각적 한계 게이트는 영구 필수**: 렌더 + 스크린샷 + **사람의 비주얼 사인오프**(Step 5-6)는 원칙 6의 **예외**다. LLM은 소스코드만으로 레이아웃 균형·색 대비·여백 리듬·폰트 느낌을 **볼 수 없다**(sensory limit). 이는 컨텍스트/계획 한계가 아니라 감각 한계이므로 모델 업그레이드로 사라지지 않는다. "Evaluator가 알아서 처리한다"는 불가 — Evaluator는 스크린샷을 읽어 보조할 뿐, 사람의 시각 사인오프를 대체하지 못한다.

## 실행 흐름 (Activation)

사용자가 주제를 포함한 뉴스 다이제스트 요청을 보내면, **오케스트레이터(이 SKILL.md를 실행하는 메인 스레드)**가 아래 단계를 진행한다. 각 단계의 `[수행: …]` 라벨은 그 단계를 실제로 실행하는 주체를 가리킨다. **모든 사용자 게이트(★)는 오케스트레이터가 수행** — 서브에이전트는 파일만 입출력하며 사용자와 직접 대화하지 않는다(비타협 원칙 7).

> 사용자 확인 게이트 3개: ★게이트 1 = 주제 확정(Step 0), ★게이트 2 = spec 확인(Step 2), ★게이트 3 = 비주얼 사인오프(Step 5). 어느 것도 조용히 건너뛰지 않는다.

### Step 0: 주제 확정 (Topic Clarification) — ★게이트 1 `[수행: 오케스트레이터 ↔ 사용자]`

사용자 요청에서 주제를 추출한다.

- 주제가 명확한 경우(예: "암호화폐 뉴스 정리해줘", "반도체 뉴스 모아줘") → 그대로 사용.
- 주제가 모호한 경우(예: "뉴스 다이제스트 만들어줘", "뉴스 정리해줘") → **사용자에게 한 번만 질문**:
  - "어떤 주제의 뉴스를 정리할까요? (예: 암호화폐, 반도체, 한국 정치, K-콘텐츠, 기후, 바이오, 부동산 등)"
  - 추가로 선택 가능한 옵션을 한 번에 제시: 지역 범위(글로벌/국내/모두), 수량(기본 15-25개), 기간(기본 최근 7일).

사용자 입력을 정규화:
- `topic_raw`: 사용자가 입력한 원본 주제 문자열 (예: "K-콘텐츠와 OTT")
- `topic_slug`: 파일명용 슬러그 — 한글은 로마자 음역 또는 영문 키워드 매핑, 공백은 하이픈, 소문자 (예: `k-content-ott`, `crypto`, `semiconductor`)
- `topic_display`: HTML 제목용 표기 (예: "K-콘텐츠 & OTT", "암호화폐")

### Step 1: 작업 디렉토리 설정 `[수행: 오케스트레이터]`

```
working_dir = ./news-digest-output/{topic_slug}/
filename = news-digest-{topic_slug}-{YYYYMMDD-HHmmss}.html
# 예: news-digest-crypto-20260417-143025.html
```

작업 디렉토리를 생성하고 모든 파일을 여기에 저장. HTML 파일명에 주제 슬러그와 생성 시점의 datetime을 포함.

### Step 2: Planner 서브에이전트 디스패치 + ★게이트 2 `[디스패치: 오케스트레이터 → 수행: Planner 서브에이전트]`

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
- Planner가 `spec.md` 작성 완료(`SPEC_READY` 출력)할 때까지 오케스트레이터가 대기
- **★게이트 2 (필수)**: **오케스트레이터가 `spec.md`를 직접 읽어** 도출된 카테고리와 뉴스 목록을 사용자에게 제시하고 확인받는다. Planner는 사용자와 대화하지 않으며, 오케스트레이터가 중개한다.
- 사용자가 수정을 요청하면: 오케스트레이터가 피드백을 정리해 Planner를 재디스패치하거나 `spec.md`를 직접 수정한 뒤 다시 확인받는다.

### Step 3: Generator 서브에이전트 디스패치 `[디스패치: 오케스트레이터 → 수행: Generator 서브에이전트]`

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

- **(Simplified harness 기본) sprint_contract 생략**: 산출물이 단일 HTML이므로 Generator는 `spec.md`를 읽고 곧바로 생성한다.
- **(Full harness로 전환 시에만) sprint_contract 사용**: Generator가 `sprint_contract.md`를 작성하면 **오케스트레이터가 검토·승인**한다. Evaluator는 별도 세션이라 실시간 협상이 불가하므로, 승인 주체는 항상 오케스트레이터다.
- Generator는 `news-digest-{topic_slug}-{datetime}.html` 생성 + `generator_report.md` 작성(`READY_FOR_QA` 출력)까지 진행하고, 오케스트레이터가 대기한다.
- Generator는 **anti-slop 디자인 규칙**과 **비주얼 디테일 규칙**(카드/태그/배지의 오버랩·클립·잘림 금지)을 준수한다 — [references/generator-prompt.md](references/generator-prompt.md) 참조.

### Step 4: 렌더 + 스크린샷 `[수행: 오케스트레이터, agent-browser 스킬]`

Generator 산출 직후, 오케스트레이터가 생성된 HTML을 **실제 브라우저에서 렌더**하고 증거를 캡처한다. 상세 절차는 [references/screenshot-checkpoint.md](references/screenshot-checkpoint.md).

- **agent-browser 스킬**(내장 브라우저 도구보다 우선)로 `file://` 경로의 HTML을 연다 — 빌드 스텝 없이 열려야 함
- **2개 이상 뷰포트 폭에서 풀페이지(스크롤 포함) 스크린샷**: 모바일(~390px) + 데스크탑(~1280px+) — 모든 뉴스 카드가 증거에 담기게
- 인터랙션 실행(다크모드 토글, 카테고리 필터) + **콘솔 로그 캡처 — 목표 에러 0**
- `screenshot-desktop.png`, `screenshot-mobile.png`, `console.log`를 `working_dir`에 저장(Evaluator가 경로를 인용)
- agent-browser를 쓸 수 없는 환경이면 **게이트를 조용히 건너뛰지 말 것** — 사용자에게 브라우저 자동화 불가를 알리고 직접 열어 확인하도록 요청한다(Step 5의 사람 사인오프는 그대로 유지). 렌더 증거가 전혀 없는 산출물은 PASS 불가.

### Step 5: Human Visual Checkpoint — ★게이트 3 `[수행: 오케스트레이터 ↔ 사용자]`

LLM이 코드만으로는 판단할 수 없는 채널(레이아웃 균형·색 대비·여백 리듬·폰트 느낌)을 **사람이 직접 확인**한다(비타협 원칙 8). 이 게이트는 모델이 강해져도 제거하지 않는다.

- 오케스트레이터가 데스크탑/모바일 스크린샷을 대화에 인라인(PNG를 Read)하거나 `open {file}`로 사용자 브라우저에서 연다
- 집중 질문 예: "렌더 결과입니다(데스크탑/모바일). 레이아웃 균형·색 대비·여백·폰트 느낌이 괜찮나요? 어색하거나 템플릿처럼 보이는 곳이 있나요?"
- 사인오프(승인자 + 메모)를 기록해 Evaluator가 critique에서 인용할 수 있게 한다. 구체성 없는 "괜찮아 보임"도 사인오프로 인정하되 그 사실을 메모한다.
- **사용자가 시각 문제를 지적하면 PASS 금지** — 문제를 다음 Generator 라운드의 blocking issue로 넣고 Step 3 → 4 → 5를 재실행한다.

### Step 6: Evaluator 서브에이전트 디스패치 `[디스패치: 오케스트레이터 → 수행: Evaluator 서브에이전트]`

```
Agent({
  description: "News Digest Evaluator",
  prompt: [references/evaluator-prompt.md 내용]
          + "\n\n작업 디렉토리: {working_dir}"
          + "\n주제(표기): {topic_display}"
          + "\n스크린샷 경로: {screenshot-desktop.png, screenshot-mobile.png}"
          + "\n콘솔 로그: {console.log}"
          + "\n비주얼 사인오프: {Step 5에서 기록한 승인자/메모}",
  mode: "auto"
})
```

- Evaluator는 `spec.md`, `generator_report.md`, 생성된 HTML, 그리고 **Step 4의 스크린샷/콘솔 로그**, **Step 5의 비주얼 사인오프 기록**을 읽고 검증
- **주제 정합성**을 필수로 확인 (주제와 무관한 기사 혼입 금지)
- **감각적 한계 게이트 확인**: 스크린샷이 존재하고 Evaluator가 읽었으며, 사람 사인오프가 기록되었는지 확인 — 셋 중 하나라도 없으면 PASS 불가
- `critique.md`에 루브릭 점수와 판정 작성

### Step 7: 반복 또는 완료

```
if critique.md verdict == "FAIL":
    # 오케스트레이터가 spec.md + critique.md만 전달하여 새 Generator 디스패치
    # critique.md의 blocking issues를 반드시 해결하도록 지시
    # Step 5에서 사용자가 지적한 시각 문제도 blocking issue로 포함
    # Generator에게 전략적 판단 권한 부여:
    #   - 점수가 개선 추세이면 현재 방향을 개선 (refine)
    #   - 점수가 정체/하락이면 접근 방식 자체를 피봇 (pivot)
    → Step 3 → Step 4(렌더) → Step 5(비주얼 사인오프) → Step 6(Evaluator) 재실행 (최대 3회 반복)

if critique.md verdict == "PASS":
    # 완료! 오케스트레이터가 사용자에게 결과 보고
    # news-digest-{topic_slug}-{datetime}.html을 브라우저에서 열기 제안

if iteration_cap(3) 도달 && still FAIL:
    # 폴백: 보존된 v1..v3 중 Evaluator 종합 점수가 가장 높은 버전을 최종 산출물로 선택
    # 오케스트레이터가 사용자에게 "PASS 미달, 최고점 버전 N을 잠정 제출" + 남은 blocking issue 목록 보고
    # 사용자에게 수동 개입(추가 라운드 / 직접 수정 / 현 상태 수용) 선택권 제시
```

**비선형 품질 향상 주의**: 원문 경험에 따르면, 마지막 이터레이션이 항상 최선은 아니다. 중간 이터레이션이 더 나은 경우가 있으므로, 각 이터레이션의 HTML을 `news-digest-{topic_slug}-{datetime}-v{N}.html`로 보존하고, **PASS 여부와 무관하게** Evaluator 종합 점수가 가장 높은 버전을 최종 산출물로 선택한다(3회 FAIL 폴백도 동일 기준).

### Step 8: 결과 전달 `[수행: 오케스트레이터]`

- 완성된 HTML 경로를 사용자에게 안내
- `open {working_dir}/news-digest-{topic_slug}-{datetime}.html`로 브라우저에서 확인 제안

## 역할 프롬프트 (Role Prompts)

- **Planner**: See [references/planner-prompt.md](references/planner-prompt.md)
- **Generator**: See [references/generator-prompt.md](references/generator-prompt.md)
- **Evaluator**: See [references/evaluator-prompt.md](references/evaluator-prompt.md)
- **비주얼 체크포인트 (Step 4-5, 렌더+스크린샷+사람 사인오프)**: See [references/screenshot-checkpoint.md](references/screenshot-checkpoint.md)

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

- Opus 4.8에서 경계가 더 바깥으로 이동하여, **단순한 작업에는 Evaluator가 불필요**할 수 있다
- 이 스킬의 도메인(단일 HTML 뉴스 페이지)은 Opus 4.8 기준으로 경계 근처에 있다
- **Evaluator를 생략할 수 있는 조건**: 뉴스 아이템이 5개 이하이고 사용자가 빠른 결과를 원할 때
- **Evaluator가 반드시 필요한 조건**: 주제 정합성 검증이 중요하거나(좁은 니치 주제), 디자인 품질이 중요하거나, 뉴스 아이템이 10개 이상이거나, 한국어 요약의 정확성이 중요할 때

## 파일 핸드오프 계약 (File Handoff Contract)

```
Files (single source of truth):
  spec.md                                             Planner → [오케스트레이터 ★게이트 2] → Generator, Evaluator
  sprint_contract.md                                  (Full harness 시) Generator → 오케스트레이터 승인
  generator_report.md                                 Generator → Evaluator
  screenshot-desktop.png / screenshot-mobile.png      오케스트레이터(Step 4 렌더) → Evaluator
  console.log                                         오케스트레이터(Step 4) → Evaluator (에러 0 확인용)
  visual_signoff.md                                   오케스트레이터(Step 5 ★게이트 3, 사용자 사인오프 기록) → Evaluator
  critique.md                                         Evaluator → 오케스트레이터 → Generator
  handoff.md                                          outgoing → fresh session (context reset)
  news-digest-{topic_slug}-{YYYYMMDD-HHmmss}.html    Generator output (최종 산출물)
  news-digest-{topic_slug}-{datetime}-v{N}.html       이터레이션별 버전 보존 (비선형 품질 대응)
```

**게이트는 항상 오케스트레이터를 경유한다**: 서브에이전트 간 통신은 파일로 직결되지 않는다. spec.md → Generator 전에 ★게이트 2(사용자 spec 확인), HTML → Evaluator 전에 Step 4-5(렌더 + ★게이트 3 비주얼 사인오프)를 오케스트레이터가 끼워 넣는다.

## Simplified vs Full Harness 가이드

이 스킬은 **Simplified Harness**를 기본으로 사용:
- 산출물이 단일 HTML 페이지 (단일 아티팩트)
- Opus 4.8 기준 한 세션에서 충분히 완성 가능
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
| 렌더 안 보고 PASS | 감각-한계 게이트 누락 | Step 4-5(렌더+스크린샷+사람 사인오프) 필수; 미충족 시 Evaluator는 BLOCKED |
| 모든 주제가 보라 그라디언트 | AI 슬롭 기본값 | Anti-Slop 대조(주제 단어 제거 테스트); 일치 시 Visual Design ≤3 |
| 카드/배지가 겹쳐 안 읽힘 | 비주얼 디테일 무시 | 비주얼 디테일 규칙(오버랩/클립/잘림); 스크린샷에서 확인 |
| 서브에이전트가 사용자에게 질문 | 게이트 주체 혼동 | 게이트는 오케스트레이터만 수행; 서브에이전트는 파일만 입출력(원칙 7) |

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

### v1.1.0 (2026-06-29)
- **게이트 주체 명확화**: 모든 사용자 게이트(주제 확정·spec 확인·비주얼 사인오프)는 오케스트레이터가 수행, 서브에이전트는 파일만 입출력 (비타협 원칙 7). 실행 흐름 각 단계에 `[수행: …]` 라벨 부여.
- **감각적 한계 게이트 추가** (비타협 원칙 8): 렌더 + 스크린샷 + 사람 비주얼 사인오프(Step 4-5). 미충족 시 Evaluator는 `BLOCKED` 판정. 모델 업그레이드로 제거 불가한 영구 게이트.
- **html-explainer-harness 비주얼 요소 이식**: Anti-Slop 대조 갤러리(슬롭 vs 비슬롭 6쌍), 비주얼 디테일 규칙(카드/태그 오버랩·클립·잘림 금지), `references/screenshot-checkpoint.md` 신설.
- **3회 FAIL 폴백 명시**: PASS 미달 시 보존된 v1..v3 중 최고점 버전 선택 + 사용자 수동 개입 선택권.
- **sprint_contract 정리**: Simplified harness(기본)에선 생략, Full harness 전환 시에만 오케스트레이터 승인.
- 모델 표기 4.6 → 4.8 갱신.

### v1.0.0 (2026-04-17)
- 초기 릴리스: ai-news-digest-harness v1.1.0을 주제 불문으로 일반화
- 사용자가 입력한 주제에서 카테고리 자동 도출
- 주제 슬러그 기반 파일명 네이밍
- 루브릭에 **Topic Relevance (2x 가중)** 추가 — 주제 정합성을 1순위 품질 축으로 강제
- Planner/Generator/Evaluator 프롬프트를 주제 불문 템플릿으로 리라이트
- 지역 범위·수량·기간 옵션 런타임 지정 지원
