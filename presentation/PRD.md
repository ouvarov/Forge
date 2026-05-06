# 

— PRD

> Reasoning engine для re-engagement push/email у Promova.
> Лог → outcome → накопичений досвід → точніший наступний удар.

**Author:** Alexandr Uvarov · **Updated:** 2026-05-06 · **Status:** Pilot-ready

---

## TL;DR

Forge — це **reasoning-шар над outcome'ами re-engagement-комунікацій**.
На вхід — context snapshot юзера. На вихід — `{title, body, reasoning, confidence}`
з прямою цитатою winners з пам'яті по 5-dim когорті.
Output → JSON → будь-який CRM (Reteno / Customer.io). **Один HTTP call upstream
від webhook step.**

Forge — **не** template generator (як Reteno «One from many») і **не** authoring
helper (як Customer.io AI Actions). Це окрема категорія: **per-user reasoning з
пам'яттю по cohort'і**, що оптимізує Activation Pattern (corr 0.8 з W4 retention),
а не CTR.

---

## Проблема

Два факти, які разом окреслюють одну прогалину:

### 1. Нуль learning-loop'ів у re-engagement
Жодна система в org не feedback'ить outcome надісланого push'а назад у
генерацію наступного per-сегмент. Дані є, voice є, AI-генерація вже довела
+41% CTR на одному з експериментів — але **engine з пам'яттю немає**.
Кожен push стартує з нуля. Reteno «One from many» оптимізує CTR-leaderboard
шаблонів. Customer.io AI Actions — authoring helper. Жоден не накопичує
win-rate per behavioral cohort.

