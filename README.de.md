# Omarchy auf dem Steam Deck OLED

Ein verschlüsseltes Arbeits-Linux auf dem Steam Deck, das sich ohne Tastatur
aufschließen lässt und auf Knopfdruck in Valves Spielmodus wechselt.

Kein Dual-Boot. Ein System, zwei Sitzungen.

*[English version](README.md)*

**Das hier ist ein volles Omarchy, kein Fork und kein abgespecktes Abbild.**
Omarchy 4 wird mit seinem eigenen Installationsmedium installiert und bezieht
seine Updates weiterhin über `omarchy-update`. Valves Kernel, Lüftersteuerung
und Spielmodus-Pakete kommen einzeln und **namentlich** aus Valves Paketquellen
dazu — es wird nichts entfernt, nichts durch einen Eigenbau ersetzt, und kein
vorgefertigtes Abbild ist im Spiel. Genau deshalb beginnt Phase 2 damit, eine
Migration von Omarchy zu blockieren: Weil wir auf Upstream bleiben, müssen wir
mit dem leben, was Upstream tut.

Das Verzeichnis [`files/`](files/) spiegelt die fertige Konfiguration an den
Pfaden, an die sie gehört — so kannst du eine Datei kopieren, statt sie
abzutippen. **Maßgeblich ist die Anleitung unten**; die Dateien sind aus ihr
herausgezogen. Die Skripte dort protokollieren auf Englisch, die Listings in
dieser Fassung zeigen die deutsche Variante — inhaltlich sind sie gleich.

---

## Was dabei herauskommt

| | |
|---|---|
| Basis | Omarchy 4 (Arch + Hyprland), LUKS-verschlüsselt, btrfs mit Snapshots |
| Kernel | Valves `linux-neptune-618` — Lüftersteuerung, Akkugrenze, TDP |
| Spielmodus | Valves echte gamescope-Session mit Steam, Wechsel in beide Richtungen |
| Entsperren | Bildschirmtastatur auf dem Touchscreen, korrekt gedreht |
| Audio | interne Lautsprecher, Kopfhörer, Mikrofon mit Valves Rauschunterdrückung |
| Absicherung | Wächter gegen Update-Regressionen, Prüfprogramm `deck-health` |

Getestet am **10. Oktober 2026** auf einem **Steam Deck OLED (Galileo)**,
BIOS F7G0114, mit Omarchy 4.0.4 und `linux-neptune-618` 6.18.53.valve1.

Für das LCD-Modell (Jupiter) sollte fast alles gelten; die Gerätenamen und
die Lüfterkurve unterscheiden sich, die Logik nicht.

---

## Bevor du anfängst

**Die interne SSD wird vollständig gelöscht.** SteamOS ist danach weg,
inklusive aller installierten Spiele. Plane die Neuinstallation ein.

Du brauchst:

- zwei USB-Sticks (Omarchy-Installationsmedium, SteamOS-Wiederherstellung)
- ein USB-C-Dock mit Tastatur und möglichst einem Monitor
- Netzteil
- eine Stunde für das Grundsystem, nochmal zwei für den Rest

Das SteamOS-Wiederherstellungsabbild ist dein Rückweg zum Auslieferungszustand.
Es gibt es bei Valve unter *help.steampowered.com*, Suchbegriff
„Steam Deck Recovery Instructions".

**Was dieses Dokument voraussetzt.** Dass du dich auf einer Kommandozeile
zu Hause fühlst: Dateien anlegen, Ausgaben lesen, erkennen, wann ein Befehl
etwas anderes getan hat als erwartet. Es erklärt Linux nicht, sondern dieses
eine Gerät. Wer gerade erst anfängt, sollte es nicht ausführen — die Schritte
in Phase 2 und 7 machen das Deck unbootbar, wenn sie halb verstanden
ausgeführt werden.

Das Installationsmedium erstellst du nach **Omarchys eigener Anleitung**
(omarchy.org) — Prüfsumme und Schreibwerkzeug stehen dort und ändern sich ohne
unser Zutun.

**Lies vorher zwei Abschnitte:** „Fallen" am Ende (vier davon kosten dich
sonst einen halben Tag) und „Notfall und zurück zu SteamOS", damit du den
Rückweg kennst, bevor du ihn brauchst.

---

## Phase 0 — Bestandsaufnahme

Auf dem noch unangetasteten SteamOS, im Desktopmodus:

```bash
cat /sys/class/dmi/id/sys_vendor /sys/class/dmi/id/board_name
uname -r
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

`board_name` muss `Galileo` (OLED) oder `Jupiter` (LCD) sein. Merk dir den
laufenden Kernel — das ist der Neptune-Kernel, den du später installierst.

SteamOS vorher einmal vollständig aktualisieren. Das bringt auch die
Controller- und Dock-Firmware auf Stand, und die bleibt erhalten.

---

## Phase 1 — Omarchy verschlüsselt installieren

Omarchy-ISO von `iso.omarchy.org` holen, Prüfsumme vergleichen, auf den Stick
schreiben.

Deck **vollständig herunterfahren** (kein Neustart). Dann **Leiser-Taste
gedrückt halten und kurz Power drücken**, Leiser halten bis der Boot Manager
erscheint. Stick auswählen.

Im Installer:

- **Full disk install** auf die interne NVMe
- **Verschlüsselung annehmen** — nicht mit Ctrl+C abwählen
- Hostname und Benutzer nach Wunsch

> **Die Verschlüsselung ist nicht optional, wenn du Autologin willst.**
> Omarchys Installer schreibt `autologin.conf` nur, wenn verschlüsselt wird.
> Ohne Autologin landest du bei jedem Start im Anmeldebildschirm — auf einem
> Gerät ohne Tastatur ein Ärgernis.

### Die Passphrase

Sie muss später über eine **Bildschirmtastatur mit US-Belegung** eingegeben
werden. Halte dich an:

- **erlaubt:** Kleinbuchstaben `a` bis `x`, Ziffern, Leerzeichen
- **verboten:** `y`, `z`, Umlaute, Sonderzeichen

Lang und banal schlägt kurz und kryptisch. Vier gewöhnliche Wörter mit
Leerzeichen sind sicherer als acht Zeichen Buchstabensalat — und auf einem
Touchscreen ungleich angenehmer.

Nach dem ersten Start: **noch kein `omarchy-update`.** Erst Phase 2.

---

## Phase 2 — Valves Kernel und die Boot-Reihenfolge

### 2.1 Den Migrationsriegel setzen

Omarchy bringt eine Migration mit, die `linux-omarchy` installiert und die
Boot-Reihenfolge auf diesen Kernel umschreibt. Das würde den Neptune-Kernel
entthronen. `omarchy-migrate` überspringt jede Migration, für die eine
Markierungsdatei existiert — also legen wir sie vorsorglich an:

```bash
mkdir -p ~/.local/state/omarchy/migrations
touch ~/.local/state/omarchy/migrations/1789325478.sh
ls -1 ~/.local/state/omarchy/migrations | wc -l
```

Das `mkdir -p` ist nötig, weil das Verzeichnis auf einem frischen System noch
gar nicht existiert. Die Zahl am Ende ist nur eine Quittung — merk sie dir;
nach dem nächsten Update muss sie um mindestens eins gestiegen sein.

> **Zwei Details, an denen es sonst scheitert:** Das Verzeichnis heißt
> `.local/state`, nicht `.local/share` und nicht `/var/lib/omarchy`.
> Und die Markierung trägt die Endung **`.sh`**.

Prüfen, ob die Migration auf deinem Stand überhaupt schon existiert:

```bash
ls -1 /usr/share/omarchy/migrations | grep 1789325478
```

Findet sie sich nicht, kommt sie mit einem späteren Update — der Riegel wirkt
trotzdem.

### 2.2 Jetzt aktualisieren

```bash
sudo omarchy-snapshot create
omarchy-update
grep BOOT_ORDER /etc/default/limine
```

Der letzte Befehl darf **nichts** ausgeben. Steht dort eine Zeile mit
`linux-omarchy`, hat der Riegel nicht gegriffen.

### 2.3 Valves Paketquellen einbinden

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

> **`Usage = Sync Search Install` ist der wichtigste Teil dieser Datei.**
> Es fehlt bewusst `Upgrade`. Dadurch darf pacman aus Valves Quellen nur
> installieren, was du **namentlich** verlangst, und fasst sie bei
> Systemaktualisierungen nie an. Ohne diese Zeile zieht dir das nächste
> `omarchy-update` SteamOS' Mesa und Konsorten ins Arch-System.

`SigLevel = Optional TrustAll` ist ein Henne-Ei-Problem: Valves Schlüsselbund
liegt in Valves Repo, und ohne den Schlüsselbund kann pacman die Signaturen
nicht prüfen. Solange die Zeile so dasteht, **akzeptiert pacman aus diesen
beiden Quellen alles, auch Untergeschobenes**. Deshalb dauert der Zustand
genau eine Paketinstallation: Der `sed`-Befehl oben setzt ihn sofort wieder
auf `Required DatabaseOptional` zurück. Prüf das nach:

```bash
grep -A2 'jupiter-main\]' /etc/pacman.conf
```

Steht dort noch `Optional TrustAll`, ist der `sed` nicht durchgelaufen — nicht
weitermachen, bevor das stimmt. Die Sicherungskopie liegt unter
`/etc/pacman.conf.bak`.

### 2.4 Kernel installieren

```bash
pacman -Si jupiter-main/linux-neptune-618
sudo pacman -S jupiter-main/linux-neptune-618 jupiter-main/linux-neptune-618-headers
```

> **Nimm `linux-neptune-618`, nicht `linux-neptune-72`.** Beide liegen in
> `jupiter-main`, aber 618 ist, was Valve auf dem Deck tatsächlich ausliefert.
> Die 7.2-Reihe ist Vorbereitung für ein kommendes SteamOS. „Läuft
> nachweislich auf dieser Hardware" schlägt „ist neuer".
>
> `jupiter-staging` ist bei beiden Reihen **älter** als `jupiter-main`. Nicht
> von der Bezeichnung täuschen lassen.

### 2.5 Boot-Reihenfolge festlegen

Erst nachsehen, ob schon eine `BOOT_ORDER` dasteht:

```bash
grep -n BOOT_ORDER /etc/default/limine
```

Kommt nichts, ist alles gut. Kommt eine Zeile, lösch sie vorher — `tee -a`
hängt sonst eine zweite an. Doppelte Zeilen sind nicht tödlich, die **letzte
gewinnt**, aber man sucht sich später dumm und dämlich:

```bash
sudo sed -i '/^[[:space:]]*BOOT_ORDER[[:space:]]*=/d' /etc/default/limine
```

Dann schreiben:

```bash
sudo tee -a /etc/default/limine > /dev/null << 'EOF'
BOOT_ORDER="linux-neptune-618, linux, *, *fallback, Snapshots"
EOF

cat /etc/default/limine
sudo mount /boot
sudo limine-mkinitcpio
```

Meldet `mount` „already mounted", ist das in Ordnung — dann war `/boot` schon
eingehängt. Jede andere Fehlermeldung ist keine.

Die Zeile muss am Ende exakt so dastehen, mit beiden Sternchen:

```
BOOT_ORDER="linux-neptune-618, linux, *, *fallback, Snapshots"
```

> **`/etc/default/limine` hat Vorrang vor `/etc/limine-entry-tool.conf`.**
> In letzterer steht ebenfalls eine `BOOT_ORDER`, die aber überschrieben wird.
> Bewachen musst du `/etc/default/limine` — das ist auch die Datei, die
> Omarchys Migration manipuliert.

Der Arch-Kernel bleibt als zweiter Eintrag installiert. Das ist dein
Rettungsboot: Bootet Neptune nicht, wählst du ihn im Limine-Menü mit dem
D-Pad aus und bestätigst mit **Enter** beziehungsweise dem **A-Knopf**.

### 2.6 Neu starten und prüfen

```bash
uname -r
grep -H . /sys/class/hwmon/hwmon?/name
find /sys/devices -name max_battery_charge_level
find /sys/devices -name fan1_target
```

Erwartet wird ungefähr das hier — die Nummern dürfen bei dir andere sein:

```
6.18.53-valve1-1-neptune-618
/sys/class/hwmon/hwmon0/name:amdgpu
/sys/class/hwmon/hwmon5/name:steamdeck_hwmon
/sys/devices/.../hwmon/hwmon5/max_battery_charge_level
/sys/devices/.../hwmon/hwmon5/fan1_target
```

`steamdeck_hwmon` muss auftauchen und beide Dateien müssen existieren. Das ist
der Beweis, dass du Valves Kernel hast und nicht nur hoffst. Fehlt
`steamdeck_hwmon`, läuft noch der alte Kernel — dann stimmt die
Boot-Reihenfolge nicht, und du gehst zurück zu 2.5.

---

## Der Boot-Wächter

Die Boot-Reihenfolge aus 2.5 ist die Einstellung, die ein Update am ehesten
wieder zerlegt — leise, und du merkst es erst beim nächsten Start mit dem
falschen Kernel. Deshalb richten wir den Wächter **jetzt** ein, direkt hinter
der Phase, die er absichert, und nicht erst am Ende.

### `/usr/local/bin/deck-boot-guard`

Stellt die `BOOT_ORDER` wieder her, sobald jemand sie ändert.

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
    umount /boot || logger -t deck-boot-guard "Warnung: /boot nicht ausgehaengt"
  fi
}
trap cleanup EXIT

current=$(grep -E "$PATTERN" "$CONF" | tail -n 1 || true)
if [ "$current" = "$WANT" ]; then
  exit 0
fi

logger -t deck-boot-guard "BOOT_ORDER falsch oder fehlt, wird repariert"
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
logger -t deck-boot-guard "BOOT_ORDER wiederhergestellt"
DECKEOF

sudo chmod 755 /usr/local/bin/deck-boot-guard
sudo bash -n /usr/local/bin/deck-boot-guard && echo "Syntax OK"
```

