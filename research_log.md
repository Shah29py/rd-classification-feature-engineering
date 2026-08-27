# Case 7 — Research Log

## 1. Цель проекта

**Case 7 — Feature Engineering для классификатора товарной группы Разрешительных Документов (RD).**

Целевые товарные группы:
- **TG 4 — Парфюмерия**;
- **TG 35 — Косметика, бытовая химия и товары личной гигиены**;
- **TG 43 — Моторные масла**.

Главная цель — исследовать RD dataset и найти признаки, позволяющие определить принадлежность разрешительного документа к одной или нескольким целевым TG.

### Уточнения эксперта

- объект классификации — разрешительный документ;
- один документ может относиться к нескольким TG;
- модель должна определять только TG 4, 35 и 43;
- остальные документы допускается обозначать `0/-1`;
- можно и нужно использовать все доступные поля JSON;
- `tg_ids` можно использовать для сверки, но в нём есть ошибки, поэтому это auxiliary / weak-label source;
- ориентир из материалов проекта: средний Recall ≥97%, Precision ≥40% по трём target TG.

---

# 2. План исследования

- [x] Блок 0. Подготовка проекта
- [x] Блок 1. Data Understanding / EDA
- [x] Блок 2. RD ↔ Truth и анализ разметки
- [x] Блок 3. Основной Feature Discovery для Product / Applicant / Manufacturer / TNVED / TR / Declaration / Text candidates
- [ ] Блок 4. Feature Evaluation
  - [ ] Coverage
  - [ ] Specificity / discriminative power
  - [ ] Stability
  - [ ] Target vs OTHER
  - [ ] Interactions
  - [ ] Final feature catalog
- [ ] Блок 5. Optional ML
- [ ] Блок 6. Финальная презентация

---

# 3. Data Understanding

## 3.1 Dataset

Основной RD dataset:

- `case_df.shape = (3 603 517, 4)`;
- поля: `rd_documentnumber`, `rank`, `rd_data`, `tg_ids`.

Truth содержит экспертную разметку и атрибуты документа. `TG_Definitions.xlsx` содержит определения и справочники, которые используются как domain knowledge.

## 3.2 JSON

В `rd_data` встречаются поля:

`type`, `active`, `number`, `docNorm`, `useArea`, `nameProd`, `protocol`, `firmGetName`, `firmMadeName`, `statusGroup`, `activeFromDate` и др.

Ключевое наблюдение: наличие ключа не означает наличие полезного значения. Для derived features необходимо различать `object_present` и `object_nonempty`.

## 3.3 `rank`

Ранее обнаружено 7 714 повторяющихся `rd_documentnumber`.

Эксперт уточнил, что `rank` не следует трактовать как «версию документа». Для рабочего document-level слоя используется `rank == 1`.

Результат:
- 3 595 803 строки;
- 3 595 803 уникальных `rd_documentnumber`;
- дубликатов по номеру нет.

---

# 4. RD ↔ Truth

## 4.1 Truth

Обновлённый Truth:
- исходно 577 260 строк;
- после удаления полных дубликатов 105 159 строк;
- 97 822 уникальных `rd_number`.

Truth агрегируется на document-level с сохранением множества TG.

По уникальным документам:
- 97 746 имеют одну TG;
- 76 имеют две TG.

Основные комбинации:
- `(35,)` — 77 341;
- `(43,)` — 16 496;
- `(4,)` — 3 909;
- `(35, 43)` — 43;
- `(35, 4)` — 32;
- `(4, 43)` — 1.

Следовательно, multi-label случаи реальны, хотя и относительно редки.

## 4.2 Match coverage

На уровне уникальных Truth-документов:

| TG | Truth docs | Matched | Coverage |
|---|---:|---:|---:|
| TG4 | 3 942 | 3 936 | 99.85% |
| TG35 | 77 416 | 68 370 | 88.32% |
| TG43 | 16 540 | 14 061 | 85.01% |

