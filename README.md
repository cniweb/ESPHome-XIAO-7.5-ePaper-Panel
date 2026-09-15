# ESPHome XIAO 7.5" ePaper Panel (Smart Home Dashboard)

Batteriebetriebenes Smart-Home-Statusdisplay auf Basis des **Seeed Studio XIAO 7.5" ePaper Panels** mit **ESPHome** und nativer Anbindung an **Home Assistant**.

Visualisiert live und tagesbasiert:
- **Solarerzeugung** (W & kWh)
- **Hausverbrauch** (W & kWh)
- **Netzbilanz** (Netzeinspeisung / Netzbezug in W)
- **Wetterdaten** (Temperatur in °C, Zustand, Luftfeuchtigkeit, Wind, Luftdruck)
- **Datum, Wochentag & Uhrzeit**
- **WLAN-Signalstärke (RSSI)**

---

## Hardware-Übersicht

- **Gerät**: [Seeed Studio XIAO 7.5" ePaper Panel](https://wiki.seeedstudio.com/xiao_075inch_epaper_panel)
- **MCU**: Seeed Studio XIAO ESP32-C3 (32-bit RISC-V @ 160 MHz, 4MB Flash, Wi-Fi & BLE)
- **Display**: 7.5" Monochromes E-Ink Panel (800 × 480 Pixel, Waveshare `7.50inv2` / UC8179)
- **Akku**: Integrierter 2000 mAh Li-Ion Akku
- **Anschluss**: USB Type-C (Laden & Flashen)

### Pinbelegung (SPI)

| Signal | XIAO Pin | ESP32-C3 GPIO | Funktion |
| :--- | :--- | :--- | :--- |
| **SCK / CLK** | D8 | GPIO8 | SPI Clock |
| **MOSI / DIN** | D10 | GPIO10 | SPI Data Out |
| **CS** | D3 | GPIO3 | Chip Select |
| **DC** | D5 | GPIO5 | Data / Command |
| **RST** | D2 | GPIO2 | Reset |
| **BUSY** | D4 | GPIO4 | Busy (`inverted: true`) |

*Hinweis: ESPHome gibt beim Kompilieren Warnungen zu GPIO8 und GPIO2 aus. Dies sind hardwareseitige Strapping Pins des ESP32-C3, die auf dem Seeed Board herstellerseitig so verschaltet sind.*

---

## Display-Layout (800 × 480 Pixel)

```text
+------------------------------------+-----------------------------------+
|  SMART HOME ENERGIE                |  Dienstag, 15.09.2026  -  14:30   |
+------------------------------------+-----------------------------------+
| [SOLARERZEUGUNG]                   | [WETTER (HOME ASSISTANT)]         |
|   532 W                            |   21.8 °C                         |
|   Heute: 0.84 kWh                  |   Leicht bewölkt                  |
+------------------------------------+-----------------------------------+
| [HAUSVERBRAUCH]                    |   Luftfeuchtigkeit: 45 %          |
|   296 W                            |   Wind:             12.0 km/h     |
|   Heute: 6.46 kWh                  |   Luftdruck:        1018 hPa      |
+------------------------------------+-----------------------------------+
| [NETZSTATUS: EINSPEISUNG]          | [SYSTEM & STATUS]             [==]|
|   +236 W                           |   Akku: 98 % | Intervall: 15 Min. |
|   Ueberschuss wird eingespeist     |   WLAN Empfang: -64 dBm           |
+------------------------------------+-----------------------------------+
```

---

## Energie-Management & Akku-Ladestand

1. **15-Minuten-Intervall**: Das Panel wacht alle 15 Minuten aus dem Deep Sleep auf.
2. **Schnell-Synchronisation**: Nach erfolgreicher WLAN-Verbindung und Eintreffen der Sensordaten aus Home Assistant wird das ePaper-Display einmalig aktualisiert.
3. **Akku-Ladestandanzeige**:
   - Da das Seeed XIAO 7.5" ePaper Panel herstellerseitig keinen ADC-Spannungsteiler verdrahtet hat, nutzt die Firmware eine präzise **RTC-Zyklen-Schätzung** (persistiert im RTC-Speicher über alle Deep-Sleep-Phasen).
   - Ausgelegt auf den 2000 mAh Akku (ca. 2800 Zyklen à 15 Minuten = ~4 Wochen Laufzeit).
   - Zeigt den Ladestand in `%` sowie als dynamisches grafisches Batteriesymbol an.
   - Exponiert die Entität `sensor.battery_level` in Home Assistant.
   - Nach dem Aufladen kann der Zähler über den Button **`button.reset_battery_100`** in Home Assistant einfach wieder auf 100 % zurückgesetzt werden.
4. **Automatischer Tiefschlaf**: Nach dem Refresh schläft der ESP32-C3 sofort wieder ein.
5. **Schutzschaltung**: Nach spätestens 45 Sekunden Wachzeit schläft das Panel in jedem Fall ein, selbst bei WLAN-Verbindungsabbrüchen, um den 2000 mAh Akku zu schonen.
6. **OTA-Wartungsmodus**: In Home Assistant steht der Schalter `switch.prevent_deep_sleep` zur Verfügung. Wird dieser aktiviert, bleibt das Panel nach dem nächsten Aufwachen dauerhaft online, um Firmware-Updates über WLAN (OTA) zu ermöglichen.

---

## Erste Schritte & Installation

### 1. secrets.yaml anlegen

Kopiere die Vorlage `secrets.yaml.example` nach `secrets.yaml`:

```bash
cp secrets.yaml.example secrets.yaml
```

Trage deine WLAN-Zugangsdaten und den Home Assistant API Encryption Key ein:

```yaml
wifi_ssid: "DeinWLAN"
wifi_password: "DeinWLANPasswort"
api_encryption_key: "DEIN_32_BYTE_BASE64_KEY"
ota_password: "DeinOtaPasswort"
fallback_password: "DeinFallbackPasswort"
```

### 2. Konfiguration validieren

```bash
esphome config seeed-xiao-esp32c3.yaml
```

### 3. Erstes Flashen per USB-C

Verbinde das XIAO 7.5" ePaper Panel per USB-C-Kabel mit dem PC:

```bash
esphome run seeed-xiao-esp32c3.yaml
```

Wähle die passende serielle Schnittstelle (z. B. `COMx` unter Windows).

### 4. Spätere Updates per WLAN (OTA)

1. Schalte in Home Assistant `Prevent Deep Sleep` ein.
2. Warte maximal 15 Minuten, bis das Panel aufwacht und wach bleibt.
3. Führe den Flash-Befehl aus:
   ```bash
   esphome run seeed-xiao-esp32c3.yaml
   ```
4. Schalte `Prevent Deep Sleep` anschließend wieder aus.

---

## Verwendete Home Assistant Entitäten

| Messwert | Entität in Home Assistant | Einheit |
| :--- | :--- | :--- |
| Solar Live-Leistung | `sensor.core2_solar_live` | W |
| Solar Tagesertrag | `sensor.core2_solar_day_energy_kwh` | kWh |
| Haus Live-Verbrauch | `sensor.core2_house_live` | W |
| Haus Tagesverbrauch | `sensor.core2_house_day_energy_kwh` | kWh |
| Netzeinspeisung | `sensor.core2_grid_export_power` | W |
| Netzbezug | `sensor.core2_grid_import_power` | W |
| Wetterdaten | `weather.home` | - |
| Zeitstempel | `time.homeassistant` | - |
