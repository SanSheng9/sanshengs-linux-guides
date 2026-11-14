# Ollama на AMD Radeon RX 5600 XT (Navi 10 / gfx1010) через Vulkan

Мини-руководство: как повторить настройку с нуля.

**Проверено на:**

- ОС: Omarchy 4.5 (Arch Linux)
- GPU: AMD Radeon RX 5600 XT, 6 ГБ VRAM (Navi 10, gfx1010)
- Драйвер: amdgpu + Mesa RADV
- Ollama: 0.40.7
- Результат: `qwen3:4b` → 100% GPU, ~75 ток/с генерация, ~150–200 ток/с промпт

## 0. Коротко: почему именно Vulkan

ROCm (родной GPGPU-стек AMD) официально **не поддерживает** Navi 10 (gfx1010). Этой карты нет в матрице поддерживаемых архитектур ROCm 7.x, поэтому пакет `ollama-rocm` с ней либо не заработает, либо потребует костыля:

```bash
HSA_OVERRIDE_GFX_VERSION=10.3.0
```

…и даже с ним возможны зависания и мусор в выводе. Плюс ~3 ГБ зависимостей.

Поэтому правильный путь для этой карты — Vulkan-бэкенд Ollama. Он использует Mesa RADV, который для Navi 10 зрелый и стабильный. В репозитории Arch есть готовый пакет `ollama-vulkan`.

## 1. Проверить, что карта и Vulkan видны системе

```bash
lspci -nn | grep -Ei 'vga|display|3d'
# -> должно быть что-то вроде:
#    ... [AMD/ATI] Navi 10 [Radeon RX 5600 OEM/5600 XT / 5700/5700 XT] [1002:731f]

lsmod | grep amdgpu
# -> модуль amdgpu должен быть загружен

vulkaninfo --summary | grep -A8 GPU0
# -> должно быть:
#    deviceName = AMD Radeon RX 5600 XT (RADV NAVI10)
#    driverName = radv
```

Если `vulkaninfo` нет:

```bash
sudo pacman -S vulkan-tools
```

Если устройство не появилось — проверьте пакеты:

```bash
sudo pacman -S mesa vulkan-radeon
```

## 2. Поставить Ollama + Vulkan-бэкенд

```bash
sudo pacman -S ollama ollama-vulkan
```

> **Важно:** пакеты `ollama` и `ollama-vulkan` должны быть **одной версии** — они обновляются вместе. Проверить: `pacman -Q ollama ollama-vulkan`

Пакет `ollama-vulkan` кладёт файл `/usr/lib/ollama/vulkan/libggml-vulkan.so`.

Если у вас стоит `ollama-cuda` (нужен только для NVIDIA), его можно убрать — это освободит ~1 ГБ:

```bash
sudo pacman -Rns ollama-cuda
```

## 3. Включить Vulkan в сервисе

Создать drop-in (переопределение) для systemd-юнита:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d

sudo tee /etc/systemd/system/ollama.service.d/vulkan.conf > /dev/null <<'EOF'
[Service]
Environment="OLLAMA_VULKAN=1"
EOF
```

Применить и запустить:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ollama
```

## 4. Проверить, что GPU реально используется

**а) В логах сервиса** должна появиться строка `inference compute` с `library=Vulkan`:

```bash
journalctl -u ollama -n 60 --no-pager | grep -i 'inference compute'
```

Ожидаемый вывод:

```
msg="inference compute" ... library=Vulkan ...
description="AMD Radeon RX 5600 XT (RADV NAVI10)"
type=discrete total="6.0 GiB" available="5.0 GiB"
```

**б) Запустить модель и посмотреть загрузку:**

```bash
ollama run qwen3:4b --verbose "привет"
ollama ps
```

