# Forge — текст прези (10 хв)

7 хв презентація + 3 хв Q&A.
**Жирним** — що кажеш голосом. *Курсивом* — що робиш руками.

---

## Слайд 01 · FORGE opener (25 сек)

*Виходиш на сцену. Пауза 2 секунди. Дивишся в зал.*

> «Я хочу попросити вашої уваги.
>
> *(пауза)*
>
> Ми давно про це говорили. У трьох різних командах, у чотирьох різних
> квартальних планах, на кожному рев'ю звучало одне й те саме питання:
> **а що, як пуші почнуть вчитися самі?**
>
> *(пауза)*
>
> Сьогодні я це покажу.
>
> Це **Forge**. Лог → outcome → накопичений досвід → точніший наступний
> удар. Сім хвилин — і ви побачите, як це працює наживо.»

*Натискаєш next.*

---

## Слайд 02 · Three facts (60 сек)

> «У продукті є три факти, які разом окреслюють одну прогалину.
>
> **П'ятнадцять відсотків** неактивних юзерів узагалі невидимі для CRM.
> У них ad-blocker'и, Амплітуда мовчить, але платежі йдуть. Це silent
> churn.
>
> **Плюс 33% CTR** ми вже отримали в org на personalized welcome
> push'ах проти broadcast. Mini-wins дали +86% start уроку.
> **Персоналізація працює — це не гіпотеза.**
>
> Третій факт — **нуль**. Нуль систем у org що feedback'ять outcome
> назад у генерацію наступного push'а per-сегмент. Дані є, voice є,
> AI-генерація вже довела +41% CTR на одному з експериментів. Але
> engine з пам'яттю — немає.
>
> **Місце для системи, яка це закриває — порожнє.**»

---

## Слайд 03 · Cited evidence (40 сек)

> «Усі цифри — внутрішні Promova, нічого з benchmark'ів галузі.
>
> Плюс 41% CTR — AI-generated push на brand voice, stat-sig на 12%
> повідомлень. Пайонер-експеримент в org — це floor, не stretch.
>
> Плюс 33% — статична персоналізація welcome flow.
>
> Плюс 58% Trial→Payment — Learning Buddy MVP, iOS A/B 10 днів,
> roll-out цього тижня. Persona на behavioral data **уже працює** в апі.
>
> Кореляція 0.8 — нова Activation Rate з W4 retention. **Наш north
> star.**
>
> Я не вигадую гіпотезу. Я будую систему поверх цифр, які в org вже
> доведені.»

---

## Слайд 04 · Hypothesis · metric · validation (45 сек)

> «Гіпотеза одна: персоналізований push з reasoning'ом дасть вищий
> win-rate ніж broadcast — і **win-rate буде зростати з хвилями**, бо
> система вчиться на власних outcome'ах.
>
> Метрика — Activation Pattern Hit-rate. Один-плюс урок у двох-плюс днях.
> Кореляція 0.8 з W4 retention — оптимізуючи за цей таргет, оптимізуємо
> retention напряму.
>
> Validation: feedback gap визнано автором +41% CTR пайонер-експерименту
> в org. **Гіпотеза не доведена в проді — пілот це і є тест.**
>
> Зараз даних немає — будуємо механіку. У пілоті 5–7 хвиль по 1000+
> юзерів кожна. Якщо trajectory росте — система вчиться. Якщо плато —
> **fail-fast за 100–150 спроб**, не за 12 тисяч.»

---

## Слайд 05 · Не всі AI-pushes однакові (60 сек) — критичний слайд

*Це слайд, який знімає головне продуктове заперечення.*

> «Передбачаю питання: 'Ми вже запускали AI-push на Reteno — не дав
> апліфту. Чим це відрізняється?'
>
> Дивіться на колонки.
>
> **Reteno «One from many»** — генерує **варіанти одного шаблону** і
> скейлить топ-performer по CTR. Шаблон має писати людина. Reteno не
> знає про рівні, мови, lesson topics — він generic. Тому AI-push на
> Reteno не дав апліфту: він оптимізував не той сигнал, без
> behavioral-контексту, на рівні кампанії.
>
> **Customer.io AI** — Agent плюс LLM Actions. Теж template helpers:
> чернетки повідомлень, підказки по сегментах, send time. Generic,
> замкнений всередині Customer.io.
>
> **Forge — це інша категорія. Reasoning-шар над outcome'ами.** Бачить
> усі 5 dimensions поведінки юзера: мова, рівень, payment status,
> дні неактивності, остання тема уроку. Тримає win-rate **per
> group_key**. Цитує конкретних winners з cohort'и в reasoning'у.
> Output — JSON, працює з будь-яким CRM. Оптимізує **Activation
> Pattern**, не CTR.
>
> *(пауза)*
>
> **AI ≠ AI. Forge не template generator — це reasoning over outcomes.
> Forge не замінює CRM — це один HTTP call upstream від webhook step.**»

