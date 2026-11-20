# Подключение MCP-1C к базе 1С — инструкция для ИИ

Файл самодостаточен: его можно положить в корень любого проекта на 1С и выполнять по шагам.
Все примеры — Windows + PowerShell 5.1. Значения в `<угловых скобках>` подставляются из проекта.

**Результат:** агент получает инструменты `mcp__1c__*` — выполнение запросов на языке запросов 1С (только чтение), метаданные, структура объектов, журнал регистрации, подсчёт записей.

**Ключевое правило:** бинарник `mcp-1c` не поставляется вместе с проектом и не берётся из общего каталога машины — он скачивается в каталог проекта:

```
<корень проекта>\tools\mcp-1c\mcp-1c-windows-amd64.exe
```

и в конфигурацию MCP попадает именно этот путь.

---

## 0. Что читаем в проекте

Из `.v8-project.json` берём:

| Поле | Зачем |
|------|-------|
| `v8path` | каталог `bin` платформы — нужен для `--platform` и для публикации |
| `databases[].type` / `.path` | `file` → каталог с `1Cv8.1CD`; иначе клиент-серверная, `path` вида `srv-1c\имя_базы` |
| `databases[].user` / `.password` | учётка базы (и для `--install`, и для HTTP-сервиса) |
| `databases[].webUrl` | адрес публикации, база для `--base` |
| `webPath` | корень Apache (если публикуем сами) |

Если файла нет или база не найдена — сначала `/db-list`, потом продолжать.

---

## 1. Определить тип базы

**Файловая** — `"type": "file"`, `path` указывает на каталог, внутри есть `1Cv8.1CD`.

**Клиент-серверная** — `"type": "server"` либо `path` в виде `сервер\база` (без диска и слэшей).
Публикация для неё описывается строкой соединения `Srvr="<сервер>";Ref="<база>";`.

Дальше шаги почти одинаковые, различия помечены **[файловая]** / **[сервер]**.

---

## 2. Скачать бинарник в каталог проекта

```powershell
$root   = '<корень проекта>'
$dir    = Join-Path $root 'tools\mcp-1c'
$exe    = Join-Path $dir 'mcp-1c-windows-amd64.exe'
New-Item -ItemType Directory -Force -Path $dir | Out-Null

$rel   = Invoke-RestMethod 'https://api.github.com/repos/feenlace/mcp-1c/releases/latest'
$asset = $rel.assets | Where-Object { $_.name -eq 'mcp-1c-windows-amd64.exe' }
Invoke-WebRequest -Uri $asset.browser_download_url -OutFile $exe -UseBasicParsing

# сверка контрольной суммы
$raw  = (Invoke-WebRequest -Uri ($rel.assets | Where-Object { $_.name -eq 'checksums.txt' }).browser_download_url -UseBasicParsing).Content
# В PowerShell 5.1 .Content для текстового ассета приходит как byte[], без декодирования
# строка ниже даст пустой $want и молча пропустит проверку
$text = if ($raw -is [byte[]]) { [Text.Encoding]::UTF8.GetString($raw) } else { [string]$raw }
$want = (($text -split "`r?`n") | Where-Object { $_ -match 'mcp-1c-windows-amd64\.exe' }) -split '\s+' | Select-Object -First 1
$got  = (Get-FileHash $exe -Algorithm SHA256).Hash.ToLower()
if (-not $want) { throw 'checksums.txt: нет строки для mcp-1c-windows-amd64.exe' }
if ($got -ne $want.Trim().ToLower()) { throw "checksum mismatch: $got != $want" }

& $exe --version          # ожидаем строку вида: mcp-1c version v1.24.0
```

Добавить в `.gitignore` проекта: `tools/mcp-1c/`.

