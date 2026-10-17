# OrderFlow: Architecture Analysis

OrderFlow лишається modular monolith. Ми не переписуємо систему й не ділимо її одразу на п'ять сервісів. Ми робимо межі модулів явними, прибираємо найдорожчі залежності (Payments → Orders, синхронний email у checkout) і виділяємо окремий deployable лише тоді, коли метрики це підтвердять.

| # | Знахідка | Evidence | Наслідок для рішення |
| --- | --- | --- | --- |
| 1 | Збій email **після** capture платежу переводить order у `FAILED`, повертає stock, але **payment лишається `CAPTURED`** (`CheckoutService`: `await sendOrderConfirmation` всередині `try`) | [code] + [measured]: з 20 000 checkout (дефолтний config) 1,92% (384) «списано, але order FAILED» | Синхронний email це не лише latency, а **defect коректності на revenue path**. Це ризик №1, а не coupling |
| 2 | Повторний `POST /checkout` (retry клієнта) створює новий order і **другий charge**. Idempotency є лише в `reserve` по `order.id` | [measured]: 2 запити → 2 CAPTURED, stock 10 | Idempotency потрібна на вході Checkout, не лише в Inventory |
| 3 | `reserve` після `release` для того ж order повертає старий `RESERVED`, stock не резервується | [measured]: stock 12 після «reserve» | Дірка в Scenario D при retry/compensation |
| 4 | Orders рахує total, читаючи `database.products` (дані Inventory) через `shared/money.js`. Order не зберігає unit price | [code] + [measured]: після зміни ціни `productPerformance` 8 900 → 9 900 при незмінному total order | Price/discount не мають власника. Це головна перешкода для Scenario F |
| 5 | Payments читає `OrderRepository` напряму | [code] | Підтверджує driver №1 |
| 6 | Reporting сканує `orders × payments × notifications` в одному event loop: O(N²) | [measured]: 5k orders = 0,2 с, 20k = 5,0 с, 50k = 36,5 с блокування. З індексом по `orderId`: 11 мс на 20k. [doc]: Reporting traffic уже виріс приблизно втричі за рік | Гіпотеза «Reporting б'є по checkout» підтверджена, але **дешевше лікується алгоритмом, не окремим service** |
| 7 | `chargeOrder` сканує всі payments на кожен charge | [measured]: CPU на checkout 0,13 → 0,39 мс при ~19k payments | Малий, але лінійно зростаючий податок |
| 8 | Provider викликається без timeout; є check-then-charge без блокування по `orderId` | [code] | Для Scenario B (fallback, async confirmation) потрібні стани платежу, інакше можливий double charge |
| 9 | p95 checkout: 2 539 мс із sync email, 2 244 мс лише з payment | [measured]. Config: `slowRate` = 10%, 1,5–3 с | Email додає ≈300 мс до p95. **Основний хвіст дає provider**, тож абсолютний поріг «p95 ≤ 1,5 с» відхиляю як недосяжний без втрати платежів. Див. розділ 8 |

Success rate checkout при дефолтах: 92,86%; 5,22% відмов payment, 1,92% «charged-but-failed».

## 1. Current architecture

![Поточна архітектура OrderFlow](./docs/current-architecture.svg)

Суцільні стрілки: свідомі виклики. Пунктир: порушення ownership.

| Модуль | Відповідальність | Залежності | Оцінка |
| --- | --- | --- | --- |
| Checkout | Координація flow, компенсація (release, fail) | Orders, Inventory, Payments, Notifications, усі sync | Orchestrator має знати учасників: **виправдано**. Невиправдане лише `await` на Notifications і відсутність idempotency на вході |
| Orders | Життєвий цикл замовлення | Свої дані + `products` Inventory | Власні дані нормально. Читання `products` порушує ownership ціни |
| Inventory | Reserve/release | Свої `products`, `inventoryOperations` | Добра межа, idempotent reserve. Дірка №3 |
| Payments | Виклик provider, запис платежу | **`OrderRepository`**, FakePay sync | Шкідлива залежність від Orders. Немає timeout і станів `PENDING/UNKNOWN` |
| Notifications | Рендер і відправка email | FakeMail sync | У критичному шляху шкідлива (знахідка №1) |
| Reporting | Sales summary | Читає 4 таблиці чужих модулів | Читання без запису: безпечний coupling, небезпечний алгоритм |

