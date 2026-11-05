# PROJECT_MAP — выжимка по DeepSeek Harness

Личная шпаргалка (не документация репозитория). Собрано 2026-08-24 из `docs/tool-catalog.md`, `docs/subsystems/{approval,permission-presets,commands}.md`, `packages/bundle/base/cordis.patch.yml`.

---

## 1. Как убрать постоянные запросы разрешения

Права состоят из двух независимых «крутилок»:

| Крутилка | Значения | Смысл |
|---|---|---|
| Режим песочницы (`sandbox/mode`) | `read-only` / `workspace-write` / `danger-full-access` | куда агенту можно писать файлы и выполнять команды |
| Политика одобрения (`approval/policy`) | `ask` / `never` | спрашивать ли человека перед действием вне песочницы |

Штатные пресеты (связка обоих):

| Пресет | Песочница | Одобрение | Что значит |
|---|---|---|---|
| `read-only` | read-only | ask | только чтение |
| `workspace-write` (по умолчанию) | запись в рабочей папке + временные папки ОС | ask | пишет в проект свободно, за пределами — спросит |
| `danger-full-access` | полный доступ ко всей ФС | never | **ничего не спрашивает никогда** |

### Способ 1 — прямо в веб-интерфейсе
Селектор **Permissions** в UI, либо команда в чате:

```
/permission danger-full-access
```

### Способ 2 — переменная окружения при запуске (на весь процесс)

```sh
DSH_PERMISSION_MODE=danger-full-access pnpm dsh web
```

Читается в `packages/bundle/base/cordis.patch.yml`: режим песочницы = `DSH_PERMISSION_MODE ?? 'workspace-write'`, политика одобрения автоматически = `never` при `danger-full-access`, иначе `ask`. Работает и для headless/ACP.

### Способ 3 — постоянно, через свой patch-слой профиля
В файл `~/.dsh/profiles/web/cordis.patch.yml` (верхнеуровневый YAML-массив) добавить:

```yaml
- id: sandbox-policy
  config:
    mode: danger-full-access
- id: approval
  config:
    policy: never
```

### Важные нюансы
- `policy: never` — это НЕ «разрешать всё», а «не спрашивать и молча отклонять» то, что требует одобрения. Если оставить песочницу `workspace-write`, записи вне рабочей папки будут просто отклоняться без вопроса. Чтобы агент реально мог всё — нужен именно пресет `danger-full-access`.
- Разовая эскалация изнутри сессии: инструменту `bash` можно передать `sandbox_permissions: "danger-full-access"` + `justification` — тогда появится один запрос на подтверждение.
- Текущее состояние видно в системном промпте сессии («Current DSH file policy…», «Approval prompts are disabled…»).

---

## 2. Запуск (CLI `dsh`)

```sh
pnpm dsh web                      # веб-UI на http://127.0.0.1:3080 (алиас --profile web)
pnpm dsh --profile headless "задача"   # одна задача: ответ в stdout и выход (нужен DEEPSEEK_API_KEY)
dsh --profile <name>              # произвольный профиль из ~/.dsh/profiles/<name> (создаётся через dsh plugin)
dsh plugin --profile <name> add <pkg>   # управление плагинами профиля (форвард в pnpm)
```

Флаги лаунчера: `--patch file.yml` (доп. overlay, повторяемый), `--dump-config`, `--dump-default-config`, `-V`.
Флаги самого web-приложения: `--host`, `--port`, `--no-open`, `--trusted-host`.
Свои переопределения композиции: `~/.dsh/profiles/web/cordis.patch.yml`. Ключи API — в `~/.dsh/.credentials.yaml`.

---

## 3. Slash-команды для человека (в чате, не уходят модели)

| Команда | Что делает |
|---|---|
| `/permission <preset>` | сменить пресет прав (песочница + одобрение) |
| `/plan [off\|сообщение]` | войти/выйти из режима планирования |
| `/goal [<цель>\|clear\|edit …\|pause\|resume]` | поставить/посмотреть цель долгой автономной задачи |
| `/compact` | сжать старую историю разговора |
| `/export` | скачать лог сессии ZIP-архивом (web) |
| `/feedback <text>` | отправить отзыв о сессии |

