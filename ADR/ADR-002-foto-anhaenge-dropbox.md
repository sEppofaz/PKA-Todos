# ADR-002: Foto-Anhänge als Dateien in Dropbox, Pfad im Todo

**Datum:** 2026-10-12
**Status:** aktiv
**Projekt:** PKA-Todos

## Problem
Josef will an Todos Fotos hängen (Todo #460) – in der App per Kamera/Galerie und per Foto an den Telegram-Bot.
Die App hat kein Backend; sie spricht direkt mit Dropbox, `Todos.json` ist die einzige Datenquelle.

## Entscheidung
- Fotos liegen als JPEG unter `/Apps/Claude/Todo-App/anhaenge/<todo-id>/<uuid>.jpg`.
- Das Todo führt sie im optionalen Feld `anhaenge` als Pfade **relativ** zum Todo-App-Ordner.
- Die App verkleinert vor dem Upload auf max. 1600 px (JPEG 0,85). Angezeigt wird über `files/get_temporary_link`.
- Abhaken oder Löschen eines Todos löscht den ganzen Ordner `anhaenge/<id>/` (Josef: „Beim Abhaken kann das Bild
  auch gelöscht werden“).
- Gleichzeitig: `Todos.json` wird mit Revisions-Prüfung gespeichert, bei Konflikt zusammengeführt.

## Begründung
- `Todos.json` bleibt klein (~200 KB), Laden/Speichern bleibt schnell. Ein Foto pro Todo hätte die Datei um ein
  Vielfaches vergrößert.
- Ein Ordner je Todo macht das Aufräumen zu einem einzigen `delete_v2`-Aufruf.
- Relative Pfade halten den Ordner verschiebbar und sind in App und Bot identisch.
- Mit Fotos dauert ein Bearbeiten länger – das Fenster für parallele Telegram-Schreibvorgänge wuchs. Ohne
  Revisions-Prüfung gingen solche Todos verloren.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Bild als Base64 in `Todos.json` | Datei wächst um MB je Foto; jeder Ladevorgang (auch Reminder-Cron, Telegram) zieht alle Bilder mit |
| Upload über einen Hetzner-Endpunkt | Neuer Dienst mit Auth nur für Uploads – die App kann mit ihrem Dropbox-Token direkt schreiben |
| Fotos beim Abhaken behalten | Josef wollte das Löschen; erledigte Todos brauchen keine Belegfotos |
| Ein gemeinsamer Ordner ohne Unterordner je Todo | Aufräumen bräuchte Listen + Einzel-Löschen |

## Gilt unter
- Fotos sind Gedächtnisstützen, keine Belege, die aufbewahrt werden müssen.
- Wenige Todos haben Fotos. Bei vielen Vorschaubildern pro Ansicht würden die `get_temporary_link`-Aufrufe spürbar.

## Konsequenzen
- Wird ein abgehaktes Todo wieder geöffnet, sind die Fotos weg (gewollt).
- Schlägt das Löschen fehl, bleibt ein verwaister Ordner (nur geloggt). Unkritisch, gelegentlich aufräumbar.
- Schreibt ein anderer Weg (Claude, Skript) Todos mit Fotos, muss er dasselbe Schema einhalten (siehe CLAUDE.md).
