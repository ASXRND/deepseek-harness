# Шпаргалка: deepseek-harness

## Полная пересборка с нуля (когда что-то странно)

```sh
cd ~/Desktop/development_locall/deepseek-harness
pon
pnpm run clean                      # снести все build-артефакты (lib/, dist/, .tsbuildinfo, .dsh-build)
pnpm install                        # переставить зависимости (если менялся lock)
pnpm run build                      # полная пересборка всех пакетов
lsof -nP -iTCP:3080 -sTCP:LISTEN -t | xargs -I{} sh -c 'kill {} && sleep 1 && pnpm dsh web --no-open'
# в браузере: Cmd+Shift+R на http://127.0.0.1:3080 (токен из вывода dsh web)
```

Когда это нужно:
- после `git pull`/`git fetch` пришли изменения в `packages/*` или `pnpm-lock.yaml`
- в UI виден старый текст/поведение после `pnpm run build` (значит `lib/` не снёсся)
- `pnpm run typecheck` или `pnpm run test` падают с ошибками про устаревшие типы
- менялся `packages/client/locale/src/locales/*.ts` и обычный `build` не помог

Что делает каждый шаг:
- `pnpm run clean` — удаляет `lib/`, `dist/`, `.tsbuildinfo`, `.dsh-build/`. Без этого
  `pnpm run build` может пропустить часть пакетов, потому что tsconfig решит «уже
  собрано».
- `pnpm install` — если в lock появились новые зависимости.
- `pnpm run build` — tsc + tsdown по всем пакетам + vite для `apps/web`.

## Пересборка после правок UI/локали/UI-тестов

**Важно:** `pnpm run build:web` собирает ТОЛЬКО vite-бандл (`apps/web/dist`).
`dsh web` отдаёт **серверный HTML-шаблон** из `packages/client/web/lib/` —
он собирается через `pnpm run build`. Если менял локаль (`en.ts`/`zh.ts`/
`ru.ts`) или компоненты в `packages/client/*` — нужен **полный** `pnpm run build`,
иначе UI продолжит показывать старый текст из устаревшего `lib/`.

```sh
cd ~/Desktop/development_locall/deepseek-harness
pon                                                # включить прокси (если ещё не)
pnpm run build                                     # полная сборка (tsc + tsdown всех пакетов)
lsof -nP -iTCP:3080 -sTCP:LISTEN -t | xargs -I{} sh -c 'kill {} && sleep 1 && pnpm dsh web --no-open'
# в браузере: Cmd+Shift+R на http://127.0.0.1:3080 (токен из вывода dsh web)
```

Зачем каждый шаг:
- `pnpm run build` — пересобирает **все** артефакты: `packages/client/locale/lib/`
  (словари), `packages/client/web/lib/` (HTML-шаблон + JS), `packages/client/ui-*`
  (sidebar, layout, frame), `apps/web/dist/` (vite-бандл). Без этого `lib/`
  остаётся со старой локалью.
- `kill PID на 3080` + перезапуск `dsh web` — старый node-процесс держит старый
  бандл в памяти. Без перезапуска увидишь старый UI даже после `build`.
- Cmd+Shift+R — жёсткая перезагрузка браузера, иначе отдаст кэшированный HTML.

Когда хватит **только** `pnpm run build:web` (vite-бандл, без правки локали):
- менял только `apps/web/vite.config.ts` или статику в `apps/web/`
- в `<title>`/sidebar всё уже ОК, хочешь только пересобрать vite-артефакт

Когда нужен **полный** `pnpm run build`:
- менял `packages/client/locale/src/locales/*.ts` (словари)
- менял что-то в `packages/client/ui-*/` (sidebar, layout, frame)
- менял `packages/client/web/src/boot.ts` или компоненты в `packages/client/web/`
- после `git pull` (там часто обновляются пакеты → пересборка обязательна)

Если после пересборки посыпались тесты (`pnpm run test` / `pnpm run test:snapshot`) — это снапшоты, не баг кода. Сравнить diff:
```sh
pnpm run test:snapshot 2>&1 | head -80
```
Если расхождения соответствуют твоей правке (новая надпись, новый ключ локали) — обновить ожидаемые файлы:
- `apps/web/tests/built-boot.expected.e2e.ts`
- `snapshots/**/*.expected.*` (владелец локальный, см. `snapshots/AGENTS.md`)
- `*.client.spec.tsx` (там зашит ожидаемый текст через `expect(document.title).toBe('...')`)

Если расхождение **не соответствует** твоей правке — не правь снапшот, откатывай правку, разбирайся.

