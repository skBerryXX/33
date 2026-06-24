# Setup: Free OpenCode models in Pi Coding Agent

> **Добавляем бесплатные opencode модели (deepseek-v4-flash-free, minimax-m3-free, mimo-v2.5-free) в Pi AI.
> Даём им 1M контекста. Включаем variant (`:high`).**

---

## 1. Регистрируем модели в Pi AI

Файл: **`~/.pi/agent/models.json`**

```json
{
  "providers": {
    "opencode": {
      "models": [
        {
          "id": "deepseek-v4-flash-free",
          "name": "DeepSeek V4 Flash Free",
          "api": "openai-completions",
          "baseUrl": "https://opencode.ai/zen/v1",
          "reasoning": true,
          "input": ["text"],
          "cost": {
            "input": 0,
            "output": 0,
            "cacheRead": 0,
            "cacheWrite": 0
          },
          "contextWindow": 1000000,
          "maxTokens": 128000
        },
        {
          "id": "minimax-m3-free",
          "name": "MiniMax M3 Free",
          "api": "openai-completions",
          "baseUrl": "https://opencode.ai/zen/v1",
          "reasoning": true,
          "input": ["text"],
          "cost": {
            "input": 0,
            "output": 0,
            "cacheRead": 0,
            "cacheWrite": 0
          },
          "contextWindow": 1000000,
          "maxTokens": 128000
        },
        {
          "id": "mimo-v2.5-free",
          "name": "MiMo V2.5 Free",
          "api": "openai-completions",
          "baseUrl": "https://opencode.ai/zen/v1",
          "reasoning": true,
          "input": ["text"],
          "cost": {
            "input": 0,
            "output": 0,
            "cacheRead": 0,
            "cacheWrite": 0
          },
          "contextWindow": 1000000,
          "maxTokens": 128000
        }
      ]
    }
  }
}
```

> **Важно:** `contextWindow` должен совпадать с тем, что реально даёт opencode (см. шаг 3).
> Если не совпадает — Pi будет компактить историю раньше, чем нужно, и вы не используете весь контекст модели.

---

## 2. Включаем variant (`:high`, `:max` etc.)

Без этого шага `/model` покажет модели, но без variant. С ним — будет показываться `deepseek-v4-flash-free:high`.

Файл: **`~/.pi/agent/settings.json`**

```json
{
  "enabledModels": [
    "opencode/deepseek-v4-flash-free:high",
    "opencode/minimax-m3-free:high",
    "opencode/mimo-v2.5-free:high"
  ]
}
```

Формат: `provider/modelId:thinkingLevel`.

---

## 3. Даём 1M контекста (opencode override)

По умолчанию opencode даёт free моделям 200K контекста.
Чтобы поднять до 1M — добавляем override.

Файл: **`~/.config/opencode/opencode.jsonc`**

```jsonc
{
  "provider": {
    "opencode": {
      "models": {
        "deepseek-v4-flash-free": {
          "limit": {
            "context": 1000000,
            "output": 64000
          }
        },
        "minimax-m3-free": {
          "limit": {
            "context": 1000000,
            "output": 32000
          }
        },
        "mimo-v2.5-free": {
          "limit": {
            "context": 1000000,
            "output": 64000
          }
        }
      }
    }
  }
}
```

> **Почему это работает:** opencode позволяет переопределять лимиты моделей через `opencode.jsonc`.
> Кеш в `~/.cache/opencode/models.json` не трогаем — override побеждает.

---

## 4. Проверяем auth

В `~/.pi/agent/auth.json` должен быть ключ для провайдера `opencode`:

```json
{
  "opencode": "oc_..."
}
```

Pi AI показывает только те модели, для которых есть auth. Если ключа нет — модели не появятся.

---

## 5. Проверяем что всё работает

```bash
node --input-type=module -e "
import { ModelRegistry, AuthStorage } from '/home/debi/.npm-global/lib/node_modules/@futurelab-studio/telepi/node_modules/@mariozechner/pi-coding-agent/dist/index.js';
const auth = AuthStorage.create();
const registry = ModelRegistry.create(auth);
console.log('--- OpenCode models ---');
for (const m of registry.getAll()) {
  if (m.provider === 'opencode') {
    console.log(m.provider + '/' + m.id, '| ctx=' + m.contextWindow, '| auth=' + auth.hasAuth(m.provider));
  }
}
"
```

