# Nulleinspeisung mit Marstek Venus E 3.0 und Home Assistant

Home Assistant regelt den Marstek Venus E 3.0 per Modbus so, dass der Saldo am
Netzanschlusspunkt um **0 W** pendelt: Bei PV-Überschuss lädt der Akku, bei
Mehrverbrauch entlädt er.

## Funktionsprinzip

Ein inkrementeller Regler läuft alle 15 Sekunden:

```
neuer Sollwert = alter Sollwert + Faktor × (Netzleistung − Ziel)
```

- **Sollwert > 0** → Akku entlädt mit dieser Leistung, **Sollwert < 0** → Akku lädt, **0** → Standby.
- Innerhalb des **Totbands** (±25 W) wird nichts geschrieben — der Saldo darf dort pendeln.
- Der Regler nutzt **ausschließlich den Netzzähler** als Regelgröße (Hichi
  Wifi v2 am Landis+Gyr E320, vorzeichenbehaftet: positiv = Bezug, negativ =
  Einspeisung). Die PV-Leistung dient nur als Ladefreigabe.

**Schutzgrenzen** (alle als Helfer einstellbar):

| Grenze | Wirkung |
|---|---|
| SoC ≤ 12 % | kein Entladen mehr |
| SoC ≥ 100 % | kein Laden mehr |
| PV < 50 W | kein Laden (verhindert Netzladen durch Messfehler) |
| max. 2500 W | Lade-/Entladeleistung begrenzt |

## Voraussetzungen

- Home Assistant ≥ 2025.9
- [marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus) (HACS) eingerichtet und funktionsfähig
- Hichi Wifi v2 via Tasmota/MQTT eingebunden, Netzleistung aktualisiert sich ca. alle 10 s

## Dateien

| Datei | Zweck |
|---|---|
| `packages/nulleinspeisung.yaml` | Komplettes Package: Helfer, Statussensoren, 3 Automationen |
| `dashboard/nulleinspeisung.yaml` | Lovelace-Karte zum Einfügen |

## Installation

1. **Package einbinden** — in dieser Installation liegen Packages als benannte
   Einzel-Includes in `/config/integrations/`, in `configuration.yaml`:

   ```yaml
   homeassistant:
     packages:
       nulleinspeisung: !include integrations/nulleinspeisung.yaml
   ```

   (Alternative bei leerem Setup: `packages: !include_dir_named packages` und
   die Datei nach `/config/packages/` legen.)

2. **Package kopieren:** Im File Editor / Studio Code Server den Inhalt von
   `packages/nulleinspeisung.yaml` als `/config/integrations/nulleinspeisung.yaml`
   einfügen.

3. **Entity-IDs prüfen** (Entwicklerwerkzeuge → Zustände). Das Package erwartet:

   | Entität | Zweck |
   |---|---|
   | `sensor.tasmota_e320_power_in` | Netzleistung in W, vorzeichenbehaftet |
   | `sensor.pv_leistung` | PV-Leistung in W |
   | `switch.marstek_venus_modbus_rs485_steuermodus` | RS485-Steuermodus |
   | `select.marstek_venus_modbus_erzwungener_modus` | Force Mode |
   | `number.marstek_venus_modbus_ladeleistung_einstellen` | Lade-Sollwert |
   | `number.marstek_venus_modbus_entladeleistung_einstellen` | Entlade-Sollwert |
   | `sensor.marstek_venus_modbus_soc_batterie` | Batterie-SoC |
   | `sensor.marstek_venus_modbus_ac_leistung` | Akku AC-Leistung (nur Dashboard) |

   Bei Abweichungen: Suchen & Ersetzen in der Package-Datei.

4. **Force-Mode-Optionen prüfen:** In den Zuständen die Select-Entität öffnen und
   im Attribut `options` nachsehen. Das Package erwartet `standby`, `charge`,
   `discharge`. Stehen dort andere Werte (z. B. deutsche), die Variablen
   `opt_standby` / `opt_laden` / `opt_entladen` in allen drei Automationen anpassen.

5. **Konfiguration prüfen** (Entwicklerwerkzeuge → YAML → Konfiguration prüfen),
   dann **Home Assistant neu starten**.

6. **Dashboard-Karte** aus `dashboard/nulleinspeisung.yaml` einfügen
   (Karte hinzufügen → Manuell).

7. **Master einschalten:** `input_boolean.nulleinspeisung_aktiv` aktivieren.
   Die Lifecycle-Automation schaltet den RS485-Steuermodus ein und startet in Standby;
   der Regler übernimmt beim nächsten 15-s-Takt.

