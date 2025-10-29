# Multi-Agent System 2 з Telegram та генерацією зображень для n8n

## Опис

Це **розширена версія мульти-агентної системи** для n8n, яка інтегрована з **Telegram Bot** та підтримує **автоматичну генерацію зображень через Fal.AI**. Система автоматично створює контент, надсилає на перевірку, генерує візуали та публікує у Telegram канал.

## Ключові можливості

Цей шаблон включає:

✅ **Інтеграцію з Telegram** замість веб-чату
- Отримання запитів через Telegram бота
- Відправка згенерованого контенту на перевірку
- Автоматична публікація в канал

✅ **Генерацію зображень через Fal.AI**
- Автоматичне створення промпта для зображення
- Генерація візуалу через Flux Dev модель
- Публікація зображення разом з текстом

✅ **Approval процес**
- Підтвердження публікації перед відправкою
- Можливість внести правки
- Автоматична регенерація з урахуванням коментарів

✅ **Векторна база даних**
- Зберігання кейсів застосування ШІ
- Повторне використання існуючих кейсів
- Економія токенів та часу

---

## Порівняння з попередніми версіями

### Lesson 6: Базовий Multi-Agent
- ✓ AI Orkestrator
- ✓ Класифікація запитів
- ✓ Генерація кейсів
- ✗ Веб-чат (не зручно)
- ✗ Без зображень
- ✗ Без approval

### Lesson 7: + Векторна пам'ять
- ✓ Все з Lesson 6
- ✓ Vector Database
- ✓ Повторне використання кейсів
- ✗ Веб-чат (не зручно)
- ✗ Без зображень
- ✗ Без approval

### Lesson 8 (ця версія): + Telegram + AI Images
- ✓ Все з Lesson 7
- ✓ **Telegram інтеграція**
- ✓ **Генерація зображень (Fal.AI)**
- ✓ **Approval workflow**
- ✓ **Автоматична публікація в канал**
- ✓ **Можливість правок**

---

## Архітектура системи

### Повний потік workflow:

```
Telegram Trigger (користувач надсилає запит)
          ↓
    AI_Orkestrator (класифікація + генерація контенту)
          ↓
    ┌─────┴─────┬──────────────┬─────────────────┐
    ↓           ↓              ↓                 ↓
Other_topic  AI_topic   get_data_from   add_data_to
                        _VDB_tool       _VDB_tool
          ↓
Send a text message (надсилає на перевірку в Telegram)
          ↓
    Switch (перевірка підтвердження)
          ↓
    ┌─────┴─────┐
    ↓           ↓
  "Так"       "Ні"
    ↓           ↓
Rewrite to  Change_Prompt
  prompt     (правки)
    ↓           ↓
Create an   AI_Orkestrator
  Image      (регенерація)
    ↓
Wait → Check Status → Switch1
    ↓
Get the image
    ↓
Send a photo message (публікація в канал з зображенням)

АЛЬТЕРНАТИВА (тільки текст):
Send a text message1 (публікація в канал без зображення)
```

---

## Детальний опис workflow

### 1. Telegram Trigger

**Призначення:** Отримує повідомлення від користувачів через Telegram бота

**Що потрібно налаштувати:**
1. Створіть бота через @BotFather
2. Отримайте Bot Token
3. Вставте в Credentials

**Параметри:**
- **Updates**: `message` (відстежує текстові повідомлення)

**Вхід:** Повідомлення від користувача в Telegram

**Вихід:**
```json
{
  "message": {
    "message_id": 16,
    "from": {
      "id": 5744144767,
      "username": "username"
    },
    "chat": {
      "id": 5744144767
    },
    "text": "ШІ в страховому бізнесі"
  }
}
```

---

### 2. AI_Orkestrator

**Модель:** GPT-4.1

**System Prompt** (той самий що в Lesson 7, але з обмеженням на 500 символів):

```
#### ФОРМАТ ВІДПОВІДІ
Відповідь надай згідно структури нижче не більше 500 символів
Тема: [тут тема статті]
Текст: [тут текст статті]
```

**User Message:**
```
Напиши контент на тему: {{ $json.message.text }}

При необхідності використай правки які знаходяться нижче:
{{ $('Change_Prompt').item.json.rewritePrompt }}
```

**Що змінено порівняно з Lesson 7:**
- Обмеження на 500 символів (для Telegram)
- Можливість регенерації з правками
- Вихід адаптований для Telegram

---

### 3-6. Other_topic, AI_topic, VDB tools

**Ідентичні Lesson 7** - без змін.

Детальний опис див. в документації Lesson 7.

---

### 7. Send a text message (Approval Request)

**Призначення:** Надсилає згенерований контент на перевірку користувачу

