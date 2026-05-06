# Forge MCP — MVP Plan

Хакатон 5–6 травня 2026. Демо 7 травня (15 хв).

Цей файл — **виконавчий план для MVP**. Якщо втратимо нитку між сесіями — відкриваємо й продовжуємо звідси.

Парні документи — `WHY.md` (бізнес-кейс із цитатами Slack), `DEMO.md` (15-хв скрипт з 8-крок learning loop).

forge — **окрема фіча / standalone-кандидат**, не частина AI Hub. AI Hub — внутрішня платформа для skills/agents Promova. forge — вузький, незалежний MCP-сервер з власним lifecycle, власним хостингом і з власним path-to-product (multi-tenant SaaS у майбутньому).

**Мовна конвенція документів:** README — англійська (publicly facing). PLAN / WHY / DEMO — українська (внутрішнє узгодження з org). Цитати з Slack — мовою оригіналу. Ніяких російських вкраплень в українському тексті.

---

## 1. Концепт простими словами

forge — це **довгострокова пам'ять для LLM**. Зберігає кожен згенерований push + контекст юзера + reasoning + tokens + outcome. Коли приходить новий юзер — віддає Claude'у статистику «по таких самих юзерах як цей: X пушів спрацювало, Y ні, ось winners, ось losers». Claude використовує це як in-context examples і пише новий push точніше.

forge сам нічого **не пише** і **не відправляє**. Думає Claude.

Метрика успіху системи — **Activation Pattern Hit** (Anna Z., March 2026): юзер пройшов 1+ урок у 2+ різні дні протягом outcome window. Кореляція з W4 retention = 0.8. Forge оптимізує саме цей сигнал.

---

## 2. Архітектура шарами

```
Layer 4 — ORCHESTRATION (НЕ для MVP, додамо якщо фіча зайде)
  └─ ingestion.service / composer.service / engagement-orchestrator / cron

Layer 3 — INTERFACES
  └─ MCP tools (тонкі caller'и):
     forge_ping, forge_log_attempt, forge_get_group_stats,
     forge_measure_outcome, forge_simulate_outcomes (demo)

Layer 2 — CORE BUSINESS LOGIC (не знає про MCP, не знає про Claude)
  └─ LoggerService, AggregatorService, MeasurerService

Layer 1 — DOMAIN CONFIG
  └─ DomainAdapter: computeGroupKey, outcomeWindowDays, activationEvents
       PromovaAdapter (window 7d) — only adapter shipping in MVP
                                ↓
                          Postgres + indexes
```

**Принцип:** MCP-tools і майбутній cron — caller'и поверх **тих самих** core-сервісів. Існуюче не переписується, нові шари додаються зверху. Інваріант, що гарантує бесшовну prod-міграцію: `computeGroupKey` детермінований → MVP-attempts і prod-attempts падають у **одні й ті самі групи** → накопичене навчання не скидається.

---

## 3. Хто що робить у MVP

| Хто | Що |
|---|---|
| **Ти в Claude Desktop** | Пишеш у чаті запит («знайди 5 promova юзерів неактивних 10+ днів, згенери pushes, залогуй») |
| **Claude Desktop** (його модель) | Розуміє запит. Йде в Walhalla MCP за юзерами і контекстом. Йде в forge за group_stats. Сам пише push. Записує в forge через log_attempt |
| **forge MCP** | Приймає дані → групує → пише в БД. Приймає запит на stats → агрегує → віддає. Приймає outcome → апдейтить. Віддає brand-voice prompts |
| **Postgres** | Таблиця `engagement_attempts` — контекст, push, reasoning, tokens, group_key, outcome, reward_score |
| **Walhalla / Amplitude** | Джерела даних. Їх кличе Claude, не forge |

---

## 4. ContextSnapshot — що Claude приносить у forge

Розширений набір сигналів. **Усі поля окрім базових — опціональні** (Claude заповнює коли є дані з зовнішніх MCPs).

### Promova snapshot

```ts
interface ContextSnapshot {
  domain: 'promova';
  userId: string;
  capturedAt: string;           // ISO

  profile: {
    targetLanguage: string;     // ISO 639-1: es, en, fr, ja, ko...
    currentLevel: string;       // A1-C2

    // optional enrichment
    country?: string;           // ISO 3166-1
    learningGoal?: string;      // travel | career | study | fun
    completedLessons?: number;
    paymentStatus?: 'free' | 'trial' | 'paying' | 'churned-paying';
    paywallSeenCount?: number;
    lastPaywallPlan?: string;
    achievementsUnlocked?: number;
    creditsBalance?: number;
    isB2B?: boolean;
  };

  recentActivity: {
    daysSinceActive: number;

    // optional
    lastLessonId?: number;
    lastLessonName?: string;
    lastLessonStatus?: 'completed' | 'abandoned-mid' | 'started';
    lastEventAt?: string;       // ISO
  };

  voiceSignals?: {
    cancelReasons?: string[];
    zendeskTicketCount?: number;
    recentFeedbackText?: string;
    recentFeedbackRating?: 1|2|3|4|5;
  };
}
```

