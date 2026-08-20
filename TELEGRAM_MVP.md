# Telegram MVP

## Статус

Telegram-версия сценария собрана как отдельная копия проверенного ручного Make MVP и протестирована реальным сообщением с текстом вакансии.

Сценарий после теста оставлен `Inactive`; instant scheduling выключен.

## Пользовательский поток

```text
Telegram Bot — Watch Updates
  ↓
Filter: Text messages only
  Message.Text Not equal to emptystring
  ↓
Tools — Set multiple variables
  vacancy_text = Telegram Message.Text
  candidate_data = сохранённый профиль кандидата
  ↓
Input Router
  ├─ Missing input
  │    → error_message
  │    → Telegram: сообщение об ошибке
  └─ Command / vacancy Router
       ├─ vacancy_text Starts with "/"
       │    → Telegram: приветствие без AI
       └─ оба input заполнены
          AND vacancy_text Does not start with "/"
            → validation_status
            → Make AI Toolkit
            → QA Router
               ├─ Known QA violation
               │    → qa_status = FAIL
               │    → Telegram: только сообщение о блокировке
               └─ fallback: Manual review required
                    → qa_status = MANUAL_REVIEW
                    → Telegram: предупреждение + qa_message + AI Answer
```

Автоматического `PASS` в Telegram-потоке нет.

## Защита от команд и приветствие

Первый защитный тест блокировал `/start` перед `Tools` и AI: trigger получил один update, filter пропустил `0` bundles, downstream не запускался. После этого схема была расширена пользовательской welcome-веткой.

Текущая защита состоит из двух фильтров:

```text
вход:
Message.Text Not equal to emptystring

AI route:
оба input заполнены
AND vacancy_text Does not start with "/"

command route:
vacancy_text Starts with "/"
```

Реальный `/start` прошёл только через trigger, входные переменные и Telegram welcome delivery. AI отсутствовал в execution log.

```text
Trigger: Manual
Duration: less than a second
Operations: 3
Credits: 3
Data size: 1.7 KB
```

Бот доставил приветствие с инструкцией прислать текст вакансии и пояснением статусов `FAIL` и `MANUAL_REVIEW`.

## Реальный end-to-end тест

В бот одним Telegram-сообщением отправлен текст тестовой вакансии. Сохранённый профиль кандидата был добавлен в `candidate_data` внутри Make.

Фактический маршрут:

```text
Telegram trigger
→ command filter
→ input variables
→ Both inputs present
→ validation_status
→ AI
→ MANUAL_REVIEW fallback
→ Telegram delivery
```

Заблокированные ветки:

- `Missing input`;
- `Known QA violation`;
- Telegram-доставка ошибки;
- Telegram-доставка `FAIL`.

Метрики History:

```text
Trigger: Manual
Duration: 11 seconds
Operations: 6
Credits: 6.94
Data size: 8.7 KB
```

Бот доставил полное сообщение:

```text
⚠️ MANUAL_REVIEW

[qa_message]

Предварительный AI-анализ:

[AI Answer]
```

Видимого обрезания ответа не было.

## Ручной смысловой QA результата

Основные решения были корректными:

```text
Отклик: откликаться
Предложение: пока недостаточно информации
```

При этом ответ не получил смысловой `PASS`. Выявлены дефекты:

1. обязанность по настройке сервиса снова была превращена в требование подтверждённого прошлого опыта внедрения;
2. модель предложила выяснить, какие сервисы кандидат внедрял, хотя такие проекты во входных данных не подтверждены;
3. модель предложила «уточнить у кандидата готовность», хотя анализ предназначен самому кандидату и его способность учиться уже подтверждена;
4. формулировка «холодные звонки не являются стоп-фактором» неточна: они являются стоп-фактором кандидата, но отсутствуют в конкретной вакансии;
5. встроенное `QA: PASS` осталось самооценкой генератора и не считается независимым доказательством.

Итог независимой проверки:

```text
MANUAL_REVIEW с существенными смысловыми замечаниями
```

