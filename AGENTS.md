# AGENTS.md

High-signal context for AI agents working on `ESPHome-XIAO-7.5-ePaper-Panel`.

## Commands

- **Config Validation**: `esphome config seeed-xiao-esp32c3.yaml`
- **Compile Firmware**: `esphome compile seeed-xiao-esp32c3.yaml`
- **Flashen per USB**: `esphome run seeed-xiao-esp32c3.yaml --device <COMx>`
- **Flashen per WLAN (OTA)**: `esphome run seeed-xiao-esp32c3.yaml` (Nur wenn das Panel wach ist bzw. `Prevent Deep Sleep` aktiv ist)
- **Live Logs**: `esphome logs seeed-xiao-esp32c3.yaml`

---

## Architecture & Hardware Specs

- **Gerät**: Seeed Studio XIAO 7.5" ePaper Panel (800 × 480 Pixel, Monochrom E-Ink).
- **MCU**: Seeed Studio XIAO ESP32-C3
  - Board-Typ: `esp32-c3-devkitm-1`
  - Framework: `arduino`
- **Display-Komponente**:
  - Plattform: `waveshare_epaper`
  - Modell: `7.50inv2`
  - Auflösung: 800 × 480 Pixel (Landscape)
- **SPI-Verdrahtung**:
  - SCK / CLK: `GPIO8`
  - MOSI / DIN: `GPIO10`
  - CS: `GPIO3`
  - DC: `GPIO5`
  - RST: `GPIO2`
  - BUSY: `GPIO4` (`inverted: true`)
- **Hinweis zu Strapping Pins**: ESPHome gibt beim Kompilieren Warnungen bezueglich GPIO8 und GPIO2 aus. Dies sind hardwareseitig festgelegte Strapping Pins des ESP32-C3, die auf diesem Seeed Studio Board exakt so verschaltet sind.

---

## Energie-Management & Deep Sleep

- **Akkubetrieb**: Das Panel laeuft mit dem integrierten 2000 mAh Li-Ion Akku.
- **Akku-Ladestands-Messung**:
  - Auf diesem Board existiert ab Werk kein ADC-Spannungsteiler vom Akku zum ESP32-C3.
  - Die Firmware nutzt einen RTC-persistierten Zyklencounter (`battery_cycles`, Basis: 2800 Zyklen à 15 Minuten = ~4 Wochen).
  - Der Ladestand wird in `%` auf dem ePaper angezeigt und als `sensor.battery_level` nach Home Assistant exponiert.
  - Reset nach Vollaufladung via HA-Button `button.reset_battery_100`.
- **Deep-Sleep-Zyklus**:
  - Schlafdauer: 15 Minuten (`sleep_duration: 15min`).
  - Max. Wachdauer: 45 Sekunden (`run_duration: 45s`).
  - Das Panel wacht auf, verbindet sich per WLAN mit Home Assistant, wartet auf valide Sensordaten, triggert `component.update: epaper_display` und geht nach 8 Sekunden Pufferzeit sofort wieder in den Tiefschlaf.
- **Wartungsmodus fuer OTA (WLAN-Updates)**:
  - In Home Assistant wird ein Schalter `switch.prevent_deep_sleep` exponiert.
  - Wenn dieser Schalter eingeschaltet ist, bleibt der ESP32 nach dem Aufwachen dauerhaft online, sodass OTA-Firmware-Updates problemlos uebertragen werden koennen.

---

## Home Assistant Entitäten & Datenquellen

Die Firmware nutzt die native Home Assistant API (`homeassistant` Plattform in ESPHome):

- **Solar**:
  - `sensor.core2_solar_live` (W, aktuelle PV-Erzeugung)
  - `sensor.core2_solar_day_energy_kwh` (kWh, heutiger Solarertrag)
- **Hausverbrauch**:
  - `sensor.core2_house_live` (W, aktueller Hausverbrauch)
  - `sensor.core2_house_day_energy_kwh` (kWh, heutiger Hausverbrauch)
