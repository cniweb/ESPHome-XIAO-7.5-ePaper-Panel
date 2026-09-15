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
