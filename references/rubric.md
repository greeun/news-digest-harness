# Evaluation Rubric — Topic-Agnostic News Digest

## Criteria

| # | Criterion | What it measures | Weight | Why this weight |
|---|-----------|-----------------|--------|-----------------|
| 1 | **Topic Relevance** | Topic fidelity, scope boundary respected, zero drift, zero AI-leftovers (title/h1/filter/strings) when topic is not AI | **2×** | This is the new #1 concern for a topic-agnostic skill — violation makes the digest useless regardless of how pretty it is |
| 2 | **Content Completeness** | All news items from spec present, correct data, no placeholders | 1× | Binary check — Claude is good at completeness |
| 3 | **Summary Quality** | Korean clarity, readability, natural phrasing, proper use of technical/industry terms with originals, informativeness | **2×** | Claude tends toward stilted Korean and mechanical summaries — this is where quality varies most per topic |
| 4 | **Visual Design Quality** | Modern, clean, topic-appropriate mood; clear typography hierarchy; color fit-for-topic; adequate whitespace; professional feel; distinct from any generic AI template | **2×** | Claude defaults to generic, unstyled HTML or recycles the last template it saw — design quality needs extra pressure |
| 5 | **Technical Correctness** | Valid HTML5, working embedded CSS/JS, responsive viewport, self-contained (works on file://), no broken dependencies | 1× | Claude reliably produces valid HTML |
| 6 | **Interactivity** | Dark mode toggle works, category filter functions with spec.md categories, hover effects present, smooth CSS transitions | 1× | Concrete functional checks — pass/fail |
| 7 | **Accessibility & Polish** | Semantic HTML (article, section, heading hierarchy), link targets (_blank + noopener noreferrer), aria attributes, footer, meta tags referencing topic | 1× | Important but not the primary differentiator |

## Scoring Guide

### Score 5 — Exceptional
- Beyond spec requirements
- Would impress a specialist editor in this topic AND a senior front-end developer
- Zero issues found
- Design choices visibly reflect the topic's character

### Score 4 — Good
- Meets all spec requirements
- Minor polish opportunities but nothing blocking
- Professional quality

### Score 3 — Adequate
- Meets most requirements but has noticeable gaps
- Functional but not polished
- Needs one more pass

### Score 2 — Below Standard
- Missing significant spec items or has topic drift
- Noticeable bugs or visual issues
- Needs substantial rework

### Score 1 — Unacceptable
- Placeholder content, broken rendering, missing features, OR major topic drift
- Requires complete redo

## Passing Threshold

- **All criteria must score ≥ 4**
- **2×-weighted criteria (Topic Relevance, Summary Quality, Visual Design Quality) must all score ≥ 4**
- If any criterion scores < 4: **FAIL**

## Instant-FAIL Conditions

Regardless of other scores, these immediately fail the digest:
- Any news item violates spec.md's OUT-OF-SCOPE list.
- `<title>`, `<h1>`, or meta description still says "AI" when the topic is not AI.
- Category filter contains categories that are not in spec.md (leftover AI-news categories count).
- Any spec.md item is missing or has placeholder content.
- Page does not render standalone (file:// protocol).

## BLOCKING Gate — 감각적 한계 (점수와 무관, 영구)

PASS를 내기 전 반드시 충족한다. 미충족 시 Verdict = **BLOCKED**(점수가 아무리 높아도):

- 렌더 스크린샷(desktop/mobile)이 존재하고, Evaluator가 그것을 읽어 Visual Design 채점에 반영했다.
- 사람의 비주얼 사인오프(`visual_signoff.md`)가 기록되었다.
- 인터랙션 중 콘솔 에러 0(`console.log`).

이 게이트는 LLM이 코드만으로 레이아웃 균형·색 대비·여백 리듬·폰트 느낌을 *볼* 수 없는 **감각적 한계**를 보상한다. 다른 모든 하네스 컴포넌트와 달리 모델 업그레이드로 제거되지 않는다(비타협 원칙 8).

## Calibration Notes

- **Topic Relevance**: Watch for these drift patterns:
  - News items that mention the topic only tangentially (e.g., "Tesla stock rose on AI hype" in an AI digest for a "부동산" topic)
  - Category labels from prior templates that were not updated (e.g., "LLM" or "Robotics" in a crypto digest)
  - Hero copy that generically says "today's tech news" instead of "today's {topic_display} news"
  - Filename missing the topic slug

- **Summary Quality**: Watch for these Korean-specific issues:
  - Unnatural sentence endings (e.g., "~하는 것이다" repeated)
  - Over-literal translations from English sources
  - Missing context that makes the summary unclear to a general reader
  - Technical / industry terms without English originals on first use
  - 낚시성 어휘 ("역대급", "충격", "무려")

- **Visual Design Quality**: Watch for these common AI-HTML pitfalls:
  - No visual hierarchy (everything same size/weight)
  - Default browser styling leaking through
  - Colors that clash or lack contrast
  - No hover states or transitions (feels static/dead)
  - Cards that are just bordered boxes with no depth
  - Mobile layout that's just "everything stacked" with no thought
  - **Palette mismatch with topic** (e.g., 기후 digest in hot red, K-POP digest in muted gray) — score ≤3 on design.
  - Purple/blue AI-style gradient used for every non-AI topic.
  - **슬롭 테스트**: "주제 단어를 다 지워도 다른 아무 주제에나 그대로 얹을 수 있는 디자인인가?" Yes면 generic-anywhere 슬롭 → ≤3. generator-prompt.md "Anti-Slop 대조"의 어느 쌍(1~6)에 걸렸는지 명시.
  - **비주얼 디테일 실패**(스크린샷에서): 카드/태그/배지 오버랩, `overflow:hidden` 클립, 긴 헤드라인 레이아웃 깨짐, 모바일(~390px) 가로 스크롤 — 발견 시 ≤3.

- **Late-stage creative leaps**: If the Generator produces a genuinely innovative design approach on a later iteration (e.g., timeline layout, data viz for prices/rankings), evaluate it on merit against the rubric — don't penalize for departing from a conventional card layout if the result scores higher on design quality AND still respects topic relevance.
