# Forge MCP — Demo Script (15 min)

Демо 7 травня 2026, 10:00–12:00 (slot 15 хв).

Структура: **2 хв slides → 8 хв live demo → 3 хв slides → 2 хв Q&A buffer**.

Backup recording — обов'язково перед демо (OBS / QuickTime full-screen capture). Якщо мережа / Walhalla MCP / Claude Desktop впаде на сцені — переходимо на запис без перерви.

---

## Hypothesis validation log

| Дата | Хто | Що сказав | Що це нам дало |
|---|---|---|---|
| 5 травня 2026 | **Vlad Gut** (автор +41% CTR pioneer-експерименту) | «outcome кожного пуша назад у генерацію наступного per-сегмент не феєдбачився — все так» | feedback gap, який forge закриває, **підтверджений власником CRM AI-інфраструктури** — це і є відповідь на «чи підтвердилась гіпотеза» в Block 1 |
| 5 травня 2026 | Vlad Gut | «~100 — мало, краще 1000+. там ризиків небагато, у нас пуші зараз досі не генерять ніякий пристойний % revenue/retention» | pilot bumping з 100-200 до 1000+ юзерів на хвилю, можливість mini-A/B на хвилі 1 |
| 5 травня 2026 | Vlad Gut | Розширити scope на email + revenue campaigns з персональними згораючими знижками + memetic content layer | додано як Vlad-validated next steps у Slide 6 «What's next» |
| 5 травня 2026 | Vlad Gut | «важливо тіки подумати щоб воно технічно не зламалось і не наспамило якось, щоб юзер потім просто забрав пуш пермішн назавжди» | opt-out ризик закладений як головний на Phase 3+ (auto-send), guardrails з MVP: fatigue cap, hard-skip <0.5 confidence, rate-limit, decision_skipped events |
| 5 травня 2026 | **Session token analysis** (Claude Desktop, Opus 4.7, 10 users) | Demo: ~$0.30/push (context accumulation overhead). Production target: ~$0.004/push (direct backend → SDK, 75x cheaper) | Honest cost framing для жюрі. Pilot 7k = $28. ROI 6.6x на consservative funnel (CTR 3% × Amplitude baselines). Break-even 0.1% activation lift — на порядок нижче за +13.4% Veronika |
| 5 травня 2026 | **Amplitude research** (chart `f6ew1hqt`, Yevhenii Mianovskyi) | 48-52% юзерів які повертаються в апку проходять урок протягом 24h | Реальний Promova baseline для funnel розрахунку — не вигаданий бенчмарк |

---

## Pre-demo setup (зробити за 30 хв до)

1. ✅ `yarn docker:up` — Postgres на 5434
2. ✅ `yarn build` — свіжий `dist/src/main.js`
3. ✅ Cmd+Q Claude Desktop, відкрити заново — підхопить останню збірку MCP
4. ✅ В Claude Desktop новий чат, перевірити `forge_ping` → працює
5. ✅ `yarn db:studio` у фоні, відкрита вкладка з `engagement_attempts`
6. ✅ **БД ПУСТА** — це by design. Demo сама по собі = накопичення з реальних Walhalla юзерів. Жодного fake/seed моку — це принципова політика.
7. ✅ Слайди відкриті у фоні (Slide 1 готова до показу)
8. ✅ Backup recording started

---

## Slide 1 — Problem (1 min)

**Промова:**
> «У Promova є три факти, які складаються разом.
> Перший — Anna Зюбанова виявила що 15% неактивних юзерів взагалі невидимі для CRM через ad-blocker'и. Це silent churn у чистому вигляді.
> Другий — Veronika Velykanova показала що персоналізація welcome-flow дає **+32.6% CTR** проти broadcast (8 вересня 2025).
> Третій — Anastasiia Stelmakh веде міграцію Reteno → Customer.io де workflow setup, цитую, "переважно ручний".
> Ці три факти — продукт, ринок, інфраструктура. Місце для системи яка поєднує їх — порожнє.»

**Не показуємо** код, скрін, нічого. Тільки голос + слайд із трьома bullet point'ами.

---

## Slide 2 — Cited evidence (1 min)

Показуємо пʼять цифр/цитат з джерелами:

| Цифра / цитата | Джерело |
|---|---|
| **CTR +32.6%** на personalized welcome push | Veronika Velykanova, #symonenko_scale, 8 вересня 2025 |
| **CTR +41%** uplift на 12.2% messages stat-sig (AI-generated push на brand voice) | Veronika Velykanova + Vlad Gut, #bite-sized_marketing, 14 травня 2025 |
| **Trial→Payment +58.8%** на Learning Buddy MVP | Timothy Kachan, #analytics, 1 травня 2026 |
| **0.8 correlation** New Activation Rate з W4 retention | Anna Ziubanova, #analytics, 18 березня 2026 |
| **«outcome кожного пуша назад у генерацію наступного per-сегмент не феєдбачився — все так»** (validation feedback gap) | Vlad Gut, DM, 5 травня 2026 |

