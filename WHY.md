# Forge — навіщо і для кого

Документ пояснює **бізнес-проблему**, яку forge розв'язує на хакатоні, спираючись на реальні внутрішні дані Promova (Slack-обговорення, A/B-тести, аналітика). Цифри — для слайдів, цитати — для credibility.

---

## TL;DR

Promova знає в цифрах: **персоналізація пушів працює**, **нова Activation Rate — кращий предиктор retention**, **Learning Buddy MVP підтвердив +58.8% Trial→Payment через персональні рекомендації**. Паралельно йде **міграція CRM на Customer.io**, і інструмент для оркестрації workflow'ів лишається **«переважно ручним»** (цитата). Forge — відсутній шар: per-user reasoning engine із замкнутим outcome-loop'ом, без якого всі зібрані сигнали (Symonenko clustering, voice signals, retention metrics) не перетворюються на *персоналізовані* пуші, а перетворюються на broadcast-копії.

---

## 1. Проблема: тихий відтік

У Promova **15% когорти неактивних юзерів взагалі не можна відстежити** в Amplitude — у них ad-blocker'и, події не приходять, але покупки приходять. Ці люди буквально невидимі для поточних CRM-флоу.

> *«Близько 15% з когорти цих неактивних юзерів мають якусь варіацію ad block, через це ми ніяк не можемо відстежити їхню поведінку — в Амплітуді по них просто пусто, хоча покупки приходять.»*
> — Anna Ziubanova, #analytics, 4 вересня 2025

Це **silent churn** у чистому вигляді. Юзери платять, юзери відвалюються, але поточні реакції (звичайні повторювані broadcast-пуші) до них не доходять — бо під фільтр когорти немає даних.

forge не лікує сам ad-block (це інфра-задача), але ставить правильний фрейм: **рішення не повинно покладатися на «середні» розсилки**. Per-user reasoning + outcome-loop у довгу закриває розрив, тому що:
- вчиться на тих юзерах, по кому **є** outcome (платежі, повторний захід)
- статистика win-rate по групах акуратно деградує на «темні» когорти, не ламаючи загальний приплив

---

## 2. Персоналізація пушів дає цифри — це вже доведено

Veronika Velykanova провела 3 тести через CRM у рамках Symonenko-проекту. Зафіксовані результати ключових:

### Тест 1 — Персонализация welcome-flow + чёткий CTA (8 вересня 2025)

Замість загального заклику «подивись відеоуроки» юзер отримував push з конкретним уроком, підібраним під його рівень (A1–C1):

| Метрика | Контроль | Тест | Δ |
|---|---|---|---|
| CTR | 1.81% | 2.40% | **+32.6%** |
| Конверсія в старт уроку | 1.72% | 1.95% | **+13.4%** |

> *«Висновок: персоналізація контенту та чіткий CTA суттєво підвищують ефективність early-stage активації.»*
> — Veronika Velykanova, #symonenko_scale

### Тест 2 — Mini wins as motivation driver

Юзерам, які не завершили жодного юніта, in-app з акцентом на маленьких досягненнях. Наприклад: «Пройди всього 3 уроки — і вже будеш готовий до візиту до лікаря».

| Метрика | Контроль | Тест | Δ |
|---|---|---|---|
| Відвідуваність платформи | 1.19% | 1.92% | **+61.3%** |
| Конверсія в старт уроку | 0.77% | 1.43% | **+85.7%** |

> *«Мікродосягнення працюють, виходячи з CTR — краще для B1–C1 сегментів.»*

**Що це значить для forge:** персоналізація по сегменту (B1–C1, level + topic + last lesson) дає дворазовий апсайд. Це і є саме те, що forge'ові `group_key` бакетизують автоматично.

---

## 3. New Activation Rate — єдина метрика якою варто міряти «ожив юзер чи ні»

Команда Anny Зюбанової в березні 2026 переглянула визначення Activation:

> **Старе:** юзер вважався активованим, якщо пройшов 5+ уроків за перші 3 дні.
> **Нове:** юзер вважається активованим, якщо протягом перших 7 днів після реєстрації він пройшов хоча б по 1 уроку в щонайменше 2 різні дні.

Зміна тривіальна виглядом, але числа разючі.

| Метрика | Активовані | Неактивовані |
|---|---|---|
| Retention Rate 4W | **22.7%** | **4.2%** |
| Trial → Payment | 33.4% | 21.3% |
| Cancel Rate 30d | 50.2% | 56.1% |
| ARPS 7d | $28.6 | $21.8 |
| ARPS 13m | $48.7 | $44.3 |