> **Das Einhängen von `/boot` ist nicht optional.** Die Limine-Werkzeuge
> hängen die Partition **nicht selbst** ein, sie verlangen, dass sie schon
> eingehängt ist, und brechen sonst mit
> `Boot path '/boot' is not a mounted FAT32 boot partition` ab. Interaktiv
> merkt man das nicht, aus einem Dienst heraus schon.

Dazu zwei Einheiten:

```bash
sudo tee /etc/systemd/system/deck-boot-guard.service > /dev/null << 'DECKEOF'
[Unit]
Description=Deck Boot-Reihenfolge absichern
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
Description=Ueberwacht /etc/default/limine

[Path]
PathChanged=/etc/default/limine
Unit=deck-boot-guard.service

[Install]
WantedBy=multi-user.target
DECKEOF
```

Aktivieren:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now deck-boot-guard.path
sudo systemctl enable deck-boot-guard.service
```

Und jetzt kaputtmachen, um zu sehen, dass er hält:

```bash
echo 'BOOT_ORDER="linux-omarchy, linux"' | sudo tee -a /etc/default/limine
sleep 5
grep BOOT_ORDER /etc/default/limine
journalctl -t deck-boot-guard -n 10 --no-pager
mountpoint /boot
```

Die `.path`-Einheit reagiert sofort; die fünf Sekunden sind nur Luft für
`limine-mkinitcpio`, das das Bootmenü neu schreibt. Danach muss wieder genau
eine `BOOT_ORDER`-Zeile dastehen, die richtige, das Journal muss
„BOOT_ORDER wiederhergestellt" melden — und `/boot` muss **nicht** eingehängt
sein, denn der Wächter räumt hinter sich auf.

---

## Phase 3 — Lüfter, Energie, Akku

```bash
sudo pacman -S jupiter-main/jupiter-fan-control holo-main/steamos-manager
sudo systemctl enable --now jupiter-fan-control.service
sudo systemctl enable --now steamos-manager.service
systemctl --user enable --now steamos-manager.service
```

> **Der Benutzerdienst ist nicht optional.** Der System-Dienst stellt nur die
> Root-Schnittstellen bereit (TDP, Lüfter, Ladegrenze). Die Sitzungsverwaltung
> — also das Umschalten zwischen Desktop und Spielmodus — liegt im
> **Benutzer**-Dienst. Valve startet den nur innerhalb der gamescope-Sitzung;
> du brauchst ihn auch im Desktop.

`jupiter-fan-control` wählt seine Kurve selbst nach `board_name`, für das
OLED-Deck also `galileo-config.yaml`. Zum Gegenrechnen:

Die hwmon-Nummern verschieben sich zwischen Starts, deshalb nie eine fest
hinschreiben, sondern den Sensor über seinen Treibernamen suchen:

```bash
DECK=$(for d in /sys/class/hwmon/hwmon*; do
         [ "$(cat "$d/name" 2>/dev/null)" = "steamdeck_hwmon" ] && echo "$d"
       done)
echo "$DECK"
cat "$DECK/fan1_input" "$DECK/fan1_target"
```

`fan1_target` ist, was die Regelung will, `fan1_input`, was der Lüfter
tatsächlich dreht. Liegen die beiden weit auseinander, stimmt etwas nicht.

Zur Einordnung der Zahlen: Die Kurve in `galileo-config.yaml` ist quadratisch
über der CPU-Temperatur, ihr Stützpunkt liegt bei `(60 °C, 2400 U/min)`. Das
ist aber **nicht** der Wert, den du dort abliest — die Regelung zieht
zusätzlich die aktuelle APU-Leistung heran und bleibt bei geringer Last
darunter. Rund 2000 U/min bei 60 °C im Leerlauf sind deshalb normal und kein
Defekt.

**Ladegrenze** — schont den Akku erheblich, wenn das Gerät überwiegend am Dock
hängt:

```bash
echo 'SUBSYSTEM=="hwmon", ATTR{name}=="steamdeck_hwmon", ATTR{max_battery_charge_level}="80"' | sudo tee /etc/udev/rules.d/99-deck-charge-limit.rules
```

Die Regel matcht über den Treibernamen, nicht über eine hwmon-Nummer — aus dem
Grund von eben. Sie wirkt aber **nicht sofort**: udev wendet neue Regeln nur
auf neu auftauchende Geräte an. Also nachhelfen und nachsehen:

```bash
sudo udevadm control --reload
sudo udevadm trigger -s hwmon
cat "$DECK/max_battery_charge_level"
```

Dort muss `80` stehen. Vorher steht `0`, das bedeutet „keine Grenze".

**Einschaltknopf.** Omarchy belegt ihn bereits selbst und öffnet damit sein
Energiemenü. logind muss sich deshalb heraushalten — sonst greifen beide zu:

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/10-deck-powerkey.conf > /dev/null << 'EOF'
[Login]
HandlePowerKey=ignore
HandlePowerKeyLongPress=poweroff
EOF
sudo systemctl reload systemd-logind
```

Langes Drücken schaltet damit sauber aus, bevor die Hardware nach zehn
Sekunden hart abschaltet.

Das Menü blitzt danach allerdings nur kurz auf und verschwindet wieder. Grund:
Das Deck meldet **zwei** Einschaltknopf-Geräte (`power-button` und
`power-button-1`, beide in `hyprctl devices` sichtbar). Omarchys
Standardbelegung ist ein Umschalter — der erste Tastendruck öffnet, der zweite
schließt sofort wieder. Abhilfe ist die Verbform `summon` statt `toggle`:

```bash
tee -a ~/.config/hypr/bindings.lua > /dev/null << 'EOF'

hl.unbind("XF86PowerOff")
o.bind("XF86PowerOff", "Power menu", "omarchy-menu summon system", { locked = true })
EOF
hyprctl reload
```

Ein zweiter Druck schließt das Menü danach nicht mehr — dafür `Esc` oder
daneben tippen.

Im Spielmodus greift all das nicht — dort ist Omarchy nicht da. Dafür braucht
es einen eigenen Dienst, siehe Phase 4.6.

**Danach sollte gelten:** Der Lüfter dreht hörbar lastabhängig,
`max_battery_charge_level` steht auf `80`, und ein kurzer Druck auf den
Einschaltknopf öffnet Omarchys Energiemenü, ohne dass das Gerät suspendiert.
Stimmt eines davon nicht, geh zurück, bevor du Phase 4 anfängst.

---

## Phase 4 — Der Spielmodus

### 4.1 gamescope vor Updates schützen

Arch hat ein **eigenes** Paket namens `gamescope`. Valves Variante enthält
zusätzlich die komplette Sitzung. Sobald Arch eine höhere Versionsnummer
ausliefert, ersetzt das nächste Systemupdate Valves Paket durch Archs — und
der Spielmodus verschwindet lautlos.