Итого matched documents: **86 301**.

Рабочий `base_df`:
- shape `(86 301, …)`;
- 86 301 уникальный `rd_documentnumber`;
- дубликатов `rd_documentnumber` нет.

## 4.3 Leakage rule

`truth_df.code_tnved` и другие признаки, существующие только в Truth, не используются как features.

Если аналогичная информация доступна непосредственно в `rd_data`, она может использоваться как feature. Пример: `rd_data → product.tnved`.

---

# 5. Экспертные уточнения и `tg_ids`

Эксперт отдельно подчеркнул:

- `tg_ids` можно использовать для сверки;
- в `tg_ids` встречаются ошибки;
- Truth остаётся главным источником экспертной разметки;
- один RD может иметь несколько TG;
- модель должна предсказывать только 4/35/43, остальные можно относить к `0/-1`.

В материалах проекта для `tg_ids` приведены ориентиры около Recall 97% и Precision 40% по трём target TG, поэтому это следует считать **noisy / weak labels**, а не gold truth.

---

# 6. Block 3 — Applicant

## 6.1 `applicant`

Типы:
- `dict` — 76 684;
- `None` — 9 617.

Заполненность основных полей в непустом Applicant:
- `address` — 74 799;
- `fullName` — 74 469;
- `type` — 72 775;
- `ogrn` — 72 480;
- `directorName` — 72 339;
- `inn` — 72 183;
- `email` — 72 093;
- `phone` — 71 890.

Типы Applicant:
- `Юридическое лицо` — 65 746;
- `Индивидуальный предприниматель` — 7 029.

## 6.2 `applicant_present`

Первоначальный общий TG signal оказался confounded `rd_type`-структурой:
- `ДС` → Applicant присутствует;
- `СС` → присутствует;
- `СГР` → отсутствует.

**Вывод:** `applicant_present` отклонён как standalone TG-specific feature.

## 6.3 `applicant_type`

Для `ДС`:
- TG35: ИП ≈10.3%;
- TG43: ИП ≈3.4%.

Общий target sample:

| TG | IP share | Lift |
|---|---:|---:|
| TG35 | 8.32% | 1.02 |
| TG4 | 22.18% | 2.72 |
| TG43 | 3.36% | 0.41 |

**Статус:** Candidate.

---

# 7. Block 3 — Manufacturer

## 7.1 Структура

`manufacturer` имеет ту же общую схему присутствия, что и Applicant.

Основные типы:
- `Индивидуальный предприниматель`;
- `Иностранное юридическое лицо`;
- `Юридическое лицо`;
- `Физическое лицо`;
- пропуски.

## 7.2 `manufacturer_present`

На рабочем matched dataset `manufacturer_present` на 100% совпадает с `applicant_present`.

**Вывод:** redundant → Rejected.

## 7.3 `manufacturer_type`

Для `ДС`:

| Manufacturer type | TG35 | TG43 |
|---|---:|---:|
| ИП | 96.8% | 3.2% |
| Иностранное ЮЛ | 81.8% | 18.2% |
| Нет данных | 80.5% | 19.5% |
| Физическое лицо | 56.2% | 43.8% |
| ЮЛ | 78.9% | 21.1% |

Наиболее интересна комбинация `rd_type == ДС AND manufacturer_type == ИП`.

**Статус:** Candidate, Priority High.

Дополнительное направление: interaction `applicant_type × manufacturer_type`.

---

# 8. Block 3 — Product object type

`product.idObjectType`:
- `Серийный выпуск` — доминирует во всех TG;
- `Партия` чаще встречается у TG43;
- отсутствие значения особенно заметно у TG35.

Распределение:

| TG | Серийный выпуск | Партия | Единичное изделие | Нет данных |
|---|---:|---:|---:|---:|
| TG35 | 82.37% | 1.15% | 0.01% | 16.47% |
| TG4 | 97.59% | 2.39% | 0.03% | ~0% |
| TG43 | 94.78% | 4.30% | 0.02% | 0.90% |

