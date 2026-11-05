# CONNECTING MODELS.md

Локальная шпаргалка: как подключить ЛЮБУЮ модель к DeepSeek Harness (dsh)
через OpenRouter (или любой OpenAI-совместимый API). Объединяет опыт
подключения `stealth/ox-alpha` (2026-08-24) и бесплатных моделей
MiniMax M3 / GLM 5.2 (2026-08-28).

Файл не коммитится — личная шпаргалка, не документация репозитория.

---

## 0. TL;DR — рецепт подключения новой модели за 5 минут

1. Узнать точный слаг модели: `https://openrouter.ai/api/v1/models`
   (брать `id`, контекст — `context_length`, выход — `top_provider.max_completion_tokens`).
2. Проверить модель живым curl-запросом (шаблон в §5.1) — сразу видно
   402/429/обязательный reasoning.
3. Добавить в `~/.dsh/settings.yaml` отдельный провайдер (шаблон в §3)
   и, если надо, `agent-default-model`.
4. Запустить с прокси: `pon` + `NODE_USE_ENV_PROXY=1` (§6).
5. Проверить end-to-end: `pnpm dsh --profile headless "тест"` (§5.3).

---

## 1. Почему нельзя «просто добавить модель в список»

- `DEFAULT_MODELS` в `packages/llm/llm-deepseek/src/index.ts:83` относится
  только к провайдеру `deepseek-official` — к OpenRouter отношения не имеет.
- Каталог OpenRouter зашит в пакет `@earendil-works/pi-ai@0.82.1` (лежит в
  `node_modules/.pnpm/@earendil-works+pi-ai@0.82.1_*/`), где новых моделей нет.
- У маршрута `llm-pi-ai` поле `models` **заменяет** весь каталог целиком,
  а `modelOverrides` отказывает для id, которых в каталоге нет.
- ВЫВОД: единственный способ — **свой провайдер (отдельный маршрут) на
  тот же baseURL** с одной-двумя моделями. Штатный маршрут `openrouter`
  и остальные ~400 моделей продолжают работать.

Репозиторий остаётся нетронутым.

---

## 2. Где что лежит

| Что | Где |
|---|---|
| Настройки dsh | `~/.dsh/settings.yaml` (бэкапы `settings.yaml.bak*`) |
| API-ключи | `~/.dsh/.credentials.yaml` (`OPENROUTER_API_KEY`, формат `sk-or-v1-…`) |
| Схема конфига llm-pi-ai | `packages/llm/llm-pi-ai/src/config.ts` |
| Допустимые уровни/модальности/форматы | `packages/llm/llm-pi-ai/src/catalog.ts` |
| Формат reasoning-dispatch (док-строка) | pi-ai `dist/types.d.ts:441-443` |
| Логи сессий | `~/.dsh/sessions/**/session.jsonl.zstd` |

---

## 3. Шаблон провайдера в `~/.dsh/settings.yaml`

```yaml
agent-default-model:                 # модель по умолчанию для новых агентов
  provider: openrouter-free          # имя твоего провайдера из секции ниже
  model: minimax/minimax-m3:free     # точный слаг (в UI выбор сессии сильнее дефолта)

llm-pi-ai:
  providers:
    openrouter:                      # штатный маршрут — НЕ ТРОГАТЬ
      apiKeyEnv: OPENROUTER_API_KEY
    <my-provider>:                   # любое уникальное имя, напр. openrouter-glm
      displayName: OpenRouter (My Model)
      apiKeyEnv: OPENROUTER_API_KEY  # тот же ключ; для не-OpenRouter — своё имя переменной
      api: openai-completions        # ОБЯЗАТЕЛЬНО для своего маршрута
      baseURL: https://openrouter.ai/api/v1
      models:                        # можно несколько моделей в одном провайдере
        - id: vendor/model-slug      # точный слаг из /api/v1/models
          name: Human Name
          contextWindow: 1048576     # реальные цифры из /api/v1/models
          maxTokens: 131072          # max выходных токенов (см. §4 про 402)
          input: [text, image]       # схема допускает ТОЛЬКО text и image
          reasoningEfforts:          # см. §4; для моделей без reasoning — не указывать
            low: low
            medium: medium
            high: high
          compat:
            thinkingFormat: openrouter  # для OpenRouter — всегда openrouter
```