Как проверить, что в UI реально новая локаль (открыть в браузере, F12 → Console):
```js
[...document.scripts].map(s => s.src)
// посмотреть, какие *.js модули загрузились, и в DevTools → Network → JS
// найти en.js / locale.js → проверить дату и содержимое
```

## Обновление репы (главное)

```sh
git pull --ff-only origin master   # подтянуть новый код (локальных коммитов нет — ff всегда ок)
pnpm install                        # если менялся pnpm-lock.yaml
pnpm run build                      # пересобрать артефакты (pnpm dsh web использует именно их)
```

- Перед pull не мешает `git status` — локальные правки затрутся, неотслеживаемые файлы (README_LOCALL.md и т.п.) не тронет.
- Если после обновления что-то странно — `pnpm run clean && pnpm install && pnpm run build`.
- Версия pnpm зашита в package.json (`"packageManager": "pnpm@11.7.0"`). Не запускать `corepack use ...` — он правит package.json.
- Проверить, отстал ли репозиторий: `git fetch && git log --oneline HEAD..origin/master | head`

## Запуск

```sh
pnpm dsh web          # Web UI на http://127.0.0.1:3080 (--no-open чтобы не открывать браузер)
pnpm dsh --profile headless "task"   # одна задача из терминала (нужен DEEPSEEK_API_KEY)
pnpm run demo:acp     # ACP-автоматизация
```

API-ключ: root `.env` → `DEEPSEEK_API_KEY` (+ опц. `DEEPSEEK_BASE_URL`). Без ключа e2e-тесты сами скипаются.

## Команды, которые могут пригодиться

```sh
pnpm run build:web    # только web-фронтенд
pnpm run dev:web      # dev-режим web
pnpm run test         # юнит-тесты (vitest)
pnpm run typecheck    # проверка типов
pnpm run lint         # oxlint
pnpm run clean        # снести билд-артефакты
pnpm mock:llm         # мок LLM-сервера (тесты без ключа)
```

## Модели (как подключить/поменять)

Список моделей лежит в `packages/llm/llm-deepseek/src/index.ts:83` — константа `DEFAULT_MODELS`:

```ts
const DEFAULT_MODELS: DeepSeekCatalogModel[] = [
  { id: 'deepseek-v4-flash', name: 'DeepSeek-V4-Flash', contextWindow: DEFAULT_CONTEXT_WINDOW },
  { id: 'deepseek-v4-pro', name: 'DeepSeek-V4-Pro', contextWindow: DEFAULT_CONTEXT_WINDOW },
  {
    id: 'deepseek-v4-flash-vision-exp',
    name: 'DeepSeek-V4-Flash-Vision-Exp',
    contextWindow: DEFAULT_CONTEXT_WINDOW,
    inputModalities: ['text', 'image'],
    ...
  },
]
```

Детали:

- Это каталог по умолчанию (advisory catalog) плагина-провайдера llm-deepseek. Он переопределяется через конфиг: поле `models?: DeepSeekCatalogModel[]` в Config (index.ts:120, схема на index.ts:166) — то есть из cordis.yml.
- Каждая модель валидируется в `resolveModels()` (index.ts:196): уникальный непустой id, положительные contextWindow/maxTokens, модальности text/image.
- Какая модель выбрана по умолчанию для агента — отдельно, в пакете `packages/core/agent-default-model`.

Если нужно поменять доступные модели — правь `DEFAULT_MODELS` или задай `models:` в cordis.yml.

## Кастомизация UI (названия, подписи, бренд)

Весь клиентский текст проходит через локаль (`packages/client/locale/src/locales/`).
Это **единственный** правильный путь менять надписи. AGENTS.md прямо требует:
`verify-client-ui-i18n` рейзит, если в `.tsx` встречается литерал вне словаря.

### Где живёт бренд/название продукта

- Ключ: `brand.localBuild` в `packages/client/locale/src/locales/en.ts:33`
  - `ru.ts` — для русского UI (если используешь)
  - `zh.ts` — для китайского
- Что в нём лежит: строка, которая показывается в `<title>`, в sidebar, в шапке.
- Это и есть «DSH Local Build» в исходниках. Поменял — пересобрать.

### Как поменять одну надпись (пошагово)

1. Найти ключ в `en.ts` (часто `grep -n "menu\|button\|label" packages/client/locale/src/locales/en.ts`).
2. Поменять **только значение**, ключ не трогать.
3. Проверить, что ключ нигде в коде не продублирован литералом:
   ```sh
   grep -rn "старое название" packages/client/ apps/web/ --include="*.ts" --include="*.tsx"
   ```
