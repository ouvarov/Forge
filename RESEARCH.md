# Forge — як це масштабувати у прод (per-user push для кожного)

Документ-довідка: коли MVP пройшов і фіча зайшла, як технічно дати **кожному окремому юзеру персональний push**? 4 ключових питання з відповідями. Цифри — для слайду «Cost economics» і відповідей жюрі.

---

## 1. Trigger — хто вирішує «час слати push цьому юзеру»?

**Не forge.** Це задача Customer.io (CRM-движка, на який Promova зараз мігрує — Anastasiia Stelmakh, lead).

### Customer.io Workflows — головний механізм

Customer.io має нативний інструмент **Workflows**: визначаєш умови входу (Audience), кроки обробки, splits, delays, send/no-send. Стандартні параметри:
- **Trigger**: подія (`lesson_started`, `paywall_viewed`, `subscription_cancelled`) або segment-membership change
- **Wait**: затримка по часу або до події
- **Branch**: A/B або conditional (за user property)
- **Send**: push / email / in-app
- **HTTP request step** ← **наш ключ**: workflow може покликати external HTTP endpoint у середині процесу

### Інтеграція форджу через HTTP step

```
Customer.io Workflow:
┌────────────────────────────────────────────┐
│ Trigger: user.last_lesson_started > 10d    │
│   ↓                                        │
│ Audience filter: lang=es AND level=B1      │
│   ↓                                        │
│ HTTP step → POST forge.promova.in/compose  │
│             { userId, snapshot? }           │
│   ↓ response.title, response.body          │
│ Send Push: title=$.title, body=$.body      │
│   ↓                                        │
│ Wait 7 days                                │
│   ↓                                        │
│ HTTP step → POST forge.promova.in/measure │
│             { attemptId }                   │
│   ↓                                        │
│ End                                        │
└────────────────────────────────────────────┘
```

**Що це нам дає:**
- Customer.io володіє «коли + кому» (його робота)
- forge володіє «що написати + чи спрацювало» (наша робота)
- Ніяких cron'ів у форджі — Customer.io їх вже має

### Альтернативні тригери

| Тригер | Коли підходить | Складність |
|---|---|---|
| Customer.io Workflow + HTTP step | Основний продакшн-шлях | Низька |
| Cron у forge (NestJS `@Cron`) | Якщо Customer.io workflow не дає flexibility | Середня |
| Event listener (Kafka / Pub/Sub) | Real-time реакція на event-stream | Висока |
| Manual ops dashboard | Ручний запуск маркетологом | Низька |

**Рекомендація:** старт через Customer.io HTTP step, fallback cron додаємо тільки якщо workflow не покриває edge cases.

---

## 2. Data enrichment — звідки беремо контекст для **кожного** юзера?

У MVP контекст збирає Claude Desktop руками через MCPs. У проді контекст приходить з **трьох рівнів**:

### Level 1 — Symonenko v4 (вже існує, 1M юзерів)

Ivan Kotiuchyi і Yelyzaveta Dorozhko вже згенерували **rich text-описи** для всіх юзерів:

```
"English learner from Spanish language, at A2 elementary level. Location:
Mexico. Prefers 5 minutes daily sessions. Comes from organic source.
Specifically: 2 sessions, 4 lessons, 75% completion rate. Engagement
score: 6. Risk profile: no apparent risk."
```

Плюс **25 структурованих features** на юзера (engagement_tier, monetization_stage, churn_risk_flag, archetype, etc.).

**Інтеграція:**
- forge'ова `IngestionService` робить запит до Symonenko BigQuery таблиці за `userId`
- Pass-through → `contextSnapshot.profile` без перетворення тексту
- BigQuery cost: ~$5 на TB sканування. На 1 юзера ~few KB → $0.000005

### Level 2 — Real-time event stream (Walhalla / Amplitude REST)

