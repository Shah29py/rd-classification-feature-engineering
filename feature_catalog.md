# Case 7 — Feature Catalog

> Этот файл содержит **актуальное состояние feature discovery** и не является полной историей экспериментов. История гипотез, ошибок, проверок и изменения методологии находится в `research_log.md`.
>
> **Важно:** `Candidate` не означает, что признак уже доказал качество в production. До Block 4 итоговый статус `Validated` не присваивается.

## Статусы

- **Candidate** — есть содержательный сигнал и достаточно evidence, чтобы передать признак в Feature Evaluation.
- **Validated** — признак прошёл системную проверку после Block 4: coverage, specificity, stability и target vs OTHER.
- **Rejected** — исследован и не показал достаточной самостоятельной ценности либо является redundant/confounded.
- **To investigate** — направление интересно, но данных пока недостаточно.

**Priority** (`High / Medium / Low`) — исследовательский приоритет, а не итоговый статус.

---

# 1. Product

## P1 — `product_object_type`

**Status:** Candidate  
**Priority:** Medium  
**Source:** `rd_data → product.idObjectType`  
**Type:** categorical

### Наблюдения

Основные значения:
- `Серийный выпуск`;
- `Партия`;
- `Единичное изделие`.

В matched target sample:

| TG | Серийный выпуск | Партия | Единичное изделие | Нет данных |
|---|---:|---:|---:|---:|
| TG4 | 97.59% | 2.39% | 0.03% | ~0% |
| TG35 | 82.37% | 1.15% | 0.01% | 16.47% |
| TG43 | 94.78% | 4.30% | 0.02% | 0.90% |

После контроля `rd_type` для `ДС`:
- TG35: 98.61% серийный выпуск, 1.38% партия;
- TG43: 95.63% серийный выпуск, 4.35% партия.

### Интерпретация

Различия есть, но они заметно слабее TNVED. Для `СГР` отсутствие значения в значительной степени определяется структурой типа РД.

### Next

Оставить как secondary structured feature и оценить в Block 4. Не использовать отсутствие значения как отдельное TG35-правило без контроля `rd_type`.

---

## P2 — TNVED feature family

**Status:** Candidate  
**Priority:** High  
**Evidence:** statistical + domain-supported

**Source:** `rd_data → product.tnved`  
**Type:** multi-value categorical / hierarchical code

### Структура

- хотя бы один TNVED есть у **86.73%** matched документов;
- 71 381 документов имеют один код;
- 2 211 — два;
- 687 — три;
- встречаются длинные списки до 279 кодов;
- отдельные значения имеют длину 2, 4, 6, 9 и 10 символов;
- основная масса кодов — 10-значные.

Один документ представляется как множество TNVED-кодов и их префиксов. Большие списки кодов не признавались автоматически ошибочными: по проверенным примерам это реальные наборы товарных кодов.

### Основные candidate anchors

| Feature | Support | Coverage основной TG | Purity внутри target TG | Target |
|---|---:|---:|---:|---|
| `has_tnved_3303` | 3 848 | ~97.0% | ~99.2% | TG4 |
| `has_tnved_3304` | 28 659 | ~41.8% | ~99.8% | TG35 |
| `has_tnved_3305` | 12 318 | ~18.0% | ~100% | TG35 |
| `has_tnved_3401` | 8 482 | ~12.4% | ~99.8% | TG35 |
| `has_tnved_2710` | 8 451 | ~60.1% | ~100% | TG43 |
| `has_tnved_3403` | 5 703 | ~40.5% | ~100% | TG43 |

### Domain validation

Проверка по `TG_Definitions.xlsx` подтвердила:

| TNVED-4 | Категория Definitions |
|---|---|
| `3303` | Парфимерия |
| `3304` | Косметика |
| `3305` | Косметика |
| `3401` | Косметика |
| `2710` | Моторные масла |
| `3403` | Моторные масла |

### Additional TG35 candidates

В residual TG35 были обнаружены дополнительные статистически сильные префиксы:

- `3306`;
- `3307`;
- `3402`;
- `3808`.

`3306`, `3307` и `3402` также присутствуют в `TG_Definitions.xlsx` в категории `Косметика`. Для `3808` прямого соответствия в Definitions не найдено, поэтому он классифицируется как **statistical-only candidate**, а не как domain-confirmed anchor.

### Coverage analysis

Исходные anchors `3304/3305/3401` покрывали около **72.0%** TG35.