Ожидаемый вывод:
```
opencode/deepseek-v4-flash-free | ctx=1000000 | auth=true
opencode/minimax-m3-free       | ctx=1000000 | auth=true
opencode/mimo-v2.5-free        | ctx=1000000 | auth=true
```

---

## 6. Проверяем в telepi

- `/model` — полный список. Все три opencode модели должны быть с галочкой ✅
- `/model deepseek` — фильтр
- `/model opencode/deepseek-v4-flash-free` — прямой switch
- Сделай `/new` если не появились сразу — Pi перечитывает models.json при старте сессии

---

---

## ⚙️ Твики telepi (model + voice)

### `/model` — поиск и filter

В telepi добавлен поиск поверх стандартного picker-а:

- **`/model deepseek`** — фильтр по тексту. Показывает только модели где есть `deepseek` в имени/id
- **`/model`** (без аргумента) — полный список со scoped моделями и variant
- **`/model opencode/deepseek-v4-flash-free`** — прямой switch
- **Кнопка ❌ Clear filter** — сбрасывает поиск
- **Индикатор** — в хедере `Filter: "текст"`

Патч в `dist/bot/commands/model.js`.

### `/voice` — выбор бэкенда транскрибации

У telepi есть встроенная транскрибация голосовых сообщений. Через `/voice` можно выбрать бэкенд — и вводить API ключи прямо из Telegram, без правки файлов.

```
/voice
Current: groq

[Sherpa-ONNX] [Groq] [OpenAI]
[🔑 Set Groq Key] [🔑 Set OpenAI Key]
[🗑 Clear Groq Key] [🗑 Clear OpenAI Key]
```

**Как работает:**
1. Нажать `🔑 Set Groq Key` → бот просит прислать ключ
2. Отправить ключ текстом в чат → сохраняется в `~/.config/telepi/voice-config.json`
3. Нажать `Groq` → активный бэкенд
4. Отправить голосовое → транскрибация через Groq

**Чтобы очистить** — `🗑 Clear ...`

**Бэкенды:**

| Бэкенд | Цена | Движок |
|--------|------|--------|
| Sherpa-ONNX | бесплатно 🔒 | локальный, off-line |
| Parakeet | бесплатно 🔒 | локальный, Apple Silicon |
| Groq | бесплатно ☁️ | whisper-large-v3-turbo |
| OpenAI | ~$0.006/мин | whisper-1 |

**Конфиг:** `~/.config/telepi/voice-config.json`

**Грабли:** Groq проверяет расширение файла в form-data. Telegram голосовые приходят как `.oga` — Groq не принимает. В коде починено: если расширение не из списка Groq — форсируется `audio.ogg`.

**Приоритет OpenAI ключа:** `voice-config.json` → `OPENAI_API_KEY` env → ошибка.

---

## Бонус: как узнать параметры модели

Все доступные opencode модели и их дефолтные лимиты лежат в кеше:

```bash
cat ~/.cache/opencode/models.json | python3 -m json.tool
```

Или одной строкой:
```bash
python3 -c "
import json
data = json.load(open('/home/debi/.cache/opencode/models.json'))
for id, m in data['opencode']['models'].items():
    print(f\"{id}: context={m.get('limit',{}).get('context','?')}, output={m.get('limit',{}).get('output','?')}\")
"
```

---

## Структура файлов (шпаргалка)

| Файл | Зачем |
|------|-------|
| `~/.pi/agent/models.json` | Кастомные модели Pi AI |
| `~/.pi/agent/settings.json` | `enabledModels` для variant |
| `~/.pi/agent/auth.json` | API ключи провайдеров |
| `~/.config/opencode/opencode.jsonc` | OpenCode конфиг (override лимитов) |
| `~/.cache/opencode/models.json` | Кеш opencode (справочно) |

---

*Гайд создан по мотивам реального включения free opencode моделей в Pi AI + telepi.*
*Автор: куратор v0.75.4+*