```bash
grep -q '^IgnorePkg = gamescope' /etc/pacman.conf \
  || sudo sed -i '/^\[options\]/a IgnorePkg = gamescope' /etc/pacman.conf
grep -n -A4 '^\[options\]' /etc/pacman.conf
```

`IgnorePkg` **muss** im Abschnitt `[options]` stehen — deshalb das `sed`, das
die Zeile direkt dahinter einfügt, statt sie anzuhängen. Ans Dateiende
geschrieben landet sie im letzten Repo-Block und wirkt nicht. Die Prüfung
davor macht den Befehl wiederholbar.

Künftig aktualisierst du gamescope von Hand:

```bash
sudo pacman -S jupiter-main/gamescope
```

Weil das Paket auf der Ignorierliste steht, fragt pacman dabei
`gamescope is in IgnorePkg. Install anyway? [Y/n]` — hier ist **`y`** die
richtige Antwort. Genau diese Rückfrage ist der Sinn der Übung: Sie taucht
auch dann auf, wenn ein Systemupdate das Paket anfassen will, und dort
antwortest du **`n`**.

### 4.2 Installieren

```bash
sudo pacman -S jupiter-main/gamescope holo-main/holo-session-selection jupiter-main/steamos-alias
sudo pacman -S jupiter-main/steam-jupiter-stable jupiter-main/steam_notif_daemon
```

Bei `steam-jupiter-stable` fragt pacman, welcher 32-Bit-Vulkan-Treiber
verwendet werden soll. Es listet die Anbieter nummeriert auf und fragt
`Enter a number (default=1)`.

**Tipp die Nummer von `lib32-vulkan-radeon` aus `multilib` ein.** Nicht die
gleichnamige Variante aus `jupiter-main`, und auf keinen Fall einfach Enter —
die Vorgabe `1` ist der NVIDIA-Treiber.

Danach eine Kontrolle, die dir später einen halben Tag spart:

```bash
pacman -Qq jupiter-legacy-support 2>/dev/null \
  && echo "ACHTUNG: installiert, siehe Fallen" \
  || echo "nicht installiert - gut so"
```

Das Paket darf hier nicht auftauchen. Warum, steht unter „Fallen".

`steamos-alias` legt den Kompatibilitäts-Symlink
`/usr/bin/steamos-session-select → holo-session-select` an, den Steam ruft.

### 4.3 Steam einmal im Desktop starten

```bash
steam
```

Anmelden, durchlaufen lassen, beenden. Das trennt die Fehlersuche: Wenn später
etwas klemmt, weißt du, dass es nicht an Steams Erstinstallation liegt.

### 4.4 Sitzungsverwaltung einrichten

```bash
systemctl --user restart steamos-manager.service
steamosctl get-valid-desktop-sessions
steamosctl set-default-desktop-session omarchy.desktop
steamosctl set-default-login-mode desktop
```

> **Der Neustart des Benutzerdienstes ist nötig**, wenn er vor gamescope
> gestartet wurde: Er prüft beim Start, ob eine Spielmodus-Sitzung existiert,
> und blendet die Schnittstelle `SessionManagement1` sonst aus. Symptom:
> `Unknown interface 'com.steampowered.SteamOSManager1.SessionManagement1'`.

### 4.5 Umschalten

```bash
steamosctl switch-to-game-mode
```

Zurück über **Steam-Knopf → Ein/Aus → Zum Desktop wechseln**.

Der Wechsel in den Spielmodus gilt nur für diese Sitzung: SDDM löscht die
temporäre Anmeldedatei bei jedem Start. Bleibt der Spielmodus hängen, bringt
dich ein Neustart zurück in den Desktop. Das ist ein echtes Sicherheitsnetz,
kein Zufall.

Für den Alltag ein anklickbarer Starter, der auch ohne Tastatur erreichbar ist:

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

Eine Tastenkombination ist **nicht** zu empfehlen: Ein Fehlgriff beendet die
komplette Desktop-Sitzung samt allen offenen Fenstern.

**Eleganter: der Wechsel direkt im Omarchy-Menü.** Dann liegt der Hinweg dort,
wo auch der Rückweg liegt — beide über ein Menü, beide ohne Tastatur.

Omarchy sieht dafür eine Erweiterungsdatei vor, die Updates übersteht:

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

Das Skript legt Verzeichnis und Datei an, falls es sie noch nicht gibt, und
setzt das Komma nur dann, wenn schon ein Eintrag davorsteht — sonst entstünde
ungültiges JSON. Zweimal ausgeführt ändert es nichts.

Die Kennung `system.gamemode` hängt den Eintrag unter **System** — also in genau
das Menü, das der Einschaltknopf öffnet. Ohne Punkt (`gamemode`) landet er im
Hauptmenü.

`when` blendet die Zeile aus, falls `steamosctl` fehlt; dieselbe Technik
benutzt Omarchy bei „Hibernate". Das Symbol wird aus seinem Codepunkt gebaut,
weil Nerd-Font-Zeichen aus dem privaten Unicode-Bereich den Weg durch
Zwischenablagen selten unbeschadet überstehen.

### 4.6 Einschaltknopf im Spielmodus

Im Desktop fängt Omarchy den Knopf ab (Phase 3). Im Spielmodus gibt es Omarchy
nicht — dort braucht es Valves eigenen Dienst, sonst passiert beim Drücken
überhaupt nichts:

```bash
sudo pacman -S holo-main/steamos-powerbuttond
```

Er ist ein **Benutzerdienst** mit `Requisite=gamescope-session.service`, startet
also ausschließlich innerhalb der Spielsitzung und lässt den Desktop in Ruhe.
Die mitgelieferte Hardware-Datenbank markiert nur Valve-Geräte und schreibt
keine Tastencodes um — die Legion-Go- und AYA-NEO-Einträge darin betreffen
andere Hardware.

Ob er aktiviert ist, prüfst du **innerhalb der Spielsitzung** nicht so leicht
— also vorher im Desktop:

```bash
systemctl --user is-enabled steamos-powerbuttond.service
```

Bei uns stand dort nach der Installation bereits `enabled`. Kommt
`disabled`, hol es nach:

```bash
systemctl --user enable steamos-powerbuttond.service
```

Starten lässt er sich im Desktop nicht — `Requisite=gamescope-session.service`
verhindert das mit Absicht. Das ist kein Fehler.

Die udev-Markierung wird erst nach einem Neustart wirksam.

**Danach sollte gelten:** `steamosctl switch-to-game-mode` bringt dich in die
Steam-Oberfläche, der Steam-Knopf bringt dich über **Ein/Aus → Zum Desktop
wechseln** zurück, und ein Neustart landet immer im Desktop. Kommst du nicht
zurück, ist der Neustart der Ausweg — das ist so gebaut.

---

## Phase 5 — Audio

