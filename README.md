# claude-desktop-plugins

Два плагина для Claude Code, которые дают Claude руки на вашем Mac — и дисциплину, чтобы этими руками не сжечь контекст:

- **browser** — ваш живой Chrome со всеми логинами через Playwright MCP (Extension Mode);
- **computer-use** — рабочий стол macOS через встроенный computer-use MCP.

## Зачем, если есть нативные механизмы

**Браузер.** Playwright MCP «из коробки» после каждого клика и перехода возвращает полный снапшот страницы — на реальной задаче это до ~114K токенов, съеденных разметкой, которую никто не читал. Здесь сервер запускается с `--snapshot-mode=none --image-responses=omit`, а скилл учит Claude лестнице чтения: `browser_find` (grep по дереву) → `browser_evaluate` (точечный JS, ~100 токенов) → scoped snapshot → полный снапшот только как последнее средство. Та же задача — ~27K токенов. Плюс это ваш настоящий Chrome: куки, сессии, 2FA уже пройдены.

**Рабочий стол.** Встроенный computer-use без дисциплины скатывается в петлю «скриншот → клик → скриншот»: каждый кадр ~1–1.8K токенов, и вся история кадров пере-отправляется на каждой итерации (18 кадров ≈ 279K токенов). Скилл задаёт правила: когда кадр действительно нужен, чем его заменить (`zoom`, проверка по живому состоянию), как батчить цепочки действий через `computer_batch` и почему пиксельный клик — последний ярус после API/MCP/CLI и браузера.

## Prerequisites

- macOS, десктопный Claude Code.
- Для **browser**: Chrome/Edge/Chromium + расширение [Playwright Extension](https://chromewebstore.google.com/detail/playwright-extension/mmlmfjhmonkocbjadbfplnigmagldckm) из Chrome Web Store.
- Для **computer-use**: план Pro или Max (research preview, на Team/Enterprise недоступен).

## Установка

```bash
claude plugin marketplace add https://github.com/beCyborg/claude-desktop-plugins
claude plugin install browser@becyborg-desktop
claude plugin install computer-use@becyborg-desktop
```

Полный HTTPS-URL — защита от клона по SSH: shorthand `owner/repo` уходит на SSH, а SSH-ключа на GitHub у нового пользователя обычно нет. Переменная `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1` нужна только для shorthand-формы; при полном HTTPS-URL она не требуется. `becyborg-desktop` — имя маркетплейса, который добавляет первая команда; в командах 2–3 подставляется именно оно.

Дальше по README плагинов:

| Плагин | Что настроить после установки |
|---|---|
| [browser](plugins/browser/README.md) | расширение из Chrome Web Store; опционально токен со страницы расширения (убирает диалог подтверждения) |
| [computer-use](plugins/computer-use/README.md) | `/mcp` → `computer-use` → Enable; выдать macOS-разрешения Accessibility и Screen Recording |

## Проверка

- **browser**: «покажи вкладки браузера» → Claude возвращает список открытых вкладок.
- **computer-use**: «сделай скриншот рабочего стола» → Claude запрашивает доступ и описывает экран.

Если computer-use не появился в списке `/mcp` — сверьте prerequisites (macOS, план Pro или Max). Если скриншот падает — выдайте оба macOS-разрешения: Системные настройки → Конфиденциальность и безопасность → Accessibility и Screen Recording.

## Безопасность

- Оба плагина дают Claude **ваши реальные привилегии** — браузер под вашими логинами, рабочий стол под вашей учёткой. Скиллы требуют явного подтверждения перед необратимыми действиями (отправка, покупка, удаление).
- `browser_run_code_unsafe` (произвольный JS) — под гейтом: только доверенные страницы, по явной необходимости.
- Токен расширения — сенситивное поле `userConfig`: уезжает в Связку ключей macOS, в файлы конфига и в репозиторий не попадает.

## Версии и обновление

У каждого плагина в `plugin.json` задан явный semver — он же ключ кэша обновлений. Пуш без бампа версии до получателей не доезжает.

У сторонних маркетплейсов (всех, кроме официальных от Anthropic) авто-обновление у получателей **выключено по умолчанию**. Чтобы получить новую версию:

```bash
claude plugin update browser@becyborg-desktop
claude plugin update computer-use@becyborg-desktop
```

Либо один раз включить авто-обновление: `/plugin` → Marketplaces → `becyborg-desktop`.

Имена маркетплейса и плагинов зафиксированы с первого релиза: переименование ломает существующие установки.

## Лицензия

MIT — см. [LICENSE](LICENSE).