> *«Регулярність повернень важливіша за інтенсивність одного сеансу. Нова метрика корелює з Retention Rate 4W на 0.8, тоді як стара — 0.58.»*
> — Anna Ziubanova, #analytics, 18 березня 2026

**Що це значить для forge:**
- Outcome reward score має міряти **саме цю активацію** («1+ урок у 2+ дні»), а не «повернувся хоч раз»
- Це вже зашито в `MeasurerService.recordOutcome` через прапор `activationPatternHit`
- Кореляція 0.8 значить: оптимізуючи за цей target у forge, оптимізуємо **довгостроковий retention напряму**

---

## 4. Персоналізація-at-scale вже працює: Learning Buddy MVP

iOS A/B-тест 10–19 квітня 2026. Маскот у вкладці My Plan з рекомендаціями невикористаних premium-фіч (на основі usage history).

| Метрика | Control | Test | Δ |
|---|---|---|---|
| **Trial → Payment** | **14.3%** | **22.7%** | **+58.8%** *(stat sig)* |
| Cancel Rate | 48.8% | 42.4% | **−13.0%** |
| ARPU 7D | $0.25 | $0.30 | **+20.2%** |
| Expected ARPU 13M | $0.34 | $0.43 | **+24.9%** |
| Retention 1D / 7D | 15.3% / 11.7% | 14.7% / 11.1% | −4% / −4.8% (guardrail) |

**Найсильніший ефект** у Level Group A: Start→Purchase +31.1%, Trial→Payment **+170.8%** (stat sig).

> *«Декіcія: Win — Roll Out.»*
> — Timothy Kachan, #analytics, 1 травня 2026

Banner показано лише ~27% юзерів у тест-групі (через тригер «відкриття My Plan» + ліміти показу). Тобто реальний effect size на тих, хто бачив, ще сильніший — а агреговані числа уже Win.

**Що це значить для forge:**
- Персоналізація на поведінкових даних — **підтверджений механізм** в Promova
- Learning Buddy робить це для активних юзерів усередині апи. Forge робить те ж саме **для тих хто вже пішов** — у пушах.
- Якщо Learning Buddy дав +58.8% Trial→Payment **усередині апи**, очікування для re-engagement пушів — як мінімум +30% CTR (про який вже говорить Veronika в Тесті 1)

---

## 5. Infrastructure migration створює вікно для forge

Anastasiia Stelmakh веде міграцію Reteno → Customer.io. Стейтус на травень 2026:
- ✅ Backend перенесений, події летять у Customer.io
- ✅ Документація з аналітичних івентів готова
- ⚠ Workflow setup у Customer.io — **«переважно ручний/напівмануальний»**

> *«Workflow налаштування буде переважно ручним/напівмануальним, але розглядаємо варіанти як можна пришвидшити та автоматизувати тут процес.»*
> — Anastasiia Stelmakh, #product_dev_team

Це прямо відкриває нішу для forge:
- старий движок (Reteno) — broadcast + статичні шаблони
- новий движок (Customer.io) — workflow editor з ручним налаштуванням
- **forge** — LLM-based decision engine, який видає готовий push + reasoning + outcome tracking. Інтеграція з Customer.io transactional API на проді = одна функція

forge **не конкурує** з Customer.io — він стає його **upstream brain**: «який саме push для якого саме юзера через Customer.io надіслати, з яким reasoning».

---

## 6. Сировина для контекста вже є — Symonenko v4

Ivan Kotiuchyi та Yelyzaveta Dorozhko в #symonenko_scale прогнали clustering pipeline (LLM + PCA + KMeans) на 1M юзерах. Один із побічних артефактів — **rich text-опис кожного юзера**, наприклад:

```
"English learner from Spanish language, at A2 elementary level. Location:
Mexico. Prefers 5 minutes daily sessions. Uses Android system. Comes from
organic source. Learning goals include advancing their career. Focuses on
improving speaking, grammar, vocabulary, and reading. Specifically: 2
sessions, 4 lessons, 75% completion rate. Engagement score: 6. Risk
profile: no apparent risk."
```

Ці описи генеруються з ~25 features:
- Engagement Score, Monetization Intent, Learning Diversity (continuous)
- User Archetype (8 категорій, включно з «Inactive»)
- Engagement Tier (T0_Inactive…T5_Elite)
- Monetization Stage (S1_Unaware…S6_Converted)
- Churn Risk Flag («already_churned», «single_session_risk», «high_abandonment»…)

