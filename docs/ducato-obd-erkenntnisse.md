# Ducato OBD – Erkenntnisse (Stand 12.11.2026)

## Verbindung
- Adapter: Vgate iCar Pro BLE, MAC `D2:E0:2F:8D:5B:5C`, ELM327 v2.3
- BLE: Service `e7810a71-73ae-499d-8c15-faa9aef0c3f2`, Characteristic `bef8d6c9-9c21-4c9e-b632-bd58c1009f9f` (Write + Notify)
- Protokoll: ISO 15765-4 CAN, 29 bit, 500 kbit/s → `ATSP7`
- SGW-Bypass: **nicht nötig.** Mode-01-Abfragen inkl. AdBlue (0185/019B, physisch adressiert an 01) funktionieren auch nach Ausbau des Bypass – im Log vom 12.11.2026 (v1.10) alle 30 s frische Antworten, Werte wechseln zwischen E1/E2 (88,2/88,6 %)
- Physisch adressiert an Steuergerät 10 (`ATSH DA10F1`) kommt auf 0131 nur `7F 01 11` (Dienst nicht unterstützt) → ab v1.11 funktional mit Empfangsfilter `ATCRA 18DAF110`, danach `ATAR`

## Steuergeräte (Antwort-IDs)
| ID | Header physisch | Bemerkung |
|---|---|---|
| `18DAF110` | `ATSH DA10F1` | Motorsteuergerät – liefert die meisten PIDs, schläft bei Zündung ohne Motor nach ~2 min |
| `18DAF101` | `ATSH DA01F1` | liefert 0185 + 019B (AdBlue), bleibt bei Zündung am längsten wach |
| `18DAF118` | `ATSH DA18F1` | wenige PIDs (vermutlich Getriebesteuergerät, 9-Gang-Automatik) |

Funktionale Abfrage (`ATSH DB33F1`): alle drei antworten → bei mehrteiligen Antworten (0185) vermischen sich die Frames → 0185 physisch an 01 senden.

## Unterstützte Mode-01-PIDs (PID-Scan)
- **Steuergerät 10:** 01 04 05 0B 0C 0D 0F 10 11 13 1C 1F · 21 2F 30 31 33 · 42 45 46 49 4A 4C 4D 4E 4F 50 5C 5E · 62 63 66 67 68 69 6A 6B 6D 71 77 78 7A · 83 85 87 88 8B 8C 8E 8F 92 9D 9E · A1 A7 A8
- **Steuergerät 01:** 01 04 05 0C 0D 1C · 21 33 · 46 · 85 9B
- **Steuergerät 18:** 01 04 05 0C 0D · 21 30 31 · 42 49
- Nicht vorhanden: A6 (Kilometerstand), A4 (eingelegter Gang)

## Verifizierte Umrechnungen
| PID | Formel | Beispiel |
|---|---|---|
| 0185 | Byte F × 100/255 = AdBlue-Füllstand % (Byte A Bit 2 = Füllstand gültig) | `E2` → 88,6 % |
| 019B | B × 0,25 = Konzentration %, C − 40 = AdBlue-Temperatur °C, D × 100/255 = Füllstand % | `0E 80 2B E1` → 32,0 %, 3 °C, 88,2 % (passt zu 0185) |
| 019D | (A·256+B) × 0,02 = Kraftstoff g/s; ÷ 0,835 × 3,6 = l/h | `000B` → 0,95 l/h im Leerlauf |
| 012F | A × 100/255 = Dieseltank % (nur Steuergerät 10) | `BF` → 74,9 % |

## Noch ungeprüft
- 0131 (km seit Fehlerspeicher-Löschen) als Basis für Kilometerstand/Tageskilometer: A·256+B km, ab v1.11 per ATCRA-Filter auf Steuergerät 10
- 0133 Luftdruck, 0146 Außentemperatur, 018B/017A DPF: Formeln nach Standard, im Fahrzeug noch nicht bestätigt

## Beobachtungen
- Lichtmaschine lädt im Leerlauf kaum (12,3–12,6 V) → Spannung taugt nicht als „Motor läuft“-Signal
- Bei Zündung an / Motor aus liefert ein Steuergerät eingefrorene Drehzahl ~194–208 rpm
- Vereinzelte `CAN ERROR` direkt nach dem Aufwachen sind normal
- 015C Öltemperatur lieferte bei kaltem Motor Rohwert `FF` (= ungültig, ergab 215 °C) → ab v1.10 verworfen; auch bei Kühlmittel 34 °C (5 min Leerlauf) noch `FF` – ob nach Warmfahren ein gültiger Wert kommt, ist offen
- Bordspannung bei laufendem Motor im Leerlauf 12,4–12,6 V (intelligente Lichtmaschine)
- Kühlmittel 0105 liefern drei Steuergeräte, 0146/0133 zwei – Werte stimmen bis auf ±1 überein