### Справочник значений (из схемы, `catalog.ts`)

- `input`: только `text`, `image` (видео у модели заявлено, но в схему не влезет).
- Ключи `reasoningEfforts`: `off, minimal, low, medium, high, xhigh, max`.
  Значение = как записать уровень в провод (wire spelling). Только `off`
  может иметь пустое значение (не слать ничего). Незаявленные уровни → null.
- `compat.thinkingFormat`: `openai` (reasoning_effort), `openrouter`
  (reasoning: {effort}), `deepseek`, `together`, `zai` (thinking: {type}),
  `qwen`, `qwen-chat-template`, `chat-template`, `string-thinking`, `ant-ling`.
  Для OpenRouter ВСЕГДА `openrouter`.

---

## 4. Подводные камни моделей (проверено на практике)

| Проблема | Признак | Решение |
|---|---|---|
| **Reasoning обязательный** | `400: Reasoning is mandatory for this endpoint and cannot be disabled` (Ox Alpha и GLM так делают) | НЕ указывать `off` в `reasoningEfforts`. dsh при уровне off шлёт отключение reasoning — сервер отклоняет |
| **Платная модель + пустые кредиты** | `402: requires more credits… You requested up to N tokens, but can only afford M` | Уменьшить `maxTokens` в конфиге ИЛИ пополнить кредиты. Бесплатные модели — см. §5.2 |
| **Rate-limit free-моделей** | `429: temporarily rate-limited upstream` (в т.ч. на /chat/completions) | Общий лимит на все free-модели OpenRouter. Ждать или переключаться на другую free-модель |
| **Ключ невалиден** | реальный 401 | Перевыпустить на openrouter.ai |
| **Запрос мимо прокси** | `403 Access denied by security policy` | `pon` + `NODE_USE_ENV_PROXY=1` (§6) |
| **Ключ не найден** | `MISSING_CREDENTIAL` | Внести ключ через UI в `~/.dsh/.credentials.yaml` |

UI любой сбой авторизации показывает как «API key is invalid» (код `AUTH`),
даже если причина сетевая. Точная причина — только в логе сессии (§7).

---

## 5. Диагностика и проверка

### 5.1. Живой curl-тест модели (до правки конфига!)

```sh
export HTTPS_PROXY=http://127.0.0.1:10809
KEY=$(grep -o 'sk-or-[A-Za-z0-9_-]*' ~/.dsh/.credentials.yaml | head -1)
curl -s --max-time 45 https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"VENDOR/SLAG","messages":[{"role":"user","content":"Скажи: работает"}],"max_tokens":400}'
# проверить уровни reasoning: добавить "reasoning":{"effort":"low|medium|high"}
```

Полезно сразу прогнать все effort-уровни, которые заявляешь в конфиге.

### 5.2. Найти бесплатные модели

```sh
export HTTPS_PROXY=http://127.0.0.1:10809
curl -s --max-time 30 https://openrouter.ai/api/v1/models | python3 -c "
import json,sys
ms=json.load(sys.stdin)['data']
free=[m for m in ms if float(m.get('pricing',{}).get('prompt','1') or 1)==0
      and float(m.get('pricing',{}).get('completion','1') or 1)==0]
for m in free:
    sp=m.get('supported_parameters',[])
    print(m['id'],'| ctx:',m.get('context_length'),'| tools:',
          'tools' in sp and 'tool_choice' in sp,'| reasoning:','reasoning' in sp)
"
```

Для агентного режима модель ОБЯЗАНА поддерживать `tools` + `tool_choice`.

Состояние на 2026-08-28 (работали): `minimax/minimax-m3:free` (1M ctx,
tools, reasoning, default dsh), `inclusionai/ling-3.0-flash-fin:free`,
`dots-studio/dots-3-note-preview:free`. Временно 429: `z-ai/glm-5.2:free`,
`google/gemma-4-26b-a4b-it:free`. Не доступна с обычным ключом:
`thinkingmachines/inkling-small:free` (403, только age…).

### 5.3. End-to-end тест через dsh (главная проверка)

```sh
cd /Users/aleksandrhohon/Desktop/development_locall/deepseek-harness
pon   # + в ~/.zshrc должна быть строка NODE_USE_ENV_PROXY=1
pnpm dsh --profile headless "Ответь ровно одним словом: работает"
```