**Що це значить для forge:**
- На прод-міграції Symonenko-описи **літерально стають** `contextSnapshot.profile`
- forge'ові `group_key` рахуються з тих самих features — обидві системи в одній онтології
- Учаснику не треба придумувати «що таке inactive юзер» — Symonenko вже відповів: `archetype = "Inactive"` OR `engagement_tier = "T0_Inactive"` OR `churn_risk_flag in (already_churned, single_session_risk)`

---

## 7. Чому **зараз** — чому хакатон

Чотири вектори зійшлися в одній точці:

1. **Цифри**: персоналізація +32–86% по тестам Veronika; Learning Buddy +58.8% Trial→Payment
2. **Метрика готова**: Anna Z завершила нову Activation Rate з кореляцією 0.8 до W4 retention
3. **Інфра**: Customer.io migration готовий до того щоб вмикати нові decision-engine'и (backend done, workflow setup manual)
4. **Сировина**: Symonenko v4 clustering вже генерує rich user descriptions

**Не зробимо forge зараз — їх з’єднає хтось інший, або не з’єднає ніхто, або зроблять руками в Customer.io workflow editor (що, як сказала Anastasiia, ручне).**

Хакатон — точка з мінімальним ризиком: 2 дні на MVP, demo на Claude Desktop, нуль API ключів, нуль продакшн-інтеграцій. Якщо зайде — ще 2 тижні на ingestion/composer/cron і live прод. Якщо не зайде — закрили гіпотезу за один спринт.

---

## 8. Що forge вирішує конкретно

| Проблема | Як forge закриває | Метрика успіху на демо |
|---|---|---|
| Broadcast пуші не персоналізовані | per-user reasoning з in-context learning по `group_key` | reasoning у чаті, group stats у БД |
| Outcome не вимірюється — push'нули і забули | замкнутий loop: log_attempt → measure_outcome → reward_score | rewardScore у БД після `simulate_outcomes` |
| Накопичений досвід не перевикористовується | агрегація winners/losers за group_key | `forge_get_group_stats` повертає win-rate |
| Персональний context розкиданий по 4 системах | один `contextSnapshot` зі всіх MCPs (Walhalla + Amplitude + Zendesk) | Claude складає JSON у чаті |
| Customer.io workflow setup ручний | forge видає готовий push + payload для transactional API | mock channel = 'mock' у БД, але payload готовий |

---

## 9. Чого forge **НЕ** вирішує — і це OK

- ❌ **Real delivery** через push (Reteno/Customer.io). На MVP мокається. На проді — одна функція.
- ❌ **Cron / автоматизація**. На MVP human-in-the-loop через Claude Desktop. У плану path-to-autonomy 4 фази довіри.
- ❌ **Tracking ad-block юзерів** (про яких писала Anna Z). Це інфра-задача, не decision-engine.
- ❌ **Multi-tenant SaaS** з онбордингом ззовні. На MVP — один хардкоджений домен (Promova). DomainAdapter pattern дозволяє додавати продукт через ~30 рядків коду; якщо engine зайде — пересядемо на tenants registry.
- ❌ **Брендований tone-of-voice editor** (як Cowork-pipeline у Dmytro Palaniichuk). Brand-voice prompts захардкоджені у репо.

---

## 10. Зв'язок з організаційними OKR

| OKR / ініціатива | Власник | Як forge долучається |
|---|---|---|
| Marketing Q2 OKR — push CTR через faster iterations | Veronika Velykanova | per-user reasoning + outcome loop = справжня ітерація, а не A/B на копії |
| New Activation Rate як north star | Anna Ziubanova | reward_score формула меряє саме activationPatternHit (1+ урок у 2+ дні) |
| Reteno → Customer.io migration | Anastasiia Stelmakh | forge готовий стати upstream brain для Customer.io transactional API (одна функція delivery) |
| Symonenko v4 clustering | Ivan Kotiuchyi, Yelyzaveta Dorozhko | rich user descriptions = `contextSnapshot.profile` на проді |
| Learning Buddy MVP rollout | Timothy Kachan | той самий принцип «персональна рекомендація → +58% Trial→Payment», але для re-engagement, не in-app |
| Наступний продукт на тому ж двигуні | відповідна команда | той самий движок, інший adapter — без переписування core/schema |

---

## 11. Питання, на які forge має відповісти на демо

