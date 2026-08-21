# Make No-Code MVP

## Статус

Ручной no-code-прототип AI Vacancy Fit Analyzer собран и фактически протестирован в Make 20 августа 2026 года.

Прототип подтверждает маршрутизацию входных данных, блокировку дорогого AI-шага при отсутствии обязательного input и создание понятного fallback-сообщения. Это не готовый пользовательский сервис: Telegram trigger, автоматическое извлечение вакансии, доставка результата и постоянное хранилище пока не реализованы.

Сценарий во время тестирования оставался `Inactive`; расписание было выключено.

## Цель MVP

Проверить минимальную архитектуру:

```text
вакансия + данные кандидата
→ техническая проверка двух inputs
→ AI-анализ только при заполненных inputs
→ fallback без AI при отсутствии хотя бы одного input
```

## Инструменты

- Make;
- `Tools → Set multiple variables`;
- `Flow Control → Router`;
- `Tools → Set variable`;
- `Make AI Toolkit → Simple Text Prompt`;
- `Make's AI Provider (default)`;
- модель `Recommended: Small`.

Внешний API-ключ не использовался.

## Реализованная схема

```text
Manual run
    ↓
Tools 2 — Set multiple variables
    ↓
Router
    ├─ Route 1: Missing input
    │      vacancy_text = emptystring
    │      OR
    │      candidate_data = emptystring
    │          ↓
    │      Tools 6 — Set variable
    │      error_message
    │
    └─ Route 2: Both inputs present
           vacancy_text ≠ emptystring
           AND
           candidate_data ≠ emptystring
               ↓
           Tools 3 — Set variable
           validation_status = inputs_received
               ↓
           Make AI Toolkit — Simple Text Prompt
```

Router расположен после создания входных переменных. Оба route-фильтра используют mapped tokens из `Tools 2`, а не напечатанные имена переменных.

## Входные данные теста

### `vacancy_text`

Синтетическая вакансия специалиста по внедрению:

- настройка сервиса;
- обучение пользователей;
- отчётность;
- Excel и коммуникация;
- зарплата 90 000 ₽;
- гибрид в Краснодаре;
- холодных звонков нет.

### `candidate_data`

Подтверждённые данные кандидата:

- координация команды;
- обучение сотрудников;
- клиенты и поставщики;
- учёт запасов и отчётность;
- Excel и iiko;
- самостоятельное изучение AI и публичные AI-проекты;
- желаемый доход от 80 000 ₽;
- Краснодар;
- офис, удалённо или гибрид;
- холодные звонки не рассматриваются.

Тестовые данные не содержали credentials.

## Filter semantics

В Make визуально незаполненный operand оказался не равен пустой строке. Первый аварийный тест показал, что условие `Not equal to` с незаполненным правым полем способно пропустить `candidate_data: empty`.

Исправление: использовать специальный keyword `emptystring`.

```text
Success:
vacancy_text Not equal to emptystring
AND
candidate_data Not equal to emptystring

Fallback:
vacancy_text Equal to emptystring
OR
candidate_data Equal to emptystring
```

Также важно выбирать mapped tokens `2.vacancy_text` и `2.candidate_data`. Напечатанные вручную строки `vacancy_text` и `candidate_data` не передают значения variables.

## Фактический тест 1 — отсутствуют данные кандидата

### Input

```text
vacancy_text: заполнен
candidate_data: empty
```

### History evidence

```text
Trigger: Manual
Duration: Less than a second
Operations: 2
Credits: 2
Data size: 733 B
```

### Журнал выполнения

```text
Tools 2 — completed
Tools 6 — completed
Tools 3 — bundle did not pass through the filter
Make AI Toolkit — отсутствует в журнале
```

### Output

`Tools 6` создал переменную `error_message`:

```text
Не хватает данных вакансии или кандидата. Полный AI-анализ не запущен. Пришлите недостающий текст, файл или читаемые скриншоты.
```

### Результат

`PASS`.

- fallback получил один bundle;
- `error_message` создан;
- успешная ветка заблокирована;
- AI не запускался;
- AI-token credits не списывались.