## Parameter (Helfer)

| Helfer | Default | Bedeutung |
|---|---|---|
| `nulleinspeisung_ziel` | 0 W | Zielwert der Netzleistung (z. B. +30 für garantiert keine Einspeisung) |
| `nulleinspeisung_totband` | 25 W | Abweichung, unterhalb derer nicht nachgeregelt wird |
| `nulleinspeisung_faktor` | 0,7 | Reglerverstärkung — kleiner = träger, größer = schneller (Schwinggefahr) |
| `nulleinspeisung_max_ladeleistung` | 2500 W | Obergrenze Laden |
| `nulleinspeisung_max_entladeleistung` | 2500 W | Obergrenze Entladen |
| `nulleinspeisung_min_soc` | 12 % | Entladestopp |
| `nulleinspeisung_pv_schwelle` | 50 W | Laden nur oberhalb dieser PV-Leistung |
| `nulleinspeisung_sollwert` | — | interner Reglerzustand, nicht manuell verstellen |

> **Wichtig:** Die Parameter tragen `initial:` und werden bei jedem HA-Neustart
> auf die YAML-Werte zurückgesetzt. Dauerhafte Änderungen in der Package-Datei
> vornehmen — die Datei in diesem Repo ist die Quelle der Wahrheit.

## Failsafe-Verhalten

| Situation | Reaktion |
|---|---|
| Keine Zählerdaten > 120 s (WLAN/Tasmota weg) | Akku → Standby, Sollwert 0, Push-Meldung |
| Marstek per Modbus > 60 s nicht erreichbar | Sollwert 0, Push-Meldung |
| RS485-Steuermodus deaktiviert sich selbst | Push-Meldung (bekanntes Firmware-Verhalten, Schalter manuell wieder einschalten) |
| Zählerdaten wieder da (30 s stabil) | Push-Meldung, Regler läuft automatisch weiter |
| HA-Neustart | Sollwert 0, Akku → Standby, dann normale Regelung |
| Master-Schalter aus | Akku → Standby |

**Restrisiko:** Die Marstek-Firmware hat **kein dokumentiertes Kommunikations-Timeout**
(kein Watchdog-Register). Stürzt Home Assistant hart ab, während der Akku im
RS485-Modus lädt/entlädt, fährt er den letzten Befehl weiter, bis HA zurück ist
(die Lifecycle-Automation räumt beim Start auf). Schlimmster Fall: Akku entlädt
ins Netz bzw. lädt aus dem Netz, bis er die eigenen SoC-Grenzen erreicht.

**Einspeisung ist nicht garantiert null:** Ist der Akku voll (oder der Überschuss
größer als 2500 W), fließt der Rest ins Netz. Eine harte Null-Export-Garantie
ginge nur über Drosselung des Wechselrichters.

## Inbetriebnahme-Tests

1. Master einschalten → Status wechselt auf „Standby", dann „Lädt/Entlädt";
   Netzleistung pendelt sich binnen ~1 min auf ±25 W ein.
2. **Lasttest:** Wasserkocher an → Entladeleistung folgt in 2–3 Takten (~45 s); aus → zurück.
3. **Ladetest** bei PV-Überschuss: Einspeisung wird weggeregelt, Akku lädt.
4. **Watchdog:** Hichi/Tasmota kurz trennen → nach ≤ 2–3 min Standby + Push-Meldung;
   nach Wiederverbinden Entwarnung und automatischer Wiederanlauf.
5. **Nachttest:** kein Ladebefehl nachts (PV-Freigabe), Entladen deckt die Grundlast.
6. SoC-Grenzen über einige Tage im History-Graph beobachten.

## Haftungsausschluss

Dieses Projekt steuert reale Hardware (Batteriespeicher) über Modbus.
Nutzung auf eigene Gefahr — insbesondere die Abschnitte
[Failsafe-Verhalten](#failsafe-verhalten) und das dort beschriebene Restrisiko
vor der Inbetriebnahme lesen. Keine Gewährleistung, siehe [LICENSE](LICENSE).

## Tuning

- **Regelung schwingt** (Sollwert pendelt hoch/runter): `faktor` senken (z. B. 0,5).
- **Regelung zu träge:** `faktor` erhöhen (max. 1,0) — nie über 1.
- **Zu viele kleine Modbus-Writes:** `totband` erhöhen (z. B. 40 W).
- Der Akku reagiert auf sehr kleine Sollwerte (< ~30 W) evtl. gar nicht — dann
  bleibt ein kleiner Restsaldo stehen. Das ist normal und unkritisch.

## Lizenz

[MIT](LICENSE)
