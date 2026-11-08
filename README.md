# ESPHome: 3 Relais am D1 mini, gestuft nach PV-Überschuss

Diese ESPHome-Konfiguration ([`d1-mini-relais.yaml`](d1-mini-relais.yaml)) schaltet drei Relais an einem Wemos D1 mini abhängig vom PV-Überschuss, z. B. für einen 3-kW-Heizstab mit 3 × 1000 W:

| Verfügbare Leistung | Relais 1 | Relais 2 | Relais 3 |
|---------------------|:--------:|:--------:|:--------:|
| unter 1000 W        | aus      | aus      | aus      |
| ab 1000 W           | **ein**  | aus      | aus      |
| ab 2000 W           | **ein**  | **ein**  | aus      |
| ab 3000 W           | **ein**  | **ein**  | **ein**  |

## So wird Flattern verhindert

1. **Kompensation:** Der Zähler misst den Überschuss *nach* dem Verbrauch der Relais. Schaltet Relais 1 bei 1100 W Überschuss ein, misst der Zähler danach nur noch 100 W. Deshalb rechnet die Regelung mit der **verfügbaren Leistung** = gemessener Überschuss + 1000 W je eingeschaltetem Relais.
2. **Hysterese** (Standard 200 W): Eine Stufe schaltet bei ihrer Schwelle ein, aber erst unter *Schwelle − Hysterese* wieder aus:

   | Stufe | Ein ab  | Aus unter |
   |-------|---------|-----------|
   | 1     | 1000 W  | 800 W     |
   | 2     | 2000 W  | 1800 W    |
   | 3     | 3000 W  | 2800 W    |

   Praktisch heißt das: Bis zu 200 W Netzbezug werden toleriert, bevor eine Stufe abschaltet.
3. **Verzögerungen:** Die Bedingung muss ununterbrochen anstehen, bevor geschaltet wird. Standard: **60 s** zum Zuschalten, **30 s** zum Abschalten. Kurze Wolken oder Lastspitzen (Wasserkocher, Föhn …) lösen so nichts aus.
4. **Zuschalten immer nur stufenweise:** Nach jeder Stufe wird erst neu gemessen. **Abschalten** kann dagegen mehrere Stufen auf einmal betreffen, damit bei einem starken Einbruch nicht unnötig lange Strom aus dem Netz bezogen wird.
5. **Ausfallsicherheit:** Ist Home Assistant nicht erreichbar oder der Sensor nicht verfügbar, schalten alle Relais nach der Ausschaltverzögerung ab.

## Entitäten in Home Assistant

| Entität | Bedeutung |
|---------|-----------|
| **PV-Automatik** | Ein: Regelung aktiv. Aus: alle Relais aus, danach von Hand schaltbar. |
| **Relais 1 / 2 / 3** | Zustand der Relais (bei aktiver Automatik bitte nicht von Hand schalten) |
| **Stufe** | Aktuelle Stufe 0–3 |
| **Verfügbare Leistung** | Überschuss inkl. Verbrauch der eingeschalteten Relais (Grundlage der Regelung) |
| **Hysterese** | einstellbar, 0–1000 W |
| **Einschaltverzögerung** | einstellbar, 10–900 s |
| **Ausschaltverzögerung** | einstellbar, 10–900 s |

Hysterese und Verzögerungen bleiben auch nach einem Stromausfall erhalten. Die Werte in der YAML sind nur die Startwerte beim allerersten Flashen; danach gilt, was in Home Assistant eingestellt ist.

## Verdrahtung

| Relais   | D1-mini-Pin | GPIO   |
|----------|-------------|--------|
| Relais 1 | D1          | GPIO5  |
| Relais 2 | D2          | GPIO4  |
| Relais 3 | D5          | GPIO14 |

Außerdem: `5V` an VCC des Relaismoduls und `G` an GND.

Die Pins D3, D4 und D8 sind absichtlich nicht belegt, weil sie beim Booten einen bestimmten Pegel brauchen und Relais dort beim Start kurz klicken oder den Start verhindern können.

Die meisten günstigen Relaismodule schalten bei LOW. Deshalb steht `relais_invertiert` auf `"true"`. Schaltet dein Relais genau verkehrt herum, setze den Wert auf `"false"`.

## Einrichtung

1. `secrets.yaml.example` nach `secrets.yaml` kopieren und WLAN-Daten, Passwörter und API-Schlüssel eintragen (Schlüssel z. B. mit `openssl rand -base64 32` erzeugen).
2. In der YAML unter `substitutions` bei `pv_sensor` die Entität deines PV-Überschuss-Sensors eintragen (Standard: `sensor.pv_ueberschuss`).
3. Flashen:
   ```bash
   esphome run d1-mini-relais.yaml
   ```
   Alternativ die YAML im ESPHome-Dashboard bzw. im Home-Assistant-Add-on einfügen.
4. In Home Assistant das Gerät hinzufügen. Den Wert des PV-Sensors holt sich der D1 mini danach automatisch über die API.

## Welcher Sensor passt?

- **Netzleistung am Zähler, Einspeisung positiv** (z. B. `sensor.pv_ueberschuss`): passt direkt.
- **Netzleistung, Bezug positiv / Einspeisung negativ** (bei vielen Smartmetern so): in der YAML beim `homeassistant`-Sensor die Zeilen `filters:` und `- multiply: -1` einkommentieren.
- **Sensor, der den Verbrauch der Relais bereits herausrechnet** (z. B. PV-Erzeugung minus Hausverbrauch ohne Heizstab): `leistung_relais` auf `"0"` setzen.

## Anpassen

Unter `substitutions` in der YAML:

- `pv_sensor`: Entität des PV-Überschuss-Sensors
- `leistung_relais`: Verbrauch je eingeschaltetem Relais (Standard 1000 W)
- `schwelle_1`, `schwelle_2`, `schwelle_3`: Einschaltschwellen (Standard 1000 / 2000 / 3000 W)
- `hysterese`, `einschaltverzoegerung`, `ausschaltverzoegerung`: Startwerte der einstellbaren Parameter
- `relais_1_pin` … `relais_3_pin`: GPIOs der Relais
- `relais_invertiert`: Relaislogik (active low / active high)

**Gar kein Netzbezug gewünscht?** Schwellen um die Hysterese erhöhen, z. B. `1200 / 2200 / 3200` bei 200 W Hysterese. Dann schaltet Stufe 1 erst ein, wenn danach noch 200 W eingespeist werden, und aus, sobald Strom aus dem Netz bezogen wird.

## Verhalten beim Start

Nach einem Neustart oder Stromausfall sind alle Relais **aus**. Ist die PV-Automatik eingeschaltet, werden die Stufen danach mit der Einschaltverzögerung wieder zugeschaltet, sobald genug Überschuss da ist.
