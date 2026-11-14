# PCSX2 + Silent Hill 2 (NTSC-U, рус. перевод Metallist) — установка на Omarchy/Arch

**Система:** Omarchy 4.5 (Arch), Ryzen 5 5600X, Radeon RX 5600 XT (RADV), 2560×1440
**Дата:** 2026-11-14

**Результат:** PCSX2 2.9.179 (Vulkan, апскейл 3x), BIOS PAL SCPH-70008, игра запускается, геймпад Logitech F310 работает, карты памяти созданы.

## 1. Установка PCSX2

В официальных репозиториях Arch пакета нет, только AUR. Готовый бинарник (без сборки):

```bash
yay -S pcsx2-latest-bin
```

Альтернативы: `pcsx2` (стабильный, сборка из исходников), `pcsx2-git`, Flatpak, AppImage.

Проверка:

```bash
pacman -Q pcsx2-latest-bin
command -v pcsx2-qt
```

Запуск: меню приложений «PCSX2» или команда `pcsx2-qt`.

## 2. BIOS

Файл BIOS (4 МиБ `.bin`) кладём в стандартную папку PCSX2:

```bash
mkdir -p ~/.config/PCSX2/bios
unzip -j "путь/к/BIOS.zip" -d ~/.config/PCSX2/bios
```

Использованный дамп: SCPH-70008, Europe v2.0 (14/06/2004), 4 МиБ. PCSX2 определяет его как `Europe v02.00 (14/06/2004)`.

Первый запуск: мастер предложит выбрать BIOS — папка `~/.config/PCSX2/bios`, в списке выбрать SCPH-70008 (должен быть помечен как валидный).

> **Важно:** в `PCSX2.ini` путь к BIOS пишется **без кавычек**, иначе в логе будет `Configured BIOS ... does not exist, trying to find an alternative` (но BIOS всё равно подхватится автопоиском). Правильная строка:

```ini
[Filenames]
BIOS = Sony PlayStation 2 BIOS (E)(v2.0)(2004-06-14)[SCPH70008].bin
```

## 3. Игра

Образ: Silent Hill 2 (Greatest Hits) v2.01 [Russian][Metallist], NTSC-U (SLUS-20228), 3.7 ГБ iso. MD5 из комплекта совпадает с Redump — образ корректный.

```bash
mkdir -p ~/Games/PS2
mv "путь/к/образу.iso" "путь/к/MD5.txt" ~/Games/PS2/
```

В `PCSX2.ini` добавляем папку в библиотеку (кавычки не обязательны, с кавычками ок):

```ini
[GameList]
RecursivePaths = "/home/sansheng/Games/PS2"
```

Или через GUI: *Settings → Game List → Add Search Directory*.

## 4. Настройки (PCSX2.ini)

Файл: `~/.config/PCSX2/inis/PCSX2.ini` (редактировать при закрытом PCSX2!)

Ключевое:

```ini
[EmuCore/GS]
Renderer = 14          ; Vulkan (Auto=-1, DX11=3, OGL=12, SW=13, VK=14, DX12=15)
upscale_multiplier = 3 ; 3x = 1536x1344 при 1440p, хватает с запасом
```

Остальное оставлено по умолчанию. Для SH2 GameDB сам применяет:

- Speedhack EE Cycle Rate = 1
- Blending Accuracy = Full (нужно для корректного тумана)

Трогать вручную не нужно.

## 5. Геймпад Logitech F310

Подключается через SDL, драйверы не нужны. Привязки можно прописать в `[Pad1]` вместе с клавиатурными (несколько значений на одну кнопку — это нормально):

```ini
[Pad1]
Up = Keyboard/Up
Up = SDL-0/DPadUp
; ... аналогично для всех кнопок
Cross = SDL-0/FaceSouth      ; A
Circle = SDL-0/FaceEast      ; B
Square = SDL-0/FaceWest      ; X
Triangle = SDL-0/FaceNorth   ; Y
L2 = SDL-0/+LeftTrigger
R2 = SDL-0/+RightTrigger
LUp = SDL-0/-LeftY           ; стики: -LeftX/+LeftX/-LeftY/+LeftY
; ...
```

Проще в GUI: *Settings → Controllers → Pad 1 → Automatic Mapping*.

Проверка в логе: `SDLInputSource: Opened gamepad 1 ... Logitech F310`.