В колонке `PROCESSOR` должно быть `100% GPU`. Если там `100% CPU` — Vulkan не подхватился, см. [раздел 7](#7-если-не-работает--что-проверять).

## 5. Какие модели ставить на 6 ГБ VRAM

Правило: модель в Q4 должна занимать не больше ~5 ГБ вместе с KV-кэшем.

**Комфортно (100% GPU):**

```bash
ollama run qwen3:4b          # ~2.5 ГБ, ~75 ток/с
ollama run gemma3:4b         # ~3 ГБ
ollama run llama3.2:3b       # ~2 ГБ
ollama run qwen2.5-coder:3b  # для кода, ~2 ГБ
```

**Впритык (может частично уйти на CPU):**

```bash
ollama run qwen2.5-coder:7b  # ~4.5 ГБ, маленький контекст
```

**Не стоит:** 14B и больше — не влезут в 6 ГБ, будут тормозить через swap.

Пользоваться в командной строке:

```bash
ollama list                               # что скачано
ollama run <модель>                       # интерактивный чат
ollama run <модель> --verbose "вопрос"    # с замерами скорости
ollama ps                                 # что загружено и где считается
ollama rm <модель>                        # удалить модель
```

## 6. (Опционально) Тонкая настройка под 6 ГБ

Добавить в тот же drop-in, затем `sudo systemctl restart ollama`:

```ini
# Держать модель в памяти дольше (по умолчанию 5 минут)
Environment="OLLAMA_KEEP_ALIVE=30m"

# Экономить VRAM на KV-кэше (немного влияет на качество)
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"

# Ограничить контекст, чтобы не вылезти за VRAM
Environment="OLLAMA_CONTEXT_LENGTH=4096"
```

Полный пример файла `/etc/systemd/system/ollama.service.d/vulkan.conf`:

```ini
[Service]
Environment="OLLAMA_VULKAN=1"
Environment="OLLAMA_KEEP_ALIVE=30m"
Environment="OLLAMA_CONTEXT_LENGTH=4096"
```

## 7. Если не работает — что проверять

1. **Пакета нет / версии разошлись**

   ```bash
   pacman -Q ollama ollama-vulkan
   sudo pacman -Syu ollama ollama-vulkan
   ```

2. **Сервис не видит Vulkan**

   ```bash
   systemctl status ollama
   journalctl -u ollama -n 100 --no-pager | grep -iE 'vulkan|error|fail'
   ```

   Проверить, что файл на месте:

   ```bash
   ls -l /usr/lib/ollama/vulkan/libggml-vulkan.so
   ```

3. **Vulkan вообще не видит карту**

   ```bash
   vulkaninfo --summary
   ```

   Если GPU0 нет — проблема в драйвере (mesa / vulkan-radeon), а не в Ollama.

4. **Ошибка про CUDA в логах**

   ```
   "ggml_cuda_init: failed to initialize CUDA: no CUDA-capable device"
   ```

   Это безвредно, если в `inference compute` есть `library=Vulkan`. Убирается удалением `ollama-cuda`.

5. **Модель грузится, но часть на CPU** (PROCESSOR: 40%/60% GPU)

   Модель слишком большая для 6 ГБ. Взять модель меньше или уменьшить `OLLAMA_CONTEXT_LENGTH`.

6. **Запасной вариант** — ROCm с подменой архитектуры (**не рекомендуется**, нестабильно, ~3 ГБ зависимостей):

   ```bash
   sudo pacman -S ollama-rocm
   ```

   В drop-in добавить:

   ```ini
   Environment="HSA_OVERRIDE_GFX_VERSION=10.3.0"
   ```

## 8. Как откатить всё назад

```bash
sudo systemctl disable --now ollama
sudo rm /etc/systemd/system/ollama.service.d/vulkan.conf
sudo systemctl daemon-reload
sudo pacman -Rns ollama-vulkan
```

Сами модели лежат в `/var/lib/ollama` — при желании удалить:

```bash
sudo rm -rf /var/lib/ollama/*
```

## 9. Итоговый чек-лист (для быстрого повтора)

- [ ] `vulkaninfo --summary` → виден `AMD Radeon RX 5600 XT (RADV NAVI10)`
- [ ] `sudo pacman -S ollama ollama-vulkan`
- [ ] drop-in с `OLLAMA_VULKAN=1`
- [ ] `sudo systemctl daemon-reload && sudo systemctl enable --now ollama`
- [ ] `journalctl -u ollama | grep "inference compute"` → `library=Vulkan`
- [ ] `ollama run qwen3:4b --verbose "привет"`
- [ ] `ollama ps` → `100% GPU`

---

*Дата составления: 2026-11-07. Проверено на реальной машине: Ryzen 5 5600X + RX 5600 XT, Omarchy/Arch.*