- **Netz**:
  - `sensor.core2_grid_import_power` (W, Netzbezug)
  - `sensor.core2_grid_export_power` (W, Netzeinspeisung)
- **Wetter**:
  - `weather.home` (Zustand als `text_sensor`, Attribute `temperature`, `humidity`, `wind_speed`, `pressure` als `sensor`).
- **Uhrzeit & Datum**:
  - `time.homeassistant` fuer Datum, Wochentag und Uhrzeit im Header.

---

## ESPHome Eigenheiten & Best Practices

1. **Glyphen in Fonts**:
   - ESPHome laedt standardmaessig nur ASCII-Zeichen.
   - Fuer deutsche Umlaute (`äöüÄÖÜß`) und das Gradzeichen (`°`) muessen diese explizit in `glyphs:` definiert sein.
   - **Achtung**: Doppelte Glyphen in einer `glyphs:`-Definition fuehren ab ESPHome 2024/2026 zu einem Konfigurationsfehler (`Found duplicate glyph`).
2. **Secrets**:
   - `secrets.yaml` enthaelt WLAN-Zugangsdaten und API-Schluessel und darf niemals committet werden (`.gitignore`).
   - `secrets.yaml.example` dient als Vorlage fuer neue Umgebungen.

---

## Lessons Learned & Hardware-Analysen

### 1. Schaltplan-Fakten zum Batterie-Monitoring (ePaper Driver Board v1.0)
- **Analyse der offiziellen Schematic (`ePaper_Driver_Board.pdf`)**:
  - Der Li-Ion Akku (`BAT_4V2`) ist ausschließlich an den Power-Management- und Boost-IC **ETA9740E8A** (`U1`) angeschlossen.
  - Der ETA9740 erzeugt daraus geregelte 5V (`VDD_5V`), welche über einen LDO auf 3.3V abgesenkt werden.
  - Die Status-LEDs des ETA9740 (`LED1`, `LED2`, `LED3`, `EPD4`) sind als Standalone-LED-Bar verschaltet und besitzen **keine Verbindung** zum Sockel des XIAO ESP32-C3 (`CN4`).
  - **Ergebnis**: Auf dem unmodifizierten Board existiert **keine physische Leiterbahn** zwischen Akkuspannung und den Analogeingängen des XIAO. Der ESP32-C3 "sieht" immer nur die geregelte Betriebsspannung.
