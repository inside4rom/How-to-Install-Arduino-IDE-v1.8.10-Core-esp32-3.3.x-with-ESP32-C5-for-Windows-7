# ESP32-C5 / ESP32-S3 на Windows 7 — Arduino IDE 1.8.10 + ядро esp32 3.3.x

Пошаговая инструкция, как заставить компилироваться ядро **esp32 3.3.x**
(первая ветка с поддержкой ESP32-C5) на **Windows 7 SP1 x64** в Arduino IDE 1.8.10.
Проверено на ядре 3.3.5 (esptool 5.1.0), платы ESP32-S3 Zero и ESP32-C5.

## Проблема

Начиная с ядра esp32 3.3.x Espressif поставляет `esptool.exe`, `espefuse.exe`,
`espsecure.exe`, собранные PyInstaller'ом на **Python 3.13**. Python ? 3.9
официально не поддерживает Windows 7, поэтому любой вызов падает:

    [PYI-xxxx:ERROR] Failed to load Python DLL '...\_MEIxxxx\python313.dll'

Компиляция обрывается на генерации `bootloader.bin`. Откатиться на ядро 3.1.3,
где esptool работает, нельзя — там нет ESP32-C5.

| Ядро | esptool | Работает на Win7 | ESP32-C5 |
|---|---|---|---|
| 3.1.3 | 4.9.dev3 | да | нет |
| 3.3.5 | 5.1.0 (Python 3.13) | нет | да |
| 3.3.12 | 5.3.1 (Python 3.13) | нет | да |

## Решение

Поставить **неофициальный патченный Python 3.11 для Win7**, установить в него
esptool нужной версии и подменить три exe в папке ядра на pip-лаунчеры
(маленькие exe ~108 КБ, которые вызывают `C:\Python311\python.exe -m esptool`).
Ядро Arduino при этом не трогаем, `platform.txt` не правим.

---

## Требования

- Windows 7 SP1 x64, Arduino IDE 1.8.x (проверено на 1.8.10)
- ~3 ГБ места под ядро esp32
- Интернет на этой машине или любой другой ПК с Python ? 3.8 (для скачивания пакетов)
- Рекомендуется (не проверялось, обычно уже стоит): KB2999226 (UCRT),
  KB2533623, KB4474419 (SHA-2), VC++ Redistributable 2015–2022 x64.
  Если `python.exe` не стартует с ошибкой про `python311.dll` — ставить их.

## Что скачиваем

| Файл | Откуда | Зачем |
|---|---|---|
| `python-3.11.18-amd64-full.exe` | https://github.com/adang1345/PythonWin7/releases/download/v3.11.18/python-3.11.18-amd64-full.exe (та же сборка есть в репозитории PythonVista) | Python 3.11 с подменёнными Win8+ API (`api-ms-win-core-path-l1-1-0.dll` и др.) |
| `esptool-5.1.0.tar.gz` | https://files.pythonhosted.org/packages/c2/03/d7d79a77dd787dbe6029809c5f81ad88912340a131c88075189f40df3aba/esptool-5.1.0.tar.gz | Сам esptool. На PyPI есть **только** sdist, wheel нет |
| 21 файл `*.whl` | pypi.org через `pip download` (список ниже) | Зависимости esptool под Python 3.11 x64 |
| Ядро esp32 **3.3.5** | Менеджер плат Arduino IDE | Поддержка C5; версия esptool видна в пути `esptool_py\5.1.0` |

Версия esptool должна совпадать с папкой `packages\esp32\tools\esptool_py\<версия>`
вашего ядра (3.3.12 > 5.3.1).

---

## Шаг 1. Патченный Python 3.11

1. Установить `python-3.11.18-amd64-full.exe` в **`C:\Python311`**
   (путь без пробелов и кириллицы; потом он зашивается в лаунчеры — не переносить).
2. Галочку **«Add Python to PATH» НЕ ставить** — все команды ниже идут по полному
   пути, а второй Python в PATH только мешает.
3. Проверка:

       C:\Python311\python.exe -V
       :: Python 3.11.18
       C:\Python311\python.exe -m pip --version
       :: pip 24.0 from C:\Python311\Lib\site-packages\pip (python 3.11)

## Шаг 2. Ядро esp32 3.3.5

1. Файл > Настройки > Дополнительные ссылки для Менеджера плат:
   `https://espressif.github.io/arduino-esp32/package_esp32_index.json`
