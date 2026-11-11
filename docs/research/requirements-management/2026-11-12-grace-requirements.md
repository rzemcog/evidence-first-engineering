# GRACE 4: что реально есть в системе управления требованиями

**Дата проверки:** 2026-11-12. **Тип доказательств:** чтение открытого репозитория `osovv/grace-marketplace`, исходных шаблонов и skills. **CLI не установлен и не исполнялся** в рамках этого среза; claims из документации не считаются успешным тестом enforcement.

## Идентификация и область

Исследован [osovv/grace-marketplace](https://github.com/osovv/grace-marketplace), **GRACE = Graph-RAG Anchored Code Engineering**, документированная версия **4.2.1 (2026-09-24)**. Система включает skills и опциональный CLI `@osovv/grace-cli`. Это **не** приложение для end-user requirements management, а методология/пакет инструментов для coding agents.

Именно этот репозиторий терминологически совпадает с постами о GRACE, EBT/LDD, module contracts и verification plans. Однако тождество конкретной авторской закрытой конфигурации, о которой говорилось в присланных постах, **не устанавливалось**. Не путать с другими одноимёнными проектами GRACE (например, `grace.guide`, где используется другой стек и другая спецификация).

## Документально подтверждённые компоненты

| Компонент | Опубликованная реализация | Предел вывода |
| --- | --- | --- |
| **Project requirements context** | `.grace/context/requirements.xml`; шаблон `GraceRequirements` содержит `ProjectName`, `Annotation`, `Keywords`, `Goals`, `Users`, `Constraints` | Это контекст проекта, **не доказанный реестр первичных пользовательских сообщений** |
| **Change request** | `.grace/changes/active/C-ID/spec.xml`, `GraceChangeSpec`, поля `Summary`, `Goals`, `Constraints`, `NonGoals`, `AcceptanceCriteria`, `AffectedAreas`, `VerificationIntent`, `Assumptions` | Есть нормативная спецификация изменения, но versioned draft history не продемонстрирована |
| **Draft → approved** | `grace-spec/SKILL.md`: `status=draft` при создании, `status=approved` только после **explicit user approval**; в противном случае оставаться draft | Явное правило в skill; проверить, как CLI проверяет **субъект** approval и конкретную версию файла |
| **Plan после approval** | `grace-plan` требует approved spec; неподходящие статусы отклоняются; план draft до явного user approval | Поддержана нормативная граница в workflow |
| **Immutable approved plan** | `grace-plan` запрещает писать поверх approved plan, требует новый `C-*` bundle и replacement reference | Прямой существующий механизм, близкий к нашему D-010 |
| **Audit/history** | `.grace/changes/archive/C-*`; терминальные статусы applied/rejected/cancelled/superseded; changelog говорит об immutable archives | Change-level история подтверждена, но не автоматический снимок **каждой промежуточной версии draft** |
| **Evidence-based gates** | `GraceChangePlan` с baseline/target assertions, `MustPassCommand`, scope; `grace lint` current/baseline/target/final; версии 4.2.x улучшают хранение run evidence | Инструментальные gates есть; их полнота, независимость оракула и адекватность исходному intent — отдельные вопросы |
| **Semantic graph** | `.grace/graph` и `.grace/verification` с route/index и `M-*`, `V-M-*` | Трассировка module/verification; не доказано покрытие **каждой** первичной нормы `REQ-*` |

## Оригинальные фрагменты спецификации

В исходном `grace-spec/SKILL.md` прямо сказано: `Create spec.xml as status="draft". Set status="approved" only after explicit user approval.`  
В `grace-plan/SKILL.md`: `If plan.xml already exists with status="approved", stop before writing.`

Эти цитаты подтверждают, что GRACE 4 **уже решает часть обсуждённой нами процедуры утверждения и защиты утверждённых артефактов**. Они не устанавливают, что система криптографически доказывает факт согласия пользователя.

## Что дополнительно утверждалось в присланных постах

В пользовательском корпусе постов описывались: первичный RFC-реестр с сохранением исходных слов пользователя, семантические якоря, прошивка требований в код и логи, автоматический анализ drift и статус «0 drift». Это **заявления автора из переданных материалов**, которые необходимо проверять отдельно.

Публичный `requirements.xml.template` на текущем проходе содержит лишь верхнеуровневые `Goals/Users/Constraints`. На основании этого шаблона нельзя утверждать, что полный RFC-реестр есть или его нет в отдельных пользовательских проектах GRACE. Нельзя также считать `0 drift` доказательством отсутствия пропущенных при извлечении исходных требований.

## Существенные неизвестные для наших D-008 / D-010

1. **Provenance:** умеет ли GRACE штатно сохранить конкретное user message, его дату/ID, исходный контекст и семантические отношения с требованиями?
2. **Versioned draft:** как получить `draft v1 → v2 → v3 → restore v1 content` с сохранением всех вариантов, не полагаясь на неявный Git history?
3. **Exact approval:** чем подтверждается согласие пользователя на **точные байты конкретного draft**, а не простое присвоение `status="approved"`?
4. **Stale-base concurrency:** что произойдёт при двух активных `C-*` изменениях одного требования? Предотвращает ли CLI автоматическое применение старого draft?
5. **Validation:** каковы trust boundaries проверочных команд, сохраняется ли идентичность oracle независимо от исполнителя и можно ли обеспечить внешнее fail-closed `DONE`?
6. **Customization:** можем ли мы получить нужные гарантии исключительно через настройку skills + Git/CI/permissions, не дописывая GRACE CLI?

**Нельзя сейчас заявить ни «GRACE уже целиком покрывает нашу систему», ни «GRACE не имеет системы требований».** Первое выходит за пределы свидетельств, второе опровергается опубликованными skills и артефактами.

## Первоисточники (проверены)

- [GRACE README, current packaged version и модель GRACE 4](https://github.com/osovv/grace-marketplace/blob/main/README.md)
- [CHANGELOG, версии 4.0–4.2.1](https://github.com/osovv/grace-marketplace/blob/main/CHANGELOG.md)
- [grace-init / артефакты .grace](https://github.com/osovv/grace-marketplace/blob/main/skills/grace/grace-init/SKILL.md)
- [Шаблон требований](https://github.com/osovv/grace-marketplace/blob/main/skills/grace/grace-init/assets/.grace/context/requirements.xml.template)
- [grace-spec / draft-approval workflow](https://github.com/osovv/grace-marketplace/blob/main/skills/grace/grace-spec/SKILL.md)
- [GraceChangeSpec XML template](https://github.com/osovv/grace-marketplace/blob/main/skills/grace/grace-spec/references/change-spec-template.xml)
- [grace-plan / approved-plan immutability](https://github.com/osovv/grace-marketplace/blob/main/skills/grace/grace-plan/SKILL.md)

## Предварительная инженерная гипотеза

**GRACE 4 — сильный кандидат на role of assurance/change backbone**, потому что нормативные change bundles, проверочные утверждения, области записи, CLI gates и module graph уже интегрированы.

**OpenSpec** может оказаться избыточным рядом с GRACE, поскольку у обоих есть change workflow. **Doorstop** может дополнить GRACE реестром индивидуальных требований и review fingerprints, но также может создать второй источник истины. До выбора композиции нужен эксперимент на одной и той же небольшой задаче.

Этот текст — исследование и гипотезы, **не согласованное архитектурное решение**.
