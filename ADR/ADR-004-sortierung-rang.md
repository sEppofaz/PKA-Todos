# ADR-004: Sortierung nach Nummer, Ziehen setzt nur einen Rang

**Datum:** 2026-10-22
**Status:** aktiv
**Projekt:** PKA-Todos

## Problem
Innerhalb einer Prio-Gruppe galt zuerst das Feld `order`. Das war ein Positionsindex über die ganze Seite und wurde beim Ziehen gesetzt. Gezogene Karten blieben dadurch für immer oben, so stand #442 vom 10.10. über neueren Todos. Danach wurde nur nach dem Tag sortiert, Todos vom selben Tag standen zufällig (#562). Am Handy ging Ziehen gar nicht, es markierte nur Text.

## Entscheidung
Sortiert wird nach `rang` aufsteigend, und ohne `rang` gilt `-nr`, also die neueste Nummer oben. Beim Ziehen bekommt nur die gezogene Karte einen `rang`: die Mitte zwischen den neuen Nachbarn, am Rand ±1. Ein Prio-Wechsel löscht `rang`, `order` wird ignoriert. Ziehen ist handytauglich gemacht (siehe `CLAUDE.md`).

## Begründung
Josef will die Reihenfolge nach Nummer und trotzdem ziehen können. Mit einem Rang je Karte ändert sich beim Ziehen genau ein Todo. Das hält Konflikte beim Zusammenführen paralleler Schreiber klein (ADR-003). Neue Todos landen ohne eigene Logik oben.

## Verworfen

| Alternative | Warum verworfen |
|---|---|
| Ziehen ganz entfernen | Josef will Ziehen behalten (2026-10-22). |
| `order` behalten, nur bei gleichem Wert nach `nr` | Gezogene Karten kleben weiter oben, neue Todos landen unter allen gezogenen. |
| Nach dem Ziehen allen Karten der Gruppe neue Indizes geben | Ändert viele Todos auf einmal, macht bei 409 viele „lokal geändert“ und überstimmt den Server. |
| Alte `order`-Werte aus der Datei löschen | Ist nicht nötig, weil das Feld ignoriert wird. Ein Massen-Schreiben brächte dasselbe Risiko wie in der Zeile oben. |

## Gilt unter
`nr` wird fortlaufend vergeben. Bei sehr häufigem Ziehen zwischen dieselben zwei Karten halbiert sich der Abstand jedes Mal. Erst nach etwa 50 Halbierungen hintereinander wäre die Genauigkeit erschöpft, das ist praktisch irrelevant.

## Konsequenzen
+ Eindeutige Reihenfolge, neue Todos oben, Ziehen bleibt.
− Ein gezogenes Todo steht über später angelegten nur dann, wenn es ganz nach oben gezogen wurde und keine neuere Nummer kommt. Neue Todos gehen immer an die Spitze.
