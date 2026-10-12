# Omarchy on the Steam Deck OLED

An encrypted Linux desktop on the Steam Deck that you can unlock without a
keyboard and switch into Valve's game mode at the press of a button.

No dual boot. One system, two sessions.

*[Deutsche Fassung](README.de.md)*

**This is stock Omarchy, not a fork and not a trimmed-down image.** Omarchy 4
is installed from its own installer and keeps getting its updates through
`omarchy-update`. Valve's kernel, fan control and game-mode packages are added
individually and **by name** from Valve's repositories — nothing is removed,
nothing is replaced by a custom build, and no prebuilt image is involved. That
is precisely why Phase 2 opens by blocking one upstream migration: because we
stay on upstream, we have to live with what upstream does.

The [`files/`](files/) directory mirrors the finished configuration at the
paths it belongs at, so you can copy a file instead of retyping it. **The guide
below is the source of truth** — the files are extracted from it, and anything
that needs explaining is explained here, not in a comment.

---

## What you end up with

| | |
|---|---|
| Base | Omarchy 4 (Arch + Hyprland), LUKS-encrypted, btrfs with snapshots |
| Kernel | Valve's `linux-neptune-618` — fan control, charge limit, TDP |
| Game mode | Valve's actual gamescope session with Steam, switching both ways |
| Unlocking | on-screen keyboard on the touchscreen, correctly rotated |
| Audio | internal speakers, headphone jack, mic with Valve's noise suppression |
| Safeguards | guards against update regressions, a `deck-health` check script |

Tested on **10 October 2026** on a **Steam Deck OLED (Galileo)**, BIOS F7G0114,
with Omarchy 4.0.4 and `linux-neptune-618` 6.18.53.valve1.

Almost all of it should apply to the LCD model (Jupiter) too; the device names
and the fan curve differ, the logic does not.

---

## Before you start

**The internal SSD is wiped completely.** SteamOS will be gone, along with every
installed game. Plan for the reinstall.

You will need:

- two USB sticks (Omarchy installer, SteamOS recovery image)
- a USB-C dock with a keyboard and preferably a monitor
- the power supply
- an hour for the base system, another two for the rest

The SteamOS recovery image is your way back to factory condition. Valve hosts it
at *help.steampowered.com*, search for "Steam Deck Recovery Instructions".

**What this document assumes.** That you are at home on a command line:
creating files, reading output, noticing when a command did something other
than what you expected. It explains this one device, not Linux. If you are
just starting out, do not run it — the steps in Phase 2 and Phase 7 will leave
the Deck unbootable if they are followed half-understood.

Build the install medium by **Omarchy's own instructions** (omarchy.org) — the
checksum and the writing tool live there and change without our involvement.

**Read two sections first:** "Pitfalls" at the end (four of them will otherwise
cost you half a day) and "Emergencies and getting back to SteamOS", so that you
know the way back before you need it.

---

## Phase 0 — Take stock

On the still-untouched SteamOS, in desktop mode:

```bash
cat /sys/class/dmi/id/sys_vendor /sys/class/dmi/id/board_name
uname -r
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

`board_name` must be `Galileo` (OLED) or `Jupiter` (LCD). Note the running
kernel — that is the Neptune kernel you will install later.

Fully update SteamOS first. That also brings controller and dock firmware up to
date, and that survives the wipe.

---

## Phase 1 — Install Omarchy, encrypted

Grab the Omarchy ISO from `iso.omarchy.org`, verify the checksum, write it to a
stick.

**Shut the Deck down completely** (not a restart). Then **hold Volume Down and
briefly press Power**, keeping Volume Down held until the Boot Manager appears.
Select the stick.

In the installer:

- **Full disk install** onto the internal NVMe
- **Accept encryption** — do not opt out with Ctrl+C
- hostname and user as you like

> **Encryption is not optional if you want autologin.** Omarchy's installer only
> writes `autologin.conf` when the disk is encrypted. Without autologin you land
> on the login screen at every boot — on a device without a keyboard, a nuisance.

### The passphrase

You will later type it on an **on-screen keyboard with a US layout**. Stick to:

- **allowed:** lowercase `a` through `x`, digits, spaces
- **forbidden:** `y`, `z`, accented characters, anything symbolic

Long and boring beats short and cryptic. Four ordinary words with spaces are
safer than eight characters of line noise — and far more pleasant on a
touchscreen.

After first boot: **no `omarchy-update` yet.** Phase 2 first.

---

## Phase 2 — Valve's kernel and the boot order

### 2.1 Block the migration

Omarchy ships a migration that installs `linux-omarchy` and rewrites the boot
order to point at it. That would dethrone the Neptune kernel. `omarchy-migrate`
skips any migration that already has a marker file — so we create it up front:

```bash
mkdir -p ~/.local/state/omarchy/migrations
touch ~/.local/state/omarchy/migrations/1789325478.sh
ls -1 ~/.local/state/omarchy/migrations | wc -l
```

The `mkdir -p` is needed because on a fresh system the directory does not exist
yet. The number at the end is just a receipt — note it; after the next update it
must have gone up by at least one.

> **Two details that will otherwise silently defeat you:** the directory is
> `.local/state`, not `.local/share` and not `/var/lib/omarchy`. And the marker
> carries the **`.sh`** suffix.

Check whether the migration even exists on your snapshot of Omarchy yet:

```bash
ls -1 /usr/share/omarchy/migrations | grep 1789325478
```

If it is not there, it will arrive with a later update — the marker works
regardless.

### 2.2 Now update

```bash
sudo omarchy-snapshot create
omarchy-update
grep BOOT_ORDER /etc/default/limine
```

The last command must print **nothing**. A line mentioning `linux-omarchy` means
the block did not take.

### 2.3 Add Valve's package repositories

```bash
sudo cp /etc/pacman.conf /etc/pacman.conf.bak
sudo tee -a /etc/pacman.conf > /dev/null << 'EOF'

