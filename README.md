# NIBE Wärmepumpe: Startwert automatisch an die Außentemperatur anpassen

[![Blueprint in Home Assistant importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ffubu2k%2Fnibe-vvm320-startwert-automation%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fnibe_gm%2Fstartwert_verdichter.yaml)

> ⚠️ **Nutzung auf eigene Verantwortung.** Diese Automation ändert einen Wert in deiner Wärmepumpe. Die Standardwerte kommen von **einer** Anlage (VVM 320 E mit Außeneinheit F2040). Sie sind **nicht** für jedes Haus richtig. Es gibt **keine Garantie und keine Haftung**. Lies Abschnitt 0 und die Datei [LICENSE](LICENSE). Dieses Projekt hat nichts mit NIBE zu tun.

**Das ist im Repository:**

| Datei | Wofür |
|---|---|
| `blueprints/automation/nibe_gm/startwert_verdichter.yaml` | Die Automation (Blueprint) |
| `packages/nibe_gm.yaml` | Die Hilfs-Entitäten, die die Automation braucht |
| `dashboard/nibe_waermepumpe_dashboard.yaml` | Ein Dashboard mit An/Aus-Schalter |
| `LICENSE` | Lizenz und Haftungsausschluss |

---

## 0. Bitte zuerst lesen

Die Automation **schreibt einen Wert in deine Wärmepumpe**. Es ist der Wert „Start Verdichter“ (Menü 4.9.3). Du machst das auf eigenes Risiko.

- Ich habe die Automation nur an **einer** Anlage getestet: NIBE VVM 320 E mit Außeneinheit F2040 und Fußbodenheizung.
- Die Standardwerte sind **kein Tipp für alle**. Prüfe, ob sie zu deinem Haus passen. Frag im Zweifel deinen Installateur.
- **Schreibe dir deinen aktuellen Startwert auf.** Dann kannst du ihn jederzeit von Hand zurückstellen.
- Schau in den ersten Tagen oft nach, was passiert.
- Ich hafte **nicht** für Schäden an der Anlage, für verlorene Garantie, für höhere Kosten oder für weniger Komfort (siehe [LICENSE](LICENSE)). Du bist selbst dafür verantwortlich, alles zu prüfen und im Blick zu behalten.
- Die Automation schreibt nur selten in die Wärmepumpe. Das ist eine Vorsichtsmaßnahme. NIBE sagt nicht öffentlich, wie oft man diesen Speicher beschreiben darf.
- Dieses Projekt ist **kein Produkt von NIBE**. NIBE hat es nicht geprüft.

## 1. Was macht die Automation?

**Das Problem:** Wenn es draußen mild ist, schaltet sich der Verdichter der Wärmepumpe oft an und aus. Das nennt man „Takten“. Das ist nicht gut.

**Die Idee:** Der Wert „Start Verdichter“ sagt, wie weit der Gradminuten-Wert fallen muss, bis der Verdichter startet. Ist es mild, darf der Wert weiter ins Minus gehen. So startet der Verdichter seltener.

Die Automation stellt diesen Wert **in 3 Stufen** ein. Sie schaut dafür auf die Außentemperatur der letzten 6 Stunden (Durchschnitt):

| Außentemperatur (6-Stunden-Durchschnitt) | Startwert (Standard) |
|---|---|
| über 5 °C | −300 |
| 0 bis 5 °C | −200 |
| unter 0 °C | −130 |

**Wann schaut sie nach?** Zweimal am Tag: um 00:00 und um 12:00 Uhr. Ging es zu diesen Zeiten nicht (zum Beispiel wegen Warmwasser), versucht sie es eine Stunde später noch einmal.

**Diese Sicherheiten sind eingebaut:**

- Sie läuft **nur in der Heizzeit**. Ist es wärmer als der „Heizungsstopp“ der Wärmepumpe, tut sie nichts.
- Sie hat einen kleinen **Puffer** an den Grenzen (Hysterese, 0,75 Grad). So springt der Wert nicht hin und her.
- Sie ändert den Wert **nur um eine Stufe** auf einmal.
- Sie schreibt **nur, wenn sich wirklich etwas ändern muss**. Zwischen zwei Schreibvorgängen vergehen mindestens 11 Stunden. Jeder Schreibvorgang wird gezählt.
- Sie schreibt **nie** bei Warmwasser, beim Abtauen oder wenn der Verdichter gerade startet.
- Sie schreibt **nichts**, wenn die Wärmepumpe nicht erreichbar ist oder die Messwerte komisch aussehen.
- Nach dem Schreiben wartet sie, bis die Wärmepumpe den neuen Wert bestätigt.
- Klappt das **zweimal hintereinander nicht**, schaltet sich die Automation selbst aus und meldet sich bei dir.
- Den laufenden Gradminuten-Wert fasst sie **nie** an.

> **Achtung beim ersten Start:** Passt dein aktueller Startwert zu keiner der drei Stufen, schreibt die Automation sofort die passende Stufe.

## 2. Was du brauchst

- Home Assistant **2024.10 oder neuer**
- Die NIBE-Wärmepumpe ist über die **myUplink-Integration** in Home Assistant eingebunden
- **myUplink Premium** (ohne Premium kannst du keine Werte schreiben)
- Eine Benachrichtigung auf dein Handy (empfohlen, damit du Änderungen und Fehler siehst)

## 3. Schritt 1: Entitäten der Wärmepumpe einschalten

Die myUplink-Integration legt sehr viele Entitäten an. **Die meisten sind aus.** Du musst diese einschalten:

| Wofür | Name (meistens so) |
|---|---|
| Startwert Verdichter (Zahl) | `…start_compressor` |
| Heizungsstopp (Zahl) | `…stop_heating` |
| Außentemperatur jetzt | `…outdoor_temperature` |
| Außentemperatur, NIBE-Durchschnitt (BT1) | `…average_outdoor_temp_bt1` |
| Verbindung | `…konnektivitat` (binary_sensor) |
| Priorität (was macht die Anlage gerade) | `…priority` |
| Status Verdichter | `…status_compressor_eb101` |
| Abtauung | `…defrosting_eb101_ep14` |
| Vorlauf (BT2) | `…supply_line_bt2` |
| Berechneter Vorlauf, Heizkreis 1 | `…calculated_supply_climate_system_1` |
| *Nur fürs Dashboard:* Gradminuten | `…degree_minutes` |
| *Nur fürs Dashboard:* Verdichterfrequenz jetzt | `…current_compressor_frequency_eb101` |
| *Nur fürs Dashboard:* Verdichterfrequenz gewünscht | `…requested_compressor_freq_eb101` |
| *Nur fürs Dashboard:* Anzahl Starts | `…number_of_starts` |

**So schaltest du eine Entität ein:**

1. Gehe zu *Einstellungen → Geräte & Dienste → myUplink → Entitäten*.
2. Schalte den Filter „Deaktivierte Entitäten anzeigen“ ein.
3. Öffne die Entität. Tippe auf das Zahnrad. Schalte **Aktiviert** ein.
4. Warte eine Minute. Die Integration lädt dann neu.

Die Namen können bei dir etwas anders sein (je nach Gerät und Sprache).

### Entitäten finden und prüfen

Öffne *Entwicklerwerkzeuge → Template* und füge diesen Text ein. Er zeigt dir alle Entitäten, die **schon eingeschaltet** sind:

```jinja
{% set suche = 'start_compressor|stop_heating|outdoor_temperature|average_outdoor_temp_bt1|konnektivitat|connectivity|_priority|status_compressor|defrosting_eb101_ep14|supply_line_bt2|calculated_supply_climate_system_1|degree_minutes|current_compressor_frequency_eb101|requested_compressor_freq_eb101|number_of_starts' %}
{% for e in states | map(attribute='entity_id') | select('search', suche) | sort %}
{{ e }}
{% endfor %}
```

Es sollten etwa **14 Zeilen** erscheinen. Die vier „nur fürs Dashboard“ brauchst du nur, wenn du das Dashboard willst. Fehlt eine Zeile, ist die Entität noch aus. Schalte sie dann ein.

Diese Namen brauchst du später im Blueprint. Siehst du mehrere ähnliche Namen (zum Beispiel mit und ohne Raumnamen), nimm den, dessen Wert zum Display deiner Wärmepumpe passt.

## 4. Schritt 2: Hilfs-Entitäten anlegen

Die Automation braucht vier Hilfs-Entitäten. Ein Blueprint kann sie nicht selbst anlegen. Du hast zwei Wege.

### Weg A: Fertige Datei (schnell)

1. Trage in der Datei `configuration.yaml` einmal diese Zeilen ein (falls noch nicht da):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Kopiere die Datei `packages/nibe_gm.yaml` aus diesem Repository nach `/config/packages/nibe_gm.yaml`.
3. Öffne die Datei und trage deinen **Außentemperatur-Sensor** bei `entity_id:` ein.
4. Willst du später das Dashboard (Abschnitt 10) nutzen, trage auch den Sensor „Anzahl Starts“ bei `source:` ein (zweimal). Ohne Dashboard kannst du den Block `utility_meter` löschen.
5. Gehe zu *Entwicklerwerkzeuge → YAML* und klicke auf **Konfiguration prüfen**. Starte dann Home Assistant neu.

Danach gibt es diese Entitäten:

- `input_datetime.nibe_gm_sperrzeit_start`
- `counter.nibe_gm_schreibfehler`
- `counter.nibe_gm_schreibzugriffe`
- `sensor.nibe_aussentemperatur_6h_mittel`
- `sensor.nibe_verdichterstarts_heute` und `sensor.nibe_verdichterstarts_woche` (nur fürs Dashboard)

### Weg B: Per Hand in der Oberfläche

Gehe zu *Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen* und lege an:

| Typ | Einstellung |
|---|---|
| Datum und/oder Uhrzeit | Name „Nibe GM Sperrzeit Start“, **Datum und Uhrzeit** einschalten |
| Zähler | Name „Nibe GM Schreibfehler“, Startwert 0 |
| Zähler | Name „Nibe GM Schreibzugriffe“, Startwert 0 |
| Statistik | Quelle: dein Außentemperatur-Sensor, Merkmal **Durchschnitt (zeitgewichtet)**, Höchstalter **6 Stunden**, „Letzten Wert behalten“ **an**, Genauigkeit 2 |

## 5. Schritt 3: Blueprint einspielen

**Am einfachsten:** Klicke oben auf den Knopf „Blueprint in Home Assistant importieren“. Folge den Fragen.

**Oder von Hand:**

1. Kopiere die Datei `blueprints/automation/nibe_gm/startwert_verdichter.yaml` nach `/config/blueprints/automation/nibe_gm/startwert_verdichter.yaml`.
2. Gehe zu *Einstellungen → Automatisierungen & Szenen → Blueprints* (lade die Seite bei Bedarf neu).
3. Wähle beim Blueprint **Automatisierung erstellen**.

## 6. Schritt 4: Automation ausfüllen

> Das Ausfüllen geht nicht von allein. Jede Home-Assistant-Installation benennt die Entitäten anders. Mit dem Entitäten-Finder und den Tabellen unten bist du aber schnell fertig. Die Bereiche 3 und 4 haben schon gute Standardwerte.

### Bereich 1: Wärmepumpe (alle Felder müssen ausgefüllt sein)

| Feld im Blueprint | Wähle |
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

### Bereich 2: Hilfs-Entitäten (alle Felder müssen ausgefüllt sein)

Bei Weg A (fertige Datei) heißen sie so:

| Feld im Blueprint | Wähle |
|---|---|
| Außentemperatur 6-h-Mittel | `sensor.nibe_aussentemperatur_6h_mittel` |
| Zeitstempel letzte Änderung | `input_datetime.nibe_gm_sperrzeit_start` |
| Zähler Schreibfehler | `counter.nibe_gm_schreibfehler` |
| Zähler Schreibzugriffe | `counter.nibe_gm_schreibzugriffe` |

Bei Weg B nimmst du die Namen, die du selbst vergeben hast.

### Bereiche 3 und 4: Stufen und Zeiten (kannst du so lassen)

Standard: Startwerte −300 / −200 / −130, Grenzen bei 5 °C und 0 °C, Puffer 0,75 Grad, Prüfung um 00:00 und 12:00 Uhr mit Wiederholung eine Stunde später, mindestens 11 Stunden zwischen zwei Schreibvorgängen.

**Ändere das nur, wenn du genau weißt, was du tust.** Die drei Startwerte müssen von klein nach groß geordnet sein (zum Beispiel −300, −200, −130). Sonst läuft die Automation nie.

### Bereich 5: Benachrichtigung (empfohlen)

Füge die Aktion „Benachrichtigung senden“ an dein Handy hinzu. In der Nachricht kannst du zwei Platzhalter nutzen: `nachricht_titel` und `nachricht_text`. Im YAML-Modus sieht das so aus:

```yaml
- action: notify.mobile_app_MEIN_HANDY
  data:
    title: "{{ nachricht_titel }}"
    message: "{{ nachricht_text }}"
```

Den Namen deines Handys findest du unter *Entwicklerwerkzeuge → Aktionen*. Suche dort nach `notify.mobile_app`.

### Oder: alles als YAML einfügen

Gehe zu *Einstellungen → Automatisierungen → Neue Automatisierung → drei Punkte → In YAML bearbeiten*. Ersetze den Inhalt durch diesen Text. Passe die Namen mit den Ergebnissen des Finders an:

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

- Schreibe dir vorher deinen Startwert auf.
- Schalte die Automation ein.
- **Vorsicht:** Im Menü der Automation gibt es „Aktionen ausführen“. Das startet die Automation **ohne** Prüfungen. Sie schreibt dann sofort. Nutze das nur, wenn du das wirklich willst.
- Besser: Warte bis zur nächsten Prüfzeit. Oder stelle die Prüfzeit kurz auf ein paar Minuten später.
- Unter **Verlauf (Ablaufverfolgung)** der Automation siehst du, welche Prüfung ein Schreiben gestoppt hat.
- Hat die Automation etwas geändert, bekommst du eine Meldung „Start Verdichter geändert“. Der Zähler „Schreibzugriffe“ geht um 1 hoch.

## 8. Wenn etwas nicht klappt

| Das siehst du | Mögliche Ursache |
|---|---|
| Die Automation läuft, schreibt aber nie | Eine Prüfung ist nicht erfüllt. Schau im Verlauf nach. Oft ist es zu warm (über dem Heizungsstopp), der Zielwert ist schon eingestellt oder die 11 Stunden sind noch nicht um. |
| Der Mittelwert-Sensor zeigt `unknown` oder `unavailable` | Die Quelle ist falsch oder der Sensor ist neu und hat noch keine Werte. |
| Meldung „NICHT bestätigt“ | Die myUplink-Cloud ist langsam oder die Wärmepumpe nimmt den Wert nicht an. Schau in myUplink oder am Display. |
| Die Automation hat sich ausgeschaltet | Zwei Fehler hintereinander. Finde die Ursache. Schalte sie dann von Hand wieder ein. |
| Die Entität „stop heating“ fehlt | Sie ist in myUplink noch aus. Schalte sie ein (Schritt 1). |
| Ein Lauf wurde wegen Warmwasser oder Abtauung übersprungen | Das ist Absicht. Der zweite Versuch eine Stunde später fängt das meistens auf. |

## 9. Zurückstellen

1. Schalte die Automation aus.
2. Stelle den Startwert (Menü 4.9.3) wieder auf den Wert, den du dir vorher aufgeschrieben hast. Das geht am Display oder in der Zahl-Entität.

## 10. Dashboard (optional, einfach)

Mit dem Dashboard siehst du auf einen Blick, was die Wärmepumpe und die Automation machen. Es hat einen **Schalter, mit dem du die GM-Automatik an und aus machst**.

**Was du siehst**

- **Startwert und Gradminuten:** Der eingestellte Startwert (rot) und die Gradminuten jetzt. Dazu eine Linie der letzten 12 Stunden.
- **Außentemperatur:** Wert jetzt, 6-Stunden-Durchschnitt und der Durchschnitt von NIBE.
- **GM-Automatik:** Der Schalter an/aus, wann zuletzt etwas geändert wurde, wie viele Fehler hintereinander es gab und wie oft die Automation geschrieben hat.
- **Verdichter:** Frequenz jetzt und gewünscht, dazu der Verlauf der letzten 24 Stunden.
- **Verdichterstarts:** Starts heute, diese Woche und gesamt. Dazu ein Balkenbild mit den Starts pro Tag.
- **Abtauung:** Ob die Wärmepumpe gerade abtaut.

**So baust du es auf**

1. Gehe zu *Einstellungen → Dashboards → Dashboard hinzufügen* und wähle „Neues Dashboard von Grund auf“. Gib einen Namen ein, zum Beispiel „Wärmepumpe“.
2. Öffne das neue Dashboard. Tippe oben rechts auf den **Stift**, dann auf die **drei Punkte** und auf **Rohkonfigurations-Editor**.
3. Lösche den ganzen Text dort.
4. Öffne die Datei `dashboard/nibe_waermepumpe_dashboard.yaml` aus diesem Repository, kopiere **alles** und füge es im Editor ein.
5. Ersetze jedes Wort mit `DEIN_` durch den echten Namen aus deinem Home Assistant (siehe Tabelle). Die Namen findest du mit dem **Entitäten-Finder** aus Schritt 1.
6. Tippe auf **Speichern**.

**Das musst du ersetzen**

| Im Text steht | Setze ein |
|---|---|
| `number.DEIN_start_compressor` | dein Startwert (endet auf `start_compressor`) |
| `sensor.DEIN_degree_minutes` | deine Gradminuten (endet auf `degree_minutes`) |
| `sensor.DEIN_outdoor_temperature` | deine Außentemperatur |
| `sensor.DEIN_average_outdoor_temp_bt1` | dein NIBE-Durchschnitt |
| `sensor.DEIN_current_compressor_frequency_eb101` | deine Verdichterfrequenz jetzt |
| `sensor.DEIN_requested_compressor_freq_eb101` | deine gewünschte Verdichterfrequenz |
| `sensor.DEIN_number_of_starts` | deine Anzahl Starts |
| `sensor.DEIN_defrosting_eb101_ep14` | deine Abtauung |
| `automation.DEINE_AUTOMATION` | deine Automation aus dem Blueprint |

**Tipp zum Ersetzen:** Kopiere den Text zuerst in einen einfachen Editor (zum Beispiel Notepad) und benutze dort „Suchen und Ersetzen“. Dann geht es schneller und du vergisst nichts.

**Namen aus dem Hilfs-Paket:** `sensor.nibe_aussentemperatur_6h_mittel`, `input_datetime.nibe_gm_sperrzeit_start`, `counter.nibe_gm_schreibfehler`, `counter.nibe_gm_schreibzugriffe`, `sensor.nibe_verdichterstarts_heute` und `sensor.nibe_verdichterstarts_woche` stehen schon richtig im Dashboard. Hast du die Hilfs-Entitäten selbst angelegt (Weg B) und anders benannt, ändere diese Namen auch.

**Wenn etwas nicht klappt**

| Problem | Lösung |
|---|---|
| Eine Kachel zeigt „Entität nicht gefunden“ | Der Name ist falsch geschrieben oder die Entität ist noch aus. Prüfe mit dem Entitäten-Finder. |
| Die Starts heute/Woche sind leer | Du hast den Sensor „Anzahl Starts“ im Hilfs-Paket nicht eingetragen oder Home Assistant nicht neu gestartet. |
| Der Schalter macht nichts | Es steht noch `automation.DEINE_AUTOMATION` drin. Setze den Namen deiner Automation ein. |

Mit dem Schalter machst du die Automation aus, ohne sie zu löschen. Hat sie sich nach zwei Fehlern selbst abgeschaltet, schaltest du sie hier wieder an.

**Zusätzlich:** Willst du nur eine kleine Kachel in einem anderen Dashboard, nimm diesen Text:

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