**Що вже добре [code]:** `createApplication` це composition root з injectable providers (`options.paymentProvider`, `options.notificationProvider`). Контракт `charge({amountCents, currency, reference, paymentToken})` фактично вже є port-ом. Тому adapter для другого provider коштує дешево.

## 2. Problems (ranked)

1. **Коректність revenue path:** charged-but-failed 1,92%, double charge при retry, false `RESERVED` [code, measured].
2. **Coupling Orders↔Payments↔Checkout:** 48% feature PR торкаються обох, Modules/Feature 1,7 → 4,3 [doc].
3. **Спільний release:** 5 → 2 release/тиждень, Rollback 2% → 9%, MTTR 35 → 95 хв, один test env з чергою 3–5 днів [doc].
4. **Provider latency без захисту:** інциденти 1 → 5 на місяць, 2 останні почалися з latency spikes [doc].
5. **Reporting:** O(N²) блокує event loop [measured].
6. **Price/discount без власника** (блокує Scenario F) [code].

Метрики кореляційні: частину росту Deployment Duration дають ручні approvals і більший test suite [doc].

## 3. Architectural characteristics (пріоритет)

| # | Характеристика | Чому | Чим поступаємось |
| --- | --- | --- | --- |
| 1 | **Reliability і коректність checkout** (гроші, stock, відсутність double charge) | Revenue [doc], знахідки 1–3 | Миттєвість email, простота «все в одному `await`» |
| 2 | **Data consistency** (Inventory, Payments) | Scenario D, overselling неприпустимий | Незалежний deployment Orders/Inventory/Payments |
| 3 | **Modifiability** (Payments, Notifications, pricing) | Scenarios A, B, F | Вільний доступ між модулями |
| 4 | **Deployability / testability** | Rollback 9%, черга test env | Окремі services (відкладено) |
| 5 | **Performance isolation (Reporting)** | Scenario C, ×8–10 | Свіжість reports (хвилини прийнятні) |
| 6 | **Operational simplicity** | 8 engineers + 2 QA, platform team немає | Це обмеження, не ціль, але воно блокує складні рішення |

## 4. Principles на конкретних місцях

| Принцип | Де порушено | Що змінюється |
| --- | --- | --- |
| **Information Hiding** | Payments знає repository Orders. Orders знає таблицю `products` | `PaymentRequest {orderId, amountCents, currency, method, idempotencyKey}` приходить від Checkout, Payments Orders не читає. Ціна виходить з Pricing |
| **Explicit Contracts** | Контракти неявні, idempotency лише в Inventory | `PaymentsApi`, `PaymentProvider` port, версіоновані events, `Idempotency-Key` на `/checkout`, contract tests |
| **Dependency Direction** | Checkout чекає на Notifications, Reporting тягне дані з усіх | Notifications і Reporting залежать від events (`OrderCompleted`, `PaymentCaptured`), а не навпаки |
| **Ownership** | Ніхто не володіє ціною, snapshot ціни відсутній, Reporting читає чужі таблиці | Pricing володіє rules, Orders зберігає price lines, Reporting володіє read model |
| **Fitness Functions** | Нічого не захищає межі | Dependency rule + failure-injection test у CI (розділ 10) |

## 5. Proposed architecture

![Запропонована архітектура OrderFlow](./docs/proposed-architecture.svg)

**Deployment boundaries:** до Фази 3 один deployable. **Sync:** Checkout → Pricing, Orders, Inventory, Payments (на revenue path лишається sync, але з timeout). **Async:** Notifications і Reporting через outbox.

### Фази (години це [assumption])