Добавление `3306/3307/3402/3808` повысило TG35 coverage до **82.59%** и оставило **11 900** residual TG35 документов.

Без `3808` coverage составлял **81.05%**, поэтому `3808` даёт заметный дополнительный coverage, но требует более осторожной интерпретации.

### Multi-anchor conflicts

При document-level проверке найдено **20 документов**, где одновременно присутствуют anchors для двух target TG. Большинство конфликтов относятся к TG4/TG35. Ручная проверка показала, что в таких РД действительно встречаются несколько TNVED-кодов и товары из нескольких товарных областей.

Следовательно, TNVED anchors — это **evidence features**, а не взаимоисключающие правила.

### Ограничения

`Purity` рассчитана только внутри matched target sample 4/35/43. Это не production Precision: полный `OTHER`-пул ещё не прошёл системную Feature Evaluation.

### Next

- проверить candidate anchors против `OTHER`;
- проверить stability на train/validation;
- оценить multi-label behavior;
- решить, какие уровни иерархии (`2/4/6/10`) нужны;
- проверить взаимодействия с TR/declaration/text.

---

## P3 — `product_origin`

**Status:** Candidate  
**Priority:** Medium / Secondary  
**Source:** `rd_data → product.idProductOrigin`  
**Type:** categorical

### Coverage

Заполнено примерно **75.0%** matched документов.

Наиболее частые значения: Россия, Корея, Франция, Италия, Германия, Япония, Китай, Испания, Турция и др.

### Наблюдения по TG

Для TG4 заметно выше доля Франции и Испании. Для TG35 высока доля России и Кореи. Для TG43 заметнее Германия и Япония. Однако эти различия частично связаны с типом производителя.

Дополнительная проверка `product_origin × manufacturer_type` показала сильную зависимость: для ряда стран (в частности Франции, Германии, Италии и Японии) более 93% документов имеют `manufacturer_type = Иностранное юридическое лицо`.

### Интерпретация

Признак несёт дополнительный сигнал, но часть информации избыточна относительно `manufacturer_type`.

### Next

Оставить как secondary candidate. В Block 4 оценивать с контролем `manufacturer_type` и support отдельных стран.

---

## P4 — `product_name` / `product_info`

**Status:** To investigate  
**Priority:** Low / Secondary

В derived product fields coverage очень низкий:
- `product_name` на TG35 residual заполнен примерно у 5.36%;
- `product_info` реально непуст примерно у 2.0% residual.

Поэтому эти поля не рассматриваются как основной residual source.

При этом исходное `rd_data → nameProd` оказалось совершенно другим по наполненности и вынесено в отдельную text feature family T4.

---

# 2. Applicant

## A1 — `applicant_present`

**Status:** Rejected  
**Priority:** Low  
**Source:** `rd_data → applicant`

Первоначально наблюдалась сильная разница между TG, но после разбивки по `rd_type` выяснилось:

- `ДС` → Applicant обычно присутствует;
- `СС` → Applicant присутствует;
- `СГР` → Applicant отсутствует.

Следовательно, исходный TG signal в основном объясняется структурой типов РД.

**Вывод:** как standalone TG-specific feature отклонён. Может быть структурным признаком типа РД.

---

## A2 — `applicant_type`

**Status:** Candidate  
**Priority:** Medium  
**Source:** `rd_data → applicant.type`  
**Type:** categorical

Основные значения:
- `Юридическое лицо`;
- `Индивидуальный предприниматель`.

Для `ДС`:
- TG35: ИП ≈10.3%;
- TG43: ИП ≈3.4%.

Lift доли ИП относительно общей target-выборки:

| TG | IP share | Lift |
|---|---:|---:|
| TG35 | 8.32% | 1.02 |
| TG4 | 22.18% | 2.72 |
| TG43 | 3.36% | 0.41 |

Признак выглядит интересным для TG4 и как отрицательный сигнал для TG43, но пока не является validated feature.

---

# 3. Manufacturer

## M1 — `manufacturer_present`

**Status:** Rejected as redundant  
**Priority:** Low

На рабочем matched dataset `manufacturer_present` на 100% совпадает с `applicant_present`, поэтому отдельной новой информации не добавляет.

---

## M2 — `manufacturer_type`

**Status:** Candidate  
**Priority:** High  
**Source:** `rd_data → manufacturer.type`  
**Type:** categorical

Значения:
- `Индивидуальный предприниматель`;
- `Иностранное юридическое лицо`;
- `Юридическое лицо`;
- `Физическое лицо`;
- пропуск.