[jupiter-main]
Usage = Sync Search Install
SigLevel = Optional TrustAll
Server = https://steamdeck-packages.steamos.cloud/archlinux-mirror/$repo/os/$arch

[holo-main]
Usage = Sync Search Install
SigLevel = Optional TrustAll
Server = https://steamdeck-packages.steamos.cloud/archlinux-mirror/$repo/os/$arch
EOF

sudo pacman -Sy
sudo pacman -S holo-keyring
sudo sed -i 's/SigLevel = Optional TrustAll/SigLevel = Required DatabaseOptional/' /etc/pacman.conf
sudo pacman -Syy
```

> **`Usage = Sync Search Install` is the most important part of that file.**
> `Upgrade` is deliberately absent. That way pacman may only install what you ask
> for **by name** from Valve's repositories, and never touches them during system
> upgrades. Without that line, your next `omarchy-update` pulls SteamOS' Mesa and
> friends into your Arch system.

`SigLevel = Optional TrustAll` is a chicken-and-egg problem: Valve's keyring
lives in Valve's repository, and without the keyring pacman cannot check the
signatures. For as long as that line stands, **pacman accepts anything from
those two repositories, including something slipped in**. That is why the state
lasts exactly one package install: the `sed` above puts it back to
`Required DatabaseOptional` immediately. Verify it:

```bash
grep -A2 'jupiter-main\]' /etc/pacman.conf
```

If it still says `Optional TrustAll`, the `sed` did not run — do not go on
until it has. The backup is at `/etc/pacman.conf.bak`.

### 2.4 Install the kernel

```bash
pacman -Si jupiter-main/linux-neptune-618
sudo pacman -S jupiter-main/linux-neptune-618 jupiter-main/linux-neptune-618-headers
```

> **Take `linux-neptune-618`, not `linux-neptune-72`.** Both sit in
> `jupiter-main`, but 618 is what Valve actually ships on the Deck. The 7.2 line
> is groundwork for a future SteamOS. "Demonstrably runs on this hardware" beats
> "is newer".
>
> `jupiter-staging` is **older** than `jupiter-main` for both lines. Don't let the
> name mislead you.

### 2.5 Pin the boot order

First check whether a `BOOT_ORDER` is already there:

```bash
grep -n BOOT_ORDER /etc/default/limine
```

Nothing printed means you are fine. If a line comes back, delete it first —
otherwise `tee -a` appends a second one. Duplicates are not fatal, the **last
one wins**, but you will hunt for it later:

```bash
sudo sed -i '/^[[:space:]]*BOOT_ORDER[[:space:]]*=/d' /etc/default/limine
```

Then write it:

```bash
sudo tee -a /etc/default/limine > /dev/null << 'EOF'
BOOT_ORDER="linux-neptune-618, linux, *, *fallback, Snapshots"
EOF

cat /etc/default/limine
sudo mount /boot
sudo limine-mkinitcpio
```

If `mount` says "already mounted", that is fine — `/boot` was mounted already.
Any other error is not.

The line must end up reading exactly this, with both asterisks:

```
BOOT_ORDER="linux-neptune-618, linux, *, *fallback, Snapshots"
```

> **`/etc/default/limine` takes precedence over `/etc/limine-entry-tool.conf`.**
> The latter also carries a `BOOT_ORDER`, which gets overridden. The file to
> guard is `/etc/default/limine` — which is also the one Omarchy's migration
> manipulates.

The Arch kernel stays installed as the second entry. That is your lifeboat: if
Neptune fails to boot, pick it from the Limine menu with the D-pad and confirm
with **Enter** or the **A button**.

### 2.6 Reboot and verify

```bash
uname -r
grep -H . /sys/class/hwmon/hwmon?/name
find /sys/devices -name max_battery_charge_level
find /sys/devices -name fan1_target
```

Expect roughly this — your numbers may differ:

```
6.18.53-valve1-1-neptune-618
/sys/class/hwmon/hwmon0/name:amdgpu
/sys/class/hwmon/hwmon5/name:steamdeck_hwmon
/sys/devices/.../hwmon/hwmon5/max_battery_charge_level
/sys/devices/.../hwmon/hwmon5/fan1_target
```

`steamdeck_hwmon` must show up and both files must exist. That is the proof that
you have Valve's kernel rather than merely hoping you do. If `steamdeck_hwmon`
is absent you are still on the old kernel — then the boot order is wrong and
you go back to 2.5.

---

## The boot guard

The boot order from 2.5 is the setting an update is most likely to undo —
quietly, and you find out at the next boot with the wrong kernel. So we set the
guard up **now**, right behind the phase it protects, rather than at the end.

### `/usr/local/bin/deck-boot-guard`

Restores `BOOT_ORDER` as soon as anything changes it.

```bash
sudo tee /usr/local/bin/deck-boot-guard > /dev/null << 'DECKEOF'
#!/usr/bin/env bash
set -euo pipefail

CONF=/etc/default/limine
PATTERN='^[[:space:]]{0,8}BOOT_ORDER[[:space:]]{0,8}='
WANT='BOOT_ORDER="linux-neptune-618, linux, *, *fallback, Snapshots"'

MOUNTED_BY_US=0
cleanup() {
  if [ "$MOUNTED_BY_US" = "1" ]; then
    umount /boot || logger -t deck-boot-guard "warning: could not unmount /boot"
  fi
}
trap cleanup EXIT

current=$(grep -E "$PATTERN" "$CONF" | tail -n 1 || true)
if [ "$current" = "$WANT" ]; then
  exit 0
fi

logger -t deck-boot-guard "BOOT_ORDER wrong or missing, repairing"
mkdir -p /var/backups
cp -a "$CONF" "/var/backups/limine.$(date +%s)"

grep -vE "$PATTERN" "$CONF" > "$CONF.new" || true
echo "$WANT" >> "$CONF.new"
cat "$CONF.new" > "$CONF"
rm -f "$CONF.new"
chmod 644 "$CONF"

if ! mountpoint -q /boot; then
  mount /boot
  MOUNTED_BY_US=1
fi

