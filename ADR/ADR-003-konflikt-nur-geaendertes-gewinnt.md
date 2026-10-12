# ADR-003: Bei Speicher-Konflikt gewinnt nur lokal Geändertes

**Datum:** 2026-10-12
**Status:** aktiv
**Projekt:** PKA-Todos (Todo #479-Fund)

## Problem
Todos.json hat mehrere Schreiber (App, Telegram-Bot, Siri-Webhook, Claude). Bei 409 (Revision veraltet) führte die App
zusammen und ließ ihre Fassung **jedes** Todos gewinnen. Ein seit Stunden offenes Fenster setzte so zehn von Claude
abgehakte Todos wieder auf offen.

## Entscheidung
Beim Laden und nach jedem erfolgreichen Speichern merkt sich die App je Todo den Ausgangsstand (`basis`, JSON je id). Bei
Konflikt gewinnt die lokale Fassung nur, wenn sie davon abweicht; sonst gilt der Server. Lokal neue kommen dazu
(nr-Kollision → ans Ende), lokal gelöschte bleiben weg, auf dem Server gelöschte und lokal unveränderte verschwinden.

## Begründung
Minimaler Eingriff in `_save()`, kein Server nötig, deckt alle bekannten Schreiber ab. Offline mit 7 Fällen geprüft.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Lokal gewinnt immer (bisher) | Macht fremde Änderungen rückgängig |
| Server gewinnt immer | Eigene Änderungen gingen bei jedem Konflikt verloren |
| Feldweise Zusammenführung je Todo | Mehr Logik für seltenen Fall (gleiches Todo gleichzeitig an zwei Stellen geändert) |
| Zeitstempel je Todo (`geaendert_am`) | Braucht Änderungen an allen Schreibern (Bot, Webhook, Claude) |

## Gilt unter
Gleichzeitige Änderung desselben Todos an zwei Stellen bleibt „lokal gewinnt“. Erst bei häufigen Fällen feldweise lösen.

## Konsequenzen
Muster steht als Standard in `PKA/BKM/Atomic-Write-Pattern.md`. Ein altes, noch offenes App-Fenster mit der Vorversion
kann bis zum Neuladen weiter überschreiben.