> **Временный каталог.** `mcp-1c` создаёт рабочий каталог в `%TEMP%`. Если он недоступен на запись (песочница/ограничения агента), получите `Installation error: creating temp dir: ...: Access is denied`. Перед вызовом задайте `$env:TEMP`/`$env:TMP` в каталог проекта:
>
> ```powershell
> $env:TEMP = Join-Path $root '.tmp-mcp'; $env:TMP = $env:TEMP
> New-Item -ItemType Directory -Force -Path $env:TEMP | Out-Null
> ```
>
> Этого достаточно на шаге 2, но не спасает `--install`, если песочница запрещает запись в каталог базы (см. шаг 3, «Если `--install` падает на DESIGNER»).

> **Версии бинарника и расширения связаны.** Ставьте расширение тем же бинарником, которым потом работаете. Если позже обновляете бинарник — переустановите расширение (шаг 3). Иначе сервер падает с `mcp-1c error: server is closing: EOF`.

---

## 3. Установить расширение MCP_HTTPService в базу

Сначала **освободить базу**: остановить Apache (веб-сеансы держат файловую базу, а установка обновляет конфигурацию БД) и закрыть сеансы 1С.

```powershell
Get-Process httpd -ErrorAction SilentlyContinue | Stop-Process -Force
```

**[файловая]**

```powershell
& $exe --install "<путь к каталогу базы>" --platform "<v8path>\1cv8.exe" --db-user "<пользователь>"
```

`--platform` передавайте явно: если на машине стоит несколько платформ (например 8.3.27 и 8.5.1), автопоиск может выбрать не ту. Минимальная поддерживаемая платформа — 8.3.14.

**[сервер]** (строка `сервер\база`)

```powershell
& $exe --install "<сервер>\<база>" --server --db-user "<пользователь>" --db-password "<пароль>"
```

Правила, без которых установка падает:

- **Не передавайте пустой пароль.** `--db-password ""` ломает авторизацию в конфигураторе с ошибкой `1C DESIGNER failed ... Пользователь ИБ не идентифицирован`. Если пароля нет — просто не передавайте ключ.
- Кавычки вокруг `--db-user` обязательны, если имя русское.
- Успех — строка `Extension installed successfully.` и `exit=0`.

> `$exe` — переменная из шага 2. Если выполняете шаги отдельными сессиями, задайте её заново.
>
> Установка **меняет конфигурацию базы** (добавляет расширение и обновляет БД). На рабочей базе делайте это только с согласия пользователя и после бэкапа (`/db-dump-dt`).

### Если `--install` падает на DESIGNER

Симптом: `Installation error: loading extension config: 1C DESIGNER failed (exit code 1): Ошибка доступа к файлу '<база>\1Cv8.cgr.cfl'. 5(0x00000005): Отказано в доступе` — при этом PowerShell в тот же каталог пишет нормально. Это ограничение записи для дочерних процессов (песочница агента), а не проблема базы. `--install` сам копирует `.cfe` в temp и вызывает `1cv8 DESIGNER`; те же два вызова можно выполнить вручную, ничем не отличаясь по результату (`.cfe` лежит в том же релизе, что и бинарник):

```powershell
$cfe = Join-Path $dir 'MCP_HTTPService.cfe'
Invoke-WebRequest -Uri ($rel.assets | Where-Object { $_.name -eq 'MCP_HTTPService.cfe' }).browser_download_url -OutFile $cfe -UseBasicParsing

$designer = "<v8path>\1cv8.exe"
$out      = Join-Path $env:TEMP 'designer-out.txt'

& $designer DESIGNER /F"<путь к каталогу базы>" /N"<пользователь>" /LoadCfg $cfe -Extension MCP_HTTPService /Out $out
& $designer DESIGNER /F"<путь к каталогу базы>" /N"<пользователь>" /UpdateDBCfg -Extension MCP_HTTPService /Out $out
```

Ожидаемо: `/LoadCfg` пишет в `$out` «Загрузка конфигурации успешно завершена»; `/UpdateDBCfg` может не вывести ничего — это нормально. Проверка результата:

```powershell
& "<v8path>\ibcmd.exe" infobase config extension list --db-path="<путь к каталогу базы>" --user="<пользователь>" | Select-String -Pattern 'MCP' -Context 0,6
```