Для `ДС` распределение TG при фиксированном manufacturer type:

| Manufacturer type | TG35 | TG43 |
|---|---:|---:|
| ИП | 96.8% | 3.2% |
| Иностранное ЮЛ | 81.8% | 18.2% |
| Нет данных | 80.5% | 19.5% |
| Физическое лицо | 56.2% | 43.8% |
| ЮЛ | 78.9% | 21.1% |

Особенно интересна комбинация `rd_type == ДС AND manufacturer_type == ИП`.

**Ограничение:** это conditional distribution внутри matched target sample, а не production Precision.

---

## M3 — `applicant_type × manufacturer_type`

**Status:** To investigate  
**Priority:** High

Совместное распределение Applicant и Manufacturer различается, поэтому interaction может дать дополнительный сигнал.

**Next:** проверить support и conditional distributions, прежде всего внутри `ДС`.

---

# 4. Technical Regulations

## R1 — `tech_reg_codes`

**Status:** Candidate  
**Priority:** High  
**Source:** `rd_data → techRegulations`  
**Type:** multi-value categorical

### Coverage

`techRegulations` заполнен примерно у **81.5%** matched RD. После выделения идентификаторов регламентов:

- `ТР ТС 009/2011` — support ≈56 175 документов; TG35 ≈93.1%, TG4 ≈6.9%; coverage TG35 ≈76.6%, TG4 ≈98.1% внутри текущей target sample;
- `ТР ТС 030/2012` — support ≈13 508; TG43 ≈99.9%; coverage TG43 ≈96.0%.

`ТР ТС 009/2011` отражает широкую парфюмерно-косметическую область, поэтому сам по себе не является идеальным TG4/TG35 separator. Комбинация `TR009 + TNVED` выглядит перспективнее.

Другие регламенты имеют значительно меньший support и рассматриваются как secondary candidates.

**Ограничение:** цифры получены внутри matched target sample; полный target vs OTHER ещё не выполнен.

---

# 5. Declaration / Certificate

## D1 — `declaration.idDeclScheme`

**Status:** Candidate  
**Priority:** High  
**Evidence:** strong statistical

Основные значения:

| TG | Основной сигнал |
|---|---|
| TG4 | `3д` ≈96.0% |
| TG35 | `3д` ≈74.5%, `6д` ≈5.6% |
| TG43 | `2д` ≈93.3%, `4д` ≈3.1% |

Reverse distribution:
- `1д` → ~92.3% TG43;
- `2д` → ~71.5% TG43;
- `3д` → ~92.8% TG35;
- `4д` → ~72.6% TG35;
- `6д` → ~98.8% TG35.

`3д` нельзя использовать как самостоятельное TG4 rule, поскольку схема одновременно характерна для TG4 и TG35.

Проверка внутри `ДС` показывает, что схема сохраняет различия между TG35 и TG43, поэтому сигнал не объясняется только `rd_type`.

### Additional candidates

- `declaration.object_nonempty`;
- `declaration.idDeclType`;
- `declaration.duration_days`;
- производные признаки формата `declaration.number`.

### Certificate

По дополнительному исследованию certificate support очень мал:
- TG4: 0 заполненных документов;
- TG35: 267 (~0.39% matched TG35);
- TG43: 22 (~0.16%).

Поэтому `certificate` не считается надёжным общим feature family. `issueBasis` ещё слабее и для общего классификатора практически бесполезен.

---

# 6. Text

## T1 — Парфюмерные keywords / phrases

**Status:** Candidate  
**Priority:** Medium

Примеры: `духи`, `одеколон`, `туалетная вода`, `парфюмерная вода`, `perfume`, `parfum`.

Standalone keyword approach имеет низкий support и не должен превращаться в жёсткие правила без проверки контекста.

---

## T2 — Косметические keywords / phrases

**Status:** Candidate  
**Priority:** Medium

Примеры: `крем`, `шампунь`, `лосьон`, `маска`, `бальзам`.

Известный риск: отдельное слово `крем` может встречаться вне косметики. Поэтому контекст и n-grams предпочтительнее одиночного keyword rule.

---

## T3 — Motor-oil textual patterns

**Status:** Candidate  
**Priority:** Medium

Примеры:
- `моторное масло`;
- `масло моторное`;
- `API`, `SAE`;
- `5W`, `10W`;
- `ACEA`;
- допуски производителей (`VW`, `MB`, `Renault` и др.).

