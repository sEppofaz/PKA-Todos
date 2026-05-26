# PKA-Todos – Claude-Kontext

## Kerninfos

- **GitHub:** `https://github.com/sEppofaz/PKA-Todos`
- **Live:** `https://seppofaz.github.io/PKA-Todos/`
- **Lokaler Clone:** `~/Developer/PKA-Todos/`
- **Deployment:** `git push` → GitHub Pages automatisch
- **Datendatei:** `/Apps/Claude/Todo-App/Todos.json` in Dropbox (SSOT)
- **Dropbox App-Key:** `s2ggv6zysmzn7fa` (gleicher wie Messwerte- und Rauchmelder-App)
- **Service Worker Cache:** `pka-todos-v9` (in `sw.js` hochzählen bei neuer Version – nur nötig bei Breaking Changes in Assets/Icons/Manifest, nicht bei `index.html`-Änderungen da Network-first)

### Deployment-Flow

```bash
cd ~/Developer/PKA-Todos
git add . && git commit -m "Beschreibung" && git push
# GitHub Pages deployed automatisch
```

---

## Todos.json – Schema

```json
{
  "v": 1,
  "todos": [{
    "id": "uuid",
    "nr": 118,
    "datum": "YYYY-MM-DD",
    "aufgabe": "Text",
    "prio": "hoch|mittel|niedrig",
    "kategorie": "pka",
    "erledigt": false,
    "erledigt_am": null,
    "faelligkeit": "YYYY-MM-DD|null",
    "faelligkeit_uhrzeit": "HH:MM|null",
    "order": 0
  }]
}
```

- **`nr`**: Stabile 3-stellige Nummer (100, 101, …). Einmalig vergeben, nie geändert. App zeigt `#nr` an. Bei neuen Todos: `Math.max(...todos.map(t => t.nr||0)) + 1`. Claude und App referenzieren mit `#nr`.
- **`kategorie`**: `pka` | `privat` | `arbeit` (Tabs wieder eingeführt 2026-05-26).
- **`order`**: Manuelle Sortierreihenfolge (ganze Zahl, DOM-global); fehlt → Sortierung nach Datum. Wird via Drag & Drop gesetzt.
- **`faelligkeit`**: Optional; `pka_todos_reminder.py` (Cron alle 15 Min) meldet fällige/überfällige Todos per Telegram.

---

## App-Funktionen

- **Tabs:** PKA / Privat / Arbeit (wieder eingeführt 2026-05-26). Aktiver Tab filtert die Ansicht; neues Todo übernimmt aktiven Tab als Kategorie.
- **Prio-Gruppen:** Hoch / Mittel / Niedrig (innerhalb per Drag & Drop sortierbar)
- **Karte antippen** → Bearbeiten-Modal
- **Checkbox** → Todo abhaken (ohne Edit-Modal zu öffnen)
- **🗑-Button** → Löschen
- **⠿ Handle** → Drag & Drop innerhalb der Prio-Gruppe (Pointer Events, iOS-kompatibel)
- **Fälligkeits-Badges:** ⚠️ überfällig (rot) · 📅 Heute (orange) · 📅 Datum (grau)
- **Erledigt-Sektion** zugeklappt (aufklappbar)
- **Auto-Reload:** `visibilitychange`-Listener ruft `load()` auf wenn App in den Vordergrund kommt → immer aktuell, kein manuelles Reload nötig

---

## Telegram-Integration

Beliebiger Text → `kategorie: pka`. Mit `#privat` oder `#arbeit` **irgendwo im Text** → entsprechende Kategorie; Hashtag wird aus dem Todo-Text entfernt.
Beispiele: `Zahnarzt Termin #privat` oder `#arbeit Angebot schreiben` oder `Meeting #arbeit morgen`.
Außerdem: Bot setzt jetzt korrekte `nr` (max+1) beim Anlegen via Telegram.

**Siri/Apple Watch – Webhook-Endpoint:**
`POST https://umbenennen.duckdns.org/webhook/todo`
Header: `X-Token: <TODO_WEBHOOK_SECRET aus secrets.env>`
Body: `{"text": "Todo-Text #privat"}`
Gleiche Hashtag-Logik wie Telegram. Shortcut-Name auf iPhone: „Todo" → „Hey Siri, Todo Zahnarzt Termin #privat"

**Fälligkeits-Erinnerungen:** `pka_todos_reminder.py` auf Hetzner Server (alle 15 Min via Cron).
- Mit `faelligkeit_uhrzeit`: Erinnerung 15 Min vorher
- Ohne Uhrzeit + überfällige: täglich beim 08:00-Lauf
- Log: `/var/log/pka-todos-reminder.log`

---

## Token-Handling (Pitfall)

- Dropbox Offline-Token (`token_access_type: offline`) läuft nach 12 Monaten Inaktivität ab
- `init()` ruft bei Start immer `refresh()` auf → bei Fehler: Token aus localStorage löschen + Setup-Screen zeigen
- Bei leerem Setup-Screen: „Mit Dropbox verbinden →" klicken, OAuth durchlaufen

---

## Modal (Hinzufügen / Bearbeiten)

- Modal schwebt **zentriert** (`align-items:center; justify-content:center`) – kein Bottom-Sheet
- Grund: Fälligkeits-Datumsfeld wurde am unteren Bildschirmrand abgeschnitten (Mac-Browser)
- `max-height:90svh; overflow-y:auto` → scrollbar auf kleinen Bildschirmen
- **iOS Zoom-Pitfall:** Input-Felder brauchen `font-size:1rem` (≥16px) – iOS zoomt automatisch rein bei <16px und kehrt nach Speichern nicht zurück.