/usr/bin/limine-mkinitcpio
logger -t deck-boot-guard "BOOT_ORDER restored"
DECKEOF

sudo chmod 755 /usr/local/bin/deck-boot-guard
sudo bash -n /usr/local/bin/deck-boot-guard && echo "Syntax OK"
```

> **Mounting `/boot` is not optional.** The Limine tools do **not** mount the
> partition themselves; they require it to be mounted already and otherwise abort
> with `Boot path '/boot' is not a mounted FAT32 boot partition`. You will not
> notice interactively, but you will from a service.

Plus two units:

```bash
sudo tee /etc/systemd/system/deck-boot-guard.service > /dev/null << 'DECKEOF'
[Unit]
Description=Keep the Deck boot order intact
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/deck-boot-guard

[Install]
WantedBy=multi-user.target
DECKEOF
```

```bash
sudo tee /etc/systemd/system/deck-boot-guard.path > /dev/null << 'DECKEOF'
[Unit]
Description=Watch /etc/default/limine

[Path]
PathChanged=/etc/default/limine
Unit=deck-boot-guard.service

[Install]
WantedBy=multi-user.target
DECKEOF
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now deck-boot-guard.path
sudo systemctl enable deck-boot-guard.service
```

Now break it on purpose, to see that it holds:

```bash
echo 'BOOT_ORDER="linux-omarchy, linux"' | sudo tee -a /etc/default/limine
sleep 5
grep BOOT_ORDER /etc/default/limine
journalctl -t deck-boot-guard -n 10 --no-pager
mountpoint /boot
```

The `.path` unit reacts instantly; the five seconds are just slack for
`limine-mkinitcpio` rewriting the boot menu. Afterwards there must be exactly
one `BOOT_ORDER` line again, the right one, the journal must say "BOOT_ORDER
restored" — and `/boot` must **not** be mounted, because the guard cleans up
after itself.

---

## Phase 3 — Fan, power, battery

```bash
sudo pacman -S jupiter-main/jupiter-fan-control holo-main/steamos-manager
sudo systemctl enable --now jupiter-fan-control.service
sudo systemctl enable --now steamos-manager.service
systemctl --user enable --now steamos-manager.service
```

> **The user service is not optional.** The system daemon only exposes the root
> interfaces (TDP, fan, charge limit). Session management — switching between
> desktop and game mode — lives in the **user** daemon. Valve only starts it
> inside the gamescope session; you need it in the desktop too.

`jupiter-fan-control` picks its curve from `board_name` on its own, so
`galileo-config.yaml` for the OLED Deck. To check the maths yourself:

hwmon numbers shift between boots, so never write one down — find the sensor
by its driver name instead:

```bash
DECK=$(for d in /sys/class/hwmon/hwmon*; do
         [ "$(cat "$d/name" 2>/dev/null)" = "steamdeck_hwmon" ] && echo "$d"
       done)
echo "$DECK"
cat "$DECK/fan1_input" "$DECK/fan1_target"
```

`fan1_target` is what the controller wants, `fan1_input` what the fan actually
does. A wide gap between them means something is wrong.

To make sense of the numbers: the curve in `galileo-config.yaml` is quadratic in
CPU temperature and its fitted point sits at `(60 °C, 2400 rpm)`. That is
**not** the figure you will read there — the controller also takes current APU
power into account and stays below it under light load. Around 2000 rpm at 60 °C
while idle is therefore normal, not a defect.

**Charge limit** — a substantial favour to the battery if the device mostly sits
on the dock:

```bash
echo 'SUBSYSTEM=="hwmon", ATTR{name}=="steamdeck_hwmon", ATTR{max_battery_charge_level}="80"' | sudo tee /etc/udev/rules.d/99-deck-charge-limit.rules
```

The rule matches on the driver name rather than on an hwmon number, for the
reason just given. It does **not** take effect immediately, though: udev only
applies new rules to devices as they appear. So nudge it and check:

```bash
sudo udevadm control --reload
sudo udevadm trigger -s hwmon
cat "$DECK/max_battery_charge_level"
```

It must read `80`. Before the rule it reads `0`, meaning "no limit".

**Power button.** Omarchy already binds it and uses it to open its own power
menu. logind therefore has to stay out of the way, or both will act on it:

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/10-deck-powerkey.conf > /dev/null << 'EOF'
[Login]
HandlePowerKey=ignore
HandlePowerKeyLongPress=poweroff
EOF
sudo systemctl reload systemd-logind
```

A long press then shuts down cleanly before the hardware cuts power after ten
seconds.

The menu will still flash up and vanish again, though. The reason: the Deck
reports **two** power button devices (`power-button` and `power-button-1`, both
visible in `hyprctl devices`). Omarchy's default binding is a toggle — the first
press opens, the second closes immediately. The fix is the `summon` verb instead
of `toggle`:

```bash
tee -a ~/.config/hypr/bindings.lua > /dev/null << 'EOF'

hl.unbind("XF86PowerOff")
o.bind("XF86PowerOff", "Power menu", "omarchy-menu summon system", { locked = true })
EOF
hyprctl reload
```

A second press no longer closes the menu — use `Esc` or tap outside it.

None of this applies in game mode — Omarchy is not there. That needs a service
of its own, see Phase 4.6.

**You should now have:** a fan that audibly tracks load,
`max_battery_charge_level` reading `80`, and a short press of the power button
opening Omarchy's power menu without suspending the device. If any of those is
off, go back before starting Phase 4.

---

## Phase 4 — Game mode

### 4.1 Shield gamescope from updates

Arch has its **own** package called `gamescope`. Valve's variant additionally
contains the entire session. The moment Arch ships a higher version number, your
next system upgrade replaces Valve's package with Arch's — and game mode
disappears without a word.

```bash
grep -q '^IgnorePkg = gamescope' /etc/pacman.conf \
  || sudo sed -i '/^\[options\]/a IgnorePkg = gamescope' /etc/pacman.conf
grep -n -A4 '^\[options\]' /etc/pacman.conf
```