**Параметри:**
- **Operation**: `sendAndWait` (чекає на відповідь)
- **Chat ID**: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Message**:
  ```
  ={{ $json.output }}

  -----------
  ПЕРЕВІРТЕ, БУДЬ ЛАСКА!
  ```

**Form Fields:**
1. **Dropdown:** "Чи підтверджуєте публікацію?"
   - Так
   - Ні

2. **Textarea:** "Ваш коментар"
   - Для внесення правок

**Options:**
- **Button Label**: "Надати відповідь"
- **Wait Time**: 5 хвилин

**Що це дає:**
- Контроль над публікацією
- Можливість правок перед публікацією
- Якість контенту

**Вхід:** Згенерований текст від AI_Orkestrator

**Вихід:**
```json
{
  "data": {
    "Чи підтверджуєте публікацію?": "Так",
    "Ваш коментар": "Додай більше деталей про автоматизацію"
  }
}
```

---

### 8. Switch (Розподіл за відповіддю)

**Призначення:** Перевіряє що обрав користувач

**Умови:**

**Вихід 0** (Так): Публікувати
```javascript
$json.data['Чі підтверджуєте публікацію?'] contains "Так"
```

**Вихід 1** (Ні): Внести правки
```javascript
$json.data['Чі підтверджуєте публікацію?'] contains "Ні"
```

---

### 9. Rewrite text to prompt (Генерація промпта для зображення)

**Призначення:** Конвертує текстстаття в промпт для Fal.AI

**Модель:** GPT-5-nano (OpenAI Chat Model3)

**Тип:** Chain LLM

**User Message:**
```
={{ $('AI_Orkestrator').item.json.output }}
```

**System Message:**
```
На основі заданого запиту Користувача згенеруй промпт для генеративного ШІ для генерації зображення.
Відповідь надай англійською мовою і тільки САМ ПРОМПТ в форматі JSON
```

**Structured Output Parser:**
```json
{
  "prompt": "Extreme close-up of a single tiger eye, direct frontal view. Detailed iris and pupil. Sharp focus on eye texture and color. Natural lighting to capture authentic eye shine and depth. The word \"FLUX\" is painted over it in big, white brush strokes with visible texture."
}
```

**Приклад:**

**Вхід:**
```
Тема: Штучний інтелект в медицині
Текст: Як ШІ революціонізує діагностику...
```

**Вихід:**
```json
{
  "prompt": "Modern medical AI interface, digital brain scan visualization, futuristic clinic setting, holographic display showing neural networks, professional medical equipment, blue and white color scheme, high-tech atmosphere"
}
```

---

### 10. Create an Image (HTTP Request до Fal.AI)

**Призначення:** Генерує зображення через Fal.AI API

**API:** Flux Dev model

**Method:** POST

**URL:** `https://queue.fal.run/fal-ai/flux/dev`

**Headers:**
- `Content-Type: application/json`
- `Authorization: Key YOUR_FAL_API_KEY` (в credentials)

**Body:**
```json
{
  "prompt": "{{ $json.output.prompt }}"
}
```

**Як отримати Fal.AI ключ:**
1. Зареєструйтесь на https://fal.ai/
2. Перейдіть до Dashboard → API Keys
3. Створіть новий ключ
4. Скопіюйте в n8n Credentials (HTTP Header Auth)

**Вихід:**
```json
{
  "request_id": "abc123-def456-ghi789",
  "status": "IN_QUEUE"
}
```

---

### 11. Wait (Очікування генерації)

**Призначення:** Чекає поки зображення генерується

**Параметри:**
- **Amount**: 15 секунд

**Навіщо:** Fal.AI генерує асинхронно, потрібен час.

---

### 12. Check Status (Перевірка статусу)

**Призначення:** Перевіряє чи готове зображення

**URL:**
```
https://queue.fal.run/fal-ai/flux/requests/{{ $json.request_id }}/status
```

**Вихід:**
```json
{
  "status": "IN_PROGRESS"  // або "COMPLETED"
}
```

---

### 13. Switch1 (Перевірка готовності)

**Призначення:** Визначає чи готове зображення

**Умови:**

**Вихід 0** (Готово):
```javascript
$json.status contains "COMPLETED"
```
→ Отримати зображення

**Вихід 1** (Не готово):
```javascript
$json.status not contains "COMPLETED"
```
→ Чекати ще

---

### 14. Wait1 (Додаткове очікування)

**Призначення:** Чекає ще 1 секунду якщо не готово

Потім повертається до Check Status (loop).

---

### 15. Get the image

**Призначення:** Отримує готове зображення

**URL:**
```
https://queue.fal.run/fal-ai/flux/requests/{{ $('Create an Image').item.json.request_id }}
```