## 6. Карты памяти

Создаются автоматически при первом запуске: `~/.config/PCSX2/memcards/Mcd001.ps2` и `Mcd002.ps2` (по 8 МБ).

Если нужно вручную: *Settings → Memory Cards → Create*.

## 7. Проверка, что всё работает

Запуск игры напрямую из терминала:

```bash
pcsx2-qt "~/Games/PS2/...iso"
```

Смотреть лог:

```bash
tail -50 ~/.config/PCSX2/logs/emulog.txt
```

Признаки успеха:

- `BIOS Found: Europe v02.00 ...` и нет строки `Configured BIOS ... does not exist`
- `Mesa ... radv / AMD Radeon RX 5600 XT (RADV NAVI10)` — Vulkan работает
- `ELF ... is executing` + `Game CRC = ...`
- `SDLInputSource: Opened gamepad ... F310`
- В окне игры внизу OSD: `Vulkan 1536x1344 (3x) ... VPS: 60`

Горячие клавиши: `Alt+Enter` — фуллскрин, `Esc` — меню PCSX2, `F1`/`F3` — сейв/лоад состояния, `F8` — скриншот. 16:9 включается в меню самой игры (*Options → Screen Position*), хаки не нужны.

Играть удобнее всего через ярлык из меню Omarchy — см. раздел 9.

## 8. Частые мелочи

- BIOS PAL(E) + игра NTSC-U работают нормально; регион BIOS на частоту игры не влияет.
- `BIOS rom1/rom2 module not found` и `Failed to open or read NVRAM` — нормально для слим-дампа, PCSX2 создаёт NVRAM сам.
- `Failed to inhibit screen saver` — PCSX2 не смог заблокировать блокировку экрана (hypridle); при долгой игре экран может гаснуть.
- AUR-пакет обновляется вместе с системой: `yay -Syu`.

## 9. Ярлык запуска (меню Omarchy / .desktop)

Чтобы игра запускалась в один клик (сразу PCSX2 + образ, без мастера), создан файл ярлыка:

```
~/.local/share/applications/silent-hill-2.desktop
```

Содержимое:

```ini
[Desktop Entry]
Type=Application
Name=Silent Hill 2
Comment=Silent Hill 2 (PS2, рус. перевод Metallist) через PCSX2
Exec=uwsm-app -- pcsx2-qt "/home/sansheng/Games/PS2/SLUS_202.28.Silent Hill 2 (Greatest Hits) [v2.01] [Russian] [Metallist].iso"
TryExec=pcsx2-qt
Icon=silent-hill-2
Terminal=false
Categories=Game;Emulator;
Keywords=Silent Hill;SH2;PS2;PCSX2;
StartupNotify=true
StartupWMClass=pcsx2-qt
```

> **Важно:** в `Exec` путь к образу **абсолютный** (в `.desktop` не раскрывается `~`), а из-за пробелов и скобок он взят в кавычки. `uwsm-app` запускает приложение в отдельном systemd-scope, как это принято в Omarchy.

Иконка — обложка Silent Hill 2 v2.01 из libretro-thumbnails:

```
~/.local/share/icons/hicolor/512x512/apps/silent-hill-2.png
```

Скачать/заменить иконку вручную:

```bash
curl -sL "https://thumbnails.libretro.com/Sony%20-%20PlayStation%202/Named_Boxarts/Silent%20Hill%202%20(USA)%20(En,Ja,Fr,De,Es,It)%20(v2.01).png" \
  -o ~/.local/share/icons/hicolor/512x512/apps/silent-hill-2.png
```

Где появляется ярлык:

- Omarchy menu → Apps (сетка) или `Super+Space` → набрать «Silent»
- любой лаунчер, читающий `.desktop`-файлы

Проверка:

```bash
desktop-file-validate ~/.local/share/applications/silent-hill-2.desktop
gio launch ~/.local/share/applications/silent-hill-2.desktop
```

Успех: открывается окно «Silent Hill 2» (class `pcsx2-qt`) и сразу грузится игра.

Если запись или иконка не видны в меню:

```bash
omarchy menu refresh    # или перелогиниться
```

Удалить ярлык:

```bash
rm ~/.local/share/applications/silent-hill-2.desktop
rm ~/.local/share/icons/hicolor/512x512/apps/silent-hill-2.png
omarchy menu refresh
```