`IgnorePkg` **must** live in the `[options]` section — hence the `sed`, which
inserts the line directly after it instead of appending. Appended to the end of the
file it lands inside the last repository block and does nothing. The `grep`
in front makes the command repeatable.

From now on you update gamescope by hand:

```bash
sudo pacman -S jupiter-main/gamescope
```

Because the package is on the ignore list, pacman will ask
`gamescope is in IgnorePkg. Install anyway? [Y/n]` — here **`y`** is the right
answer. That prompt is the whole point of the exercise: it also appears when a
system upgrade wants to touch the package, and there you answer **`n`**.

### 4.2 Install

```bash
sudo pacman -S jupiter-main/gamescope holo-main/holo-session-selection jupiter-main/steamos-alias
sudo pacman -S jupiter-main/steam-jupiter-stable jupiter-main/steam_notif_daemon
```

With `steam-jupiter-stable` pacman asks which 32-bit Vulkan driver to use. It
lists the providers with numbers and prompts `Enter a number (default=1)`.

**Type the number of `lib32-vulkan-radeon` from `multilib`.** Not the
identically named one from `jupiter-main`, and under no circumstances just
Enter — the default `1` is the NVIDIA driver.

Then one check that saves you half a day later on:

```bash
pacman -Qq jupiter-legacy-support 2>/dev/null \
  && echo "WARNING: installed, see Pitfalls" \
  || echo "not installed - good"
```

The package must not show up here. Why is under "Pitfalls".

`steamos-alias` creates the compatibility symlink
`/usr/bin/steamos-session-select → holo-session-select` that Steam calls.

### 4.3 Run Steam once on the desktop

```bash
steam
```

Sign in, let it finish, quit. This separates concerns: if something breaks later,
you know it is not Steam's first-run setup.

### 4.4 Wire up session management

```bash
systemctl --user restart steamos-manager.service
steamosctl get-valid-desktop-sessions
steamosctl set-default-desktop-session omarchy.desktop
steamosctl set-default-login-mode desktop
```

> **Restarting the user daemon is required** if it started before gamescope was
> installed: it checks at startup whether a game-mode session exists and hides
> the `SessionManagement1` interface otherwise. Symptom:
> `Unknown interface 'com.steampowered.SteamOSManager1.SessionManagement1'`.

### 4.5 Switching

```bash
steamosctl switch-to-game-mode
```

Back via **Steam button → Power → Switch to Desktop**.

Switching into game mode only applies to that session: SDDM deletes the temporary
login file at every start. If game mode hangs, a reboot puts you back on the
desktop. That is a real safety net, not an accident.

For everyday use, a clickable launcher that works without a keyboard:

```bash
mkdir -p ~/.local/share/applications
tee ~/.local/share/applications/game-mode.desktop > /dev/null << 'EOF'
[Desktop Entry]
Type=Application
Name=Game Mode
Exec=steamosctl switch-to-game-mode
Icon=steam
Terminal=false
Categories=Game;
EOF
```

A keyboard shortcut is **not** recommended: one misfire ends the whole desktop
session along with every open window.

**Nicer still: put the switch into the Omarchy menu.** Then the way in sits
where the way out already is — both through a menu, both without a keyboard.

Omarchy provides an extensions file for exactly this, and it survives updates:

```bash
python3 - << 'EOF'
import os
d = os.path.expanduser("~/.config/omarchy/extensions")
os.makedirs(d, exist_ok=True)
p = os.path.join(d, "omarchy-menu.jsonc")
if not os.path.exists(p):
    open(p, "w").write("{\n}\n")

s = open(p).read()
entry = ('"system.gamemode": {"icon":"' + chr(0xF11B) +
         '","label":"Game Mode","when":"command -v steamosctl",'
         '"action":"steamosctl switch-to-game-mode"}')

if "system.gamemode" not in s:
    i = s.rstrip().rfind("}")
    head = s[:i].rstrip()
    sep = "" if head.endswith("{") else ","
    open(p, "w").write(head + sep + "\n  " + entry + "\n" + s[i:])

print(open(p).read())
EOF

omarchy-menu refresh
```

The script creates the directory and the file if they do not exist yet, and
only adds the comma when an entry already precedes it — otherwise you would get
invalid JSON. Running it twice changes nothing.

The id `system.gamemode` files the entry under **System** — the very menu the
power button opens. Without the dot (`gamemode`) it lands on the root menu.

`when` hides the row if `steamosctl` is missing; Omarchy uses the same technique
for "Hibernate". The icon is assembled from its code point, because Nerd Font
glyphs from the Unicode private use area rarely survive a trip through a
clipboard intact.

### 4.6 Power button in game mode

On the desktop, Omarchy catches the button (Phase 3). In game mode there is no
Omarchy — there you need Valve's own daemon, or pressing it does nothing at all:

```bash
sudo pacman -S holo-main/steamos-powerbuttond
```

It is a **user service** with `Requisite=gamescope-session.service`, so it only
starts inside the game session and leaves the desktop alone. Its hardware
database only tags Valve devices and rewrites no scancodes — the Legion Go and
AYA NEO entries in it concern other hardware.

Checking whether it is enabled is awkward from **inside** the game session, so
do it on the desktop beforehand:

```bash
systemctl --user is-enabled steamos-powerbuttond.service
```

Ours already said `enabled` after installation. If it says `disabled`, fix that:

```bash
systemctl --user enable steamos-powerbuttond.service
```

It cannot be started on the desktop — `Requisite=gamescope-session.service`
prevents that deliberately. That is not an error.

The udev tag only takes effect after a reboot.

**You should now have:** `steamosctl switch-to-game-mode` taking you into the
Steam interface, the Steam button bringing you back via **Power → Switch to
Desktop**, and a reboot always landing on the desktop. If you cannot get back,
the reboot is the way out — that is by design.

---

## Phase 5 — Audio

Without this step you get sound through the monitor while docked and **nothing
at all** undocked. The volume indicator then reports `auto_null`.