Нужна проверка support и false positives.

---

## T4 — `nameProd` lexical feature family

**Status:** Candidate  
**Priority:** High for residual TG35  
**Source:** `rd_data → nameProd`  
**Type:** text / lexical / n-gram

### Coverage

В полной matched target sample `nameProd` заполнен крайне неравномерно:
- TG4 — практически отсутствует / 0% в проверке;
- TG35 — ≈16.46%;
- TG43 — ≈0.90%.

Это означает, что `nameProd` не является универсальным текстовым полем для всех трёх TG. Однако на **residual TG35** после TNVED anchors он заполнен примерно у **94.56%** документов, поэтому является очень перспективным локальным источником признаков.

### Наблюдение residual

В residual встречаются информативные товарные конструкции, например:
- `мытья посуды`;
- `бытовой химии`;
- `окрашивания волос`;
- `крем краска`;
- `интимной гигиены`;
- `зубная паста`;
- `детский шампунь`;
- `парфюмерно косметическая`;
- `жидкое мыло`;
- `осветления волос`.

### Lexical candidate search

На train выделялись unigram и real-bigram кандидаты. При построении биграмм stop words удаляются **после сохранения исходного порядка**, чтобы не создавать искусственные фразы.

Первичный exploratory набор из 30 unigram + 30 bigram кандидатов покрывал значительную часть residual TG35. Более ранняя оценка на тех же данных показала высокое покрытие, но эта цифра не рассматривается как итоговое качество из-за selection bias.

### Validation protocol

Важная методологическая поправка: validation должен делиться **по всем target-документам до фильтрации по `nameProd`**. Иначе возникает selection bias: validation превращается в подвыборку документов с заполненным текстом и может иметь другое распределение TNVED.

В cleaned `02_feature_discovery.ipynb` split выполняется на document IDs всех target TG, после чего lexical candidates выбираются только на train и проверяются на validation.

**Текущий статус:** promising candidate family, но ещё не `Validated`.

### Next

- завершить корректную residual validation;
- проверить word + character n-grams;
- оценить false positives на TG43 и особенно target vs OTHER;
- проверить, какой вклад даёт text layer поверх TNVED.

---

# 7. Structural / interaction candidates

## S1 — JSON field presence

**Status:** To investigate

Presence/absence структурированных полей может быть полезен, но некоторые такие признаки являются proxy для `rd_type`. Каждый candidate требует confounder check.

## S2 — JSON structure patterns

**Status:** To investigate

Комбинации присутствующих/непустых полей могут быть полезны как derived features. Особый интерес представляют комбинации Product + Declaration + Manufacturer.

## S3 — `anchor_conflict`

**Status:** To investigate  
**Priority:** Medium

Обобщённый бинарный признак «у документа есть anchors нескольких target TG одновременно» потенциально отражает multi-label/ambiguous cases. Сейчас обнаружено 20 таких документов в TNVED anchor audit. Признак не должен использоваться как самостоятельный target rule, но может быть полезен для обработки неоднозначных случаев.

---

# 8. `tg_ids`

**Status:** Diagnostic only / weak-label source  
**Not a model feature**

Для residual TG35 (`11 900` документов):

- `35` присутствует в `tg_ids` у **96.55%**;
- `4` или `43` присутствует у **24.12%**;
- хотя бы одна non-target TG присутствует у **15.77%**.

Это подтверждает две вещи:

1. residual TG35 в целом не выглядит массовой ошибкой Truth;
2. multi-TG структура реальна, а `tg_ids` может содержать дополнительные noisy labels.

`tg_ids` не используется как feature из-за риска leakage и потому, что эксперт явно предупреждал о noisy nature этого поля.

---

# 9. Current priority

1. Завершить корректную validation TNVED + `nameProd` residual pipeline.
2. Проверить основные TNVED anchors против `OTHER`.
3. Провести системную Feature Evaluation для TNVED / TR / Declaration.
4. Проверить `nameProd` через word + character n-grams.
5. Исследовать interactions сильных структурных признаков.
6. Сформировать final target vs OTHER evaluation.
7. Только после этого присваивать `Validated`.

---

# 10. Главный принцип каталога

```text
Observation
    ↓
Statistic
    ↓
Comparison by TG
    ↓
Confounder check
    ↓
Candidate
    ↓
Train/Validation evaluation
    ↓
Target vs OTHER
    ↓
Validated / Rejected
```

**Ни один текущий Candidate не считается окончательно validated до Block 4.**