4. Пересобрать:
   ```sh
   pnpm run build:web       # пересобирает клиент и vite-конфиг
   # или полный сценарий:
   pnpm run build
   ```
5. **Перезапустить `dsh web`** (старый node-процесс держит старый бандл в памяти):
   ```sh
   lsof -nP -iTCP:3080 -sTCP:LISTEN -t | xargs -I{} sh -c 'kill {} && sleep 1 && pnpm dsh web --no-open'
   ```
6. Сбросить кэш браузера (Cmd+Shift+R) — иначе увидишь старый title.

### Если в браузере ничего не поменялось — чеклист

- [ ] Сборка прошла без ошибок? (`pnpm run build:web` — последние 5 строк)
- [ ] `dsh web` перезапущен после сборки?
- [ ] Браузер жёстко перезагружен (Cmd+Shift+R, не F5)?
- [ ] Литерал заменён **везде**, не только в `en.ts`? Часто копия строки сидит в:
  - `apps/web/vite.config.ts` — `DEFAULT_CLIENT_TITLE` и `html.replace('<title>...</title>', ...)`
  - `apps/web/tests/built-boot.expected.e2e.ts` — снапшот-тест (если меняешь видимый в `<title>` текст, тут будет расхождение)
  - спецификациях `*.client.spec.tsx` — если там зашит ожидаемый текст
- [ ] Если правил ключ (не значение) — обновил ли все вызовы `t('key')`?
- [ ] Если добавил новый ключ в `en.ts` — добавил ли в `ru.ts`/`zh.ts`? (иначе упадёт typecheck или рендер покажет ключ)

### Где НЕЛЬЗЯ менять UI-текст

- Внутри `.tsx` файлов (минуя `t('...')` или `<Localized>`) — `verify-client-ui-i18n` зарежет.
- В `vite.config.ts` `DEFAULT_CLIENT_TITLE` — это fallback для бутстрапа, до загрузки словаря. Синхронизировать с `brand.localBuild`, не считать за «правду».
- В `built-boot.expected.e2e.ts` — снапшот-тест, отражает **что должно быть**. Меняется вместе с правкой, иначе CI упадёт.

### Кастомизация через `~/.dsh/` (без правки исходников)

Часть настроек можно менять **только в юзер-конфиге**, не трогая репу:

- `~/.dsh/settings.yaml` — провайдеры, модели, агенты, agent-presets, тема UI (`ui-theme.preference: dark`)
- `~/.dsh/profiles/` — собственные cordis.yml-профили поверх базовых
- `~/.dsh/.env` — env-уровень (после апдейта 0.1.2-rc.1 сюда можно `HTTP_PROXY`/`NO_PROXY`; в проектом `.env` они запрещены)

UI-тексты (`brand.*` и т.п.) в `~/.dsh/` не выносятся — только через `en.ts`. Это by design.

### Свой цвет/тема

`~/.dsh/settings.yaml` → секция `ui-theme`. Пресет `dark`/`light` и переопределение палитры (см. комментарии в `settings.yaml.bak`).
Глобальная CSS-кастомизация — отдельная тема, не покрыта здесь.

## Прочее полезное

- Репа в developer preview, ломающие изменения — обычное дело: после pull пересборка обязательна.
- Доки: https://deepseek-harness.github.io/deepseek-harness/ ; локально docs/ (architecture.md — как всё устроено).
- Всё в репе — плагины на vendored Cordis; каждый пакет `@deepseek-ai/dsh-*` в packages/<группа>/<имя>.

## Локальная модель (Qwen 3.8 27B на 192.168.0.220:8080)

> Подключение локальной llama.cpp модели к dsh. Полная дока по серверу:
> `~/Desktop/local_storage/obsidian/asx_storage/llm_gpu_serverver_setup.md`.
> Инструкция для **провайдера и шаблона**:
> `connecting models.md` §11 (там же §12 — карта файлов dsh).

### Что на сервере

- llama-server (`Qwen3.8-27B-UD-Q4_K_XL.gguf`, 3×GPU, 24 GB VRAM)
- Контекст 98 304, KV-кэш Q8, один слот, **reasoning ВКЛЮЧЁН**
- Слушает `0.0.0.0:8080`, base URL: `http://192.168.0.220:8080/v1`

### Провайдер в `~/.dsh/settings.yaml`