Ответ в stdout за ~10-30 c = вся цепочка (конфиг → ключ → прокси →
модель) работает. НБ: `pnpm dsh --dump-config` показывает ДО наложения
пользовательских настроек и НЕ годится для проверки settings.yaml.

---

## 6. Прокси — обязательное условие запуска

Прямой доступ к OpenRouter из сети блокируется (403 security policy).
Рабочий путь — Xray: HTTP-прокси `127.0.0.1:10809` (SOCKS5 `:10808`).

Node fetch (undici) игнорирует `HTTP(S)_PROXY` без `NODE_USE_ENV_PROXY=1`
(Node ≥ 24, строка уже в `~/.zshrc`). Нужны ОБЕ вещи одновременно:

| Переменная | Откуда |
|---|---|
| `HTTPS_PROXY=…10809` | команда `pon` |
| `NODE_USE_ENV_PROXY=1` | строка в `~/.zshrc` |

Проверить окружение запущенного приложения:

```sh
ps eww $(pgrep -f "bin.ts web" | head -1) | tr ' ' '\n' \
  | grep -E "^HTTPS_PROXY=|^NODE_USE_ENV_PROXY="
```

Проверить валидность ключа:

```sh
KEY=$(grep -o 'sk-or-[A-Za-z0-9_-]*' ~/.dsh/.credentials.yaml | head -1)
curl -s -H "Authorization: Bearer $KEY" https://openrouter.ai/api/v1/key
```

---

## 7. Диагностика через лог сессии

```sh
F=$(find ~/.dsh/sessions -name "session.jsonl.zstd" | head -1)
zstd -dc "$F" | grep '"finish"' | tail -3
```

---

## 8. Как запускать

```sh
pon                 # включить прокси в текущем терминале
pnpm dsh web        # веб-UI: http://127.0.0.1:3080
```

Алиас одной командой (в `~/.zshrc`):

```sh
alias dsh-web='pon --quiet; NODE_USE_ENV_PROXY=1 pnpm dsh web'
```

---

## 9. История: как это подключалось (кейсы)

### Кейс 1 — stealth/ox-alpha (2026-08-24, платная, работает)

`agent-default-model: provider openrouter-ox-alpha, model stealth/ox-alpha`.
Особенность: reasoning обязательный → в `reasoningEfforts` только
`low/high/max`. Контекст 1M. Провайдер `openrouter-ox-alpha` остался в
конфиге — модель доступна в UI.

### Кейс 2 — glm-5.3-flash, «модель Cline» (2026-08-28, платная — отклонено)

Модель, на которой работает Cline (агентская), доступна только через
OAuth-прокси Cline (токен живёт 1 час) — напрямую не подключается.
Через OpenRouter (`z-ai/glm-5.3-flash`, ctx 1.31M) подняли маршрут —
всё работало, кроме 402: free-tier кредитов хватает лишь на ~54k
выходных токенов. Решение по требованию: только бесплатные модели.
Заметка: если появятся кредиты, конфиг из §3 с этим слагом заведётся
сразу (уровни low/medium/high проверены живьём, reasoning обязательный).

### Кейс 3 — бесплатные модели (2026-08-28, текущий конфиг)

`agent-default-model: provider openrouter-free, model minimax/minimax-m3:free`.
В провайдере `openrouter-free` две модели:

- `minimax/minimax-m3:free` — ctx 1M, maxTokens 65536, tools+reasoning,
  effort low/medium/high — работает, стоит дефолтом;
- `z-ai/glm-5.2:free` — ctx 256k, text-only, maxTokens 32768 — то же
  семейство, что модель Cline; на момент настройки 429 upstream (лимит
  общий на free-модели, со временем пройдёт).

---

## 10. Если обновится pi-ai

Когда модель появится во встроенном каталоге OpenRouter новой версии
`@earendil-works/pi-ai`, её отдельный маршрут можно удалить из
`~/.dsh/settings.yaml` и переключить дефолт на штатный провайдер:
`provider: openrouter, model: <slug>`.

---

## 11. Локальный Qwen 3.8 27B (192.168.0.220:8080)

> Дата: 2026-08-31. Задача: подключить к dsh **полноценную** reasoning-модель,
> работающую на домашнем GPU-сервере (3×GPU, 24 GB VRAM), контекст 98k.
> Серверная дока: `~/Desktop/local_storage/obsidian/asx_storage/llm_gpu_serverver_setup.md`.