После контроля `rd_type` для `ДС`:
- TG35: 98.61% серийный выпуск, 1.38% партия;
- TG43: 95.63% серийный выпуск, 4.35% партия.

**Статус:** Candidate / Secondary.

---

# 9. Block 3 — TNVED

## 9.1 Structure

`product.tnved` заполнен хотя бы одним кодом у **86.73%** matched RD.

Количество кодов:

| Код count | Документы |
|---:|---:|
| 0 | 11 449 |
| 1 | 71 381 |
| 2 | 2 211 |
| 3 | 687 |
| 4+ | редкие случаи, включая длинные списки до 279 |

Длины отдельных кодов: 2, 4, 6, 9 и 10 символов, основная масса — 10 знаков.

## 9.2 Main statistical anchors

| TNVED4 | Support | Coverage основной TG | Purity в target sample |
|---|---:|---:|---:|
| 3303 | 3 848 | TG4 ~97.0% | ~99.2% |
| 3304 | 28 659 | TG35 ~41.8% | ~99.8% |
| 3305 | 12 318 | TG35 ~18.0% | ~100% |
| 3401 | 8 482 | TG35 ~12.4% | ~99.8% |
| 2710 | 8 451 | TG43 ~60.1% | ~100% |
| 3403 | 5 703 | TG43 ~40.5% | ~100% |

## 9.3 Definitions validation

Сверка с `TG_Definitions.xlsx` подтвердила:
- 3303 → Парфимерия;
- 3304/3305/3401 → Косметика;
- 2710/3403 → Моторные масла.

Дополнительные TG35 кандидаты:
- 3306;
- 3307;
- 3402;
- 3808.

`3306/3307/3402` подтверждены Definitions как косметические группы. `3808` не найден в Definitions и поэтому считается **statistical-only candidate**.

## 9.4 Residual strategy

Изначальные TG35 anchors `3304/3305/3401` давали около 72.0% coverage.

После добавления `3306/3307/3402/3808`:
- TG35 coverage ≈ **82.59%**;
- residual = **11 900 TG35 документов**.

Без `3808`:
- coverage ≈ **81.05%**;
- residual был больше.

Следовательно, `3808` даёт реальный statistical gain, но его domain evidence слабее.

## 9.5 Anchor conflicts

На document-level найдено **20 документов** с anchors двух target TG одновременно. Большинство — TG4/TG35.

Ручная проверка показывает, что multi-code документы действительно могут содержать товары нескольких target categories.

**Вывод:** TNVED anchors — evidence, а не жёсткие mutually-exclusive rules.

---

# 10. Block 3 — Product origin

`product.idProductOrigin` заполнен примерно у **75.0%** matched документов.

Наиболее частые страны:
- Россия — 34 844;
- Корея — 5 879;
- Франция — 5 362;
- Италия — 3 456;
- Германия — 2 985;
- Япония — 1 787;
- Китай — 1 566;
- Испания — 1 426;
- Турция — 1 363;
- США — 1 105.

Пример различий по TG:

- TG4: выше доля Франции и Испании;
- TG35: высокая доля России и Кореи;
- TG43: заметнее Германия и Япония.

Дополнительная таблица `origin × manufacturer_type` показала сильную зависимость: для Франции, Германии, Италии и Японии более 93% документов имеют `manufacturer_type = Иностранное юридическое лицо`.

**Вывод:** Candidate / Secondary; вероятна частичная избыточность относительно Manufacturer.

---

# 11. Block 3 — Technical Regulations

`techRegulations` заполнен примерно у 81.5% matched RD.

После извлечения регламентов:

### `ТР ТС 009/2011`

- support ≈56 175;
- TG35 ≈93.1%;
- TG4 ≈6.9%;
- coverage TG4 ≈98.1%;
- coverage TG35 ≈76.6%.