Ohne diesen Schritt hast du am Dock Ton über den Monitor und abgedockt
**gar nichts**. Die Lautstärkeanzeige meldet dann `auto_null`.

Ursache: Der Audiochip des Decks braucht SOF-Firmware und eine
Topologiebeschreibung, die es **nur bei Valve** gibt. Arch' `sof-firmware`
enthält sie nicht.

```bash
sudo pacman -S jupiter-main/steamdeck-dsp
sudo modprobe -r snd_sof_amd_vangogh
sudo modprobe snd_sof_amd_vangogh
cat /proc/asound/cards
```

Es muss eine zweite Karte erscheinen:

```
1 [sofnau8821max]: sof-nau8821-max - sof-nau8821-max
                   Valve-Galileo-1
```

Die Lautsprecherentzerrung läuft danach **im DSP des Chips**, nicht in
PipeWire — deshalb taucht sie nicht als Filter auf. Das ist korrekt.

Valves Mikrofonverarbeitung (Rauschunterdrückung) dagegen läuft in PipeWire,
und die Konfiguration landet an einem Ort, den nur SteamOS' angepasstes
PipeWire durchsucht:

```bash
sudo mkdir -p /etc/pipewire/filter-chain.conf.d
sudo cp /usr/share/pipewire/hardware-profiles/valve-galileo/filter-chain.conf.d/filter-chain.conf \
        /usr/share/pipewire/hardware-profiles/valve-galileo/filter-chain.conf.d/context-properties.conf \
        /etc/pipewire/filter-chain.conf.d/
systemctl --user restart filter-chain
wpctl status
```

**Danach sollte gelten:** `cat /proc/asound/cards` zeigt zwei Karten,
`wpctl status` listet Lautsprecher, Kopfhörer und Mikrofon, und abgedockt
kommt Ton aus dem Gerät selbst. Steht in der Lautstärkeanzeige weiterhin
`auto_null`, fehlt die Firmware — nicht weitermachen, sondern oben nachsehen.

---

## Phase 6 — Helligkeit

Auf Arch darf standardmäßig niemand außer root die Hintergrundbeleuchtung
verstellen. Im Desktop fällt das nicht auf, weil Omarchy den Umweg über den
Anmeldedienst nimmt. gamescope schreibt direkt in die Systemdatei und
scheitert still — der Regler bewegt sich, das Panel nicht.

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

Danach gehört die Datei der Gruppe `video` mit `rw-rw-r--`. Die
Gruppenmitgliedschaft greift erst nach erneuter Anmeldung.

Arch' `brightnessctl` liefert **keine** udev-Regel mit, anders als auf manchen
anderen Distributionen. Deshalb schreiben wir sie selbst.

Zum Testen **abdocken**: Angedockt ist der aktive Bildschirm der externe
Monitor, und dessen Helligkeit kann Linux nicht regeln. Ein ausgegrauter
Regler ist dort die richtige Antwort, kein Fehler.

**Danach sollte gelten:** Abgedockt verstellt der Regler im Spielmodus die
Helligkeit tatsächlich, und `brightnessctl set 50%` läuft im Desktop ohne
`sudo` durch.

---

## Phase 7 — Entsperren ohne Tastatur

Der heikelste Teil. Er verändert das initramfs — also genau das Stück
Software, das deine verschlüsselte Platte aufschließt.

### Warum buffyboard und nicht unl0kr

Beide stammen aus demselben Projekt (**buffybox**, postmarketOS).

- **unl0kr** zeichnet ein vollständiges Passwortfeld und **ersetzt** den
  `encrypt`-Hook. Startet es nicht, dreht sich die Entsperrschleife endlos —
  du kommst nicht an dein System.
- **buffyboard** zeichnet nur eine Tastatur und speist Tastendrücke über
  `uinput` ein. Der normale Passwort-Prompt bleibt bestehen. Startet es nicht,
  tippst du mit einer USB-Tastatur weiter.

Dazu der entscheidende Unterschied: **unl0kr kann nicht drehen.** Die
Softwaredrehung wurde dort wegen Geschwindigkeitsproblemen ausgebaut
(siehe `CHANGELOG.md` im Quelltext). buffyboard hat sie — und übernimmt sogar
automatisch den Wert aus `/sys/class/graphics/fbcon/rotate`.

Das Deck-Panel ist hochkant verbaut. Ohne Drehung liegt alles quer.

### 7.1 Sicherheitsnetz

```bash
sudo omarchy-snapshot create
```

Jeder Snapshot speichert eine **eigene Kopie des Kernel-Abbilds samt
initramfs** und erscheint im Limine-Menü. Das ist ein vollständiger Rückweg,
der mit deinem neuen initramfs nichts zu tun hat — und mit dem D-Pad
auswählbar.

USB-Tastatur während des gesamten Tests angeschlossen lassen.

### 7.2 Konsolendrehung bestimmen

```bash
echo 1 | sudo tee /sys/class/graphics/fbcon/rotate_all
```

Dann `Strg+Alt+F3` und auf dem **Deck-Bildschirm** nachsehen; mit
`Strg+Alt+F1` beziehungsweise `Strg+Alt+F2` kommst du zurück in die grafische
Sitzung. Für diese Tastenkombinationen brauchst du die USB-Tastatur aus 7.1 —
das ist einer der Gründe, warum sie angeschlossen bleiben soll.

Für das OLED-Deck ist **`1`** richtig (90° im Uhrzeigersinn). Werte `0` bis `3`
durchprobieren, die Änderung wirkt sofort. Beim LCD-Modell (Jupiter) kann ein
anderer Wert richtig sein — probier es aus, statt die `1` zu übernehmen.

### 7.3 buffyboard installieren und gefahrlos testen

```bash
yay -S buffyboard
```

**Teste es, bevor du das initramfs anfasst** — auf einer Textkonsole, wo
nichts kaputtgehen kann:

```bash
# Strg+Alt+F3, anmelden
echo 1 | sudo tee /sys/class/graphics/fbcon/rotate_all
sudo buffyboard
```

Prüfe: Erscheint die Tastatur? Ist sie richtig herum? Reagiert der
Touchscreen? Kommen die **richtigen** Zeichen an?

Beenden mit `Strg+C` auf der USB-Tastatur, zurück in den Desktop mit
`Strg+Alt+F1` oder `Strg+Alt+F2`.

Dieser Test beantwortet in zehn Minuten die drei Fragen, an denen das ganze
Vorhaben hängt. Nutze ihn.

Zum Layout: buffyboard sendet **Tastencodes einer US-Tastatur**. Bei
deutscher Konsolenbelegung sind `y` und `z` vertauscht und Sonderzeichen
liegen falsch. Daher die Einschränkung an die Passphrase aus Phase 1.

### 7.4 initramfs-Einbindung

buffyboard bringt keinen mkinitcpio-Hook mit. Drei Dateien sind nötig.

**Vorher nachsehen, wie dein Entschlüsselungs-Hook heißt:**