Cause: the Deck's audio chip needs SOF firmware and a topology description that
exist **only at Valve**. Arch's `sof-firmware` does not contain them.

```bash
sudo pacman -S jupiter-main/steamdeck-dsp
sudo modprobe -r snd_sof_amd_vangogh
sudo modprobe snd_sof_amd_vangogh
cat /proc/asound/cards
```

A second card must appear:

```
1 [sofnau8821max]: sof-nau8821-max - sof-nau8821-max
                   Valve-Galileo-1
```

The speaker equalisation then runs **inside the chip's DSP**, not in PipeWire —
which is why it never shows up as a filter. That is correct.

Valve's microphone processing (noise suppression) does run in PipeWire, and its
configuration lands somewhere only SteamOS' patched PipeWire looks:

```bash
sudo mkdir -p /etc/pipewire/filter-chain.conf.d
sudo cp /usr/share/pipewire/hardware-profiles/valve-galileo/filter-chain.conf.d/filter-chain.conf \
        /usr/share/pipewire/hardware-profiles/valve-galileo/filter-chain.conf.d/context-properties.conf \
        /etc/pipewire/filter-chain.conf.d/
systemctl --user restart filter-chain
wpctl status
```

**You should now have:** `cat /proc/asound/cards` listing two cards, `wpctl
status` showing speakers, headphones and a microphone, and sound coming out of
the device itself while undocked. If the volume indicator still says
`auto_null`, the firmware is missing — do not move on, go back up.

---

## Phase 6 — Brightness

On Arch, by default nobody but root may change the backlight. You will not notice
on the desktop, because Omarchy goes through logind. gamescope writes the sysfs
file directly, fails, and says nothing — the slider moves, the panel does not.

```bash
sudo pacman -S brightnessctl
sudo usermod -aG video "$USER"

sudo tee /etc/udev/rules.d/90-deck-backlight.rules > /dev/null << 'EOF'
ACTION=="add", SUBSYSTEM=="backlight", RUN+="/usr/bin/chgrp video /sys/class/backlight/%k/brightness", RUN+="/usr/bin/chmod g+w /sys/class/backlight/%k/brightness"
EOF
cat /etc/udev/rules.d/90-deck-backlight.rules

sudo udevadm control --reload
sudo udevadm trigger -s backlight -c add
ls -la /sys/class/backlight/amdgpu_bl0/brightness
```

Afterwards the file belongs to group `video` with `rw-rw-r--`. The group
membership only takes effect after a fresh login.

Arch's `brightnessctl` ships **no** udev rule, unlike some other distributions.
Hence writing our own.

Test it **undocked**: while docked the active display is the external monitor,
and Linux cannot control its brightness. A greyed-out slider is the right answer
there, not a bug.

**You should now have:** the slider in game mode actually changing brightness
while undocked, and `brightnessctl set 50%` running on the desktop without
`sudo`.

---

## Phase 7 — Unlocking without a keyboard

The delicate part. It modifies the initramfs — the very piece of software that
unlocks your encrypted disk.

### Why buffyboard and not unl0kr

Both come from the same project (**buffybox**, postmarketOS).

- **unl0kr** draws a complete password field and **replaces** the `encrypt` hook.
  If it fails to start, the unlock loop spins forever — you cannot get into your
  system.
- **buffyboard** only draws a keyboard and injects keystrokes through `uinput`.
  The normal password prompt stays. If it fails to start, you keep typing on a
  USB keyboard.

On top of that, the decisive difference: **unl0kr cannot rotate.** Software
rotation was removed there over performance regressions (see `CHANGELOG.md` in
the source). buffyboard has it — and even picks up the value from
`/sys/class/graphics/fbcon/rotate` automatically.

The Deck's panel is mounted in portrait. Without rotation, everything lies
sideways.

### 7.1 Safety net

```bash
sudo omarchy-snapshot create
```

Every snapshot stores **its own copy of the kernel image including the
initramfs** and appears in the Limine menu. That is a complete way back,
unrelated to your new initramfs — and selectable with the D-pad.

Keep a USB keyboard attached throughout the test.

### 7.2 Determine the console rotation

```bash
echo 1 | sudo tee /sys/class/graphics/fbcon/rotate_all
```

Then `Ctrl+Alt+F3` and look at the **Deck's screen**; `Ctrl+Alt+F1` or
`Ctrl+Alt+F2` takes you back to the graphical session. Those key combinations
need the USB keyboard from 7.1 — one of the reasons it should stay plugged in.

For the OLED Deck, **`1`** is correct (90° clockwise). Try values `0` through
`3`; the change takes effect instantly. On the LCD model (Jupiter) a different
value may be right — try it rather than copying the `1`.

### 7.3 Install buffyboard and test it safely

```bash
yay -S buffyboard
```

**Test it before touching the initramfs** — on a text console, where nothing can
break:

```bash
# Ctrl+Alt+F3, log in
echo 1 | sudo tee /sys/class/graphics/fbcon/rotate_all
sudo buffyboard
```

Check: does the keyboard appear? Is it the right way up? Does the touchscreen
respond? Do the **correct** characters arrive?

Quit it with `Ctrl+C` on the USB keyboard, and return to the desktop with
`Ctrl+Alt+F1` or `Ctrl+Alt+F2`.

This test answers the three questions the whole undertaking hinges on, in ten
minutes. Use it.

On layout: buffyboard sends **US keyboard scancodes**. With a German console
layout, `y` and `z` are swapped and symbols land elsewhere. Hence the passphrase
restriction from Phase 1.

### 7.4 Wiring it into the initramfs

buffyboard ships no mkinitcpio hook. Three files are needed.

**First check what your decryption hook is called:**

```bash
grep '^HOOKS' /etc/mkinitcpio.conf
```

It must say `encrypt`. If yours says `sd-encrypt` (the systemd variant),
replace `encrypt` in the third file below with `sd-encrypt` — otherwise the
loop inserts **nothing at all**, silently, the build succeeds, and you find out
at the next boot.

**`/etc/initcpio/install/buffyboard`**