- **Community-Konsens (Seeed Forum Thread #292932 & Wiki Discussions #69)**:
  - Ein direktes Auslesen per ADC erfordert das manuelle Einlöten eines 2:1 Spannungsteilers (z. B. 2× 100 kΩ) zwischen den Akkupads und einem freien ADC-Pin (oder den Tausch auf ein anderes Board).
  - Ohne Lötarbeiten ist die **RTC-Zyklen-Schätzung** (`battery_cycles`) die robusteste und genaueste Methode, um den Ladestand auf dem Display und in Home Assistant bereitzustellen.

### 2. Windows Defender Application Control (WDAC / Device Guard) & Toolchains
- **Lokaler CLI-Build unter Windows**:
  - Auf Windows-Systemen mit aktiver Device Guard- / WDAC-Richtlinie (Constrained Language Mode) werden frisch von PlatformIO heruntergeladene RISC-V-Cross-Compiler (`riscv32-esp-elf-ar.exe`, `gcc`, etc.) blockiert (`FAILED: code=4551`).
  - **Lösung / Best Practice**: Drahtlose OTA-Updates primär über das **Home Assistant ESPHome Add-on** (Weg 1) ausführen. Dort läuft der Build innerhalb eines Linux-Docker-Containers ohne Windows-Richtlinienbeschränkungen.

### 3. Exakte Secrets-Namenskonvention
- Das ESPHome Add-on in Home Assistant vergibt beim Anlegen gerätespezifische Secrets mit Präfix:
  - `seeed_xiao_esp32c3__encryption_key`
  - `seeed_xiao_esp32c3__ota_password`
  - `seeed_xiao_esp32c3__ap_password`
- Lokale YAML-Dateien müssen exakt diese Schlüssel verwenden, damit Copy-Paste in das Home Assistant Dashboard ohne manuelle Anpassung der Secrets-Namen funktioniert.

### 4. Build-Absturz durch Out-of-Memory (cc1plus Killed / Signal terminated)
- **Problem**: Bei Hosts mit begrenztem Arbeitsspeicher (z. B. Raspberry Pi 3/4, HA Green oder Docker-Container ohne ausreichend Swap) bricht der Compiler mit folgendem Fehler ab:
  ```text
  riscv32-esp-elf-g++: fatal error: Killed signal terminated program cc1plus
  compilation terminated.
  ninja: build stopped: subcommand failed.
  ```
- **Ursache**: Ninja startet standardmäßig so viele parallele C++-Compilerprozesse wie CPU-Kerne vorhanden sind. Da jeder `riscv32-esp-elf-g++`-Prozess mehrere hundert Megabyte RAM benötigt, greift der Linux OOM-Killer (`SIGKILL`) ein.
- **Lösung**: Begrenzung der parallelen Compiler-Prozesse auf `1` direkt im YAML-Header via:
  ```yaml
  esphome:
    name: ${name}
    friendly_name: ${friendly_name}
    compile_process_limit: 1
  ```
  Zusätzlich kann im Home Assistant ESPHome Add-on unter *Konfiguration* der Wert `default_compile_process_limit: 1` gesetzt werden.

---

## Relevante Skills & Werkzeuge für Agenten

Für Arbeiten an diesem Projekt sollten Agenten folgende Domänen-Skills und Werkzeuge gezielt heranziehen:

### 1. Home Assistant Skill / REST API
- **Einsatzbereich**: Abfragen von Entitätszuständen (`/api/states`), Attributen und Verifizieren verfügbarer Sensoren (`weather.*`, `sensor.core2_*`).
- **Best Practice**: Bevor neue Sensoren in `seeed-xiao-esp32c3.yaml` referenziert werden, immer per REST API prüfen, ob die Entität in Home Assistant tatsächlich existiert, aktiv ist und Daten liefert.

### 2. ESPHome Skill / CLI
- **Einsatzbereich**: 
  - Validierung von Konfigurationsdateien (`esphome config seeed-xiao-esp32c3.yaml`).
  - C++ Lambda-Entwicklung für ePaper-Rendering (`it.print`, `it.rectangle`, `it.filled_rectangle`, `it.strftime`, `TextAlign`).
  - Deep-Sleep-Orchestrierung (`deep_sleep:`, RTC-Variablen mit `restore_value: yes`, `prevent_deep_sleep` Schalter).
  - Schriftarten- und Glyphen-Management (`gfonts://`, UTF-8 Glyphendeklaration, zwingende Vermeidung doppelter Glyphen).
  - Build-Ressourcen-Steuerung (`compile_process_limit: 1` gegen OOM-Kills).

### 3. esptool / Serial Recovery Skill
- **Einsatzbereich**:
  - Direktes Flashen über USB/Seriell (`esptool.py` oder `esphome run seeed-xiao-esp32c3.yaml --device <COMx>`).
  - Wiederherstellung bei fehlerhafter Firmware, Bootloops oder unresponsiven Deep-Sleep-Zuständen.
  - **Hardware-Bootloader-Einstieg am Panel**:
    1. USB-C-Kabel mit dem PC verbinden.
    2. **BOOT-Taste** (hinter dem Ausklappständer) gedrückt halten.
    3. **RESET-Taste** einmal kurz drücken.
    4. BOOT-Taste loslassen.
    5. Der ESP32-C3 befindet sich nun im ROM-Download-Modus und kann zuverlässig geflasht werden.
  - **Nützliche Befehle**:
    - Chip-Erkennung & Port-Test: `esptool.py chip_id`
    - Flash-Speicher löschen: `esptool.py erase_flash`
    - Flash-Spezifikationen prüfen: `esptool.py flash_id`