```bash
grep '^HOOKS' /etc/mkinitcpio.conf
```

Dort muss `encrypt` stehen. Heißt er bei dir `sd-encrypt` (die
systemd-Variante), dann ersetze `encrypt` in der dritten Datei unten durch
`sd-encrypt` — sonst fügt die Schleife **stillschweigend nichts** ein, der Bau
läuft fehlerfrei durch, und du merkst es erst beim Neustart.

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
Startet buffyboard, eine Bildschirmtastatur auf dem Framebuffer.
HELPEOF
}
DECKEOF
```

> **Die fünf `add_module`-Zeilen sind der Kern.** unl0krs eigener Hook benutzt
> `add_checked_modules '/input/touchscreen/'` — und der Touchscreen des Decks
> liegt nicht dort, sondern bei den HID-Geräten. Verlässt du dich darauf,
> startet die Tastatur und reagiert auf keine Berührung.

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

> **Warum `-r 1`, obwohl buffyboard die Drehung aus fbcon liest?** Weil beides
> zusammenwirkt und der Schalter die Absicherung ist: Sollte die
> Kernel-Befehlszeile aus 7.5 einmal verlorengehen, steht die Tastatur
> trotzdem richtig. Beide Wege setzen denselben Wert, sie widersprechen sich
> nicht. **Beim LCD-Deck** musst du hier denselben Wert eintragen, den du in
> 7.2 ermittelt hast — nicht blind die `1`.

**`/etc/mkinitcpio.conf.d/zz-deck-unlock.conf`**

```bash
sudo tee /etc/mkinitcpio.conf.d/zz-deck-unlock.conf > /dev/null << 'DECKEOF'
# Plymouth aus dem initramfs nehmen und buffyboard vor encrypt einhaengen.
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

Die drei Dateien werden von mkinitcpio eingelesen, nicht ausgeführt — sie
brauchen kein Ausführungsrecht. Kontrolle:

```bash
ls -l /etc/initcpio/install/buffyboard /etc/initcpio/hooks/buffyboard \
      /etc/mkinitcpio.conf.d/zz-deck-unlock.conf
sudo bash -n /etc/mkinitcpio.conf.d/zz-deck-unlock.conf && echo "Syntax OK"
```

> **Plymouth muss raus.** Es belegt den Bildschirm, und buffyboard will die
> Textkonsole verkleinern und sich darunter setzen. Ohne Plymouth zeigt der
> `encrypt`-Hook einen schlichten Textprompt — der respektiert die
> fbcon-Drehung und steht aufrecht.
>
> Die Schleife ersetzt keine fest verdrahtete Hook-Liste, sondern baut sie um.
> Ändert Omarchy seine Hooks, passt sich das an.

### 7.5 Konsolendrehung dauerhaft

```bash
sudo tee -a /etc/default/limine > /dev/null << 'EOF'
KERNEL_CMDLINE[default]+=" fbcon=rotate:1"
EOF
sudo bash -n /etc/default/limine && echo "Syntax OK"
```

### 7.6 Bauen und prüfen

```bash
sudo mount /boot          # "already mounted" ist in Ordnung
sudo limine-mkinitcpio
```

In der Ausgabe muss `[buffyboard]` auftauchen und `[plymouth]` fehlen.

```bash
sudo objcopy -O binary --only-section=.initrd \
  /boot/EFI/Linux/omarchy_linux-neptune-618.efi /tmp/initrd.img
lsinitcpio -l /tmp/initrd.img | awk -F/ '{print $NF}' \
  | grep -E "^(hid-multitouch|i2c-hid|i2c-hid-acpi|amdgpu|uinput)" | sort -u
lsinitcpio -l /tmp/initrd.img | grep -ci plymouth
```

Erwartet: fünf Module, und `0` für Plymouth. `amdgpu` ist nicht optional —
ohne Grafiktreiber gibt es kein `/dev/fb0` und buffyboard hat nichts, worauf
es zeichnen könnte.

Siehst du nur vier, prüf welches fehlt. Bei `uinput` ist das kein Fehler: Es
kann fest in den Kernel gebaut sein statt als Modul vorzuliegen, und dann gibt
es keine Datei, die ins Abbild wandern könnte. Entscheidend ist dann nur, ob
`/dev/uinput` beim Start existiert:

```bash
ls -l /dev/uinput
```

Fehlt dagegen eines der HID-Module, reagiert der Touchscreen später nicht.

### 7.7 Neustart

Oben der Prompt, darunter die Tastatur, beides aufrecht.

Geht etwas schief: Power-Knopf zehn Sekunden, im Limine-Menü mit dem D-Pad zu
**Snapshots** und den obersten Eintrag wählen.

---

## Das Prüfskript `deck-health`

Der Boot-Wächter hinter Phase 2 greift ein. Dieses hier greift **nicht** ein — es misst nur
und sagt dir, was nicht stimmt. Vierzehn Prüfungen, die den tatsächlichen Zustand messen — nicht, ob ein Paket
installiert ist:

1. Neptune-Kernel läuft
2. `BOOT_ORDER` zeigt zuerst darauf
3. Migrationsriegel gesetzt
4. `steamdeck_hwmon` da, Ladegrenze aktiv
5. `jupiter-fan-control` läuft
6. `steamos-manager` (System) läuft
7. `steamos-manager` (Benutzer) läuft
8. gamescope ist Valves Variante
9. gamescope von Updates ausgenommen
10. Boot-Wächter aktiv
11. `/boot` eingehängt
12. Entsperren ohne Tastatur eingerichtet
13. interne Soundkarte erkannt
14. Helligkeit regelbar