## Фактический тест 2 — оба input заполнены

### Input

```text
vacancy_text: заполнен
candidate_data: заполнен
```

### History evidence

```text
Trigger: Manual
Duration: 5 seconds
Operations: 3
Credits: 3.31
Data size: 5.4 KB
```

### Журнал выполнения

```text
Tools 2 — completed
Tools 6 — bundle did not pass through the filter
Tools 3 — completed
Make AI Toolkit — Simple Text Prompt — completed
```

### AI operation

```text
Operations: 1
Credits: 1.31
Data size: 4.4 KB
Output: Answer — Long String
```

Стоимость AI-модуля в инспекторе была разделена на `1 credit` за operation и `0.31 credits` за AI tokens.

### Результат маршрутизации

`PASS`.

- success route получил один bundle;
- fallback не выполнился;
- AI-модуль запустился ровно один раз;
- текстовый `Answer` создан.

## Семантический QA AI-ответа

**Статус:** `CONDITIONAL PASS`.

Ответ правильно:

- признал данные достаточными;
- сопоставил Краснодар, гибрид, Excel и зарплату;
- не придумал холодные звонки;
- отделил отсутствие прямого внедрения от отсутствия входных данных;
- отметил обучение сотрудников и отчётность как переносимый опыт.

Оставшиеся дефекты:

- повторный запрос примеров проектов внедрения, которых нет в подтверждённых данных;
- противоречие: гибрид в Краснодаре сначала признан совпадением, затем необоснованно поставлен под сомнение;
- преждевременное решение `можно принимать`, хотя реального предложения ещё нет;
- служебные и неестественные формулировки: `exactly`, `deployment`, `переносимое право`;
- повторение элементов output-шаблона вместо чистого пользовательского ответа.

Технический успех MVP не означает production-качество текста. Перед использованием для реальных карьерных решений промпт и модель требуют дополнительного QA.

## Сравнение моделей

| Модель | Reasoning | Время | Operations | Credits | Ручной QA |
|---|---|---:|---:|---:|---|
| Small (`gpt-5-nano`) | minimal | 5 секунд | 3 | 3.31 | FAIL |
| Medium (`gpt-5-nano`) | low | 12 секунд | 3 | 3.97 | FAIL, но основные вердикты лучше |

Medium стоила на `0.66 credits` больше Small и работала в `2.4 раза` дольше. Улучшение вердиктов не устранило выдуманные навыки, неправильные типы доказательств и завышенную самооценку QA.

## Детерминированный QA gate

После AI добавлен второй Router без дополнительного AI-вызова:

```text
AI Answer
→ QA Router
   ├─ Known QA violation
   │     Answer contains один из известных паттернов
   │         ↓
   │     qa_status = FAIL
   │     qa_message = ответ заблокирован
   │
   └─ fallback: Manual review required
         ↓
      qa_status = MANUAL_REVIEW
      qa_message = semantic PASS не подтверждён
```

Production-паттерны:

```text
junior
стажиров
ментор
примеры проектов внедрения
+ причина + следующий шаг
можно принимать /
```

Одна фраза хранится в одном OR-условии. Gate использует mapped `Answer` из Make AI Toolkit и оператор `Contains (case insensitive)`.

### MANUAL_REVIEW test

```text
Duration: 11 seconds
Operations: 4
Credits: 4.85
Data size: 4.3 KB
```

`Known QA violation` был заблокирован, fallback выполнился, `Tools 13` вернул `qa_status = MANUAL_REVIEW`.

### FAIL test

Для контролируемого теста временно добавлялся паттерн `Статус данных`, гарантированно присутствующий в формате ответа.

```text
Duration: 13 seconds
Operations: 4
Credits: 4.83
Data size: 3.8 KB
```

`Known QA violation` выполнился, `Tools 12` вернул `qa_status = FAIL`, fallback не запускался. После теста временный паттерн удалён.

### Граница защиты