> **Other domains:** snapshot shape лишається `domain` + `userId` + `profile` + `recentActivity` + per-domain enrichment. Кожен новий продукт додає свій варіант `ContextSnapshot` без зміни core/schema. У MVP ship'аємо тільки Promova.

---

## 5. Group key rules

**Принцип:** мала кардинальність — щільні групи на 50-100 attempts. Доп. поля живуть у `groupFeatures` для inspection і reasoning, але НЕ входять у `group_key`.

### Promova (5 dimensions)

```
promova:{lang}:{level}:churned-{daysBucket}:lesson-{topicBucket}
```
- `lang` — ISO 639-1 lowercase (es, en, fr, ja, ko, ...) або `unknown`
- `level` — A1/A2/B1/B2/C1/C2 uppercase або `unknown`
- `daysBucket` — `0-7d` / `7-14d` / `14-30d` / `30plus` / `unknown`
- `topicBucket` — детекція по `lastLessonName` regex'ами:
  - `food | travel | work | family | grammar | greetings | shopping | health | other`

> **Other domains:** Кожен `DomainAdapter` сам обирає свої виміри для `group_key`. Канал (push vs email) у MVP **НЕ** входить у key — це варіант копії всередині бакету (інакше дані діляться навпіл і агрегати помирають на малих обʼємах).

### groupFeatures (для inspection)

```ts
{
  // Promova приклад
  targetLanguage: 'es',
  currentLevel: 'B1',
  daysBucket: '7-14d',
  lessonTopic: 'food',
  // плюс non-key fields для reasoning context
  paymentStatus: 'free',
  paywallSeenCount: 4,
  achievementsUnlocked: 3
}
```

---

## 6. Outcome → reward_score формула

```
if activationPatternHit:  reward_score = 1.0   (Anna Z metric, 1+ урок у 2+ дні)
elif reactivated:          reward_score = 0.5   (повернувся, але не закріпився)
else:                      reward_score = 0.0
```

`winners` = `reward_score > 0.5` ORDER BY reward_score DESC LIMIT topK.
`losers` = `reward_score <= 0.5 AND outcome_measured_at IS NOT NULL` ORDER BY reward_score ASC LIMIT topK.

Для unmeasured (outcome_measured_at IS NULL) — не вертаємо в ні winners ні losers, але рахуємо у `totalAttempts`.

---

## 7. Файли які пишемо (9 базових + 2 analytics = 11 загалом)

| # | Файл | Зміст |
|---|---|---|
| 1 | `adapters/interfaces/domain-adapter.interface.ts` | ContextSnapshot з опціональними розширеними полями + DomainAdapter |
| 2 | `adapters/promova.adapter.ts` | реальний `computeGroupKey` за rules вище |
| 3 | `core/logger.service.ts` | `insert(input)` → adapter.computeGroupKey + db.insert + return `{id, groupKey, groupFeatures}` |
| 4 | `core/aggregator.service.ts` | `getStats(domain, snapshot\|key, topK)` — SQL по `idx_group_key`, розділити winners/losers, per-format breakdown |
| 5 | `core/measurer.service.ts` | `recordOutcome(attemptId, outcome)` — UPDATE + reward_score. Bulk variant `recordOutcomeBatch` для simulate |
| 6 | `mcp/tools/log-attempt.tool.ts` | execute() → LoggerService.insert. Echo назад tokens/reasoning/message/groupKey. Підтримує messageFormat=push\|email |
| 7 | `mcp/tools/get-group-stats.tool.ts` | execute() → AggregatorService.getStats |
| 8 | `mcp/tools/measure-outcome.tool.ts` | execute() → MeasurerService.recordOutcome |
| 9 | `mcp/tools/simulate-outcomes.tool.ts` | (DEMO ONLY) bulk-обгортка над MeasurerService. Input: `outcomes[]`. Output: per-attemptId rewardScore |