1. **«Чим це відрізняється від AI-пушів які Veronika + Vlad Gut уже тестували?»**
   Veronika Velykanova + Vlad Gut раніше провели A/B-тест AI-generated welcome push'ів — отримали **+41% CTR uplift на 12.2% повідомлень** зі stat-sig результатом (Vlad натренував модель на brand voice, Veronika запустила тест). Це **pioneer-робота на якій forge будується**, не конкурент. Той експеримент довів що AI-personalization копії працюють для одноразової генерації. forge — **наступний шар поверх**: не одноразова генерація, а per-user reasoning з outcome loop'ом, який вчиться з кожним push'ом і агрегує win-rate per group_key. Перша ітерація — AI-copy. Наступна — AI-reasoning з накопиченою пам'яттю.

2. **«Чому ви проєктували це як org-wide engine, а не Promova-only tool?»**
   Це не Promova-tooling, а **org-wide engine**. У MVP ship'аємо одного adapter'а (Promova), але core/schema/MCP-surface/outcome loop універсальні. Наступний продукт додається як ~30 рядків DomainAdapter + voice prompt'и — без переписування core. Якщо робити Promova-only зараз і універсалізувати потім — це переписування на проді під тиском, чого ми хочемо уникнути.

3. **«Як перевіримо що працює?»**
   Чотири фази довіри: manual review → manual approval → confidence threshold → cron. На демо це окремий слайд — ми **знаємо** що ризик відправити поганий push для mental-health юзера значно вищий за ризик не відправити нічого.

4. **«Як підключити в production?»**
   forge-as-MCP уже працює. Прод = той же код, інший caller. Cron job викликає Anthropic API → forge MCP виконує tools → результат йде через Customer.io transactional API. Migration — додаємо 3 файли (`ingestion`, `composer`, `engagement-orchestrator`), не переписуємо.

5. **«Скільки коштує?»**
   На демо видно: 1500–3000 input + 300–600 output tokens на юзера через Claude Desktop. Сирий API equivalent ~$0.015 на рішення. 1000 юзерів/день = $15/день. З time дешевше через caching prompts або дорожче через сильнішу модель — кожен tool повертає `tokensInput/tokensOutput/costUsd`, видно одразу.

---

## 12. Цитати для слайдів (ready-to-paste)

> «Близько 15% з когорти цих неактивних юзерів мають якусь варіацію ad block — в Амплітуді по них просто пусто, хоча покупки приходять.»
> — Anna Ziubanova, #analytics, 4 вересня 2025

> «Персоналізація + чіткий CTA: CTR +32.6%, конверсія в старт уроку +13.4%. Mini wins: відвідуваність +61.3%, конверсія в старт +85.7%.»
> — Veronika Velykanova, #symonenko_scale, 8 вересня 2025

> «Нова Activation Rate корелює з Retention 4W на 0.8, стара — 0.58. Активовані юзери: Retention 4W 22.7% проти 4.2%, ARPS 13m $48.7 проти $44.3.»
> — Anna Ziubanova, #analytics, 18 березня 2026

> «Workflow налаштування в Customer.io переважно ручне/напівмануальне, розглядаємо варіанти автоматизації.»
> — Anastasiia Stelmakh, #product_dev_team, 26 лютого 2026

> «[iOS] Learning Buddy MVP: Trial→Payment 14.3% → 22.7% (+58.8%). Cancel Rate −13%. ARPU 13M +24.9%. Decision: Win — Roll Out.»
> — Timothy Kachan, #analytics, 1 травня 2026

---

## 13. Бібліографія

Усі цитати — з внутрішнього Slack workspace `promova-hq.slack.com`. Тред-permalink'и доступні в історії пошуку, для PR-репорту вкажемо посилання.

| Канал | Тема | Дата |
|---|---|---|
| #analytics | Inactive users + ad-block coverage | 2025-09-04 |
| #symonenko_scale | CRM personalization tests (welcome / mini wins / monetization) | 2025-09-08 |
| #symonenko_scale | Symonenko v4 clustering pipeline (LLM + PCA + KMeans) | 2025-12-03 |
| #product_dev_team | Reteno → Customer.io migration timeline | 2026-02-26 |
| #analytics | New Activation Rate definition + correlation analysis | 2026-03-18 |
| #release_prod | Learning Buddy MVP launch | 2026-04-10 |
| #analytics | Learning Buddy MVP results — Win, Roll Out | 2026-05-01 |

---

*Документ підготовлений 2026-05-04, день 0 хакатону. Цифри й цитати — на момент підготовки. Якщо щось зміниться під час хакатону, оновити перед демо.*