```bash
sudo tee /usr/local/bin/deck-health > /dev/null << 'DECKEOF'
#!/usr/bin/env bash

ok=0
fail=0

say() {
  if [ "$1" = "1" ]; then
    echo "  [ OK ]  $2"; ok=$((ok+1))
  else
    echo "  [FEHLT] $2"; fail=$((fail+1))
  fi
}

echo "Deck-Status"; echo

k=$(uname -r)
echo "$k" | grep -q neptune && say 1 "Kernel $k" || say 0 "Kernel $k ist kein Neptune-Kernel"

b=$(grep -E '^[[:space:]]{0,8}BOOT_ORDER' /etc/default/limine 2>/dev/null | tail -n 1)
echo "$b" | grep -q "linux-neptune-618, linux" && say 1 "BOOT_ORDER korrekt" || say 0 "BOOT_ORDER: $b"

[ -f "$HOME/.local/state/omarchy/migrations/1789325478.sh" ] \
  && say 1 "Migrationsriegel gesetzt" || say 0 "Migrationsriegel fehlt"

f=$(find /sys/devices -name max_battery_charge_level -print -quit 2>/dev/null)
[ -n "$f" ] && say 1 "steamdeck_hwmon da, Ladegrenze $(cat "$f") Prozent" \
            || say 0 "steamdeck_hwmon fehlt"

for s in jupiter-fan-control steamos-manager; do
  systemctl is-active --quiet "$s" && say 1 "Systemdienst $s laeuft" \
                                   || say 0 "Systemdienst $s laeuft nicht"
done

systemctl --user is-active --quiet steamos-manager \
  && say 1 "Benutzerdienst steamos-manager laeuft" \
  || say 0 "Benutzerdienst steamos-manager laeuft nicht"

pacman -Qlq gamescope 2>/dev/null | grep -q start-gamescope-session \
  && say 1 "gamescope ist die Valve-Variante" \
  || say 0 "gamescope ist Arch-Variante, Spielmodus fehlt"

grep -q "IgnorePkg = gamescope" /etc/pacman.conf \
  && say 1 "gamescope von Updates ausgenommen" \
  || say 0 "gamescope nicht ausgenommen"

systemctl is-active --quiet deck-boot-guard.path \
  && say 1 "Boot-Waechter aktiv" || say 0 "Boot-Waechter nicht aktiv"

mountpoint -q /boot && say 1 "Boot-Partition eingehaengt" \
                    || say 0 "Boot-Partition nicht eingehaengt"

[ -f /etc/initcpio/hooks/buffyboard ] && [ -f /etc/mkinitcpio.conf.d/zz-deck-unlock.conf ] \
  && say 1 "Entsperren ohne Tastatur eingerichtet" \
  || say 0 "Entsperren ohne Tastatur fehlt"

grep -q nau8821 /proc/asound/cards && say 1 "Interne Soundkarte erkannt" \
                                   || say 0 "Interne Soundkarte fehlt"

[ -w /sys/class/backlight/amdgpu_bl0/brightness ] && say 1 "Helligkeit regelbar" \
                                                  || say 0 "Helligkeit nicht regelbar"

echo
echo "$ok in Ordnung, $fail zu pruefen"
[ "$fail" = "0" ]
DECKEOF

sudo chmod 755 /usr/local/bin/deck-health
sudo bash -n /usr/local/bin/deck-health && echo "Syntax OK"
deck-health
```

Automatisch nach jedem Update:

```bash
echo 'omarchy-update() { command omarchy-update "$@"; echo; deck-health; }' >> ~/.bashrc
```

Als **Shell-Funktion** und nicht als Paket-Hook: Ein Hook liefe als root, und
die Prüfungen auf Benutzerdienst und Migrationsriegel gingen ins Leere.

> **Die Grenze dieser Lösung:** Sie greift nur, wenn du `omarchy-update` im
> Terminal aufrufst. Aktualisierst du über **Omarchys Menü**, läuft das Update
> an der Shell-Funktion vorbei und `deck-health` startet nicht. Wer meistens
> über das Menü aktualisiert, ruft es danach von Hand auf — oder gewöhnt sich
> das Terminal an.

---

## Fallen

Die vier, die wirklich Zeit kosten — und ein paar kleinere.

### `jupiter-legacy-support` niemals installieren

Das Paket sieht harmlos aus: 16 KB, vier gewöhnliche Abhängigkeiten. Es
enthält aber `/etc/systemd/system/boot.mount` als Symlink auf `/dev/null` und
**maskiert damit die Einhängung der Boot-Partition**. Auf SteamOS richtig, auf
Arch fatal: Jedes Kernel-Update scheitert dann beim Erzeugen des Abbilds, und
zwar mit einer Fehlermeldung, die am Ende einer langen Paketausgabe steht.

Es bringt außerdem mit: `HandlePowerKey=ignore` (für sich genommen sogar
richtig, siehe Phase 3 — hier aber unbemerkter Beifang), `KillUserProcesses=True`, ein
`holo-install-grub.service` auf einem System mit Limine, und einen
GRUB-Installer unter `/usr/bin`.

Es kommt **nicht** als Abhängigkeit herein, weder über Steam noch über
gamescope — es muss ausdrücklich installiert werden. Taucht es bei dir auf,
hat es jemand absichtlich getan. Dann entfernen:
`sudo pacman -Rns jupiter-legacy-support` und `/boot` prüfen.

**Lehre:** Bei Fremdpaketen nicht auf Größe und Abhängigkeiten schauen,
sondern auf `pacman -Fl` — also darauf, welche **Konfigurationsdateien** sie
mitbringen.

### Der Migrationsriegel braucht den exakten Pfad

`~/.local/state/omarchy/migrations/1789325478.sh`

Nicht `share`, nicht `/var/lib`, und die Endung `.sh` gehört dazu. Alles
andere wirkt nicht, und du merkst es erst, wenn dein Kernel getauscht wurde.

### `/boot` muss eingehängt sein

Für jedes `mkinitcpio`, jedes Kernel-Update, jeden Aufruf von
`limine-mkinitcpio`. Die Werkzeuge hängen nicht selbst ein.

### Das richtige Paket für den Ton

Die SOF-Firmware für den Vangogh-Chip gibt es **nur** in
`jupiter-main/steamdeck-dsp`. Arch' `sof-firmware` hat sie nicht. Ohne sie
erscheint die interne Soundkarte überhaupt nicht — `aplay -l` zeigt dann nur
HDMI, und PipeWire meldet `auto_null`.

### Kleinere

- **`steamos-update` muss fehlen oder fehlschlagen.** Kehrt es erfolgreich
  zurück, startet das Deck im Spielmodus endlos neu. Auf einem Arch-System ist
  es normalerweise gar nicht da — den Alias bringt nur
  `jupiter-legacy-support` mit, und das installierst du nach dem Abschnitt
  oben ohnehin nicht. Nachsehen:
  `command -v steamos-update || echo "nicht vorhanden - gut"`.
- **`holo-session-select`** heißt nicht mehr `steamos-session-select`. Den
  Kompatibilitäts-Symlink liefert `steamos-alias`.
- **unl0krs Install-Hook** findet den Touchscreen des Decks nicht.
- **PipeWire sucht nicht in `/run/pipewire`.** Valves `pipewire-sysconf`
  installiert das Hardwareprofil dorthin; auf Arch muss es nach `/etc`.
- **Arch' `brightnessctl` bringt keine udev-Regel mit.**
- **Das Deck meldet zwei Einschaltknopf-Geräte.** Jede umschaltende
  Tastenbelegung darauf feuert doppelt und hebt sich selbst auf.

---

## Im Alltag

| Aufgabe | |
|---|---|
| Nach jedem Update | `deck-health` (läuft automatisch) |
| In den Spielmodus | Omarchy-Menü → System → Game Mode (auch per Touch) |
| Zurück | Steam-Knopf → Ein/Aus → Zum Desktop wechseln |
| gamescope aktualisieren | `sudo pacman -S jupiter-main/gamescope` |
| Unterwegs | herunterfahren statt schlafen legen, siehe unten |
| Wenn gar nichts geht | Limine-Menü → Snapshots → obersten Eintrag |

---

## Notfall und zurück zu SteamOS