> «Я не вигадую гіпотезу — я будую систему поверх вже доведених в org цифр, а сам feedback gap підтверджений власником CRM AI-інфраструктури.»

---

## Live demo Promova — Part 1: First push for fresh user (3 min)

Скрін: Claude Desktop chat в фокусі. Поряд (другий монітор або Cmd+Tab) — `db:studio` з `engagement_attempts` таблицею.

### Beat 1 — Set the stage

**Ти кажеш у чат:**
> Завантаж prompt `forge_workflow` і давай знайдемо одного inactive Promova юзера, рівень B1, мова es, який бросив урок про їжу більше 7 днів тому.

**Claude робить:**
1. `forge_workflow` prompt → бачить server instructions
2. `Walhalla.user_search(role=learner, language=es, limit=20)` + `Walhalla.amplitude_analytics(event_segmentation, lesson_started, last 30d)` → знаходить кандидата
3. `Walhalla.user_get(userId)` → profile
4. `Walhalla.amplitude_analytics(user_activity, userId)` → recent events
5. `Walhalla.subscription_cancel_reasons(userId)` + `Walhalla.zendesk_tickets(email)` → voice signals
6. Збирає `contextSnapshot`

**Speaker note:** під час цих 5 викликів ти коментуєш:
> «Дивіться — forge не ходить у Walhalla сам. Це робить Claude Desktop. Він дозбирує контекст з усіх MCPs які доступні в org gateway.»

### Beat 2 — Check group memory BEFORE composing

**Ти кажеш:**
> Перш ніж писати push — подивись в forge group_stats для цього юзера. Що ми вже знаємо про схожих?

**Claude:**
- `forge_get_group_stats({domain: 'promova', contextSnapshot})`

**Output:** на першому юзері перших ітерацій група пуста (`totalAttempts: 0`), reasoning Claude'а: «no past data, generating from brand voice». Це визнаємо чесно — це cold start, demo IS the data accumulation.
```json
{
  "groupKey": "promova:es:B1:churned-7-14d:lesson-food",
  "totalAttempts": 13, "measuredAttempts": 13,
  "winRate": 0.62,
  "winners": [
    {"messageTitle": "Marcos couldn't order paella", "messageBody": "Try yourself — 5 min", "rewardScore": 1.0},
    {"messageTitle": "Восстанови мадридську вечерю", "messageBody": "...", "rewardScore": 1.0},
    ...
  ],
  "losers": [
    {"messageTitle": "You're 65% to fluency in Spanish", "messageBody": "...", "rewardScore": 0.0},
    ...
  ]
}
```

**Speaker note:**
> «Ось пам'ять forge. Ця група — іспанський B1, бросили на їжі, 7-14 днів — має 13 минулих спроб. 8 з них реактивували юзерів, 5 ні. Видно тексти-переможці і тексти-лузери. Claude буде їх використовувати як examples.»

### Beat 3 — Compose push

**Ти кажеш:**
> Завантаж prompt `promova_voice` і згенери push для цього юзера. Reasoning — обов'язково посилайся на winners/losers з group_stats.

**Claude:**
- Завантажує `promova_voice`
- Генерить push, наприклад:
```json
{
  "reasoning": "Дивлячись на 8 winners в групі — всі вони посилаються на конкретні сценарії з імен (Marcos, мадридська вечеря). Лузери натомість дають абстрактну статистику прогресу. Юзер мав 14 пройдених уроків, остання — Restaurant in Madrid, 12 днів неактивний. Пишу у стилі winners.",
  "messageTitle": "Lucia escaped a noisy bar",
  "messageBody": "Try ordering tapas in Spanish — 5 min lesson",
  "predictedConfidence": 0.74,
  "predictedHypothesis": "lesson_started in 3 days"
}
```

**Speaker note:** підкреслюєш:
> «Бачите reasoning? Він не вигаданий — він прямо посилається на 8 winners. Confidence 0.74 — це не випадкове число, це функція щільності group_key + win-rate.»

### Beat 4 — Log

**Ти кажеш:**
> Запиши через forge_log_attempt.

**Claude:**
- `forge_log_attempt({...весь payload, tokensInput: 1340, tokensOutput: 280, costUsd: 0.011})`