---

## Слайд 06 · Як це працює (45 сек)

> «Тепер як саме — людською мовою, у п'ять кроків.
>
> **Один — збираємо контекст** про юзера: хто він, що вивчав, давно не
> заходив, платить чи ні, з яких пристроїв.
>
> **Два — питаємо пам'ять**: «ось такі ж юзери. На цих текстах вони
> повернулись. На цих — мовчали».
>
> **Три — пишемо push**. Claude бачить winners і losers і складає
> текст. Reasoning поруч пояснює — чому саме такий.
>
> **Чотири — логуємо все**: сам текст, контекст, reasoning, токени,
> ціна. Видно в адмінці у proof panel.
>
> **П'ять — через 7 днів вимір**. Юзер виконав Activation Pattern? Так —
> winner, додаємо в пам'ять. Ні — loser, теж в пам'ять.
>
> Внизу — **як ми бакетизуємо когорти**. Мова, рівень, платить чи ні,
> дні неактивності, остання тема. **П'ять вимірів — і кожен push
> порівнюється з минулими у тій самій когорті.**»

*Перехід на live demo.*

---

## LIVE DEMO (3 хв) — Claude Desktop + admin

*Виходиш зі слайдів у Claude Desktop. Поряд — admin на localhost:5174.*

**Beat 1 (40 сек).** Кажеш Claude'у: «знайди inactive es B1 user, 7-14 днів
неактивний». Claude робить 4-5 викликів Walhalla MCP.
> «Forge не ходить у Walhalla напряму — це робить Claude Desktop. Forge
> збирає тільки snapshot з результатів.»

**Beat 2 (40 сек).** «Подивись group_stats перед тим як писати push».
`forge_get_group_stats` → winners/losers.
> «Це пам'ять forge. Видно тексти-переможці і тексти-лузери з cohort'и.
> Claude буде на них спиратись.»

**Beat 3 (60 сек).** «Згенери push, посилайся на winners». Claude видає
reasoning з прямою цитатою winner'ів.
> «Reasoning не вигаданий — посилається на конкретні приклади з cohort'и.
> Confidence — функція щільності group_key, не випадкове число.»

**Beat 4 (40 сек).** «Залогуй через `forge_log_attempt`,
`simulateOutcomeNow: true`». Переключаєшся в admin.
> «Я зараз руками призначаю outcome — це **симуляція для демо**, щоб
> побачити повний цикл за хвилину. У проді reward_score рахується
> автоматично через 7 днів з `lesson_started` event у Walhalla.»
- stat cards оновились
- щойно з'явився новий attempt
- proof panel з reasoning'ом

**Beat 5 (20 сек).**
> «Один цикл — discover, group memory, compose, log, outcome. Усе
> автоматично логується. **Це і є closed feedback loop.**»

*Назад у слайди.*

---

## Слайд 07 · Trade-off (45 сек)

> «Найскладніший trade-off — економіка.
>
> На демо — **30 центів за push**. Claude Desktop читає кожну Walhalla
> response як токени, контекст накопичується. Це dev tool.
>
> У проді — **менше пів цента, конкретно 0.4**. Бекенд тягне дані по
> HTTP, LLM бачить тільки snapshot. **75 разів дешевше.** Pilot на 7
> тисяч атемптів — 28 доларів.
>
> Чому MCP на демо: нуль backend-інтеграцій, нуль API-ключів. Прод —
> той самий MCP server, прямий SDK caller замість Claude Desktop.»

---

## Слайд 08 · Admin: інспекція reasoning'у (45 сек)

> «Адмінка — це наш analytics surface. **І на демо, і у проді. Amplitude
> не потрібен.**
>
> Три речі, які вона показує.
>
> Перше — **stat cards**: total sent, reactivated, silent, pending,
> win-rate, cost per reactivation. Операційний health у real-time.
>
> Друге — **per-attempt proof panel**. Відкрив будь-який push — бачиш
> reasoning chain з цитованими winners з cohort'и, повідомлення,
> group features, verdict, reward. **Ніколи не «модель так вирішила».
> Завжди — чому саме так.**
>
> Третє — **win-rate by group_key**. Видно яка комбінація
> lang × level × payment × inactive працює, а яка — ні.
>
> Чому не Amplitude — Amplitude чудово агрегує події, але для
> інспекції reasoning chain потрібна форма яку він не дає. Адмінка
> дає **reasoning і stats разом**, в одному місці.»