| Фаза | Зміст | ≈Години | Що закриває |
| --- | --- | ---: | --- |
| **0: виправлення дефектів** (дні) | Email більше не валить оплачений order (try/catch + запис `PENDING`, потім outbox). `Idempotency-Key` на `/checkout`. Індекс по `orderId` у Reporting. Фікс false `RESERVED`. Тести на кожен | 60 | Знахідки 1, 2, 3, 6, 7 |
| **1: межі всередині monolith** (2–4 тижні) | `PaymentsApi` без `OrderRepository` зі станами `PENDING/UNKNOWN`, timeout provider-а (≈ 3,5 с) і reconciliation job. Pricing + price lines в Orders. Durable outbox, Notifications як consumer, templates як config. Reporting read model. Dependency-rule test | 300 | Driver №1, знахідка 8, Scenarios A, C, E, F |
| **2: процес** | Contract tests, feature flags, ephemeral test env, більше release windows, менші PR | 160 | Driver №2, TTM |
| **3: умовне виділення** | Reporting (перший кандидат), Notifications worker. Лише за trigger з розділу 12 | — | Scenario C на масштабі |

Фази 0–2 разом: 520 год × $50 = **$26k** (+ вартість test environment, невідома).

**Timeout у Фазі 1, не в Фазі 0:** без станів `PENDING/UNKNOWN` і reconciliation він породжує нову форму double charge (розділ 8). Сума годин не змінюється; до кінця Фази 1 latency лишається ризиком R2.

## 6. Decision summary

| Problem | Evidence | Decision | Trade-offs | Success |
| --- | --- | --- | --- | --- |
| Гроші й stock розходяться при збоях | 1,92% charged-but-failed, double charge при retry | Фаза 0: idempotency на вході, email поза критичним шляхом, фікс compensation | Email може запізнитись або прийти двічі | Charged-but-failed = 0 у failure-injection тесті |
| Payments ↔ Orders ↔ Checkout змінюються разом | 48% PR, 4,3 modules/feature | `PaymentsApi`, Pricing, price lines, dependency test | Більше контрактів і mapping-коду | Modules/feature < 2,5 |
| Reporting душить checkout | 5 с блокування на 20k orders | Спершу індекс (11 мс), потім read model, services потім | Reports відстають на хвилини | Event loop block < 100 мс на 20k |
| Спільний release | Rollback 9%, MTTR 95 хв, 2 release/тиждень | Менші release, flags, ephemeral env (не services) | Core лишається одним deployable | Rollback < 4%, Lead Time ≤ 3 дні |

## 7. Зв'язок зі scenarios

| Scenario | Stimulus → response | Affected components | Що змінюємо | Response measure |
| --- | --- | --- | --- | --- |
| **A** Notifications | Marketing править template → без deploy і без регресії checkout | Notifications, Checkout, outbox | Templates як versioned config, kill switch, Notifications як consumer | Зміна template не чіпає Orders/Checkout, rollback < 5 хв |
| **B** Другий provider | Provider B з async confirmation | Payments (`PaymentsApi`, adapter), Checkout | Port + adapter, стани платежу `PENDING/UNKNOWN`, reconciliation | Новий provider = новий adapter, 0 змін в Orders |
| **C** Reporting ×8–10 | Більше запитів і довші ranges | Reporting, outbox / read model | Індекс → read model → (trigger) окремий deployable | Checkout overhead p95 не росте при ×10 report load |
| **D** Inventory idempotency | Повтор reserve/release | Inventory, Checkout | Фікс false `RESERVED`, idempotency на вході | 0 подвійних змін stock у тесті |
| **E** Checkout availability | Provider 2–3 с або down | Checkout, Payments (provider port), Notifications | Timeout, circuit breaker, email async | Overhead p95 (без часу provider) ≤ 150 мс, 0 charged-but-failed |
| **F** Discount rule | Правило за payment method + набором товарів, змінюється щомісяця | Pricing, Orders, Checkout, Payments, Reporting | Pricing як pure rules, price lines з discount в Orders, projection у Reporting | Сума до списання == total order (assert), правило = 1–2 modules |

Жоден зі scenarios A–F не вимагає окремого service на цьому етапі.

## 8. Cost of change