```bash
sudo tee /etc/initcpio/install/buffyboard > /dev/null << 'DECKEOF'
#!/bin/bash

build() {
    add_module 'uinput'
    add_module 'hid-generic'
    add_module 'hid-multitouch'
    add_module 'i2c-hid'
    add_module 'i2c-hid-acpi'

    add_binary '/usr/bin/buffyboard'

    if [ -f /etc/buffyboard.conf ]; then
        add_file '/etc/buffyboard.conf'
    fi

    add_file '/usr/lib/udev/hwdb.bin'
    add_binary '/usr/lib/udev/libinput-device-group'
    add_binary '/usr/lib/udev/libinput-fuzz-extract'
    add_binary '/usr/lib/udev/libinput-fuzz-to-zero'

    map add_udev_rule '60-evdev.rules' '60-input-id.rules' '65-libwacom.rules' '70-mouse.rules' '70-touchpad.rules' '80-libinput-device-groups.rules' '90-libinput-fuzz-override.rules'

    add_full_dir '/usr/share/libinput'

    add_runscript
}

help() {
    cat <<HELPEOF
Starts buffyboard, an on-screen keyboard on the framebuffer.
HELPEOF
}
DECKEOF
```

> **Those five `add_module` lines are the crux.** unl0kr's own hook uses
> `add_checked_modules '/input/touchscreen/'` — and the Deck's touchscreen does
> not live there, it lives among the HID devices. Rely on that hook and the
> keyboard comes up and responds to nothing.

**`/etc/initcpio/hooks/buffyboard`**

```bash
sudo tee /etc/initcpio/hooks/buffyboard > /dev/null << 'DECKEOF'
#!/usr/bin/ash

run_hook() {
    modprobe -q uinput >/dev/null 2>&1

    if [ -x /usr/bin/buffyboard ]; then
        /usr/bin/buffyboard -r 1 >/dev/null 2>&1 &
        echo $! > /run/buffyboard.pid
        sleep 1
    fi
}

run_cleanuphook() {
    if [ -f /run/buffyboard.pid ]; then
        kill "$(cat /run/buffyboard.pid)" >/dev/null 2>&1
        rm -f /run/buffyboard.pid
    fi
}
DECKEOF
```

> **Why `-r 1` when buffyboard reads the rotation from fbcon anyway?** Because
> the two work together and the flag is the belt to the braces: should the
> kernel command line from 7.5 ever get lost, the keyboard still comes up the
> right way round. Both paths set the same value; they do not conflict. **On
> the LCD Deck** put the value you determined in 7.2 here, not a blind `1`.

**`/etc/mkinitcpio.conf.d/zz-deck-unlock.conf`**

```bash
sudo tee /etc/mkinitcpio.conf.d/zz-deck-unlock.conf > /dev/null << 'DECKEOF'
# Drop plymouth from the initramfs and slot buffyboard in before encrypt.
_deck_hooks=()
for _deck_h in "${HOOKS[@]}"; do
    if [ "$_deck_h" = "plymouth" ]; then
        continue
    fi
    if [ "$_deck_h" = "encrypt" ]; then
        _deck_hooks+=(buffyboard)
    fi
    _deck_hooks+=("$_deck_h")
done
HOOKS=("${_deck_hooks[@]}")
unset _deck_hooks _deck_h
DECKEOF
```

mkinitcpio sources these three files rather than executing them, so they need
no execute bit. Check:

```bash
ls -l /etc/initcpio/install/buffyboard /etc/initcpio/hooks/buffyboard \
      /etc/mkinitcpio.conf.d/zz-deck-unlock.conf
sudo bash -n /etc/mkinitcpio.conf.d/zz-deck-unlock.conf && echo "Syntax OK"
```

> **Plymouth has to go.** It occupies the display, while buffyboard wants to
> shrink the text console and sit underneath it. Without Plymouth the `encrypt`
> hook shows a plain text prompt — which respects the fbcon rotation and stands
> upright.
>
> The loop rewrites the hook list rather than hard-coding it. If Omarchy changes
> its hooks, this adapts.

### 7.5 Make the console rotation permanent

```bash
sudo tee -a /etc/default/limine > /dev/null << 'EOF'
KERNEL_CMDLINE[default]+=" fbcon=rotate:1"
EOF
sudo bash -n /etc/default/limine && echo "Syntax OK"
```

### 7.6 Build and verify

```bash
sudo mount /boot          # "already mounted" is fine
sudo limine-mkinitcpio
```

`[buffyboard]` must appear in the output and `[plymouth]` must be gone.

```bash
sudo objcopy -O binary --only-section=.initrd \
  /boot/EFI/Linux/omarchy_linux-neptune-618.efi /tmp/initrd.img
lsinitcpio -l /tmp/initrd.img | awk -F/ '{print $NF}' \
  | grep -E "^(hid-multitouch|i2c-hid|i2c-hid-acpi|amdgpu|uinput)" | sort -u
lsinitcpio -l /tmp/initrd.img | grep -ci plymouth
```

Expected: five modules, and `0` for plymouth. `amdgpu` is not optional — without
the graphics driver there is no `/dev/fb0` and buffyboard has nothing to draw on.

If you only see four, work out which one is missing. For `uinput` that is not a
fault: it may be built into the kernel rather than existing as a module, in
which case there is no file to go into the image. All that matters then is
whether `/dev/uinput` exists at runtime:

```bash
ls -l /dev/uinput
```

If one of the HID modules is missing, however, the touchscreen will not respond
later.

### 7.7 Reboot

Prompt on top, keyboard below, both upright.

If something goes wrong: hold Power for ten seconds, then in the Limine menu use
the D-pad to reach **Snapshots** and pick the top entry.

---

## The `deck-health` check script

The boot guard behind Phase 2 intervenes. This one does **not** — it only
measures and tells you what is off. Fourteen checks that measure the actual state — not whether a package is
installed:

1. Neptune kernel running
2. `BOOT_ORDER` points at it first
3. migration blocked
4. `steamdeck_hwmon` present, charge limit active
5. `jupiter-fan-control` running
6. `steamos-manager` (system) running
7. `steamos-manager` (user) running
8. gamescope is Valve's build
9. gamescope excluded from upgrades
10. boot guard active
11. `/boot` mounted
12. keyboard-less unlocking wired up
13. internal sound card detected
14. brightness controllable

```bash
sudo tee /usr/local/bin/deck-health > /dev/null << 'DECKEOF'
#!/usr/bin/env bash

ok=0
fail=0

say() {
  if [ "$1" = "1" ]; then
    echo "  [ OK ]  $2"; ok=$((ok+1))
  else
    echo "  [MISS]  $2"; fail=$((fail+1))
  fi
}

echo "Deck status"; echo

k=$(uname -r)
echo "$k" | grep -q neptune && say 1 "kernel $k" || say 0 "kernel $k is not a Neptune kernel"

b=$(grep -E '^[[:space:]]{0,8}BOOT_ORDER' /etc/default/limine 2>/dev/null | tail -n 1)
echo "$b" | grep -q "linux-neptune-618, linux" && say 1 "BOOT_ORDER correct" || say 0 "BOOT_ORDER: $b"

[ -f "$HOME/.local/state/omarchy/migrations/1789325478.sh" ] \
  && say 1 "migration blocked" || say 0 "migration block missing"

f=$(find /sys/devices -name max_battery_charge_level -print -quit 2>/dev/null)
[ -n "$f" ] && say 1 "steamdeck_hwmon present, charge limit $(cat "$f") percent" \
            || say 0 "steamdeck_hwmon missing"

for s in jupiter-fan-control steamos-manager; do
  systemctl is-active --quiet "$s" && say 1 "system service $s running" \
                                   || say 0 "system service $s not running"
done

systemctl --user is-active --quiet steamos-manager \
  && say 1 "user service steamos-manager running" \
  || say 0 "user service steamos-manager not running"

pacman -Qlq gamescope 2>/dev/null | grep -q start-gamescope-session \
  && say 1 "gamescope is Valve's build" \
  || say 0 "gamescope is Arch's build, game mode gone"

grep -q "IgnorePkg = gamescope" /etc/pacman.conf \
  && say 1 "gamescope excluded from upgrades" \
  || say 0 "gamescope not excluded"

systemctl is-active --quiet deck-boot-guard.path \
  && say 1 "boot guard active" || say 0 "boot guard not active"

mountpoint -q /boot && say 1 "boot partition mounted" \
                    || say 0 "boot partition not mounted"

[ -f /etc/initcpio/hooks/buffyboard ] && [ -f /etc/mkinitcpio.conf.d/zz-deck-unlock.conf ] \
  && say 1 "keyboard-less unlocking wired up" \
  || say 0 "keyboard-less unlocking missing"

grep -q nau8821 /proc/asound/cards && say 1 "internal sound card detected" \
                                   || say 0 "internal sound card missing"

[ -w /sys/class/backlight/amdgpu_bl0/brightness ] && say 1 "brightness controllable" \
                                                  || say 0 "brightness not controllable"

echo
echo "$ok fine, $fail to check"
[ "$fail" = "0" ]
DECKEOF

sudo chmod 755 /usr/local/bin/deck-health
sudo bash -n /usr/local/bin/deck-health && echo "Syntax OK"
deck-health
```

Run it automatically after every update:

```bash
echo 'omarchy-update() { command omarchy-update "$@"; echo; deck-health; }' >> ~/.bashrc
```

A **shell function** rather than a pacman hook: a hook would run as root, and the
checks on the user service and the migration marker would come up empty.

> **The limit of this approach:** it only fires when you call `omarchy-update`
> from a terminal. If you update through **Omarchy's menu**, the update bypasses
> the shell function and `deck-health` never runs. If the menu is your usual
> route, run it by hand afterwards — or get into the habit of the terminal.

---

## Pitfalls

The four that genuinely cost time — plus a few smaller ones.

### Never install `jupiter-legacy-support`

The package looks harmless: 16 KB, four ordinary dependencies. But it contains
`/etc/systemd/system/boot.mount` as a symlink to `/dev/null` and thereby **masks
the mounting of the boot partition**. Correct on SteamOS, fatal on Arch: every
kernel update then fails while generating the image, with an error message buried
at the end of a long package transcript.

It also brings `HandlePowerKey=ignore` (right in itself, see Phase 3 — but
unnoticed collateral here), `KillUserProcesses=True`, a `holo-install-grub.service`
on a system that boots with Limine, and a GRUB installer under `/usr/bin`.

It does **not** arrive as a dependency, neither through Steam nor through
gamescope — it has to be installed explicitly. If it shows up on your system,
somebody put it there on purpose. Then remove it:
`sudo pacman -Rns jupiter-legacy-support`, and check `/boot`.

**Lesson:** with third-party packages, don't judge by size and dependencies.
Judge by `pacman -Fl` — that is, by which **configuration files** they drop.

### The migration block needs the exact path

`~/.local/state/omarchy/migrations/1789325478.sh`

Not `share`, not `/var/lib`, and the `.sh` suffix is part of it. Anything else
has no effect, and you find out once your kernel has been swapped.

### `/boot` must be mounted

For every `mkinitcpio`, every kernel update, every `limine-mkinitcpio`. The tools
do not mount it themselves.

### The right package for sound

The SOF firmware for the Vangogh chip exists **only** in
`jupiter-main/steamdeck-dsp`. Arch's `sof-firmware` does not have it. Without it
the internal sound card never appears at all — `aplay -l` shows HDMI only, and
PipeWire reports `auto_null`.

### Smaller ones

- **`steamos-update` must be absent or must fail.** If it returns successfully,
  the Deck reboots endlessly in game mode. On an Arch system it is normally not
  there at all — the alias only arrives with `jupiter-legacy-support`, which
  after the section above you will not be installing anyway. Check with
  `command -v steamos-update || echo "not present - good"`.
