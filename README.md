<a id="top"></a>

**🌐 Language / Язык:** [English](#english) · [Русский](#russian)

---

<a id="english"></a>

# NVIDIA 340.108 on Kubuntu 26.04 LTS (kernel 7.0)

[🇷🇺 Русская версия](#russian)

**What has been checked in practice and what hasn't:**

| | Status |
|---|---|
| Driver, X11 session, GLX | ✅ works |
| Brightness of the built-in and external screen | ✅ works |
| `nvidia-settings` | ✅ builds and works |
| Sleep (suspend) | ✅ works if Wi-Fi/Bluetooth are blocked during sleep (see the "Sleep" section) |
| Sudden power-offs | ⚠️ only on my laptop (as far as I know, other users don't have them), cause not found |

<img width="1600" height="896" alt="Screenshot_20260928_065932" src="https://github.com/user-attachments/assets/b49e039a-c35e-4d42-96ff-2b34878d8e76" />

This repository does not build its own driver: all the heavy lifting (patches
for kernel 7.x, building packages for 26.04) was done by
[**piernov**](https://github.com/piernov). What you'll find here is an
installation guide and the nuances I ran into along the way.

Tested on: Acer Aspire 7738 (2009 BIOS, **legacy BIOS, not EFI**), GeForce
GT 240M (GT216), **no battery in the laptop**, kernels `7.0.0-30-generic` and
`7.0.0-34-generic`.

If you need 24.04, use
[nvidia-340-ubuntu-24.04](https://github.com/kda2210/nvidia-340-ubuntu-24.04);
this project does not replace it.

## Steps at a glance

1. Connect piernov's repositories and build `nvidia-support` (steps 1-2).
2. Install the driver and the X11 session and write `xorg.conf` (steps 3-5), reboot and choose "Plasma (X11)" (step 6).
3. Brightness: kernel parameter `acpi_backlight=video`, and `ddcutil` for the external monitor.
4. Optionally, build `nvidia-settings`.
5. Sleep: disable user-session freezing in systemd and block Wi-Fi/Bluetooth during sleep.
6. If your internet is slow, prepare the package set in advance (section "If your internet is slow").

## What you need

- Kubuntu 26.04 LTS (or 26.04.1). Driver 340.108 does not support Wayland,
  and Kubuntu 26.04 installs the Wayland session by default.
- A card of the **pre-Fermi** generation: GeForce 8, 9, 100, 200, 300 series
  (chips G8x/G9x/GT2xx). Fermi (400/500) needs the 390 branch, Kepler needs 470.
- Internet access during installation.

### If you don't want to deal with Wayland by default

According to reviews, some official Ubuntu 26.04 flavours stay on X11 by
default: **[Xubuntu](https://xubuntu.org/)** (Xfce; Wayland support is experimental) and **[Lubuntu](https://lubuntu.me/)**
(LXQt; X11 is the default session, Wayland is experimental). [Ubuntu MATE](https://ubuntu-mate.org/) has
no official 26.04 image (the flavour skipped this release), but MATE can be
added to a regular Ubuntu install with the `ubuntu-mate-desktop` package. I
have **not tested** any of them with this driver, but packages from the same
Ubuntu 26.04 archive should connect the same way as described below.

## Installation

### 1. Key and piernov's repositories

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

### 2. ⚠️ Build `nvidia-support`: this package is not in the Ubuntu archive

`nvidia-legacy-340xx-driver` pre-depends on `nvidia-installer-cleanup` and
depends on `nvidia-support`. These packages are absent from the package
indexes of Ubuntu 24.04, 25.10 and 26.04 (main/restricted/universe/multiverse,
amd64). Without them `apt install` fails with unresolvable dependencies, and
apt blames all the packages in the command at once (`build-essential`,
`cmake`, etc.) although only one is at fault: apt rejects the whole
transaction.

The sources are in Debian (contrib component); we build them manually:

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

The build was verified: the packages are produced. Installing them with
`dpkg -i` exactly like this was not tested separately (in my case they went
through a local repository).

### 3. Install the driver itself

```bash
sudo apt update
sudo apt install -y nvidia-legacy-340xx-driver glx-alternative-nvidia dkms \
  build-essential linux-headers-generic
```

`dkms`, a compiler and kernel headers are needed to build the module; they
are usually pulled in automatically and are listed explicitly just in case.
According to piernov's description, `glx-alternative-nvidia` (from the
`glx-alternatives-resolute` repository) replaces `libGLX.so.0` so that the
GLVND stack of 26.04 finds the driver's libraries. I haven't tried without it.

### 4. ⚠️ Install the X11 session: Kubuntu 26.04 doesn't have it out of the box

According to reviews, since Kubuntu 25.10 Plasma is installed without an X11
session by default. Without this step there will be no "Plasma (X11)" entry
in SDDM.

```bash
sudo apt install -y plasma-session-x11 xorg
```

**Keep Plasma on 6.6.** According to OMG! Ubuntu (October 2026), Plasma 6.8
no longer has the `plasma-session-x11` package at all, and Kubuntu 26.10
(Plasma 6.7) is the last release where the X11 session can still be installed.
So on 26.04 don't move to a newer Kubuntu release, and if you add the Kubuntu
Backports PPA, make sure Plasma doesn't get upgraded to 6.8 or later: that
version has no X11 session, which this driver needs.

### 5. ⚠️ Manual `xorg.conf`

Without `xorg.conf`, `glxinfo` gave me `GLX missing on display`, even though
`nvidia-smi` saw the card. After creating the file GLX worked: `glxinfo`
shows NVIDIA and `glxgears` renders. Apparently Xorg was picking the
built-in driver instead of `nvidia` (I didn't look at the Xorg log before
that). The build from piernov has no `nvidia-xconfig` utility, so we write the
file by hand:

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

There are no `Monitor`/`InputDevice` sections on purpose: Xorg sorts them out
itself via udev and EDID. `BusID` isn't needed in my case (a single video card).

The options I used to have in the config (`NoLogo`, `Coolbits "4"`,
`RegistryDwords "EnableBrightnessControl=1"`) are not needed for it to work,
so they are not here. `NoLogo` hides the NVIDIA splash screen, which is
cosmetic. According to the documentation, `Coolbits` enables manual fan
control, but I haven't tested it on this laptop (the fan is controlled by the
board's controller). I added `EnableBrightnessControl` while looking for a way
to fix brightness, but brightness started working thanks to the kernel
parameter `acpi_backlight=video`; that the option is needed is not proven.

The Xorg log will contain a warning that the X server ABI (25.2) is not
officially supported by the driver and that "the driver will continue to
load". This is expected.

### 6. Reboot

On the login screen choose the **"Plasma (X11)"** session in the session
selector. Check after logging in:

```bash
nvidia-smi
echo $XDG_SESSION_TYPE   # should be "x11"
glxinfo | grep vendor    # "NVIDIA Corporation" in all three lines
```

## If your internet is slow

My laptop's internet is about 20 KB/s, so I prepared the whole package set
(driver, `nvidia-support`, DKMS, compiler, kernel headers,
`plasma-session-x11`, `xorg`) in advance on another machine, in a live session
of the same Kubuntu version, and then installed it from a local apt
repository. There are no ready-made scripts here, but this is what worked or
broke along the way:

- A live session stores nothing between reboots. Prepare the set in one
  session and copy it to a persistent disk before rebooting.
- The kernel version in the live session must match the kernel of the
  installed system: the module is built by DKMS from the headers. Until the
  set is used, it's better not to update the kernel on the target system
  (`sudo apt-mark hold linux-generic linux-image-generic linux-headers-generic`).
- `nvidia-support` must be built and put into the local repository **before**
  downloading the driver, otherwise `apt-get --download-only` fails on
  unresolvable dependencies.
- Keep the repository in `/srv`, not in your home directory. While it was in
  `/home`, `apt update` didn't see it (apt reads `file://` as the `_apt` user,
  and the home directory is usually closed to it); after moving it to `/srv`
  everything worked.
- The index (needs the `dpkg-dev` package):
  `cd /srv/nvidia-offline-repo && dpkg-scanpackages . /dev/null | gzip -9c | sudo tee Packages.gz > /dev/null`.
  The source: `deb [trusted=yes] file:///srv/nvidia-offline-repo ./`. The
  `trusted=yes` parameter disables signature checking: you are responsible for
  the contents of the folder yourself.
- Don't forget `plasma-session-x11` and `xorg`: without them there will be no
  "Plasma (X11)" entry in SDDM on the laptop, and there is no way to fix that
  without a network.

## Brightness control

### Built-in laptop screen

**What worked:** the kernel parameter `acpi_backlight=video`. After it, the
device `/sys/class/backlight/acpi_video0` appeared and the brightness slider
in the Plasma tray started working by itself (PowerDevil picks up any device
from `/sys/class/backlight/`). This happens with the proprietary driver loaded
too.

**The main trap, which cost me the most time.** In `/etc/default/grub` on this
laptop the line was `GRUB_CMDLINE_LINUX_DEFAULT='quiet splash'`, with
**single** quotes. A `sed` command written for double quotes silently changed
nothing and reported no errors, so the parameter never reached the kernel and
we thought it "didn't work". That's why only `video` has really been tested.
The values `vendor` and `native` were effectively never applied, so it can't
be said that they don't work.

A command that works with both kinds of quotes:

```bash
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub
sudo sed -i -E "s/^(GRUB_CMDLINE_LINUX_DEFAULT=['\"])/\1acpi_backlight=video /" /etc/default/grub
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub   # make sure the parameter appeared
sudo update-grub
sudo reboot
```

After the reboot, be sure to check that the parameter reached the kernel:

```bash
cat /proc/cmdline             # must contain acpi_backlight=video
ls /sys/class/backlight/      # acpi_video0 must appear
```

**Fallback if `acpi_video0` did not appear:** calling the firmware's ACPI
methods directly via `acpi_call` (`acpi-call-dkms`). On this laptop it worked
(brightness changed with commands written to `/proc/acpi/call`), but in the
end it wasn't needed. Method names differ for every model; they are found in
the DSDT table:

```bash
sudo apt install -y acpica-tools
sudo cat /sys/firmware/acpi/tables/DSDT > dsdt.dat
iasl -d dsdt.dat
grep -n "_BCL\|_BCM\|_BQC" dsdt.dsl
```

On the Aspire 7738 the method was at `\_SB.PCI0.PEGP.VGA.LCD._BCM`, and the
allowed values (10, 20, …, 100) were in the package `\BCLP`.

### External monitor (VGA)

It started working after installing `ddcutil`: PowerDevil controls external
monitors over DDC/CI, and separate sliders appeared for both screens.

```bash
sudo modprobe i2c-dev
sudo apt install -y ddcutil
sudo ddcutil detect   # should find the monitor
```

`ddcutil` marks the built-in panel as a "laptop display" that doesn't support
DDC/CI. This is normal: it is controlled through `acpi_video0`. Not all
monitors support DDC/CI (it may be turned off in the monitor's menu). A known
PowerDevil shortcoming: its developers admit that when the built-in and an
external panel are used at the same time, the implementation is essentially
designed for a single display. If the slider gets confused, control the
monitor directly: `sudo ddcutil setvcp 10 <0-100>`.

## nvidia-settings: also has to be built by hand

`nvidia-settings-legacy-340xx` is a separate Debian source package (contrib);
it is neither in piernov's build nor in the Ubuntu 26.04 archive. The build
was verified and the program starts:

```bash
sudo apt install -y m4 libgtk2.0-dev libjansson-dev libvdpau-dev \
  libgl-dev libxext-dev libxv-dev libxxf86vm-dev xserver-xorg-dev

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

**GPU monitoring.** With this driver `nvidia-smi` doesn't answer every query.
What works: `nvidia-smi --query-gpu=temperature.gpu,pstate --format=csv`
(temperature and P-state). Power, clock and utilization queries return
`[Not Supported]`. For utilization use
`nvidia-settings -t -q [gpu:0]/GPUUtilization`. The file
`/proc/driver/nvidia/gpus/*/performance` does **not** exist in this build.

## Sleep (suspend): works

Without any fixes, going to sleep hung: the screen went dark, the fan and
LEDs kept running, and only `Magic SysRq` responded.

### What was found and helped

Checked step by step through `/sys/power/pm_test` (see the diagnostics below).

**1. systemd 256+ freezes the user session before sleep.** Discussed in
[systemd#33626](https://github.com/systemd/systemd/issues/33626) and
[systemd#33083](https://github.com/systemd/systemd/issues/33083); according to
reports, it affects more than just NVIDIA, and the discussions include a
suggestion that the root of the problem is in the kernel. On this machine I
never saw the message "Failed to freeze unit 'user.slice'" (the journal had no
time to be written at the moment of the hang), so the link is established only
by the effect: after the fix, the `freezer` level test stopped hanging.

```bash
for s in suspend hibernate hybrid-sleep suspend-then-hibernate; do
  sudo mkdir -p /etc/systemd/system/systemd-$s.service.d
  printf '[Service]\nEnvironment="SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false"\n' \
    | sudo tee /etc/systemd/system/systemd-$s.service.d/no-freeze-user-sessions.conf >/dev/null
done
sudo systemctl daemon-reload
```

**2. Disabling Wi-Fi and Bluetooth during sleep.** This is what finally cured
sleep completely. The combined Intel Wi-Fi/Bluetooth module (`iwlwifi`,
`hci0`) interfered with going to sleep: the `devices` level test hung until I
blocked **both** radio modules (Bluetooth alone turned out to be not enough).
After the module was blocked, the sleep problem disappeared completely for me.
The log shows Bluetooth command timeouts (`0x0c24`, `0x0c52`) and a reset of
the `hci0` device; I haven't proven the mechanism of the hang. The "root hub
lost power or was reset" messages in the same log are probably the ordinary
messages of USB controllers coming out of sleep, not the cause.

A hook that does the same thing automatically: it blocks both radio modules
before sleep and removes the block after waking up:

```bash
sudo tee /usr/lib/systemd/system-sleep/wifi-bt-suspend-fix > /dev/null << 'EOF'
#!/bin/bash
case "$1" in
  pre)  rfkill block wifi; rfkill block bluetooth ;;
  post) rfkill unblock wifi; rfkill unblock bluetooth ;;
esac
EOF
sudo chmod +x /usr/lib/systemd/system-sleep/wifi-bt-suspend-fix
```

### What remains unclear

- Why exactly communication with the combined Wi-Fi/Bluetooth module breaks
  going to sleep: the mechanism is not proven (see above).
- Whether the `mem_sleep_default=deep` parameter is needed. I added it as an
  experiment; it didn't affect the hang on entering sleep (which happened
  before actual sleep). If `cat /sys/power/mem_sleep` already shows `[deep]`
  on your machine, the parameter is definitely not needed.

### How to diagnose a sleep hang

The usual `journalctl -b -1` after a hard reboot is almost always empty:
`SysRq+B` doesn't do a `sync`, and at the moment of the hang journald is
already frozen. A working combination: `echo 1 | sudo tee
/sys/power/pm_debug_messages` and `pm_test=<level>` (`freezer` → `devices` →
`platform` → `processors` → `core`). The test runs the transition step by step
and returns control by itself if the level passed. If it hangs, it is more
reliable to switch to a clean console (`sudo systemctl stop sddm`,
`Ctrl+Alt+F3`), turn on verbose output (`echo 8 | sudo tee
/proc/sys/kernel/printk`) and, after the test, save the relevant part of the
journal to a file (`journalctl -k --no-pager | sed -n 'N,Mp' > file.txt`). The
settings are reset after a reboot; to reset them manually: `echo none | sudo
tee /sys/power/pm_test`.

## Known issues

### ⚠️ Sudden power-offs: a quirk of my laptop, cause not found

**Symptom:** power disappears instantly, without a single message in the
journal or in `dmesg`. It happens at idle and quite often right after
plugging a USB flash drive into a cold laptop. The temperature on conky is
35-40 °C at that moment (sometimes higher), and the fan behaves as on 24.04.
On 24.04 (kernel 6.14, my earlier build of the driver) nothing like this was
observed on the same hardware. On 26.04 with the NVIDIA driver the power-offs
happened to me at least twice an hour.

**Conclusion for now:** as far as I know, other users of this driver on 26.04
don't have such power-offs, so I consider it a quirk of my particular laptop,
not a general problem of the driver or of this guide. What exactly is wrong
with it, I haven't found out. If you don't have this, you can skip this
section.

**Established:**
- In the last captured `dmesg -w`, the record broke off about 1.3 s after the
  flash drive was plugged in, on the first SCSI commands to it
  (`sd 6:0:0:0: [sdd] Mode Sense`), before udisks and Dolphin started
  (accuracy about 0.5 s).
- On a clean system without the NVIDIA driver (nouveau, kernel 7.0.0-30) the
  laptop ran for 3 h 18 min without a cut-off, and the journal ended with a
  normal session shutdown. The flash drive was plugged into the cold system
  161 s after boot, and there was no power-off.
- Overheating is unlikely (low temperatures, the fan works), and there is no
  battery.
- The kernel parameters `tsc=unstable trace_clock=local` didn't help: they
  concern hangs caused by clock problems, not a power cut-off.
- The "console without the NVIDIA driver" test (`module_blacklist=nvidia
  systemd.unit=multi-user.target`) gave no clean result: the laptop was found
  switched off, and how exactly is unknown.

**Not checked:** the same 26.04 on an older kernel (6.14); comparing kernels
6.19 and 7.x on another distribution; the power adapter and connector;
limiting C-states (`processor.max_cstate=1`); `usbcore.autosuspend=-1` (the
parameter was not applied in the captured boot); `thermal.nocrt=1` (unlikely:
there is no characteristic line about a critical temperature);
`GRUB_GFXPAYLOAD_LINUX=text` (the console runs on simpledrm/fbcon rather than
on text VGA).

**If you have the same power-offs, here is how to capture the moment of the
cut-off.** The journald journal didn't survive between boots for me, so don't
rely on it. Start this after every boot (background processes disappear on
reboot):

```bash
sudo sysctl -w kernel.dmesg_restrict=0
stdbuf -oL dmesg -w > ~/crash-kernel.log 2>&1 &
disown
stdbuf -oL journalctl -f > ~/crash-journal.log 2>&1 &
disown
( while true; do sleep 0.5; sync; done ) &
disown
```

`dmesg -w` writes about a second further than `journalctl`, and because of
the `sync` every 0.5 s the moment of the cut-off is determined with an
accuracy of about half a second. If the record breaks off without a single
error message, the system had no time to write anything.

### ✅ Avalanche of `kioworker`/`drkonqi` crashes: fixed (confirmed indirectly)

The journal of one of the power-offs contained 113 launches of
`drkonqi-coredump-launcher` within 24 seconds. The first to crash was
`kioworker` (thumbnail generation, `thumbnail.so`): judging by the call stack,
the Plasma theme (`KDEPlasmaPlatformTheme6`) creates a `QOpenGLContext` inside
Qt's headless `offscreen` platform, and that makes the process abort. After
that, the crash handlers spawned one another.

What was done:
```bash
mkdir -p ~/.config/environment.d
echo "QT_OPENGL=software" > ~/.config/environment.d/qt-opengl-software.conf   # after re-login
systemctl --user mask 'drkonqi-coredump-launcher@.service'                     # breaks the avalanche
```
After this, according to the author, the idle temperature stays below 60 °C,
and no crashes are visible in the log of plugging in a flash drive. No
separate check via `coredumpctl` was made after the fix. The power-offs didn't
stop, so this is not the only cause.

### 🟡 Other observations

- In a two-hour measurement the GPU never left `P0` (the maximum state), even
  at idle. A likely explanation is the known NVIDIA behaviour with two active
  monitors; it isn't proven on this machine. The refresh rates of the screens
  (60.08 and 60.02 Hz) don't match; the check with aligned rates was not
  completed.
- On kernel 7.0 the driver produces two kernel `WARNING`s (`Unpatched return
  thunk in use`, `drm_open_helper`) and the message `NVRM: Your system is not
  currently configured to drive a VGA console`. On BIOS systems with the
  graphical GRUB mode, the console runs on simpledrm/fbcon instead of text
  VGA. A link to the power-offs is **not proven**. The check with
  `GRUB_GFXPAYLOAD_LINUX=text` has not been done.
- `QT_QUICK_BACKEND=software` (put into `~/.config/plasma-workspace/env/`) was
  suggested against crashes of `drkonqi` itself; **the effect was not
  verified**.

## About this repository

These are personal installation notes, not a maintained project. I don't
write or build the driver myself; all the technical work is done by
[piernov](https://github.com/piernov). I can't fix driver bugs, kernel patches
or the GLX mechanism: for those, go straight to
[piernov's issues](https://github.com/piernov/nvidia-graphics-drivers-340xx-resolute/issues)
or [glx-alternatives-resolute](https://github.com/piernov/glx-alternatives-resolute/issues).

[⬆ Back to top / language switch](#top)

---

<a id="russian"></a>

# NVIDIA 340.108 на Kubuntu 26.04 LTS (ядро 7.0)

[🇬🇧 English version](#english)

**Что проверено на практике, а что нет:**

| | Статус |
|---|---|
| Драйвер, X11-сессия, GLX | ✅ работает |
| Яркость встроенного и внешнего экрана | ✅ работает |
| `nvidia-settings` | ✅ собирается и работает |
| Сон (suspend) | ✅ работает, если на время сна блокировать Wi-Fi/Bluetooth (см. раздел «Сон») |
| Внезапные выключения | ⚠️ только на моём ноутбуке (по моим сведениям, у других пользователей их нет), причина не найдена |

<img width="1600" height="896" alt="Снимок экрана_20260928_065932" src="https://github.com/user-attachments/assets/b49e039a-c35e-4d42-96ff-2b34878d8e76" />

Этот репозиторий не собирает свой собственный драйвер: вся тяжёлая работа
(патчи под ядро 7.x, сборка пакетов для 26.04) сделана
[**piernov**](https://github.com/piernov). Здесь инструкция по установке и
нюансы, обнаруженные по пути.

Проверено на: Acer Aspire 7738 (BIOS 2009 года, **legacy BIOS, не EFI**),
GeForce GT 240M (GT216), **батареи в ноутбуке нет**, ядра `7.0.0-30-generic`
и `7.0.0-34-generic`.

Если нужен 24.04, используй
[nvidia-340-ubuntu-24.04](https://github.com/kda2210/nvidia-340-ubuntu-24.04),
этот проект его не заменяет.

## Порядок действий в двух словах

1. Подключить репозитории piernov и собрать `nvidia-support` (шаги 1-2).
2. Поставить драйвер, X11-сессию и написать `xorg.conf` (шаги 3-5), перезагрузиться и выбрать «Plasma (X11)» (шаг 6).
3. Яркость: параметр ядра `acpi_backlight=video`, для внешнего монитора `ddcutil`.
4. По желанию собрать `nvidia-settings`.
5. Сон: отключить заморозку пользовательских сессий в systemd и блокировать Wi-Fi/Bluetooth на время сна.
6. Если интернет медленный, набор пакетов лучше подготовить заранее (раздел «Если интернет медленный»).

## Что понадобится

- Kubuntu 26.04 LTS (или 26.04.1). Драйвер 340.108 не поддерживает Wayland, а
  Kubuntu 26.04 по умолчанию ставит именно Wayland-сессию.
- Карта поколения **до Fermi**: GeForce 8, 9, 100, 200, 300 серий (чипы
  G8x/G9x/GT2xx). Для Fermi (400/500) нужна ветка 390, для Kepler — 470.
- Интернет на время установки.

### Если не хочется возиться с Wayland по умолчанию

По данным обзоров, часть официальных флейворов Ubuntu 26.04 по умолчанию
остаётся на X11: **[Xubuntu](https://xubuntu.org/)** (Xfce, поддержка Wayland экспериментальная) и
**[Lubuntu](https://lubuntu.me/)** (LXQt, сессия по умолчанию X11, Wayland экспериментальный). У
[Ubuntu MATE](https://ubuntu-mate.org/) официального образа 26.04 нет (флейвор пропустил этот релиз), но
MATE можно добавить к обычной Ubuntu пакетом `ubuntu-mate-desktop`. Я ни один
из них с этим драйвером **не проверял**, но пакеты из того же архива Ubuntu
26.04 должны подключаться так же, как описано ниже.

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

### 2. ⚠️ Собрать `nvidia-support`: этого пакета нет в архиве Ubuntu

`nvidia-legacy-340xx-driver` предзависит от `nvidia-installer-cleanup` и
зависит от `nvidia-support`. В индексах пакетов Ubuntu 24.04, 25.10 и 26.04
(main/restricted/universe/multiverse, amd64) этих пакетов нет. Без них `apt
install` откажет с ошибкой неразрешимых зависимостей, причём apt обвинит
сразу все пакеты в команде (`build-essential`, `cmake` и т.д.), хотя виноват
один: apt отклоняет всю транзакцию целиком.

Исходники есть в Debian (компонент contrib), собираем вручную:

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

Сборка проверена, пакеты получаются. Установка именно через `dpkg -i` в
таком виде отдельно не проверялась (у меня они шли через локальный
репозиторий).

### 3. Ставим сам драйвер

```bash
sudo apt update
sudo apt install -y nvidia-legacy-340xx-driver glx-alternative-nvidia dkms \
  build-essential linux-headers-generic
```

`dkms`, компилятор и заголовки ядра нужны для сборки модуля; обычно они
подтягиваются сами, указаны явно на всякий случай. По описанию piernov,
`glx-alternative-nvidia` (из репозитория `glx-alternatives-resolute`)
подменяет `libGLX.so.0` так, чтобы GLVND-стек 26.04 нашёл библиотеки
драйвера. Без него я не проверял.

### 4. ⚠️ Поставить X11-сессию: на Kubuntu 26.04 её нет из коробки

По данным обзоров, начиная с Kubuntu 25.10 Plasma по умолчанию ставится без
X11-сессии. Без этого шага в SDDM не будет пункта «Plasma (X11)».

```bash
sudo apt install -y plasma-session-x11 xorg
```

**Держи Plasma на 6.6.** По данным OMG! Ubuntu (октябрь 2026), в Plasma 6.8
пакета `plasma-session-x11` больше нет вовсе, а Kubuntu 26.10 (Plasma 6.7)
стал последним выпуском, где X11-сессию ещё можно поставить. Поэтому на 26.04
не переходи на более новый выпуск Kubuntu, а если подключаешь Kubuntu
Backports PPA, следи, чтобы Plasma не обновилась до 6.8 и выше: в этой версии
X11-сессии уже нет, а драйверу она нужна.

### 5. ⚠️ Ручной `xorg.conf`

Без `xorg.conf` у меня `glxinfo` выдавал `GLX missing on display`, хотя
`nvidia-smi` видел карту. После создания файла GLX заработал: `glxinfo`
показывает NVIDIA, `glxgears` рисует. Судя по всему, Xorg брал встроенный
драйвер вместо `nvidia` (лог Xorg до этого я не смотрел). Утилиты
`nvidia-xconfig` в сборке piernov нет, пишем вручную:

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

Секций `Monitor`/`InputDevice` нет намеренно, Xorg сам разбирается через udev
и EDID. `BusID` в моём случае не нужен (одна видеокарта).

Опции, которые раньше были у меня в конфиге (`NoLogo`, `Coolbits "4"`,
`RegistryDwords "EnableBrightnessControl=1"`), для работы не нужны, поэтому
здесь их нет. `NoLogo` убирает заставку NVIDIA, это косметика. `Coolbits`
по документации включает ручное управление вентилятором, но на этом
ноутбуке не проверялось (вентилятором управляет контроллер платы).
`EnableBrightnessControl` я добавлял, когда искал способ починить яркость,
а она заработала от параметра ядра `acpi_backlight=video`; что опция нужна,
не доказано.

В логе Xorg будет предупреждение, что ABI X-сервера (25.2) официально драйвером
не поддерживается, «драйвер продолжит загрузку». Это ожидаемо.

### 6. Перезагрузка

На экране входа выбери сессию **«Plasma (X11)»** в селекторе сессий. Проверка
после входа:

```bash
nvidia-smi
echo $XDG_SESSION_TYPE   # должно быть "x11"
glxinfo | grep vendor    # "NVIDIA Corporation" во всех трёх строках
```

## Если интернет медленный

У меня на ноутбуке интернет около 20 КБ/с, поэтому весь набор пакетов
(драйвер, `nvidia-support`, DKMS, компилятор, заголовки ядра,
`plasma-session-x11`, `xorg`) я готовил заранее на другой машине в live-сессии
той же версии Kubuntu и потом ставил с локального apt-репозитория. Готовых
скриптов здесь нет, но вот что у меня сработало или сломалось по пути:

- Live-сессия ничего не хранит между перезагрузками. Готовить набор надо за
  один сеанс и до перезагрузки скопировать на постоянный диск.
- Версия ядра в live-сессии должна совпадать с ядром установленной системы:
  модуль собирается DKMS из заголовков. Пока набор не использован, ядро на
  целевой системе лучше не обновлять
  (`sudo apt-mark hold linux-generic linux-image-generic linux-headers-generic`).
- `nvidia-support` нужно собрать и положить в локальный репозиторий **до**
  скачивания драйвера, иначе `apt-get --download-only` падает на
  неразрешимых зависимостях.
- Репозиторий лучше держать в `/srv`, а не в домашнем каталоге. Пока он лежал
  в `/home`, `apt update` его не видел (apt читает `file://` от имени
  пользователя `_apt`, а домашний каталог обычно закрыт); после переноса в
  `/srv` всё заработало.
- Индекс (нужен пакет `dpkg-dev`):
  `cd /srv/nvidia-offline-repo && dpkg-scanpackages . /dev/null | gzip -9c | sudo tee Packages.gz > /dev/null`.
  Источник: `deb [trusted=yes] file:///srv/nvidia-offline-repo ./`. Параметр
  `trusted=yes` отключает проверку подписи: за содержимое папки отвечаешь ты сам.
- Не забудь `plasma-session-x11` и `xorg`: без них на ноутбуке не будет пункта
  «Plasma (X11)» в SDDM, а чинить это без сети нечем.

## Регулировка яркости

### Встроенный экран ноутбука

**Что сработало:** параметр ядра `acpi_backlight=video`. После него
появилось устройство `/sys/class/backlight/acpi_video0`, и ползунок яркости
в трее Plasma заработал сам (PowerDevil подхватывает любое устройство из
`/sys/class/backlight/`). Это происходит и с загруженным проприетарным
драйвером.

**Главная ловушка, на которой ушло больше всего времени.** В `/etc/default/grub`
на этом ноутбуке строка была `GRUB_CMDLINE_LINUX_DEFAULT='quiet splash'`, с
**одинарными** кавычками. Команда `sed`, написанная под двойные кавычки,
молча ничего не меняла и не выдавала ошибок, так что параметр не доходил до
ядра, а мы думали, что он «не работает». Поэтому реально проверен только
`video`. Значения `vendor` и `native` фактически ни разу не применялись, и
говорить, что они не работают, нельзя.

Команда, работающая с обоими видами кавычек:

```bash
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub
sudo sed -i -E "s/^(GRUB_CMDLINE_LINUX_DEFAULT=['\"])/\1acpi_backlight=video /" /etc/default/grub
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub   # убедись, что параметр появился
sudo update-grub
sudo reboot
```

После перезагрузки обязательно проверь, что параметр дошёл до ядра:

```bash
cat /proc/cmdline             # должен быть acpi_backlight=video
ls /sys/class/backlight/      # должен появиться acpi_video0
```

**Запасной вариант, если `acpi_video0` не появился:** вызов ACPI-методов
прошивки напрямую через `acpi_call` (`acpi-call-dkms`). На этом ноутбуке он
сработал (яркость менялась командами в `/proc/acpi/call`), но в итоге не
понадобился. Нужные имена методов у каждой модели свои, их находят в таблице
DSDT:

```bash
sudo apt install -y acpica-tools
sudo cat /sys/firmware/acpi/tables/DSDT > dsdt.dat
iasl -d dsdt.dat
grep -n "_BCL\|_BCM\|_BQC" dsdt.dsl
```

На Aspire 7738 метод лежал по пути `\_SB.PCI0.PEGP.VGA.LCD._BCM`, а допустимые
значения (10, 20, …, 100) были в пакете `\BCLP`.

### Внешний монитор (VGA)

Заработал после установки `ddcutil`: PowerDevil управляет внешними
мониторами по DDC/CI, и появились отдельные ползунки для обоих экранов.

```bash
sudo modprobe i2c-dev
sudo apt install -y ddcutil
sudo ddcutil detect   # должен найти монитор
```

Встроенную панель `ddcutil` помечает как «laptop display, DDC/CI не
поддерживает», это нормально, она регулируется через `acpi_video0`. Не все
мониторы поддерживают DDC/CI (бывает выключено в меню монитора). Известная
недоработка PowerDevil: разработчики признают, что при одновременном
использовании встроенной и внешней панели реализация по сути рассчитана на
один дисплей. Если ползунок путается, управляй напрямую:
`sudo ddcutil setvcp 10 <0-100>`.

## nvidia-settings: тоже нужно собирать вручную

`nvidia-settings-legacy-340xx` — отдельный Debian source-пакет (contrib),
его нет ни в сборке piernov, ни в архиве Ubuntu 26.04. Сборка проверена,
программа запускается:

```bash
sudo apt install -y m4 libgtk2.0-dev libjansson-dev libvdpau-dev \
  libgl-dev libxext-dev libxv-dev libxxf86vm-dev xserver-xorg-dev

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

**Мониторинг GPU.** У этого драйвера `nvidia-smi` отвечает не на все запросы.
Работает: `nvidia-smi --query-gpu=temperature.gpu,pstate --format=csv`
(температура и P-состояние). Запросы мощности, частот и загрузки
возвращают `[Not Supported]`. Для загрузки используй
`nvidia-settings -t -q [gpu:0]/GPUUtilization`. Файла
`/proc/driver/nvidia/gpus/*/performance` на этой сборке **нет**.

## Сон (suspend): работает

Без правок уход в сон зависал: экран гас, вентилятор и светодиоды
продолжали работать, реагировал только `Magic SysRq`.

### Что найдено и помогло

Проверено пошагово через `/sys/power/pm_test` (см. диагностику ниже).

**1. systemd 256+ замораживает пользовательскую сессию перед сном.**
Обсуждается в [systemd#33626](https://github.com/systemd/systemd/issues/33626) и
[systemd#33083](https://github.com/systemd/systemd/issues/33083); по отзывам,
затрагивает не только NVIDIA, а в обсуждениях звучит версия, что корень
проблемы в ядре. На этой машине сообщение «Failed to freeze unit
'user.slice'» я не видел (журнал в момент зависания не успевал записаться),
так что связь установлена только по эффекту: после правки тест уровня
`freezer` перестал зависать.

```bash
for s in suspend hibernate hybrid-sleep suspend-then-hibernate; do
  sudo mkdir -p /etc/systemd/system/systemd-$s.service.d
  printf '[Service]\nEnvironment="SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false"\n' \
    | sudo tee /etc/systemd/system/systemd-$s.service.d/no-freeze-user-sessions.conf >/dev/null
done
sudo systemctl daemon-reload
```

**2. Отключение Wi-Fi и Bluetooth на время сна.** Это то, что в итоге
вылечило сон полностью. Комбинированный модуль Wi-Fi/Bluetooth Intel
(`iwlwifi`, `hci0`) мешал уходу в сон: тест уровня `devices` зависал, пока я
не заблокировал **оба** радиомодуля (одного Bluetooth оказалось
недостаточно). После блокировки модуля проблема со сном у меня исчезла
полностью. В логе при этом видны таймауты команд Bluetooth (`0x0c24`,
`0x0c52`) и сброс устройства `hci0`; механизм зависания я не доказал.
Сообщения «root hub lost power or was reset» в том же логе — вероятно,
обычные сообщения при выходе USB-контроллеров из сна, а не причина.

Хук, который делает то же самое автоматически: блокирует оба радиомодуля
перед сном и снимает блокировку после пробуждения:

```bash
sudo tee /usr/lib/systemd/system-sleep/wifi-bt-suspend-fix > /dev/null << 'EOF'
#!/bin/bash
case "$1" in
  pre)  rfkill block wifi; rfkill block bluetooth ;;
  post) rfkill unblock wifi; rfkill unblock bluetooth ;;
esac
EOF
sudo chmod +x /usr/lib/systemd/system-sleep/wifi-bt-suspend-fix
```

### Что осталось невыясненным

- Почему именно обмен с комбинированным модулем Wi-Fi/Bluetooth ломает
  уход в сон: механизм не доказан (см. выше).
- Нужен ли параметр `mem_sleep_default=deep`. Я добавлял его на пробу, на
  зависание при уходе в сон он не повлиял (оно происходило до реального
  сна). Если `cat /sys/power/mem_sleep` у вас и так показывает `[deep]`,
  параметр точно не нужен.

### Как диагностировать зависание сна

Обычный `journalctl -b -1` после жёсткой перезагрузки почти всегда пуст:
`SysRq+B` не делает `sync`, а в момент зависания journald уже заморожен.
Рабочая связка: `echo 1 | sudo tee /sys/power/pm_debug_messages` и
`pm_test=<уровень>` (`freezer` → `devices` → `platform` → `processors` →
`core`). Тест прогоняет переход пошагово и сам возвращает управление, если
уровень прошёл. Если завис, надёжнее уйти в чистую консоль (`sudo systemctl
stop sddm`, `Ctrl+Alt+F3`), включить подробный вывод
(`echo 8 | sudo tee /proc/sys/kernel/printk`) и после теста сохранить нужный
кусок журнала в файл (`journalctl -k --no-pager | sed -n 'N,Mp' > file.txt`).
Настройки сбрасываются после перезагрузки; вручную: `echo none | sudo tee
/sys/power/pm_test`.

## Известные проблемы

### ⚠️ Внезапные выключения: особенность моего ноутбука, причина не найдена

**Симптом:** питание пропадает мгновенно, без единого сообщения в журнале и в
`dmesg`. Бывает в простое и нередко сразу после вставки USB-флешки в холодный
ноутбук. Температура по conky при этом 35-40 °C (иногда выше), вентилятор
работает как на 24.04. На 24.04 (ядро 6.14, моя прежняя сборка драйвера) на том
же железе такого не наблюдалось. На 26.04 с драйвером NVIDIA выключения у меня
случались не реже двух раз в час.

**Вывод на сегодня:** по моим сведениям, у других пользователей этого
драйвера на 26.04 таких выключений нет, поэтому я считаю это особенностью
моего конкретного ноутбука, а не общей проблемой драйвера или этой
инструкции. Что именно с ним не так, я не выяснил. Если у вас такого нет,
этот раздел можно пропустить.

**Установлено:**
- В последнем захваченном `dmesg -w` запись оборвалась примерно через 1,3 с
  после вставки флешки, на первых SCSI-командах к ней
  (`sd 6:0:0:0: [sdd] Mode Sense`), до запуска udisks и Dolphin (точность
  около 0,5 с).
- На чистой системе без драйвера NVIDIA (nouveau, ядро 7.0.0-30) ноутбук
  отработал 3 ч 18 мин без обрыва, журнал закончился штатным завершением
  сессии. Флешку вставляли в холодную систему на 161-й секунде после старта,
  выключения не было.
- Перегрев маловероятен (низкие температуры, вентилятор исправен), батареи
  нет.
- Параметры ядра `tsc=unstable trace_clock=local` не помогли: они относятся к
  зависаниям по часам, а не к обрыву питания.
- Тест «консоль без драйвера NVIDIA» (`module_blacklist=nvidia
  systemd.unit=multi-user.target`) чистого результата не дал: ноутбук нашли
  выключенным, как именно, неизвестно.

**Не проверено:** то же 26.04 на более старом ядре (6.14); сравнение ядер
6.19 и 7.x на другом дистрибутиве; блок питания и разъём; ограничение
C-состояний (`processor.max_cstate=1`); `usbcore.autosuspend=-1` (в
захваченной загрузке параметр не был применён); `thermal.nocrt=1`
(маловероятно: нет характерной строки о критической температуре);
`GRUB_GFXPAYLOAD_LINUX=text` (консоль работает на simpledrm/fbcon, а не на
текстовой VGA).

**Если у вас такие же выключения, как зафиксировать момент обрыва.**
Журнал journald между загрузками у меня не сохранялся, на него полагаться
нельзя. Запускать после каждой загрузки (при перезагрузке фоновые процессы
пропадают):

```bash
sudo sysctl -w kernel.dmesg_restrict=0
stdbuf -oL dmesg -w > ~/crash-kernel.log 2>&1 &
disown
stdbuf -oL journalctl -f > ~/crash-journal.log 2>&1 &
disown
( while true; do sleep 0.5; sync; done ) &
disown
```

`dmesg -w` пишет примерно на секунду дальше `journalctl`, а из-за `sync` раз в
0,5 с момент обрыва определяется с точностью около полусекунды. Если запись
обрывается без единого сообщения об ошибке, система не успела ничего записать.

### ✅ Лавина падений `kioworker`/`drkonqi`: исправлено (подтверждено косвенно)

В журнале одного из выключений нашлось 113 запусков
`drkonqi-coredump-launcher` за 24 секунды. Первым падал `kioworker`
(генерация превью, `thumbnail.so`): по стеку вызовов тема Plasma
(`KDEPlasmaPlatformTheme6`) создаёт `QOpenGLContext` внутри headless-платформы
Qt `offscreen`, и она завершает процесс аварийно. Дальше обработчики падений
порождали друг друга.

Что сделано:
```bash
mkdir -p ~/.config/environment.d
echo "QT_OPENGL=software" > ~/.config/environment.d/qt-opengl-software.conf   # после перелогина
systemctl --user mask 'drkonqi-coredump-launcher@.service'                     # обрывает лавину
```
После этого, по словам автора, температура в простое держится ниже 60 °C, а
в логе вставки флешки падений не видно. Отдельной проверки через
`coredumpctl` после правки не делалось. Выключения при этом не прекратились,
значит, это не единственная причина.

### 🟡 Прочие наблюдения

- GPU в двухчасовом замере ни разу не вышел из `P0` (максимальное
  состояние) даже при простое. Вероятное объяснение: известное поведение
  NVIDIA с двумя активными мониторами; на этой машине не доказано.
  Частоты экранов (60.08 и 60.02 Гц) не совпадают; проверка с выравниванием
  не доведена до конца.
- Драйвер на ядре 7.0 выдаёт два `WARNING` ядра (`Unpatched return thunk in
  use`, `drm_open_helper`) и сообщение `NVRM: Your system is not currently
  configured to drive a VGA console`. На BIOS-системах при графическом режиме
  GRUB консоль работает на simpledrm/fbcon, а не на текстовой VGA. Связь с
  выключениями **не доказана**. Проверка `GRUB_GFXPAYLOAD_LINUX=text` не
  проводилась.
- `QT_QUICK_BACKEND=software` (положить в
  `~/.config/plasma-workspace/env/`) предлагался против падений самого
  `drkonqi`; **эффект не проверялся**.

## О репозитории

Это личные заметки по установке, а не поддерживаемый проект. Я не пишу и не
собираю сам драйвер, всю техническую работу делает
[piernov](https://github.com/piernov). Багов драйвера, патчей под ядро или
GLX-механизма я чинить не могу: по ним сразу в
[issues piernov](https://github.com/piernov/nvidia-graphics-drivers-340xx-resolute/issues)
или [glx-alternatives-resolute](https://github.com/piernov/glx-alternatives-resolute/issues).

[⬆ Наверх / выбор языка](#top)