**Drivers (за впливом):** (1) Orders↔Payments↔Checkout через спільний доступ до даних, 48% PR; (2) спільний release, test env і rollback; (3) sync Notifications, 37% змін template тягнуть повний regression; (4) price без owner (Scenario F зачіпає 5 точок: `shared/money.js`, `OrderService.createDraft`, `CheckoutService`, `PaymentService`, `ReportingService`); (5) нерівномірне knowledge, review чекає на двох senior.

```text
Change Cost = Engineering Hours × Hourly Cost
Architecture Tax = Current Change Cost − Baseline Change Cost
```

**Scenario F, нове discount rule [assumption]:**

| | Зараз | Після Фаз 0-1 |
| --- | ---: | ---: |
| Modules touched | 4 (+shared) | 2 (Pricing, Reporting projection) |
| Implementation | 24 h | 14 h |
| Coordination | 12 h (≈3 h на кожен зайвий module) | 4 h |
| Testing | 14 h (регресія checkout) | 8 h (contract + Pricing) |
| Release | 6 h | 6 h (release досі спільний) |
| **Change Cost** | **56 h × $50 = $2 800** | **32 h × $50 = $1 600** |

Baseline (за `analysis-tools.md`) = очікувана вартість того самого правила після Фаз 0-1. Architecture Tax = $2 800 − $1 600 = $1 200 на правило, при щомісячних змінах ≈ $14 400 на рік.

**Notification experiment [assumption]:** ≈4 h правка + 37% × ≈30 h регресії ≈ **15 h** зараз проти ≈ **5 h** після. При 2,5 експерименті на тиждень економія ≈ 10 h × 2,5 × $50 ≈ $1 250 на тиждень ≈ $65k на рік.

**Окупність:** інвестиція $26k проти економії ≈ $79k на рік дає ≈ 4 місяці. Якщо реальна економія вдвічі менша (менше експериментів, менша частка regression), ≈ 8 місяців. **Це перше місце, де assumption треба перевірити:** виміряти фактичну кількість експериментів і години на зміну template до старту Фази 1.

**Trade-off 1 (timeout vs latency).** Короткий timeout (1,5 с) «покращив» би p95, але відрізав би ≈ 9–10% повільних, здебільшого успішних платежів. Тому timeout ≈ 3,5 с (трохи вище хвоста), а ціль задана як **overhead без часу provider**. User-facing latency реально знижує лише async confirmation (Scenario B): це зміна UX і нові стани платежу.

**Trade-off 2 (email).** Sync email додає лише ≈ 300 мс до p95 (2 539 → 2 244). Реальний виграш від його прибирання це **1,92% коректності**, а не latency.

**Trade-off 3 (timeout → невизначений стан).** Після timeout невідомо, чи provider списав гроші. Без `UNKNOWN` і reconciliation фікс породжує нову форму double charge (особливо з provider B), тому timeout іде у Фазі 1 разом з `PaymentsApi`.

## 9. Risk analysis

`Expected Risk Cost = Probability × Business Impact`. Вірогідності й impact це [assumption], якщо не позначено інакше. Припущення обсягу: 20 000 checkout на квартал.

**Ризики поточної architecture:**

| # | Risk | Probability | Business Impact | Expected Risk Cost | Mitigation |
| --- | --- | ---: | ---: | ---: | --- |
| R1 | Charged-but-failed order (знахідка 1) | 1,92% [measured] × 20 000 = 384 події на квартал | $40 (refund, support, відтік) | **≈ $15,4k на квартал** | Фаза 0: email поза критичним шляхом, failure-injection тест |
| R2 | Деградація checkout від provider latency | 6 інцидентів на квартал (15 × 40% [assumption]; у [doc] лише 2 останні почалися з latency spikes) | $4k | **≈ $24k на квартал** | Timeout (Фаза 1, разом зі станами платежу), circuit breaker, alert на overhead p95 |
| R3 | Rollback всього release | 9% × ≈26 release = ≈2,3 на квартал [doc] | $3k | **≈ $7k на квартал** | Менші release, flags, canary |
| R4 | Reporting блокує event loop на піку | 30% на квартал (зростання ×8–10) | $10k | **$3k** | Індекс, read model, trigger на виділення |

