# 주제 불문 뉴스 다이제스트 하네스

> **v1.0.0** · Planner → Generator → Evaluator 하네스로 임의 주제의 한국어 뉴스 페이지를 생성

**사용자가 입력한 임의의 주제**(예: 암호화폐, 반도체, 한국 정치, 기후, K-콘텐츠, 부동산, 테슬라, 바이오)에 대해 웹에서 실제 최신 뉴스를 수집하고, 자연스러운 한국어로 요약하여, 정갈한 디자인의 단일 HTML 페이지로 제공하는 Claude Code 스킬입니다. Anthropic의 [Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps) (Prithvi Rajasekaran, 2026)에서 제시한 **Planner → Generator → Evaluator** 하네스 패턴을 적용합니다.

[`ai-news-digest-harness`](https://github.com/greeun/ai-news-digest-harness)를 주제 불문으로 일반화한 자매 스킬입니다. 주제를 고정하지 않고, 사용자가 가져옵니다.

## 이 스킬이 필요한 이유

"뉴스 좀 요약해줘" 같은 순진한 프롬프트는 세 가지 방식으로 망합니다:

1. **주제 드리프트** — 모델이 화제성 이유로 주제에서 벗어난 기사를 끌어옵니다.
2. **기계번역투 한국어** — 문법은 맞지만 교과서 예문처럼 읽히는 문장.
3. **제네릭 AI HTML** — 매번 똑같은 보라색 그라데이션 템플릿.

이 하네스는 세 문제를 **단일 에이전트에게 지시를 더 하는 방식이 아니라**, 구조적으로 공격합니다.

## 작동 방식

```
사용자: "암호화폐 뉴스 정리해줘"
        │
        ▼
┌──────────────────────┐   spec.md   ┌──────────────────────┐
│       Planner        │ ──────────▶ │      Generator        │
│  주제 → 카테고리      │             │   주제 인식 HTML      │
│  + 웹 검색            │             │   (단일 파일)         │
└──────────────────────┘             └──────────┬────────────┘
        │                                       │
  사용자가 spec                       generator_report.md
  확인 후 진행                                   │
                                                ▼
                                     ┌──────────────────────┐
                                     │      Evaluator        │
                                     │  주제 정합성 2x       │
                                     │  + 적대적 QA          │
                                     └──────────┬────────────┘
                                                │
                                      critique.md (PASS/FAIL)
                                                │
                        ┌───────────────────────┴───────────────────────┐
                        │                                               │
                      PASS                                            FAIL
                        │                                               │
                        ▼                                               ▼
             news-digest-{topic}-                              새 Generator에
             {datetime}.html                                   critique.md 전달
                                                               (최대 3회 반복)
```

### 세 에이전트의 구조적 분리

| 에이전트 | 역할 | 입력 | 출력 |
|---------|------|------|------|
| **Planner** | 주제 정규화, 4–8개 하위 카테고리 도출, WebSearch/WebFetch로 실제 뉴스 리서치 | 사용자 요청 + 오늘 날짜 + 주제 + 범위 | `spec.md` |
| **Generator** | 주제 분위기에 맞는 self-contained HTML 제작 | `spec.md` (+ 재시도 시 `critique.md`) | `news-digest-{topic_slug}-{datetime}.html` |
| **Evaluator** | 7개 기준 루브릭 + 증거 기반 채점 (**주제 정합성 2× 가중**) | `spec.md` + HTML + `generator_report.md` | `critique.md` |

에이전트는 서로의 추론을 볼 수 없습니다. **오직 파일을 통해서만** 소통합니다.

## 적용된 하네스 원칙

원문 아티클의 6가지 핵심 원칙을 인코딩합니다:

1. **구조적 역할 분리** — 생성과 평가를 별도 서브에이전트 프로세스로 (GAN 영감)
2. **파일 기반 핸드오프** — `spec.md`, `sprint_contract.md`, `generator_report.md`, `critique.md`
3. **구현 전 스프린트 계약** — Generator가 코딩 전에 범위와 검증 항목을 합의
4. **압축 대신 컨텍스트 리셋** — 압축은 연속성은 유지하지만 불안은 해소 못 함; 리셋은 둘 다 제거
5. **증거 기반 루브릭 평가** — "괜찮아 보인다"는 절대 통과가 아님
6. **모든 컴포넌트는 가정을 인코딩** — 모델 업그레이드 시 한 번에 하나씩 제거; 급진적 단순화는 실패

## AI 전용 자매 스킬과의 차이

| | `ai-news-digest-harness` | **`news-digest-harness` (이 스킬)** |
|---|---|---|
| 주제 | AI로 고정 | **사용자가 입력하는 임의 주제** |
| 카테고리 | 고정 11개 (LLM, Robotics, …) | **Planner가 주제별 4–8개 도출** |
| 파일명 | `ai-news-digest-{datetime}.html` | `news-digest-{topic_slug}-{datetime}.html` |
| 작업 디렉토리 | `./ai-news-digest-output/` | `./news-digest-output/{topic_slug}/` |
| 루브릭 기준 | 6개 | **7개 — 주제 정합성 (2× 가중) 추가** |
| Instant-FAIL 조건 | — | **있음**: OOS 기사 · `<title>`/`<h1>`에 "AI" 잔존 · AI 카테고리 잔존 · spec 기사 누락 · 독립 렌더 실패 |
| 디자인 무드 | 고정 | **주제 유도** (암호화폐→테키, K-콘텐츠→에디토리얼, 기후→그라운디드) |
| 템플릿 잔존 적대적 탐지 | — | **HTML 내 AI 템플릿 잔존 문자열 검색** |

## 평가 루브릭

| # | 기준 | 가중치 | 근거 |
|---|------|--------|------|
| 1 | **주제 정합성** | **2×** | 1순위 관심사 — 드리프트나 AI 잔존은 다이제스트를 무용지물로 만듦 |
| 2 | 콘텐츠 완성도 | 1× | Binary, Claude가 잘 처리 |
| 3 | **요약 품질** | **2×** | 한국어 문장 품질 편차가 주제별로 가장 큼 |
| 4 | **시각 디자인 품질** | **2×** | Claude가 템플릿을 재활용; 주제 적합성까지 압박 필요 |
| 5 | 기술적 정확성 | 1× | Claude가 안정적으로 유효한 HTML 생성 |
| 6 | 인터랙티비티 | 1× | 구체적 기능 검증 항목 |
| 7 | 접근성 & 완성도 | 1× | 중요하지만 주요 차별화는 아님 |

**통과 기준**: 모든 항목 4/5 이상, 2× 가중 항목 3개(주제 정합성, 요약 품질, 디자인 품질) 모두 4/5 이상.

**Instant-FAIL** 조건은 루브릭을 우회하여 즉시 실패 처리합니다 (예: `spec.md`의 OUT-OF-SCOPE 리스트 위반 기사가 하나라도 있으면).

## 주제 예시와 자동 도출 카테고리

| 사용자 입력 | `topic_slug` | 자동 도출 카테고리 |
|------------|--------------|---------------------|
| 암호화폐 뉴스 정리해줘 | `crypto` | 시장/시세, 규제, 프로젝트, 보안/해킹, 기업/투자, 기술, 한국 |
| 반도체 업계 최근 소식 | `semiconductor` | 제조/공정, 설계/IP, 장비/소재, 시장/수요, M&A/투자, 정책/규제, 한국 |
| K-콘텐츠 뉴스 페이지 | `k-content` | 드라마, 영화, K-POP, 예능/OTT, 해외 진출, 산업/정책, 인물 |
| 기후변화 뉴스 다이제스트 | `climate` | 과학/관측, 정책/COP, 재생에너지, 탄소시장, 기후재난, 기업 ESG, 한국 |
| 부동산 최근 뉴스 | `real-estate` | 거래/시세, 정책/규제, 공급/분양, 금융/대출, 지역 동향, 해외, 인물/기업 |

## 산출물

단일 self-contained HTML 파일 (`news-digest-{topic_slug}-{YYYYMMDD-HHmmss}.html`):

- 브라우저에서 직접 열기 가능 (`file://` 프로토콜)
- 기본 15–25개 뉴스 항목 (사용자 조정 가능)
- 한국어 요약, 첫 언급 시 영어 원문 괄호 병기
- `spec.md`에서 동적으로 생성된 카테고리 필터
- 다크 모드 토글, 호버 효과, 반응형 레이아웃
- 임베디드 CSS/JS만 사용 (Google Fonts / jsdelivr Pretendard 허용)

## 설치

`news-digest-harness/` 폴더를 Claude Code 스킬 디렉토리에 복사:

```
~/.claude/skills/news-digest-harness/
```

또는 직접 클론:

```bash
git clone https://github.com/greeun/news-digest-harness.git ~/.claude/skills/news-digest-harness
```

## 사용법

**주제를 포함한** 다음 문구로 스킬을 트리거합니다:

```
암호화폐 뉴스 정리해줘
반도체 뉴스 페이지 만들어줘
K-콘텐츠 뉴스 다이제스트
테슬라 최근 소식 모아줘
기후변화 뉴스 요약 페이지

news digest about crypto
daily news page for semiconductors
latest K-content news page
```

주제를 생략하면 스킬이 한 번 물어봅니다. 추가 옵션:

- **지역 범위**: global / korea / both (기본: both, 해외 60–70% / 국내 30–40%)
- **수량**: 기본 15–25개
- **기간**: 기본 최근 7일

## 파일 구조

```
news-digest-harness/
├── SKILL.md                        # 오케스트레이션 로직 + 원칙
├── README.md                       # 영문 문서
├── README.ko.md                    # 이 파일 (한국어)
└── references/
    ├── planner-prompt.md           # 주제 리서치 + spec 작성 + 카테고리 자동 도출
    ├── generator-prompt.md         # 주제 충실성 규칙을 적용한 HTML 생성
    ├── evaluator-prompt.md         # 루브릭 기반 QA + Instant-FAIL 조건
    └── rubric.md                   # 7개 평가 기준, 점수 가이드, 캘리브레이션
```

## 요구 사항

- [Claude Code](https://claude.ai/claude-code) CLI
- Claude Opus 4.6 (권장) 또는 Sonnet 4.5+
- 인터넷 접속 (뉴스 수집을 위한 WebSearch/WebFetch)

## 크레딧

Anthropic의 [Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps) (Prithvi Rajasekaran, 2026년 3월) 기반. 원본 [`ai-news-digest-harness`](https://github.com/greeun/ai-news-digest-harness) 템플릿을 주제 불문으로 일반화.

## 라이선스

MIT
