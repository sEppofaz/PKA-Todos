# PKA-Todos – Claude-Kontext

## Kerninfos

- **GitHub:** `https://github.com/sEppofaz/PKA-Todos`
- **Live:** `https://seppofaz.github.io/PKA-Todos/`
- **Lokaler Clone:** `~/Developer/PKA-Todos/`
- **Deployment:** `git push` → GitHub Pages automatisch
- **Datendatei:** `/Apps/Claude/Todo-App/Todos.json` in Dropbox (SSOT)
- **Dropbox App-Key:** `s2ggv6zysmzn7fa` (gleicher wie Messwerte- und Rauchmelder-App)
- **Service Worker Cache:** `pka-todos-v12` (in `sw.js` hochzählen bei neuer Version – nur nötig bei Breaking Changes in Assets/Icons/Manifest, nicht bei `index.html`-Änderungen da Network-first)

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
    "prio": "kunde|hoch|mittel|niedrig",
    "kategorie": "pka",
    "erledigt": false,
    "erledigt_am": null,
    "faelligkeit": "YYYY-MM-DD|null",
    "faelligkeit_uhrzeit": "HH:MM|null",
    "rang": -563.5,
    "anhaenge": ["anhaenge/<id>/<uuid>.jpg"],
    "quelle": "claude|app|telegram|siri|skript|feedback"
  }]
}
```

- **`nr`**: Stabile 3-stellige Nummer (100, 101, …). Einmalig vergeben, nie geändert. App zeigt `#nr` an. Bei neuen Todos: `Math.max(...todos.map(t => t.nr||0)) + 1`. Claude und App referenzieren mit `#nr`.
- **`kategorie`**: `pka` | `privat` | `arbeit` (Tabs wieder eingeführt 2026-05-26).
- **`rang`** (seit 2026-10-22, #562): Sortierwert innerhalb der Prio-Gruppe, aufsteigend. Fehlt er, gilt `-nr` → **neueste Nummer oben**. Beim Ziehen bekommt nur die gezogene Karte einen `rang` (Mitte zwischen den neuen Nachbarn, am Rand ±1), alle anderen bleiben bei ihrer Nummer – neue Todos stehen daher immer oben. Prio-Wechsel im Dialog löscht `rang`. Nur die App schreibt das Feld.
- **`order`** (veraltet bis 2026-10-22): alter Positionsindex, wird nicht mehr beachtet. Er klebte gezogene Karten dauerhaft nach oben (#442), und bei gleichem Tag war die Reihenfolge zufällig (#562). Steht noch bei alten Einträgen, stört nicht.
- **`anhaenge`** (seit 2026-10-12, #460): Optional, Liste von Foto-Pfaden **relativ zu `/Apps/Claude/Todo-App/`**. Fehlt das Feld, hat das Todo keine Fotos – nie `null` schreiben, sondern das Feld weglassen.
- **`quelle`** (seit 2026-10-12, #470): Herkunft, beim Anlegen gesetzt – App `app`, Bot `telegram`, iOS-Kurzbefehl `siri`, Server-Skripte (`heimat_import`, `modell_check`) `skript` (alle über `shared/pka_todos.todo_anlegen` im Vereinskalender-Repo). **Claude setzt beim Anlegen immer `"quelle": "claude"`.** Nur `claude` bekommt in der Liste einen Chip „Claude“ (Lucide `bot`) neben der Prio. Bestand ohne Feld bleibt unmarkiert – nicht nachträglich befüllen.
- **`prio: "kunde"`** (seit 2026-10-16, #509): setzt **nur der Server** für Feedback aus dem Vereinskalender (`quelle: "feedback"`). Gruppe „Kunde“ steht über „Hoch“ (Farbe `--kunde`), im Bearbeiten-Dialog erscheint der Knopf nur bei Kunde-Todos – neu wählbar ist die Prio nicht. Unbekannte/fehlende Prio zeigt die App als „Niedrig“ (`prioVon()`); vorher brach `g[t.prio].push` bei unbekannter Prio die ganze Liste. Neue Prio-Werte ⇒ zuerst App deployen, dann schreiben.
- **`faelligkeit`**: Optional; `pka_todos_reminder.py` (Cron alle 15 Min) meldet fällige/überfällige Todos per Telegram.

---

## App-Funktionen

- **Tabs:** PKA / Privat / Arbeit (wieder eingeführt 2026-05-26). Aktiver Tab filtert die Ansicht; neues Todo übernimmt aktiven Tab als Kategorie.
- **Suche (seit 2026-08-15):** 🔍-Icon im Header öffnet Suchfeld. Filtert `db.todos` live, kategorieübergreifend, inkl. erledigter Todos (Substring-Match auf `aufgabe`, case-insensitive). Trefferkarten zeigen Kategorie-Badge, Erledigt-Sektion klappt bei Treffern automatisch auf. Drag & Drop ist während der Suche deaktiviert (Reihenfolge über Kategorien hinweg unsinnig). Tab-Wechsel beendet die Suche automatisch. Nutzt das `.search-wrap`/`#search-clear`-Pflichtpattern aus `PKA/BKM/PWA-Standards.md` (Suchfeld-Lösch-Button-Standard).
- **Prio-Gruppen:** Hoch / Mittel / Niedrig (innerhalb per Drag & Drop sortierbar), Default „Mittel" beim Anlegen (Claude-Regel dazu: `PKA/CLAUDE.md`)
- **Karte antippen** → Bearbeiten-Modal
- **Checkbox** → Todo abhaken (ohne Edit-Modal zu öffnen)
- **🗑-Button** → Löschen (Rückfrage über eigene Karte `frage()` statt `confirm()` – der Safari-Dialog lief im schmalen Fenster aus dem Bild, #474)
- **⠿ Handle** → Drag & Drop innerhalb der Prio-Gruppe (Pointer Events). Seit 2026-10-22 (#562) handytauglich: Griff ca. 29×34 px (negativer Rand, Karte bleibt gleich groß), `touchstart` mit `preventDefault` (`passive:false`) – ohne das startet iOS Scrollen/Textauswahl statt Ziehen –, `pointercancel` beendet sauber, am oberen/unteren Rand scrollt die Liste mit (`dragScroll`). Karten sind `user-select:none` (ein Ziehversuch markierte sonst viel Text; Text kopieren geht im Bearbeiten-Dialog). Erledigte Karten haben keinen Griff.
  **Test ohne Erweiterung:** Chrome headless per CDP (`Emulation.setTouchEmulationEnabled` + `Input.dispatchTouchEvent`, 390×844 mobil) gegen eine Kopie von `index.html`, in der `init()` durch `db = {todos: FIXTURE}; show('app'); render()` ersetzt ist und `save`/`fetch` Attrappen sind. Pitfall: Ein Tippen auf eine Karte öffnet den Bearbeiten-Dialog – vor dem nächsten Ziehtest `closeModal()`.
- **Fälligkeits-Uhrzeit (seit 2026-10-11, #454):** `#m-faell-uhr` ist ein `<input type=time>`, das der Baustein am Dateiende (Kopie von Vereinskalender `src/shared/uhrzeit_ui.py`, Farben `--bg2`/`--accent`) beim Laden zu einem Textfeld macht: Antippen öffnet ein Fenster in der Bildschirmmitte mit zwei Rädern (Stunde, Minute in 5er-Schritten, krumme Bestandszeit bleibt), leeres Feld startet bei 12:00, „Ohne Uhrzeit“ leert. `setUhr()`/`getUhr()` lesen/schreiben nur noch das Feld, gespeichert wird `HH:MM`. Änderungen am Baustein in **beiden** Repos nachziehen. Das Datumsfeld hat `appearance:none` – iOS zeichnete es sonst höher als die übrigen Felder (Josef-Bild 2026-10-11). Vorher (2026-10-10): zwei Auswahllisten 00/15/30/45.
- **Uhrzeit ohne Datum (2026-10-11, #448):** Das leere `<input type=date>` sieht auf dem iPhone ausgefüllt aus (graues Datum) – Josef wählte nur die Uhrzeit, `saveModal()` verwarf sie still, es kam keine Erinnerung. Jetzt setzt eine Uhrzeit-Auswahl bei leerem Datum sichtbar `heuteLokal()` (Ortszeit, nicht `toISOString()` = UTC), `saveModal()` macht dasselbe als Sicherheitsnetz.
- **Fälligkeits-Badges:** ⚠️ überfällig (rot) · 📅 Heute (orange) · 📅 Datum (grau)
- **Erledigt-Sektion** zugeklappt (aufklappbar)
- **Auto-Reload:** `visibilitychange`-Listener ruft `load()` auf wenn App in den Vordergrund kommt → immer aktuell, kein manuelles Reload nötig

---

## Überschrift + Detailtext auf der Karte (seit 2026-09-27, Todo #305)

Lange `aufgabe`-Texte werden auf der Karte nicht mehr vollständig angezeigt. `splitAufgabe(txt)`
zerlegt den Text in `{head, rest}`; `card()` rendert `head` in `.ctext` und `rest` in `.cmore`
(eingeklappt, Chevron-Button `.cexp`, Lucide `chevron-down`).

**Reine Anzeigelogik – `Todos.json` bleibt unverändert.** Kein `titel`-Feld, siehe `ADR/ADR-001`.

**Optik (geändert 2026-09-27):** `.cmore` sieht aus wie `.ctext` – `.9rem`, `line-height:1.4`,
geerbtes `var(--text)`, **keine** Einrückung und kein linker Rahmen, nur `margin-top:6px` als
Trenner. Grund: der verborgene Teil ist der eigentliche Inhalt, keine Fußnote; die alte gedimmte
Zitatblock-Optik war am Handy zu kontrastarm (Dark Mode 4,27:1 → jetzt 12,80:1). Wer hier wieder
eine eigene Textfarbe/-größe einführt, macht genau das rückgängig. Das Durchstreichen erledigter
Todos gilt für **beide** Teile (`.card.done .ctext,.card.done .cmore`) – sonst wäre die Karte
halb durchgestrichen.

Regeln von `splitAufgabe()`:
- Texte ≤ `MIN_SPLIT` (110 Zeichen) bleiben ungeteilt – kein Chevron.
- Überschrift = erste Zeile (wenn 15–90 Zeichen), sonst erstes echtes Satzende, sonst harter
  Schnitt an der Wortgrenze bei `MAX_HEAD` (90) mit „…".
- **Kein Satzende** nach einer Ziffer (`15.07.`, `2. Sep.`), bei bekannten Abkürzungen (`ABBR`-Set:
  `ca`, `bzw`, `inkl`, Monatskürzel …) oder wenn danach kein Großbuchstabe/Ziffer/Listenzeichen folgt.
- Ein Satzende wird bis `MAX_HEAD + 30` akzeptiert – ein vollständiger Satz ist besser als ein
  abgeschnittener.
- `head` wird auf eine Zeile normalisiert (`\s+` → Leerzeichen), `rest` behält Zeilenumbrüche
  (`white-space:pre-wrap`).

**Pitfalls:**
- Der aufgeklappte Zustand liegt in `let expanded = new Set()` (Todo-IDs), **nicht** im DOM –
  sonst würde ihn jedes `render()` (Abhaken, Drag & Drop, Auto-Reload) zurücksetzen.
- Die ganze Karte hat `onclick="showEdit(...)"`. Der Chevron-Button **muss** `event.stopPropagation()`
  aufrufen (`toggleMore(id, ev)`), sonst öffnet Aufklappen das Bearbeiten-Modal.
- **Suche:** Der Filter in `render()` läuft weiter über den Volltext. Steht der Treffer nur im
  verborgenen Teil, klappt `card()` die Karte automatisch auf (`hitInRest`) – sonst wäre der
  gefundene Begriff unsichtbar.
- Beim Prüfen der gerenderten Klasse: erledigte Todos ergeben `class="card done open"`, nicht
  `card open`.

**Test:** `card()` lässt sich ohne Browser gegen den echten Bestand laufen – Script-Blöcke aus
`index.html` per Regex ziehen, in einer `vm`-Sandbox mit DOM-Stubs ausführen, `card(t)` über alle
Todos aus `Todos.json` aufrufen. Beim Umbau von `card()` wieder so verifizieren.

**Pitfall beim Testrezept:** `expanded` ist mit `let` deklariert und liegt daher **nicht** als
Property auf dem Sandbox-Objekt (`sandbox.expanded` ist `undefined`) – mit
`vm.runInContext('expanded', sandbox)` holen. Funktionsdeklarationen wie `card` sind dagegen direkt
am Sandbox-Objekt erreichbar.

**Optik im Browser prüfen:** `file://`-URLs lässt die Chrome-Erweiterung nicht zu, und die Live-App
zu öffnen schreibt beim Start einen Dropbox-Token-Refresh in Josefs `localStorage`. Stattdessen:
CSS + gerenderte Karten in eine eigene Testseite schreiben, per `python3 -m http.server` auf
`127.0.0.1` ausliefern, dort messen (Handybreite per iframe) – und den Server danach beenden.

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

**Prio (geändert 2026-09-27, Todo #409):** Per Telegram oder Siri angelegte Todos bekommen
`prio: "mittel"` (vorher `"niedrig"`). `_save_todo()` setzt außerdem `faelligkeit` und
`faelligkeit_uhrzeit` explizit auf `None`. Quelle: `services/telegram/routes.py` im
**Vereinskalender-Repo** (stellt den `rename-webhook`-Service), nicht in diesem Repo.

**Fälligkeits-Erinnerungen:** `pka_todos_reminder.py` auf Hetzner Server (alle 15 Min via Cron).
- Mit `faelligkeit_uhrzeit`: Erinnerung 15 Min vorher
- Ohne Uhrzeit + überfällige: täglich beim 08:00-Lauf
- Log: `/var/log/pka-todos-reminder.log`

---

## Schema-Pflicht (Pitfall)

- **Claude Code muss beim Anlegen neuer Todos ZWINGEND das Schema aus dieser CLAUDE.md verwenden** – insbesondere `aufgabe` (nicht `text`), `datum` (nicht `erstellt`), `prio`, `nr`.
- Abweichendes Schema (z.B. zusätzliche Felder `text`, `erstellt`, `projekt`) bricht `render()` mit `null.localeCompare()` → gesamter Tab bleibt leer.
- Vorfall 2026-06-08: Todo mit Fremdschema von Claude Code selbst angelegt → PKA-Tab komplett unsichtbar.
- **Vorfall 2026-09-27 (dritte Ausprägung, Feld `nr`):** **102 der 301 Einträge** hatten überhaupt kein Feld `nr` – über Monate hinweg angesammelt (Mai 4, Juni 6, Juli 17, August 26, September 49). Die App zeigt bei ihnen keine `#Nummer`, und im PKA-Logbuch mussten sie über ihre UUID referenziert werden statt über `#nr`, was sie zwischen Sessions praktisch unauffindbar machte. **Die Ursache war eindeutig zuzuordnen:** alle drei produktiven Schreibwege setzen `nr` korrekt – die App (`index.html`, `nr:nextNr`), der Telegram-Bot und der Siri-Webhook (beide über `_save_todo()` in `services/telegram/routes.py`, `max(nr)+1`). Die lückenhaften Einträge stammen also ausnahmslos aus direkten Schreibzugriffen von Claude Code. Am 2026-09-27 nachvergeben: 102 Nummern **307–408**, sortiert nach `datum`; die 8 Lücken im Altbereich (211–213, 218, 219, 234, 235, 289) blieben bewusst frei, weil dort gelöschte Todos standen, deren Nummern im Logbuch referenziert sein können. **Lehre:** Die drei Vorfälle betrafen drei verschiedene Felder derselben Datei. Nach jedem Schreiben in `Todos.json` gegen ein vorher gezogenes Backup verifizieren, dass jeder Eintrag alle Pflichtfelder trägt – nicht nur die selbst angefassten.
- **Vorfall 2026-09-27 (gleiche Regel, zweites Mal gebrochen):** Über mehrere Sessions hinweg wurden Todos mit `prioritaet` statt `prio` angelegt – am Ende **66 Einträge**. Die App bricht dabei nicht sichtbar ab, sondern greift still auf `t.prio || 'niedrig'` zurück: Alle betroffenen Todos erschienen als **„niedrig"**, obwohl 60 davon als „mittel" und eines als „hoch" gemeint waren. Da „niedrig" für Josef das *irgendwann*-Fach ist, bewirkte der Fehler genau das Gegenteil der Prio-Regel aus `PKA/CLAUDE.md`. Am 2026-09-27 migriert (61 umbenannt, bei 5 Einträgen mit beiden Feldern das Altfeld entfernt – `prio` gewinnt, weil die App es zuletzt geschrieben hat). **Lehre:** Der stille Fallback ist gefährlicher als der harte Crash von 2026-06-08 – ein falscher Feldname fällt hier nicht sofort auf. Vor dem Schreiben in `Todos.json` **immer diese CLAUDE.md lesen**, nie das Schema aus einem Stichproben-Eintrag der Datei ableiten (der kann selbst falsch sein).

---

## Service Worker (Pitfall)

- `reg.update()` + `controllerchange` + `location.reload()` verursacht einen Endlos-Reload-Loop, wenn GitHub Pages CDN `sw.js` auch nur minimal anders ausliefert.
- Fix (2026-06-22): SW-Registrierung auf reines `register()` reduziert – der SW ruft `self.skipWaiting()` bereits selbst in seinem Install-Handler auf.
- `load()` hat einen `_loading`-Guard um Race Conditions zwischen `init()` und `visibilitychange`-Listener zu verhindern.
- `dbxGet`/`dbxPut` haben einen `_retried`-Flag um Endlos-Rekursion bei dauerhaft ungültigem Token zu verhindern.
- **Korrektur (2026-08-15):** Diese Zeile war veraltet und widersprach dem tatsächlichen Code. `index.html` läuft **network-first** (fetch → Cache-Fallback nur bei Offline), siehe `sw.js`. CSS/HTML-Änderungen sind daher sofort ohne App-Neustart sichtbar. `CACHE`-Konstante nur bei Breaking Changes an `manifest.json`/Icons hochzählen (Assets laufen Cache-first).

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

---

## Foto-Anhänge (seit 2026-10-12, Todo #460)

- **Ablage:** `/Apps/Claude/Todo-App/anhaenge/<todo-id>/<uuid>.jpg`, im Todo als relativer Pfad in `anhaenge`.
- **App:** Bearbeiten-Dialog → „Kamera“ / „Files“ / „Einfügen“ (`capture="environment"` bzw. `multiple`; „Einfügen“ = `navigator.clipboard.read()`, dazu ⌘V/Strg+V per `paste`-Listener, solange der Dialog offen ist – #471). Alle Wege laufen über `fotosHinzu()`. `fotoVerkleinern()`
  (Muster Vokabeltrainer) skaliert auf max. 1600 px, JPEG 0,85. Hochgeladen wird erst beim **Speichern**
  (`fotosSpeichern()`, `dbxPut(..., 'add')`); schlägt der Upload fehl, bleibt der Dialog mit allen Eingaben offen.
  Bei neuen Todos vergibt `showAdd()` die ID vorab (`mId`), damit der Ordner feststeht.
- **Anzeige:** 48-px-Vorschaubilder auf der Karte, Antippen → Vollbild (`#viewer`). Bild-URLs über
  `files/get_temporary_link` (4 h gültig), gecacht in `linkCache`. `dropboxusercontent.com` ist im `sw.js` vom Cache
  ausgenommen (v12).
- **Löschen:** Abhaken **oder** Löschen eines Todos entfernt das Feld und löscht den Ordner `anhaenge/<id>/`
  (`delete_v2`, Josef-Wunsch). Wird ein abgehaktes Todo wieder geöffnet, sind die Fotos weg – gewollt.
  Fehler beim Löschen werden nur geloggt.
- **Telegram:** Foto an den Bot → Todo mit Anhang, Bildunterschrift = Aufgabe (`#privat`/`#arbeit` wie bei Text,
  ohne Text „Foto-Todo“). Code im Vereinskalender-Repo: `services/telegram/routes.py` (`_telegram_foto()`) und
  `shared/pka_todos.py` (`todo_anlegen(..., foto=)`, lädt das Foto **vor** dem JSON hoch).

## Speichern mit Revisions-Prüfung (seit 2026-10-12)

Bis dahin überschrieb `save()` die Datei blind – ein Todo, das Telegram/Siri zwischen Laden und Speichern anlegte,
ging verloren. Jetzt merkt sich `load()` die `rev` aus dem Header `Dropbox-API-Result`, `save()` schreibt mit
`mode: update(rev)`. Bei **409** holt `_save()` den Serverstand und führt zusammen: lokale Fassung gewinnt bei
gleicher ID, neue Server-Todos kommen dazu (außer lokal gelöschte, `deletedIds`), kollidierende `nr` lokal
neu angelegter Todos rücken ans Ende. Ohne bekannte `rev` → Überschreiben wie früher. `save()` läuft serialisiert
(`_saving`), damit zwei schnelle Aktionen nicht mit derselben `rev` schreiben.

**Test (offline, Node):** Script-Block aus `index.html` in eine `vm`-Sandbox laden, `fetch` durch eine
Dropbox-Attrappe ersetzen (Download mit `Dropbox-API-Result`, Upload mit 409 bei falscher `rev`) und
`saveModal()`/`toggle()`/`del()` durchspielen. So am 2026-10-12 verifiziert (12 Prüfungen) – dabei fiel die
`nr`-Kollision auf. Zusätzlich `card()` über alle echten Todos (353, 0 Fehler).

## ⚠️ Vorfall 2026-10-21: Claude überschreibt App-Todo (#557)

Eine Claude-Session las die lokale Dropbox-Kopie (Stand 16:17), hängte ein Todo mit `max(nr)+1` an und schrieb die
ganze Datei zurück (17:09). Josefs App-Todo #556 lag da nur auf dem Server: Es ging verloren, und die Nummer wurde
doppelt vergeben. Dropbox legte keine Konfliktkopie an. App, Telegram und Siri schreiben mit `update(rev)`, Claude
nicht. Übergangsregel steht in `PKA/CLAUDE.md` (Metadaten-Abgleich direkt vor dem Schreiben). Lösung offen: #557.

## Speichern bei parallelen Schreibern (#479, 2026-10-12)

Bei 409 (Revision veraltet) führt `_save()` zusammen: **lokale Fassung gewinnt nur bei Todos, die in der App seit dem
Laden geändert wurden** (Vergleich mit `basis`, JSON je Todo, gesetzt bei `load()` und nach jedem erfolgreichen Speichern).
Sonst gilt der Server. Vorher gewann lokal immer – ein offenes Fenster mit altem Stand hat am 2026-10-12 das Abhaken von
#470–#478 durch Claude beim Anlegen von #479 rückgängig gemacht. Lokal neue Todos kommen dazu (bei `nr`-Kollision ans
Ende), lokal gelöschte bleiben weg, auf dem Server gelöschte verschwinden auch lokal (außer lokal geändert).