2. Инструменты > Плата > Менеджер плат > `esp32 by Espressif Systems` > версия **3.3.5** > Установить.
3. Компиляция на этом этапе падает с `PYI-xxxx ... python313.dll` — это ожидаемо.

## Шаг 3. Скачивание пакетов (на машине с интернетом)

Нужен любой Python ? 3.8 с pip ? 23. Папка назначения — `C:\esptool_pack`.

> На патченном Python 3.11 под Win7 `pip download` может падать с
> `WinError 10051` (сеть). Тогда качать другим Python (у меня уже был — 3.8 на той же
> машине) .

    python -m pip install --upgrade pip

    :: esptool — только sdist (с --only-binary pip ответит "from versions: none")
    python -m pip download esptool==5.1.0 --no-deps --no-binary :all: -d C:\esptool_pack

    :: зависимости — wheel под Python 3.11 x64
    python -m pip download "pyserial>=3.3" "cryptography<42" "ecdsa>=0.16" ^
      "reedsolo>=1.5.3,<1.8" "PyYAML>=5.1" intelhex "bitstring>=3.1.6" ^
      "click<8.2" rich-click "argcomplete>=3" pyelftools ^
      -d C:\esptool_pack --python-version 3.11 --implementation cp ^
      --abi cp311 --abi abi3 --abi none --platform win_amd64 --platform any ^
      --only-binary=:all:

Содержимое `C:\esptool_pack` (версии могут быть новее — нормально):

    esptool-5.1.0.tar.gz
    pyserial-3.5-py2.py3-none-any.whl          cryptography-41.0.7-cp37-abi3-win_amd64.whl
    ecdsa-0.19.2-py2.py3-none-any.whl          reedsolo-1.7.0-py3-none-any.whl
    pyyaml-6.0.3-cp311-cp311-win_amd64.whl     intelhex-2.3.0-py2.py3-none-any.whl
    bitstring-5.0.0-py3-none-any.whl           tibs-2.0.2-cp311-abi3-win_amd64.whl
    click-8.1.8-py3-none-any.whl               rich_click-1.9.9-py3-none-any.whl
    rich-15.0.0-py3-none-any.whl               markdown_it_py-4.2.0-py3-none-any.whl
    mdurl-0.1.2-py3-none-any.whl               pygments-2.21.0-py3-none-any.whl
    argcomplete-3.7.2-py3-none-any.whl         pyelftools-0.33-py3-none-any.whl
    cffi-2.1.1-cp311-cp311-win_amd64.whl       pycparser-3.12-py3-none-any.whl
    six-1.17.0-py2.py3-none-any.whl            typing_extensions-4.16.0-py3-none-any.whl
    colorama-0.4.6-py2.py3-none-any.whl

Без pip можно скачать то же самое браузером: `https://pypi.org/project/<имя>/<версия>/#files`.

## Шаг 4. Установка в Python 3.11 (офлайн)

`install_esptool.bat` — **сохранять в кодировке ANSI**, без `chcp 65001` и без
кириллицы (иначе на Win7: `The system cannot write to the specified device`).

    @echo off
    cd /d C:\Python311

    echo === 1. deps ===
    C:\Python311\python.exe -m pip install --no-index --no-deps --find-links C:\esptool_pack pyserial cryptography ecdsa reedsolo PyYAML intelhex bitstring click rich-click argcomplete pyelftools cffi six tibs typing_extensions colorama markdown-it-py pygments pycparser mdurl rich

    echo === 2. esptool (no-deps) ===
    C:\Python311\python.exe -m pip install --no-index --no-build-isolation --no-deps --find-links C:\esptool_pack C:\esptool_pack\esptool-5.1.0.tar.gz

    echo === 3. check ===
    C:\Python311\python.exe -m esptool version
    C:\Python311\python.exe -m esptool --chip esp32c5 version
    dir C:\Python311\Scripts\esp*.exe
    pause

Почему `--no-deps` для esptool: в его метаданных `cryptography>=43.0.0`, а у нас
41.0.7. Для `elf2image` / `merge-bin` / `write-flash` cryptography не нужна —
проверено, работает. Поведение cryptography 43+ на Win7 не проверялось.

Ожидаемо: `esptool v5.1.0`, в `C:\Python311\Scripts` появились
`esptool.exe`, `espefuse.exe`, `espsecure.exe`, `esp_rfc2217_server.exe` (~108 КБ).
Предупреждения pip «...is not on PATH» безвредны.

## Шаг 5. Подмена утилит в ядре

