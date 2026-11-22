# waipu-tv-root-2


Kurz vorab, weil das den Aufbau bestimmt: TV Tools ist eine **Handy-App**, die über WLAN-ADB mit dem Stick redet. Ihr Shell-Fenster schickt **Zeile für Zeile** an den Stick, jede Zeile läuft als eigener `adb shell`-Aufruf – also **keine Variablen, keine Funktionen, kein `adb`-Präfix**. Und es läuft als `shell`-User, nicht als root.

Zwei Konsequenzen, die du kennen musst:

| geht über die App | geht nur am PC |
|---|---|
| Zustand auslesen, Partitionen dumpen, Debloat, Launcher, Apps, Dateien | **OEM-Unlock (DNL braucht USB-Host + `dnl_send.py` + `adnl`)** |
| alles, was `shell` darf | TWRP flashen (macht der PC) |

Der Unlock bleibt also am PC. Alles danach – inklusive Backups und Root-Arbeit, wenn die Shell als root läuft – kann die App. Deshalb sind die Blöcke hier **zeilenunabhängig**: jede Zeile funktioniert allein, in beliebiger Reihenfolge, ohne Vorbedingung außer der, die dabeisteht.

## Block 0 – Arbeitsordner

```
mkdir -p /sdcard/wt
```

## Block 1 – Zustand (immer zuerst, ändert nichts)

```
getprop ro.build.fingerprint
```
```
getprop ro.boot.verifiedbootstate
```
```
getprop ro.boot.vbmeta.device_state
```
```
id
```
```
command -v su || echo KEIN_SU
```
```
ls -l /dev/block/by-name/
```
```
ls -l /dev/block/mapper/
```

Erwartet: Fingerprint beginnt mit `waiputv/SEI700WP/SEI700WP:11/`. `verifiedbootstate` ist `green` (zu) oder `orange` (offen). `id` sagt `uid=2000(shell)` – alles andere heißt, du bist schon root. `KEIN_SU` ist normal.

## Block 2 – Backup (vor jedem Eingriff, auch ohne Root)

```
dd if=/dev/block/by-name/boot_a of=/sdcard/wt/boot_a.img bs=1M
```
```
sha256sum /sdcard/wt/boot_a.img
```
```
dd if=/dev/block/by-name/vendor_boot_a of=/sdcard/wt/vendor_boot_a.img bs=1M
```
```
sha256sum /sdcard/wt/vendor_boot_a.img
```
```
dd if=/dev/block/by-name/misc of=/sdcard/wt/misc.img bs=1M
```
```
sha256sum /sdcard/wt/misc.img
```
```
dd if=/dev/block/by-name/vbmeta of=/sdcard/wt/vbmeta.img bs=1M
```
```
dd if=/dev/block/by-name/vbmeta_system of=/sdcard/wt/vbmeta_system.img bs=1M
```
```
dd if=/dev/block/by-name/dtbo_a of=/sdcard/wt/dtbo_a.img bs=1M
```
```
ls -l /sdcard/wt/
```

Prüfen: `boot_a.img` **muss** `9e52620ce0c060aebed5bdf3d2d423b983e6791f49b84cd4f886c76aa9181ce8` ergeben, solange du Stock läufst. Weicht der Hash ab, ist schon etwas gepatcht – dann nicht weitermachen, ohne das geklärt zu haben. Danach alles über den Dateimanager der App aufs Handy ziehen (`/sdcard/wt`), denn auf dem Stick ist `/sdcard` bei einem Wipe weg.

## Block 3 – Waipu-Zwangsdienste stilllegen

```
pm suspend com.nes.skywayclient
```
```
pm suspend com.nes.coreservice
```
```
pm suspend com.nes.leanbacklauncher.partnercustomizer
```
```
pm suspend com.nes.remoteota.waipu
```
```
cmd appops set com.nes.skywayclient RUN_ANY_IN_BACKGROUND ignore
```
```
cmd appops set com.nes.coreservice RUN_ANY_IN_BACKGROUND ignore
```
```
pm list packages --suspended
```

`skywayclient` reaktiviert die anderen wieder – deshalb ist es mit dabei. `coreservice` ist der, der Fremd-APK-Installationen blockt. Kontrolle: `pm list packages --suspended` zeigt alle vier.

## Block 4 – Eigener Launcher als Home

```
cmd package set-home-activity com.spocky.projengmenu/.ui.home.MainActivity
```
```
cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME
```

Zweite Zeile muss den neuen Launcher ausgeben. Anderer Launcher: Paket/Activity entsprechend tauschen (mit `pm list packages | grep -i launcher` findest du, was installiert ist). Vorher in der App installieren, sonst zeigt Home auf nichts.

## Block 5 – Rückgängig

```
pm unsuspend com.nes.skywayclient
```
```
pm unsuspend com.nes.coreservice
```
```
pm unsuspend com.nes.leanbacklauncher.partnercustomizer
```
```
pm unsuspend com.nes.remoteota.waipu
```
```
cmd package set-home-activity com.nes.leanbacklauncher.partnercustomizer/.MainActivity
```
```
dd if=/sdcard/wt/boot_a.img of=/dev/block/by-name/boot_a bs=1M
```

