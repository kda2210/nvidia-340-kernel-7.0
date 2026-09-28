# NVIDIA 340.108 на Kubuntu 26.04 LTS (ядро 7.0+)

**Статус: базовая установка работает (драйвер, X11, GLX, яркость обоих экранов). Есть два серьёзных открытых вопроса — сон и возможный перегрев, см. раздел «Открытые проблемы» в конце. Не считать проект готовым, пока они не закрыты.**

<img width="1600" height="896" alt="Снимок экрана_20260928_065932" src="https://github.com/user-attachments/assets/b49e039a-c35e-4d42-96ff-2b34878d8e76" />

Этот репозиторий не собирает свой собственный драйвер — вся тяжёлая работа (патчи под ядро 7.0+, сборка пакетов для 26.04) уже сделана [**piernov**](https://github.com/piernov). Здесь — инструкция по установке и все найденные по пути нюансы, которых нет ни у piernov, ни где-либо ещё.

Проверено на: Acer Aspire 7738, GeForce GT 240M, ядро `7.0.0-34-generic`.

Если тебе нужен 24.04 — используй [nvidia-340-ubuntu-24.04](https://github.com/kda2210/nvidia-340-ubuntu-24.04), этот проект его не заменяет.

## Что понадобится

- Kubuntu 26.04 LTS (или 26.04.1) — 340.108 не поддерживает Wayland вообще, а Kubuntu 26.04 по умолчанию ставит именно Wayland-сессию.
- Карта NVIDIA поколения Kepler и старше (последняя линейка, которую поддерживает ветка 340xx).
- Интернет на время установки.

### Не хочешь возиться с Wayland-по-умолчанию вообще?

Есть официальные флейворы Ubuntu 26.04, которые **до сих пор используют X11 по умолчанию**, без танцев с доп. пакетами:

- **[Xubuntu](https://xubuntu.org/)** (Xfce) — X11 как основной режим.
- **[Lubuntu](https://lubuntu.me/)** (LXQt) — в цикле 26.04 Wayland-сессия (labwc) пока не включена по умолчанию.
- **[Ubuntu MATE](https://ubuntu-mate.org/)** — MATE 1.26 в 26.04 Wayland вообще не умеет.

Все три собираются из того же архива Ubuntu 26.04, пакеты piernov подключаются к ним так же, как описано ниже.

## Установка

### 1. Ключ и репозитории piernov

```bash
sudo mkdir -p /etc/apt/keyrings
wget -O- https://github.com/piernov/nvidia-graphics-drivers-340xx-resolute/releases/latest/download/piernov-keyring.gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/piernov-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/piernov-keyring.gpg] https://github.com/piernov/nvidia-graphics-drivers-340xx-resolute/releases/latest/download/ ./" \
  | sudo tee /etc/apt/sources.list.d/piernov-nvidia.list

echo "deb [signed-by=/etc/apt/keyrings/piernov-keyring.gpg] https://github.com/piernov/glx-alternatives-resolute/releases/latest/download/ ./" \
  | sudo tee /etc/apt/sources.list.d/piernov-glx-alt.list

sudo apt update
```

### 2. ⚠️ Собрать `nvidia-support` — этого пакета нет в архиве Ubuntu 26.04

`nvidia-legacy-340xx-driver` предзависит от `nvidia-installer-cleanup` и зависит от `nvidia-support` — этих пакетов **нет ни в одном компоненте** архива Ubuntu 26.04 (main/restricted/universe/multiverse — проверено напрямую по индексам). Без этого шага `apt install` откажет с ошибкой неразрешимых зависимостей, причём в характерной, вводящей в заблуждение форме: apt обвинит в проблеме сразу ВСЕ пакеты в команде (build-essential, cmake и т.д.), хотя реально виноват только этот один пакет — просто apt так устроен, что валит всю транзакцию при одной неразрешимой зависимости.

Пакет доступен в Debian (компонент contrib), собираем вручную:

```bash
sudo apt install -y devscripts build-essential dpkg-dev debhelper \
  po-debconf pkgconf systemd-dev fakeroot

mkdir -p ~/nvidia-support-build && cd ~/nvidia-support-build
wget https://deb.debian.org/debian/pool/contrib/n/nvidia-support/nvidia-support_20240109+1.dsc
wget https://deb.debian.org/debian/pool/contrib/n/nvidia-support/nvidia-support_20240109+1.tar.xz
dpkg-source -x nvidia-support_20240109+1.dsc
cd nvidia-support-20240109+1
dpkg-buildpackage -us -uc -b
cd ..

sudo dpkg -i nvidia-support_20240109+1_amd64.deb \
             nvidia-installer-cleanup_20240109+1_amd64.deb \
             nvidia-kernel-common_20240109+1_amd64.deb
```

### 3. Ставим сам драйвер

```bash
sudo apt update
sudo apt install -y nvidia-legacy-340xx-driver glx-alternative-nvidia dkms
```

`glx-alternative-nvidia` (из репозитория `glx-alternatives-resolute`) обязателен — без него современный GLVND-стек 26.04 не найдёт библиотеки 340-го драйвера.

### 4. ⚠️ Доставить X11-сессию — на Kubuntu 26.04 её нет из коробки

Начиная с 25.10, Kubuntu ставит по умолчанию только Wayland-сессию Plasma. Без этого шага в SDDM просто не будет опции «Plasma (X11)».

```bash
sudo apt install -y plasma-session-x11 xorg
```

### 5. ⚠️ Ручной `xorg.conf` — автоопределение Xorg не подхватывает legacy-драйвер

Это критично и легко упустить: даже когда драйвер и модуль ядра установлены и загружены (`nvidia-smi` показывает карту), **Xorg без `xorg.conf` тихо откатывается на встроенный `modesetting`**, и `glxinfo` будет писать `GLX missing on display`, хотя сам модуль ядра при этом полностью рабочий. Официальной утилиты `nvidia-xconfig` в сборке piernov нет — пишем вручную:

```bash
sudo tee /etc/X11/xorg.conf > /dev/null << 'EOF'
Section "ServerLayout"
    Identifier     "Layout0"
    Screen      0  "Screen0"
EndSection

Section "Device"
    Identifier     "Device0"
    Driver         "nvidia"
    VendorName     "NVIDIA Corporation"
    Option         "NoLogo" "1"
    Option         "Coolbits" "4"
    Option         "RegistryDwords" "EnableBrightnessControl=1"
EndSection

Section "Screen"
    Identifier     "Screen0"
    Device         "Device0"
    DefaultDepth    24
    SubSection     "Display"
        Depth       24
    EndSubSection
EndSection
EOF
sudo systemctl restart sddm
```

Секций `Monitor`/`InputDevice` намеренно нет — современный Xorg сам справляется через udev/EDID, задача конфига — только заставить Xorg взять `nvidia` вместо `modesetting`. `BusID` тоже не нужен, если в системе только одна видеокарта (актуально для большинства ноутбуков этой линейки).

Что делают опции:
- `NoLogo` — убирает заставку NVIDIA при старте X, косметика.
- `Coolbits "4"` — открывает вкладку ручного управления вентилятором в `nvidia-settings`.
- `RegistryDwords "EnableBrightnessControl=1"` — см. следующий раздел, без него яркость ноутбучной панели не регулируется вообще.

### 6. Перезагрузка

На экране входа нажми на значок шестерёнки рядом с именем пользователя и выбери **«Plasma (X11)»**. Проверка после входа:

```bash
nvidia-smi
echo $XDG_SESSION_TYPE   # должно показать "x11"
glxinfo | grep vendor    # должно показать "NVIDIA Corporation" во всех трёх строках
```

## Регулировка яркости

### Встроенный экран ноутбука

Это самая запутанная часть, пишу по шагам — здесь легко потратить время впустую на ложные пути.

**Почему `acpi_backlight=` сам по себе не помогает.** Современный ACPI backlight в ядре регистрируется не автоматически, а инициируется DRM/KMS-хелпером во время probe'а выходов — тем самым кодом, что используют i915, nouveau, amdgpu. 340.108 — это архитектура **UMS** (появилась до DRM/KMS в современном виде) и в DRM-подсистему не регистрируется вообще. Раньше яркость работала именно потому, что этим занимался `nouveau` — как только он заблокирован в пользу проприетарного драйвера, регистрировать backlight в sysfs становится некому, и `ls /sys/class/backlight/` пуст независимо от
значения `acpi_backlight=`.

**Обязательно проверь синтаксис своего `/etc/default/grub` перед правкой.**
На части систем там двойные кавычки (`GRUB_CMDLINE_LINUX_DEFAULT="..."`), на части — одинарные (`GRUB_CMDLINE_LINUX_DEFAULT='...'`). Команда `sed` должна совпадать с тем, что реально в файле — иначе она молча ничего не поменяет, без единой ошибки, и часы отладки уйдут впустую именно на это. Сначала посмотри:

```bash
cat /etc/default/grub | grep GRUB_CMDLINE_LINUX_DEFAULT
```

Дальше — под двойные кавычки:
```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="acpi_backlight=video /' /etc/default/grub
```
или под одинарные:
```bash
sudo sed -i "s/GRUB_CMDLINE_LINUX_DEFAULT='/GRUB_CMDLINE_LINUX_DEFAULT='acpi_backlight=video /" /etc/default/grub
```

Затем в любом случае:
```bash
sudo update-grub
sudo reboot
```

И после перезагрузки — обязательно проверь, что параметр реально долетел до ядра (это единственный надёжный способ убедиться, что правка сработала):
```bash
cat /proc/cmdline   # ищи acpi_backlight=video в строке
ls /sys/class/backlight/   # должен появиться acpi_video0
```

Если появился `acpi_video0` — дальше ничего писать не нужно: `powerdevil` (ползунок в трее Plasma) сам подхватывает любое устройство в `/sys/class/backlight/` автоматически.

**Если `acpi_video0` так и не появился** — значит на конкретной модели ACPI video bus не регистрирует backlight сам, и остаётся вариант с прямым вызовом ACPI-методов прошивки через `acpi_call` (не костыль — это тот же самый официальный интерфейс, который иначе вызвал бы DRM-хелпер, только без автоматической обвязки). Путь к методам смотрится через дизассемблирование DSDT (`acpica-tools`, `iasl -d`) — ищи `_BCL`/`_BCM`/`_BQC` и их полный Device-путь, значения яркости — в пакете `\BCLP`.

### Внешний монитор (VGA/DVI-A)

Работает штатно через **DDC/CI** — Plasma (`powerdevil`) умеет управлять внешними мониторами через `libddcutil`, если пакет установлен:

```bash
sudo modprobe i2c-dev
sudo apt install -y ddcutil
sudo ddcutil detect   # должен найти монитор с его EDID
```

Если найден — слайдер в трее должен появиться сам после перелогина. Учти известную недоработку `powerdevil`: при одновременно активных встроенной (не-DDC) и внешней (DDC) панелях разработчики сами признают, что реализация поддерживает по сути один дисплей полноценно — если слайдер путается, управляй напрямую: `sudo ddcutil setvcp 10 <0-100>`.

Не все мониторы поддерживают DDC/CI (может быть отключено в OSD-меню) — если `ddcutil detect` монитор не находит, дальше двигаться некуда, это ограничение самого монитора, а не системы.

## nvidia-settings — тоже нужно собирать вручную

Как и `nvidia-support`, пакет `nvidia-settings-legacy-340xx` — отдельный Debian source-пакет (contrib), которого нет ни в сборке piernov, ни в архиве Ubuntu 26.04:

```bash
sudo apt install -y m4 libgtk2.0-dev libjansson-dev libvdpau-dev \
  libxext-dev libxv-dev libxxf86vm-dev xserver-xorg-dev

mkdir -p ~/nvidia-settings-build && cd ~/nvidia-settings-build
wget https://deb.debian.org/debian/pool/contrib/n/nvidia-settings-legacy-340xx/nvidia-settings-legacy-340xx_340.108-7.dsc
wget https://deb.debian.org/debian/pool/contrib/n/nvidia-settings-legacy-340xx/nvidia-settings-legacy-340xx_340.108.orig.tar.bz2
wget https://deb.debian.org/debian/pool/contrib/n/nvidia-settings-legacy-340xx/nvidia-settings-legacy-340xx_340.108-7.debian.tar.xz
dpkg-source -x nvidia-settings-legacy-340xx_340.108-7.dsc
cd nvidia-settings-legacy-340xx-340.108
dpkg-buildpackage -us -uc -b
cd ..
sudo dpkg -i nvidia-settings-legacy-340xx_340.108-7_amd64.deb
```

Полезная деталь: у легаси-драйвера `nvidia-smi` **не отдаёт** `utilization`/большинство полей через обычный запрос (только температуру и память) — для мониторинга GPU в скриптах используй `nvidia-settings -t -q [gpu:0]/GPUCoreTemp -q [gpu:0]/GPUUtilization`, а для perf-уровней — `/proc/driver/nvidia/gpus/*/performance` (не зависит от X, работает даже без запущенной сессии).

## Известные проблемы — НЕ решены

### 🔴 Suspend/resume вешает всю систему

Уход в сон (клавишами или `systemctl suspend`) не просыпается — экран гаснет, а по факту происходит полное зависание, требующее жёсткой перезагрузки. `PowerDevil` сам пишет в лог `SuspendSession action not available!`. Вероятная причина — 340.108 (сборка датирована 2019 годом) не содержит инфраструктуры сохранения/восстановления состояния GPU (`nvidia-sleep.sh` и systemd-хуки появились в куда более поздних версиях драйвера), а на кристалле такого возраста PCI power-management callback'и не рассчитаны на то, как это устроено в ядре 7.x. Возможно, что это в принципе не чинится для этой связки железо+драйвер+ядро без честного предупреждения в README «suspend не поддерживается».

### 🔴 Периодические самопроизвольные выключения, похожие на перегрев

Наблюдались выключения при **разных** показанных в conky температурах (от ~60°C до 70-80°C) — сам разброс говорит о том, что это, скорее всего, не срабатывание штатного critical-порога (он у CPU/ACPI-зон стоит на 100°C, проверено напрямую). Что установлено точно на данный момент:

- GPU в ходе двухчасового мониторинга **ни разу не вышел из `P0`** (максимальный режим питания) даже при полном простое рабочего стола — и за это же время температуры (`acpitz`, ядра CPU) медленно, но непрерывно росли без видимой нагрузки CPU. Вероятный вклад — PowerMizer легаси-драйвера может не понижать состояние при активных двух дисплеях одновременно (известное поведение NVIDIA на этой линейке карт).
- Тяжёлая, но короткая нагрузка (обновление системы) довела ноутбук до сильного нагрева, но **не привела к выключению** — что скорее указывает на длительность/кумулятивный эффект нагрева, а не на мгновенный пиковый перегрев от какой-то конкретной операции.

Дальнейшая диагностика — через `sensors-detect --auto` (ещё не завершена) и через захват `journalctl -b -1` сразу после очередного самопроизвольного выключения.

### 🟡 Периодические падения `drkonqi` (сам crash-репортер KDE)

Падает, пытаясь отрисовать QML-интерфейс через OpenGL — вероятно, GT 240M не тянет требования рендерера Qt6 Quick. Обходной путь (не чинит первопричину, но снимает целый класс QML-related падений):
```bash
mkdir -p ~/.config/plasma-workspace/env
echo 'export QT_QUICK_BACKEND=software' | tee ~/.config/plasma-workspace/env/qt-quick-backend.sh
```

## О репозитории

Это личные заметки по установке, а не поддерживаемый проект — я не пишу и не собираю сам драйвер, всю техническую работу делает [piernov](https://github.com/piernov). Соответственно, багов драйвера, патчей под ядро или самого GLX-механизма я чинить не могу: по ним — сразу в [issues резолюции piernov](https://github.com/piernov/nvidia-graphics-drivers-340xx-resolute/issues) или [glx-alternatives-resolute](https://github.com/piernov/glx-alternatives-resolute/issues).