Symonenko оновлюється раз на добу. Для свіжих сигналів (хто зараз onboarding'у, хто щойно бачив paywall) — REST виклики:

| Endpoint | Що дає | Latency |
|---|---|---|
| `Walhalla.user_get(userId)` | поточний профіль, billing, credits | ~150ms |
| `Walhalla.amplitude_analytics(user_activity)` | останні 50 events | ~300ms |
| `Walhalla.subscription_cancel_reasons(userId)` | voice signal | ~100ms |
| `Walhalla.zendesk_tickets(email)` | support voice | ~200ms |

**Параллельне виконання:** `Promise.all` 4 виклики = ~300ms total. Прийнятно для синхронного workflow step.

### Level 3 — Forge memory (per-group winners/losers)

Це те що MVP вже робить. У проді `AggregatorService.getStats` додатково:
- Кешується (Redis) ключем `domain:groupKey` на 5 хвилин — щоб масові батчі не ддосили БД
- Повертає не лише top-K, а й trend (win-rate за останні 7d vs all-time)

### Cost per user enrichment

| Operation | Cost |
|---|---|
| Symonenko BigQuery lookup | $0.000005 |
| 4× Walhalla REST | $0 (internal) |
| forge group_stats query (with cache hit) | $0.000001 |
| **Total enrichment** | **~$0.00001** = noise |

---

## 3. Generation strategy — три моделі, ранжовані

### Strategy A — Per-user real-time generation (найдорожче, найкращий tone)

Кожен push = окремий LLM-call з повним bespoke контекстом.

```
Customer.io HTTP → forge.compose(userId)
  ↓
forge збирає Level 1+2+3 → contextSnapshot
  ↓
forge.composer.service.ts:
  ↓ Anthropic API call:
  - system: brand_voice + safety + group_stats winners/losers (CACHED)
  - user: contextSnapshot
  ↓ response: { reasoning, title, body, confidence }
  ↓
forge.log_attempt → Postgres
  ↓
return { title, body } → Customer.io → send push
```

**Cost per push (Anthropic API):**
- Sonnet 4.6: ~1500 input + 280 output tokens = $0.0090
- Haiku 4.5 (для простих груп): ~1500 + 200 = $0.0014
- Opus 4.7 (для critical groups): ~1500 + 280 = $0.0250

**Optimization layers:**
1. **Prompt caching** (Anthropic feature) — system prompt (brand voice + winners/losers) кешується. -90% на cached tokens. Реальна economy: $0.0090 → $0.0035
2. **Batch API** — для не-real-time push'ів (daily run). -50% cost. $0.0035 → $0.0017
3. **Model routing** — Haiku для груп з win-rate > 0.7 (signal стабільний), Sonnet для решти, Opus тільки для high-stakes сегментів (наприклад churned-paying з великим LTV). Mix ~$0.005 average

**Final realistic cost:** **$0.005/push** при mature optimization.

### Strategy B — Pre-computed group templates with slot-filling (найдешевше)

Раз на добу forge генерить **один шаблон на group_key** з placeholder'ами.

```
Daily cron у forge:
  for each active group_key:
    contextSnapshot = average snapshot of group
    template = LLM.compose(brand_voice, group_stats, average_snapshot)
    // template має slots: {{user_name}}, {{lesson_topic}}, {{streak_days}}
    save template → Postgres (group_template table)

Customer.io workflow:
  HTTP → forge.fill_template(userId, groupKey)
    forge: knows user's group, fills slots from user data
    return: { title: "Marcos couldn't order paella, ${user_name}",
              body: "Try yourself — 5 min" }
```

**Cost:**
- 100 active groups × 1 LLM-call/day × $0.005 = **$0.50/day** generation cost
- Per-push fill_template = БД lookup ~$0.000001
- При 100k push/day = **$0.50/day total** ≈ **$15/month**

**Trade-off:** менш bespoke. Юзер у тій самій групі отримує **той самий шаблон** з різними змінними. Нагадує сучасний CRM, але з LLM-tone замість статичних шаблонів.

### Strategy C — Hybrid (РЕКОМЕНДОВАНО)

Best of both:

```
For each group_key, decide:
  if group.totalAttempts < 30 OR group.winRateConfidence < 0.6:
    → Strategy A (bespoke, треба сигнал)
  elif group.isCritical (high-LTV paying / churn-risk segments):
    → Strategy A (quality > cost)
  else:
    → Strategy B (templated)
```

**Cost model для Promova scale (50k inactive users/day):**

| Tier | % of pushes | Cost/push | Volume/day | Cost/day |
|---|---|---|---|---|
| A — bespoke (Opus) | 5% (critical) | $0.025 | 2,500 | $62.50 |
| A — bespoke (Sonnet+caching) | 25% (low signal) | $0.005 | 12,500 | $62.50 |
| B — templated (Haiku) | 70% (mature groups) | $0.0001 | 35,000 | $3.50 |
| **Total** | 100% | **$0.0026 avg** | **50,000** | **$128/day** |

**$128/day = $3,840/month** на 50k push/day для всього Promova.

Без оптимізації (pure Strategy A Sonnet): $0.012 × 50k = $600/day = $18k/month.

**Економія Hybrid → 30% від наївного підходу.** При 1M push/day (всі продукти org) — економія $60k+/місяць.

---

## 4. Push fatigue — щоб forge не вбив LTV

Push fatigue — реальна загроза. Червень 2025: тест в Promova, де **виключили частину пушів**, дав +46.6% return rate і +58% lesson start rate. Тобто **менше пушів = більше повернень** у певних умовах.

forge може випадково **збільшити** push frequency, якщо просто оптимізує CTR без фрейму.

### Захисні механізми

1. **Per-user push budget** — Customer.io tracks `pushes_sent_last_7d` per user property. forge перевіряє перед генерацією. Якщо `> 5`, повертає `{skip: true, reason: "fatigue cap"}`.

2. **Group-level cap** — `forge_get_group_stats` повертає avg push interval. Якщо група має winners з median interval 5d, не слати 10 пушів за тиждень тому ж юзеру.

3. **Cancel rate guardrail** — forge моніторить `outcome.churned_after_push` як negative outcome (юзер скасував підписку після push'у). Якщо для group_key cancel rate > 5%, **зменшити confidence threshold** для відправки.

4. **Phase 1 manual review** — перших 4 тижні людина переглядає кожен push перед send. Customer.io workflow має «Approval step» natively.

### Як це подати на демо

> «forge не оптимізує "більше пушів". Він оптимізує "правильні пуші у правильний момент". push fatigue handled через Customer.io budget + group-level cap + cancel-rate guardrail. Червневий тест 2025 показав що менше пушів = більше повернень — ми це поважаємо.»

---

## 5. Quality control — 4 фази довіри

З плану v3, але адаптовано під per-user prod:

| Фаза | Тривалість | Що робить forge | Хто approve |
|---|---|---|---|
| **1. Manual review** | Week 1-2 | Genrує і логує. НЕ шле. | Маркетолог + я переглядаємо в db:studio |
| **2. Manual approval** | Week 3-4 | Генерує + ставить у queue. Слає тільки після human approve. | Customer.io approval step |
| **3. Confidence threshold** | Week 5-8 | Auto-send if `predictedConfidence > 0.7`. Manual review if lower. | Daily review of low-confidence skips |
| **4. Full autonomy** | Month 3+ | Cron. Dashboard. SLO на cancel_rate. | Weekly metric review by lead |

**Чутливі домени (наприклад mental health) застрягають на Phase 2 назавжди** — auto-send на тригерних snapshot'ах = ризик реальної шкоди. Це частина дизайну: per-domain `phaseCap` у DomainAdapter обмежує максимальну фазу довіри.

---

## 6. Privacy / Compliance

| Concern | Як закриваємо |
|---|---|
| GDPR — особисті дані у promptах LLM | Anthropic зобов'язується по DPA не зберігати prompts довше 30 днів. Симонів archetype не містить email/name — використовуємо тільки behavioural features |
| User opt-out з push notifications | Customer.io сам це знає (системний prefs API). Voronka навіть до forge не доходить |
| Data retention | engagement_attempts retention = 12 місяців (для outcome trend). Після — TRUNCATE. PII ніколи не зберігається в forge — тільки `user_id` (хеш) + `groupKey` |
| Audit trail | Кожен attempt має reasoning + tokens + cost. Прозорий аудит «чому система написала саме так» — завжди можна показати legal/compliance |
| Sensitive-domain handling (post-MVP) | Hard rules живуть у per-domain voice prompt'ах (на зразок `promova_voice` / `promova_email_voice`); collect-recipe інструктує Claude замінювати distressing phrases абстрактними labels на стороні snapshot'а |

---

## 7. Migration path — з MVP до full prod

### Phase 0 (готово на хакатоні)

forge MCP, Postgres, group_key SQL, Claude Desktop chat. **Manual end-to-end через чат.** Demo learning loop.

### Phase 1 (1-2 тижні після хакатону)

Додати **3 файли** (orchestration layer):
- `orchestration/ingestion.service.ts` — Walhalla REST + Symonenko BigQuery wrapper
- `orchestration/composer.service.ts` — Anthropic SDK + prompt assembly
- `orchestration/engagement-orchestrator.service.ts` — high-level `runOne(userId)` flow

Додати **2 NestJS routes**:
- `POST /compose` — Customer.io HTTP step entry point
- `POST /measure` — outcome update from workflow

Все працює. Customer.io workflow підключається. forge ще не cron'ить — Customer.io тригерить.

### Phase 2 (3-4 тижні)

Додати **Strategy B (templates)**:
- Daily cron генерить group templates
- `POST /fill_template` endpoint
- Customer.io routing: bespoke vs template based on group maturity

### Phase 3 (місяць 2)

Додати **Symonenko BigQuery integration** через service account. Замінити мокнуту `IngestionService` на реальну.

### Phase 4 (місяць 3+)

Додати другий продукт **через тільки конфіг**: новий `DomainAdapter`, нові prompt'и, нові window/activation events. **Жодного перепису core.**

---

## 8. Open questions для product / legal / infra

1. **Хто власник forge-сервера у проді?** Анастасія (CRM lead)? Ivan (Symonenko lead)? Або новий person? — впливає на Phase 1 timeline.

2. **Де хоститься?** Cloudflare Workers (як AI Hub)? AWS Lambda? Окремий ECS? — впливає на cold start, latency, cost.

3. **Скільки юзерів реально в когорті «inactive 10+ днів» одночасно?** Без цього cost model — гіпотетична. Треба запит у BigQuery.

4. **Чи Customer.io workflow може зробити HTTP call з 1s+ latency?** Якщо ні — потрібен async pattern: workflow ставить job у queue, forge відповідає пізніше через інший event.

5. **Anthropic API rate limits на org-tier?** При 50k push/day = 35 RPM avg, peaks ~200 RPM. Standard tier = 50 RPM. Може потребувати Tier 4 ($)|.

6. **Push fatigue cap від Customer.io — який зараз існує?** Якщо немає cap'у — forge введе свій. Якщо є — узгодити з ним.

7. **Чутливі домени (mental health тощо): чи юристи погодять auto-send навіть на Phase 4?** Швидше за все ні — такий домен лишається на Phase 2 назавжди. Це закладено в `DomainAdapter.phaseCap`.

8. **Cross-product юзер**: один email у двох продуктах одночасно. Чий push має пріоритет? — потребує rule engine у Customer.io.

---

## 9. TL;DR для пітча

**Питання жюрі: «Як це масштабується щоб дати кожному юзеру свій push?»**

Відповідь:

> Customer.io workflows тригерять forge через HTTP step. forge збирає контекст з трьох рівнів — Symonenko архетипу (BigQuery), real-time event stream (Walhalla), власна group memory. Викликає Anthropic API з prompt caching і model routing — Hybrid Strategy: bespoke Sonnet для unstable groups, templated Haiku для mature groups, Opus для high-stakes сегментів. На Promova scale 50k push/day це ~$128/день, $4k/місяць. Push fatigue handled через Customer.io budget + group-level cap. 4 фази довіри від manual review до full autonomy. Чутливі домени (mental health, медицина) — стоп-крок на Phase 2 назавжди через `DomainAdapter.phaseCap`. Migration delta — 3 нові файли поверх існуючого core, нічого не переписуємо.

Це і є **прод-roadmap у чотирьох реченнях**. Жюрі бачить що ти **вже подумав** про масштаб, не тільки демо.