---

## 4. Инструменты агента (то, что видит модель)

Полные JSON-схемы: [docs/tool-catalog.md](docs/tool-catalog.md) (генерируется автоматически).

### Файлы
| Инструмент | Назначение |
|---|---|
| `read`, `write`, `edit` | чтение / создание-замена / точечная замена текста; действует правило «сначала прочитай» |
| `read_image` | прочитать PNG/JPEG/WebP/GIF как изображение (нужна мультимодальная модель) |
| `str_replace_editor` | альтернативный редактор: view/create/str_replace/insert |
| `glob`, `grep` | поиск файлов по маске и по содержимому (встроенный ripgrep) |

### Оболочка и процессы
| Инструмент | Назначение |
|---|---|
| `bash` | одна команда в свежем шелле; параметры `workdir`, `timeoutMs`, `run_in_background`, `sandbox_permissions`+`justification` |
| `pwsh` | то же для Windows/PowerShell |
| `bash` / `pwsh` (persistent-варианты) | команды в живом PTY-шелле с сохранением состояния |
| `terminal_open/read/send/signal/list/close` | постоянные терминальные сессии (REPL, интерактив) |
| `job_list`, `job_output`, `job_kill` | управление фоновыми задачами (bash, PTY, субагенты) |

### Веб и код
| Инструмент | Назначение |
|---|---|
| `web_search` | поиск в интернете (1–4 запроса) |
| `web_fetch` | скачать содержимое URL |
| `run_code` | выполнить TS-программу, вызывающую другие инструменты как `await tools.name(...)` (режим code) |
| `lsp` | goToDefinition / findReferences / goToImplementation / hover |

### Планирование и автономность
| Инструмент | Назначение |
|---|---|
| `todo_write` | чеклист задач сессии |
| `exit_plan_mode` | показать план на утверждение (только в plan mode) |
| `create_goal`, `get_goal`, `update_goal` | долгая цель с автопродолжением раундов |
| `schedule_create/delete/list` | напоминания внутри живой сессии |

### Субагенты и оркестрация
| Инструмент | Назначение |
|---|---|
| `subagent` | делегировать задачу отдельному контексту (по умолчанию фон + продолжаемый) |
| `subagent_fork` | то же, one-shot, по умолчанию в фоне ожидания нет |
| `send_message`, `interrupt_agent`, `list_agents` | управление фоновыми субагентами |
| `report` | отчёт от ребёнка родителю (виден только внутри ребёнка) |
| `workflow` | JS-скрипт-оркестратор: `agent()`, `pipeline()`, `parallel()` над множеством субагентов |
| `ralph` | цикл «свежий агент на раунд» по неизменной цели |

### Прочее
| Инструмент | Назначение |
|---|---|
| `skill` | загрузить инструкции навыка из каталога |
| `ask_user_question` | задать человеку вопрос с вариантами |
| `session_search`, `session_trace`, `session_event_read/search/trace` | поиск по прошлым сессиям и событиям (opt-in) |
| `cordis_define/run/stop/undefine/inspect_*` | само-модификация: динамические плагины (осознанно выключено по умолчанию) |
| инструменты Agent Team (10 шт.) | экспериментальные командные агенты (выключены в базовом бандле) |

---

## 5. Куда смотреть в репозитории

| Нужно | Где |
|---|---|
| Архитектура | `docs/architecture.md`, `docs/capability-seams.md` |
| Все инструменты со схемами | `docs/tool-catalog.md` |
| Права/одобрение | `docs/subsystems/approval.md`, `docs/subsystems/permission-presets.md`, `docs/subsystems/sandbox.md` |
| Конфиг плагинов | `packages/bundle/base/cordis.patch.yml`, `docs/config-catalog.md` |
| Команды человека | `docs/subsystems/commands.md` |
| Как добавить инструмент | `docs/cookbook/adding-a-tool.md` |
| Мои заметки по запуску/модели | `README_LOCALL.md`, `OX_ALPHA_SETUP.md` |
