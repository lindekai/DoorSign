# DoorSign // Digitales Türschild für Konferenzräume

ESP32 + Waveshare 7.5" E-Ink Display zeigt Raumbelegung aus einem Kalender-Server an.
Der Server rendert pro Raum ein Bild (PNG oder BMP), der ESP32 lädt und zeigt es an.

---

## Inhaltsverzeichnis

1. [Funktionsübersicht](#funktionsübersicht)
2. [Hardware](#hardware)
3. [Pinbelegung](#pinbelegung)
4. [Architektur](#architektur)
5. [Bibliotheken installieren](#bibliotheken-installieren)
6. [Gerät konfigurieren](#gerät-konfigurieren)
7. [Kompilieren und Flashen](#kompilieren-und-flashen)
8. [Betriebsmodi](#betriebsmodi)
9. [Bildformat](#bildformat)
10. [Display-Versionen V1 / V2 / V3](#display-versionen-v1--v2--v3)
11. [Stromversorgung](#stromversorgung)
12. [Energieverbrauch und Akku-Laufzeit](#energieverbrauch-und-akku-laufzeit)
13. [OTA-Updates](#ota-updates)
14. [Fehlersuche](#fehlersuche)
15. [Projektstruktur](#projektstruktur)
16. [Bekannte Einschränkungen](#bekannte-einschränkungen)

---

## Funktionsübersicht

- Lädt regelmäßig ein Bild (PNG oder BMP) vom Server und zeigt es auf dem E-Ink-Display
- Aktiv nur Mo–Fr, 07:55–17:00 Uhr (minutengenau konfigurierbar, Europe/Berlin, Sommerzeit automatisch)
- Außerhalb der Betriebszeit: letztes Bild bleibt sichtbar, ESP32 schläft
- **Drei Display-Versionen** unterstützt: V1, V2 und V3 (mit Rot-Kanal)
- **Zwei Bildformate** umschaltbar: PNG (1-Bit/Graustufen) und BMP (1-Bit oder 4-Bit Palette)
- **Zwei Betriebsmodi**: Deep Sleep (Akku) und Dauerbetrieb (Netzteil)
- HTTP-ETag-Caching → kein Display-Refresh wenn Bild unverändert
- Letztes Bild in LittleFS persistiert → überlebt Stromausfall
- OTA-Firmware-Updates über WLAN

---

## Hardware

| Komponente | Beschreibung |
|---|---|
| ESP32 | Waveshare ESP32 Driver Board |
| Display | Waveshare 7.5" E-Paper V1, V2 oder V3 |
| Stromversorgung | USB-C 5V/1A Netzteil, oder LiPo-Akku mit Lademodul und Low-Iq-Buck — siehe [Stromversorgung](#stromversorgung) |

---

## Pinbelegung

Waveshare ESP32 Driver Board ↔ E-Paper HAT:

| E-Paper Pin | ESP32 GPIO | Funktion |
|---|---|---|
| VCC | 3V3 | Versorgungsspannung |
| GND | GND | Masse |
| DIN / MOSI | GPIO 14 | SPI Daten |
| SCLK | GPIO 13 | SPI Takt |
| CS | GPIO 15 | Chip Select (aktiv LOW) |
| DC | GPIO 27 | Data/Command |
| RST | GPIO 26 | Reset (aktiv LOW) |
| BUSY | GPIO 25 | Busy-Signal |

Alle Pins in `config.h` (`PIN_EPD_*` und `PIN_SPI_*`) konfigurierbar.

---

## Architektur

### Deep Sleep Modus (Akku)

```
Boot / Wake-Up
  │
  ├─ Kaltstart? → Startbildschirm anzeigen
  ├─ WLAN verbinden  →  Fehler? → letztes Bild → Sleep
  ├─ OTA-Fenster 60s (nur bei Kaltstart)
  ├─ NTP synchronisieren
  ├─ Zeitfenster prüfen (Mo–Fr 07:55–17:00)
  │   └─ Inaktiv? → Sleep bis 07:55
  ├─ Bild laden (ETag-Check)
  │   ├─ Neu  → Display aktualisieren
  │   ├─ 304  → kein Update
  │   └─ Fehler → letztes Bild behalten
  └─ Display hibernate → Deep Sleep N Minuten
```

### Dauerbetrieb (Netzteil)

State-Machine mit 7 Zuständen:
`BOOT → WIFI_CONNECTING → NTP_SYNCING → IDLE → DOWNLOADING → UPDATING_DISPLAY → ERROR_RECOVERY`

---

## Bibliotheken installieren

Arduino IDE → Sketch → Bibliotheken einbinden → Bibliotheken verwalten

| Bibliothek | Autor | Suchbegriff |
|---|---|---|
| GxEPD2 | ZinggJM | `GxEPD2` |
| Adafruit GFX Library | Adafruit | `Adafruit GFX` |
| PNGdec | Larry Bank | `PNGdec` |

WiFi, HTTPClient, LittleFS und ArduinoOTA sind im ESP32-Board-Paket enthalten.

---

## Gerät konfigurieren

Alle Einstellungen in `config.h`.

### Schritt 1 — Geräteblock auswählen

```cpp
#define DEVICE_NAME    "DoorSign-Brandenburg"
#define ROOM_NAME      "Brandenburg"
#define IMAGE_URL      "https://shelf.example.com/348896.bmp"
#define WIFI_SSID      "MeinNetzwerk"
#define WIFI_PASSWORD  "MeinPasswort"
```

### Schritt 2 — Display-Version

```cpp
#define DISPLAY_TYPE  DISPLAY_V1   // V1: 640×384, S/W
// #define DISPLAY_TYPE  DISPLAY_V2   // V2: 800×480, S/W
// #define DISPLAY_TYPE  DISPLAY_V3   // V3: 800×480, S/W/Rot
```
Auflösung und Treiber werden automatisch aus dieser Einstellung abgeleitet.

### Schritt 3 — Betriebsmodus

```cpp
#define DEEP_SLEEP_ENABLED  1   // 1 = Akku, 0 = Netzteil
```

### Schritt 4 — Update-Intervall

```cpp
#define UPDATE_INTERVAL_SEC  (20UL * 60UL)  // 20 Minuten (Standard)
// Empfehlungen:
//   Netzteil:  5UL * 60UL
//   Akku:     20UL * 60UL
//   Akku:     30UL * 60UL
```

Aktivfenster minutengenau in `config.h`:
```cpp
#define ACTIVE_START_MIN  (7 * 60 + 55)   // 07:55 Uhr
#define ACTIVE_END_MIN    (17 * 60 + 0)   // 17:00 Uhr (letzter Abgleich)
```

### Schritt 5 — Bildformat

```cpp
#define IMAGE_FORMAT_PNG  1   // PNG aktiv (Standard)
#define IMAGE_FORMAT_BMP  0   // oder umgekehrt für BMP
```

### Schritt 6 — Bildqualität (nur PNG)

```cpp
#define IMG_GRAY_THRESHOLD  180  // 128=Standard, 180=empfohlen
```

---

## Kompilieren und Flashen

### Board-Einstellungen

| Einstellung | Wert |
|---|---|
| Board | ESP32 Dev Module |
| Upload Speed | 921600 |
| Flash Size | 4MB (32Mb) |
| Partition Scheme | **Minimal SPIFFS (1.9MB APP with OTA)** |
| PSRAM | Disabled |

> **Wichtig:** Standard-Partition „Default 4MB with spiffs" hat nur 1,25 MB für die App
> — reicht NICHT mit OTA. Falls Partition nicht umstellbar: `#define OTA_ENABLED 0`.

### Flash-Vorgang

1. `config.h` öffnen → Gerät, Display, Modus konfigurieren
2. Sketch → Hochladen (`Cmd+U` / `Ctrl+U`)
3. Seriellen Monitor öffnen: 115200 Baud
4. Startsequenz beobachten

### Mehrere Geräte

`config.h` vor jedem Flash anpassen — Firmware ist identisch.

---

## Betriebsmodi

### Deep Sleep — `DEEP_SLEEP_ENABLED 1` (Akku)

**Ablauf pro Wake-Up (~12–15 Sekunden):**
1. Boot (~1s)
2. WLAN verbinden (~3–5s)
3. NTP sync (~1s)
4. Bild herunterladen (~2–5s)
5. E-Ink Refresh (~3s V1/V2 / ~7s V3)
6. Deep Sleep

**OTA im Deep-Sleep-Modus:**
Nur beim Kaltstart (Reset oder Stromunterbrechung) — 60s-Fenster.

**Intelligenter Sleep:**
Außerhalb Mo–Fr 07:55–17:00 schläft das Gerät direkt bis zum nächsten Werktag 07:55.

### Dauerbetrieb — `DEEP_SLEEP_ENABLED 0` (Netzteil)

WLAN permanent aktiv, State Machine prüft alle `UPDATE_INTERVAL_SEC` Sekunden.
OTA jederzeit möglich.

---

## Bildformat

### PNG — `IMAGE_FORMAT_PNG 1`

| Eigenschaft | Wert |
|---|---|
| Farbraum | Graustufen oder 1-Bit |
| Typische Dateigröße | 4–30 KB |
| Max. Größe | 200 KB |
| Dekodierung | ~100ms (PNGdec) |
| Graustufen-Wandlung | via `IMG_GRAY_THRESHOLD` |

```python
from PIL import Image, ImageDraw
img = Image.new("1", (800, 480), 1)  # 1-Bit für schärfstes Ergebnis
draw = ImageDraw.Draw(img)
draw.text((10, 10), "Brandenburg", fill=0)
img.save("image.png")
```

### BMP — `IMAGE_FORMAT_BMP 1`

| Eigenschaft | Wert |
|---|---|
| Bit-Tiefe | **1-Bit oder 4-Bit** (16 Farben) |
| Format | Windows BMP, unkomprimiert |
| Typische Dateigröße | 30 KB (1-Bit) bis 192 KB (4-Bit) |
| Max. Größe | 256 KB |
| Dekodierung | ~5–20ms (direktes Bit-Mapping) |

**1-Bit BMP** (klassisch S/W):
```python
img = Image.new("1", (800, 480), 1)
img.save("image.bmp")
```

**4-Bit BMP** (für V3 mit Rot — oder S/W mit Anti-Aliasing):
```python
# 4-Bit Palette mit Schwarz, Weiß, Rot
palette = [0,0,0, 255,255,255, 255,0,0] + [0,0,0]*13
img = Image.new("P", (800, 480), 1)
img.putpalette(palette)
# ... draw ...
img.save("image.bmp")
```

Der ESP32 erkennt anhand der Palette automatisch:
- **Helligkeit < 128** → schwarzer Pixel
- **R≥150 G<120 B<120** → roter Pixel (nur V3, sonst → schwarz)
- **sonst** → weißer Pixel

### ETag-Unterstützung (empfohlen)

Server sollte `ETag`-Header liefern. Bei unverändertem Bild: `304 Not Modified`
→ kein Download, kein Display-Refresh.

```nginx
location /api/room/ {
    etag on;
    add_header Cache-Control "no-cache";
}
```

---

## Display-Versionen V1 / V2 / V3

| Version | Modell | Auflösung | Farben | Refresh |
|---|---|---|---|---|
| **V1** | GDEW075T8 | 640×384 | S/W | ~3s |
| **V2** | GDEW075T7 | 800×480 | S/W | ~3,5s |
| **V3** | GDEW075Z90 | 800×480 | S/W/**Rot** | ~7s |

In `config.h` umstellen — eine Zeile:
```cpp
#define DISPLAY_TYPE  DISPLAY_V2
```

### V3 — Rot-Kanal nutzen

Der Renderer nutzt automatisch den Rot-Kanal des V3-Displays, wenn der Server
eine 4-Bit-BMP mit roten Palette-Einträgen liefert (R≥150, G<120, B<120).
Auf V1/V2 wird Rot zu Schwarz konvertiert — gleiches Bild funktioniert auf allen
Displays.

---

## Stromversorgung

### Netzteil-Betrieb
USB-C 5V/1A direkt am Board. Kein zusätzliches Modul nötig.

### Akku-Betrieb — Referenzaufbau

Über die Laufzeit entscheidet **nicht** der ESP32, sondern der Ruhestrom der
Stromversorgung. Der ESP32 zieht im Deep Sleep rund 0,01 mA — ein ungeeignetes
Wandler-Modul zieht dauerhaft einige Hundert Mal so viel und bestimmt die Laufzeit
damit praktisch allein. Der folgende Aufbau ist gemessen und im Dauereinsatz:

```
USB-C --> TC4056 --> LiPo-Zelle --> TPS62827 Buck --> 3,3 V --> ESP32 3V3-Pin
          Lademodul  3,7 V          3,3 V / 2 A       |
          1 A        3500 mAh       Iq ~4 µA          +-- 1500 µF Elko gegen GND
```

| Komponente | Typ | Warum dieser |
|---|---|---|
| Lademodul | TC4056, USB-C, 1 A | Ladeschluss 4,2 V, Tiefentladeschutz integriert |
| Zelle | LiPo 3,7 V — hier EEMB LP104567, 3500 mAh | Kapazität nach gewünschter Laufzeit wählen |
| Spannungswandler | **TPS62827** Buck 3,3 V / 2 A (Adafruit-Produkt 4920) | Ruhestrom ~4 µA, liefert 3,3 V direkt |
| Pufferelko | 1500 µF zwischen 3V3 und GND am ESP32 | fängt die WLAN-Stromspitzen (~500 mA) ab |

**Die 3,3 V gehen auf den `3V3`-Pin, nicht auf `VIN`.** Das ist der zweite Teil des
Gewinns: über `VIN` liefe der Strom durch den AMS1117-LDO des Boards, der die
Spannungsdifferenz verheizt und selbst Ruhestrom zieht. Bei Einspeisung auf `3V3`
ist er umgangen.

#### Ersatzteile auswählen

Der TPS62827 ist nicht zwingend — das Auswahlkriterium zählt mehr als das konkrete
Modul:

- **Ruhestrom (Iq) im einstelligen µA-Bereich.** Der wichtigste Wert überhaupt. Er
  steht im Datenblatt des Wandler-Chips, nicht in der Shop-Beschreibung — deshalb
  immer nach dem Chip-Aufdruck gehen.
- **3,3 V Ausgang direkt**, kein Umweg über 5 V und den Board-LDO.
- **Eingang LiPo-tauglich** (rund 3,0–4,2 V).

Geeignet sind z. B. TPS62827, TPS62840 oder TPS63802. **Ungeeignet sind die
verbreiteten LM2596-Module**, oft als „3 A Step-Down“ verkauft und an der Angabe
*150 kHz Schaltfrequenz* zu erkennen: Ihr Eigenverbrauch liegt bei ~5 mA — das
Fünfhundertfache dessen, was der schlafende ESP32 braucht.

#### Was *nicht* nötig ist

- **Kein Auslöten** von CP2102, Power-LED oder AMS1117 auf dem Waveshare-Board. Das
  Board bleibt unverändert: Der LDO ist durch die 3V3-Einspeisung umgangen, und der
  CP2102 hängt an USB-5 V, die im Akkubetrieb gar nicht anliegt.
- **Kein Boost-Konverter.** Der frühere Weg LiPo → 5 V → Board-LDO → 3,3 V bezahlte
  zwei Wandlungsstufen für nichts.
- **Die Status-LEDs des Lademoduls** dürfen bleiben; sie fallen gegenüber dem
  Gesamtruhestrom nicht ins Gewicht.

#### Grenze nach unten

Der TPS62827 ist für 3,4–5,5 V Eingang spezifiziert. Fällt die Zellspannung
darunter, geht er in Durchgang und der Ausgang folgt der Akkuspannung. Praktisch
ist das unkritisch: Das Türschild scheitert vorher am WLAN-Sendebetrieb, der
stabile ~3,0 V braucht — lange bevor der Tiefentladeschutz des TC4056 bei 2,5 V
greift. Nutzbar ist die Zelle daher bis rund 3,0 V, nicht bis zur Schutzabschaltung.

### Brownout-Diagnose

Symptom `E BOD: Brownout detector was triggered` im seriellen Monitor →
Stromversorgung kann WLAN-Spitzen (~500 mA) nicht liefern. Lösung: Pufferelko
prüfen bzw. vergrößern, oder stabileres Netzteil.

---

## Energieverbrauch und Akku-Laufzeit

Gemessene und gerechnete Werte sind hier bewusst getrennt.

### Gemessen

Am Referenzaufbau oben, Stand 18.09.2026:

| | früherer Aufbau | Referenzaufbau |
|---|---|---|
| Topologie | LiPo → Boost 5 V → Board-LDO → 3,3 V | LiPo → Buck → 3,3 V auf `3V3` |
| Wandler | LM2596 (150 kHz) | TPS62827 (2,2 MHz, Iq ~4 µA) |
| **Ruhestrom im Deep Sleep** | **~12,5 mA** | **~0,4 mA** |
| Erreichte Laufzeit (3500 mAh) | ~3 Wochen | **≥65 Tage — Test läuft** |

> **Messmittel:** Multimeter im mA-Bereich. Es erfasst den Mittelwert; kurze
> Stromspitzen im Schlaf bleiben unsichtbar. Für die Größenordnung ausreichend,
> für eine Optimierung unterhalb von 0,4 mA nicht.
>
> **Die Laufzeitangabe ist eine laufende Beobachtung** seit 15.07.2026, kein
> abgeschlossener Test. Sie wird fortgeschrieben, sobald die Zelle leer ist.
>
> **Zum alten Aufbau:** Messwert (12,5 mA) und beobachtete Laufzeit (~3 Wochen)
> passen nicht exakt zusammen — aus 3 Wochen folgt ein mittlerer Strom von eher
> ~6 mA. Der höhere Messwert war vermutlich eine nicht repräsentative
> Momentaufnahme. Die Größenordnung — Ruhestrom im Milliampere-Bereich statt im
> Mikroampere-Bereich — ist in beiden Lesarten dieselbe.

### Verbrauch pro Wake-Up-Zyklus

| Phase | Dauer | Strom |
|---|---|---|
| Boot + WLAN (Schnellverbindung aus RTC-Cache) | ~3 s | ~120 mA |
| Download bzw. 304-Abgleich | ~0,5–2 s | ~120 mA |
| BMP/PNG Decode | ~0,01–0,1 s | 80 mA |
| E-Ink Refresh (nur bei geändertem Bild) | 3–7 s | 20 mA |
| Deep Sleep (ESP32 allein) | Rest der Zeit | ~0,01 mA |

Ein Wake kostet damit rund **0,2 mAh**.

### Hochrechnung

Annahmen: 20-Minuten-Intervall, Aktivfenster Mo–Fr 07:55–17:00 (≈20 Wakes pro Tag
im Sieben-Tage-Mittel), 0,2 mAh je Wake, Zelle 3500 mAh mit ~90 % nutzbarer
Kapazität.

| Posten | Rechnung | mAh/Tag |
|---|---|---|
| Ruhestrom | 0,4 mA × 24 h | 9,6 |
| Wake-Zyklen | ~20 × 0,2 mAh | ~4,0 |
| **Summe** | | **~13,6** |

→ rund **230 Tage, also 6–8 Monate**.

Wirkung des Update-Intervalls bei sonst gleichen Annahmen:

| Intervall | Wakes/Tag | mAh/Tag | Laufzeit |
|---|---|---|---|
| 5 Min | ~78 | 25,2 | ~125 Tage |
| 10 Min | ~39 | 17,4 | ~180 Tage |
| **20 Min** (Standard) | **~20** | **13,6** | **~230 Tage** |
| 30 Min | ~13 | 12,2 | ~260 Tage |
| Dauerbetrieb | — | — | ~24 h |

**Den Ausschlag gibt der Ruhestrom, nicht das Intervall.** Von 20 auf 30 Minuten
gewinnt man 30 Tage und verliert Aktualität; von 20 auf 5 Minuten verliert man über
100 Tage. Jenseits von 20 Minuten läuft die Rechnung gegen die 9,6 mAh Grundlast,
die vom Intervall unberührt bleibt — der Hebel für längere Laufzeit liegt deshalb
in der Hardware, nicht in der Konfiguration.

> Eine frühere Fassung dieses Abschnitts nannte bis zu „760 Tage bei 30 Minuten“.
> Sie setzte 0,01 mA Ruhestrom an — also den ESP32 allein, ohne die Peripherie —
> und lag dadurch um rund eine Größenordnung zu hoch.

### PNG vs BMP

Praktisch identisch — Unterschied < 2% der Laufzeit.
Format-Wahl nach Server-Komfort, nicht nach Akku-Verbrauch.

---

## OTA-Updates

### Netzteil-Betrieb
Jederzeit aktiv. Arduino IDE → Werkzeuge → Port → Netzwerk-Port wählen.

### Akku-Betrieb (Deep Sleep)
1. Reset-Taste oder Strom kurz trennen
2. Startbildschirm erscheint → **60-Sekunden-OTA-Fenster**
3. Arduino IDE → Port → Netzwerk-Port → Hochladen
4. OTA-Passwort eingeben (`OTA_PASSWORD` in `config.h`)

OTA deaktivieren: `#define OTA_ENABLED 0`

---

## Fehlersuche

Serieller Monitor: **115200 Baud**

### Häufige Meldungen

| Meldung | Bedeutung | Lösung |
|---|---|---|
| `HTTP Response: 304` | Bild unverändert — normal | — |
| `HTTP-Fehler: -1` | Server nicht erreichbar | URL + WLAN prüfen |
| `Falsche Dimensionen` | Server-Auflösung passt nicht | `DISPLAY_TYPE` prüfen |
| `validateBmp: Bit-Tiefe X (erwartet 1 oder 4)` | BMP nicht unterstützt | Server auf 1- oder 4-Bit umstellen |
| `Datei zu gross` | > `IMG_MAX_BYTES` | URL prüfen / Server-Output verkleinern |
| `Sketch zu groß` | Falsche Partition | Minimal SPIFFS wählen |
| `Brownout detector triggered` | Stromversorgung schwach | Elko + stärkeres Netzteil |

### Reset-Ursachen

| Reset | Bedeutung |
|---|---|
| `rst:0x5 (DEEPSLEEP_RESET)` | Aufwachen aus Deep Sleep — **normal** |
| `rst:0x1 (POWERON_RESET)` | Kaltstart — normal |
| `rst:0x8 (TG1WDT_SYS_RESET)` | Watchdog — Problem |
| `rst:0xc (SW_CPU_RESET)` | Software-Reset / OTA |

### Display zeigt nichts

- `_Update_Full` Wert prüfen — V1 ~2,6s, V2 ~3,5s, V3 ~7s
- Werte deutlich kleiner → falscher `DISPLAY_TYPE`
- Server-Auflösung muss zur Display-Version passen

---

## Projektstruktur

```
DoorSign/
├── DoorSign.ino              # Hauptdatei (setup/loop)
├── config.h                  # ⚙️ ALLE Einstellungen
├── Logger.h                  # Logging
├── WifiManager.h/.cpp        # WLAN mit Auto-Reconnect
├── TimeManager.h/.cpp        # NTP + Europe/Berlin
├── ImageManager.h/.cpp       # HTTP, ETag, PNG+BMP Validierung
├── DisplayManager.h/.cpp     # GxEPD2, V1/V2/V3, PNG+BMP Renderer
├── StateMachine.h/.cpp       # Zustandsautomat (Dauerbetrieb)
├── DeepSleepManager.h/.cpp   # Deep Sleep + intelligenter Wakeup
├── README.md                 # Diese Datei
├── CHANGELOG.md              # Versionshistorie
└── GITHUB_ANLEITUNG.md       # Schritt-für-Schritt Git-Setup

```

---

## Bekannte Einschränkungen

- **HTTPS:** Kein Zertifikats-Check (`setInsecure()`). Für öffentliche Server CA hinterlegen.
- **PNG:** Schwarzweiß-Ausgabe. Farb-PNGs werden via Schwellwert gewandelt — Rot-Kanal aus PNG wird derzeit nicht für V3 erkannt (nur 4-Bit BMP).
- **BMP:** 1-Bit oder 4-Bit Palette, unkomprimiert. 8/24-Bit BMP werden abgelehnt.
- **OTA im Deep-Sleep:** Nur 60s nach Kaltstart.
- **WLAN:** Kein 802.1X/RADIUS (Enterprise-WLAN).
- **Zwei Geräte:** Identische Firmware, `config.h` pro Flash anpassen.
- **Partition:** Erfordert „Minimal SPIFFS" oder OTA deaktivieren.
- **Akku-Messwerte:** Ruhestrom mit einem Multimeter ermittelt (Mittelwert, keine Spitzenerfassung); die Laufzeitangabe ist eine laufende Beobachtung, kein abgeschlossener Entladetest.
- **Keine Akkuüberwachung:** Die Firmware misst die Zellspannung nicht und meldet keinen niedrigen Ladestand — das Schild fällt am Ende ohne Vorwarnung aus.

---

## Lizenz

MIT License frei verwendbar für private und kommerzielle Projekte.