**Ризики, які створює власне рішення:**

| # | Risk | Probability | Business Impact | Expected Risk Cost | Mitigation |
| --- | --- | ---: | ---: | ---: | --- |
| S1 | Outbox на non-durable storage. Зараз in-memory DB, тож рестарт або failover на резервний instance **губить непрочитані events і сам стан**. Outbox має сенс лише на durable store | 100% при рестарті, якщо не замінити storage | Втрата листів і projection | **Блокер Фази 1**, не ризик, який можна прийняти | Durable DB (хоча б одна таблиця outbox) до Фази 1; in-memory лишається лише для тестів |
| S2 | At-least-once: дубль листа або події | 20% на квартал | $2k | **$400** | Dedupe key `orderId+template`, idempotent consumers |
| S3 | Timeout → невизначений платіж (`UNKNOWN`), особливо з provider B | 10% на квартал | $20k | **$2k** | Стани платежу, idempotency key до provider, reconciliation job |
| S4 | Pricing: неправильне правило, клієнт бачить не ту суму | 15% на квартал | $20k | **$3k** | Pure function, golden-кейси, assert «сума charge == total order», flag для правил |
| S5 | Core лишається одним release: Payments rule чекає спільного window | 70% на квартал | $4k | **$2,8k** | Flags, менші release, перегляд за trigger |

## 10. Metrics та Fitness Functions

| Metric | Зараз | Ціль (6 міс.) | Чому важлива |
| --- | ---: | ---: | --- |
| Charged-but-failed rate | 1,92% [measured] | 0 | Прямий показник коректності revenue path |
| Modules Changed per Feature | 4,3 | < 2,5 | Вимірює coupling, driver №1 |
| Change Lead Time | 6 днів | ≤ 3 дні | Підсумок friction |
| Rollback Rate | 9% | < 4% | Blast radius спільного release |
| Incidents / MTTR | 5 на міс. / 95 хв | ≤ 2 / ≤ 45 хв | Ефект для reliability |
| Report event-loop block при 20k orders | 5 010 мс | < 100 мс | Ізоляція Reporting |
| Average PR Size | 520 LOC | < 250 LOC | Review capacity, критично для AI |

**Fitness Functions (CI):**

1. **Dependency rule з ratchet:** тільки `orders/` імпортує `OrderRepository`; модуль звертається лише до своїх колекцій `database.*`. Поточні порушення (Payments→OrderRepository, Orders→`products`, Reporting→4 таблиці) занесені в allow-list, який може лише зменшуватись, тож тест зелений сьогодні й червоніє на нових порушеннях.
2. **Failure injection:** email provider кидає помилку після capture → order `COMPLETED`, payment `CAPTURED`, одне notification у `PENDING`. **Сьогодні цей тест падає** (знахідка 1).
3. **Idempotency:** два `POST /checkout` з тим самим `Idempotency-Key` → один charge, один order.
4. **Overhead p95:** checkout p95 мінус час provider ≤ 150 мс при injected latency 3 с і 5% помилок. (Абсолютний поріг «p95 ≤ 1,5 с» відхиляю: див. trade-off 1.)
5. **Template change:** зміна template не змінює файли Orders/Checkout і запускає лише тести Notifications.

## 11. AI та Time to Market

### 11.1 Перевірка припущення

```text
Implementation 40 → 10 днів (4x):  TTM = 20 + 10 + 25 + 25 + 10 = 90 днів ≈ 3 міс., не 1
Implementation = 0:                 TTM = 80 днів ≈ 2,7 міс.
Implementation = 40/120 = 33% pipeline ⇒ межа прискорення 120/80 = 1,5x (закон Амдала)
```

**Implementation speed ≠ end-to-end delivery speed:** 4x на стадії з третиною pipeline дає 1,33x для всього pipeline.

### 11.2 Оцінка по stages (дні)