### 2. Персоналізація працює, але не компаундить
Внутрішні цифри Promova (не benchmark'и галузі):

| Експеримент | Lift | Контекст |
|---|---|---|
| AI brand voice push | **+41% CTR** | stat-sig на 12% повідомлень |
| Welcome flow персоналізація | **+33% CTR** | broadcast → segmented |
| Learning Buddy MVP | **+58% Trial→Payment** | iOS A/B 10 днів |
| Mini-wins | **+86% start уроку** | post-lesson nudge |
| Activation Rate ↔ W4 retention | **0.8 correlation** | north star |

Це **floor, не stretch.** Кожен експеримент стартував з нуля — досвід нікуди
не записувався. Місце для системи, що накопичує його через хвилі — порожнє.

---

## Гіпотеза та метрика

**Гіпотеза:** win-rate персоналізованого push з cited reasoning'ом > broadcast,
**і win-rate зростає з хвилями**, бо система вчиться на власних outcome'ах.

**Primary metric:** Activation Pattern Hit-rate (1+ урок у 2+ днях після push).
Корреляція 0.8 з W4 retention → оптимізуючи activation, оптимізуємо retention.

**Validation plan:** 5–7 хвиль по 1000+ юзерів кожна.
- Trajectory росте → система вчиться, скейлимо.
- Trajectory плато → **fail-fast за 100–150 спроб** (не за 12 тисяч).

---

## Як це працює (5 кроків)

```
┌─ 1. Context snapshot ──────────────────────────────────┐
│  domain (Promova) збирає payload: lang, level,         │
│  paymentStatus, daysInactive, lastLessonTopic, channel │
└────────────────────────────────────────────────────────┘
                          │
┌─ 2. Group memory ──────────────────────────────────────┐
│  forge_get_group_stats(domain, contextSnapshot)        │
│  → winners[], losers[], density, win-rate per group    │
└────────────────────────────────────────────────────────┘
                          │
┌─ 3. Compose ───────────────────────────────────────────┐
│  Claude (Anthropic SDK) бачить cited examples,         │
│  пише {title, body, reasoning, confidence}             │
│  Brand voice prompts: окремі для push та email         │
└────────────────────────────────────────────────────────┘
                          │
┌─ 4. Log ───────────────────────────────────────────────┐
│  forge_log_attempt → attempt_id                        │
│  зберігає: snapshot + message + reasoning + cost +     │
│            tokens + group_key                          │
└────────────────────────────────────────────────────────┘
                          │
┌─ 5. Measure (T+7 days) ────────────────────────────────┐
│  Activation Pattern hit? → reward_score → пам'ять      │
│  winner стає cited example для наступної хвилі         │
└────────────────────────────────────────────────────────┘
```

---

## group_key (5-dim cohort)

`lang × level × paymentStatus × inactivityBucket × lastLessonTopic`

- **lang** — інтерфейсна мова юзера
- **level** — рівень курсу (A1…C1)
- **paymentStatus** — `free` / `paying` / `churned-paying`
- **inactivityBucket** — `7-14d` / `15-30d` / `31-60d` / `60+d`
- **lastLessonTopic** — остання тема уроку (з keeper[])

**Канал (push / email) — НЕ в ключі.** Окремий voice prompt на кожен.

---

## Differentiation

| | Reteno «One from many» | Customer.io AI | **Forge** |
|---|---|---|---|
| **Що оптимізує** | CTR per кампанія | — (helper) | **Activation Pattern** |
| **Reasoning** | Немає | Draft suggestions | **Cited winners з cohort'и** |
| **Пам'ять** | CTR-leaderboard шаблонів | Send-time | **Win-rate per 5-dim group_key** |
| **Категорія** | Variant generator | Authoring helper | **Reasoning over outcomes** |
| **Output** | Push в Reteno | Push в CIO | **JSON → будь-який CRM** |
| **Apliฟt в Promova** | Не дав апліфту | Не тестовано | **Pilot: 5–7 хвиль** |

Reteno не дав апліфту бо оптимізував не той сигнал (CTR per кампанія, без
behavioral context). Customer.io — це template helpers всередині CRM.
Forge — **HTTP call upstream від webhook step**, працює з обома.

---

## Economics (пілот, консервативний funnel)

| | Value |
|---|---|
| Push'ів | 7,000 |
| CTR | 3% (нижче за broadcast baseline) |
| Кліків | 210 |
| Lesson start (50% з кліку) | 105 |
| Activation Pattern hit (40%) | 42 |
| ARPS-дельта на активацію | $4.40 |
| **Виручка** | **$185** |
| **Витрати (LLM + infra)** | **$28** |
| **ROI** | **6.6x** |

**Cost per push (prod):** ~0.4¢ (HTTP pull + Anthropic SDK).

**Break-even:** 0.1% activation lift (7 активацій з 7к). На два порядки
нижче за floor +13%, який ми вже бʼємо.

**Sensitivity:** на 1% CTR (worst plausible) ROI = **2.2x**.

### Revenue funnel — паралельна ROI-доріжка

Deep link у тілі push/email веде на **Promova app2web checkout** з UTM:

```
https://promova.com/app2web_checkout
  ?productId=<billing_plan_id>
  &utm_source=forge
  &utm_medium=push|email
  &utm_campaign=reengagement
  &utm_content=<plan_id>
  &attempt_id=<id>
```

Це простий checkout (без sales-page'у), яким легко керувати — UTM
приходять як query params, проходять через payment flow і **фіксуються
в Amplitude на Promova-side** як `payment_succeeded` з `utm_source=forge`.
Видно **скільки оплат прийшло від Forge** і per-attempt drill-down через
`attempt_id`.

Per-user offer обирається автоматично за `paymentStatus + completedLessons +
daysSinceActive`:

| Сегмент | Offer |
|---|---|
| `free` + active ≤14d | weekly trial ($9.99/wk, 7d free) |
| `free` + inactive >14d | quarterly trial ($29.99/3mo, 7d free) |
| `churned-paying` + ≥15 lessons | annual 20% off ($55.99/yr) |
| `churned-paying` + <15 lessons | quarterly trial |
| `paying` / `trial` / B2B | no offer (skip) |

**Activation funnel і revenue funnel — два незалежні ROI-треки з одного
push'а.** Перший рахує retention-uplift через Activation Pattern hit.
Другий — прямі підписки через UTM-attribution в Amplitude. Не
взаємовиключні. Це і є відповідь на «як це повертається в підписки».

---

## Production readiness

**2 тижні. 4 файли.**

| File | Change |
|---|---|
| `ingestion` | HTTP pull замість MCP (Walhalla → backend, не Claude Desktop) |
| `composer` | Прямий Anthropic SDK call (не Claude Desktop) |
| `cron + endpoints` | `/compose` (батч генерація), `/measure` (T+7d outcome) |
| `webhook step у CRM` | Один HTTP call із Reteno/CIO journey |

### Path to autonomy (4 фази довіри)

1. **Human-gated send** на перших 1000 юзерів — людина ревʼю кожну партію
2. **A/B vs baseline** через CRM branch
3. **Auto-send за confidence threshold** з guardrails:
   - fatigue cap (max N/тиждень на юзера)
   - hard-skip < 0.5 confidence
   - rate-limit 24h dedup
4. **Cron з SLO на cancel rate**

**Sensitive domains** (mental health, dating) застряють на Phase 2 назавжди.

---

## Analytics surface

Усі метрики живуть у **admin UI** (`localhost:5174`, NestJS backend
`/api/*`). Amplitude для інспекції reasoning chain не підходить —
потрібна форма яку він не дає. Amplitude використовується лише
для revenue-attribution (див. вище) на Promova-side.

### Overview cards
`total_attempts` · `reactivated` · `silent` · `pending` · `win_rate` ·
`total_cost_usd` · `cost_per_reactivation_usd`

Сума cost = `cost_usd + discovery_cost_usd + orchestration_cost_usd` —
чесна одиниця економіки per attempt.

### Attempts list (`GET /api/attempts`)
Filterable by `status` (reactivated / silent / pending) і `format`
(push / email). Сортування: measured first, потім newest.

Per row: `id`, `user_id`, `group_key`, `message_format`, `message_title`,
`predicted_confidence`, `generated_at`, `verdict`, `reward_score`,
total cost USD.

### Proof panel (`GET /api/attempts/:id/proof`)
Per-attempt drill-down — найважливіша поверхня для жюрі:

- **Frozen context snapshot** — group features на момент send'а
- **Message** — title + body (push), + subject + preheader (email)
- **Reasoning chain** — з прямими цитатами winners/losers з cohort'и
- **Predicted hypothesis** + confidence
- **Outcome timeline** —
  `frozen_last_seen_at → sent_at → matures_at → fresh_last_seen_at → measured_at`
- **New keeper keys** — що з'явилось у юзера після push'а
- **Activation Pattern hit** · **Reward score** · **Verdict**
- **Offer plan_id** + deep link (якщо є)
- **Outcome purchased plan_id** (заповнюється на T+7, якщо юзер купив)

### Top groups (`GET /api/groups/top`)
Per-group_key: total attempts, measured attempts, overall win-rate,
**push win-rate**, **email win-rate** (split). Видно яка комбінація
`lang × level × payment × inactive × topic` працює, а яка ні.

### Iteration trajectory
Schema несе `experiment_name` + `iteration_number` — для побудови
trajectory chart по хвилях (validation primary signal).

---

## Найбільший ризик

**Не opt-out spam.** Він структурно неможливий на Phase 1–2: людина в
loop'і перед кожним send'ом.

**Trajectory не росте за 5–7 хвиль.** Якщо win-rate тримається на cold-start —
один з трьох:
- group_key неправильний (мало dimensions / зайві dimensions)
- Activation Pattern — не той outcome
- brand voice не транслює досвід winners у новий текст

**План:** fail-fast за 100–150 спроб, переглянути group_key та outcome metric.

---

## Next steps (validated)

- ✅ Email канал — окремий voice prompt уже працює
- 🔜 Revenue campaigns — персональні згораючі знижки через UTM deep link
- 🔜 Symonenko archetypes (BigQuery) — додатковий dimension у group_key
- 🔜 A/B vs baseline у Phase 2

---

## Stack

NestJS 10 · Drizzle ORM · Postgres 16 (port 5434) · Anthropic SDK ·
@modelcontextprotocol/sdk (stdio) · Zod

```bash
yarn docker:up && yarn db:migrate && yarn start:dev
```

---

## Links

- **Repo:** `forge-mcp-server` (private)
- **Demo recording:** TBD (буде після demo day)
- **Admin UI:** `localhost:5174` (live на демо)
- **Demo script:** `presentation/SCRIPT.md`