### 11.1. Что на сервере

- **Железо:** RTX 3070 (8 GB) + 2×RTX 2060 Super (8 GB), суммарно 24 GB VRAM
- **Софт:** llama.cpp (CUDA), модель `Qwen3.8-27B-UD-Q4_K_XL.gguf` (~17.6 GB)
- **Сервис:** systemd `llama-server.service`, слушает `0.0.0.0:8080`
- **Контекст:** `-c 98304` (KV-кэш Q8, 96k — потолок для 24 GB VRAM;
  128k не влезает, OOM на 2060 при `cudaMalloc`)
- **Reasoning:** ВКЛЮЧЁН (флаг `--reasoning off` снят). Модель отдаёт
  `reasoning_content` перед `content`. На скрине Open WebUI это видно как
  «💡 reasoning»-плашка.
- **Base URL для клиентов:** `http://192.168.0.220:8080/v1`

### 11.2. Параметры запуска (systemd override)

**Файлов два** (см. также §12.3 и серверную доку
`llm_gpu_serverver_setup.md` §«Структура unit»):

| Файл | Что внутри |
|---|---|
| `/etc/systemd/system/llama-server.service` | каркас (User, Restart, Environment). `ExecStart` — **только путь к бинарю**, без аргументов |
| `/etc/systemd/system/llama-server.service.d/override.conf` | **все аргументы** llama-server |

Содержимое `override.conf` на 2026-08-31:

```
[Service]
ExecStart=
ExecStart=/home/x888x/llama.cpp/build/bin/llama-server \
  -m /home/x888x/models/Qwen3.8-27B-UD-Q4_K_XL.gguf \
  -ngl 99 -sm layer -ts 1,1,1 \
  -c 98304 -np 1 \
  -ctk q8_0 -ctv q8_0 \
  --host 0.0.0.0 --port 8080
```

- `-c 98304` — контекст 98 304 токенов (больше 128k на 24 GB не влезает)
- `-np 1` — один параллельный слот (KV не раздувается на N слотов)
- `-ctk q8_0 -ctv q8_0` — KV-кэш в Q8 (экономит ~50% VRAM, потеря качества
  незначительна). С Q16/F16 — 128k не влезло бы даже близко.
- `-ts 1,1,1` — равномерное размазывание слоёв по 3 GPU. **НЕ ТРОГАТЬ**
  (см. §1 известных граблей в `llm_gpu_serverver_setup.md`)
- `--reasoning` — **без `off`**, иначе модель не думает, ответы тупеют
- `--host 0.0.0.0` — слушать на всех интерфейсах (иначе будет только 127.0.0.1)
- Загрузка после рестарта: ~2.5 минуты (HTTP 503 «Loading model» это норма)

**Правильный способ править**: `sudo systemctl edit llama-server` (он
откроет именно `override.conf`). **Не использовать** `systemctl edit --full` —
он сольёт всё в основной unit и сбросит наш override.

### 11.3. Зачем `no_proxy` в `~/.zshrc`

dsh на маке работает через xray (`127.0.0.1:10809`). Без `192.168.0.0/24`
в `no_proxy`/`NO_PROXY` Node (undici) шлёт запрос к локалке через прокси,
а xray с `routeOnly: true` дропает IP-only трафик с 503.

Уже сделано в `~/.zshrc:91`:
```
export HIDDIFY_NO_PROXY="localhost,127.0.0.1,::1,192.168.0.0/24,192.168.0.220"
```

### 11.4. Конфиг в `~/.dsh/settings.yaml`

```yaml
qwen-local:
  displayName: Qwen Local (LAN)
  apiKeyEnv: LLAMA_LOCAL_KEY            # заглушка, сервер ключ игнорирует
  api: openai-completions
  baseURL: http://192.168.0.220:8080
  models:
    - id: Qwen3.8-27B                   # слаг — отдаётся /v1/models (или полный путь к .gguf, тоже работает)
      name: Qwen 3.8 27B Local
      contextWindow: 98304              # = -c на сервере
      maxTokens: 16384                  # reasoning ест токены; 16k — компромисс
      input: [ text ]                   # локалка без vision
      reasoningEfforts:                 # reasoning ВКЛЮЧЁН
        low: low
        medium: medium
        high: high
      compat:
        thinkingFormat: openai          # НЕ openrouter — у нас локалка
```