Секция `qwen-local` (добавлена 2026-08-30, обновлялась 31-го):
```yaml
qwen-local:
  displayName: Qwen Local (LAN)
  apiKeyEnv: LLAMA_LOCAL_KEY            # заглушка, сервер ключ игнорирует
  api: openai-completions
  baseURL: http://192.168.0.220:8080
  models:
    - id: Qwen3.8-27B
      name: Qwen 3.8 27B Local
      contextWindow: 98304              # = -c на сервере
      maxTokens: 16384                  # reasoning ест токены
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

### `no_proxy` в `~/.zshrc` (ОБЯЗАТЕЛЬНО)

dsh на маке работает через xray (`127.0.0.1:10809`). Без `192.168.0.220` в
`no_proxy`/`NO_PROXY` Node (undici) шлёт запрос к локалке через прокси,
а xray с `routeOnly: true` дропает IP-only трафик с 503. В dsh это
выглядит как `Connection error, code: TRANSPORT` в логе сессии.

**Важно про CIDR** (`192.168.0.0/24`): **не работает** в `NO_PROXY` для
curl и Node — они не понимают диапазоны. Указывать конкретные IP.

Текущая строка `~/.zshrc:91`:
```sh
export HIDDIFY_NO_PROXY="localhost,127.0.0.1,::1,192.168.0.220"
```

### ⚠️ После правки `.zshrc` — ПЕРЕЗАПУСТИТЬ dsh

`dsh web` — node-процесс. Он **не делает** `source ~/.zshrc` сам, берёт
переменные (`HTTP_PROXY`, `NO_PROXY`, и т.д.) от родительского шелла
**в момент запуска**. Если ты правишь `.zshrc` или `HIDDIFY_NO_PROXY`
уже после того, как dsh стартовал, изменения **не подхватятся** — dsh
продолжит ходить через xray на 192.168.0.220:8080 и падать с TRANSPORT.

**Симптом** (виден в логе сессии `~/.dsh/sessions/.../session.jsonl.zstd`):
```
{"type":"assistant/chunk","chunk":{"type":"finish","reason":{"kind":"error",
 "failure":{"message":"Connection error.","code":"TRANSPORT"}}}}
{"type":"llm/retry","provider":"qwen-local","retry":2,…}
```

**Лечится так** (одна команда):
```sh
lsof -nP -iTCP:3080 -sTCP:LISTEN -t | xargs -I{} sh -c 'kill {} && sleep 1 && pnpm dsh web --no-open'
```

Или проще — закрыть окно терминала, в котором запущен `dsh web`, и
запустить заново:
```sh
cd ~/Desktop/development_locall/deepseek-harness
pon                                    # включить прокси
pnpm dsh web --no-open                 # стартует web UI на 127.0.0.1:3080
```

Проверить, что у dsh в окружении правильный `NO_PROXY`:
```sh
PID=$(lsof -nP -iTCP:3080 -sTCP:LISTEN -t | head -1)
ps eww $PID | tr ' ' '\n' | grep -E "^NO_PROXY=|^no_proxy="
# должно содержать 192.168.0.220
```

### Обновление локального llama-server (когда меняются параметры)

Все аргументы llama-server (модель, контекст, KV, GPU) лежат в **drop-in
override**, не в основном unit. Поэтому:

```sh
ssh -p 22523 x888x@192.168.0.220
sudo systemctl edit llama-server      # правит /etc/systemd/system/llama-server.service.d/override.conf
# … правим …
sudo systemctl daemon-reload
sudo systemctl restart llama-server   # ждать ~2.5 мин (503 «Loading model» = грузится)
```

**НЕ использовать** `sudo systemctl edit --full` — он снесёт наш override
и сольёт всё в основной unit. Подробнее — в `llm_gpu_serverver_setup.md`
§«Структура unit».

### Проверка end-to-end

```sh
# 1. Сервер жив?
curl -s --max-time 5 http://192.168.0.220:8080/health
# {"status":"ok"}

# 2. Прямой smoke-test (минуя прокси через no_proxy):
curl -s --max-time 30 http://192.168.0.220:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen3.8-27B","messages":[{"role":"user","content":"Скажи: ок"}],"max_tokens":20}'
# → "content": "ок", "reasoning_content": "...", "usage": {"completion_tokens": N, "prompt_tokens": M}

# 3. Через dsh:
cd ~/Desktop/development_locall/deepseek-harness
pon
pnpm dsh --profile headless "Ответь одним словом: работает"
# в UI: выпадашка модели → "Qwen Local (LAN) / Qwen3.8-27B"
```
