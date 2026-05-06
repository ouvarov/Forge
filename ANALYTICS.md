# Forge — Analytics & A/B Testing

Документ-довідка з двома темами:
1. **Що трекаємо** — таксономія івентів `forge_*` для Amplitude
2. **Як доводимо що це працює** — A/B framework для порівняння forge-пушів зі звичайними

Анастасія прямо сказала в #solo-challenge (4 травня 2026, 18:04): *«одна з умов оцінювання це аналітика — логічно шо на фічу треба буде накинути івенти»*. Цей документ — наша відповідь.

---

## 1. Проблема — звідки ми знаємо що forge кращий?

Я завтра скажу «forge дав +30% CTR». Жюрі (або Анастасія, або Влад, або хтось з продактів) питає:

> Звідки ти знаєш? Може просто був кращий тиждень? Може ці 30% юзерів так чи інакше повернулись би?

Без правильного A/B експерименту у нас **нема доказів**. forge на демо може бути красивою конструкцією, але без telemetry і control-групи — це презентація, не доведена фіча.

Тому два паралельні треки:
- **Telemetry** — кожна взаємодія з forge генерує event у Amplitude (без цього взагалі нічого не міряємо)
- **A/B framework** — половина юзерів отримує звичайний push, половина forge, через 14 днів дивимось різницю

Те і те **вже є в Promova як стандартний паттерн** — Learning Buddy MVP так і міряли. Ми не винаходимо колесо.

---

## 2. Event taxonomy — що трекаємо

### Принцип

Кожна tool-взаємодія з forge → один event у Amplitude HTTP API + запис у локальну таблицю `forge_events` (audit trail на демо якщо мережа впала).

Namespace: `forge_*` — щоб не змішуватись з існуючою таксономією Promova.

### Iteration phase (нові — для bootstrap trajectory chart)

| Event | Коли спрацьовує | Properties |
|---|---|---|
| `forge_iteration_started` | початок batch'у | `iteration_number`, `iteration_id` (uuid), `batch_size`, `target_groups` |
| `forge_iteration_completed` | усі outcomes виміряні | `iteration_number`, `iteration_id`, `total_attempts`, `winners_count`, `win_rate`, `avg_confidence`, `brier_score` |

### Generation phase

| Event | Коли спрацьовує | Properties |
|---|---|---|
| `forge_group_stats_queried` | Claude викликав `forge_get_group_stats` | `domain`, `groupKey`, `totalAttempts`, `winRate`, `winnersCount`, `losersCount`, `latency_ms`, `iteration_number` |
| `forge_attempt_generated` | Claude викликав `forge_log_attempt` | `attempt_id`, `domain`, `userId`, `groupKey`, `predictedConfidence`, `model`, `tokensInput`, `tokensOutput`, `costUsd`, `channel`, `iteration_number`, `experiment_name` |
| `forge_decision_skipped` | forge відмовився генерити (guardrail) | `domain`, `userId`, `groupKey`, `skip_reason` (fatigue / cancel-rate / cold-start) |

### Delivery phase (поточні Promova events, ми тільки додаємо properties)

| Event | Хто фірить | Що ми додаємо |
|---|---|---|
| `push_sent` | Customer.io workflow | `+ variant`, `+ last_attempt_id`, `+ groupKey` |
| `push_delivered` | Reteno SDK / iOS / Android | те саме |
| `push_opened` | Promova app on tap | те саме |
| `push_dismissed` | Promova app on swipe | те саме |
| `push_variant_assigned` | Customer.io workflow на entry | новий: `userId`, `variant`, `audience` |

### Engagement phase (existing Promova, додаємо attribution)

| Event | Що каже | Атрибуція |
|---|---|---|
| `lesson_started` | юзер почав урок | user_property `last_attempt_id` дозволяє join |
| `app_session_started` | відкрив апку | те саме |
| `paywall_viewed` | дійшов до paywall | те саме |
| `purchase_completed` | заплатив | те саме |

### Outcome phase