Заглушка ключа в `~/.dsh/.credentials.yaml`:
```yaml
LLAMA_LOCAL_KEY: local-qwen            # сервер не проверяет, но pi-ai требует
```

### 11.5. Что НЕ работает / ограничения

- **Vision нет** — Qwen3.8-27B умеет multimodal, но llama-server на этом билде
  не отдаёт vision через OpenAI API. `input: [ text ]` поэтому жёстко.
- **Reasoning жрёт токены** — на длинных задачах 8k выхода мало. `maxTokens:
  16384` в провайдере; если reasoning «съел всё», ответ будет пустым — увеличить.
- **Скорость:** ~18 t/s на коротких ответах, ~5–10 t/s с длинным reasoning.
  На префилле (большой системный промпт dsh) — заметная пауза 5–20 сек.
- **Один слот** (`-np 1`). Параллельные запросы — в очереди.

### 11.6. Быстрая диагностика

```bash
# Сервер жив?
curl -s --max-time 5 http://192.168.0.220:8080/health
# {"status":"ok"}

# Прямой smoke-test (минуя прокси — no_proxy в .zshrc уже настроен):
curl -s --max-time 30 http://192.168.0.220:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen3.8-27B","messages":[{"role":"user","content":"Скажи ровно одно слово: ок"}],"max_tokens":20}'

# End-to-end через dsh:
cd ~/Desktop/development_locall/deepseek-harness
pon                                          # включает прокси
pnpm dsh --profile headless "Ответь одним словом: работает"
# в UI: выпадашка модели → "Qwen Local (LAN) / Qwen3.8-27B"
```

Если запрос через dsh возвращает 503 — скорее всего не отработал `no_proxy`
(новый шелл, `source ~/.zshrc` забыли). Проверить:
```bash
env | grep -E "no_proxy|NO_PROXY"
# должно содержать 192.168.0.0/24
```

---

## 12. Карта файлов dsh: где что менять, чтобы переключить модель

> Полный путь от «хочу другую модель» до «агент её видит». Удобно держать
> под рукой, когда правишь конфиги или расследуешь, откуда что берётся.

### 12.1. Конфиги на маке (пользовательские)

| Что | Путь | Зачем трогать |
|---|---|---|
| Список провайдеров LLM | `~/.dsh/settings.yaml` | добавить/убрать провайдера, поменять модель по умолчанию |
| API-ключи | `~/.dsh/.credentials.yaml` | ключи OpenRouter и заглушки для локалок |
| Прокси/no_proxy | `~/.zshrc` (`HIDDIFY_NO_PROXY`) | добавить новый LAN-диапазон, если поднимаешь ещё сервер |
| Бэкапы конфигов | `~/.dsh/settings.yaml.bak-*` | перед любой правкой `cp` с датой |
| Сессии и логи | `~/.dsh/sessions/*/session.jsonl.zstd` | `zstd -dc … \| grep '"finish"'` для диагностики |
| Профили запуска | `~/.dsh/profiles/{headless,web}/cordis.yml` | `[]` — пустые, всё через `cordis.patch.yml` и bundle |

### 12.2. Исходники dsh (проект)

Корень: `~/Desktop/development_locall/deepseek-harness`