| Stage | Зараз | Conservative | Expected | Aggressive | Assumption |
| --- | ---: | ---: | ---: | ---: | --- |
| Analysis | 20 | 18 | 16 | 15 | AI-research, prototype (−2…−5); залежить від stakeholders |
| Implementation | 40 | 20 | 16 | 10 | Scaffolding, tests, refactoring; обмежено review і якістю коду |
| Integration | 25 | 21 | 17 | 15 | Contracts, менші changesets, contract tests (−4…−10) |
| Validation | 25 | 20 | 17 | 15 | Додатковий test env, automation (−5…−10) |
| Release | 10 | 8 | 6 | 4 | Більше windows, flags, менший scope (−2…−6) |
| **TTM** | **120** | **87** | **72** | **59** | |

Діапазони команди [doc] перекриваються (contract tests і менший deployment scope лікують ту саму чергу), тому до Expected додаю запас: **≈ 75 днів**.

### 11.3 Target

| Горизонт | Target | Умова |
| --- | --- | --- |
| Перша major feature після Фаз 0-1 (≈ 6 міс.) | **≈ 95 днів (≈ 3 міс.; діапазон 85–105)** | Impl 24, Analysis 18, Integration 23, Validation 22, Release 8; частина benefits ще недоступна, а Фази 0-1 забирають ≈ 360 год ємності |
| 12 міс. | **65–80 днів (≈ 2–2,5 міс.)** | Фаза 2 завершена, contract tests, flags, ephemeral env |
| Цілий місяць | **Лише для першого вертикального slice**, не для цілої major feature | Розділ 12.6 |

### 11.4 Що AI прискорює, а що ні

- **Прискорює:** Implementation (scaffolding, tests, refactoring, docs); частково Analysis і Validation (generated tests, лише з review).
- **Не прискорює:** черга test env (3–5 днів), 2 release windows на тиждень, approvals, human review для security/finance, review двох senior, доступність stakeholders. Bottleneck переміщується з Implementation у Review → Integration → Validation → Release.
- **Architecture constraints:** Orders↔Payments coupling, один deployable, sync Notifications. **Process constraints** (код їх не лікує): один test env, 2 windows, approvals, review capacity, відсутній baseline дефектів AI-коду.

### 11.5 Нові ризики від більшого throughput

**Черга review росте нелінійно** (M/M/1, [assumption]): при ρ ≈ 0,7 очікування ∝ ρ/(1−ρ) ≈ 2,3, а після +40% потоку PR ρ ≈ 0,98 і множник ≈ 49. Тому PR < 250 LOC і ширше коло reviewers це **передумова**. Також: невідомий defect rate AI-коду, AI відтворює наявний coupling, більше змін у release підвищує rollback, test env і release window стають жорсткішим bottleneck.

### 11.6 Як виміряти вплив AI (не LOC і не кількість PR)

Cycle time по stages з акцентом на queue time; Change Failure Rate і Rollback Rate окремо для PR з міткою «AI-assisted»; rework за 7 днів; escaped defects; review wait time; PR size.

**Що змусить переглянути прогноз:** Implementation > 20 днів або review wait > 2 днів (завищено на стороні review); defect rate AI-PR вище baseline (знижую ціль); Integration + Validation не зменшились (проблема в процесі); TTM < 70 днів без росту incidents (можна наближати 2-місячну ціль).

## 12. Architectural judgment

### 12.1 Proposal відхилено частково

`architecture-proposal.md` пропонує п'ять independently deployable services.

| Частина | Рішення | Чому |
| --- | --- | --- |
| Крок 1: зафіксувати contracts | Приймаю | Основа всього, дешево |
| Крок 2: data ownership, прибрати чужі repositories | Приймаю | Прямо б'є по driver №1. Додаю Pricing і price snapshot, яких у proposal немає |
| Крок 3: async там, де не потрібна негайна відповідь | Приймаю частково | Notifications і Reporting так. Payments, Orders, Inventory ні: на revenue path лишається sync з timeout |
| Крок 4: виділяти services після operational readiness | Приймаю як gate | Умови нижче |
| П'ять independent services | **Відхиляю** | 8 engineers + 2 QA без platform team; Scenario D вимагає consistency між Orders, Inventory, Payments; найбільші дефекти (1,2,3) лежать у **логіці compensation**, яку network boundary не виправить, а погіршить |