Закрыть Arduino IDE. `patch_arduino_esptool.bat` (ANSI):

    @echo off
    set ESP=%LOCALAPPDATA%\Arduino15\packages\esp32\tools\esptool_py\5.1.0
    cd /d "%ESP%" || (echo folder not found & pause & exit /b 1)

    if not exist esptool_orig.bak.exe   ren esptool.exe   esptool_orig.bak.exe
    if not exist espefuse_orig.bak.exe  ren espefuse.exe  espefuse_orig.bak.exe
    if not exist espsecure_orig.bak.exe ren espsecure.exe espsecure_orig.bak.exe

    copy /y C:\Python311\Scripts\esptool.exe   .
    copy /y C:\Python311\Scripts\espefuse.exe  .
    copy /y C:\Python311\Scripts\espsecure.exe .

    echo === test ===
    esptool.exe version
    pause

Должно вывести `esptool v5.1.0` из папки ядра, без `PYI-ERROR`.

## Шаг 6. Компиляция

Запустить IDE, выбрать плату (`ESP32S3 Dev Module` / `ESP32C5 Dev Module`),
выставить Flash Mode / Size / PSRAM / Partition под свою плату,
включить «Показать подробный вывод при компиляции», собрать.

### Если падает `gen_esp32part.exe`
Тоже PyInstaller. Рядом лежит `gen_esp32part.py`
(`%LOCALAPPDATA%\Arduino15\packages\esp32\hardware\esp32\3.3.5\tools\`).
Создать там `gen_esp32part.bat`:

    @echo off
    "C:\Python311\python.exe" "%~dp0gen_esp32part.py" %*

и в `platform.txt` заменить `gen_esp32part.exe` > `gen_esp32part.bat`.



---

## Troubleshooting

| Симптом | Причина / действие |
|---|---|
| `api-ms-win-core-path-l1-1-0.dll` или `python313.dll` при компиляции | Вызывается оригинальный exe. Проверить: IDE была закрыта при подмене; `esptool.exe` в папке ядра ~108 КБ, а не ~13 МБ; есть `esptool_orig.bak.exe` |
| `Failed to load ... python311.dll` | Не хватает UCRT / VC++ Redist (см. Требования) |
| `WinError 10051` при pip на Python 3.11 | Сеть у патченной сборки не работает — качать другим Python (шаг 3) |
| `No matching distribution found for esptool==5.1.0 (from versions: none)` | Использован `--only-binary`, а у esptool нет wheel. Качать с `--no-binary :all:` |
| `No matching distribution found for cryptography>=43.0.0` | Забыт `--no-deps` при установке esptool |
| `No matching distribution found for <пакет>` на шаге 4 | В `C:\esptool_pack` нет этого wheel — докачать |
| pip ставит не туда | Всегда `C:\Python311\python.exe -m pip ...`, не просто `pip` |
| bat падает на первом `echo` | Файл в UTF-8 или есть `chcp 65001` — пересохранить в ANSI |

## Ограничения

- Лаунчеры содержат абсолютный путь `C:\Python311\python.exe`. Переместить Python — сломаются все три exe.
- Обновление ядра через Менеджер плат заменит папку `esptool_py\<версия>` > повторить шаги 3–5 под новую версию esptool. Не нажимать «Обновить» в менеджере плат.
- Secure Boot / Flash Encryption (`espsecure`, `espefuse`) с cryptography 41 не проверялись.

## Бекап и перенос

Сохранить:

    %LOCALAPPDATA%\Arduino15\     (можно без подпапки staging)
    C:\Python311\
    C:\esptool_pack\
    <папка скетчей>\libraries\    (путь — Файл > Настройки)

Восстановление на другом пк Win7: поставить ту же IDE, запустить и закрыть один раз,
скопировать `Arduino15` поверх, установить Python 3.11.18 **в тот же путь**
`C:\Python311`, выполнить шаги 4–5. На Win10/11 подмена не нужна —
вернуть `*_orig.bak.exe` обратно.

## Файлы репозитория

    README.md       
    esptool_pack/              (~8 МБ: wheel-файлы + esptool-5.1.0.tar.gz + готовые exe esptool.exe... )


## Ссылки

- Патченный Python: https://github.com/adang1345/PythonWin7 (и https://github.com/adang1345/PythonVista)
- esptool: https://github.com/espressif/esptool
- Ядро esp32: https://github.com/espressif/arduino-esp32
- Arduino IDE 1.8.x: https://github.com/arduino/Arduino