### `ТР ТС 030/2012`

- support ≈13 508;
- TG43 ≈99.9%;
- coverage TG43 ≈96.0%.

`ТР ТС 009/2011` отражает широкую парфюмерно-косметическую область и лучше работает в комбинации с TNVED.

**Статус:** Candidate, Priority High.

Другие регламенты имеют меньший support.

---

# 12. Block 3 — Declaration / Certificate

## 12.1 Declaration

Заполненная декларация встречается у:
- TG4 — 3 936 документов;
- TG35 — 56 841;
- TG43 — 13 913.

### `idDeclScheme`

Основные распределения:
- TG4: `3д` ≈96.0%;
- TG35: `3д` ≈74.5%, `6д` ≈5.6%;
- TG43: `2д` ≈93.3%, `4д` ≈3.1%.

Reverse distribution:
- `1д` → ~92.3% TG43;
- `2д` → ~71.5% TG43;
- `3д` → ~92.8% TG35;
- `4д` → ~72.6% TG35;
- `6д` → ~98.8% TG35.

Проверка внутри `ДС` показывает, что различия между TG35 и TG43 сохраняются.

**Статус:** Candidate, Priority High.

Дополнительные candidates:
- `declaration.object_nonempty`;
- `declaration.idDeclType`;
- длительность `declEndDate - declRegDate`;
- производные признаки номера.

## 12.2 Certificate

Дополнительное исследование показало крайне низкий support:

| TG | Заполненный certificate | Доля |
|---|---:|---:|
| TG4 | 0 | 0% |
| TG35 | 267 | ~0.39% |
| TG43 | 22 | ~0.16% |

`issueBasis` ещё слабее:
- TG4: 0;
- TG35: 164 (~0.24%);
- TG43: 4 (~0.03%).

**Вывод:** certificate family не является надёжным общим feature source. Связь с `rd_type` понятна: `СС` чаще связано с certificate, `ДС`/`N/A` — с declaration, `СГР` часто имеет пустые вложенные объекты.

---

# 13. Block 3 — Residual TG35 audit

После расширенного TNVED anchor family:

**Residual TG35 = 11 900 документов.**

Для residual после всех TNVED anchors coverage structured fields оказался низким:

| Field | Coverage residual |
|---|---:|
| `product_object_type` | 5.36% |
| `manufacturer_type` | 5.27% |
| `applicant_type` | 5.27% |
| `declaration scheme` | ~5.24% |
| `product_origin` | 4.76% |
| `tech_reg_codes` | 4.08% |

Таким образом, уже исследованные structured fields не закрывают основную часть residual.

---

# 14. `tg_ids` sanity check на residual

Для 11 900 residual TG35:

- `35` присутствует в `tg_ids` у **96.55%**;
- `4` или `43` присутствует у **24.12%**;
- non-target TG присутствует у **15.77%**.

Интерпретация:

1. residual TG35 в целом не выглядит массовой ошибкой Truth;
2. multi-TG документы и noisy auxiliary labels действительно присутствуют;
3. `tg_ids` полезен как diagnostic source, но не как feature.

---

# 15. Text Discovery — `nameProd`

## 15.1 Почему `product_name` не стал основным источником

Derived fields:
- `product_name` на residual TG35 заполнен примерно у 5.36%;
- `product_info` реально непуст примерно у 2.0%.

Поэтому переход сделан на исходное `rd_data → nameProd`.

## 15.2 Coverage `nameProd`

В matched target sample:
- TG4 — 0% в текущей проверке;
- TG35 — ~16.46%;
- TG43 — ~0.90%.

Но на residual TG35:
- `nameProd` заполнен примерно у 97.71%;
- действительно непуст — примерно у **94.56%**.

Это сильный аргумент в пользу `nameProd` как **residual specialist source**, хотя не как универсального поля для всех TG.

## 15.3 Term / phrase discovery

