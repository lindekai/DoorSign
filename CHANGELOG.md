# Changelog — DoorSign

## Version 1.3 — Energie / Batterielaufzeit

### Firmware-Optimierungen (Deep-Sleep-Pfad)
- **WLAN-Schnellverbindung** — BSSID + Kanal des APs werden im RTC-RAM
  gecacht; beim Aufwachen verbindet der ESP32 ohne AP-Scan (`WiFi.begin`
  mit Kanal/BSSID). Fällt bei Fehlschlag sauber auf den normalen
  Scan-Connect zurück. Spart ~2–3 s Funkzeit pro Wake
  (`WifiManager`, `DeepSleepManager`, `DoorSign.ino`)
- **NTP nur ~1×/Tag** — die ESP32-RTC hält die Zeit über den Deep Sleep;
  NTP wird nur re-synchronisiert, wenn `NTP_RESYNC_INTERVAL_SEC` (24 h)
  überschritten ist oder die Zeit ungültig ist. Spart ~1 s pro Wake
  (`TimeManager::markSyncedIfValid`, `DoorSign.ino`)
- **Update-Intervall auf 20 min** (`config.h`)
- **Minutengenaues Aktivfenster** — `ACTIVE_START_MIN`/`ACTIVE_END_MIN`
  statt voller Stunden; Standard **07:55–17:00** Mo–Fr (Schild ist um 08:00
  aktuell, letzter Abgleich 17:00 → spart die 17–18-Uhr-Stunde)
- **RTC-Prüfsumme** deckt jetzt alle RTC-Felder ab (nicht nur `bootCount`) —
  korrupter Cache führt zu sicherem Fallback statt Fehlverhalten

### Dokumentation
- **README „Stromversorgung“** als Referenzaufbau neu gefasst: Teileliste mit
  Chipbezeichnungen, Auswahlkriterium für Ersatzmodule (Iq im einstelligen
  µA-Bereich, 3,3 V direkt) und ein Abschnitt, was ausdrücklich *nicht* nötig ist.
- **README „Energieverbrauch“** trennt jetzt gemessene von gerechneten Werten und
  nennt alle Annahmen. Die frühere Hochrechnung (bis „760 Tage“) setzte 0,01 mA
  Ruhestrom an — den ESP32 ohne Peripherie — und lag um rund eine Größenordnung
  zu hoch.

### Hinweis Hardware
- Die Firmware-Hebel dieser Version wirken erst, wenn die Stromversorgung stimmt:
  Den Ruhestrom bestimmt sie, nicht der ESP32. Ein LM2596-Modul (~5 mA) ist durch
  einen Low-Iq-Wandler zu ersetzen (z. B. TPS62827) und die 3,3 V direkt auf den
  `3V3`-Pin zu führen — siehe Referenzaufbau im README.
- **Korrektur gegenüber einer früheren Fassung dieses Eintrags:** Dort stand, die
  Always-on-Chips des Waveshare-Boards (CP2102/LDO/LED) müssten entfernt werden,
  sonst bleibe der Ruhestrom im mA-Bereich. Die Messung widerlegt das: Mit
  TPS62827 und Einspeisung auf `3V3` wurden **0,4 mA am unveränderten Board**
  erreicht (zuvor ~12,5 mA). Der Board-LDO ist durch die 3V3-Einspeisung umgangen,
  und der CP2102 hängt an USB-5 V, die im Akkubetrieb nicht anliegt.

## Version 1.2 — Robustheit

### Behobene Fehler
- **Akku-Schutz bei NTP-Ausfall** — ohne gültige Zeit schläft das Gerät jetzt
  `NTP_FAIL_SLEEP_SEC` (30 min) statt `UPDATE_INTERVAL_SEC`. Verhindert häufiges
  Aufwachen und Batterie-Entleerung bei anhaltendem NTP-Ausfall
  (`DeepSleepManager.cpp`, `config.h`)
- **Abgeschnittene Downloads werden erkannt** — Download-Schleife leert den
  Empfangspuffer nach Verbindungsende, `write()`-Rückgabe wird geprüft, und die
  geschriebene Bytezahl wird gegen `Content-Length` verifiziert. Verhindert, dass
  ein unvollständiges Bild das letzte gute überschreibt (`ImageManager.cpp`)
- **Stärkere PNG-Validierung** — `validatePng()` prüft nun IHDR-Dimensionen und
  den IEND-Chunk am Dateiende (definitiver Trunkierungs-Detektor), nicht nur die
  Signatur (`ImageManager.cpp`)
- **Wirklich atomarer Bild-Austausch** — `rename` wird zuerst versucht, sodass ein
  fehlgeschlagener Tausch das letzte gute Bild nicht mehr vernichten kann
  (`ImageManager.cpp`)

### Härtung
- **Compile-Guards** in `config.h`: ungültiger `DISPLAY_TYPE` oder eine falsche
  Bildformat-Wahl (keins/beide aktiv) bricht jetzt sichtbar zur Compile-Zeit ab

## Version 1.1 (Final)

### Neue Features
- **4-Bit BMP-Support** — Server kann nun auch 4-Bit-Palette-BMPs liefern (16 Farben)
- **V3-Display Rot-Kanal** — bei 4-Bit BMP mit roten Palette-Einträgen wird Rot
  tatsächlich rot dargestellt; auf V1/V2 wird es zu Schwarz konvertiert
- **`IMG_MAX_BYTES` für BMP** auf 256 KB erhöht (vorher 64 KB)

### Verbesserte Validierung
- `validateBmp()` akzeptiert jetzt 1-Bit UND 4-Bit
- Compression-Check (nur unkomprimierte BMPs)

## Version 1.0

### Funktionen
- Zwei Betriebsmodi: Deep Sleep (Akku) und Dauerbetrieb (Netzteil)
- Drei Display-Versionen: V1 (640×384 S/W), V2 (800×480 S/W), V3 (800×480 S/W/Rot)
- Zwei Bildformate: PNG und BMP, umschaltbar via `IMAGE_FORMAT_PNG`/`IMAGE_FORMAT_BMP`
- HTTP ETag/304-Caching für minimalen Display-Refresh
- Intelligenter Sleep außerhalb der Betriebszeiten (Mo–Fr 08:00–18:00)
- LittleFS-Persistenz für letztes Bild + Metadaten
- OTA-Firmware-Updates (60s-Fenster im Sleep-Modus)
- Robustes Wiederherstellen nach Stromausfall

### Behobene Hardware-Probleme während der Entwicklung
- GxEPD2-Header-Pfad: `<epd/...>` (V1/V2) bzw. `<epd3c/...>` (V3) — nicht direkt
- SPI-Pins müssen vor UND nach `_display.init()` gesetzt werden (Waveshare ESP32 Driver Board)
- ArduinoOTA.begin() erfordert aktives WLAN — lazy initialisieren
- Task-Watchdog-Timeout während E-Ink-Refresh erhöhen (60s statt 5s)
- PNG-Decode-Loop: `yield()` alle 32 Zeilen verhindert TG1WDT-Reset
- PNG-Objekt MUSS auf Heap (`new PNG()`, ~15 KB) — Stack hat nur 8 KB
- E-Ink Settle-Zeit zwischen zwei Full-Refreshes beachten
- `drawBitmap()` statt `writeImage()` für korrekte Bit-Polarität
- V1-Display benötigt `GxEPD2_750`, NICHT `GxEPD2_750_T7`
- Partition Scheme „Minimal SPIFFS" für Sketch-Größe + OTA
- Brownout bei WLAN-Start: dicker Pufferelko (1500µF) am ESP32
