# NIBE: Verdichter-Startwert (Gradminuten) nach Außentemperatur – Anleitung zum Nachbauen

[![Blueprint in Home Assistant importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffubu2k%2Fnibe-vvm320-startwert-automation%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fnibe_gm%2Fstartwert_verdichter.yaml)

Blueprint: `blueprints/automation/nibe_gm/startwert_verdichter.yaml` · Hilfs-Entitäten als Paket: `packages/nibe_gm.yaml`

## 0. Bitte zuerst lesen – Nutzung auf eigene Verantwortung

Diese Automation **schreibt einen Parameter in deine Wärmepumpe** („Start Verdichter“, Menü 4.9.3). Sie wurde an **einer** Anlage (NIBE VVM 320 E mit Außeneinheit F2040, Fußbodenheizung) entwickelt und getestet. Die Standardwerte sind dort sinnvoll, **aber kein allgemeiner Richtwert**.

- Prüfe vorher, ob die Stufen zu deinem Haus und deiner Anlage passen (Herstellerhinweise, Installateur).
- **Notiere dir deinen aktuellen Startwert**, damit du ihn jederzeit von Hand zurücksetzen kannst.
- Beobachte die ersten Tage genau (Meldungen, Dashboard).
- Für Schäden, Garantiefolgen oder Mehrverbrauch wird keine Haftung übernommen.
- Die Begrenzung der Schreibzugriffe ist eine **Vorsichtsmaßnahme**. Wie oft der Parameterspeicher der Wärmepumpe beschrieben werden darf, ist von NIBE nicht öffentlich dokumentiert.

## 1. Was die Automation macht

Zweimal täglich (00:00 und 12:00 Uhr, jeweils mit Wiederholung 1 h später) wird ein **6-Stunden-Mittel der Außentemperatur** ausgewertet. Je nach Wert wird der Startwert in drei Stufen eingestellt:

| 6-h-Mittel Außentemperatur | Startwert (Standard) |
|---|---|
| über 5 °C | −300 |
| 0 … 5 °C | −200 |
| unter 0 °C | −130 |

Je milder es ist, desto größer der Abstand (Startwert stärker negativ), bevor der Verdichter anspringt. Das soll das Takten bei Übergangswetter verringern.

Schutzmechanismen:

- **Nur in der Heizperiode:** Liegt das Mittel über dem Heizungsstopp der Wärmepumpe, passiert nichts.
- **Hysterese** (±0,75 K) an den Schwellen und **höchstens eine Stufe** pro Änderung.
- **Schreibt nur, wenn sich der Zielwert ändert**, und frühestens nach der Sperrzeit (Standard 11 h).
- **Nie** bei Warmwasserbetrieb, Abtauung oder während des Verdichterstarts.
- Nichts bei nicht erreichbarer Wärmepumpe, unplausibler Außentemperatur (Abweichung zum NIBE-Eigenmittel BT1) oder auffälligem Vorlauf.
- Nach dem Schreiben wird die **Rückmeldung abgewartet** (Cloud-Latenz). Bleibt sie aus, zählt ein Fehler. **Nach 2 Fehlern in Folge schaltet sich die Automation selbst ab** und meldet das.
- Der laufende Gradminuten-Zähler wird **nie** verändert.

> Achtung beim ersten Lauf: Entspricht dein aktueller Startwert keiner der drei Stufen, schreibt die Automation direkt die zur Temperatur passende Stufe.

## 2. Voraussetzungen

- Home Assistant **2024.10 oder neuer**
- NIBE-Wärmepumpe über die **myUplink-Integration** eingebunden
- **myUplink Premium** (ohne Premium lassen sich Parameter nicht schreiben)
- Eine Benachrichtigung aufs Handy (empfohlen, damit du Änderungen und Fehler siehst)

## 3. Schritt 1: Entitäten der Wärmepumpe aktivieren

Die myUplink-Integration legt sehr viele Entitäten an, die meisten sind **standardmäßig deaktiviert**. Aktiviere (bzw. suche) diese:

| Zweck | typischer Name (Englisch) |
|---|---|
| Startwert Verdichter (Zahl) | `…start_compressor` |
| Heizungsstopp (Zahl) | `…stop_heating` |
| Außentemperatur aktuell | `…outdoor_temperature` |
| Außentemperatur Mittel (BT1) | `…average_outdoor_temp_bt1` |
| Verbindung | `…konnektivitat` (binary_sensor) |
| Priorität | `…priority` |
| Status Verdichter | `…status_compressor_eb101` |
| Abtauung | `…defrosting_eb101_ep14` |
| Vorlauf BT2 | `…supply_line_bt2` |
| Berechneter Vorlauf Heizkreis 1 | `…calculated_supply_climate_system_1` |

So geht es: *Einstellungen → Geräte & Dienste → myUplink → Entitäten*. Filter „deaktivierte Entitäten anzeigen“ einschalten, Entität öffnen, Zahnrad → **Aktiviert** einschalten. Nach dem Aktivieren lädt die Integration neu, das dauert eine Minute. Die genauen Namen unterscheiden sich je nach Gerät und Sprache.

## 4. Schritt 2: Hilfs-Entitäten anlegen

Es werden vier Hilfs-Entitäten gebraucht. Ein Blueprint kann sie nicht selbst anlegen.

### Variante A: YAML-Paket (schnell)

1. In `configuration.yaml` einmalig eintragen (falls noch nicht vorhanden):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Datei `packages/nibe_gm.yaml` aus diesem Repository nach `/config/packages/nibe_gm.yaml` kopieren.
3. In der Datei bei `entity_id:` den **eigenen Außentemperatur-Sensor** eintragen.
4. *Entwicklerwerkzeuge → YAML → Konfiguration prüfen*, dann Home Assistant neu starten.

Es entstehen: `input_datetime.nibe_gm_sperrzeit_start`, `counter.nibe_gm_schreibfehler`, `counter.nibe_gm_schreibzugriffe`, `sensor.nibe_aussentemperatur_6h_mittel`.

### Variante B: Per Oberfläche (*Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen*)

| Typ | Einstellung |
|---|---|
| Datum und/oder Uhrzeit | Name „Nibe GM Sperrzeit Start“, **Datum und Uhrzeit** |
| Zähler | Name „Nibe GM Schreibfehler“, Startwert 0 |
| Zähler | Name „Nibe GM Schreibzugriffe“, Startwert 0 |
| Statistik | Quelle: Außentemperatur-Sensor, Merkmal **Durchschnitt (zeitgewichtet)**, Höchstalter **6 Stunden**, „Letzten Wert behalten“ **an**, Genauigkeit 2 |

## 5. Schritt 3: Blueprint einspielen

**Am einfachsten:** den Button „Blueprint in Home Assistant importieren“ oben in dieser Anleitung anklicken. Alternativ von Hand:

1. Datei `blueprints/automation/nibe_gm/startwert_verdichter.yaml` aus diesem Repository nach `/config/blueprints/automation/nibe_gm/startwert_verdichter.yaml` kopieren.
2. *Einstellungen → Automatisierungen & Szenen → Blueprints* aufrufen (ggf. neu laden).
3. Beim Blueprint **Automatisierung erstellen** wählen.

## 6. Schritt 4: Automation ausfüllen

Der Blueprint hat fünf Bereiche:

1. **Wärmepumpe:** die aktivierten Entitäten aus Schritt 1 auswählen.
2. **Hilfs-Entitäten:** die vier Entitäten aus Schritt 2 auswählen.
3. **Stufen und Schwellen:** Standardwerte nur ändern, wenn du weißt, was du tust. Die Stufen müssen **aufsteigend** sein (z. B. −300, −200, −130), sonst läuft die Automation nie.
4. **Zeiten und Sicherheit:** Standard 00:00/12:00 mit Wiederholung 01:00/13:00, Sperrzeit 11 h.
5. **Benachrichtigung:** Aktion „Benachrichtigung senden“ an dein Handy hinzufügen. In der Nachricht stehen die Variablen `nachricht_titel` und `nachricht_text` zur Verfügung, im YAML-Modus z. B.:
   ```yaml
   - action: notify.mobile_app_MEIN_HANDY
     data:
       title: "{{ nachricht_titel }}"
       message: "{{ nachricht_text }}"
   ```

## 7. Schritt 5: Testen

- Vor dem ersten Lauf den Startwert notieren.
- Automation einschalten. Im Menü der Automation **Aktionen ausführen** startet die Aktionen **ohne** Bedingungen. Das schreibt also sofort. Zum Testen besser den nächsten Prüfzeitpunkt abwarten oder die Prüfzeit vorübergehend auf wenige Minuten später stellen.
- Im **Ablaufprotokoll (Traces)** der Automation siehst du, welche Bedingung ein Schreiben verhindert hat.
- Nach einer Änderung kommt die Meldung „Start Verdichter geändert“, der Zähler „Schreibzugriffe“ steigt um 1.

## 8. Fehlersuche

| Beobachtung | Mögliche Ursache |
|---|---|
| Automation läuft, schreibt aber nie | Bedingung nicht erfüllt: Trace prüfen. Häufig Mittel über Heizungsstopp, Zielwert gleich aktuellem Wert oder Sperrzeit. |
| Mittel-Sensor `unknown`/`unavailable` | Quelle falsch oder Statistik-Helfer neu, braucht erst Werte. |
| „Nicht bestätigt“ | myUplink-Cloud langsam oder Wert wird abgelehnt. In myUplink/am Display prüfen. |
| Automation hat sich abgeschaltet | 2 Fehler in Folge. Ursache klären, Automation von Hand wieder einschalten. |
| Entität „stop heating“ fehlt | In myUplink deaktiviert, siehe Schritt 1. |
| Lauf blockiert wegen Warmwasser/Abtauung | Gewollt. Die Wiederholung 1 h später fängt das meistens auf. |

## 9. Zurücksetzen

1. Automation ausschalten.
2. Startwert (Menü 4.9.3) auf deinen vorher notierten Wert stellen, entweder am Display oder in der Zahl-Entität.

## 10. Dashboard (optional)

Kachel für Startwert, Gradminuten und Zähler:

```yaml
type: vertical-stack
cards:
  - type: tile
    entity: number.DEIN_START_COMPRESSOR
    name: Start Verdichter (eingestellt)
  - type: tile
    entity: counter.nibe_gm_schreibzugriffe
    name: Schreibzugriffe
    icon: mdi:memory
  - type: history-graph
    title: Gradminuten und Startwert (12 h)
    hours_to_show: 12
    entities:
      - sensor.DEIN_DEGREE_MINUTES
      - number.DEIN_START_COMPRESSOR
```