Відповідь на open question proposal «чи достатньо спершу покращити boundaries всередині monolith?»: так, і це дешевше.

Окремо відхиляю свою ж першу інтуїцію виділити Payments першим (найчастіше змінюється). Це поставило б network hop і partial failure на сам revenue path, а проблема Payments лежить у межі й станах, не в process.

### 12.2 Що свідомо не змінюється

- Orders, Inventory, Payments лишаються in-process: consistency дешева, overselling дорогий.
- Checkout лишається orchestrator і власником end-to-end reliability (відповідь на open question proposal).
- Один deployable для core до Фази 3.
- Sync Inventory ↔ Orders через Checkout.
- Shared DB як модель даних core. Водночас замінити in-memory на durable store **необхідно** (ризик S1), але це заміна storage, не зміна architecture.

### 12.3 Чому складніша architecture ще не виправдана

- Evidence кореляційне: Modules per Feature і Lead Time могли зрости через розмір features і процес (ручні approvals, один test env), а не лише через межі.
- П'ять services це п'ять pipelines, tracing, on-call і eventual consistency для 10 людей без platform team.
- Найбільший виміряний вплив на checkout (Reporting) лікується індексом: 5 010 → 11 мс, тобто виграш у ≈ 450 разів без жодного нового deployable.
- Окремий deployment дає вигоду там, де різне навантаження або незалежні команди. Зараз такої потреби не показано.

### 12.4 Умови майбутньої evolution

- **Reporting → окремий deployable**, якщо після індексу й read model report load все ще піднімає overhead p95 понад поріг, або трафік ≥ 6× baseline, або read model не витримує навантаження.
- **Notifications → worker**, якщо черга листів потребує окремого масштабування. Для Scenario A окремий deployable не потрібен: templates це дані.
- **Payments → окремий service**, якщо provider B з async confirmation уже в проді, є окрема команда Payments і підтверджено, що спільний release це bottleneck (Lead Time не покращився після Фази 2).
- **Загальні prerequisites:** ≥ 2 команди з власними modules (≈ понад 14 engineers), CI/CD на service, distributed tracing, on-call процес.

### 12.5 Що спростовує assumptions

- Modules per Feature після Фази 1 не нижче 3 → coupling не єдина причина, шукаю процес, services не рухаю.
- Rollback Rate не падає при менших release → причина не в розмірі release.
- Checkout overhead p95 не змінюється при ×10 report load після індексу → Reporting виділяти не потрібно.
- Фактична кількість template-експериментів нижча за 2,5 на тиждень → окупність у розділі 8 завищена, пріоритет Фази 1 знижується.
- Реальний (не симульований) email failure rate нижчий за 2% → R1 менший, але idempotency і compensation лишаються потрібними.

### 12.6 Що має змінитися для реального місячного delivery cycle

Бюджет місяця ≈ 28 днів: Analysis ≤ 5, Implementation ≤ 10, Integration ≤ 5, Validation ≤ 6, Release ≤ 2. Це скорочення non-implementation stages з 80 до 18 днів (≈ 78%, по кожній stage 75–80%), а діапазони команди підтримують лише 13–31 день із 80 (≈ 16–39%). Потрібно: (1) **slicing** за feature flags, TTM рахується до першого використовуваного slice; (2) trunk-based, PR < 250 LOC, contract tests; (3) ephemeral test environments; (4) release on demand; (5) ширше коло reviewers і автоматичні policy checks (human review лишається для security/finance); (6) автоматизована validation замість ручної регресії.

**Висновок:** 1 місяць для цілої major feature за наявних умов нереалістичний (4x на Implementation дає ≈ 90 днів). Він досяжний для першого slice після Фази 2. Реалістичний target: ≈ 3 місяці для найближчої major feature і 2–2,5 місяці через 12 місяців.