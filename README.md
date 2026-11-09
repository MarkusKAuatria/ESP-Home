# ESPHome: Heizpatrone im Pufferspeicher (ESP32 + Display)

Die Konfiguration [`heizpatrone.yaml`](heizpatrone.yaml) steuert eine 9-kW-Heizpatrone in 1-kW-Stufen über 6 Relais. Sie läuft auf einem ESP32 mit Farbdisplay, nutzt den PV-Überschuss aus Home Assistant und schaltet bei Übertemperatur ab.

![Display: Automatik, Übertemperatur, Handbetrieb](docs/display-vorschau.png)

## Leistungsstufen

Laut Spezifikation (● = Relais ein):

| Leistung | K1 L1 | K2 L2/N | K3 L2 | K4 L3/N | K5 L3 | K6 L1/N |
|---------:|:-----:|:-------:|:-----:|:-------:|:-----:|:-------:|
| 0 W      |       |         |       |         |       |         |
| 1000 W   | ●     |         |       |         |       |         |
| 2000 W   | ●     |         | ●     |         |       |         |
| 3000 W   | ●     |         | ●     |         | ●     |         |
| 4000 W   | ●     | ●       | ●     |         |       |         |
| 5000 W   | ●     | ●       | ●     |         | ●     |         |
| 6000 W   | ●     | ●       | ●     | ●       |       |         |
| 7000 W   | ●     | ●       | ●     | ●       | ●     |         |
| 9000 W   | ●     | ●       | ●     | ●       | ●     | ●       |

Die Tabelle entspricht einer Heizpatrone mit **3 Heizelementen zu je 3 kW an 400 V**. Jedes Element hängt mit einem Ende an seiner Phase (K1, K3, K5) und mit dem anderen Ende am Umschalter K2, K4 oder K6:

- **Ruhestellung (aus):** Elementende liegt am **N** → Stern, 230 V, ca. **1 kW**
- **Angezogen (ein):** Elementende liegt an der **Nachbarphase** → Dreieck, 400 V, **3 kW**

Nachgerechnet ergibt jede Zeile der Tabelle genau die angegebene Leistung. 8 kW ist mit 3 Elementen nicht möglich, deshalb fehlt diese Stufe.

> **Hinweis zur Spezifikation:** Die 6. Spalte heißt dort ebenfalls „L2/N“. Elektrisch muss es **L1/N** sein: Das dritte Element (an L3) wird im Dreieck gegen L1 geschaltet. Die Logik folgt unverändert der Tabelle, nur die Bezeichnung ist angepasst.

### Umschalten nur stromlos

K2, K4 und K6 schalten ein Elementende zwischen **N und einer Phase** um. Würde das unter Last passieren, könnte der Abschaltlichtbogen Phase und N kurzschließen. Die Firmware hält deshalb immer diese Reihenfolge ein, wenn sich eines dieser Relais ändert:

1. Alle Phasenrelais K1, K3, K5 aus
2. 0,5 s Pause
3. K2, K4, K6 umschalten
4. 0,5 s Pause
5. Phasenrelais der neuen Stufe ein

Dafür sind die Relais in Home Assistant **nicht direkt schaltbar**, sondern nur als Anzeige sichtbar. Geschaltet wird ausschließlich über die Stufenlogik.

## Betriebsarten

Umschaltbar über die Entität **Betriebsart**.

### Automatik (PV-Überschuss)

Gerechnet wird mit der **verfügbaren Leistung** = gemessener PV-Überschuss + aktuell geschaltete Heizleistung. Ohne diese Kompensation würde jedes Zuschalten den Messwert sofort wieder unter die Schwelle drücken.

- **Einschalthysterese** (Standard 100 W): Eine Stufe wird zugeschaltet, wenn die verfügbare Leistung mindestens *Stufe + Einschalthysterese* beträgt.
- **Ausschalthysterese** (Standard 100 W): Eine Stufe wird verlassen, wenn die verfügbare Leistung unter *Stufe − Ausschalthysterese* fällt.
- **Einschalt- und Ausschaltverzögerung** (Standard 60 s / 30 s): So lange muss die Bedingung ununterbrochen anstehen. Kurze Wolken oder Lastspitzen lösen nichts aus.
- Nach Ablauf der Verzögerung springt die Regelung direkt auf die passende Stufe, auch über mehrere Stufen.
- Ohne Verbindung zu Home Assistant oder ohne PV-Wert wird nach der Ausschaltverzögerung abgeschaltet.

Beispiel mit Standardwerten für die 4-kW-Stufe: ein ab **4100 W** verfügbar, aus unter **3900 W**.

### Handbetrieb

Die Leistung wird über die Entität **Handleistung** vorgegeben (0 … 9000 W) und sofort geschaltet. Die Übertemperatur-Abschaltung gilt weiterhin. Der Handbetrieb läuft auch ohne Verbindung zu Home Assistant weiter. Beim Wechsel zurück auf Automatik regelt diese ab der aktuellen Stufe weiter.

## Maximaltemperatur