| Event | Коли | Properties |
|---|---|---|
| `forge_outcome_recorded` | `measure_outcome` спрацював | `attempt_id`, `groupKey`, `reactivated`, `activationPatternHit`, `rewardScore`, `daysToOutcome` |
| `forge_activation_pattern_hit` | юзер закрив Anna's metric (1+ урок у 2+ дні) | `attempt_id`, `groupKey`, `daysToActivation` |
| `forge_outcomes_simulated_batch` | demo only — `simulate_outcomes` | `totalUpdated`, `byGroupKey` distribution |

---

## 3. Iterative bootstrap — як ми доводимо що forge вчиться

На MVP **не робимо A/B vs baseline**. Замість цього використовуємо **iterative self-validation**: forge порівнюється з самим собою у часі. Якщо win-rate росте від ітерації до ітерації — система вчиться. Якщо ні — щось не так і ми це бачимо за кілька ітерацій, не за тиждень очікування.

### Схема одного циклу

```
Iteration N:
  1. Customer.io workflow або ручний запит → batch з K inactive юзерів
     (на демо K=5, на проді pilot K=200-500, у проді K=10k+)
  2. Для кожного:
       forge_get_group_stats → forge_compose → forge_log_attempt
       (event: forge_attempt_generated з iteration_number=N)
  3. Кожен attempt має predicted_confidence
  4. (Через outcome window 7 днів)
  5. Customer.io HTTP → forge_measure_outcome для кожного
       (event: forge_outcome_recorded з iteration_number=N, reward_score)
  6. Iteration aggregate:
       win_rate_N = winners / total
       avg_confidence_N
       brier_score_N (calibration error)
       (event: forge_iteration_completed)

Iteration N+1:
  Той самий цикл, але forge_get_group_stats тепер бачить winners з iter N
  → reasoning явно посилається на минулі winners
  → predictedConfidence вища
  → текст pushes відрізняється від iter N
```

### Trajectory chart — головний звіт

Один chart в Amplitude, **`win_rate` over `iteration_number`**:

```
forge_v1 trajectory:
   Iter 1: 40%   (5 attempts, 2 winners)   confidence avg 0.62
   Iter 2: 60%   (5 attempts, 3 winners)   confidence avg 0.71  → +20pp
   Iter 3: 80%   (5 attempts, 4 winners)   confidence avg 0.76  → +20pp
```

Що дивимось:
1. **Win rate trend** — головний сигнал. Росте → вчиться. Плато → проблема.
2. **Confidence calibration trend** (Brier score) — чи модель краще передбачає. Падає → calibration покращується.
3. **Group_key coverage** — чи forge накриває всі важливі групи чи концентрується на одній.

### Decision criteria

| Trajectory pattern | Рішення |
|---|---|
| Win rate монотонно росте 3+ ітерації | ✅ **Перехід у Phase 2** — A/B vs baseline через Customer.io |
| Win rate плато на 50-60% за 3 ітерації | ⚠ Або групи занадто широкі (треба deeper key), або brand voice prompt не працює — review reasoning |
| Win rate коливається без тренду | ❌ Reward function не валідна, або segmentation шумна — back to drawing board |
| Win rate падає | ❌ Серйозна проблема — модель overfit'ить на ранніх winners які виявилися випадковими |
| Brier score не падає при зростаючому win_rate | ⚠ Confidence не калібрується — predicted_confidence не корисний як signal |

### Sample size — скільки треба ітерацій

Для довіри що тренд не випадковий:
- **3 ітерації** — мінімум для патерну. На демо це 15 attempts (5 × 3).
- **5-7 ітерацій** — для високої довіри. На pilot це 25-35 attempts.
- **10+ ітерацій** — для прода. Можна автоматизувати через cron.

Кожна ітерація = тиждень real-time (через outcome window 7d). На демо все симулюємо за 5 хвилин.

### Чому iterative > A/B на старті

| | Iterative bootstrap | A/B vs baseline |
|---|---|---|
| Sample size | 15-50 attempts ок для тренду | 12k+ для stat sig |
| Час до сигналу | 3 ітерації | 2-14 днів |
| Інфра | один call в forge | Customer.io з branch step + dashboards |
| Ризик помилки | низький (немає fake control) | висока (невідомо чи baseline був представницьким) |
| Сприйняття жюрі | очевидний learning curve | абстрактна цифра «+27% CTR» |