В residual наиболее частые содержательные лексемы включали:

- `косметическая`;
- `гель`;
- `крем`;
- `волос`;
- `шампунь`;
- `пилинг`;
- `интимной`;
- `моющее`;
- `дезинфицирующее` и др.

Были построены document-support таблицы для unigram и real-bigram candidates.

У сильных биграмм:
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

## 15.4 Почему bigram construction была исправлена

Первоначальный вариант сначала удалял stop words, а затем строил биграммы. Это могло создавать искусственные соседства слов, которых не было в исходной фразе.

Корректный вариант:
1. сохранить исходный порядок токенов;
2. сформировать реальные соседние пары;
3. после этого удалить пары, содержащие стоп-слова.

Эта логика теперь используется в cleaned notebook.

## 15.5 Preliminary text coverage

На exploratory residual subset набор из 30 unigram + 30 bigram candidates давал высокое покрытие residual TG35. Однако эта оценка не считается final metric, потому что признаки были отобраны на том же наборе.

## 15.6 Validation methodology correction

Первая ручная версия validation строилась от документов, у которых уже был непустой `nameProd`. Это создавало selection bias.

Корректная схема:

```text
all target document IDs
        ↓
train / validation split
        ↓
text candidates fit only on train
        ↓
validation
        ↓
residual relative to TNVED
        ↓
text coverage
```

Во время ручной реализации возникла техническая ошибка: TNVED в validation оказался пустым у всех документов из-за использования неподходящей промежуточной таблицы/состояния notebook. Поэтому полученное тогда `TNVED coverage = 0%` признано **недействительным**.

Cleaned `02_feature_discovery.ipynb` пересобран так, чтобы:
- split делался по всем target documents;
- TNVED подтягивался напрямую из `base_df`;
- anchor TG хранились как множество;
- был assertion, запрещающий продолжение при нулевой validation TNVED coverage.

**Текущий статус text family:** Candidate / promising, not validated.

---

# 16. Current technical correction — notebook reproducibility

В ходе работы было обнаружено несколько классов повторяющихся ошибок:

- использование старых копий exploded tables;
- duplicate columns после повторного extraction;
- Arrow dtype при `value_counts()` / `mean()`;
- попытка делать `one_to_one` merge после `explode` без контроля уникальности;
- смешивание document-level и exploded-level таблиц;
- selection bias при создании validation только из документов с заполненным `nameProd`.

Из-за этого `02_feature_discovery` был полностью пересобран в cleaned notebook с единым extraction и sanity checks.

---

# 17. Итог текущего Feature Discovery

## Сильнейшие направления

### 1. TNVED

Самый сильный на текущем этапе source of structured evidence:
- high support;
- высокая purity внутри target sample;
- согласование с `TG_Definitions.xlsx`;
- явные target-specific 4-digit anchors.

### 2. Technical Regulations

Особенно сильны `ТР ТС 009/2011` и `ТР ТС 030/2012`, причём второй очень специфичен для TG43.

### 3. Declaration scheme

`idDeclScheme` даёт сильный structured signal, особенно между TG35 и TG43 внутри `ДС`.

### 4. `nameProd`

Не универсальный text field, но потенциально очень сильный residual specialist для TG35.

## Вторичные кандидаты

- `manufacturer_type`;
- `applicant_type`;
- `product_object_type`;
- `product_origin`.

## Отклонено

- `applicant_present` как standalone TG feature;
- `manufacturer_present` как redundant feature;
- certificate / `issueBasis` как общий feature source из-за малого support.

---

# 18. Следующий этап — Block 4

Приоритет:

1. корректно завершить train/validation pipeline для TNVED + text;
2. провести target vs OTHER для ключевых features;
3. оценить stability;
4. оценить interactions;
5. определить финальные feature families;
6. сформировать validated / rejected statuses.

До завершения Block 4 никакой текущий Candidate не должен называться окончательно validated.
