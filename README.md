# NIBE: Verdichter-Startwert (Gradminuten) nach Außentemperatur – Anleitung zum Nachbauen

[![Blueprint in Home Assistant importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffubu2k%2Fnibe-vvm320-startwert-automation%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fnibe_gm%2Fstartwert_verdichter.yaml)

> ⚠️ **Nutzung auf eigene Verantwortung.** Diese Automation schreibt einen Parameter in deine Wärmepumpe. Die Standardwerte stammen von **einer** Anlage (VVM 320 E + F2040) und sind kein allgemeiner Richtwert. Es gibt **keine Gewährleistung und keine Haftung**, siehe Abschnitt 0 und die Datei [LICENSE](LICENSE). Das Projekt steht in keiner Verbindung zu NIBE.

Blueprint: `blueprints/automation/nibe_gm/startwert_verdichter.yaml` · Hilfs-Entitäten als Paket: `packages/nibe_gm.yaml` · Dashboard: `dashboard/nibe_waermepumpe_dashboard.yaml`

## 0. Bitte zuerst lesen – Nutzung auf eigene Verantwortung

Diese Automation **schreibt einen Parameter in deine Wärmepumpe** („Start Verdichter“, Menü 4.9.3). Sie wurde an **einer** Anlage (NIBE VVM 320 E mit Außeneinheit F2040, Fußbodenheizung) entwickelt und getestet. Die Standardwerte sind dort sinnvoll, **aber kein allgemeiner Richtwert**.

- Prüfe vorher, ob die Stufen zu deinem Haus und deiner Anlage passen (Herstellerhinweise, Installateur).
- **Notiere dir deinen aktuellen Startwert**, damit du ihn jederzeit von Hand zurücksetzen kannst.
- Beobachte die ersten Tage genau (Meldungen, Dashboard).
- Für Schäden an der Anlage, Garantiefolgen, Mehrverbrauch, Komfortverlust oder sonstige Folgen wird **keine Haftung** übernommen (siehe [LICENSE](LICENSE)). Du bist selbst dafür verantwortlich, die Werte zu prüfen und die Anlage zu überwachen.
- Dieses Projekt ist **kein Produkt von NIBE** und nicht mit dem Hersteller abgestimmt.
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

- Home Assistant **2026.10 oder neuer**
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
| *Nur fürs Dashboard:* Gradminuten | `…degree_minutes` |
| *Nur fürs Dashboard:* Verdichterfrequenz jetzt | `…current_compressor_frequency_eb101` |
| *Nur fürs Dashboard:* Verdichterfrequenz gewünscht | `…requested_compressor_freq_eb101` |
| *Nur fürs Dashboard:* Anzahl Starts | `…number_of_starts` |

So geht es: *Einstellungen → Geräte & Dienste → myUplink → Entitäten*. Filter „deaktivierte Entitäten anzeigen“ einschalten, Entität öffnen, Zahnrad → **Aktiviert** einschalten. Nach dem Aktivieren lädt die Integration neu, das dauert eine Minute. Die genauen Namen unterscheiden sich je nach Gerät und Sprache.

### Entitäten finden und prüfen

Öffne *Entwicklerwerkzeuge → Template* und füge diesen Code ein. Er listet alle benötigten Entitäten auf, die in deiner Installation **aktiv** sind:

```jinja
{% set suche = 'start_compressor|stop_heating|outdoor_temperature|average_outdoor_temp_bt1|konnektivitat|connectivity|_priority|status_compressor|defrosting_eb101_ep14|supply_line_bt2|calculated_supply_climate_system_1|degree_minutes|current_compressor_frequency_eb101|requested_compressor_freq_eb101|number_of_starts' %}
{% for e in states | map(attribute='entity_id') | select('search', suche) | sort %}
{{ e }}
{% endfor %}
```

Es sollten etwa **14 Zeilen** erscheinen (je eine pro Zeile der Tabelle oben; die vier „nur fürs Dashboard“ brauchst du nur, wenn du das Dashboard willst). Fehlt eine, ist die Entität noch **deaktiviert** und muss wie beschrieben aktiviert werden. Diese Namen trägst du später im Blueprint ein. Erscheinen mehrere ähnliche Namen (z. B. mit und ohne Raumnamen), nimm den, dessen Wert zum Display deiner Wärmepumpe passt.

## 4. Schritt 2: Hilfs-Entitäten anlegen

Es werden vier Hilfs-Entitäten gebraucht. Ein Blueprint kann sie nicht selbst anlegen.

### Variante A: YAML-Paket (schnell)

1. In `configuration.yaml` einmalig eintragen (falls noch nicht vorhanden):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Datei `packages/nibe_gm.yaml` aus diesem Repository nach `/config/packages/nibe_gm.yaml` kopieren.
3. In der Datei den **eigenen Außentemperatur-Sensor** bei `entity_id:` eintragen. Willst du später das Dashboard (Abschnitt 10) benutzen, trage auch den Sensor „Anzahl Starts“ bei `source:` ein (zweimal, steht im Text). Ohne Dashboard kannst du den Block `utility_meter` löschen.
4. *Entwicklerwerkzeuge → YAML → Konfiguration prüfen*, dann Home Assistant neu starten.

Es entstehen: `input_datetime.nibe_gm_sperrzeit_start`, `counter.nibe_gm_schreibfehler`, `counter.nibe_gm_schreibzugriffe`, `sensor.nibe_aussentemperatur_6h_mittel`, `sensor.nibe_verdichterstarts_heute` und `sensor.nibe_verdichterstarts_woche`.

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

> Ein vollautomatisches Ausfüllen ist nicht möglich, weil Home Assistant jede Installation anders benennt (Gerätename, Raum, Sprache). Mit dem Entitäten-Finder aus Schritt 1 und der Tabelle unten ist es aber in wenigen Minuten erledigt. Die Felder 3 und 4 haben sinnvolle Standardwerte und müssen nur bei Bedarf angefasst werden.

### Bereich 1: Wärmepumpe (alle Felder Pflicht)

| Feld im Blueprint | Auswählen |
|---|---|
| Start Verdichter (Gradminuten-Startwert) | `number.…start_compressor` |
| Heizungsstopp (Außentemperatur) | `number.…stop_heating` |
| Außentemperatur (aktuell) | `sensor.…outdoor_temperature` |
| Außentemperatur Mittel (BT1, NIBE-intern) | `sensor.…average_outdoor_temp_bt1` |
| Verbindung der Wärmepumpe | `binary_sensor.…konnektivitat` (oder `…connectivity`) |
| Priorität (Betriebsart) | `sensor.…priority` |
| Status Verdichter | `sensor.…status_compressor_eb101` |
| Abtauung (0/1) | `sensor.…defrosting_eb101_ep14` |
| Vorlauftemperatur (BT2) | `sensor.…supply_line_bt2` |
| Berechnete Vorlauftemperatur (Heizkreis 1) | `sensor.…calculated_supply_climate_system_1` |

### Bereich 2: Hilfs-Entitäten (alle Felder Pflicht)

Bei Variante A (YAML-Paket) sind die Namen:

| Feld im Blueprint | Auswählen |
|---|---|
| Außentemperatur 6-h-Mittel | `sensor.nibe_aussentemperatur_6h_mittel` |
| Zeitstempel letzte Änderung | `input_datetime.nibe_gm_sperrzeit_start` |
| Zähler Schreibfehler | `counter.nibe_gm_schreibfehler` |
| Zähler Schreibzugriffe | `counter.nibe_gm_schreibzugriffe` |

Bei Variante B die Namen, die du selbst vergeben hast.

### Bereiche 3 und 4: Stufen und Zeiten (optional)

Die Standardwerte (−300 / −200 / −130, Schwellen 5 °C und 0 °C, Hysterese 0,75 K, Prüfungen 00:00 und 12:00 mit Wiederholung eine Stunde später, Sperrzeit 11 h) stammen von einer einzelnen Anlage. **Ändere sie nur, wenn du weißt, was du tust.** Die Stufen müssen **aufsteigend** sein, sonst läuft die Automation nie.

### Bereich 5: Benachrichtigung (empfohlen)

Aktion „Benachrichtigung senden“ an dein Handy hinzufügen. In der Nachricht stehen die Variablen `nachricht_titel` und `nachricht_text` zur Verfügung. Im YAML-Modus z. B.:

```yaml
- action: notify.mobile_app_MEIN_HANDY
  data:
    title: "{{ nachricht_titel }}"
    message: "{{ nachricht_text }}"
```

Den Namen deines Handys findest du unter *Entwicklerwerkzeuge → Aktionen*, wenn du nach `notify.mobile_app` suchst.

### Alternative: komplette Automation als YAML

*Einstellungen → Automatisierungen → Neue Automatisierung → ⋮ → In YAML bearbeiten*, den Inhalt ersetzen, die Entitäten mit den Ergebnissen des Finders anpassen und speichern:

```yaml
alias: NIBE Startwert Verdichter nach Außentemperatur
use_blueprint:
  path: nibe_gm/startwert_verdichter.yaml
  input:
    start_value: number.DEIN_start_compressor
    heating_stop: number.DEIN_stop_heating
    outdoor_raw: sensor.DEIN_outdoor_temperature
    nibe_avg: sensor.DEIN_average_outdoor_temp_bt1
    connectivity: binary_sensor.DEIN_konnektivitat
    priority: sensor.DEIN_priority
    compressor_status: sensor.DEIN_status_compressor_eb101
    defrost: sensor.DEIN_defrosting_eb101_ep14
    supply_actual: sensor.DEIN_supply_line_bt2
    supply_calc: sensor.DEIN_calculated_supply_climate_system_1
    outdoor_mean: sensor.nibe_aussentemperatur_6h_mittel
    lock_helper: input_datetime.nibe_gm_sperrzeit_start
    error_counter: counter.nibe_gm_schreibfehler
    write_counter: counter.nibe_gm_schreibzugriffe
    notify_actions:
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

## 10. Dashboard (optional, einfach)

Mit dem Dashboard siehst du auf einen Blick, was die Wärmepumpe und die Automation machen. Es hat einen **Schalter, mit dem du die GM-Automatik an und aus machst**.

**Was du siehst**

- **Startwert und Gradminuten:** Der eingestellte Startwert (rot) und die Gradminuten jetzt. Dazu eine Linie der letzten 12 Stunden.
- **Außentemperatur:** Wert jetzt, 6-Stunden-Mittel und das Mittel von NIBE.
- **GM-Automatik:** Der Schalter an/aus, wann zuletzt etwas geändert wurde, wie viele Fehler in Folge es gab und wie oft die Automation geschrieben hat.
- **Verdichter:** Frequenz jetzt und gewünscht, dazu der Verlauf der letzten 24 Stunden.
- **Verdichterstarts:** Starts heute, diese Woche und gesamt. Dazu ein Balkenbild mit den Starts pro Tag.
- **Abtauung:** Ob die Wärmepumpe gerade abtaut.

**So baust du es auf**

1. Gehe zu *Einstellungen → Dashboards → Dashboard hinzufügen* und wähle „Neues Dashboard von Grund auf“. Gib einen Namen ein, zum Beispiel „Wärmepumpe“.
2. Öffne das neue Dashboard. Tippe oben rechts auf den **Stift**, dann auf die **drei Punkte** und auf **Rohkonfigurations-Editor**.
3. Lösche den ganzen Text dort.
4. Öffne die Datei `dashboard/nibe_waermepumpe_dashboard.yaml` aus diesem Repository, kopiere **alles** und füge es im Editor ein.
5. Ersetze jetzt jedes Wort mit `DEIN_` durch den echten Namen aus deinem Home Assistant (siehe Tabelle). Die Namen findest du mit dem **Entitäten-Finder** aus Schritt 1.
6. Tippe auf **Speichern**.

**Das musst du ersetzen**

| Im Text steht | Setze ein |
|---|---|
| `number.DEIN_start_compressor` | dein Startwert (endet auf `start_compressor`) |
| `sensor.DEIN_degree_minutes` | deine Gradminuten (endet auf `degree_minutes`) |
| `sensor.DEIN_outdoor_temperature` | deine Außentemperatur |
| `sensor.DEIN_average_outdoor_temp_bt1` | dein NIBE-Mittel |
| `sensor.DEIN_current_compressor_frequency_eb101` | deine Verdichterfrequenz jetzt |
| `sensor.DEIN_requested_compressor_freq_eb101` | deine gewünschte Verdichterfrequenz |
| `sensor.DEIN_number_of_starts` | deine Anzahl Starts |
| `sensor.DEIN_defrosting_eb101_ep14` | deine Abtauung |
| `automation.DEINE_AUTOMATION` | deine Automation aus dem Blueprint |

**Tipp zum Ersetzen:** Kopiere den Text zuerst in einen einfachen Editor (zum Beispiel Notepad) und benutze dort „Suchen und Ersetzen“. Dann geht es schneller und du vergisst nichts.

**Hilfs-Namen:** Die Namen `sensor.nibe_aussentemperatur_6h_mittel`, `input_datetime.nibe_gm_sperrzeit_start`, `counter.nibe_gm_schreibfehler`, `counter.nibe_gm_schreibzugriffe`, `sensor.nibe_verdichterstarts_heute` und `sensor.nibe_verdichterstarts_woche` kommen aus dem Hilfs-Paket und stehen schon richtig im Dashboard. Hast du die Hilfs-Entitäten selbst angelegt (Variante B) und anders benannt, ändere diese Namen auch.

**Wenn etwas nicht klappt**

| Problem | Lösung |
|---|---|
| Eine Kachel zeigt „Entität nicht gefunden“ | Der Name ist falsch geschrieben oder die Entität ist noch deaktiviert. Prüfe mit dem Entitäten-Finder. |
| Die Starts heute/Woche sind leer | Du hast den Sensor „Anzahl Starts“ im Hilfs-Paket nicht eingetragen oder Home Assistant nicht neu gestartet. |
| Der Schalter macht nichts | Es steht noch `automation.DEINE_AUTOMATION` drin. Setze den Namen deiner Automation ein. |

Mit dem Schalter machst du die Automation aus, ohne sie zu löschen. Wenn sich die Automation nach zwei Fehlern selbst abgeschaltet hat, schaltest du sie hier wieder an.

**Zusätzlich:** Eine einfache Kachel mit Verlauf kannst du auch ohne das ganze Dashboard in ein anderes Dashboard setzen:

```yaml
type: vertical-stack
cards:
  - type: tile
    entity: automation.DEINE_AUTOMATION
    name: GM-Automatik (an / aus)
    features:
      - type: toggle
  - type: tile
    entity: counter.nibe_gm_schreibzugriffe
    name: Schreibzugriffe
    icon: mdi:memory
  - type: history-graph
    title: Gradminuten und Startwert (12 Stunden)
    hours_to_show: 12
    entities:
      - sensor.DEIN_degree_minutes
      - number.DEIN_start_compressor
```
