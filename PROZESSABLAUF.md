# Prozessablauf – PKA-Todos

---

## Hauptflüsse

**1. Todo per Browser erstellen**
App öffnen → Dropbox-Auth (PKCE) → `Todos.json` laden → Neues Todo eingeben (Kategorie, Priorität, Fälligkeit + Uhrzeit) → Dropbox speichern → sofort sichtbar.

**2. Todo per Telegram erstellen**
Beliebiger Text an den Bot → Bot schreibt direkt in `Todos.json` in Dropbox → erscheint sofort in der App.

**3. Todo erledigen**
Todo antippen → als erledigt markieren → `Todos.json` aktualisieren → Dropbox speichern.

**4. Sortieren**
Drag & Drop innerhalb einer Prioritätsgruppe → neue Reihenfolge in `Todos.json` speichern.

---

## Automatische Erinnerung

Cron alle 15 Min → `pka_todos_reminder.py` → `Todos.json` aus Dropbox laden → fällige Todos prüfen (Datum + Uhrzeit) → Telegram-Benachrichtigung 15 Min vor Fälligkeit.

## Beteiligte Dateien

| Datei | Zugriff |
|-------|---------|
| `Todos.json` (Dropbox) | Lesen + Schreiben (SSOT aller Todos) |

## Externe Dienste

| Dienst | Zweck |
|--------|-------|
| Dropbox API | Datenhaltung (OAuth PKCE, kein Backend) |
| Telegram Bot | Todo-Erstellung + Fälligkeitserinnerungen |
