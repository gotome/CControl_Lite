# CControl_Lite – Projektdokumentation für Claude

## Projekt-Übersicht

**Projekt:** CControl_Lite  
**Steuerung:** Schneider Electric TM173O (EcoStruxure Machine Expert HVAC)  
**Firma:** RIG Automation  
**Basis:** CControl (B&R-basierte Heutrocknungssteuerung, Lasco)  
**Stand:** V0.03 / 2026-05-16  

CControl_Lite ist eine abgespeckte Version von CControl für einfache Anlagen mit:
- 1× Ventilator (FU-geregelt, analoger Sollwert)
- 2× Box (Box 1 + optional Box 2)
- Bis zu 2× Boxenklappe (Wechsler-Relais-Ansteuerung)
- 1× Luftaufbereiter (optional, digital + analog)
- 1× Abluftventilator (optional, nur digital)

---

## Hardware

**Digitale Eingänge:**
| Signal | HW-Pin | Beschreibung |
|--------|--------|--------------|
| diBox1Offen | hw_di1 | Endlage Box 1 offen |
| diBox1Geschl | hw_di2 | Endlage Box 1 geschlossen |
| diBox2Offen | hw_di3 | Endlage Box 2 offen |
| diBox2Geschl | hw_di4 | Endlage Box 2 geschlossen |
| diBetriebLuftaufber | hw_di5 | Rückmeldung Luftaufbereiter läuft |
| diBetriebVentilator | hw_di6 | Rückmeldung Ventilator läuft |

**Digitale Ausgänge:**
| Signal | HW-Pin | Beschreibung |
|--------|--------|--------------|
| doFrgLuftaufber | hw_do1 | Freigabe Luftaufbereiter |
| doFrgBox1 | hw_do2 | Klappe Box 1 (Wechsler: FALSE=zu, TRUE=auf) |
| doFrgBox2 | hw_do3 | Klappe Box 2 (Wechsler: FALSE=zu, TRUE=auf) |
| doFrgAbluftventilator | hw_do4 | Freigabe Abluftventilator |
| doFrgVentilator | hw_do5 | Freigabe Ventilator |

**Analoge Eingänge:**
| Signal | HW-Pin | Beschreibung |
|--------|--------|--------------|
| ai Frischlufttemp | hw_ai1 | Frischlufttemperatur (0..32767) |
| ai Frischluftfeuchte | hw_ai2 | Frischluftfeuchte (0..32767) |
| ai AbluftBox1 Temp | hw_ai3 | Ablufttemperatur Box 1 |
| ai AbluftBox1 Feuchte | hw_ai4 | Abluftfeuchte Box 1 |
| ai AbluftBox2 Temp | hw_ai5 | Ablufttemperatur Box 2 |
| ai AbluftBox2 Feuchte | hw_ai6 | Abluftfeuchte Box 2 |
| ai Stocktemp | hw_ai7 | Stocktemperatur (fix 0..100 °C) |

> **Hinweis:** Trockenluft-Sensoren verwenden aktuell dieselben Rohwerte wie AbluftBox1 (hw_ai3, hw_ai4) – dies ist im aktuellen Projektstand bewusst so, da kein eigener Trockenluft-Sensor vorgesehen ist.

**Analoge Ausgänge:**
| Signal | HW-Pin | Beschreibung |
|--------|--------|--------------|
| aoFuVentilator | hw_ao1 | FU-Sollwert Ventilator (0..100% → 0..32767) |
| aoLeistVorgLuftaufber | hw_ao4 | Leistungsvorgabe Luftaufbereiter (hw_ao4 – ao2/ao3 frei) |

---

## Programmstruktur