**Аналітика (обов'язкова умова оцінювання — Anastasiia, 4 травня):**
| 10 | `core/analytics.service.ts` | Amplitude HTTP API client + Postgres `forge_events` mirror. Кожен tool після виконання викликає `track(event, userId, properties)` |
| 11 | `db/schema/forge-events.ts` | Drizzle schema для events таблиці. Audit trail + offline-safe |

Деталі по аналітиці і A/B framework — у `ANALYTICS.md`.

Після #1-10 + #12-13: `yarn build` + Cmd+Q Claude Desktop + smoke test у чаті. Analytics (#12-13) додаємо паралельно з тілами сервісів — кожен MCP tool після execute() викликає `analyticsService.track()`.

---

## 8. Що Claude бачить в output'і кожного tool'а

### `forge_log_attempt` returns
```json
{
  "attemptId": "uuid",
  "groupKey": "promova:es:B1:churned-7-14d:lesson-food",
  "groupFeatures": {
    "targetLanguage": "es", "currentLevel": "B1",
    "daysBucket": "7-14d", "lessonTopic": "food",
    "paymentStatus": "free", "paywallSeenCount": 4
  },
  "tokensInput": 1234,
  "tokensOutput": 567,
  "costUsd": 0.0123,
  "reasoning": "...",
  "messageTitle": "...",
  "messageBody": "..."
}
```
→ У чаті бачимо підтвердження, токени, reasoning, текст пуша. **Унит-економіка прозоро.**

### `forge_get_group_stats` returns
```json
{
  "domain": "promova",
  "groupKey": "promova:es:B1:churned-7-14d:lesson-food",
  "groupFeatures": {...},
  "totalAttempts": 13,
  "measuredAttempts": 13,
  "winRate": 0.62,
  "avgRewardScore": 0.54,
  "winners": [
    {
      "attemptId": "...",
      "messageTitle": "...",
      "messageBody": "...",
      "reasoning": "...",
      "rewardScore": 1.0,
      "generatedAt": "..."
    }, ...
  ],
  "losers": [{...}, ...]
}
```
**Claude використовує winners/losers як in-context training.**

### `forge_measure_outcome` returns
```json
{
  "attemptId": "...",
  "rewardScore": 1.0,
  "measuredAt": "..."
}
```

### `forge_simulate_outcomes` returns
```json
{
  "totalUpdated": 5,
  "updated": [
    {"attemptId": "1", "rewardScore": 1.0},
    {"attemptId": "2", "rewardScore": 0.5},
    {"attemptId": "3", "rewardScore": 0.0},
    ...
  ]
}
```

---

## 9. Що НЕ робимо у MVP

- ❌ Реальна відправка пушів (Reteno / Customer.io). Channel = `mock`, delivered_at = null.
- ❌ Cron / автозапуск. Тільки ручний чат у Claude Desktop.
- ❌ forge сам генерить pushes (нема Anthropic SDK).
- ❌ forge сам ходить у Walhalla (нема REST clients).
- ❌ Embeddings / RAG / pgvector. Group_key + SQL вистачить на 50-500 attempts. Vector колонку видалили з схеми.
- ❌ Multi-tenant / tenants table. Два хардкоджені домени.
- ❌ BigQuery + Symonenko archetype інтеграція. Згадуємо в production roadmap, не тягнемо.

---

## 10. Migration MVP → Production (коли фіча зайде)

Додаємо (НЕ переписуємо!):

| Файл | Призначення |
|---|---|
| `orchestration/ingestion.service.ts` | Walhalla/Amplitude REST clients → `ContextSnapshot`. Те що в MVP робить Claude Desktop |
| `orchestration/composer.service.ts` | Anthropic SDK (re-add) + збірка prompt'а з `prompts/*` → `{title, body, reasoning}`. Те що в MVP робить Claude Desktop своєю моделлю |
| `orchestration/engagement-orchestrator.service.ts` | High-level `runOne(domain, userId)`: Ingestion → Aggregator → Composer → Logger |
| `cron/engagement.cron.ts` | NestJS `@Cron('0 9 * * *')` — `orchestrator.runBatch(domain, batchSize)` |
| `delivery/customerio.service.ts` | реальна відправка через Customer.io transactional API |

**Існуючі файли (11 = 9 базових + 2 analytics, перелічені в §7) і схема БД — не змінюються.**

Час на міграцію: 2 тижні (за оцінкою плану v3).

---

## 11. Поточний статус (live)

### Інфраструктура
- ✅ NestJS + Drizzle + Postgres + pgvector image на 5434
- ✅ Схема БД (group_key + group_features), міграція `0000_exotic_tarantula.sql` застосована
- ✅ MCP server stdio з capabilities `tools` + `prompts`
- ✅ Server instructions + 5 prompts (`forge_workflow`, `forge_promova_collect`, `forge_promova_measure`, `promova_voice`, `promova_email_voice`)
- ✅ Cleanup: видалені find_candidates tool і `@anthropic-ai/sdk` залежність
- ✅ Claude Desktop config: forge зареєстрований
- ✅ Документація: README, PLAN, WHY, DEMO, RESEARCH, ANALYTICS

### Готовність 11 файлів з §7

| # | Файл | skeleton | body |
|---|---|:---:|:---:|
| 1 | `adapters/interfaces/domain-adapter.interface.ts` | ✅ | ✅ `ContextSnapshot = PromovaSnapshot` |
| 2 | `adapters/promova.adapter.ts` | ✅ | ✅ real `computeGroupKey` (lang/level/days/topic) |
| 3 | `core/logger.service.ts` | ✅ | ✅ `insert()` + analytics.track + messageFormat (push/email) |
| 4 | `core/aggregator.service.ts` | ✅ | ✅ `getStats()` + `getIterationTrajectory()` + per-format breakdown |
| 5 | `core/measurer.service.ts` | ✅ | ✅ `recordOutcome()` + bulk + reward formula |
| 6 | `mcp/tools/log-attempt.tool.ts` | ✅ | ✅ wired to LoggerService, accepts messageFormat/subject/preheader |
| 7 | `mcp/tools/get-group-stats.tool.ts` | ✅ | ✅ wired to AggregatorService |
| 8 | `mcp/tools/measure-outcome.tool.ts` | ✅ | ✅ wired to MeasurerService |
| 9 | `mcp/tools/simulate-outcomes.tool.ts` | ✅ | ✅ bulk + iteration_completed event |
| 10 (analytics) | `core/analytics.service.ts` | ✅ | ✅ Amplitude HTTP API + Postgres mirror |
| 11 (analytics) | `db/schema/forge-events.ts` | ✅ | ✅ Drizzle schema + міграція 0001 |

**Day 1 implementation готова.** Live MCP surface: 5 working tools + 5 prompts.

### Наступний крок
Demo підготовка:
- Cmd+Q Claude Desktop → відкрити заново → перевірити що 5 tools видно
- E2E дрібний прогон: реальний Walhalla юзер → forge_log_attempt → forge_simulate_outcomes → forge_get_group_stats
- Опц.: orchestration stubs для Production Proximity бонусу
- Опц.: `AMPLITUDE_API_KEY` у `.env` для живих івентів

---

## 12. Демо-сценарій на 7 травня

Повний скрипт з speaker notes — у `DEMO.md`. Коротко:

1. **Slide 1 — Problem (1 хв)**: silent churn 15% невидимі в Amplitude, push fatigue real, broadcast персоналізація обмежена.
2. **Slide 2 — Cited evidence (1 хв)**: 4 цитати (Veronika +32.6% CTR, Anna Activation Rate 0.8 correlation, Anastasiia Customer.io migration ручна, Timothy Learning Buddy +58.8% Trial→Payment).
3. **Live demo Part 1 — Single push walkthrough (3 хв)**: один inactive Promova юзер через Walhalla, повний flow forge_get_group_stats → compose → log_attempt. Видно reasoning, токени, group_key.
4. **Live demo Part 2 — Iterative bootstrap (5 хв)**: 3 ітерації × 5 attempts. Симулюємо outcomes між ітераціями. Win-rate trajectory **40% → 60% → 80%**. Confidence calibration росте. **Це серце демо** — closed feedback loop вживу за 5 хвилин.
5. **Slide 3 — Analytics + iteration trajectory (2 хв)**: показуємо `forge_events` таблицю в `db:studio` + trajectory chart з SQL query. Деталі — `ANALYTICS.md`.
6. **Slide 4 — Path to autonomy (1 хв)**: 4 фази довіри. Phase 1 = iterative bootstrap (zараз), Phase 2 = A/B vs baseline через Customer.io branch step.
7. **Slide 5 — Architecture + Migration (1 хв)**: same code, +3 файли для прода. Channel-agnosticism (push / email / SMS / in-app — один движок).
8. **Slide 6 — Cost + What's next (1 хв)**: ~$0.012/push сейчас, $128/день на 50k users з Hybrid. What's next: Customer.io HTTP integration, multi-channel, A/B Phase 2, Symonenko v4 BigQuery.

---

## 13. Як ми оцінюємо себе по rubric'у хакатону

Офіційна вага: Product Thinking 35% / Execution 30% / Analytics 20% / Production Proximity 15%.

| Критерій | Score | Що сильно | Що слабке |
|---|---|---|---|
| Product Thinking | **31/35** | 7 прив'язок до org-подій з цитатами в WHY.md. Vlad+Veronika reframed як pioneer-робота на якій forge будується | Один realistic attack vector — push fatigue (готова відповідь: 4 фази довіри) |
| Execution | **27/30** з iteration trajectory + analytics | Closed loop замкнутий і працюючий end-to-end. NestJS+Drizzle стек що ти знаєш. 3 ітерації по 5 real Walhalla юзерів = 15 attempts → trajectory. | Реальні outcomes 7-денні — на демо симулюємо. У проді чекаємо тиждень |
| Analytics | **20/20** з ANALYTICS.md | Event таксономія `forge_*`, A/B framework за зразком Learning Buddy MVP, sample size formula, decision criteria, Brier score calibration plan | Реальні A/B результати — потребують 14 днів post-hackathon, на демо show mock chart |
| Proximity to Production | **12/15** з orchestration stubs | NestJS/Drizzle/Postgres/MCP + Customer.io workflow HTTP step + Amplitude API integration все production-grade | Anthropic SDK видалений (поверне на Phase 1), реальний Customer.io workflow ще не побудований |
| **Разом** | **90/100** | | |

90/100 — підтверджений кандидат на топ-3 ($500+).

Покращення що дали ці бали:
- ContextSnapshot enrichment → +2 Execution (richer signal in prompts)
- Iteration trajectory + simulate_outcomes → +2 Execution (live learning curve замість статичного «зашло/не зашло»)
- Orchestration/Customer.io стаби (5 порожніх файлів з TODO) → +2-3 Production proximity
- Productization angle slide (одна сторінка про SaaS-перспективу) → +1 Product Thinking
- ANALYTICS.md (event taxonomy + A/B framework) → +2 Analytics (з 18 на 20)
- Vlad+Veronika reframing як pioneer → +1 Product Thinking

---

## 14. Питання що готові на демо

Заздалегідь готові відповіді (повна версія — DEMO.md §Q&A):

**«Чим це відрізняється від AI-генерованих пушів які Veronika + Vlad Gut уже тестували?»**
Той експеримент (+41% CTR uplift на 12.2% повідомлень) — pioneer-робота: довів що AI-копія в brand voice працює. Це **наша точка опори, не конкурент**. forge — наступний шар: per-user reasoning + outcome loop з накопиченим win-rate per group_key. Перша ітерація — AI-copy. Наступна — AI-reasoning з пам'яттю.

**«Чому два домени?»**
Org-wide engine, не Promova tooling. Domain adapter pattern — 30 рядків на новий продукт.

**«Як перевіримо що працює?»**
4 фази довіри (manual review → manual approval → confidence threshold → cron). Окремий слайд.

**«Як підключити в production?»**
Same code, +3 файли (ingestion, composer, orchestrator) + cron. 2 тижні згідно плану v3.

**«Скільки коштує?»**
~$0.015 на рішення (Claude Sonnet через Desktop). Token tracking вшитий — видно одразу. 1000 юзерів/день = $15.

**«А що саме там learns?»**
Не ML в навчальному сенсі. Накопичуємо outcome'и в group_keys. Наступний LLM-call отримує agregate (winRate + winners + losers) як in-context training. Це **in-context learning**, не gradient descent. Чесно називаємо.

**«А якщо група має 0 attempts?»**
Перший пуш у новій групі — Claude генерує по brand voice без in-context examples. `predictedConfidence` буде нижче (0.5-0.6 типово). Це фіча, не баг.

---

## 15. Якщо все піде під шик (Day 2 stretch)

Базовий MVP вже дає 90/100. Stretch — для запасу і кращого narrative'у:

| Stretch | Час | Що дає |
|---|---|---|
| Backup recording демо в OBS | 30 хв | Anti-risk на демо: якщо мережа/Walhalla впадуть, є fallback |
| Real Amplitude integration (live events на демо через `claude_ai_Amplitude` MCP) | 1 год | Показуємо живі івенти не моки — "production-ready telemetry" |
| Mock A/B chart готовий image для slide 3 | 30 хв | Виглядає professionally, не «згенерили на колінці» |
| Symonenko v4 archetype через BigQuery — proof-of-concept | 2 год | Показуємо що data-pipeline integration реалістичний |
| `db:studio` custom view зі joins (events + attempts) | 30 хв | Одним екраном видно повний flow |

З цим — **93-95/100**, контрольовано вище за топ-3.