**Вихід:**
```json
{
  "images": [
    {
      "url": "https://fal.media/files/abc123/image.png",
      "width": 1024,
      "height": 1024
    }
  ]
}
```

---

### 16. Send a photo message (Публікація з зображенням)

**Призначення:** Публікує пост з зображенням у Telegram канал

**Параметри:**
- **Operation**: `sendPhoto`
- **Chat ID**: `-1002946983842` (ID вашого каналу)
- **File**: `={{ $json.images[0].url }}`
- **Caption**: `={{ $('AI_Orkestrator').item.json.output }}`

**Як знайти Chat ID каналу:**
1. Додайте бота в адміністратори каналу
2. Надішліть повідомлення в канал
3. Відкрийте: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
4. Знайдіть chat id (починається з -)

**Результат:** Пост у каналі з зображенням та текстом

---

### 17. Send a text message1 (Публікація тільки текст)

**Призначення:** Публікує пост без зображення (альтернативний шлях)

**Параметри:**
- **Chat ID**: `-1002946983842`
- **Text**: `={{ $('AI_Orkestrator').item.json.output }}`

**Коли використовується:** Якщо генерація зображення не потрібна або fail.

---

### 18. Change_Prompt (Підготовка правок)

**Призначення:** Формує промпт для регенерації з правками

**Тип:** Set node

**Assignment:**
```
rewritePrompt = "Перегенеруй текст використовуючи правки нижче:
{{ $json.data['Ваш коментар'] }}

ОРИГІНАЛЬНИЙ ТЕКСТ:
{{ $('AI_Orkestrator').item.json.output }}"
```

**Куди передається:** Назад в AI_Orkestrator (loop)

---

## Налаштування workflow

### Крок 1: Імпорт

1. Завантажте `Multi-Agent System 82.json`
2. n8n → Workflows → Import from File

### Крок 2: Telegram Bot

#### Створення бота:

1. Відкрийте Telegram
2. Знайдіть @BotFather
3. Надішліть `/newbot`
4. Введіть назву та username
5. Отримайте **Bot Token**

#### Налаштування в n8n:

1. **Telegram Trigger:**
   - Credentials → Create New
   - Access Token: вставте Bot Token

2. **Send a text message:**
   - Використайте той же credential

3. **Send a text message1:**
   - Використайте той же credential

4. **Send a photo message:**
   - Використайте той же credential

### Крок 3: Telegram Channel

#### Створення каналу:

1. Telegram → Створити канал
2. Налаштуйте назву та опис
3. Додайте бота в адміністратори:
   - Канал → Адміністратори → Додати
   - Знайдіть свого бота
   - Надайте права на публікацію

#### Отримання Chat ID:

1. Надішліть повідомлення в канал
2. Відкрийте в браузері:
   ```
   https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
   ```
3. Знайдіть `"chat":{"id":-1234567890}`
4. Скопіюйте ID (з мінусом!)

#### Вставте Chat ID:

1. **Send a text message1** → Chat ID
2. **Send a photo message** → Chat ID

### Крок 4: Fal.AI

#### Реєстрація:

1. Перейдіть на https://fal.ai/
2. Sign Up
3. Dashboard → API Keys
4. Create New Key
5. Скопіюйте ключ

#### Налаштування в n8n:

1. Credentials → Create New
2. Тип: **HTTP Header Auth**
3. Name: `Authorization`
4. Value: `Key YOUR_FAL_API_KEY`

Використайте цей credential в:
- Create an Image
- Check Status
- Get the image

### Крок 5: OpenAI

Налаштуйте API ключ для 4 моделей:
1. OpenAI Chat Model (GPT-4.1)
2. OpenAI Chat Model1 (GPT-5-mini)
3. OpenAI Chat Model2 (GPT-5-mini)
4. OpenAI Chat Model3 (GPT-5-nano)
5. OpenAI Chat Model4 (GPT-5-nano)

Якщо моделей немає - замініть на доступні.

### Крок 6: Vector Database

Перевірте Memory Key:
- Simple Vector Store1 (insert): `ai_cases`
- get_data_from_VDB_tool (retrieve): `ai_cases`

Має збігатися!

### Крок 7: Workflow ID

**add_data_to_VDB_tool:**
- Workflow ID → Multi-Agent System 2

### Крок 8: Активація

1. Активуйте workflow
2. Надішліть тестове повідомлення боту в Telegram

---

## Тестування системи

### Тест 1: Повний цикл з підтвердженням

**Крок 1:** Надішліть боту:
```
Застосування ШІ в освіті
```

**Крок 2:** Отримаєте:
```
Тема: Як ШІ трансформує освіту...
Текст: [згенерований текст]

-----------
ПЕРЕВІРТЕ, БУДЬ ЛАСКА!
```