A/B стає relevant **після** того як bootstrap довів trajectory. Це Phase 2.

---

## 4. Phase 2 — A/B vs baseline (post-hackathon, 2-4 тижні)

Коли iterative bootstrap довів що win_rate росте, переходимо на A/B щоб виміряти **uplift над baseline'ом**.

### Структура тесту

| Параметр | Значення |
|---|---|
| **Test name** | `forge_v1_ab` |
| **Control** | поточний static template або existing AI broadcast (обираємо разом з marketing-led) |
| **Treatment** | forge per-user generated |
| **Allocation** | 50/50 random, sticky per `userId` |
| **Assignment seed** | `hash(userId) % 2` — стабільне між сесіями |
| **Audience** | inactive 7+ днів, opt-in, fatigue cap не досягнутий |
| **Period** | 14 днів |

### Customer.io workflow з branch step

```
Audience entry: user.last_lesson_started < (NOW - 7d)
                AND user.push_opt_in == true

  ↓ Wait random 0-6h (розподіл навантаження)
  ↓
[Branch by hash(userId) % 2]
  │
  ├─ Control (50%):
  │     ↓ Send static template
  │     ↓ user_property: ab_variant = 'control'
  │
  └─ Treatment (50%):
        ↓ HTTP step → forge.compose(userId)
        ↓ Send: $.title, $.body
        ↓ user_property: ab_variant = 'forge', last_attempt_id = $.attemptId
  ↓
[Wait 7 days]
  ↓
[HTTP step → forge.measure(attemptId)]   ← тільки treatment arm
```

### Метрики (5 шарів)