Die letzte Zeile nur, wenn du vorher TWRP draufgeflasht hattest – dann zurück auf Stock-`boot_a`, sonst bootet der Stick weiter ins Recovery. Datei vorher vom Handy zurück auf den Stick schieben.

## Block 6 – nur wenn `id` root sagt

```
twrp set tw_enable_adb 1
```
```
twrp set tw_no_screen_timeout 1
```
```
mkdir -p /data/local/tmp/wt
```
```
dd if=/dev/block/by-name/super of=/data/local/tmp/wt/super.img bs=1M
```
```
sha256sum /dev/block/by-name/misc
```
`misc` muss `1c16ac591e620d5553b4ab0ef839ad11843c8fee4490462e5f6252b674eb53da` bleiben. `/system` lässt sich auch als root nicht rw mounten (EXT4-Feature 0x4000) – Schreibzugriff aufs System gibt's nur über neu geflashte Images, nicht über `mount -o remount,rw`.

## Wenn du die Shell lieber als Datei fährst

Die App hat einen Dateimanager – schieb `wt.sh` aufs Handy und von dort nach `/sdcard/wt.sh`, dann:

```
sh /sdcard/wt.sh doctor
```

`wt.sh`:

```sh
#!/system/bin/sh
# TVTools-Fassung: laeuft als shell, degradiert bei fehlendem root
set -u
W=/sdcard/wt
L(){ echo "[wt] $*"; }
root(){ [ "$(id -u)" = 0 ]; }

case "${1:-help}" in
doctor)
  L "fp  : $(getprop ro.build.fingerprint)"
  L "vb  : $(getprop ro.boot.verifiedbootstate) / $(getprop ro.boot.vbmeta.device_state)"
  L "uid : $(id)"
  L "su  : $(command -v su || echo NEIN)"
  L "--- by-name ---"; ls -l /dev/block/by-name/ 2>/dev/null
  L "--- mapper ---";  ls -l /dev/block/mapper/ 2>/dev/null
  ;;
backup)
  mkdir -p "$W"
  for p in boot_a vendor_boot_a misc vbmeta vbmeta_system dtbo_a; do
    [ -e "/dev/block/by-name/$p" ] || { L "$p fehlt"; continue; }
    dd if="/dev/block/by-name/$p" of="$W/$p.img" bs=1M 2>/dev/null \
      && L "$p $(sha256sum "$W/$p.img" | awk '{print $1}')" || L "$p LESEFEHLER"
  done
  L "boot_a erwartet 9e52620c...1ce8"
  ;;
debloat)
  for p in com.nes.skywayclient com.nes.coreservice \
           com.nes.leanbacklauncher.partnercustomizer com.nes.remoteota.waipu; do
    pm suspend "$p" && L "$p suspendiert" || L "$p fehlgeschlagen"
    cmd appops set "$p" RUN_ANY_IN_BACKGROUND ignore 2>/dev/null
  done
  ;;
undo)
  for p in com.nes.skywayclient com.nes.coreservice \
           com.nes.leanbacklauncher.partnercustomizer com.nes.remoteota.waipu; do
    pm unsuspend "$p" && L "$p laeuft wieder"
  done
  cmd package set-home-activity com.nes.leanbacklauncher.partnercustomizer/.MainActivity
  ;;
launcher)
  [ -n "${2:-}" ] || { L "Aufruf: launcher <paket/.Activity>"; exit 1; }
  cmd package set-home-activity "$2" && L "Home -> $2"
  ;;
rootcheck)
  if root; then L "root bestaetigt: $(id)"; else L "kein root (shell-User)"; fi
  ;;
*) sed -n '2,3p' "$0" ;;
esac
```

## Wo die App dir Arbeit abnimmt

Doppelt musst du nicht arbeiten: Apps installieren/deaktivieren, Dateien schieben und APK-Backups macht TV Tools über seine eigenen Menüs. Nutze die Shell nur für das, was die UI nicht kann – `pm suspend`, `appops`, `set-home-activity`, `dd` auf Blockgeräte. Und für die Diagnose vor dem Debloat, weil die App dir keinen `verifiedbootstate` zeigt.

## Root-Shell über die App – der ehrliche Stand

Wenn TWRP in `boot_a` liegt, läuft dort `adbd` als root. Theoretisch könntest du in TWRP per TCP draufgehen:

```
setprop service.adb.tcp.port 5555
```
```
setprop ctl.restart adbd
```
```
ip -4 addr show wlan0
```

Dann in TV Tools gegen `<stick-ip>:5555` verbinden. Praktisch kommt WLAN in diesem TWRP meist nicht hoch (kein `wpa_supplicant`, Module nicht geladen) – dann hast du keine IP und die App kommt nicht ran. Verlass dich für Root-Arbeit deshalb weiter auf den PC, die App ist dein Android-seitiges Werkzeug.

Wenn du willst, baue ich dir den Datei-Weg noch als echtes Install-Skript aus (Push nach `/sdcard/wt.sh`, Aufruf über eine Kachel in TV Tools) – oder den `system.img`-Patch am PC, weil dort erst der dauerhafte `su` herkommt.