**Крок 3:** Натисніть "Надати відповідь"

**Крок 4:** Оберіть "Так"

**Крок 5:** Система:
- Генерує промпт для зображення
- Створює зображення через Fal.AI
- Публікує в канал

**Очікуваний результат:** Пост у каналі з текстом та зображенням

---

### Тест 2: Внесення правок

**Крок 1:** Надішліть запит

**Крок 2:** В approval формі:
- Оберіть "Ні"
- Напишіть коментар: "Додай статистику"

**Крок 3:** Натисніть "Надати відповідь"

**Крок 4:** AI_Orkestrator регенерує текст з урахуванням правок

**Крок 5:** Знову отримуєте approval форму

**Очікуваний результат:** Текст містить статистику

---

### Тест 3: Повторний запит (VDB)

**Крок 1:** Надішліть:
```
Автоматизація в ресторані
```

**Крок 2:** Надішліть схожий запит:
```
ШІ для ресторанів
```

**Очікуваний результат:**
- Другий запит швидший
- Other_topic НЕ викликався
- Використані існуючі кейси з VDB

---

## Типові проблеми

### Проблема 1: Бот не відповідає

**Причини:**
- Bot Token неправильний
- Workflow не активований

**Рішення:**
1. Перевірте Token в Telegram Trigger
2. Переконайтесь що workflow Active

### Проблема 2: Approval форма не відображається

**Причина:** SendAndWait потребує webhook

**Рішення:**
- Використовуйте n8n з публічним URL
- Або n8n Cloud
- Локальний n8n + ngrok

### Проблема 3: Зображення не генерується

**Причини:**
- Fal.AI ключ неправильний
- Недостатньо credits на Fal.AI
- Timeout занадто короткий

**Рішення:**
1. Перевірте ключ
2. Поповніть баланс на fal.ai
3. Збільшіть Wait з 15 до 30 секунд

### Проблема 4: Не публікує в канал

**Причини:**
- Chat ID неправильний
- Бот не адміністратор каналу

**Рішення:**
1. Перевірте Chat ID (має бути з мінусом)
2. Додайте бота в адміністратори каналу

### Проблема 5: Loop не зупиняється в Check Status

**Причина:** Fal.AI не встигає згенерувати

**Рішення:**
Додайте максимум ітерацій:
1. Додайте Counter node
2. Після 10 спроб → exit loop

---

## Розширення функціональності

### Додати вибір стилю зображення

В Rewrite text to prompt додайте в промпт:

```
Стиль: {{ $json.style }}
- realistic
- cartoon
- abstract
- minimalist
```

Користувач обирає стиль в approval формі.

---

### Додати scheduling публікацій

1. Після approval зберігайте в Google Sheets
2. Створіть окремий workflow з Cron Trigger
3. Зчитуйте з Sheets і публікуйте за розкладом

---

### Додати аналітику

Після публікації зберігайте:
- Дату публікації
- Тему
- Чи була картинка
- Кількість правок

В Google Sheets або Airtable.

---

### Multiple каналів

Замість одного Chat ID:

```javascript
const channels = [
  "-1002946983842",  // Основний
  "-1002946983843",  // Резервний
];

for (const chatId of channels) {
  // публікація
}
```

---

## Use Cases

### 1. Контент для Telegram каналу про технології

**Сценарій:**
- Щодня генерувати 3-5 постів
- Тематика: ШІ, автоматизація, бізнес
- З зображеннями для залучення

**Налаштування:**
- Додати Cron Trigger для автоматизації
- База тем в Google Sheets
- Scheduling публікацій

### 2. Персональний бренд експерта

**Сценарій:**
- Експерт надсилає тезу боту
- Система створює професійний пост
- З візуалом та публікацією

**Налаштування:**
- Налаштувати стиль під експерта
- Додати підпис
- Посилання на соц мережі

### 3. Новинний канал з ШІ

**Сценарій:**
- RSS feeds з новинами
- ШІ аналізує та створює пости
- Автоматична публікація

**Налаштування:**
- RSS Trigger замість Telegram
- Категоризація новин
- Тільки важливі події

---

## Автор

**Розробник:** Сергій Щербаков

**Email:** sergiyscherbakov@ukr.net

**Telegram:** @s_help_2010

---

## Ліцензія

Цей проект доступний для вільного використання та модифікації.

## Підтримка

Якщо виникли питання:
1. Перевірте всі credentials (Telegram, Fal.AI, OpenAI)
2. Перевірте Chat ID каналу (має бути з мінусом)
3. Перевірте що бот є адміністратором каналу
4. Execution History для debug
5. Документація Fal.AI: https://fal.ai/docs
6. Документація Telegram Bot API: https://core.telegram.org/bots/api