---

## Слайд 09 · Економіка пілоту (55 сек)

> «Конкретні числа пілоту, **консервативний** funnel.
>
> 7000 пушів. CTR 3% — нижче за звичайний broadcast. 210 кліків.
>
> Із цих кліків 50% повертаються в апку і проходять урок — baseline з
> Амплітуди. 105 уроків.
>
> 40% хіттять Activation Pattern. 42 активації.
>
> $4.40 ARPS-дельти на активацію. **$185 виручки на $28 витрат. ROI 6.6x.**
>
> Break-even — **0.1% activation lift**. Сім активацій з семи тисяч
> пушів. На два порядки нижче за floor, який ми вже бʼємо — +13%.
>
> Sensitivity: на 1% CTR — worst plausible — ROI все одно 2.2x.
>
> **І важливе зверху:** Forge вже вміє вкладати **персональний billing
> offer + deep link** у push або email. Покупка прив'язана до конкретного
> push'а через UTM. Це **окрема ROI-доріжка** поверх activation funnel —
> не врахована в цифрах вище. Це і є відповідь на «як це повертається в
> підписки».»

---

## Слайд 10 · To-prod · biggest risk · phases · next steps (60 сек)

> «Що треба для першого live-пуша. **Два тижні. Чотири файли.**
>
> ingestion — HTTP pull замість MCP. composer — прямий SDK call. cron
> плюс два HTTP endpoints — `/compose` і `/measure`. webhook step у CRM.
>
> Path to autonomy — чотири фази довіри. **Human-gated send** на
> першій тисячі юзерів. Потім **A/B vs baseline** через CRM branch.
> Далі **auto-send за confidence threshold** з guardrails — fatigue
> cap, hard-skip, rate-limit. І нарешті **cron з SLO на cancel rate**.
> Чутливі домени — mental health, dating — застряють на Phase 2 назавжди.
> Це частина дизайну.
>
> **Найбільший ризик не там, де думають.** Не opt-out spam — він
> структурно неможливий на Phase 1-2, людина в loop'і. **Найбільший
> ризик — trajectory не росте.** Якщо за 5-7 хвиль win-rate тримається
> на cold-start — або group_key неправильний, або Activation Pattern не
> той outcome, або brand voice не транслює досвід winners. **Fail-fast
> за 100-150 спроб. Не за 12 тисяч.**
>
> Validated next steps: email канал, revenue campaigns з персональними
> згораючими знижками, Symonenko archetypes через BigQuery, A/B vs
> baseline у Phase 2.
>
> *(пауза)*
>
> **Forge — це не AI-фіча. Це інструмент, що накопичує організаційний
> досвід Promova у re-engagement'і.** Через 6 місяців це буде
> найцінніший внутрішній dataset Promova по цій темі. Кожен push,
> який ми колись надішлемо, стає частиною памʼяті системи.
>
> *(пауза)*
>
> Дякую. Питання.»

---

## Q&A — короткі відповіді (3 хв)

### «AI пуші на Reteno НЕ давали апліфту — чим Forge відрізняється?»
Reteno «One from many» оптимізує **CTR per кампанія**, генерує варіанти **одного шаблона**, не має behavioral context. Forge — інша категорія: per-user reasoning з пам'яттю по group_key (5-dim cohort), оптимізує Activation Pattern (corr 0.8 з W4 retention), не CTR. Якщо trajectory не росте за 5-7 хвиль — fail-fast за 100 спроб.

### «А це ж можна в самому Customer.io / Reteno зробити?»
Customer.io AI Agent + LLM Actions — це **template helpers** (drafts, suggest segments, send time). Reteno «One from many» — варіанти одного шаблону. Жоден з них не: накопичує win-rate per 5-dim cohort, не робить in-context learning з minulых winners тієї ж когорти, не калібрує confidence по щільності даних, не оптимізує Activation Pattern. Forge — **reasoning layer upstream** від CRM. Output → JSON → CRM.

### «Покажи одним реченням capability що Forge має, а workflow CIO з journey-attribute + LLM Action — не має.»
Customer.io LLM Action генерує текст з контексту цього юзера. Forge — **query до пам'яті 5-dim cohort'и до генерації**, з cited winners у reasoning chain і калібрацією confidence по щільності даних. Це не template+LLM, це reasoning over outcomes.