| Шар | Метрика | Тип |
|---|---|---|
| **Engagement** | CTR (opened / sent) | Primary |
| | Time-to-open median | Secondary |
| **Behavioral** | Lesson started 24h after push | Primary |
| | Activation pattern hit (Anna's metric, 1+ урок у 2+ дні) | **North star** |
| **Revenue** | ARPS 7d | Secondary |
| | Trial→Payment | Secondary |
| **Guardrails** | Cancel rate 30d | **Stop signal** |
| | Push opt-out rate | **Stop signal** |
| **Cost** | $/activated user | Unit economics |

### Sample size

База ~1.5% CTR retention. Для +25% uplift, alpha=0.05, power=0.8:
- **n ≈ 6 200 per arm**, ≈ 12 400 total
- При 50k inactive/day → **2 дні до stat sig CTR**
- Для activation pattern → **7-10 днів**

### Decision criteria

| Сценарій | Рішення |
|---|---|
| forge CTR > control AND activation > control AND cancel ≤ control | ✅ **Roll out 100%** |
| forge CTR > control AND activation = control | ⚠ Клікбейтить — review prompts, додати safety |
| forge CTR = control AND activation > control | ✅ Roll out (силова продуктова метрика) |
| forge CTR < control OR cancel +1pp vs control | ❌ **Roll back**, аналіз reasoning |
| Mixed signals — переміг на одних group_keys | 📊 **Per-group rollout** через conditional steps |

### Як виглядатиме звіт

```
forge_v1_ab — 14 days (6,200 sent per arm)
                Control    Forge      Δ          Stat sig
CTR:            1.52%      1.94%      +27.6%     ✅ p=0.012
Lesson 24h:     4.1%       5.3%       +29.3%     ✅ p=0.008
Activation 7d:  3.8%       5.0%       +31.6%     ✅ p=0.014
Cancel rate:    0.8%       0.7%       -12.5%     — n.s.
ARPS 7d:        $24.30     $27.10     +11.5%     ✅ p=0.041

Decision: ROLL OUT
```

Той самий патерн що Learning Buddy MVP. Marketing-led знає як це робити, інфраструктура у Promova існує.

---

## 5. Channel-agnosticism — forge для всіх каналів, не тільки push

forge генерить **контент**, не канал. Output `{title, body, reasoning}` універсальний.

### Підтримувані канали

| Канал | Як працює | Constraints |
|---|---|---|
| **Native push** (iOS APNS / Android FCM) | Customer.io → Reteno SDK → APNS/FCM | title ≤40 char, body ≤120 char |
| **Email** | Customer.io transactional API. title → subject, body → preheader/body | Дуже широкі ліміти, body 100-500 char оптимально |
| **In-app** | Promova app notification center | Same as push |
| **SMS** | Customer.io → Twilio | title+body merged ≤160 char |
| **Web push** | Customer.io browser SDK | Same as native push |
| **Slack/Discord** (B2B) | Customer.io webhook | markdown ok |

### Як forge тюнить під канал

У production додаємо `channel` параметр в `/compose`:

```ts
forge.compose({ userId, channel: 'push' | 'email' | 'sms' | 'in_app' | 'web_push' })
```

forge всередині:
1. **Length constraints у промпті** — push «титул ≤40 char, body ≤120», email «body 100-500 char»
2. **Tone варіація** — push casual/punchy, email повніший наратив
3. **Group key розширюється каналом** — `promova:es:B1:churned-7-14d:lesson-food:push` ≠ `:email`. Те що зайде в push не обов'язково зайде в email — різні winners для різних каналів
4. **Метрики окремо** — CTR push (~1.5%) vs email open (~30%) — несумісні baseline'и

### Native push специфіка

Native push в Promova вже працює через Reteno SDK (iOS APNS + Android FCM). Customer.io має нативну integration з Reteno. forge → Customer.io → Reteno API call → push на пристрої. Жодних змін у forge для native vs web — Customer.io єдина точка контакту.

### Що міняти в схемі для channel'ів

`engagement_attempts.channel` колонка вже існує (default 'mock'). У проді:
- 'push' / 'email' / 'sms' / 'in_app' / 'web_push'
- group_key включає channel
- метрики розраховуються per-channel

---

## 6. Implementation impact

Додаємо **2 файли** до §7 PLAN.md:

| # | Файл | Зміст |
|---|---|---|
| 12 | `core/analytics.service.ts` | Amplitude HTTP API v2 client + Postgres `forge_events` mirror. ~30 рядків |
| 13 | `db/schema/forge-events.ts` | Drizzle schema для events table (id, event_type, user_id, properties jsonb, created_at, attempt_id FK) |

Кожен MCP tool після успішного execute() викликає `analyticsService.track(eventName, userId, properties)`. ~5 рядків на tool.

**Жодного додаткового MCP tool'у не треба** — analytics — внутрішній service, не зовнішній API.

### Pseudocode

```ts
// core/analytics.service.ts
@Injectable()
export class AnalyticsService {
  constructor(private readonly drizzle: DrizzleService) {}

  async track(
    eventType: string,
    userId: string,
    properties: Record<string, any>,
  ): Promise<void> {
    // 1. Save to local Postgres (audit trail, offline-safe)
    await this.drizzle.db.insert(forgeEvents).values({
      eventType, userId, properties,
      attemptId: properties.attempt_id ?? null,
    });

    // 2. Fire to Amplitude (async, fire-and-forget)
    fetch('https://api2.amplitude.com/2/httpapi', {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({
        api_key: process.env.AMPLITUDE_API_KEY,
        events: [{
          user_id: userId,
          event_type: eventType,
          event_properties: properties,
          time: Date.now(),
        }],
      }),
    }).catch(() => {/* don't block on Amplitude */});
  }
}
```

---

## 7. Production rollout protocol

Послідовність після хакатону:

| Phase | Тривалість | Що робиться | Хто approve |
|---|---|---|---|
| **0. MVP demo** | 2 дні | forge + iterative bootstrap (3 ітерації × 5 attempts) на симульованих outcomes | хакатон |
| **1. Wire-up** | 3-5 днів | Customer.io HTTP step → forge.compose. Amplitude events live. **No A/B yet** — single-arm bootstrap | marketing-led + я |
| **2. Iterative pilot** | 4-6 тижнів | 5-7 ітерацій × 100-500 attempts на closed cohort. Watch trajectory. Якщо win_rate росте — Phase 3 | data analyst + product |
| **3. A/B vs baseline** | 14 днів | Customer.io branch step (control / forge), 50/50 sticky. CTR + activation pattern stat sig | data analyst |
| **4. Decision** | 1 день | звіт за rubric'ом §4 decision criteria, go/no-go | продакт-команда |
| **5. Rollout** | 7 днів | 50% → 75% → 100% gradual, або per-group rollout | marketing-led |
| **6. Continuous improvement** | ongoing | кожна зміна prompt'у → новий iterative bootstrap → новий A/B | marketing + я |

Це **не one-shot тест** — це **постійний механізм**. Кожна зміна промптів проходить bootstrap → A/B цикл. Forge ніколи не зупиняє вчитись.

**Ключова відмінність від традиційного A/B-only підходу:** ми не починаємо з 12k юзерів навмисне. Iterative bootstrap дозволяє ловити проблеми (плато, overfitting, шумна segmentation) **на 50 attempts** замість 12000. Економія часу і ризиків.

---

## 8. Why this matters for hackathon scoring

Анастасія сказала прямо: **аналітика — критерій оцінки**. Що ми маємо:

| Що жюрі побачить | Доказ |
|---|---|
| Telemetry як design principle, не afterthought | Event таксономія — частина MVP, не «коли-небудь додамо» |
| Розуміння як вимірювати ефект на малих даних | Iterative bootstrap з trajectory chart замість потреби 12k attempts |
| Зрілий A/B плановий підхід | Phase 2 з sample size formula, decision criteria — той самий патерн що Learning Buddy MVP |
| Зрілий mindset до власної фічі | Iterative trajectory + roll-back criteria explicit |
| Production-ready пайплайн вимірювання | Customer.io HTTP + Amplitude funnel + Brier calibration |
| Channel-agnosticism | forge не прив'язаний до push, легко розширюється на email/SMS/in-app |

Це бере rubric з 17/20 на Analytics → **20/20**.

---

## 9. Open questions

1. **Який baseline для control?** Static Reteno template або existing AI broadcast Vlad'а? Якщо AI broadcast — отримаємо чистіший «forge vs AI broadcast» sigнал. Це обговоримо з marketing-led.
2. **Який segment для pilot?** Conservative pick: free-tier учні з language=es (найбільше даних) і daysSinceActive=7-14d.
3. **Хто owner дашборду?** В Amplitude треба людину з admin-правами для створення saved chart і funnel. Ймовірно — Anna Z (analytics-lead).
4. **AMPLITUDE_API_KEY** — secret для forge сервера. Vault через AI Hub паттерн або env через Cloudflare Workers.
5. **Чи Amplitude HTTP API rate limits витримує 50k events/day?** Стандартна межа — 100/sec. У нас peak ~5/sec — комфортно.
6. **Privacy**: чи можна юзерські `groupKey` слати в Amplitude? Це не PII (це bucket label), але треба узгодити з legal.

---

## 10. TL;DR для пітча

**Питання жюрі: «А як ви взагалі зрозумієте що forge працює краще?»**

Відповідь:

> Двофазно. **Phase 1 — iterative bootstrap.** На pilot cohort 200-500 юзерів запускаємо 5-7 ітерацій по 50-100 attempts кожна. Кожна ітерація: forge генерить → ми чекаємо outcome window → forge сам себе оцінює (winner/loser) → наступна ітерація використовує цей feedback. Один chart в Amplitude показує trajectory: win_rate × iteration_number. Якщо росте — система вчиться. Якщо плато — щось не так, бачимо за 50 attempts а не за 12k.
>
> **Phase 2 — A/B vs baseline.** Коли trajectory довела свій тренд, переходимо на 50/50 split з Customer.io branch step. Той самий патерн що Learning Buddy MVP: 14 днів, sample size 12k, decision criteria по CTR + activation pattern + cancel rate.
>
> На демо показуємо Phase 1 вживу за 5 хвилин — 3 ітерації, win_rate 40% → 60% → 80%. Phase 2 — слайд з готовою інфраструктурою: «запустимо одразу після хакатону, infra існує».

Конкретніше нічого не треба. **Це і є відповідь.**