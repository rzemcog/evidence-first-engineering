# GRACE 4 — технический аудит контура требований и доказательств

**Дата:** 2026-11-22. **Тип:** анализ исходников, skills, шаблонов и тестов; не запуск CLI/агентов.  
**Исследованная ревизия:** [osovv/grace-marketplace@70f5ec4305bb2cc6ed82dea9470a3b810603874b](https://github.com/osovv/grace-marketplace/commit/70f5ec4305bb2cc6ed82dea9470a3b810603874b). Документированный выпуск: 4.2.1.  
**Статус:** исследовательские выводы, а не утверждённая архитектура нашей композиции. Ссылки фиксированы на SHA.

## 1. Контур артефактов

```text
.grace/context/requirements.xml   цели, пользователи, ограничения
           ↓              (смысловая интерпретация — не formal proof)
.grace/changes/active/C-*/spec.xml   GraceChangeSpec, draft → approved
           ↓
.grace/changes/active/C-*/plan.xml   GraceChangePlan; baseline/target;
           ↓                        durable/observed scope, T-* tasks
.grace/graph/ + source markup        граф модулей и контракты
.grace/verification/                V-M-* verification contracts
           ↓
grace lint (current / baseline / target / final), optional run-commands
           ↓
.grace/changes/archive/C-*          applied/rejected/cancelled/superseded
```

Это **сводная схема документированного workflow и программных проверок**, а не доказательство, что CLI самостоятельно защищает всю цепочку от ошибочной интерпретации пользовательского intent.

## 2. Проверенные механизмы

### G01. Контекст требований и нормативный документ изменения

- [Шаблон требований](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/skills/grace/grace-init/assets/.grace/context/requirements.xml.template): ProjectName, Annotation, Keywords, Goals, Users, Constraints. Это контекст проекта, но **не подтверждённый реестр отдельных исходных реплик пользователя**.
- [Проверка проекта](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/grammar.ts#L400-L438) требует существования ожидаемых документов контекста и графа; [валидация корня](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/grammar.ts#L105-L163) проверяет тип, версию и атрибуты XML. Смысловую полноту требований код здесь не доказывает.
- [grace-spec](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/skills/grace/grace-spec/SKILL.md) создаёт нормативный GraceChangeSpec (draft → approved после явного пользовательского согласия). Необязательный design-context.xml помечен как **ненормативный**.
- [Шаблон change spec](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/skills/grace/grace-spec/references/change-spec-template.xml): Summary, Problem, Goals, Constraints, NonGoals, AcceptanceCriteria, AffectedAreas, VerificationIntent, Assumptions.
- [Валидация спецификации](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/grammar.ts#L262-L345) проверяет статусы, один C-* wrapper, обязательные содержательные разделы. [Bundle validation](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/grammar.ts#L497-L561) проверяет согласованность ID spec/plan и выдаёт ошибку, если активный plan существует рядом с не-approval spec.
- **Граница:** [resolveSelectedApprovedPlan](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/lint/core.ts#L297-L343) проверяет значение XML status=approved и совпадение ID, но **не удостоверяет, кто именно дал согласие на конкретную версию документа**. Внешний механизм approval следует анализировать отдельно.

### G02. План с отдельными фазами, областями изменения и зависимостями

- [Шаблон плана](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/skills/grace/grace-plan/references/change-plan-template.xml) содержит BaselineAssertions, TargetAssertions, DurableScope, ObservedWriteScope и T-* tasks с verification commands.
- [grammar.ts](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/grammar.ts#L641-L715) проверяет структуру assertions и scopes, [тесты](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/grammar.test.ts#L125-L150) проверяют запрет plan рядом с draft spec.
- DurableScope описывает **семантические элементы .grace**, ObservedWriteScope — **фактические файлы записи**. Два разных представления полезны для управления изменениями без приравнивания файла к требованию.
- [detectScopeOverlaps](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/scope.ts#L202-L218) выдаёт **предупреждения** о пересечениях семантических областей. [detectUnsafeConcurrentExecution](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/scope.ts#L219-L239) формирует **ошибки** по пересечению durable и observed scopes в режиме явного parallel preflight.
- Это защита пересечений исполнения, **не универсальное разрешение конфликтов версий утверждённых требований**.

### G03. Проверяемая структура, реальный запуск и пользовательский intent — разные гарантии

- [evaluateMustVerify](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/assertions.ts#L336-L345) **проверяет наличие** записи V-M-* в verification projection. Он **не запускает тест**.
- [validateModuleVerificationCoverage](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/projections.ts#L397-L415) требует V-M-* для каждого M-* и обнаруживает ссылки на отсутствующие модули.
- [evaluateMustPassCommand](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/assertions.ts#L347-L376) анализирует actual command results, timeout/exit code и требует явного включения runCommands.
- [validateAssertions](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/lint/core.ts#L139-L229) различает current active baselines, selected baseline, selected target и final, где проверяется selected target плюс другие approved baselines. Архивные планы не переоцениваются семантически как текущий контракт.
- [runDeclaredCommands](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/command-runner.ts#L76-L196): последовательные команды, fail-fast, логи, метаданные, таймауты.
- **Вывод:** coverage record ≠ успешный запуск команды ≠ доказательство, что принятый критерий верно выражает исходное пожелание пользователя.

### G04. Жизненный цикл изменений и происхождение evidence

- [types.ts](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/types.ts#L27-L39) определяет draft/approved и applied/rejected/cancelled/superseded; [grammar.ts](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/grammar.ts#L325-L345) требует ссылку на replacement у superseded.
- [readVcsIdentity](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/run-log-store.ts#L112-L140) собирает HEAD, branch, dirty, а [command-runner](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/command-runner.ts#L104-L129) связывает их с командным запуском и PID в meta.json.
- [pruneRuns](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace4/run-log-store.ts#L186-L230) сохраняет при ротации самый свежий успешный запуск каждого change.
- **Граница:** наличие идентификатора HEAD и dirty-флага не означает криптографической идентификации всех байтов незакоммиченного дерева. Архив change bundles не демонстрирует автоматически историю каждой промежуточной редакции draft.

### G05. Фактический механизм drift и пределы наблюдаемости

- [collectObservedDrift](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace-status.ts#L361-L395) использует git status --porcelain, делит изменённые файлы на объясняемые approved scope и необъяснённые.
- [collectApprovedContractDrift](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace-status.ts#L475-L482) отслеживает незакоммиченные изменения tracked approved spec.xml/plan.xml и помечает approved-contract-drift; [chooseNextAction](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/src/grace-status.ts#L167-L176) предлагает hard stop.
- **Граница:** это контроль наблюдаемого worktree, а не доказательство неизменности любых ранее закоммиченных утверждённых редакций без внешней политики.
- [grace-refresh](https://github.com/osovv/grace-marketplace/blob/70f5ec4305bb2cc6ed82dea9470a3b810603874b/skills/grace/grace-refresh/SKILL.md) предписывает оформлять reconciliations новым change, не переписывая молча текущие документы.

## 3. Что стоит взять в универсальный каталог вариантов

| Код | Переиспользуемый механизм | Польза | Ограничение |
| --- | --- | --- | --- |
| G01 | Нормативная spec отдельно от design context | Разделение обязательства и обоснования | Не обязан быть XML |
| G02 | Машинная валидация change bundles | Структурные гарантии вне слов агента | Не доказывает user approval |
| G03 | Baseline/target/final phases | Сравнение исходного и целевого состояния | Не вся задача имеет измеримый baseline |
| G04 | Durable и observed scopes | Управление параллельными изменениями | Не проверяет stale draft автоматически |
| G05 | Индексированный graph + verification projection | Поиск связей модуль → проверка | V-M-* лишь наличие контракта |
| G06 | Replacement bundles и архив | История решений | Не полная история draft-версий |
| G07 | Gate evidence + VCS metadata + retention | Проверяемое происхождение запусков | Dirty worktree требует доп. фиксации |
| G08 | Read-only status и выявление drift | Диагностика без дублирующей нормативной базы | Контроль Git worktree, не всю историю |
| G09 | Parallel preflight как отдельная политика | Вариативная степень строгости | Не гарантия безопасности любого concurrency |

## 4. Кандидаты для нашей композиции, ПОКА НЕ УТВЕРЖДЕНЫ

- **P-GRACE-01:** взять различение нормативной spec и поясняющего контекста (G01).
- **P-GRACE-02:** взять парные baseline/target assertions и evidence, связанное с ревизией (G03/G07).
- **P-GRACE-03:** рассмотреть dual scopes + explicit parallel preflight как опцию (G04/G09).
- **P-GRACE-04:** оценить graph/verification projection GRACE до проектирования собственной модели (G05).
- **P-GRACE-05:** проверить, дешевле ли расширить GRACE на user intent provenance и draft history, чем добавлять отдельный источник истины.

**Не становиться требованиями автоматически:** найденные возможности и предлагаемые компоненты требуют отдельного обсуждения и утверждения согласно D-010/D-013.

## 5. Ограничения и следующий источник

Подтверждены конкретные функции и ветки кода по фиксированному SHA; **GRACE не запускался**, LLM-агенты не тестировались. Из прочитанного кода нельзя делать универсальные утверждения о качестве и стойкости всех реальных инсталляций.

**Далее:** исследовать OpenSpec source-level по модели proposal → delta specs → apply/archive, затем сравнить подходы к G01–G09. Цель — выявление альтернатив и комбинаций, а не поиск продукта с максимальным числом выполненных чекбоксов.

[Утверждённая методика исследования D-014](2026-11-12-research-methodology.md) · [предыдущий GRACE-срез](2026-11-12-grace-requirements.md)