| Programm | Inhalt | Status |
|----------|--------|--------|
| `IOMappingCyclic` | I/O-Anbindung, Skalierung, Filterung, Testbetrieb | Fertig |
| `HmiMappingCyclic` | Parameterübergabe HMI↔PLC, Daten für Visualisierung | Fertig |
| `aktorenMapping` | Aktorsteuerung (Klappen FB, Status-Mapping) | Fertig |
| `BerechnungCyclic` | Berechnungen (Feuchte, Sollwerte) | Leer – geplant |
| `AblaufstrgCyclic` | Ablaufsteuerung (Betriebszustände) | Leer – geplant |
| `LeistungCyclic` | Leistungsregelung Ventilator/Luftaufbereiter | Leer – geplant |
| `AlarmCyclic` | Alarmverarbeitung | Leer – geplant |

---

## Funktionsbaustein fbBoxenklappe

Steuert eine Boxenklappe mit Wechsler-Relais:
- **OeffnenOut = TRUE** → Relais angezogen → Motor dreht Richtung "Auf"
- **OeffnenOut = FALSE** → Relais abgefallen → Motor dreht Richtung "Zu"

Features:
- Laufzeitüberwachung (Öffnen + Schließen getrennt)
- 40 ms Anzugsverzögerung (TON_AnzVerz)
- Betrieb mit und ohne Endlagen
- Fehlerquittierung via `QuitFehler`

Stoerungseingang: `diBox1Offen AND diBox1Geschl` (beide Endlagen gleichzeitig = Fehler)

---

## Globale Variablen (wichtigste)

| Variable | Typ | Beschreibung |
|----------|-----|--------------|
| `gAktModus` | eBetriebsmodus | AUTO / IOTEST |
| `gStatusAnlage` | T_StatAnlage | Status aller Aktoren [0..39] |
| `gBetriebsparameter` | T_Betriebsparameter | Sensor-Parameter, Ausstattung, Klappenzeiten |
| `gAktoren` | T_Aktor | Sollwerte/Status Klappen + Freigaben |
| `gIoTest` | T_IoTest | I/O-Test-Struktur (Spiegelwerte) |
| `gQuitErr` | BOOL | Fehlerquittierung (von HMI) |
| `IBN_TRUE/FALSE` | BOOL | Inbetriebnahme-Merker |

---

## Betriebsmodi

**AUTO:** Normalbetrieb. Hardware-I/O via interne Variablen. Test-HMI-Ausgänge werden auf 0 zurückgesetzt.

**IOTEST:** I/O-Testbetrieb. Hardware-Ausgänge direkt über `gIoTest.DigitalOut.*` und `gIoTest.AnalogOut.*` steuerbar. Eingänge werden in `gIoTest.DigitalIn.*` / `gIoTest.AnalogIn.*` gespiegelt. Umschaltung über `HMI_TEST_doIoTestAktivieren`.

---

## Bekannte offene Punkte / TODOs

1. `gBetriebsparameter.Ausstattung.*` Flags werden in HmiMappingCyclic gesetzt, aber in der Steuerlogik noch nicht ausgewertet.
2. `EndlagenVhd` in aktorenMapping ist hardcoded TRUE – Ausstattungsflags noch nicht angebunden.
3. `HMI_TEST_doFrgVentilator` wird im Normalbetrieb nicht zurückgesetzt (Fehler).
4. `gAktoren.xFrgVentilator` etc. sind in der Struktur vorhanden aber werden noch nicht genutzt (stattdessen direkte globale do*-Variablen).
5. Trockenluft-Sensor: aktuell auf hw_ai3/ai4 (= AbluftBox1). Klären ob eigener Sensor geplant ist.

---

## Konventionen / Namensgebung

- `hw_di*` / `hw_do*` / `hw_ai*` / `hw_ao*` → direkte Hardware-Pins
- `di*` / `do*` / `ai*` / `ao*` → interne Prozessvariablen
- `HMI_IN_*` → PLC schreibt → HMI liest (Istwerte, Status)
- `HMI_TEST_*` → Testbetrieb-Variablen (bidirektional)
- `g*` → globale Variablen
- `fb*` → Funktionsbausteine
- `ST_*` / `T_*` / `e*` → Datentypen
