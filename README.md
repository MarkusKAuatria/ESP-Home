# ESPHome: 3 Relais am D1 mini (gestufte Schaltung)

Diese ESPHome-Konfiguration ([`d1-mini-relais.yaml`](d1-mini-relais.yaml)) steuert drei Relais an einem Wemos D1 mini abhängig von einem Sollwert:

| Sollwert       | Relais 1 | Relais 2 | Relais 3 |
|----------------|:--------:|:--------:|:--------:|
| unter 1000     | aus      | aus      | aus      |
| 1000 bis 1999  | **ein**  | aus      | aus      |
| 2000 bis 2999  | **ein**  | **ein**  | aus      |
| ab 3000        | **ein**  | **ein**  | **ein**  |

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
2. Flashen:
   ```bash
   esphome run d1-mini-relais.yaml
   ```
   Alternativ die YAML im ESPHome-Dashboard bzw. im Home-Assistant-Add-on einfügen.
3. In Home Assistant das Gerät hinzufügen. Danach gibt es die Entitäten:
   - **Sollwert** (Schieberegler 0–3000): steuert die Relais
   - **Relais 1 / 2 / 3**: zeigen den Zustand der Relais an

## Anpassen

Alles Wichtige steht oben in der YAML unter `substitutions`:

- `schwelle_1`, `schwelle_2`, `schwelle_3`: Schaltschwellen (Standard 1000 / 2000 / 3000)
- `einheit`: Einheit des Sollwerts (Standard `W`)
- `relais_1_pin` … `relais_3_pin`: GPIOs der Relais
- `relais_invertiert`: Relaislogik (active low / active high)

### Wert direkt aus einem Home-Assistant-Sensor

Soll der Wert automatisch von einem Sensor kommen (z. B. PV-Überschuss), den auskommentierten `sensor:`-Block in der YAML aktivieren und die `entity_id` anpassen. Werte über 3000 schalten dann ebenfalls alle drei Relais ein.

## Verhalten beim Start

Nach einem Neustart oder Stromausfall sind alle Relais **aus** und der Sollwert steht auf 0. Soll stattdessen der letzte Sollwert wiederhergestellt werden, in der YAML beim `number` die Option `restore_value: true` setzen.

> Hinweis: Wenn du ein Relais in Home Assistant von Hand schaltest, setzt die nächste Änderung des Sollwerts es wieder nach der Stufenlogik.