Нужны строки `name: "MCP_HTTPService"`, `active: yes` и `version`, совпадающая с `GET .../hs/mcp-1c/version` (на релизе v1.24.0 — `0.4.10`).

---

## 4. Снять безопасный режим и защиту от опасных действий

Расширение, впервые загруженное в базу, приходит с включённым безопасным режимом.

**[файловая]** — через `ibcmd` (есть в комплекте платформы):

```powershell
$ib = "<v8path>\ibcmd.exe"
& $ib infobase config extension update --db-path="<путь к каталогу базы>" `
      --user="<пользователь>" --name=MCP_HTTPService `
      --safe-mode=no --unsafe-action-protection=no
```

Если база занята — будет `Ошибка исключительной блокировки информационной базы`: остановите Apache и повторяйте.

**[сервер]** — `ibcmd` в навыках не поддержан; снять флаги в Конфигураторе: «Конфигурация → Расширения конфигурации → MCP_HTTPService → свойства → Безопасный режим = Нет, Защита от опасных действий = Нет».

---

## 5. Опубликовать базу (HTTP-сервис)

Сервис живёт по адресу `<webUrl>/hs/mcp-1c`, где `<webUrl>` — например `http://localhost:8081/MyBase`.
Публикация должна разрешать сервисы расширений. В `default.vrd` это:

```xml
<httpServices publishByDefault="true" publishExtensionsByDefault="true"/>
<ws pointEnableCommon="true" publishExtensionsByDefault="true"/>
```

Если публикации ещё нет — `/web-publish <база>` (навык сам создаст `default.vrd`, настроит Apache и запустит его). Если есть — проверьте `publishExtensionsByDefault="true"` в `<webPath>\publish\<AppName>\default.vrd`.

**После установки/обновления расширения Apache обязательно перезапустить** — веб-модуль 1С кеширует конфигурацию, до перезапуска новые сервисы отдают 404.

```powershell
& "<webPath>\bin\httpd.exe" -t -d <webPath>        # проверка конфигурации
Get-Process httpd | Stop-Process -Force
Start-Process "<webPath>\bin\httpd.exe" -ArgumentList '-d', '<webPath>' -WindowStyle Hidden
```

---

## 6. Проверить сервис — до подключения к агенту

```powershell
$base = '<webUrl>/hs/mcp-1c'

Invoke-WebRequest "$base/version" -UseBasicParsing | Select-Object -ExpandProperty Content
# { "version": "0.4.10" }

$body = @{ query = 'ВЫБРАТЬ ПЕРВЫЕ 3 Ссылка, Наименование ИЗ Справочник.Валюты КАК К' } | ConvertTo-Json -Compress
Invoke-WebRequest "$base/query" -Method Post -Body ([Text.Encoding]::UTF8.GetBytes($body)) `
    -ContentType 'application/json' -UseBasicParsing | Select-Object -ExpandProperty Content
