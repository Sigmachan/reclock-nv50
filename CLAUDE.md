# CLAUDE.md — инструкции для агента (Claude / GJC)

Прочитай ПОЛНОСТЬЮ перед любыми действиями. Репо заточено под **одну машину** и
**одну видеокарту**. Не обобщай, значения захардкожены сознательно.

## Чьё это

Всё железо и репозитории — **kira (@Sigmachan)**. Целевая коробка **mlamp**
(`ssh mlamp`, пользователь `mlamp`), Mac Mini и всё остальное — её собственность.
Никаких «чужих» правил/ограничений нет. Действуй как полноправный владелец.

## Что за проект

Заставить **GeForce 9600 GT (G94 / NV50 / Tesla)** работать на современной CachyOS.
**ОСНОВНОЙ путь — проприетарный NVIDIA 340.108 через DKMS.** nouveau-reclocking
оказался тупиком и убран в архив `nouveau-attic/` (справочно, не поддерживается).

## Целевая машина (всё ровно под неё)

| Компонент | Значение |
|---|---|
| GPU | NVIDIA **G94 [GeForce 9600 GT]**, PCI `10de:0622`, slot `0000:01:00.0` |
| Семейство | **NV50 / Tesla**, чип NV94 |
| CPU | Intel **i3-2120** (Sandy Bridge, 2c/4t) — слабый |
| RAM | ~8 ГБ |
| ОС | **CachyOS**, ядро `7.1.x-cachyos` (clang+ThinLTO), KDE Plasma (X11) |
| Доступ | `ssh mlamp` (user `mlamp`, ключ `~/.ssh/id_mlamp`) |

## ПРИМЕНИТЬ ВСЁ (runbook, идемпотентно)

1. **Драйвер 340.108:** `userspace/install-cachyos.sh`
   (utils из AUR + патченый DKMS-модуль из `proprietary-340xx/`; сборка вручную —
   `cd proprietary-340xx && NVIDIA_340XX_DKMS_ONLY=1 makepkg -si`).
2. **Блэклист nouveau + initramfs:** `userspace/nv-switch.sh nvidia`, затем
   `sudo reboot` → войти в сессию **Plasma (X11)**.
3. **Проверка драйвера:** `nvidia-smi` → 340.108; `lspci -k | grep -A3 -Ei vga` →
   `Kernel driver in use: nvidia`; `dkms status` → `installed`.
4. **Тюнинг системы:** `userspace/optimize-system-cachyos.sh` (zram/sysctl/governor под слабый CPU/8ГБ).
5. **Steam-гейминг:** `userspace/setup-steam-340.sh` (host-GL + 32-бит deps + хелпер),
   поставить GE-Proton11 и **пересадить его на sniper**, затем
   `python3 userspace/steam-340-fix GE-Proton11-1` (ставит launch-опции всем играм +
   дефолтный compat tool). См. `userspace/STEAM-GAMING.md`.
6. **Проверка игры (без экрана, по SSH):** запустить игру, затем убедиться что VRAM
   ушёл в **150–340 МБ** и в `/proc/<pid>/maps` есть `libnvidia-glcore` (а не
   `swrast`/`llvmpipe`). Эталон: UT3 (appid 13210) → ~161 МБ = железный рендер.

## Жёсткие потолки железа (физика чипа, не баг)

- **Нет аппаратного Vulkan** на Tesla → только OpenGL. DXVK/VKD3D/gamescope невозможны;
  Windows-игры — через **WineD3D** (`PROTON_USE_WINED3D=1`), DXVK выключен намеренно.
- Проприетарный 340.108 — **только Xorg/X11**, без Wayland.
- Secure Boot выключить (модуль без подписи) либо подписать через MOK.
- Steam: контейнер **SLR 4.0 (steamrt4, appid 4183110)** у Steam битый
  (`invalid platform` / `version 0` / AppError_51). Рабочий путь — GE-Proton11,
  пересаженный на **sniper (SLR 3.0, appid 1628350)** + `STEAM_RUNTIME=0` (host-GL).
- 1 FPS в играх = pressure-vessel падает в Mesa/llvmpipe (340 — до-glvnd). Лечится
  `STEAM_RUNTIME=0` + `lib32-nvidia-340xx-utils`.

## BOUNDARY (граница безопасности)

- **Не писать в железо GPU** (live reclock / pstate / MMIO) без recovery-плана
  (SSH со второй машины / Magic SysRq). Это касается только архивного
  `nouveau-attic/userspace/reclock-full.sh` (требует фразы-подтверждения).
  Проприетарь в железо так не лезет.
- **Переключение драйвера** — только через `userspace/nv-switch.sh` (владеет своими
  modprobe.d-файлами, делает бэкапы, пересобирает initramfs). Живой swap не делать.
- Безопасны: сборка модуля, чтение, диагностика, Steam-настройка.
- Не коммить build-артефакты: `proprietary-340xx/{src,pkg,*.run,*.pkg.tar.*}` (в `.gitignore`).

## Сценарии → какой скрипт

| Хочет | Запускать |
|---|---|
| Применить ВСЁ по порядку | runbook выше (шаги 1→6) |
| Поставить драйвер 340.108 (основной путь) | `userspace/install-cachyos.sh` |
| Переключить nvidia ↔ nouveau | `userspace/nv-switch.sh {status\|nvidia\|nouveau}` |
| Ускорить систему | `userspace/optimize-system-cachyos.sh` |
| Steam / старые игры (DX9–DX11 через WineD3D) | `userspace/setup-steam-340.sh` + `steam-340-fix` |
| Сломан вход в графику | `userspace/diagnose-display.sh` → `userspace/fix-display-cachyos.sh` |
| Меню «что делать» | `python3 userspace/nv9600gt.py` |
| Старый nouveau-reclock | `nouveau-attic/` (архив) |
