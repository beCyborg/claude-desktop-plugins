---
name: computer-use
description: "Use when the user wants to act on their Mac desktop or a native macOS app (Finder, Заметки/Notes, Календарь, Карты, Photos, Системные настройки, Сообщения и т.п.) — screenshot the screen, click, type, open an app, cross-app workflows. Third-person tool: drives the built-in computer-use MCP. Triggers: «на моём компьютере», «на рабочем столе», «на экране», «в приложении», «открой приложение», «покажи экран», «сделай скриншот рабочего стола», «в Заметках/Календаре/Картах». Do NOT use for: web/logged-in sites → скилл browser (плагин browser, Playwright MCP); terminal/shell → Bash; project files → Read/Write/Edit; web search → поисковые инструменты."
---

# Computer Use — управление рабочим столом Mac

Управление native-приложениями macOS через встроенный computer-use MCP. Полный префикс инструментов — `mcp__computer-use__*`; ниже короткие имена.

**Запрос пользователя:** `$ARGUMENTS`

## Что здесь, а что в инструкциях MCP-сервера

Tier-модель (read/click/full), frontmost-гейт, link-safety, запрет финансовых операций и правило «сначала `request_access`» уже загружены как инструкции MCP-сервера в каждой сессии — здесь их НЕ дублируем (иначе рассинхрон при обновлениях). Ниже только дельта сетапа и рабочий процесс.

**Локальная дельта маршрутизации:**
- Веб / залогиненные сайты → скилл browser (плагин browser из этого же маркетплейса, Playwright MCP в extension mode) — DOM-aware и дешевле пикселей.
- Терминал → Bash tool. Файлы проекта → Read/Write/Edit.
- Приложение имеет свой MCP (Slack, Obsidian, задачники…) или CLI → туда. computer-use — последнее средство для native UI без API.

## Иерархия поверхностей (от дешёвой к дорогой)

API/MCP/CLI → структурный доступ → браузер (скилл browser) → computer-use. Пиксельный клик дороже по токенам и менее надёжен, чем вызов API — это последний ярус, а не первый ход.

## Скриншот — самая дорогая операция

~1000–1800 токенов за кадр, и история пере-отправляется каждой итерацией (18 кадров ≈ 279K токенов). Поэтому скриншот **не на каждый шаг**.
- **Нужен:** после открытия приложения / навигации / ожидания загрузки; перед необратимым действием; когда UI изменился непредсказуемо.
- **Избыточен:** повторный кадр статичного экрана; «проверка» после каждого клика в предсказуемой цепочке — вместо этого один скриншот в конце `computer_batch`.
- **Дешёвые замены кадру:** `list_granted_applications` (что разрешено/запущено); `zoom(region=[x0,y0,x1,y1])` — hi-res зум региона последнего скриншота для мелкого текста; верификация по живому состоянию (файл появился, вывод CLI, clipboard). Отсутствие tool-error ≠ успех — проверяй результат по факту.

## Workflow

1. **Доступ.** `request_access(apps=["Заметки","Finder"], reason="…")` — `apps` (не applications) + одно предложение `reason`. Опц. гранты: `clipboardRead`, `clipboardWrite` (включает clipboard fast path для многострочного `type`), `systemKeyCombos` (quit/switch app, lock). Нужно ещё приложение позже — вызови снова, ранее выданные остаются.
2. **Осмотр.** Один `screenshot()` для исходного состояния (или `list_granted_applications()`, если достаточно проверить allowlist).
3. **Действие.** `open_application(app="Заметки")` для запуска (работает на любом tier), затем клики/ввод. Механические self-contained цепочки — через `computer_batch`.
4. **Проверка.** По живому состоянию или одним финальным скриншотом — не после каждого шага.

Лимит итераций петли (скриншот→клик→скриншот) задавай явно: если 2–3 итерации не продвинули задачу — стоп, переформулировать или спросить пользователя.

## Инструменты (валидные сигнатуры)

| Действие | Вызов |
|----------|-------|
| Скриншот | `screenshot(save_to_disk?)` — save_to_disk только чтобы отдать файл пользователю |
| Зум региона | `zoom(region=[x0,y0,x1,y1])` — read-only; координаты кликов всё равно от полного скриншота |
| Клик | `left_click(coordinate=[x,y], text?)` — text = модификаторы («shift», «ctrl+shift») |
| Двойной / тройной / правый / средний | `double_click` / `triple_click` (выделить строку) / `right_click` / `middle_click` — `coordinate=[x,y]` |
| Ввод текста | `type(text="…")` — многострочный поддерживается |
| Клавиша / аккорд | `key(text="cmd+c", repeat?)` — параметр `text`, не key |
| Удержание клавиши | `hold_key(text="shift", duration=1.5)` — НЕ action:press/release |
| Удержание мыши | `left_mouse_down()` / `left_mouse_up()` — по текущей позиции курсора |
| Перемещение / курсор | `mouse_move(coordinate=[x,y])` / `cursor_position()` |
| Перетаскивание | `left_click_drag(coordinate=[x,y], start_coordinate?)` — coordinate = конец |
| Прокрутка | `scroll(coordinate=[x,y], scroll_direction="down", scroll_amount=3)` — все три обязательны |
| Открыть приложение | `open_application(app="Notes")` — параметр app |
| Мультимонитор | `switch_display(display="…")` — имя из ноты скриншота или "auto" |
| Ожидание | `wait(duration=2)` — параметр duration, не seconds |
| Allowlist / гранты | `list_granted_applications()` — без параметров |
| Буфер обмена | `read_clipboard()` / `write_clipboard(text="…")` — по грантам |
| Батч | `computer_batch(actions=[…])` |

**`computer_batch` — семантика:** действия исполняются последовательно, стоп на первой ошибке; frontmost-гейт проверяется перед КАЖДЫМ действием; координаты внутри батча — относительно ПРЕД-батчевого скриншота (мид-батч скриншоты только для инспекции). Батчить **только** self-contained цепочки (заполнение формы, хоткеи, клики по известным целям). Exploratory-навигацию и error-recovery не батчить: промах действия 1 → хвост исполняется по устаревшему состоянию.

Пример: `computer_batch(actions=[{"action":"left_click","coordinate":[100,200]},{"action":"type","text":"hello"},{"action":"key","text":"Return"}])`

**Промахи кликов** — чаще всего по мелким целям: сначала `zoom` для точного прицела, клавиатурная навигация (tab/стрелки/Enter) надёжнее пиксельного клика по мелким контролам.

**При ошибке параметра** — сверить живую схему через `ToolSearch("select:mcp__computer-use__<tool>")`, схема побеждает эту таблицу.

## Когда НЕ использовать

- **Веб / залогиненный сайт** → скилл browser (Playwright) — DOM-aware, дешевле и надёжнее пикселей.
- **Терминал** → Bash. **Файлы проекта** → Read/Write/Edit. **Веб-поиск** → поисковые инструменты.
- **Есть свой MCP/CLI у приложения** → используй его, не пиксели.
