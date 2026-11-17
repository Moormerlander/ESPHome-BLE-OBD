# ESPHome BLE OBD

ESPHome-Firmware für einen **ESP32-C3**, die einen **Bluetooth-LE-OBD2-Adapter** (getestet: Vgate iCar Pro BLE) im **Fiat Ducato** ausliest und die Werte an **Home Assistant** liefert – inklusive **AdBlue-Füllstand**.

Entstanden im Wohnmobil „Pepper“ (Fiat Ducato mit 9-Gang-Automatik).

## Was ausgelesen wird

| Wert | PID | Takt | Status |
|---|---|---|---|
| Drehzahl, Geschwindigkeit, Kühlmittel | 010C, 010D, 0105 | 3 s | geprüft |
| Kraftstoffverbrauch l/h und l/100 km | 019D | 3 s | geprüft |
| Bordspannung | ATRV | 3 s / 15 s | geprüft |
| **AdBlue-Füllstand** | 0185 (physisch an Steuergerät 01) | 30 s | geprüft |
| AdBlue-Temperatur und -Konzentration | 019B | 30 s | geprüft |
| Dieseltank | 012F | 30 s | geprüft |
| Außentemperatur, Luftdruck | 0146, 0133 | 30 s | plausibel |
| Öltemperatur | 015C | 30 s | liefert bisher nur `FF` (ungültig) |
| Rußpartikelfilter (Regeneration, Beladung, Differenzdruck) | 018B, 017A | 30 s | ungeprüft |
| Kilometerstand und Tageskilometer (aus „km seit Fehlerspeicher-Löschen“) | 0131 | 30 s | in Erprobung |

Details zu Steuergeräten, Formeln und Beobachtungen: [docs/ducato-obd-erkenntnisse.md](docs/ducato-obd-erkenntnisse.md)

**SGW-Bypass:** Für alle oben genannten Werte ist am Ducato **kein** Secure-Gateway-Bypass nötig (getestet nach Ausbau).

## Funktionen

- **Sparmodus:** Schläft das Motorsteuergerät, fragt der ESP nur noch alle 15 s die Spannung und einmal pro Minute die Steuergeräte ab.
- **Adapter-Erkennung (ab v1.12):** feste MAC über `fallback_mac`, alternativ automatische Suche nach BLE-OBD-Adaptern per Name. Unterstützte BLE-Profile:
  1. Vgate iCar Pro (`E7810A71…` / `BEF8D6C9…`) – getestet
  2. `FFF0` / `FFF1` / `FFF2` – viele ELM327-Klone, vLinker
  3. `FFE0` / `FFE1` – ELM327-Klone, LELink
  4. `18F0` / `2AF0` / `2AF1` – Veepeak, OBDLink

  Profile 2–4 sind nach gängigen Angaben gebaut und nicht mit echten Adaptern getestet. Klassische Bluetooth-Adapter (SPP, Kopplungs-PIN) funktionieren nicht.
- **Diagnose-Buttons:** PID-Scan, Steuergeräte-IDs, freier OBD-Befehl.

## Installation

> **Gerätename:** In der Firmware heißt das Gerät weiterhin `ducato-obd-ble`. So bleiben bestehende Home-Assistant-Entitäten (`…pepper_ducato_obd_ble_…`) und Dashboards unverändert. Für eine Neuinstallation kann `esphome: name:` frei gewählt werden.


1. `esphome/esphome-ble-obd.yaml` in ESPHome als neues Gerät übernehmen.
2. `secrets.yaml` mit `wifi_ssid` und `wifi_password` anlegen (Vorlage: `esphome/secrets.yaml.example`).
3. In den `substitutions` die eigene Adapter-MAC bei `fallback_mac` eintragen – oder `00:00:00:00:00:00` für die automatische Suche.
4. **Fallback-Hotspot:** Das Passwort unter `wifi: ap:` ist ein Platzhalter (`12345678`) – vor dem Flashen ändern.
5. Flashen, in Home Assistant hinzufügen, bei „Kilometerstand setzen“ einmal den Tachostand eintragen.

> Die Header (`adblue_hdr`, `engine_rx`) und das Protokoll `ATSP7` (CAN 29 bit, 500 kbit/s) sind auf den Ducato zugeschnitten. Bei anderen Fahrzeugen zuerst den PID-Scan nutzen.

## Home Assistant

Im Ordner `home-assistant/`:

| Datei | Inhalt |
|---|---|
| `obd2-view.yaml` | kompletter Dashboard-View „OBD2“ (Rundinstrumente für AdBlue, Diesel, Verbrauch, Drehzahl, dazu Kacheln) |
| `obd-kachel.yaml` | einzelne Übersichtskachel mit allen Werten |
| `kachel-kilometerstand.yaml`, `kachel-tageskilometer.yaml` | eigenständige Kacheln |

Voraussetzungen: [custom:button-card](https://github.com/custom-cards/button-card); das Farbschema folgt `input_select.cockpit_farbschema` (Hell / Dunkel / Automatisch). Die Entitäts-IDs beginnen mit `pepper_ducato_obd_ble_` – bei anderem Gerätenamen per Suchen/Ersetzen anpassen.

## Versionen

Jede Firmware-Version ist ein eigener Commit mit Tag (`v1.0` … `v1.12.2`). Die Änderungen stehen zusätzlich im Kopfkommentar von `esphome/esphome-ble-obd.yaml`.

Letzte im Fahrzeug bei laufendem Motor geprüfte Version: **v1.10**. v1.12.1 verbindet sich nachweislich mit dem Adapter (Fahrzeug aus); v1.12.2 ändert nur die Anzeige der Adapter-MAC.
