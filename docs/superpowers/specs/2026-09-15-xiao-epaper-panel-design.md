# Design Spec: Seeed Studio XIAO 7.5" ePaper Panel mit ESPHome und Home Assistant

- **Datum**: 15.09.2026
- **Status**: Genehmigt
- **Ziel**: Autonomes, batteriebetriebenes Smart-Home-Display zur Visualisierung von Solarerzeugung, Hausverbrauch, Netzbilanz und Wetterdaten.

---

## 1. Hardware & Systemarchitektur

### Hardwarekomponenten
- **Gerät**: Seeed Studio XIAO 7.5" ePaper Panel
- **Mikrocontroller**: Seeed Studio XIAO ESP32-C3
  - Prozessor: 32-Bit RISC-V Single-Core @ 160 MHz
  - Flash: 4 MB
  - RAM: 400 KB SRAM
  - Konnektivität: 2.4 GHz Wi-Fi 802.11 b/g/n, BLE 5.0
- **Display**:
  - Typ: Waveshare 7.5" Monochrom E-Ink / ePaper
  - Auflösung: 800 × 480 Pixel (16:9 bzw. 5:3 Aspekt)
  - Controller: UC8179 (unterstützt über ESPHome `waveshare_epaper`, Modell `7.50inv2`)
- **Energieversorgung**:
  - Integrierter 2000 mAh Li-Ion Akku
  - Ladung und Programmierung über USB Type-C

### Pinbelegung (SPI & Steuersignale)
| Signal | XIAO Pin | ESP32-C3 GPIO | Funktion |
| :--- | :--- | :--- | :--- |
| **SCK / CLK** | D8 | GPIO8 | SPI Clock |
| **MOSI / DIN** | D10 | GPIO10 | SPI Master Out Slave In |
| **CS** | D3 | GPIO3 | Chip Select (Active Low) |
| **DC** | D5 | GPIO5 | Data / Command Control |
| **RST** | D2 | GPIO2 | Display Reset |
| **BUSY** | D4 | GPIO4 | Busy Signal (`inverted: true`) |

---

## 2. Energie- und Betriebsmanagement (Deep Sleep)

Um mit dem 2000 mAh Akku eine Laufzeit von mehreren Wochen bis Monaten zu erreichen:
1. **Zyklus**: 15 Minuten Deep Sleep (`deep_sleep.sleep_duration: 15min`).
2. **Wachphase**:
   - Schnelle WLAN-Verbindung mit statischer IP oder schnellem DHCP (`fast_connect: true`).
   - Verbindung zur Home Assistant API (`homeassistant`).
   - Sobald die Sensoren valide Daten liefern (`wifi.connected` && HA-Sensoren aktualisiert), wird die Display-Aktualisierung getriggert (`component.update: epaper_display`).
   - Nach erfolgreichem Rendern und ePaper-Refresh geht der Controller direkt wieder in den Tiefschlaf.
3. **Safety Fallback**:
   - Maximal 45 Sekunden Gesamtwachzeit (`run_duration: 45s`). Falls Home Assistant oder das WLAN temporär nicht erreichbar sind, schläft das Panel trotzdem ein, um den Akku vor Tiefentladung zu schützen.
4. **OTA-Sicherheit**:
   - Vor Firmware-Updates über WLAN kann per Home Assistant Switch / Service der Deep Sleep verhindert werden, oder das Flashen erfolgt direkt über USB-C.

---

## 3. Datenfluss & Home Assistant Integration

### Verwendete Home Assistant Entitäten
| Kategorie | Entity-ID | Attribut | Einheit | Beschreibung |
| :--- | :--- | :--- | :--- | :--- |
| **Solar Leistung** | `sensor.core2_solar_live` | - | W | Aktuelle Solarerzeugung |
| **Solar Tag** | `sensor.core2_solar_day_energy_kwh` | - | kWh | Heutige Gesamterzeugung |
| **Haus Leistung** | `sensor.core2_house_live` | - | W | Aktueller Hausverbrauch |
| **Haus Tag** | `sensor.core2_house_day_energy_kwh` | - | kWh | Heutiger Gesamtverbrauch |
| **Netz Import** | `sensor.core2_grid_import_power` | - | W | Live-Netzbezug |
| **Netz Export** | `sensor.core2_grid_export_power` | - | W | Live-Netzeinspeisung |
| **Wetter Zustand** | `weather.home` | state | - | Wetterlage (`sunny`, `cloudy`, etc.) |
| **Wetter Temp** | `weather.home` | `temperature` | °C | Außentemperatur |
| **Wetter Feuchte** | `weather.home` | `humidity` | % | Relative Luftfeuchtigkeit |
| **Wetter Wind** | `weather.home` | `wind_speed` | km/h | Windgeschwindigkeit |
| **Wetter Druck** | `weather.home` | `pressure` | hPa | Luftdruck |
| **Zeit** | `time.homeassistant` | - | - | Synchronisierte Uhrzeit & Datum |

---

## 4. UI / Display-Layout (800 × 480 px)

Das Display ist vertikal in zwei Zonen unterteilt (Trennlinie bei X = 410):

### Kopfzeile (Y: 0 – 45)
- Links: `"SMART HOME ENERGIE"`
- Rechts: Wochentag, Datum und Uhrzeit (z. B. `"Dienstag, 15.09.2026 - 14:30"`)

### Linke Spalte (X: 10 – 400): Energie-Kacheln
- **Kachel 1 (Solarerzeugung)**:
  - Header: Symbol ☀️ + Label `"SOLARERZEUGUNG"`
  - Hauptwert: Aktuelle Leistung (z. B. `532 W`)
  - Subwert: `"Heute: 0.84 kWh"`
- **Kachel 2 (Hausverbrauch)**:
  - Header: Symbol 🏠 + Label `"HAUSVERBRAUCH"`
  - Hauptwert: Aktuelle Leistung (z. B. `296 W`)
  - Subwert: `"Heute: 6.46 kWh"`
- **Kachel 3 (Netzbilanz)**:
  - Dynamischer Status:
    - Bei Einspeisung: `+236 W Einspeisung`
    - Bei Bezug: `-120 W Netzbezug`
  - Subwert: Tagesbilanz (Export/Import)

### Rechte Spalte (X: 420 – 790): Wetter & Systemstatus
- **Wetter-Hauptanzeige**:
  - Großes MDI Wetter-Icon (Sonne, Wolken, Regen, etc.)
  - Große Temperaturanzeige (z. B. `21.8 °C`)
  - Deutscher Zustandstext (`"Sonnig"`, `"Leicht bewölkt"`, `"Regnerisch"`, etc.)
- **Wetter-Details**:
  - Luftfeuchtigkeit (`💧 45 %`)
  - Windgeschwindigkeit (`💨 12 km/h`)
  - Luftdruck (`⏲ 1018 hPa`)
- **System-Statusbereich**:
  - Zeitpunkt des letzten erfolgreichen Syncs
  - WLAN-Empfangsstärke (RSSI)
  - Hinweis auf 15-Minuten-Aktualisierungsintervall