**Der neue Kernel startet nicht.** Einschaltknopf zehn Sekunden halten, dann
wieder einschalten. Im Limine-Menü mit dem D-Pad auswählen, mit **Enter** oder
dem **A-Knopf** bestätigen:

- `linux` — der Arch-Kernel. Er bleibt absichtlich installiert, kennt die
  Deck-Hardware weniger gut, startet aber zuverlässig.
- `linux-neptune-618-fallback` — derselbe Kernel mit einem initramfs, das alle
  Treiber enthält. Der richtige Griff, wenn du dir in Phase 7 das Abbild
  zerlegt hast.
- **Snapshots** → oberster Eintrag — der Stand vor deinem letzten
  `omarchy-snapshot create`, mit eigenem Kernel-Abbild und eigenem initramfs.

**Das Limine-Menü kommt gar nicht.** Dann ist die EFI-Partition betroffen. Vom
Omarchy-Stick starten, im Live-System das entschlüsselte Wurzeldateisystem und
`/boot` einhängen, mit `arch-chroot` hineinwechseln und `limine-mkinitcpio`
erneut laufen lassen.

**Die Bildschirmtastatur erscheint beim Entsperren nicht.** Hier hilft kein
Snapshot, denn um einen auszuwählen, müsstest du erst entsperren. Genau
deshalb die Regel aus 7.1: USB-Tastatur anstecken, damit entsperren, dann in
Ruhe reparieren.

**Ganz zurück zu SteamOS.** Valves Wiederherstellungsabbild auf einen Stick
schreiben, davon starten, „Reimage Steam Deck" wählen. Das löscht die SSD
vollständig und stellt den Auslieferungszustand her — von Omarchy und der
Verschlüsselung bleibt nichts übrig. Suchbegriff bei Valve:
„Steam Deck Recovery Instructions".

> **Erstell diesen Stick, bevor du mit Phase 1 anfängst.** Man braucht ihn
> genau dann, wenn das Gerät nicht mehr startet — und dann ist es zu spät, ihn
> herunterzuladen.

---

## Was offen bleibt

**Sperrbildschirm ohne Tastatur.** Sperrst du im Desktopmodus den Bildschirm
und nimmst das Deck danach vom Dock, kommst du nicht mehr hinein: Omarchys
Sperrbildschirm ist ein Quickshell-Plugin ohne Bildschirmtastatur. Die
Touch-Tastatur aus Phase 7 sitzt im initramfs und hilft nur bei der
LUKS-Abfrage, also nur beim Kaltstart.

Praktisch trifft einen das seltener, als es klingt:

- **Einen Untätigkeits-Timer gibt es nicht.** `hypridle` läuft auf Omarchy
  nicht; der Bildschirm sperrt sich also nie von selbst.
- Die einzige automatische Sperre ist `omarchy-sleep-lock.service`. Sie lauscht
  über `omarchy-system-sleep-monitor` auf `PrepareForSleep` und sperrt **vor
  dem Suspend** — bedingungslos, anders als das Power-Menü, das vorher
  `omarchy-toggle-enabled suspend-off` abfragt.
- Es ist eine **User**-Unit mit `PartOf=graphical-session.target` und
  `ConditionEnvironment=WAYLAND_DISPLAY`, hängt also an der Hyprland-Sitzung.
  Im Spielmodus läuft die nicht, dort sollte die Sperre folglich nicht greifen.
  Nachgemessen haben wir das nicht.

Am Schreibtisch hängt ohnehin eine Tastatur am Dock, und unterwegs hilft
Herunterfahren statt Schlafenlegen — dann greift die LUKS-Abfrage mit der
Touch-Tastatur. Wem das zu wacklig ist, der hat drei Wege:

1. **Sperre vor dem Schlafen abschalten.** Überlebt Updates.

   ```bash
   systemctl --user mask --now omarchy-sleep-lock.service
   ```

2. **Bedingt sperren.** Die Unit maskieren und durch einen eigenen Monitor
   ersetzen, der vor dem Sperren prüft, ob eine echte Tastatur am Gerät hängt.
   Abgedockt schläft das Deck dann ungesperrt, gedockt verhält es sich wie
   bisher. Klein, aber man muss erst herausfinden, was das Deck intern alles
   als Tastatur meldet — die beiden Einschaltknopf-Geräte reden mit.

3. **Tastatur in den Sperrbildschirm bauen.** `LockView.qml` unter
   `/usr/share/omarchy/shell/plugins/lock/` exportiert `submitPassword(string)`,
   `passwordTextEdited(string)`, `forcePasswordFocus()` und `clearPassword()` —
   eine Bildschirmtastatur könnte das also von innen füttern. Der Pfad gehört
   dem Paket und wird bei Updates überschrieben; es bräuchte zusätzlich einen
   Weg, Plugins nutzerseitig zu überlagern.

Wir haben nichts davon gemacht und leben mit Weg null.

**WLAN und Bandwahl.** Lässt NetworkManager das Band frei wählen, versucht das
Deck in ein 5-GHz-Netz zu wechseln, auch wenn dieses viel schwächer ist als das
2,4-GHz-Netz daneben. Die Anmeldung läuft in eine Zeitüberschreitung, es fällt
zurück, und das alle 30 Sekunden. Erkennbar an:

```bash
nmcli -f IN-USE,BSSID,SIGNAL,CHAN device wifi list
sudo dmesg | grep -i "by local choice"
```

Steht der aktive Eintrag unter 50, ist das die Ursache. Abhilfe: Band im
Netzwerk-Widget auf 2,4 GHz festlegen, oder
`nmcli connection modify '<SSID>' 802-11-wireless.band bg`.

Beteiligt sind vermutlich auch 802.11r (Schnellroaming) und der WPA2/WPA3-
Übergangsmodus — der `ath11k`-Treiber hat mit beiden bekannte Schwierigkeiten.
Beides lässt sich an einer FRITZ!Box abschalten, betrifft dann aber alle
Geräte.

**Drehung in unl0kr.** Der Quelltext von buffybox enthält die Drehlogik
(`lv_display_set_rotation`) in buffyboard. Sie nach unl0kr zurückzuportieren
wäre überschaubar — dann bekäme man ein vollwertiges Entsperrfenster statt
Prompt plus Tastatur. Wir haben es nicht gebraucht.

**Umlaute und Sonderzeichen in der Passphrase.** Lösbar, indem man
buffyboard mit deutschem Layout baut und die Konsolenbelegung passend setzt.
Nicht versucht.

---

## Dank

An das **postmarketOS**-Projekt für buffybox, an **Valve** dafür, dass die
Deck-Pakete offen und einzeln installierbar sind, und an die
**Omarchy**-Entwickler für eine Arch-Distribution, die sich so weit
auseinandernehmen lässt, ohne zu zerbrechen.

Entstanden an zwei Abenden im Oktober 2026, im Wechsel zwischen einem Menschen
mit einem Steam Deck und Claude. Mehrere Sackgassen inklusive — die
interessanten Stellen sind oben unter „Fallen" vermerkt.