### «Чому окремий продукт + адмінка — не дублює функціонал CRM?»
Адмінка не для маркетолога — для **інспекції**. Видно **чому** модель видала той текст (reasoning chain з посиланням на winners), як trajectory росте за iter'ами, де модель невпевнена. У жодному CRM-у немає механіки «покажи чим iter 3 reasoning відрізняється від iter 1 для тієї самої групи».

### «Скільки коштує?»
Демо ~30 центів/push (Claude Desktop overhead). Прод ~0.4 цента/push (бекенд → SDK). Пілот 7k = $28. ROI 6.6x на 3% CTR. На 1% CTR (worst) — 2.2x. Break-even 0.1% lift.

### «А як ти прив'язуєш покупку до push'а?»
Forge генерує deep link з UTM (`utm_source=forge`, `attempt_id=<ID>`). Юзер натискає → попадає на paywall з персональним offer'ом → купує → ми бачимо який саме push приніс підписку. **Built-in revenue attribution.** Per-user план обирається автоматично за paymentStatus + completed lessons + days inactive (churned-paying → recovery offer, free → trial; paying — нічого, він уже платить). У ROI-цифри пілоту вище це **не враховано** — це окрема доріжка зверху.

### «Як перевіримо що працює?»
Trajectory як primary signal. 1000+ юзерів кожна хвиля. Якщо не росте за 5-7 хвиль — fail-fast. Final — Activation Pattern Hit-rate, корелює 0.8 з W4 retention.

### «Опт-аут spam ризик?»
Структурно неможливий на Phase 1-2 — людина ревʼю кожну партію перед send. На Phase 3 guardrails з MVP: fatigue cap, hard-skip <0.5, rate-limit 24h, decision_skipped events.

### «Org-wide engine — а чому не Promova-only?»
Domain adapter pattern — 30 рядків на новий продукт. Universalізація з нуля коштує менше ніж переписування потім, коли наступний продукт зайде.

### «Закохався у рішення, не в проблему?»
Рішення виросло з даних, не навпаки. Я провалідував з автором +41% CTR експерименту в org — feedback gap підтверджений: outcome кожного push не повертається в генерацію наступного per-сегмент. Це **проблема**. Якщо була б вирішена в Reteno — там був би trajectory growth по cohort'ах. Його немає, бо Reteno генерує варіанти одного шаблону, не reasoning'ить per-user з пам'яттю.

### «Це ж симуляція outcome'ів — як ти знаєш що Claude використовував iter 1 winners для iter 2?»
Чесна відповідь: на демо я руками призначаю outcome, щоб показати механіку за хвилину. У пілоті outcome рахується автоматично з `lesson_started` event у Walhalla — без людини. Те що Claude дійсно використовує winners видно в reasoning chain — кожен push має пряму цитату з examples. Це інспекційно перевіряється в адмінці.

### «Якщо група має 0 attempts?»
Cold start. Claude генерить по brand voice, confidence нижче (0.5–0.6). **Це фіча — модель чесно сигналить що не знає.**

### «Якщо не зайде що?»
**16 годин коду. Викинути не шкода.** Group key, brand voice prompts, schema — будь-яка частина переходить у CRM workflow editor або наступний продукт без втрат.

---

## Pre-demo checklist (за 30 хв)

1. `yarn docker:up` — Postgres готовий
2. `forge_ping` у Claude Desktop повертає OK
3. БД сидована — admin показує >10 attempts на >5 cohort'ах
4. Для seed user — group_key має ≥10 attempts (інакше cold-start ламає Beat 2)
5. Admin запущений: `localhost:5174` — attempts list завантажується
6. Backup recording started (OBS)
7. Слайди відкриті, slide 01 на екрані
8. Pre-picked seed user_id у нотатках (на випадок Walhalla timeout)
9. Wipe attempts table для seed user (24h dedup може хіт)
10. Вода поряд

## Worst-case fallbacks

| Що зламалось | План B |
|---|---|
| Claude Desktop завис >20с | «Покажу запис того ж сценарію» → OBS clip |
| Walhalla MCP timeout | «Ось user якого я витяг учора» → pre-seeded snapshot |
| 24h dedup hits | wipe attempts → retry |
| Cold-start group | «Це cold-start — модель чесно сигналить» (передбачено) |
| Демо за 7хв вилазить | Скіп деталі slide 03 (cited evidence) — назвати тільки 2 цифри |