Детерминированный filter умеет обнаруживать только перечисленные паттерны. Отсутствие совпадения не доказывает смысловую корректность, поэтому автоматический `PASS` запрещён. Все остальные ответы получают `MANUAL_REVIEW`.

## Не реализовано

- Telegram trigger и webhook;
- автоматическое открытие ссылки на вакансию;
- извлечение HTML или текста страницы;
- PDF/OCR;
- загрузка резюме;
- доставка результата пользователю;
- Google Docs или другое постоянное хранилище;
- retry сетевых ошибок;
- журналирование без персональных данных;
- end-to-end тест от пользовательского сообщения до доставки результата.

## Следующие шаги

1. Улучшить semantic QA без доверия к self-reported `QA: PASS` генератора.
2. Добавить чтение ссылок и файлов вакансий с отдельной проверкой извлечённого текста.
3. Добавить разбиение слишком длинных Telegram-ответов.
4. Решить, где хранить профиль кандидата и историю анализов.
5. Принять решение об активации Telegram-сценария после оценки стоимости и процесса ручной проверки.

## Telegram delivery layer

Ручной Make MVP был клонирован в отдельный Telegram-сценарий. Добавлены:

- `Telegram Bot — Watch Updates` как instant trigger;
- mapping `Message.Text → vacancy_text`;
- входной фильтр `Message.Text Not equal to emptystring`;
- отдельный command / vacancy Router: команды `/...` идут в welcome, обычный текст — в защищённую AI-ветку;
- Telegram send modules после `Missing input`, `FAIL` и `MANUAL_REVIEW`;
- динамический `Chat ID` из `Message.Chat.ID`.

Первый `/start`-тест был заблокирован до AI. После добавления welcome-ветки повторный `/start` выполнил trigger, input Tools и Telegram delivery за менее чем секунду: `3 operations`, `3 credits`, `1.7 KB`; AI отсутствовал в логе.

Реальный текст вакансии прошёл до fallback `MANUAL_REVIEW` и был доставлен в исходный Telegram chat. Метрики:

```text
11 seconds
6 operations
6.94 credits
8.7 KB
```

Отдельный Telegram missing-input run выполнил trigger, входные переменные, error Tools и отправку сообщения за менее чем секунду: `4 operations`, `4 credits`, `1.3 KB`; AI отсутствовал в execution log. После теста `candidate_data` восстановлен без нового запуска.

Контролируемый Telegram FAIL run на текущей Router-архитектуре выполнил trigger, input Tools, success marker, AI, FAIL Tools и Telegram block delivery: `17 seconds`, `6 operations`, `6.85 credits`, `5.7 KB`. `MANUAL_REVIEW` не выполнялся, а AI Answer не отправлялся пользователю. Временный паттерн `Статус данных` после фиксации доказательств удалён; filter и scenario сохранены без повторного запуска.

Полный поток и ограничения описаны в [`TELEGRAM_MVP.md`](TELEGRAM_MVP.md). Сценарий после теста оставлен `Inactive`.

## Owner allowlist и minimum-length guard

Перед input Tools входной фильтр расширен без публикации literal Chat ID:

```text
Message.Text Not equal to emptystring
AND Message.Chat.ID Equal to owner allowlist literal
```

Current Router filters:

```text
AI route:
  both inputs present
  AND vacancy_text Does not start with "/"
  AND vacancy_text Matches ^[\s\S]{100,}$

welcome route:
  vacancy_text Starts with "/"
  OR vacancy_text Does not match ^[\s\S]{100,}$
```

Положительный owner-path и short-text guard подтверждены сообщением `Привет`: менее секунды, `3 operations`, `3 credits`, `1.7 KB`; Telegram welcome выполнен, AI отсутствовал. Отклонение чужого Chat ID пока не тестировалось.

Длинная вакансия прошла guarded AI path: `9 seconds`, `6 operations`, `6.89 credits`, `8.9 KB`; выполнилась только delivery `MANUAL_REVIEW`. Технический результат — `PASS`, внешний semantic QA AI Answer — `FAIL`. Сценарий оставлен `Inactive`.