- Ein eigener DS18B20 am **Eingang 1** überwacht die Maximaltemperatur.
- Liegt der Messwert **über der Maximaltemperatur** (Standard 80 °C), werden sofort alle Relais abgeschaltet (0 W), in beiden Betriebsarten.
- **Wieder eingeschaltet** wird erst, wenn die Temperatur um die **Temperatur-Hysterese** gefallen ist (Standard 5 K, also bei 75 °C).
- **Liefert der Fühler keinen Messwert** (defekt, abgesteckt, kurz nach dem Start), bleibt ebenfalls alles aus.

> ⚠️ Diese Abschaltung ist eine Komfortfunktion, **keine Sicherheitseinrichtung**. Die Heizpatrone braucht ihren eigenen Sicherheitstemperaturbegrenzer (STB) bzw. Thermostat, der direkt die Leistung trennt. Die Maximaltemperatur am besten unter dem Thermostat der Patrone einstellen.

## Entitäten in Home Assistant

| Entität | Art | Bedeutung |
|---------|-----|-----------|
| **Betriebsart** | Auswahl | Automatik / Hand |
| **Handleistung** | Auswahl | 0 … 9000 W für den Handbetrieb |
| **Heizleistung** | Sensor | aktuell geschaltete Leistung |
| **Verfügbare Leistung** | Sensor | PV-Überschuss + Heizleistung |
| **Temperatur Max-Fühler** | Sensor | Fühler an Eingang 1 |
| **Speicher 1 (oben) … Speicher 5 (unten)** | Sensor | Temperaturverlauf im Speicher |
| **Übertemperatur** | Binärsensor | Abschaltung aktiv |
| **Relais K1 L1 … K6 L1-N** | Binärsensor | Relaiszustand (nur Anzeige) |
| **Status** | Text | z. B. „Automatik: 4000 W“, „Übertemperatur 81.0 °C“ |
| **Maximaltemperatur**, **Temperatur-Hysterese** | Einstellung | Übertemperatur-Abschaltung |
| **Einschalthysterese**, **Ausschalthysterese** | Einstellung | Automatik |
| **Einschaltverzögerung**, **Ausschaltverzögerung** | Einstellung | Automatik (10–900 s) |

Einstellungen, Betriebsart und Handleistung bleiben nach einem Neustart erhalten. Die Werte unter `substitutions` sind nur die Startwerte beim ersten Flashen.

## Display