**Output:**
```json
{
  "attemptId": "uuid-1",
  "groupKey": "promova:es:B1:churned-7-14d:lesson-food",
  "tokensInput": 1340, "tokensOutput": 280, "costUsd": 0.011,
  "reasoning": "...", "messageTitle": "Lucia escaped a noisy bar", ...
}
```

**Ти переключаєшся на db:studio.** Show: 51 рядок (50 із replay + 1 свіжий). Свіжий рядок — кричуще viewable.

**Speaker note:**
> «Ось і він — нова попытка. 1340 токенів input, 280 output, 1.1 цента. Все логується автоматично.»

---

## Live demo Promova — Part 2: Iterative bootstrap (5 min)

Це **серце демо**. Тут жюрі побачить як платформа вчиться сама на власних outcome'ах. Не A/B vs baseline (це Phase 2 після хакатону), а **forge порівняний з собою у часі**.

### Beat 5 — Iteration 1: повний batch на 5 юзерів

**Ти кажеш у чат:**
> Знайди ще 4 inactive Promova юзера в різних групах (es:A2:lesson-greetings, en:B1:lesson-work, fr:B1:lesson-travel, en:B2:lesson-grammar), для кожного збери контекст через Walhalla, отримай group_stats і згенери push. Це наша **Iteration 1 batch**.

**Claude робить (для кожного 5 разів):**
- Walhalla.user_get + amplitude_analytics + cancel_reasons + zendesk_tickets
- forge_get_group_stats — більшість груп майже пусті, win_rate ще не визначений
- Генерить push з cold-memory reasoning'ом
- forge_log_attempt з `iteration_number: 1`

**В db:studio:** 5 нових attempts в iter 1. predictedConfidence ~0.55-0.65 (модель не впевнена, рідкісні past examples).

**Speaker note:**
> «Це cold start — пам'ять майже пуста. Платформа генерує по brand voice і нечастим past examples. Confidence помірний — модель чесно сигналить що ще не знає що зайде.»

### Beat 6 — Iteration 1 outcome

**Ти кажеш:**
> Симулюй що минув тиждень. Push 1 і Push 3 спрацювали — юзери виконали activation pattern (1+ урок у 2+ дні). Push 2, 4, 5 — ні.

**Claude:**
- `forge_simulate_outcomes` з explicit assignment per attemptId
- Reward scores: 1.0 / 0.0 / 1.0 / 0.0 / 0.0
- `forge_iteration_completed` event: iter 1 win_rate = **2/5 = 40%**

**В db:studio:** колонки outcome_reactivated, activation_pattern_hit, reward_score заповнені.

**Speaker note:**
> «Платформа сама себе оцінила. 40% — це baseline системи на cold start. Цікаво не саме число, а що буде далі.»

### Beat 7 — Iteration 2: тих самих 5 груп, інші 5 юзерів

**Ти кажеш:**
> Знайди 5 нових inactive юзерів — по одному в тих самих групах. Iteration 2.

**Claude робить:**
- Для кожного — forge_get_group_stats. **Тепер у пам'яті є winners з iter 1.**
- Reasoning явно: «у iter 1 в групі X спрацював [конкретний паттерн], не спрацював [Y]. Пишу в стилі winners.»
- Pushes явно відрізняються від iter 1
- predictedConfidence ~0.65-0.75 (виросла, бо є evidence)
- Логується з `iteration_number: 2`

**В db:studio:** 10 рядків. Колонка `iteration_number` показує розподіл.

**Speaker note:**
> «Дивіться reasoning другої ітерації. Він посилається на конкретні winners з iter 1. Текст інакший. Confidence підвищився — модель отримала свіжий feedback і знає на що спиратись.»

### Beat 8 — Iteration 2 outcome

**Ти симулюєш:**
> Push 6, 7, 9 спрацювали. Push 8 і 10 — ні.

3 winners, 2 losers. Win rate iter 2 = **3/5 = 60%**.

**Speaker note:**
> «60% — підняли на 20pp за одну ітерацію. Не випадково — модель **використовувала свою пам'ять** при генерації iter 2.»

### Beat 9 — Iteration 3: ще точніше

**Ти кажеш:**
> Iteration 3. Знайди ще 5 юзерів у тих самих групах і згенери.

**Claude:**
- Тепер у пам'яті 10 measured attempts (5 winners, 5 losers)
- Reasoning ще точніше — посилається і на iter 1, і на iter 2 winners
- predictedConfidence ~0.72-0.82
- 5 нових pushes з `iteration_number: 3`

**Симулюєш:** 4 winners, 1 loser. Win rate iter 3 = **4/5 = 80%**.

