# Cemu + The Legend of Zelda: Breath of the Wild

Краткая памятка по настройке на этом ПК (Omarchy, Ryzen 5 5600X, Radeon RX 5600 XT).

## Что установлено

```bash
yay -S cemu-bin gamemode mangohud
```

- `cemu-bin 2.6` — нативный Linux-билд Cemu (без Wine).
- `gamemode`, `mangohud` — приоритеты CPU/GPU и оверлей (MangoHud в Cemu FPS показывает неверно, поэтому используется встроенный оверлей Cemu).

## Файлы игры

```
/home/sansheng/Downloads/The Legend of Zelda Breath of the Wild/
├── ... [Game]   [00050000101c9500]   # база, EUR
├── ... [Update] [0005000e101c9500]   # v1.6.0 (v208)
└── ... [DLC]    [0005000c101c9500]   # DLC v80
```

Дамп расшифрован, `keys.txt` не нужен. Путь добавлен в Cemu → Options → General settings → Game paths.
**Не перемещайте папку** без обновления пути в настройках Cemu.

## Ключевые исправления (почему раньше не работало)

1. **Зависание на «loading…» при запуске игры**
   Причина: Cemu в фоне проверял обновления на `cemu.info`, а сайт из этой сети не отвечает; главный поток ждал сетевой запрос.
   Решение: `~/.config/Cemu/settings.xml` → `<check_update>false</check_update>`.

2. **Второе зависание и полное отсутствие звука**
   Причина: в конфиге стоял `audio api = 0` (DirectSound — только Windows). Игра ждала аудио-бэкенд.
   Решение: `<Audio><api>3</api>` (Cubeb) и реальное устройство:
   `<TVDevice>alsa_output.usb-ASUSTeK_Xonar_SoundCard-00.analog-stereo</TVDevice>`
   (посмотреть свои: `pactl list short sinks`).

3. **Размытая картинка и ~20 FPS**
   Причина: пути графических паков в `settings.xml` были без `/rules.txt`, Cemu их не применял — игра шла в 720p и без FPS++, то есть 20–30 fps.
   Решение: правильные пути + пресет разрешения:

   ```xml
   <GraphicPack>
       <Entry filename="graphicPacks/downloadedGraphicPacks/BreathOfTheWild/Mods/FPS++/rules.txt"/>
       <Entry filename="graphicPacks/downloadedGraphicPacks/BreathOfTheWild/Graphics/rules.txt">
           <Preset>
               <category>Resolution</category>
               <preset>2560x1440 (2K)</preset>
           </Preset>
       </Entry>
   </GraphicPack>
   ```

   Плюс `<fullscreen>true</fullscreen>` — полноэкранный запуск 2560×1440, 60 fps.

4. **DualSense**
   Профиль: `~/.config/Cemu/controllerProfiles/controller0.xml`
   - API `SDLController`, гироскоп (`<motion>true</motion>`) и вибрация (`<rumble>0.6</rumble>`) включены.
   - UUID зависит от способа подключения: Bluetooth `0_0500…`, USB `0_0300…`. Если геймпад «не видится» — переподключите контроллер в Cemu: Options → Input settings → «−», затем «+» → SDLController → DualSense → Add.
   - Раскладка: ○ = A, × = B, □ = X, △ = Y, L1/R1 = L/R, триггеры = ZL/ZR.

## Запуск

- Меню приложений: **Cemu** (GUI со списком игр) или **Zelda: Breath of the Wild** (сразу игра).
- Терминал: `botw` (скрипт `~/.local/bin/botw`) — `gamemoderun cemu -t 00050000101c9500`.
- Обычный запуск: `gamemoderun cemu -t 00050000101c9500`.
- Полный экран в игре: `F11` или `Alt+Enter`.

## Особенности и мелочи

- **30 fps в меню и катсценах — это норма.** FPS++ ограничивает их, в игре — 60.
- Оверлей FPS: правый верхний угол (Debug → Open logging window не нужен).
- **Клик мышью/тачпадом по окну Cemu = касание экрана Wii U GamePad.** Если телевизор почернел с надписью «Коснитесь экрана Wii U GamePad…» — игра ушла в режим GamePad, просто кликните/коснитесь окна ещё раз.
- Тачпад DualSense работает как мышь (в Hyprland ничего не отключалось).
- Первые минуты в новых локациях — сообщения «Compiled N new shaders / pipelines» и микрозадержки: Vulkan собирает кэш, дальше плавно.
- Сохранения: `~/.local/share/Cemu/mlc01/usr/save/00050000/101C9500/`
- Лог Cemu: `~/.local/share/Cemu/log.txt` (создаётся при загрузке игры/краше).
- Настройки: `~/.config/Cemu/settings.xml`, профили геймпадов: `~/.config/Cemu/controllerProfiles/`.
- Графические паки: `~/.local/share/Cemu/graphicPacks/downloadedGraphicPacks` (версия Github991, поставлены вручную, т.к. `cemu.info` недоступен).

## Если что-то сломалось

- **Зависло на «loading…»** — смотрите конец `~/.local/share/Cemu/log.txt`; чаще всего это сеть (`cemu.info`) или аудио-устройство.
- **GPU-сброс / `VK_ERROR_DEVICE_LOST`** — один раз наблюдался на Mesa 26.2.5. Понизьте разрешение до `1920x1080 (Full HD)` в `settings.xml` (проверено: 60 fps) и перезапустите игру.
- **Низкий FPS** — проверьте, что оба пака реально включены (пути с `/rules.txt`) и включён FPS++.
- **Сменил разрешение/обновил паки** — приложение закрывайте через File → Exit, чтобы Cemu сохранил конфиг.
- Бэкап настроек: `cp ~/.config/Cemu/settings.xml ~/settings.xml.bak`.
