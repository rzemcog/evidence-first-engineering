# OpenSpec — первичный source-level разбор управления изменениями

**Дата:** 2026-11-22. **Репозиторий:** [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec).  
**Фиксированная ревизия:** [`0c737dc1b543249b38b0f4716816fcf493d894dd`](https://github.com/Fission-AI/OpenSpec/commit/0c737dc1b543249b38b0f4716816fcf493d894dd).  
**Статус:** **начало технического исследования** для последующего сравнения с GRACE. Использованы исходники и skills; CLI и LLM-агенты не запускались. Конкретные возможности не считать характеристиками всех версий OpenSpec.

## Контур

```text
openspec/changes/<name>/                 рабочая область предложения
    proposal / design / tasks / specs/<capability>/spec.md  (дельты)
                        ↓
      sync specs / archive: обработка ADDED, MODIFIED, REMOVED, RENAMED
                        ↓
openspec/specs/<capability>/spec.md     действующая спецификация capability
                        +
архив change                         история предложенного изменения
```

**Ключевое отличие от GRACE:** OpenSpec описывает изменения требований **операциями над существующими requirement blocks** (`ADDED`, `MODIFIED`, `REMOVED`, `RENAMED`). GRACE 4 в исследованном контуре сильнее акцентирует утверждённый change spec, baseline/target assertions, scopes и verification graph. Это различие механизмов, а не рейтинг качества.

## Выводы с исходниками

### O01. Delta spec как язык изменения требований

- [openspec-sync-specs/SKILL.md](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/skills/openspec-sync-specs/SKILL.md#L75-L85) перечисляет четыре вида delta operations.
- [findSpecUpdates](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/src/core/specs-apply.ts#L120-L154) обнаруживает delta specs по capability path и определяет соответствующие целевые main specs.
- [buildUpdatedSpec](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/src/core/specs-apply.ts#L158-L209) создаёт обновлённое содержимое, учитывая added/modified/removed/renamed.
- [Обработка некорректного delta](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/src/core/specs-apply.ts#L210-L273) отказывает при непарном rename или требовании под неизвестным заголовком delta, а в иных случаях предупреждает о неучтённом содержимом.
- **Заимствуемая идея:** формальный diff требований как самостоятельный объект исследования и ревью, а не просто свободная новая версия текста.

### O02. Двунаправленное согласование артефактов

- [openspec-update-change/SKILL.md](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/skills/openspec-update-change/SKILL.md#L62-L94) требует проверить согласованность **в любом направлении**: изменение позднего артефакта может требовать пересмотра раннего, даже если build order другой.
- Сначала правки предлагаются **в разговоре**; затем по одному артефакту показываются пользователю, и запись производится **после подтверждения**.
- **Особенно интересно для D-013:** уже существующий подход к двунаправленной эволюции без жесткого «требования всегда перед проектированием».
- **Граница:** подтверждение в skill само по себе не означает технически защищённого approval конкретной версии исходного требования.

### O03. Отделение workflow readiness от семантического качества

- [openspec-propose/SKILL.md](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/skills/openspec-propose/SKILL.md#L126-L138) прямо указывает, что status артефакта может отражать **наличие файла**, а не корректность и полноту содержимого.
- **Полезная инженерная граница:** статус workflow и доказательство качества артефакта не должны быть одним понятием. Это напрямую соотносится с нашим правилом независимой приёмки, но не является доказательством его полной реализации в OpenSpec.

### O04. Особые случаи при применении и архивировании

- [specs-apply.ts](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/src/core/specs-apply.ts#L210-L273) явно учитывает неполные rename-пары и нераспознанные delta blocks; это полезные меры против тихой потери требований.
- [archive.ts](https://github.com/Fission-AI/OpenSpec/blob/0c737dc1b543249b38b0f4716816fcf493d894dd/src/core/archive.ts#L55-L150) отдельно разбирает удаление последнего requirement и ситуацию, когда неучтённый текст не должен исчезнуть при архивировании.
- **Идея:** проверять изменения на **потерю нераспознанного содержания** прежде чем переносить их в действующую спецификацию.
- **Ограничение:** такой контроль не равен привязке к первичным словам пользователя, а также не демонстрирует полной защиты от concurrent stale-base.

## Быстрое сопоставление с GRACE 4

| Измерение | GRACE 4 | OpenSpec | Вопрос для композиции |
| --- | --- | --- | --- |
| Изменение требований | GraceChangeSpec + пакет | Delta spec по capability | Разделять intent change и операции над требованиями? |
| Review | status=approved по skill, XML-check в CLI | предложенные правки по артефактам + user confirmation в skill | Как избежать второго конкурирующего normative source? |
| Контракты проверки | baseline/target assertions, V-M-*, MustPassCommand | в этом срезе не найден эквивалент такого же integrated graph/evidence контура | Как сохранить независимую проверку без привязки к одному продукту? |
| Корректировка артефактов | утверждённый plan нельзя переписать по skill | reconciliation в любом направлении | Какие части следует сделать настраиваемыми? |
| Контроль потери содержания | структурная грамматика XML | проверки orphaned/unaccounted content при delta/apply | Как избежать тихой потери пользовательского смысла? |

Ссылки на закреплённый [подробный аудит GRACE](2026-11-22-grace-source-audit.md).

## Кандидаты на идеи, не утверждённые изменения требований

- **P-OS-01:** явно представлять добавление, изменение, отзыв и переименование требования как типизированные операции delta (O01).
- **P-OS-02:** проверять при переносе изменений отсутствие непреднамеренно потерянного содержимого (O04).
- **P-OS-03:** рассмотреть двунаправленное приведение связанных артефактов к согласованности (O02) при неизменных нормативных границах.
- **P-OS-04:** разделить `artifact exists`, `artifact reviewed`, `artifact verified`, `requirement satisfied` (O03).

## Следующий объём OpenSpec

Разобрать в коде: построение и слияние delta по каждому типу операции, валидаторы и archive workflow, concurrency/base version awareness, состав нормативных файлов. Затем дополнить сравнение GRACE vs OpenSpec source-level, и перейти к Doorstop, Kiro, Spec Kit.

**Не проверялось:** реальное исполнение инструментов, поведение агентов, варианты Git/IAM authorisation, статистическая надёжность.