- **`holo-session-select`** is the new name of `steamos-session-select`. The
  compatibility symlink comes from `steamos-alias`.
- **unl0kr's install hook** does not find the Deck's touchscreen.
- **PipeWire does not search `/run/pipewire`.** Valve's `pipewire-sysconf`
  installs the hardware profile there; on Arch it has to go to `/etc`.
- **Arch's `brightnessctl` ships no udev rule.**
- **The Deck reports two power button devices.** Any toggling key binding on it
  fires twice and cancels itself out.

---

## Day to day

| Task | |
|---|---|
| After every update | `deck-health` (runs automatically) |
| Into game mode | Omarchy menu → System → Game Mode (touch works) |
| Back | Steam button → Power → Switch to Desktop |
| Update gamescope | `sudo pacman -S jupiter-main/gamescope` |
| On the road | shut down rather than suspend, see below |
| When nothing works | Limine menu → Snapshots → top entry |

---

## Emergencies and getting back to SteamOS

**The new kernel will not boot.** Hold the power button for ten seconds, then
switch on again. Pick from the Limine menu with the D-pad, confirm with
**Enter** or the **A button**:

- `linux` — the Arch kernel. It stays installed on purpose; it knows the Deck
  hardware less well but boots reliably.
- `linux-neptune-618-fallback` — the same kernel with an initramfs that carries
  every driver. The right choice when you have wrecked the image in Phase 7.
- **Snapshots** → top entry — the state before your last
  `omarchy-snapshot create`, with its own kernel image and its own initramfs.

**The Limine menu does not come up at all.** Then the EFI partition is
affected. Boot from the Omarchy stick, mount the decrypted root filesystem and
`/boot` in the live system, `arch-chroot` into it and run `limine-mkinitcpio`
again.

**The on-screen keyboard does not appear at the unlock prompt.** No snapshot
helps here, because choosing one would require unlocking first. That is exactly
why 7.1 says to keep a USB keyboard attached: unlock with it, then repair at
your leisure.

**All the way back to SteamOS.** Write Valve's recovery image to a stick, boot
it, choose "Reimage Steam Deck". That wipes the SSD completely and restores
factory condition — nothing of Omarchy or the encryption survives. Search term
at Valve: "Steam Deck Recovery Instructions".

> **Make that stick before you start Phase 1.** You need a recovery stick
> precisely when the device will not boot — and by then it is too late to
> download one.

---

## Still open

**Lock screen without a keyboard.** Lock the screen on the desktop, then take
the Deck off the dock, and you are locked out for good: Omarchy's lock screen is
a Quickshell plugin with no on-screen keyboard. The touch keyboard from Phase 7
lives in the initramfs and only ever helps at the LUKS prompt, i.e. on a cold
boot.

In practice this bites less often than it sounds:

- **There is no idle timer.** `hypridle` does not run on Omarchy, so the screen
  never locks itself.
- The only automatic lock is `omarchy-sleep-lock.service`. Through
  `omarchy-system-sleep-monitor` it listens for `PrepareForSleep` and locks
  **before suspend** — unconditionally, unlike the power menu, which checks
  `omarchy-toggle-enabled suspend-off` first.
- It is a **user** unit with `PartOf=graphical-session.target` and
  `ConditionEnvironment=WAYLAND_DISPLAY`, so it is tied to the Hyprland session.
  That session is not running in game mode, so the lock should not fire there.
  We did not measure it.

At a desk there is a keyboard on the dock anyway, and on the road shutting down
beats suspending — that way you get the LUKS prompt with the touch keyboard. If
that is too fragile for you, there are three ways out:

1. **Turn the pre-sleep lock off entirely.** Survives updates.

   ```bash
   systemctl --user mask --now omarchy-sleep-lock.service
   ```

2. **Lock conditionally.** Mask the unit and replace it with your own monitor
   that checks for a real keyboard before locking. Undocked the Deck then sleeps
   unlocked, docked it behaves as before. Small, but you first have to work out
   what the Deck reports internally as a keyboard — the two power button devices
   join in.

3. **Build a keyboard into the lock screen.** `LockView.qml` under
   `/usr/share/omarchy/shell/plugins/lock/` exposes `submitPassword(string)`,
   `passwordTextEdited(string)`, `forcePasswordFocus()` and `clearPassword()`, so
   an on-screen keyboard could feed it from the inside. The path is owned by the
   package and gets overwritten on updates; you would also need a way to shadow
   plugins from the user side.

We did none of these and live with option zero.

**Wi-Fi and band selection.** Left to choose freely, NetworkManager will try to
move the Deck onto a 5 GHz network even when it is far weaker than the 2.4 GHz
one next to it. Authentication times out, it falls back, and repeats every 30
seconds. You can spot it with:

```bash
nmcli -f IN-USE,BSSID,SIGNAL,CHAN device wifi list
sudo dmesg | grep -i "by local choice"
```

If the in-use entry sits below 50, that is your cause. Remedy: pin the band to
2.4 GHz in the network widget, or
`nmcli connection modify '<SSID>' 802-11-wireless.band bg`.

802.11r (fast roaming) and the WPA2/WPA3 transitional mode are probably involved
too — the `ath11k` driver has known trouble with both. Both can be turned off on
a typical home router, but that affects every device on it.

**Rotation in unl0kr.** buffybox's source contains the rotation logic
(`lv_display_set_rotation`) in buffyboard. Back-porting it to unl0kr looks
tractable — you would get a full unlock screen instead of prompt-plus-keyboard.
We did not need it.

**Accented characters and symbols in the passphrase.** Solvable by building
buffyboard with a matching layout and setting the console keymap to match. Not
attempted.

---

## Thanks

To the **postmarketOS** project for buffybox, to **Valve** for keeping the Deck
packages open and individually installable, and to the **Omarchy** developers for
an Arch distribution that can be taken this far apart without falling over.

Written across two evenings in October 2026, between a human with a Steam Deck
and Claude. Several dead ends included — the interesting ones are recorded above
under "Pitfalls".
