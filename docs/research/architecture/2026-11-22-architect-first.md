# Architect First (Stanislav Rumega): модели intent/architecture и независимые gates

**Дата анализа:** 2026-11-22  
**Статус:** исследовательский анализ; не решение об интеграции и не утверждённое требование.  
**Репозиторий:** [styrumg/Architect-First-Article](https://github.com/styrumg/Architect-First-Article), pin [e8407e6067cdc5f70f81e906ca57f5d2556b4142](https://github.com/styrumg/Architect-First-Article/commit/e8407e6067cdc5f70f81e906ca57f5d2556b4142).  
**Автор и статус:** Stanislav Rumega, v1.0 (2026-07-25), [CITATION.cff](https://github.com/styrumg/Architect-First-Article/blob/main/CITATION.cff); paper по CC BY 4.0.  
**Просмотрено непосредственно:** README, CHANGELOG, CITATION, reference-run README, исходные prose requirements, JSON SIM/FAM, gate reports, decision memory, два файла Python реализации. **Ограничение:** двоичные PDF полного paper и двухстраничного summary доступны в GitHub, но их содержимое не удалось непосредственно извлечь в текущей инструментальной среде. Поэтому подробные положения разделов PDF вне изложенного в README, changelog и артефактах не выдаются за самостоятельно проверенные.

## 1. Тезис и принцип

Автор предлагает **pre-code assurance**: агент сначала делает формальный для машины (структурированный, типизированный) декларативный артефакт замысла — **System Intent Model (SIM)**, затем на его основе **Formal Architecture Model (FAM)**. Перед переходом к следующей стадии каждый проверяется **детерминированным admission gate**, правила которого не принадлежат агенту-исполнителю. Затем код проверяют существующие analyzers (Gate 3). Автор признаёт концепцию инженерной гипотезой без статистического доказательства улучшения качества на независимых задачах.

Представленный workflow: prose requirements → clarification/Loop Zero → SIM → Gate 1 → FAM → Gate 2 → code → Gate 3 → decision memory. Недостающие гарантии, неясные механизмы и failure semantics должны обнаруживаться **до генерации кода**. Аналогия с intermediate representations компилятора — инженерная метафора, не гарантия семантического сохранения естественного языка.

## 2. Механизмы, которые подтверждаются исходными артефактами

### SIM / Gate 1 (System Intent)

[Пример начального SIM](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/01-sim-initial.json): акторы, ответственность, порты, сообщения и delivery semantics, state machines, claims/guarantees, failure model. Важное поле: declared guarantee и **механизм, который его должен обеспечивать**.

В [Gate-1 report](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/02-gate1-initial-report.json) зафиксированы:
- **H3a blocking**: команда `ChargePayment` отправляется `at-most-once` без подтверждения доставки (риск немой потери);
- **H4 high**: restart supervisor без ограничений/backoff/escalation;
- **H13 high**: supervision заявлен с одной стороны, но не прослежен `supervised_by` для дочернего актора;
- **H3b medium**: заявлена идемпотентность без устойчивого механизма дедупликации;
- **H14 low**: отсутствует явный перечень зависимостей, включая пустой список.

Исправленный [SIM](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/04-sim-passing.json) содержит `at-least-once` + ack + механизм дедупликации по order ID, ограничение перезапусков, модель graceful shutdown, явные зависимости и обработку unexpected событий. [Gate 1 passing](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/05-gate1-passing-report.json) оставляет medium по зависимости, используемой в критичном пути. **SOUND в данном отчёте означает «нет blocking/high у этого набора эвристик», а не математическое доказательство no-double-charge.**

### Decision memory / authority

[Decision memory](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/03-decision-memory.jsonl) фиксирует `decision/value/authority/status/after` и обоснования; например, `5 per minute` и ack/delivery выбран агентом по умолчанию, а gateway idempotency contract — автором примера. Это полезная provenance-запись, но **нельзя автоматически считать agent defaults утверждёнными пользовательскими требованиями**. Наша D-010 требует отдельного принятия новых/изменённых нормативных требований.

### FAM / Gate 2 (Formal Architecture)

[Формальная архитектурная модель примера](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/06-payment-worker.fam.json) содержит `derived_from`, именованные actors/ports/state/event/transition, `trace`, guard/claim/layer/code_location, имена handlers и terminal states.

[Отчёт Gate 2](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/07-gate2-report.json) содержит проверки F1–F6:
- F1 — покрытие пар (state,event), в т.ч. unexpected;
- F2 — соответствие типа behaviour block объявленным свойствам;
- F3 — integrity lowering и traceability SIM → FAM;
- F4 — соответствие интерфейсов/портов;
- F5 — конкретные места реализации для protective guards;
- F6 — именованные обработчики для входящих сообщений.

**Название Formal Architecture Model не означает автоматического формального доказательства корректности системы.** Пример демонстрирует machine-readable consistency и coverage checks.

### Gate 3 (код и анализаторы)

[Уязвимый код](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/code/payment-worker-vulnerable.py) намеренно содержит `os.system`, `pickle.loads`, `bare except`, бесконечный busy loop. [Отчёт Gate 3](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/08-gate3-vulnerable-report.json) показывает два blocking, один high и два medium. В примере используется **mini-ast**, поскольку Bandit недоступен; это не доказательство общей эффективности SAST.

[Исправленный файл](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/code/payment-worker-fixed.py) и [Gate-3 SOUND](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/09-gate3-fixed-report.json) устраняют указанные паттерны. Однако в [decision memory](https://github.com/styrumg/Architect-First-Article/blob/main/examples/reference-run/03-decision-memory.jsonl) запись `gateway contract test / version pin` прямо помечена `declared, not demonstrated`. **Следовательно, в опубликованных артефактах не продемонстрировано независимое подтверждение ключевого внешнего контракта «не списывать дважды».** Пример обоснован как демонстрация этапов gate, не end-to-end доказательство платёжной корректности.

## 3. Концептуальная ценность для evidence-first-engineering

1. **Pre-code assurance / ранняя проверка:** проверяем claims, предполагаемые guarantees, recovery/failure policies и архитектурные обязательства до написания кода, а не только постфактум.
2. **Разделение intent и architecture:** SIM задаёт, что система обещает/предполагает; FAM — какие элементы и взаимодействия должны реализовать обещанное. Это другая проекция требований, дополняющая D-008 и D-013.
3. **Claim → mechanism → location → evidence:** в архитектурной модели требование гарантии связывается с enforceable mechanism, местом применения и впоследствии с фактическими свидетельствами. Очень пригодно для EBT-first, но фактические доказательства исполнения требуют последующих независимых проверок.
4. **Проверка архитектуры до кода:** типы сообщений, semantics доставки, полнота переходов, лимиты восстановления, декларированные зависимости и интерфейсные ограничения можно механически проверить ещё до написания реализации.
5. **Gate outside writer's authority:** проверяющий и политика должны быть за пределами прав исполнения; по [CHANGELOG](https://github.com/styrumg/Architect-First-Article/blob/main/CHANGELOG.md) автор различает независимость ролей, свидетельств и полномочий, а также охватывает CI config, runner, merge rules, credentials и evidence store.
6. **Точность детерминизма:** policy decision может быть детерминированным над frozen evidence bundle, даже если сенсоры/LLM вероятностны; новая проверка с новыми наблюдениями не обязана повторить прежний verdict. Это согласуется с нашей D-004 и D-006.
7. **Reviewable decision memory:** запись источника принятого решения, его инициатора и оснований помогает выявлять незаметное изменение intent.

## 4. Ограничения и контраргументы

- **Self-declared SIM может быть неверен или неполон.** Проверка схемы и internal consistency не восстанавливает упущенные требования автоматически; нужен D-008 provenance и механизм проверки исходного пользовательского intent.
- **Список эвристик и gate policy может иметь пробелы.** SOUND значит соответствие конкретным проверкам в конкретной редакции. Даже идеальная структурная проверка не доказывает correctness.
- **FAM не равен Lean/TLA+/Verus proof.** Формальная модель в контексте автора — машинно-читаемая строгая архитектурная декларация; доказательная сила зависит от того, какие свойства реально проверяет gate.
- **Риск избыточного workflow:** правило «всегда SIM → FAM → код» может противоречить согласованной D-002/D-007 адаптивности; для локального bugfix, UI-прототипа или исследовательского spike полноценная архитектурная модель может быть чрезмерной. Предлагается сделать gate-профили настраиваемыми по признакам задачи, сохраняя нормативную защиту intent.
- **Agent defaults как решения:** в reference-run часть вопросов решена `authority: agent default`, что допустимо для демонстрации проектирования, но не должно становиться новым утверждённым требованием без пользовательского согласования.
- **Trust boundary:** даже детерминированный gate бесполезен, если агент может менять policy/CI или подменять evidence. Требуется отделение **policy decision** от **enforcement** в действительной среде.
- **Репозиторий — не полный внедряемый продукт:** README прямо сообщает, что полные SIM/FAM schemas, правила и gate implementations не опубликованы до cold-test validation. Значит на данном этапе напрямую переиспользуемы прежде всего **идеи, конкретные опубликованные артефакты и примеры**, а не весь движок целиком.
- **Степень проверки:** автор признаёт отсутствие опубликованной слепой A/B-валидации пользы метода; эта научная проверка не является текущей целью EFE по D-014, но влияет на степень уверенности.

## 5. Не изобретать существующее: related prior art / альтернативы

| Инструмент | Класс возможности | Отличие от Architect First |
| --- | --- | --- |
| [GRACE 4](https://github.com/osovv/grace-marketplace) | change spec, module graph, assertions, evidence/CLI gates | Потенциальная точка интеграции pre-code архитектурных assertions без второго автономного workflow |
| [Structurizr DSL](https://docs.structurizr.com/dsl) | architecture-as-code, C4 diagrams/models | Модель структуры и визуализации; не подменяет failure-semantics Gate 1 и не гарантирует intent correctness |
| [ArchUnit](https://www.archunit.org/userguide/html/000_Index.html) | архитектурные правила для готовых Java-классов | Проверяет реальные кодовые зависимости, может быть post-code Gate 3/4; не проверяет первичный замысел |
| [Open Policy Agent](https://www.openpolicyagent.org/docs) | policy-as-code и решение по JSON input | Возможная реализация независимых gate rules; OPA decision point требует действительного enforcement в harness/CI |
| [TLA+](https://docs.tlapl.us/) | моделирование и model checking распределённых систем | Более строгая проверка сформулированных динамических свойств; специализированный метод, не обязательная форма FAM |

Таблица — **сопоставление функций и гипотез потенциальной композиции**, а не результат интеграционных тестов.

## 6. Предложения для нашей композиции (P-002; НЕ утверждены)

- Сделать **SIM/FAM-inspired intermediate representations** одним из способов проецирования требования на системные свойства и конкретную архитектуру; **не предписывать** обязательный отдельный файл или двухпроходный workflow для каждой задачи.
- Поддерживать **проверку обязательств до начала затратной реализации**: проверяемость failure model, механизм, границы ответственности, связность интерфейсов, происхождение решения, выявленные неопределённости. Вид и уровень gate выбирать по заданной политике.
- Развивать **claim → mechanism → enforcement location → observed evidence** как карту инженерной доказательности, соединяя D-008 с независимой приёмкой D-004/D-006.
- Разделить **evidence from sensor**, **deterministic verdict**, **permission to perform action**, **approval of normative requirement** как независимые ответственности.
- Использовать **decision memory** для пояснения архитектурных решений, но не превращать agent-suggested assumptions в принятые требования.
- Приоритетно проверить, какие из этих возможностей уже есть в GRACE, существующих policy engines и архитектурных DSL, прежде чем писать собственный schema/checker.

**Возможная композиция:** D-008 (исходный intent и реестр утверждённых требований) → альтернативные концепции (idea-generator-2 как источник идей) → **выборочная структурная проекция intent+design с gates (Architect First)** → реализация с прослеживаемостью и evidence → внешняя приёмка. Процесс итеративен; стрелки означают логические зависимости для конкретного изменения, не жёсткую последовательность всех задач.

## Источники

### GitHub автора
- [README](https://github.com/styrumg/Architect-First-Article/blob/main/README.md)
- [CHANGELOG](https://github.com/styrumg/Architect-First-Article/blob/main/CHANGELOG.md)
- [CITATION.cff](https://github.com/styrumg/Architect-First-Article/blob/main/CITATION.cff)
- [Worked example, все artifacts](https://github.com/styrumg/Architect-First-Article/tree/main/examples/reference-run)
- [Полная статья (PDF, существует в репозитории; непосредственно не прочитана)](https://github.com/styrumg/Architect-First-Article/blob/main/Tell-Your-Coding-Agent-to-Work-as-an-Architect-First.pdf)
- [Краткое изложение (PDF, существует в репозитории; непосредственно не прочитано)](https://github.com/styrumg/Architect-First-Article/blob/main/Architect-First-2pager.pdf)

### Дополнительный авторский текст и сопутствующие средства
- [Авторский LinkedIn: постановка и оговорки](https://www.linkedin.com/pulse/we-built-superb-reviewers-code-have-nothing-reviews-intent-rumega-a1pec)
- [OPA](https://www.openpolicyagent.org/docs) · [Structurizr](https://docs.structurizr.com/as-code) · [ArchUnit](https://www.archunit.org/userguide/html/000_Index.html) · [TLA+](https://docs.tlapl.us/)

**Метод:** анализ кода, JSON-артефактов и первоисточников (D-014), без массового тестирования LLM-агентов.