Das Display (2,8", 320 × 240) zeigt:

- **Kopfzeile:** Betriebsart (grün AUTOMATIK, orange HAND)
- **Heizleistung:** groß, mit Balken
- **Relais:** wie die LEDs im Schaltschrank, rot = ein, /N-Relais blau = aus
- **PV:** PV-Überschuss und verfügbare Leistung, in Automatik mit Countdown bis zur nächsten Umschaltung
- **Speicher:** bis zu 5 Schichten, farbig von blau (kalt) bis rot (heiß); nicht vorhandene Fühler grau
- **Max-Fühler:** Messwert und Grenze
- **Statuszeile:** bei Übertemperatur rot

Zeigt dein Display falsche Farben oder ein Negativbild, beim `display:` die Optionen `invert_colors: true` bzw. `color_order: bgr` ergänzen.

## Hardware

### ESP32-Empfehlung

**ESP32-DevKitC-32E** (Espressif, ESP32-WROOM-32E, 4 MB Flash) mit **2,8"-TFT ILI9341 (SPI, 320 × 240)**.

- Genug freie GPIOs für 6 Relais, 2 Fühlerbusse und das Display, ohne kritische Boot-Pins
- Von ESPHome voll unterstützt, günstig, überall erhältlich
- Das ILI9341-Display ist gut ablesbar, läuft an 3,3 V und braucht keinen Touch

Wer lieber ein Board mit eingebautem Display nimmt (z. B. „Cheap Yellow Display“ ESP32-2432S028), hat zu wenige freie GPIOs für 6 Relais und bräuchte zusätzlich einen I/O-Expander.

### Stückliste (Vorschlag)

| Anzahl | Teil |
|-------:|------|
| 1 | ESP32-DevKitC-32E |
| 1 | TFT 2,8" ILI9341, SPI, 320 × 240 |
| 1 | ULN2803A (Darlington-Treiber, 8 Kanäle) |
| 6 | Koppelrelais 12 V DC, 1 Wechsler, 250 V~ / ≥ 6 A (z. B. Finder Serie 38, Hutschiene) |
| 6 | Installationsschütz 230-V-Spule, passend zur Patrone (z. B. 20/25 A); **Q2, Q4, Q6 mit 1 Schließer + 1 Öffner** |
| 2–6 | DS18B20 wasserdicht (für Tauchhülse) |
| 1 | Netzteil 12 V DC ≥ 1 A, Hutschiene (z. B. Mean Well HDR-15-12) |
| 1 | DC/DC-Wandler 12 V → 5 V, ≥ 1 A (z. B. Traco TSR 1-2450) |
| 3 | NPN-Transistor BC547 |
| 6 / 3 | LED rot / LED blau |
| 9 / 3 / 3 / 2 | Widerstand 1 kΩ / 47 kΩ / 10 kΩ / 4,7 kΩ |

### Schaltungsvorschlag Steuerung

![Schaltungsvorschlag Steuerung](docs/schaltplan-steuerung.svg)

**Signalweg:**
1. Der ESP32 steuert über eine **ULN2803A** die Spulen von 6 **Koppelrelais (12 V)**. GPIO high bedeutet Relais ein.
2. Die Kontakte der Koppelrelais schalten die 230-V-Spulen der **Schütze Q1–Q6**.
3. Die Schütze schalten die Heizelemente.

Die Freilaufdioden für die Relaisspulen sind in der ULN2803A eingebaut (Pin 10 COM an +12 V).

**LED-Anzeigen laut Spezifikation:**
- **Typ A (K1, K3, K5):** Rote LED mit 1 kΩ parallel zur Spule, leuchtet bei *ein*.
- **Typ B (K2, K4, K6):** Rote LED wie Typ A. Zusätzlich eine blaue LED über einen Transistor-Inverter (BC547). Sie leuchtet, solange der ULN-Ausgang sperrt, also bei *aus* (Stellung N). Der Spannungsteiler 47 kΩ / 10 kΩ belastet die Spule dabei nur mit etwa 0,2 mA, das Relais bleibt sicher abgefallen.

**Belegung:**

| Funktion | ESP32 | Anschluss |
|----------|-------|-----------|
| K1 L1 | GPIO32 | ULN2803A IN1 |
| K2 L2/N | GPIO33 | ULN2803A IN2 |
| K3 L2 | GPIO25 | ULN2803A IN3 |
| K4 L3/N | GPIO26 | ULN2803A IN4 |
| K5 L3 | GPIO27 | ULN2803A IN5 |
| K6 L1/N | GPIO13 | ULN2803A IN6 |
| Fühler Eingang 1 (Max) | GPIO4 | DS18B20 DQ, 4,7 kΩ nach 3V3 |
| Fühler Eingang 2 (Speicher) | GPIO16 | DS18B20 DQ (1–5 parallel), 4,7 kΩ nach 3V3 |
| Display SCK | GPIO18 | SCK |
| Display MOSI | GPIO23 | SDI (MOSI) |
| Display CS | GPIO21 | CS |
| Display DC | GPIO17 | DC |
| Display RESET | GPIO19 | RESET |
| Versorgung | 5V, GND | vom DC/DC-Wandler |
| Display, Fühler | 3V3, GND | VCC / LED bzw. VDD |

Alle Pins lassen sich oben in der YAML unter `substitutions` ändern.

### Leistungsteil (Prinzip)

![Leistungsteil](docs/leistungsteil.svg)

> ⚠️ **Nur durch eine Elektrofachkraft.** Der Leistungsteil arbeitet mit 400 V und bis zu 9 kW. Die Zeichnung zeigt nur das Prinzip, das der Schaltlogik zugrunde liegt. Absicherung, Fehlerstromschutz, Querschnitte, STB und Ausführung gehören in die Hand einer Elektrofachkraft. **Q2, Q4 und Q6 müssen echte Umschalter sein** (Öffner und Schließer im selben Schütz, mechanisch gekoppelt), damit N und Phase nie gleichzeitig verbunden sein können.

## Einrichtung

1. `secrets.yaml.example` nach `secrets.yaml` kopieren und WLAN, Passwörter und API-Schlüssel eintragen (Schlüssel z. B. mit `openssl rand -base64 32` erzeugen).
2. Unter `substitutions` bei `pv_sensor` die Entität deines PV-Überschuss-Sensors eintragen.
   - Ist bei deinem Sensor die Einspeisung **negativ**: beim `homeassistant`-Sensor den Filter `multiply: -1` einkommentieren.
3. Erstes Flashen per USB:
   ```bash
   esphome run heizpatrone.yaml
   ```
   Danach geht es auch über WLAN.
4. **Speicherfühler zuordnen:** Im Log stehen beim Start die gefundenen Adressen je Bus. Bei den Sensoren `Speicher 1 … 5` jeweils `index: X` durch `address: 0x…` ersetzen, sodass *Speicher 1* der oberste Fühler ist. Nicht vorhandene Fühler aus der YAML löschen.
5. **Erst ohne Leistungsteil testen:**
   - Mit abgeklemmten Schützen im Handbetrieb alle Stufen durchschalten.
   - Relais und LEDs mit der Tabelle oben vergleichen.
   - Bei Stufenwechseln mit K2/K4/K6 auf die Pausen achten.

## Anpassen

Unter `substitutions` in der YAML:

- `pv_sensor`: Entität des PV-Überschuss-Sensors
- `stufen_leistung`, `stufen_relais`: Leistungsstufen und Relaismuster (Reihenfolge K1 … K6). Bei Änderungen auch die Optionen von *Handleistung* anpassen.
- `umschaltpause`: Pause beim Umschalten der /N-Relais (Standard 500 ms)
- Startwerte: `max_temperatur`, `temperatur_hysterese`, `einschalthysterese`, `ausschalthysterese`, `einschaltverzoegerung`, `ausschaltverzoegerung`
- `pin_…`: alle GPIOs