| Что | Где |
|---|---|
| **Системный промпт** (то, что шлётся в каждом сообщении перед юзером) | `packages/core/system-prompt/src/` — собирает `ctx.systemPrompt` из секций (identity, persona, инструкции) |
| **Реестр инструментов (tools)** | `packages/core/tools/src/` — `edit`, `read`, `bash`, `grep`, `glob`, `webfetch`, и т.д. Каждый tool — отдельный plugin. Чтобы **убрать** tool — вычеркнуть из `cordis.yml` (НЕ из кода) |
| **Агентный цикл** (как промпт + tools + история превращаются в запрос) | `packages/core/agent-loop/src/` — main loop, решает когда вызвать tool |
| **Агент (composition)** | `packages/core/agent/src/` — собирает провайдера + модели + tools для конкретной сессии |
| **Default model** (env → file fallback) | `packages/core/agent-default-model/src/` — читает `agent-default-model` из settings.yaml |
| **Сессии** (персист, журнал, телеметрия) | `packages/core/session/src/` — пишет `~/.dsh/sessions/...` |
| **Каталог LLM** (что модель умеет) | `packages/llm/llm-pi-ai/src/catalog.ts` — схема `input`, `reasoningEfforts`, форматов |
| **Парсер конфига llm-pi-ai** | `packages/llm/llm-pi-ai/src/config.ts` — Zod-схема для `providers` в settings.yaml |
| **Схема LLM DeepSeek (не OpenRouter)** | `packages/llm/llm-deepseek/src/index.ts:83` — `DEFAULT_MODELS` |
| **Поставщик OpenAI-совместимого API** | `packages/llm/llm-pi-ai/src/providers/openai-completions.ts` (или аналог) |
| **Пресеты агентов** (например `cordis`) | `examples/web-cordis/cordis.yml`, `examples/headless-agent/cordis.yml` |
| **Пресет, который грузится по умолчанию** | `agent-presets.default: cordis` в `~/.dsh/settings.yaml` |
| **Конфиг, который dsh читает первым** | `~/.dsh/settings.yaml` (НЕ `--dump-config` — он показывает до наложения) |

### 12.3. Серверная часть (llama-server на 192.168.0.220)

| Что | Где на сервере |
|---|---|
| Модель (файл) | `/home/x888x/models/Qwen3.8-27B-UD-Q4_K_XL.gguf` |
| Каркас сервиса (User, Restart, Environment) | `/etc/systemd/system/llama-server.service` |
| **Все аргументы llama-server** (модель, контекст, GPU, KV, host/port) | `/etc/systemd/system/llama-server.service.d/override.conf` |
| Править параметры запуска | `sudo systemctl edit llama-server` (правит override.conf) |
| **НЕ использовать** | `sudo systemctl edit --full llama-server` — сносит наш override |
| Логи запуска | `sudo journalctl -u llama-server -f` |
| Состояние GPU | `nvidia-smi --query-gpu=index,memory.used --format=csv` |
| Здоровье | `curl http://192.168.0.220:8080/health` |
| Серверная дока (полная) | `~/Desktop/local_storage/obsidian/asx_storage/llm_gpu_serverver_setup.md` |

Подробнее про разделение «каркас + override» — в серверной доке, §«Структура unit».

### 12.4. Чек-лист «поменять модель»

1. **Новый бэкенд** (например LM Studio локально, или vLLM на другой машине):
   - добавить провайдера в `~/.dsh/settings.yaml` по шаблону §3
   - если он на LAN — добавить IP в `HIDDIFY_NO_PROXY` в `~/.zshrc`
     (CIDR `/24` **не работает** в `NO_PROXY` для curl/Node — указывать
     конкретные IP, см. §11.3)
   - заглушку ключа в `~/.dsh/.credentials.yaml`
2. **Новая модель на существующем бэкенде** (например, скачал другой .gguf):
   - положить .gguf в `/home/x888x/models/` на сервере
   - на сервере: `sudo systemctl edit llama-server` → поменять `-m …/новая.gguf`
     (откроется `override.conf`, **не** основной unit)
   - `sudo systemctl daemon-reload && sudo systemctl restart llama-server`
   - ждать ~2.5 мин (HTTP 503 = грузится, 200 = готово)
   - на маке: поменять `id:`/`name:`/`contextWindow:`/`maxTokens:` в `qwen-local`
3. **Поменять модель по умолчанию**:
   - `~/.dsh/settings.yaml` → `agent-default-model: {provider: …, model: …}`
4. **Включить/выключить reasoning на сервере**:
   - в `override.conf` добавить/убрать `--reasoning off`
   - на маке в `qwen-local` секции добавить/убрать `reasoningEfforts`
   - в `qwen-local` поднять `maxTokens` (для Qwen3.8 нужно ≥8k, иначе
     reasoning «съест» всё и `content` будет пустым)
5. **Поменять reasoning-effort в dsh** (low/medium/high):
   - `reasoningEfforts` в секции модели в `~/.dsh/settings.yaml`. Только
     те ключи, что сервер реально поддерживает.
6. **Проверить end-to-end**: `pnpm dsh --profile headless "тест"`.
   Заголовок ответа в UI должен содержать имя выбранного провайдера/модели.