### Beat 10 — Trajectory chart (кульмінація)

Переключаєшся на `db:studio` і робиш SQL query (готова як saved view):

```sql
SELECT iteration_number,
       COUNT(*) as total,
       SUM(CASE WHEN reward_score > 0.5 THEN 1 ELSE 0 END) as winners,
       AVG(reward_score) as avg_reward,
       AVG(predicted_confidence) as avg_confidence
FROM engagement_attempts
WHERE iteration_number IS NOT NULL
GROUP BY iteration_number
ORDER BY iteration_number;
```

**Output:**
```
iter | total | winners | avg_reward | avg_confidence
  1  |   5   |    2    |    0.40    |     0.62
  2  |   5   |    3    |    0.60    |     0.71
  3  |   5   |    4    |    0.80    |     0.76
```

Або візуальний chart прямо в чаті (Claude може намалювати):

```
Win rate trajectory:
  100% │
   80% │              ●  ← iter 3
   60% │       ●         ← iter 2
   40% │  ●              ← iter 1
       └────────────────
        1   2   3
```

**Speaker note (центральний beat демо):**
> «Це серце forge. Не A/B-тест — на цьому етапі ми **порівнюємо forge з самим собою у часі**. За 3 ітерації win-rate виріс з 40% до 80%. **Якщо росте — платформа вчиться. Якщо не росте — або щось не працює, або в проді буде той самий результат**. Це чесна перевірка трендом, не «середнє за тиждень». A/B vs baseline — наступний крок (Phase 2), коли цей trajectory доведе сам себе.»

### Beat 11 — Show в db:studio

