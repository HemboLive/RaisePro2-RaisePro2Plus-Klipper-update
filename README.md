# Klipper auf STM32H743VI flashen (ROM-DFU über USB) — inkl. Read-Protection entfernen

Anleitung für ein STM32H743VI-Board, das sich per USB im **ROM-DFU-Modus** (`0483:df11`)
meldet und bei dem das Flashen mit dem System-`dfu-util` (0.11) an
`ERASE_PAGE not correctly executed` bzw. `MASS_ERASE not correctly executed` scheitert.

**Ergebnis:** Klipper läuft, der MCU meldet sich als
`/dev/serial/by-id/usb-Klipper_stm32h743xx_*-if00`.

---

## TL;DR — die drei entscheidenden Erkenntnisse

1. **Das Board ist ab Werk read-protected (RDP Level 1).** Erst RDP entfernen, dann flashen.
   Der `ERASE_PAGE`-Fehler ist **kein** dfu-util-Bug, sondern die aktive Read-Protection.
2. **Kein Bootloader** → Firmware auf **`0x08000000`** bauen/flashen (NICHT `0x08020000`).
3. **12-MHz-Quarz** in der Klipper-Config wählen. Falscher Referenztakt (z. B. 8 MHz)
   → die Firmware wird zwar korrekt geschrieben, der MCU bootet aber „dunkel"
   (kein USB, weil der 48-MHz-USB-Takt nicht stimmt).

> ⚠️ **RDP entfernen löscht den kompletten Flash (Mass-Erase).** Das ist hier gewollt.
> **Niemals RDP Level 2 setzen** — das ist irreversibel (Brick). Level 2 ist hier aber
> ausgeschlossen, solange sich das Board überhaupt im DFU-Modus meldet.

---

## Voraussetzungen

- Raspberry Pi (o. ä.) mit Klipper-Umgebung, `~/klipper` vorhanden.
- Zugriff auf die **BOOT0/RESET**- bzw. **SW1/SW2**-Taster des Boards.
- Board per USB angeschlossen.

---

## Schritt 1 — Aktuelle `dfu-util`-Entwicklerversion bauen

Das System-`dfu-util` 0.11 wird für den Ablauf nicht zwingend gebraucht, aber die
Dev-Version ist die Referenz und bringt die `unprotect`-Option sauber mit.

```bash
sudo apt install -y automake libtool pkg-config libusb-1.0-0-dev git
cd ~
git clone https://git.code.sf.net/p/dfu-util/dfu-util dfu-util
cd dfu-util
./autogen.sh
./configure
make -j"$(nproc)"
./src/dfu-util --version      # sollte "dfu-util 0.11-dev" zeigen
```

Alle folgenden `dfu-util`-Aufrufe nutzen dieses Binary: `~/dfu-util/src/dfu-util`.

---

## Schritt 2 — Klipper-Dienst stoppen

Damit der Host nicht versucht, den MCU zu belegen:

```bash
sudo service klipper stop
```

---

## Schritt 3 — Board in den DFU-Modus bringen

Am Board **BOOT0 halten + RESET** kurz drücken/loslassen, dann BOOT0 loslassen.
(Bei diesem Board: **SW1 (links) + SW2 (rechts) gleichzeitig** drücken, dann rechts zuerst los lassen.)

Prüfen:

```bash
lsusb | grep df11
# Bus 001 Device 00X: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
```

---

## Schritt 4 — Read-Protection (RDP Level 1) entfernen

Das ist der Kern-Fix gegen `ERASE_PAGE not correctly executed`.

```bash
sudo ~/dfu-util/src/dfu-util -d ,0483:df11 -a 0 -s 0x08000000:unprotect:force
```

- Der `READ_UNPROTECT`-Befehl löst einen **Mass-Erase + RDP 1→0** aus, danach
  **trennt/reset**et sich das Board selbst.
- Ein abschließendes `get_status: LIBUSB_ERROR_PIPE` ist hier **normal** (das Board
  ist bereits offline gegangen, um zu löschen/resetten).

**Falls der Befehl mit STALL / `state(15)` abbricht** (Bootloader aus vorherigen
Fehlversuchen verklemmt): einmal USB-seitig resetten und den Befehl **als ersten**
erneut ausführen:

```bash
# Geräteknoten aus 'lsusb' ableiten: Bus 001 Device 00X -> /dev/bus/usb/001/00X
sudo python3 -c "import fcntl; fcntl.ioctl(open('/dev/bus/usb/001/00X','wb'), 0x5514, 0)"
# danach sofort erneut den unprotect-Befehl aus Schritt 4
```