```

Ожидаемо: `version` отвечает, `/query` возвращает `{ "columns": [...], "rows": [...], "total": N }`.

Полезные методы: `GET /configuration`, `GET /metadata`, `GET /object/{type}/{name}`, `POST /query`, `POST /validate-query`, `POST /eventlog`, `GET /extensions`.

---

## 7. Подключить как MCP-сервер

Подключить MCP сервер к текущей среде (claudeCode, dsh, cursor)
В зависимости от среды инструкции разные, поэтому тут не описаны

**Kilo (CLI / VS Code)** — проект `kilo.json`, имя сервера даёт префикс инструментов (`1c` → `mcp__1c__*`):

```jsonc
{
  "$schema": "https://app.kilo.ai/config.json",
  "mcp": {
    "1c": {
      "type": "local",
      "command": [
        "<проект>\\tools\\mcp-1c\\mcp-1c-windows-amd64.exe",
        "--base", "<webUrl>/hs/mcp-1c"
      ],
      "enabled": true
    }
  }
}
```

Проверка без клиента — рукопожатие через stdio (ответ на `initialize` должен содержать `capabilities.tools`, на `tools/list` — 17 инструментов `execute_query`, `get_metadata_tree`, `get_object_structure`, `get_event_log`, …). Пустой stdin сразу после запросов даёт `server is closing: EOF` — это EOF, а не рассинхрон версий.

---

## 8. Диагностика

| Симптом | Причина | Что делать |
|---|---|---|
| `mcp-1c error: server is closing: EOF` (после `Extension version verified. got=0.4.8`) | версия расширения старее, чем ждёт бинарник | переустановить расширение бинарником той же версии (шаг 3) |
| `mcp-1c error: server is closing: EOF` сразу при ручном тесте stdio, `resp` пуст, версии совпадают | stdin закрылся раньше ответа (передали файл и закрыли поток) | держать поток открытым; в клиенте не проявляется |
| `Installation error: creating temp dir: mkdir ...: Access is denied` | `%TEMP%` недоступен на запись процессу (песочница агента) | задать `$env:TEMP`/`$env:TMP` в каталог проекта (шаг 2) |
| `1C DESIGNER failed ... Ошибка доступа к файлу '<база>\1Cv8.cgr.cfl'. Отказано в доступе`, хотя PowerShell в тот каталог пишет | запрет записи дочерним процессам в каталог базы | установить расширение вручную (шаг 3, «Если `--install` падает на DESIGNER») |
| Проверка hash «проходит», но `$want` пустой | в PS 5.1 `.Content` для `checksums.txt` — `byte[]` | декодировать через `[Text.Encoding]::UTF8.GetString` (шаг 2) |
| `404` на `/hs/mcp-1c/...` | Apache не перезапущен после установки расширения; либо в `default.vrd` нет `publishExtensionsByDefault="true"`; либо расширение не применено | перезапустить Apache; проверить vrd; `/db-update -Extension MCP_HTTPService` |
| `1C DESIGNER failed: Пользователь ИБ не идентифицирован` | передан пустой `--db-password` | убрать ключ, оставить только `--db-user` |
| `Ошибка исключительной блокировки информационной базы` | базу держат веб-сеансы 1С | остановить Apache, повторить |
| `extension 'MCP_HTTPService' not found in the infobase` при `ibcmd` | в базе много расширений, часть инструментов видит не всё; либо расширение не установлено | проверить `1cv8 DESIGNER /DumpDBCfgList -AllExtensions`; при необходимости переустановить |
| `mcp-1c error: invalid argument` | аргументы переданы через `Start-Process -ArgumentList` массивом | запускать через `cmd /c ...` или через DSH-бандл |
| Инструменты `mcp__1c__*` «unknown tool» | бандл установлен в этом же ходе | новый ход/сессия |

---

## 9. Чек-лист

- [ ] тип базы определён (файловая / клиент-серверная)
- [ ] бинарник скачан в `<проект>\tools\mcp-1c\`, версия выведена, hash сверен
- [ ] расширение установлено (`Extension installed successfully.` либо вручную `LoadCfg`+`UpdateDBCfg`, проверено `ibcmd ... extension list`)
- [ ] безопасный режим и защита сняты
- [ ] vrd разрешает сервисы расширений, Apache перезапущен
- [ ] `GET .../hs/mcp-1c/version` отвечает
- [ ] `POST .../hs/mcp-1c/query` возвращает строки
- [ ] бандл установлен (`application: applied`)
- [ ] инструменты `mcp__1c__*` вызваны в новом ходе

---

## 10. Ограничения, о которых надо помнить

- Сервис выполняет **только чтение** (`ВЫБРАТЬ`/`SELECT`), в правах пользователя, под которым опубликована база. Изменять данные через MCP нельзя.
- Учётка должна иметь права на объекты, иначе запрос вернёт отказ по правам, а не данные.
- Журнал регистрации читается только с правом «Администрирование».
- Файловая база + Apache: веб-сервер должен быть на той же машине, что и файл базы.
- Публикация — единственная точка отказа: не запущен Apache → MCP не работает (в отличие от COM, где нужен сеанс 1С).