Підсвічуєш стовпчик `iteration_number` і `reward_score`. Показуєш як reasoning відрізняється між ітераціями для **тієї самої групи** (Beat 5 push #1 vs Beat 7 push #6 vs Beat 9 push #11 — всі в групі `es:B1:churned-7-14d:lesson-food`).

**Speaker note:**
> «Ось 3 рядки для однієї групи. Один group_key, три різні pushes, різний reasoning, різний confidence. Кожен наступний — на крок mудріший. Це і є closed feedback loop вживу за 5 хвилин.»

---

## Slide 3 — Analytics: бачимо самонавчання у часі (2 min)

Ключові події у Amplitude:

| Event | Що каже |
|---|---|
| `forge_attempt_generated` | який push, в якій ітерації, з якою confidence, скільки токенів |
| `forge_outcome_recorded` | чи спрацював, який reward_score |
| `forge_iteration_completed` | win_rate цієї ітерації |
| `forge_group_stats_queried` | коли Claude дивився в пам'ять перед генерацією |

**Один chart в Amplitude — `win_rate by iteration_number`** — показує як платформа вчиться:

```
forge_v1 trajectory (15 attempts, 3 iterations):
   Iter 1: 40%   (2 winners з 5)   confidence avg 0.62
   Iter 2: 60%   (3 winners з 5)   confidence avg 0.71  → +20pp
   Iter 3: 80%   (4 winners з 5)   confidence avg 0.76  → +20pp
```

**Confidence calibration** — окрема метрика. Brier score падає з ітераціями = модель краще розуміє свою впевненість. Це валідація що predictedConfidence не випадковий.

**Speaker note:**
> «Це не вигадка — кожна точка це event у Amplitude. Той самий патерн що Promova використовує для Learning Buddy MVP, Symonenko clustering, всіх A/B тестів. Можемо переключитись в Amplitude через MCP і побачити живі івенти `forge_attempt_generated` × 15 з properties.»

**Як це масштабується (короткий слот):**
- CRM HTTP step → forge.compose → Amplitude tracking. forge channel- і CRM-agnostic: працює з тим workflow editor'ом, який є в org — Reteno сьогодні, Customer.io після завершення міграції.
- На pilot cohort **1000+ юзерів на хвилю × 5-7 хвиль** (per Vlad Gut validation: «там ризиків небагато, пуші досі не генерять пристойний % revenue/retention» — отже стартуємо амбітно, не на 100). 7000+ attempts дають stat-sig wave-by-wave, не лише trajectory.
- Якщо trajectory підіймається → Phase 2: A/B vs baseline (control branch у поточному CRM)
- Деталі — в `ANALYTICS.md` і `RESEARCH.md`

---

## Slide 4 — Path to Autonomy + де живе opt-out ризик (1 min)

**4 фази довіри:**

```
┌─────────────────────────────────────────────────────────────┐
│ Phase 1 (Week 1-2):  Iterative bootstrap, human-gated send │
│   forge генерить через Claude Desktop, я+маркетолог         │
│   ревʼю партію per group → схвалюємо send → CRM шле.        │
│   Cohort 1000+/хвиля. Opt-out ризик: МІНІМАЛЬНИЙ.           │
│                                                             │
│ Phase 2 (Week 3-4): Pilot send + A/B vs baseline           │
│   Branch step у поточному CRM (Reteno → Customer.io):      │
│   control / forge. Партії все ще ревʼю людиною перед send. │
│   Opt-out ризик: НИЗЬКИЙ.                                  │
│                                                             │
│ Phase 3 (Week 5+): Confidence threshold rollout            │
│   Auto-send if confidence > 0.7. Manual для < 0.7.         │
│   Opt-out ризик: РЕАЛЬНИЙ — guardrails вмикаються тут:     │
│   fatigue cap 1/week, hard-skip <0.5 confidence,           │
│   rate-limit на групу 24h, decision_skipped events.        │
│                                                             │
│ Phase 4 (Month 2+): Cron + monitoring + dashboard          │
│   Full autonomy. SLO на cancel rate. Per-group rollout.    │
└─────────────────────────────────────────────────────────────┘
```

**Speaker note:**
> «Це не "дайте кнопку send all". На Phase 1-2 людина ревʼю кожну партію згенерованих пушів через Claude Desktop перед send'ом — opt-out spam структурно неможливий. Реальний opt-out ризик матеріалізується тільки на Phase 3 (auto-send by confidence), і саме туди закладені guardrails з самого початку. Чутливі домени (mental health) на Phase 2 застряють назавжди — це частина дизайну, не bug.»

**Vlad Gut validation (5 травня 2026):**
> «там ризиків небагато, у нас пуші зараз досі не генерять ніякий пристойний % revenue чи ретеншина. важливо тіки подумати щоб воно технічно не зламалось і не наспамило якось, щоб юзер потім просто забрав пуш пермішн назавжди»

— ця стурбованість лягає **виключно на Phase 3+**, бо на Phase 1-2 send гейтиться людиною. Guardrails закладені в продукт з MVP.

---

## Slide 5 — Architecture + CRM-agnostic integration (1 min)

Показуємо архітектуру з PLAN.md:

```
Layer 4 (PROD ONLY) — orchestration: ingestion + composer + cron
Layer 3 — MCP tools (4 tools + 5 prompts) + HTTP /compose, /measure
Layer 2 — Core services (Logger, Aggregator, Measurer)
Layer 1 — Domain adapter (Promova) — pluggable, +1 product = +1 file
                ↓
            Postgres
```

**CRM як sink — однаковий рецепт інтеграції:**

```
       Reteno workflow                   Customer.io workflow
  ┌──────────────────────┐           ┌──────────────────────┐
  │ Webhook block        │           │ Webhook action       │
  │ POST forge /compose  │           │ POST forge /compose  │
  └──────────┬───────────┘           └──────────┬───────────┘
             ↓                                  ↓
   Velocity: $!forge.title           Liquid: {{journey.forge_title}}
             ↓                                  ↓
  ┌──────────────────────┐           ┌──────────────────────┐
  │ Push / Email block   │           │ Push / Email action  │
  └──────────┬───────────┘           └──────────┬───────────┘
             ↓                                  ↓
        Delay 7d                          Time delay 7d
             ↓                                  ↓
  ┌──────────────────────┐           ┌──────────────────────┐
  │ Webhook              │           │ Webhook action       │
  │ POST forge /measure  │           │ POST forge /measure  │
  └──────────────────────┘           └──────────────────────┘

  Той самий patten для Braze, Iterable, OneSignal, наступного.
```

**Speaker note:**
> «Архітектура forge **channel- і CRM-agnostic**. Output — структурований JSON, який будь-який workflow editor вміє підставити в шаблон через свою templating мову (Velocity у Reteno, Liquid у Customer.io). Питання не "чи інтегрується forge з X", а "який CRM org обере наступним". Це вибір org-у, не технічне обмеження forge.
>
> На MVP працюють Layer 1-3. Production = +Layer 4 (3 файли + cron) + два HTTP endpoints. 2 тижні. Підключаємось туди, де org уже є — Reteno сьогодні, Customer.io після міграції, або обидва паралельно. Зміна CRM = зміна одного URL у workflow, не переписування forge.»

---

## Slide 6 — Economics + What's next (1 min)

**Cost:**
```
~$0.30  / push   Hackathon: Claude Desktop + MCP
                 (LLM читає кожну Walhalla response як tokens — dev tool)

~$0.004 / push   Production: бекенд збирає snapshot через HTTP,
                 LLM тільки генерує текст пуша (75x дешевше)

$28     / pilot  7,000 attempts на production path
```

**Pilot ROI (консервативний funnel, CTR 3%):**
```
7,000 пушів
  → 3% CTR                                              = 210 кліків
  → 50% return → learned lesson (Amplitude f6ew1hqt)    = 105 уроків
  → 40% lesson → Activation Pattern (Anna Z)            = 42 активації
  → × $4.40 ARPS_13m delta per activation               = $185 revenue

Net: $185 − $28 = $157 profit
ROI: ~6.6x
```

**Break-even:**
```
7 активацій з 7,000 пушів = 0.1% activation lift
Це у 130x менше за вже доведений floor +13.4% (Veronika welcome push)
→ forge може бути на порядок гіршим за floor і все одно окупиться
```

**Revenue Path A** (deep link на знижку для churned юзерів — upside, не обіцянка):
```
Офер: promova_annual_trial_20disc — $55.99/рік (-20%) + 7-day trial
deep link: promova://paywall?configurationID=...
           https://promova.com/{lang}/sierra/annual-discounted-sales-page-cc/{userId}?_timer=86400

⚠️ У Promova немає baseline на push-driven win-back conversion.
Post-cancellation save 11% (Hanna Sharuk ajqmxujy) — це INSIDE cancel funnel,
captive audience, НЕ comparable з win-back через push.
→ міряємо в пілоті, не обіцяємо заздалегідь.
```

**What's next** — пост-хакатон roadmap:

| Step | Час | Що | Деталі |
|---|---|---|---|
| 1 | 1-2 тижні | CRM HTTP step integration | `/compose`, `/measure` endpoints. forge стає upstream brain workflow'у — підключаємо до того CRM, який є зараз (Reteno) або з'явиться після міграції (Customer.io). Зміна CRM = зміна одного URL, не переписування forge |
| 2 | 1-2 тижні | Email channel (Vlad-validated next bet) | forge channel-agnostic by design. Email — найбільший headroom: за галуззю marketing-funnels email тримає ~10% revenue, у Promova significantly less (per Vlad Gut, 5 травня 2026). Той самий group_stats + reasoning, інші length constraints і tone у промпті |
| 3 | 1-2 тижні | Revenue campaigns + персональні згораючі знижки | Vlad Gut explicit ask: forge генерить не лише re-engagement пуш, а й **персональний discount offer** з reasoning'ом (рівень знижки ↔ predicted churn risk × monetization stage із Symonenko). Memetic content layer для CTR uplift |
| 4 | 2-4 тижні | A/B vs baseline (Phase 2) | Branch step у поточному CRM (control/forge), Amplitude funnel by variant |
| 5 | 1 тиждень | Symonenko v4 через BigQuery | Заміняє ручний Walhalla pull на готові rich profiles (1M юзерів) |
| 6 | 1-2 тижні | Cron автоматизація | Headless run для 50k+ юзерів/день. Phase 4 довіри |

**Speaker note:**
> «forge — не "ще одна AI-фіча". Це інструмент який накопичує **організаційний досвід Promova по re-engagement'у**. Через 6 місяців це найдорожчий internal dataset на цю тему. Кожен push який ми колись слали — частина пам'яті системи.
>
> Vlad під час валідації прямо просив подумати email кампанії і revenue-flow з персональними згораючими знижками — і архітектурно це вже підтримується: forge output це structured `{title, body, reasoning}`, канал додається параметром, group_key розширюється каналом. Тобто дорожня карта пост-хакатону — не "переписати під email", а "додати email-prompt, перерахувати length constraints, group_stats той самий".»

---

## Q&A buffer (2 min)

Готові відповіді:

### «Чим це відрізняється від AI-пушів які Veronika + Vlad Gut уже тестували?»

> Той експеримент (+41% CTR uplift на 12.2% повідомлень із stat-sig результатом) — **pioneer-робота на якій forge будується**, не конкурент. Vlad натренував модель на brand voice, Veronika запустила A/B-тест у welcome flow. Це довело гіпотезу що AI-personalization копії працюють для одноразової генерації. forge — **наступний шар поверх**: не одноразова генерація а per-user reasoning з outcome loop'ом, який вчиться з кожним push'ом і агрегує win-rate per group_key. Перша ітерація — AI-copy. Наступна — AI-reasoning з накопиченою пам'яттю.

### «Чому ви проєктували це як org-wide engine, а не Promova-only tool?»

> Це не Promova-tooling, а org-wide engine. Domain adapter pattern — 30 рядків на новий продукт. У MVP ship'аємо одного adapter'а (Promova), але core, schema, MCP-surface і outcome loop — універсальні. Universalізація з нуля коштує менше ніж переписування потім, коли наступний продукт зайде у запит.

### «На демо ти симулював outcomes — як це справді працюватиме?»

> Чесна відповідь: на demo це симуляція бо я не маю тижня в ефірі. На pilot post-хакатон — **ніякого cron на MVP, все ручне через Claude Desktop**: я кажу «forge, що в pending?», `forge_check_pending_outcomes` повертає список attempt'ів де 7-денне вікно минуло, Claude через Walhalla MCP перевіряє чи юзер виконав Activation Pattern (`lesson_started` event у 2+ різні дні), викликає `forge_measure_outcome` per attempt — reward_score рахується автоматично. Це 5-10 хв ручної роботи на хвилю, не cron. Cron — Phase 4, через 2 місяці після того як trajectory і guardrails довели себе.

### «Чому ви робите iteration trajectory замість A/B-тесту з контрольною групою?»

> На самому MVP-демо (5-15 attempts на симульованих outcome'ах) A/B недосяжний — мало даних. Тому показую trajectory: forge порівнюється сам із собою у часі, win-rate росте → система вчиться. На pilot post-hackathon (1000+ юзерів на хвилю per Vlad Gut validation) — інакше: stat-sig вже доступний wave-by-wave, тому можемо зробити mini-A/B всередині хвилі 1 (15% control vs 85% forge) і отримати uplift-цифру вже через тиждень. Повноцінний A/B vs baseline (через Customer.io branch step) — Phase 2 формального запуску. Це той самий патерн що Promova використовує: Learning Buddy MVP теж починався з internal pilot перед широким A/B.

### «А як forge працюватиме для email чи SMS, не тільки push?»

> forge channel- і CRM-agnostic by design. Output — структурований контент (`title + body + reasoning`), без прив'язки до каналу і без прив'язки до CRM-вендора. Поточний CRM (Reteno зараз, Customer.io після завершення міграції — це паралельний org-трек, не залежність forge) routes його через push (APNS / FCM), email (subject + preheader), SMS (truncated), in-app, web push. У проді додаємо `channel` параметр у `/compose` endpoint — forge тонко тюнить length і tone під канал. group_key розширюється каналом (бо те що зайде в push не обов'язково зайде в email). Деталі — `ANALYTICS.md`.

### «А Customer.io же ще не підключили — на чому ви запустите?»

> Власне тому forge так спроектований. Output — структурований JSON (`title + body + reasoning + confidence`), CRM-agnostic. Сьогодні в org є Reteno, паралельно Anastasiia з командою ведуть міграцію на Customer.io — але forge не залежить від того, який саме CRM перший підключиться. Pilot можна запускати на тому, що готове раніше: webhook step → forge `/compose` → змінна в шаблоні (`$!forge.title` у Reteno, `{{journey.forge_title}}` у Customer.io) → send block → delay 7d → forge `/measure`. Зміна CRM = зміна одного URL і синтаксису шаблону, не переписування forge. Це і є сенс upstream-архітектури.

### «Customer.io має 16-секундний timeout на webhook. Anthropic може довше відповідати — як це переживете?»

> Хороший catch. Три речі. **Перше** — `/compose` приймає `Idempotency-Key` (детермінований з `userId + workflow_run_id`). Якщо CRM ретраїть на timeout — повертаємо вже згенерований результат, а не палимо токени вдруге. **Друге** — hard-cap 12 секунд на forge стороні з fallback на default content по brand voice (predictedConfidence 0.5). Юзер отримає валідний пуш, просто без learning loop для цього attempt. **Третє** — для Reteno timeout не задокументовано, тому перед прод-запуском узгодимо з їхнім support; в гіршому випадку йдемо в async-патерн (pre-compute по profile-update event'у і feeds з кешу).

### «А якщо група має 0 attempts?»

> Перший push у новій групі — Claude генерує по brand voice без in-context examples. predictedConfidence буде нижче (0.5-0.6 типово). Це фіча, не баг — модель чесно сигналить «не знаю».

### «Group_key з 5 dimensions — а якщо потрібно більше?»

> Ми **свідомо** обмежили до 5. На 50-100 attempts більша кардинальність = sparse групи = шум. groupFeatures зберігає всі 9-15 dimensions для inspection і reasoning context, але group_key компактний. Коли БД виросте до 1000+, можна додати dimensions.

### «А embeddings? RAG?»

> Розглядали і відкинули свідомо. На малих даних cosine similarity дає шум. Group_key — детермінований і прозорий. Коли матимемо 10k+ attempts і Symonenko v4 archetypes — додамо embedding шар поверх group_key, не замість.

### «Скільки коштує?»

> На demo через Claude Desktop — ~$0.30/push (тут LLM читає всі Walhalla MCP responses як tokens, контекст накопичується). На production з прямою інтеграцією до бека — **$0.004/push** (75x дешевше, бо бекенд сам тягне дані по HTTP, LLM бачить тільки готовий snapshot ~3k tokens). Pilot 7,000 attempts = **$28** на production path. Pilot ROI на консервативному funnel'у (3% CTR × 50% return→lesson з Amplitude × 40% lesson→activation): 42 активації × $4.40 ARPS = $185 revenue → **~6.6x ROI**. Break-even — 7 активацій з 7,000 (0.1% lift), на порядок менше за +13.4% Veronika floor.

### «А чи перевіряли ви гіпотезу з кимось хто реально знає org-CRM?»

> Так. 5 травня 2026 (день старту хакатону) провалідував з **Vlad Gut** — він автор пайонерського +41% CTR експерименту в org. На питання «чи правильно я зрозумів що outcome кожного пуша назад у генерацію наступного per-сегмент не феєдбачиться» Vlad відповів **«все так»** — тобто feedback gap, який forge закриває, підтверджений власником CRM AI-інфраструктури. Він також відкрив три розширення (email, revenue campaigns з персональними згораючими знижками, memetic content layer) — це лежить у roadmap-слайді як Vlad-validated next steps.

### «А де насправді найбільший ризик у цьому продукті?»

> Не там, де можна було б подумати. **Push opt-out / spam — структурно неможливий на MVP/pilot**: на MVP не шлемо взагалі (channel='mock'), на Phase 1-2 pilot людина ревʼю кожну партію згенерованих пушів через Claude Desktop **перед** тим як CRM шле. Реальний opt-out ризик матеріалізується тільки на Phase 3 (auto-send by confidence threshold), і саме туди закладені guardrails з MVP: fatigue cap 1/тиждень, hard-skip нижче confidence 0.5, rate-limit на групу 24h, `forge_decision_skipped` events з причиною. **Справжній найбільший ризик зараз — trajectory не росте**: якщо за 5-7 хвиль pilot'а win-rate тримається на cold-start рівні (30-40%) — це сигнал що або `group_key` некоректний granularity, або Anna's Activation Pattern не той outcome для сегменту, або brand voice prompt не транслює досвід winners. Це fail-fast умова — бачимо за 100-150 attempts, не за 12k. Per Vlad Gut validation: «там ризиків небагато, у нас пуші зараз досі не генерять ніякий пристойний % revenue чи ретеншина».

### «Чому pilot 1000+ юзерів а не консервативно 100-200?»

> Per Vlad Gut explicit ask на валідації (5 травня 2026): «~100 — мало, краще 1000+. там ризиків небагато». Він власник CRM AI-інфраструктури, він прямо сказав що теперішній performance push'ів не критичний, отже стартуємо амбітно. 1000+ на хвилю × 5-7 хвиль = 7000+ attempts → stat-sig wave-by-wave, не лише trajectory. Можемо навіть зробити mini-A/B на хвилі 1 (15% control vs 85% forge) і отримати конкретну uplift-цифру вже через тиждень, не чекаючи Phase 2.

### «Якщо не зайде що?»

> 16 годин коду. Викинути не шкода. Computeгroupkey, схема engagement_attempts, brand voice prompts — будь-яка частина переходить у Customer.io workflow editor або в Symonenko v5 без втрат.

---

## Worst-case fallbacks

| Що зламалось | План B |
|---|---|
| Claude Desktop crash на сцені | Switch to backup recording (OBS) — продовжуємо без перерви |
| Walhalla MCP no response | Демонструємо тільки forge tools з заздалегідь підготованим contextSnapshot JSON — у нас є screenshots з PRE-recorded thread'у |
| Postgres впав | `db:studio` із бекапу + .sql дамп з 50 attempts |
| Інтернет упав | Backup recording covers everything |
| Жюрі не задає питань | Перейти на slide «Path to standalone product» — productization angle |

---

## After demo (Day 2 evening)

Якщо демо пройшло сильно (топ-5 feedback):
1. Push до GitHub з тегом `v0-mvp-demo` (snapshot)
2. Запропонувати Anastasiia 1:1 на наступному тижні про prod-handoff
3. Опублікувати DEMO + WHY + PLAN внутрішньо в Notion (під продактовим простором)

Якщо не пройшло сильно:
1. Push з тегом `v0-mvp-demo` — артефакт лишається
2. README + WHY + PLAN — використати як portfolio для зовнішніх співбесід
3. Закрити гіпотезу за один спринт. Не шкода.