Alternativ: Board komplett stromlos machen (~5 s) und sauber neu in DFU
(BOOT0/SW1+SW2) — dann `unprotect` als ersten Befehl.

### Kontrolle: RDP ist weg

Board erneut in DFU bringen und den Flash-Anfang lesen — muss jetzt klappen und
lauter `0xFF` zeigen:

```bash
sudo ~/dfu-util/src/dfu-util -a 0 -s 0x08000000:256 -U /tmp/flash.bin
xxd /tmp/flash.bin | head
```

---

## Schritt 5 — Klipper mit den RICHTIGEN Einstellungen bauen

`cd ~/klipper && make menuconfig` und Folgendes setzen:

- **Micro-controller Architecture:** STMicroelectronics STM32
- **Processor model:** STM32H743
- **Bootloader offset:** **No bootloader**  ← wichtig (= Flash-Start `0x08000000`)
- **Clock Reference:** **12 MHz crystal**  ← wichtig
  (dafür ggf. oben **„Enable extra low-level configuration options"** aktivieren)
- **Communication interface:** USB (on PA11/PA12)

Ergebnis in `~/klipper/.config` (zur Kontrolle):

```
CONFIG_MACH_STM32H743=y
CONFIG_STM32_FLASH_START_0000=y
CONFIG_FLASH_APPLICATION_ADDRESS=0x8000000
CONFIG_LOW_LEVEL_OPTIONS=y
CONFIG_STM32_CLOCK_REF_12M=y
CONFIG_CLOCK_REF_FREQ=12000000
CONFIG_STM32_USB_PA11_PA12=y
CONFIG_USBSERIAL=y
```

Bauen:

```bash
cd ~/klipper
make -j"$(nproc)"
ls -l out/klipper.bin
```

---

## Schritt 6 — Firmware flashen (auf `0x08000000`)

Board in DFU bringen (Schritt 3), dann:

```bash
sudo ~/dfu-util/src/dfu-util -d ,0483:df11 -a 0 -s 0x08000000:leave -D ~/klipper/out/klipper.bin
```

- Erwartete Ausgabe endet mit **`File downloaded successfully`**.
- Das anschließende `get_status: LIBUSB_ERROR_PIPE` nach `Submitting leave request...`
  ist **harmlos** — durch `:leave` verlässt der MCU DFU und resettet in die Firmware.

Optional prüfen (zurücklesen und vergleichen), Board dafür wieder in DFU:

```bash
SZ=$(stat -c %s ~/klipper/out/klipper.bin)
sudo ~/dfu-util/src/dfu-util -a 0 -s 0x08000000:$SZ -U /tmp/readback.bin
cmp ~/klipper/out/klipper.bin /tmp/readback.bin && echo "identisch"
```

---

## Schritt 7 — Prüfen & Klipper starten

Nach dem Flashen (ggf. einmal **RESET ohne BOOT0** drücken) sollte der MCU als
Klipper-USB-Gerät erscheinen:

```bash
lsusb | grep 1d50:614e
ls -l /dev/serial/by-id/
# usb-Klipper_stm32h743xx_XXXXXXXXXXXX-if00 -> ../../ttyACM0
```

Diesen `serial`-Pfad in die `printer.cfg` unter `[mcu]` eintragen, dann:

```bash
sudo service klipper start
tail -f ~/printer_data/logs/klippy.log
```

Erfolg sichtbar an `Configured MCU 'mcu'` und laufenden `Stats`-Zeilen mit
`freq=~400000000` (der korrekte 400-MHz-Takt dank 12-MHz-Referenz).

---

## Häufige Stolpersteine

| Symptom | Ursache / Lösung |
|---|---|
| `ERASE_PAGE`/`MASS_ERASE not correctly executed` | Read-Protection aktiv → **Schritt 4** (RDP entfernen). |
| Jeder DFU-Befehl STALLt, `state(15) = (null)` | Bootloader verklemmt → USB-Reset (ioctl `0x5514`) oder Board stromlos machen, dann Befehl als ERSTEN absetzen. |
| Flash OK, aber MCU „dunkel" (kein `/dev/ttyACM`) | Falscher Quarz (z. B. 8 statt **12 MHz**) → USB-Takt falsch. Firmware mit 12 MHz neu bauen. |
| MCU bootet nicht, obwohl Firmware @0x08020000 | Kein Bootloader vorhanden → **No bootloader / 0x08000000** bauen. |
| `LIBUSB_ERROR_PIPE` nach `:leave` / nach `unprotect` | **Harmlos** — Gerät hat sich zum Reset/Erase getrennt. |
| Board flasht/enumeriert unzuverlässig | Stromversorgung prüfen (im Log tauchten „Undervoltage detected"-Warnungen des Pi auf). |

---

*Getestet auf: Raspberry Pi OS (Bookworm, ARM64), STM32H743VI, `dfu-util 0.11-dev`, Klipper v0.13.*

---

# Anhang: Betrieb mit den Original-Komponenten (board-spezifische Workarounds)

Dieses Board/Drucker läuft **nicht** mit einer 0815-Klipper-Config — die Original-Hardware
(zwei Druckköpfe, Servo-Kopfanhebung, Onboard-Stepper-Treiber mit PWM-Referenzspannung,
AD595-Thermoelemente) braucht ein paar spezielle Einstellungen. Ohne die bewegen sich
insbesondere die **Extruder nicht**. Hier die relevanten Teile, damit alles mit den
**Originalteilen** funktioniert.

> Hinweis: Pressure Advance ist hier bewusst **nicht** dokumentiert — die Druckköpfe sind
> Sonderanfertigungen, die Werte sind nicht übertragbar.

## A) Pinbelegung der Steckerleisten (Referenz)

```
J1  X-  PG0        J2  X+  PG1        J4  Y-  PG2        J5  Y+  PG3
J7  Z-  PG4        J8  Z+  PG5        J9  Filament-Runout 1/2  PG7, PG6
J27 Bett-Heizung   PB15              J28 HE1/HE2  PB6, PB9
J29 Kopfkabel: E1-Thermo PA5, E2-Thermo PC5, Lüfter L/R PA6/PA7, Kopf-Lift-Servo PB14
J31 Bett-Thermistor PB0
J33 X  step/dir/ena  PH11 / PH7 / PE15
J34 Y  step/dir/ena  PH10 / PH6 / PE12
J38 Z  step/dir/ena  PA3  / PH5 / PE2
J37 E1 step/dir/ena  PA2  / PH4 / PE10
J36 E2 step/dir/ena  PA1  / PH3 / PE8
LEDs: USB PC0, Error PC1, Run PC2, Beleuchtung PG15
```

## B) WICHTIG: Onboard-Treiber „scharf schalten" (Referenzspannung/Motorstrom)

Die Onboard-Stepper-Treiber der Extruder bekommen ihren **Motorstrom über eine
PWM-erzeugte Referenzspannung**. Ist dieser Wert falsch (oder 0), liefern die Treiber
keinen Strom → **der Extruder-Motor bewegt sich nicht**.

Umgesetzt als zwei PWM-Ausgänge (technisch als `fan_generic`, **es sind keine Lüfter!**):

```ini
[fan_generic current_E0]
pin = PE13
cycle_time = 0.001

[fan_generic current_E1]
pin = PE11
cycle_time = 0.001
```

Gesetzt wird die Referenz mit `SET_FAN_SPEED FAN=current_E0 SPEED=0.3` (bzw. `current_E1`).
**Der Wert `0.3` ist der funktionierende Referenzwert — nicht blind ändern.** Zu niedrig →
Treiber liefert zu wenig Strom / Motor bleibt stehen.

Zusätzlich werden Treiber-Konfig-/Enable-Leitungen einmalig beim Start statisch gesetzt
(nötig, damit die Onboard-Treiber überhaupt arbeiten):

```ini
[static_digital_output my_output_pins]
pins =
    PB7,
    PE3,
    !PI1,
    PH13,
    !PH14,
    !PH15
```

Besonderheit der Enable-Pins: Die **Extruder-Treiber sind NICHT invertiert**
(`enable_pin = PE10` bzw. `PE8` — ohne `!`), während X/Y/Z invertiert sind (`!PE15` etc.).
Falsch herum → Treiber bleibt dauerhaft aus/an.

## C) WICHTIG: Druckkopf-Anhebung per Servo/Aktor (PB14)

Bei zwei Druckköpfen wird der **inaktive Kopf angehoben**. Das übernimmt ein Aktor/Servo
an **PB14**, als einfacher PWM-Ausgang eingebunden (verwirrend „extruder" benannt):

```ini
[output_pin extruder]
pin = PB14
pwm: True
value = 1          # Startzustand
```

Umschaltung passiert im Werkzeugwechsel (siehe D): `SET_PIN PIN=extruder VALUE=1` senkt
Kopf 0 / hebt Kopf 1, `VALUE=0` umgekehrt.

## D) Werkzeugwechsel T0/T1 — verbindet Servo + Treiberstrom + Sensoren

Die Makros `T0`/`T1` schalten in einem Rutsch: aktiven Extruder, Kopf-Servo,
Treiber-Referenzstrom und den passenden Filamentsensor. **Das ist der Kern-Workaround,
damit der jeweils aktive Kopf fährt:**

```ini
[gcode_macro T0]
gcode =
    G1 F21000
    SET_GCODE_OFFSET Z=0 X=0 Y=0 MOVE=1
    ACTIVATE_EXTRUDER extruder=extruder
    SAVE_VARIABLE VARIABLE=currentextruder VALUE='"extruder"'
    SET_PIN PIN=extruder VALUE=1                 # Kopf-Servo Position 0
    G4 P500
    SET_FAN_SPEED FAN=current_E0 SPEED=0.3        # Treiberstrom Extruder 0
    {% set svv = printer.save_variables.variables %}
    {% if svv.filament_sensor_enabled|int != 0 %}
    SET_FILAMENT_SENSOR SENSOR=F1 ENABLE=0
    SET_FILAMENT_SENSOR SENSOR=F0 ENABLE=1
    {% endif %}

[gcode_macro T1]
gcode =
    G1 F21000
    ACTIVATE_EXTRUDER extruder=extruder1
    SAVE_VARIABLE VARIABLE=currentextruder VALUE='"extruder1"'
    SET_PIN PIN=extruder VALUE=0                  # Kopf-Servo Position 1
    G4 P500
    SET_FAN_SPEED FAN=current_E1 SPEED=0.3        # Treiberstrom Extruder 1
    SET_GCODE_OFFSET Z=0.27 X=-25 Y=0 MOVE=1      # Offset 2. Kopf
    ... (Filamentsensor-Umschaltung analog)
```

## E) Thermofühler: AD595-Thermoelemente statt Thermistoren

Beide Extruder nutzen **AD595-Thermoelement-Verstärker** (nicht die üblichen Thermistoren):

```ini
[extruder]
sensor_type = AD595
sensor_pin = PA5
adc_voltage = 3.3
# extruder1: sensor_type = AD595, sensor_pin = PC5
```

Bett dagegen klassisch: `EPCOS 100K B57560G104F` an `PB0`.

## F) Startverhalten (Autostart beim Verbinden)

Ein `[delayed_gcode]` bringt den Drucker nach dem Klipper-Connect in den Grundzustand:
aktiviert Extruder 0, setzt Kopf-Servo + Treiberstrom, wählt Filamentsensor F0 — **und
heizt das Bett automatisch auf 80 °C vor**:

```ini
[delayed_gcode startup_macro]
initial_duration: 1
gcode:
    ...
    SET_PIN PIN=extruder VALUE=1
    SET_FAN_SPEED FAN=current_E0 SPEED=0.3
    SET_FILAMENT_SENSOR SENSOR=F0 ENABLE=1
    SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=80    # <-- Bett-Vorheizen

```

> Daher heizt das Bett **direkt nach dem Start** auf 80 °C. Wer das nicht will,
> entfernt die `SET_HEATER_TEMPERATURE ...`-Zeile aus dem `startup_macro`.

## G) Weitere nötige Grundeinstellungen

- `[force_move] enable_force_move = True` — erlaubt manuelles Verfahren ohne Homing
  (praktisch/erforderlich bei der Kopf-/Servo-Justage).
- Teil-Kühllüfter `[fan]` liegt auf zwei Pins (links **PA6**, rechts **PA7**) via
  `[multi_pin my_fan_pins]`.
- `[save_variables]` speichert u. a. den zuletzt aktiven Extruder (`currentextruder`) und
  den Filamentsensor-Status in `~/printer_data/config/variable.cfg` — damit nach Neustart
  der richtige Kopf/Servo/Treiberstrom-Zustand wiederhergestellt wird.

---

### Kurz-Checkliste „Extruder bewegt sich nicht"

1. Läuft der **Treiberstrom-Ausgang**? (`SET_FAN_SPEED FAN=current_E0 SPEED=0.3` bzw. E1)
2. Sind die **statischen Treiber-Pins** gesetzt? (`[static_digital_output my_output_pins]`)
3. **Enable-Pin-Polung** korrekt? (Extruder **ohne** `!`, X/Y/Z **mit** `!`)
4. Richtiger **Kopf per Servo** unten? (`SET_PIN PIN=extruder VALUE=0/1`)
5. Wurde der Extruder **aktiviert**? (`ACTIVATE_EXTRUDER` bzw. `T0`/`T1`)
