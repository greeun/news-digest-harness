# News Digest Harness (Topic-Agnostic)

> **v1.0.0** · Planner → Generator → Evaluator harness for any-topic Korean news digests

A Claude Code skill that takes a **user-provided topic** (e.g. crypto, semiconductors, Korean politics, climate, K-content, real estate, Tesla, biotech), collects the latest real-world news from the web, summarizes them in natural Korean, and delivers a polished, self-contained HTML page — powered by the **Planner → Generator → Evaluator** harness pattern from Anthropic's [Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps) (Prithvi Rajasekaran, 2026).

This is a generalized sibling of [ai-news-digest-harness](https://github.com/greeun/ai-news-digest-harness): the topic is not hard-coded to AI. You bring the subject.

## Why this skill exists

Naive "summarize some news" prompts fail in three reliable ways:

1. **Topic drift** — the model pulls in tangentially related stories because they're popular.
2. **Machine-translation-smell Korean** — sentences that parse but read like a textbook exercise.
3. **Generic AI-looking HTML** — the same purple-gradient template every single time.

The harness attacks all three structurally, not with more instructions to a single agent.

## How It Works

```
User: "암호화폐 뉴스 정리해줘"
        │
        ▼
┌──────────────────────┐   spec.md   ┌──────────────────────┐
│       Planner        │ ──────────▶ │      Generator        │
│  Topic → categories  │             │   Topic-aware HTML    │
│  + WebSearch         │             │   (single file)       │
└──────────────────────┘             └──────────┬────────────┘
        │                                       │
  User confirms                      generator_report.md
  spec.md                                       │
                                                ▼
                                     ┌──────────────────────┐
                                     │      Evaluator        │
                                     │  Topic Relevance 2x   │
                                     │  + adversarial QA     │
                                     └──────────┬────────────┘
                                                │
                                      critique.md (PASS/FAIL)
                                                │
                        ┌───────────────────────┴───────────────────────┐
                        │                                               │
                      PASS                                            FAIL
                        │                                               │
                        ▼                                               ▼
             news-digest-{topic}-                              Fresh Generator
             {datetime}.html                                   with critique.md
                                                               (up to 3 rounds)
```

### Three Agents, Structurally Separated

| Agent | Role | Input | Output |
|-------|------|-------|--------|
| **Planner** | Normalize topic, derive 4–8 sub-categories, research real news via WebSearch/WebFetch | User prompt + today's date + topic + scope | `spec.md` |
| **Generator** | Build a self-contained HTML page whose mood reflects the topic | `spec.md` (+ `critique.md` on retry) | `news-digest-{topic_slug}-{datetime}.html` |
| **Evaluator** | Grade against a 7-criterion rubric with evidence, with **Topic Relevance weighted 2×** | `spec.md` + HTML + `generator_report.md` | `critique.md` |

Agents never see each other's reasoning. They communicate **exclusively through files**.

## Harness Principles Applied

1. **Structural role separation** — Generation and evaluation are different subagent processes (GAN-inspired)
2. **File-based handoffs** — `spec.md`, `sprint_contract.md`, `generator_report.md`, `critique.md`
3. **Sprint contracts before implementation** — Generator proposes scope + observable checks before coding
4. **Context resets over compaction** — Compaction preserves continuity but not context anxiety; resets eliminate both
5. **Rubric-based evaluation with evidence** — "It looks fine" is never a pass
6. **Every component encodes an assumption** — Remove one at a time on model upgrade; radical simplification fails

## What's different from the AI-only sibling

| | `ai-news-digest-harness` | **`news-digest-harness` (this skill)** |
|---|---|---|
| Topic | Hard-coded to AI | **User-provided, any domain** |
| Categories | Fixed 11 (LLM, Robotics, …) | **Planner derives 4–8 per topic** |
| Filename | `ai-news-digest-{datetime}.html` | `news-digest-{topic_slug}-{datetime}.html` |
| Working dir | `./ai-news-digest-output/` | `./news-digest-output/{topic_slug}/` |
| Rubric criteria | 6 | **7 — adds Topic Relevance (2× weighted)** |
| Instant-FAIL conditions | — | **Yes**: OOS item · `<title>`/`<h1>` leftover "AI" · AI categories in filter · missing spec item · broken standalone render |
| Design mood | Fixed | **Topic-derived** (crypto → techy, K-content → editorial, climate → grounded) |
| Adversarial leftover probe | — | **Searches HTML for AI-template leftovers** |

## Evaluation Rubric

| # | Criterion | Weight | Why this weight |
|---|-----------|--------|-----------------|
| 1 | **Topic Relevance** | **2×** | #1 concern — drift or AI-leftovers make the digest useless |
| 2 | Content Completeness | 1× | Binary, Claude handles well |
| 3 | **Summary Quality** | **2×** | Korean phrasing quality varies most per topic |
| 4 | **Visual Design Quality** | **2×** | Claude recycles templates; needs pressure to fit the topic |
| 5 | Technical Correctness | 1× | Claude reliably produces valid HTML |
| 6 | Interactivity | 1× | Concrete functional checks |
| 7 | Accessibility & Polish | 1× | Important but not the differentiator |

**Pass threshold**: all criteria ≥ 4/5, all three 2×-weighted criteria ≥ 4/5.

**Instant-FAIL** conditions short-circuit the rubric (e.g., any item violates the `spec.md` OUT-OF-SCOPE list).

## Example Topics and Auto-Derived Categories

| User says | `topic_slug` | Auto-derived categories |
|-----------|--------------|-------------------------|
| 암호화폐 뉴스 정리해줘 | `crypto` | 시장/시세, 규제, 프로젝트, 보안/해킹, 기업/투자, 기술, 한국 |
| 반도체 업계 최근 소식 | `semiconductor` | 제조/공정, 설계/IP, 장비/소재, 시장/수요, M&A/투자, 정책/규제, 한국 |
| K-콘텐츠 뉴스 페이지 | `k-content` | 드라마, 영화, K-POP, 예능/OTT, 해외 진출, 산업/정책, 인물 |
| 기후변화 뉴스 다이제스트 | `climate` | 과학/관측, 정책/COP, 재생에너지, 탄소시장, 기후재난, 기업 ESG, 한국 |
| 부동산 최근 뉴스 | `real-estate` | 거래/시세, 정책/규제, 공급/분양, 금융/대출, 지역 동향, 해외, 인물/기업 |

## Output

A single self-contained HTML file (`news-digest-{topic_slug}-{YYYYMMDD-HHmmss}.html`):

- Opens directly in a browser (`file://` protocol)
- 15–25 news items by default (user-adjustable)
- Korean summaries with English originals in parentheses on first mention
- Category filter dynamically populated from `spec.md`
- Dark mode toggle, hover effects, responsive layout
- Embedded CSS/JS only (Google Fonts / jsdelivr Pretendard permitted)

## Installation

Copy the `news-digest-harness/` folder into your Claude Code skills directory:

```
~/.claude/skills/news-digest-harness/
```

Or clone directly:

```bash
git clone https://github.com/greeun/news-digest-harness.git ~/.claude/skills/news-digest-harness
```

## Usage

Trigger the skill with any of these phrases, **including your topic**:

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

If you omit the topic, the skill asks you once. You can also specify:

- **Region scope**: global / Korea / both (default: both, 60–70% global / 30–40% Korea)
- **Item count**: default 15–25
- **Period**: default last 7 days

## File Structure

```
news-digest-harness/
├── SKILL.md                        # Orchestration logic + principles
├── README.md                       # This file (English)
├── README.ko.md                    # Korean documentation
└── references/
    ├── planner-prompt.md           # Topic research + spec writing + auto-categorize
    ├── generator-prompt.md         # HTML generation with topic fidelity rules
    ├── evaluator-prompt.md         # Rubric-based QA + Instant-FAIL conditions
    └── rubric.md                   # 7 criteria, scoring guide, calibration notes
```

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI
- Claude Opus 4.6 (recommended) or Sonnet 4.5+
- Internet access for WebSearch/WebFetch (news collection)

## Credits

Based on Anthropic's [Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps) by Prithvi Rajasekaran (March 2026). Generalized from the original [`ai-news-digest-harness`](https://github.com/greeun/ai-news-digest-harness) template.

## License

MIT
