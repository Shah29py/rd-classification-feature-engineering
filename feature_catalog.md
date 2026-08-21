# Case 7 — Feature Catalog

> Этот файл содержит **текущее состояние feature discovery**, а не историю всех промежуточных гипотез.
> История исследований и отвергнутых гипотез находится в `research_log.md`.

## Статусы

- `Candidate` — потенциально полезный признак, требует Feature Evaluation.
- `Validated` — статистически подтверждённая полезность после Block 4.
- `Rejected` — исследован, но не показал достаточной разделяющей способности или является redundant.
- `To investigate` — направление ещё недостаточно исследовано.

Дополнительно можно использовать **Priority: High / Medium / Low** как исследовательский приоритет. Priority не равен финальному статусу.

---

# 1. Product

## P1 — `product_object_type`

**Status:** Candidate  
**Priority:** Medium  
**Source:** `rd_data → product.idObjectType`  
**Type:** categorical

Основные значения:
- `Серийный выпуск`
- `Партия`
- `Единичное изделие`

Наблюдения:
- TG4: 97.59% серийный выпуск, 2.39% партия;
- TG35: 82.37% серийный выпуск, 1.15% партия, 16.47% нет данных;
- TG43: 94.78% серийный выпуск, 4.30% партия.

После контроля `rd_type` часть различий сохраняется, но признак не выглядит столь сильным, как TNVED.

**Ограничение:** для `СГР` отсутствие значения практически определяется структурой типа РД.

**Next:** при необходимости оценить как secondary structural feature.

---

## P2 — TNVED feature family

**Status:** Candidate  
**Priority:** High  
**Source:** `rd_data → product.tnved`  
**Type:** categorical / multi-value / hierarchical code

### Coverage

- хотя бы один TNVED есть у **86.73%** matched документов;
- 71 381 документов имеют один код;
- 2 211 — два;
- 687 — три;
- встречаются списки до 279 кодов;
- основная масса отдельных кодов — 10-значные;
- встречаются 2-, 4-, 6- и 9-значные значения.

Документ рассматривается как множество TNVED-кодов и их префиксов, а не как одно значение.

### High-priority candidates

#### P2.1 — `has_tnved_3303`

**Status:** Candidate  
**Priority:** High

- Support: 3 848 документов;
- Coverage TG4: ≈97.0%;
- Purity среди target TG: ≈99.2%.

Интерпретация: очень сильный candidate для TG4.

#### P2.2 — `has_tnved_3304`

**Status:** Candidate  
**Priority:** High

- Support: 28 659;
- Coverage TG35: ≈41.8%;
- Purity среди target TG: ≈99.8%.

#### P2.3 — `has_tnved_3305`

**Status:** Candidate  
**Priority:** High

- Support: 12 318;
- Coverage TG35: ≈18.0%;
- Purity среди target TG: ≈100%.

#### P2.4 — `has_tnved_3401`

**Status:** Candidate  
**Priority:** High

- Support: 8 482;
- Coverage TG35: ≈12.4%;
- Purity среди target TG: ≈99.8%.

#### P2.5 — `has_tnved_2710`

**Status:** Candidate  
**Priority:** High

- Support: 8 451;
- Coverage TG43: ≈60.1%;
- Purity среди target TG: ≈100%.

#### P2.6 — `has_tnved_3403`

**Status:** Candidate  
**Priority:** High

- Support: 5 703;
- Coverage TG43: ≈40.5%;
- Purity среди target TG: ≈100%.

### Important limitation

Purity рассчитана внутри текущей matched target-выборки 4/35/43. Она **не является production Precision**, поскольку полная выборка OTHER пока не добавлена.

### Required validation

- проверить коды по `TG_Definitions.xlsx`;
- проверить stability;
- проверить target vs OTHER;
- оценить взаимодействия нескольких TNVED-признаков;
- определить, нужны ли уровни 2/4/6/10 знаков.

---

## P3 — `product_origin`

**Status:** Candidate  
**Priority:** Medium  
**Source:** `rd_data → product.idProductOrigin`  
**Type:** categorical

Примеры значений: Россия, Китай, Италия, Германия, Франция и др.

На текущем этапе закономерность по TG ещё не оценена системно.

**Next:** coverage, top-N по TG, reverse distribution и объединение редких категорий.

---

## P4 — `product_name` / `product_info`

**Status:** To investigate  
**Priority:** Medium

Текстовые поля продукта перспективны, но их исследование отложено до завершения структурных признаков и TNVED validation.

---

# 2. Applicant

## A1 — `applicant_present`

**Status:** Rejected  
**Priority:** Low  
**Source:** `rd_data → applicant`

В общей выборке сначала наблюдалось сильное различие между TG, но после контроля `rd_type` выяснилось, что наличие Applicant практически определяется типом РД:

- ДС → присутствует;
- СС → присутствует;
- СГР → отсутствует.

**Вывод:** как standalone TG-specific feature не подтверждён. Может быть структурным признаком типа РД.

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
| 35 | 8.32% | 1.02 |
| 4 | 22.18% | 2.72 |
| 43 | 3.36% | 0.41 |

**Интерпретация:** потенциально полезен для TG4 и как отрицательный сигнал для TG43; для TG35 близок к базовому уровню.

**Required validation:** support, stability, target vs OTHER, interaction with manufacturer type.

---

# 3. Manufacturer

## M1 — `manufacturer_present`

**Status:** Rejected  
**Priority:** Low  
**Reason:** 100% совпадает с `applicant_present` на рабочем matched dataset.

Отдельное использование не даёт новой информации.

---

## M2 — `manufacturer_type`

**Status:** Candidate  
**Priority:** High  
**Source:** `rd_data → manufacturer.type`  
**Type:** categorical

Основные значения:
- `ИП`
- `Иностранное юридическое лицо`
- `Юридическое лицо`
- `Физическое лицо`
- пропуск

Для `ДС` наблюдается:

| Manufacturer type | TG35 | TG43 |
|---|---:|---:|
| ИП | 96.8% | 3.2% |
| Иностранное ЮЛ | 81.8% | 18.2% |
| Нет данных | 80.5% | 19.5% |
| Физическое лицо | 56.2% | 43.8% |
| ЮЛ | 78.9% | 21.1% |

Особенно интересна комбинация:

`rd_type == ДС AND manufacturer_type == ИП`.

**Important:** эти значения — conditional distribution внутри matched target sample, а не финальный Precision модели.

---

## M3 — `applicant_type × manufacturer_type`

**Status:** To investigate  
**Priority:** High

`applicant_type` и `manufacturer_type` не являются взаимозаменяемыми. Их совместное распределение различается, поэтому комбинации могут дать дополнительную информацию.

**Next:** проверить support и conditional distribution для комбинаций, особенно внутри `ДС`.

---

# 4. Text

## T1 — perfume keyword / phrase features

**Status:** Candidate  
**Priority:** Medium

Примеры:
- `духи`
- `одеколон`
- `туалетная вода`
- `парфюмерная вода`
- `perfume`
- `parfum`

Первичная проверка показала низкое количество совпадений по отдельным словам, поэтому одиночные keywords не считаются доказанными сильными признаками.

**Next:** контекст, морфология, n-grams, phrase patterns, false positives.

## T2 — cosmetics keyword / phrase features

**Status:** Candidate  
**Priority:** Medium

Примеры: `крем`, `шампун`, `лосьон`, `маска`, `бальзам`.

Важный риск: слово `крем` встречалось и вне косметики, например в названиях пищевых продуктов.

**Next:** контекст и комбинации, а не standalone keyword.

## T3 — motor oil specification features

**Status:** Candidate  
**Priority:** Medium

Примеры:
- `моторное масло`
- `масло моторное`
- `API`
- `SAE`
- `5W`, `10W`
- `ACEA`
- допуски `VW`, `MB`, `Renault` и др.

Первичная проверка отдельных слов дала мало совпадений. Нужен анализ паттернов и комбинаций.

---

# 5. Structural / interaction candidates

## S1 — `json_field_presence`

**Status:** To investigate

Идея: presence/absence структурированных полей может быть информативна. Однако Applicant/Manufacturer показали, что часть таких признаков является proxy для `rd_type`, поэтому каждый field-presence candidate нужно проверять на confounders.

## S2 — `json_structure_pattern`

**Status:** To investigate

Комбинации присутствующих полей могут быть характерны для отдельных типов РД и TG, но их нужно проверять, чтобы не получить proxy для `rd_type`.

---

# 6. Правило добавления новых features

Для каждого нового кандидата фиксируем:

```text
## Feature X

Source:
Type:
Status:
Priority:

Observation:
Support:
Coverage:
TG 4:
TG 35:
TG 43:
Specificity:
Confounders:
OTHER:
Limitations:
Next validation:
```

Главный принцип каталога:

```text
идея
  ↓
наблюдение
  ↓
статистика
  ↓
сравнение TG
  ↓
confounder check
  ↓
Candidate
  ↓
Feature Evaluation
  ↓
Validated / Rejected
```

---

# 7. Текущий приоритет

1. **TNVED definitions validation** по `TG_Definitions.xlsx`.
2. Проверка TNVED-кандидатов против `OTHER`.
3. `product.idProductOrigin`.
4. Certificate / Declaration.
5. Текстовые признаки.
6. Комбинации сильных кандидатов.
7. Block 4 — системная Feature Evaluation.

До завершения Block 4 никакой candidate feature не считается окончательно validated.
