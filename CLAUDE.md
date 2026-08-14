# PKA-Todos – Claude-Kontext

## Kerninfos

- **GitHub:** `https://github.com/sEppofaz/PKA-Todos`
- **Live:** `https://seppofaz.github.io/PKA-Todos/`
- **Lokaler Clone:** `~/Developer/PKA-Todos/`
- **Deployment:** `git push` → GitHub Pages automatisch
- **Datendatei:** `/Apps/Claude/Todo-App/Todos.json` in Dropbox (SSOT)
- **Dropbox App-Key:** `s2ggv6zysmzn7fa` (gleicher wie Messwerte- und Rauchmelder-App)
- **Service Worker Cache:** `pka-todos-v10` (in `sw.js` hochzählen bei neuer Version – nur nötig bei Breaking Changes in Assets/Icons/Manifest, nicht bei `index.html`-Änderungen da Network-first)

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
- **Prio-Gruppen:** Hoch / Mittel / Niedrig (innerhalb per Drag & Drop sortierbar), Default „Mittel" beim Anlegen (Claude-Regel dazu: `PKA/CLAUDE.md`)
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

## Schema-Pflicht (Pitfall)

- **Claude Code muss beim Anlegen neuer Todos ZWINGEND das Schema aus dieser CLAUDE.md verwenden** – insbesondere `aufgabe` (nicht `text`), `datum` (nicht `erstellt`), `prio`, `nr`.
- Abweichendes Schema (z.B. zusätzliche Felder `text`, `erstellt`, `projekt`) bricht `render()` mit `null.localeCompare()` → gesamter Tab bleibt leer.
- Vorfall 2026-06-08: Todo mit Fremdschema von Claude Code selbst angelegt → PKA-Tab komplett unsichtbar.

---

## Service Worker (Pitfall)

- `reg.update()` + `controllerchange` + `location.reload()` verursacht einen Endlos-Reload-Loop, wenn GitHub Pages CDN `sw.js` auch nur minimal anders ausliefert.
- Fix (2026-06-22): SW-Registrierung auf reines `register()` reduziert – der SW ruft `self.skipWaiting()` bereits selbst in seinem Install-Handler auf.
- `load()` hat einen `_loading`-Guard um Race Conditions zwischen `init()` und `visibilitychange`-Listener zu verhindern.
- `dbxGet`/`dbxPut` haben einen `_retried`-Flag um Endlos-Rekursion bei dauerhaft ungültigem Token zu verhindern.
- **Kein network-first für HTML** (anders als bei den Hetzner-Apps) – SW cached alles inkl. `index.html`. Nach jeder CSS/HTML-Änderung `CACHE`-Konstante in `sw.js` hochzählen, sonst sehen installierte PWAs das Update nicht.

---

## Tab-Leiste (Standard ab 2026-08-14)

`.tabs` (PKA/Privat/Arbeit) ist jetzt am unteren Bildschirmrand fixiert statt im Header (`PKA/BKM/PWA-Standards.md` „Tab-Leiste am unteren Bildschirmrand", Variante A). `z-index:60` bewusst zwischen `header` (10) und `.overlay`-Modal (100) gewählt – Modal muss über der Tab-Leiste liegen. `main`-Padding-bottom von `16px` auf `68px` (+ safe-area) erhöht.

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
- **iOS Zoom-Pitfall:** Input-Felder brauchen `font-size:1rem` (≥16px) – iOS zoomt automatisch rein bei <16px und kehrt nach Speichern nicht zurück. **Fix (2026-05-30):** `closeModal()` setzt Viewport-Meta kurz auf `maximum-scale=1` und entfernt das per `requestAnimationFrame` wieder → erzwingt Zoom-Reset ohne Pinch-Zoom dauerhaft zu sperren.