Защитный QA Router сработал корректно: отсутствие известных строк не было ошибочно превращено в автоматический `PASS`.

## Telegram missing-input test

Для bounded-теста `candidate_data` был временно очищен, а обычный Telegram-текст обеспечил непустой `vacancy_text`. Выполнился маршрут:

```text
Telegram trigger
→ input variables
→ Missing input
→ error_message
→ Telegram delivery
```

History evidence:

```text
Trigger: Manual
Duration: less than a second
Operations: 4
Credits: 4
Data size: 1.3 KB
```

`Both inputs present` был заблокирован, `Tools 3` не прошёл фильтр, а AI отсутствовал в execution log. Бот доставил ожидаемое сообщение о недостающих данных. После фиксации доказательств полный `candidate_data` был восстановлен и сценарий сохранён без повторного запуска.

## Telegram FAIL test на текущей архитектуре

Для контролируемого теста во временный конец `Known QA violation` был добавлен отдельный OR-паттерн `Статус данных`. Обычный Telegram-текст вакансии прошёл текущий путь:

```text
Telegram trigger
→ input Tools
→ input Router
→ command / vacancy Router
→ success marker
→ AI
→ Known QA violation
→ qa_status = FAIL
→ Telegram block delivery
```

History evidence:

```text
Trigger: Manual
Duration: 17 seconds
Operations: 6
Credits: 6.85
Data size: 5.7 KB
```

Missing-input и command routes были заблокированы. `MANUAL_REVIEW` и его Telegram delivery не выполнялись. Пользователь получил только:

```text
❌ FAIL

Ответ заблокирован: обнаружена запрещённая или служебная формулировка. Нужна ручная проверка.
```

AI Answer пользователю не отправлялся. После фиксации operation bubbles, сообщения и History metrics временный `Статус данных` был удалён. В production filter остались только постоянные паттерны; сценарий сохранён без повторного запуска.

## Что проверено, а что ещё нет

Проверено фактически:

- получение обычного текстового сообщения Telegram;
- mapping `Message.Text → vacancy_text`;
- первоначальная блокировка `/start` до AI;
- текущая `/start` welcome-ветка без AI;
- success-вход через `Both inputs present`;
- AI-вызов;
- fallback `MANUAL_REVIEW`;
- отправка предупреждения, QA-сообщения и полного AI-ответа в исходный Telegram chat;
- missing-input маршрут без AI и доставка `error_message` в Telegram;
- текущая vacancy route через AI, `FAIL` Tools и Telegram block delivery без показа AI Answer.

Все четыре пользовательских Telegram path подтверждены отдельными реальными тестами: welcome, missing input, `MANUAL_REVIEW` и `FAIL`.

## Сводная test matrix

| Path | Duration | Operations | Credits | Data size | AI |
|---|---:|---:|---:|---:|---|
| `/start` welcome | < 1 s | 3 | 3 | 1.7 KB | No |
| Missing input | < 1 s | 4 | 4 | 1.3 KB | No |
| MANUAL_REVIEW | 11 s | 6 | 6.94 | 8.7 KB | Yes |
| FAIL | 17 s | 6 | 6.85 | 5.7 KB | Yes, answer blocked |

## Ограничения

- профиль кандидата пока статически хранится в Make;
- бот принимает текст вакансии, но не читает ссылки, PDF или изображения;
- команды получают одно общее приветствие; отдельные ответы для `/help` и неизвестных команд пока не реализованы;
- Telegram-сценарий не активирован для постоянной работы;
- слишком длинный AI-ответ потенциально может превысить лимит одного Telegram-сообщения;
- детерминированный QA gate ищет известные строки, но не заменяет смысловую проверку;
- bot token, webhook URL и служебные ID не публикуются.

## Безопасность

Секреты вводятся только в Make connection. В репозиторий и документацию не добавляются:

- Telegram bot token;
- webhook URL;
- connection credentials;
- chat ID;
- Make run ID и внутренние service IDs.
