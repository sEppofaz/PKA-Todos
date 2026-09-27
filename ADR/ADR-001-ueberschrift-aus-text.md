# ADR-001: Überschrift aus dem Aufgabentext ableiten statt eigenes Feld

**Datum:** 2026-09-27
**Status:** aktiv
**Projekt:** PKA-Todos

## Problem

Die Todo-Karten zeigten den vollständigen `aufgabe`-Text. Durch die Praxis, Todos mit
„Update YYYY-MM-DD: …"-Ketten fortzuschreiben, waren von 29 offenen Todos **12 länger
als 200 Zeichen**, 9 länger als 400, das längste **1.808 Zeichen**. Eine einzelne Karte
füllte damit mehrere Bildschirmhöhen – die Liste war nicht mehr überfliegbar, was der
Zweck der App ist.

## Entscheidung

Die Karte zeigt eine **aus dem Text selbst abgeleitete Überschrift**; der Rest steht in
einem eingeklappten Block darunter (`splitAufgabe()` in `index.html`). `Todos.json`
bleibt unverändert – die Zerlegung ist reine Anzeigelogik zur Laufzeit.

## Begründung

Der Altbestand wirkt sofort: alle 300 vorhandenen Todos bekommen ohne Nacharbeit eine
Überschrift (209 davon mit Aufklapp-Block). Ein zusätzliches Datenfeld hätte dagegen nur
für künftige Todos gewirkt und den Altbestand – also genau die langen Einträge, die das
Problem verursachen – unangetastet gelassen.

Zudem schreiben **drei Quellen** in `Todos.json` (App, Telegram-Bot, Siri-Webhook, dazu
Claude Code). Jedes neue Pflichtfeld vergrößert die Fläche für genau den Schema-Fehler,
der sich in diesem Projekt bereits zweimal materialisiert hat (2026-06-08 `text`/`erstellt`,
2026-09-27 `prioritaet` statt `prio`). Eine Anzeigeregel ohne Schemaänderung hat diese
Fehlerfläche nicht.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Neues Feld `titel` in `Todos.json` | Wirkt nur für neue Todos; der Altbestand mit den langen Texten bliebe unverändert. Zusätzlich ein viertes Pflichtfeld über vier Schreibquellen hinweg – siehe die zwei Schema-Vorfälle des Projekts. |
| Claude generiert Überschriften einmalig per API | Verursacht API-Kosten für ein Anzeigeproblem, das eine Heuristik löst. Ergebnis wäre außerdem eingefroren: bei jeder Textänderung veraltet die Überschrift still. |
| Reines CSS `line-clamp` | Kappt mitten im Wort, ohne Satzgrenze, und der verborgene Text bleibt unerreichbar, solange die Karte nicht ins Edit-Modal führt. Kein Aufklappen möglich. |
| Volltext nur im Edit-Modal, Karte immer gekürzt | Zum Nachlesen müsste man in den Bearbeiten-Zustand wechseln – ein Schreibmodus für eine Lesefrage, mit dem Risiko versehentlicher Änderungen. |

## Gilt unter

Die Heuristik setzt voraus, dass der Anfang eines Todo-Textes die Aufgabe benennt und
Nachträge hinten angehängt werden. Das trifft auf den heutigen Bestand zu. Würden Todos
künftig überwiegend mit Kontext beginnen und die Aufgabe erst am Ende nennen, wäre die
Überschrift wertlos und ein gepflegtes `titel`-Feld die bessere Wahl.

## Konsequenzen

**Positiv:** Liste wieder überfliegbar, ohne Datenmigration und ohne Schemaänderung; wirkt
rückwirkend auf den gesamten Bestand; keine laufenden Kosten.

**Negativ:** Die Überschrift ist geraten, nicht gepflegt – bei ungünstig formulierten Texten
kann sie unscharf ausfallen. Die Heuristik trägt Sonderfälle mit sich (Datumsangaben wie
„15.07.", Abkürzungsliste), die bei neuen Textmustern nachgezogen werden müssen. Ein
Suchtreffer, der nur im verborgenen Teil steht, muss beim Rendern erkannt und die Karte
automatisch aufgeklappt werden – sonst wäre der Treffer unsichtbar.
