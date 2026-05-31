# QA Report — 2026-05-31

**Geprüfte App:** Kasse-App (Single-File `index.html`, React 18 via CDN + Babel-Standalone, PWA mit `sw.js`)
**Methodik:** Live-Preview-Tests bei 375 px / 768 px / 1280 px, Dark-Mode-Emulation, Overflow-Scan (alle Screens), WCAG-Kontrastberechnung der Design-Tokens, Print-CSS-Analyse, Konsolen- & A11y-Check.
**Geprüfte Views:** Kasse (leer + befüllt), Verlauf (Woche/Monat, auf-/zugeklappt), Einstellungen (inkl. Kategorie-Manager), Eintrags-Sheet, Cover/Login.

> **Gesamteindruck:** Solide. **Keine render-brechenden Layout-Bugs** — kein horizontaler Overflow bei irgendeinem Breakpoint, Zentrierung bei `max-width:540px` korrekt, leere Zustände sauber, lange Texte & 7-stellige Beträge brechen/skalieren wie geplant. Die wichtigsten Befunde liegen bei **Barrierefreiheit (Kontrast, Zoom, Labels)** und **mehrseitigem PDF-Druck**.

---

## ✅ Status (2026-05-31): alle umsetzbaren Befunde behoben

Alle 10 Layout-/A11y-Befunde und alle 5 PDF-Befunde wurden gefixt und im Preview verifiziert:

- **Zoom freigegeben** (`maximum-scale` entfernt) · **`color-scheme: light`** ergänzt
- **Kontrast:** `--ink-3` #8C8A82 → **#6A6860** (5.34:1 / 4.64:1 auf `--line`), `--accent` #C25C2E → **#AD4E20** (5.17:1 / 4.69:1 auf `--accent-soft`) — alles ≥ 4.5:1 verifiziert
- **`.kc-saldo-label`** 9 px → 10 px · leeres „—" jetzt `--ink-3`
- **A11y:** `<html lang>` folgt jetzt der Sprache · `aria-label` an Navigationspfeilen, Druck-/Plus-Button und allen Formularfeldern (Datum/Beschreibung/Kategorie/Betrag)
- **Jump-Date-Picker** an den oberen Titelbereich verschoben
- **PDF:** `.card { overflow: visible }` im Druck (kein Clipping mehr auf Folgeseiten) · Tabellentext 12 px → **13 px** (≈10 pt) · `tfoot` als `table-row-group` → „Total" erscheint **einmal** statt pro Seite · tote `tfoot td:first-child`-Regel (font-weight 400) auf 700 angeglichen

**Nicht umgesetzt (bewusst, Architektur):** Babel-im-Browser/unpkg-CDN bleibt — ein Wechsel zu vorkompiliertem JS würde die „Single-File, kein Build"-Architektur aufbrechen, die die App bewusst nutzt. Separat zu entscheiden.

*Befunde unten dokumentieren den ursprünglichen Audit-Stand.*

---

## 🔴 Layout-Bugs (blockierend / hohe Priorität)

> Hinweis: Es wurde **kein** Bug gefunden, der das Rendering bricht oder Inhalte am Bildschirm abschneidet. Die folgende Liste ist nach Schweregrad priorisiert; Schwerpunkt ist Accessibility/Lesbarkeit.

| # | Problem | Datei / Selector | Schweregrad |
|---|---------|------------------|-------------|
| 1 | **Pinch-Zoom deaktiviert** — `maximum-scale=1.0` verhindert das Vergrössern (WCAG 1.4.4). Auf einem Kassenbuch mit kleinen Zahlen besonders relevant. | `index.html:5` (viewport-meta) | Hoch |
| 2 | **`--ink-3` (#8C8A82) = Kontrast 3.31:1** auf Hintergrund — unter 4.5:1. Wird pervasiv für **Kleintext** genutzt: Datum (`.kc-date`), Kicker, `.srow-sub`, Saldo-Label, Eintragszähler, Total-Labels (`.kt-row`), Leerzustand-Text. | `index.html:21` (Token); Verwendung u.a. `.kc-date:278`, `.kt-row:290`, `.srow-sub:319` | Mittel |
| 3 | **`.kc-saldo-label` 9 px + `--ink-3`** — winzige Schrift **und** niedriger Kontrast (3.31:1) kombiniert. Auf dem Handy kaum lesbar. | `index.html:287` | Mittel |
| 4 | **Formularfelder ohne `<label>`** — 5 Inputs (Datum, Beschreibung, Kategorie, Betrag, Jump-Datum), **0** programmatisch verknüpfte Labels. Felder werden nur durch optische `.kicker`-Divs beschriftet → Screenreader nennt keine Feldnamen. | `EntryEditor` ~`index.html:1057`–`1115` | Mittel |
| 5 | **Navigations-Pfeile ohne Namen** — Vor/Zurück-Buttons (Woche/Monat) sind icon-only, **ohne `aria-label`/`title`**. Für Screenreader unbenannt. | `index.html:1282`, `index.html:1294` | Mittel |
| 6 | **`--accent` (#C25C2E) = 4.13:1** auf BG / 4.32:1 auf Weiss — knapp unter 4.5:1. Betrifft kleine Kategorie-Chips (`.kc-cat`, 10 px uppercase) & aktuellen-Woche-Kicker. | `index.html:25` (Token); `.kc-cat:283` | Niedrig–Mittel |
| 7 | **`--accent` auf `--accent-soft` = 3.74:1** — Custom-Kategorie-Chips & ×-Löschbuttons in den Einstellungen (oranger Text auf hellorangem Grund). | `CategoryManager` (Settings), `accent-soft` Token `index.html:26` | Niedrig |
| 8 | **`<html lang="de">` bleibt statisch** bei Sprachwechsel (EN/IT/FR). `document.title` wird aktualisiert, `lang` nicht → falsche Sprachauszeichnung für Screenreader/SEO. | `index.html:2`; Effect `index.html:1820` | Niedrig |
| 9 | **Jump-Date-Picker erscheint mittig-unten** — verstecktes Input liegt bei `left:50%; bottom:80`. Auf Desktop-Chromium öffnet der native Picker an dieser Position statt am Titel → wirkt deplatziert (keine Klick-Blockade, da kein Backdrop auf dem Kasse-Screen). | `index.html:2140`–`2142` | Niedrig |
| 10 | **`--ink-4` (#C2BFB6) = 1.76:1** — Platzhalter & „—" (leere Custom-Kategorien) kaum sichtbar. Bei Platzhaltern tolerierbar, beim „—"-Leerhinweis grenzwertig. | `index.html:22` (Token) | Niedrig |

---

## 🟡 PDF / Print-Probleme

| # | Problem | Ursache | Fix-Hinweis |
|---|---------|---------|-------------|
| 1 | **Mehrseitige Tabelle kann auf Folgeseiten abgeschnitten werden** | `.card { overflow: hidden }` (`index.html:326`) wird im `@media print` **nicht** zurückgesetzt — die `.card.print-only` umschliesst die Tabelle. Bei vielen Einträgen (langer Monat) reicht eine A4-Seite nicht. | Im `@media print`-Block ergänzen: `.card { overflow: visible !important; }` (neben den bestehenden border/shadow-Resets ~`index.html:374`). |
| 2 | **Drucktext nur ~9 pt** | `.kasse-table { font-size:12px }` (`index.html:375`) + `td` 12 px = **9 pt**, unter der 10-pt-Empfehlung der Checkliste. | Druck-`td`/Zahlenzellen auf **13 px (≈10 pt)** anheben. Spaltenköpfe sind bereits 12 px/fett (`index.html:376`). |
| 3 | **„Total"-Zeile wiederholt sich auf jeder Druckseite** | `<tfoot>` rendert als `table-footer-group` und wird per Default am Fuss **jeder** Seite gedruckt. Bei mehrseitiger Tabelle erscheint Total mehrfach. | Falls Total nur einmal am Ende gewünscht: Total-Zeile aus `<tfoot>` in normalen `<tbody>`-Bereich verschieben, oder bewusst akzeptieren. Nur relevant bei >1 Seite. |
| 4 | **Kein expliziter Seitenumbruch-Schutz für Kopf/Total** | Nur `tr { break-inside: avoid }` (`index.html:384`) ist gesetzt. Print-Header & Total haben keinen `break-after/before`-Schutz; in Kombination mit Problem 1 (`overflow:hidden`) kann der Tabellenkopf fehlen. | Nach Fix von #1 erneut prüfen; ggf. `thead { display: table-header-group }` (Default) verifizieren, damit Kopf je Seite wiederholt. |
| 5 | **Widersprüchliche/tote CSS-Regel für Total-Label** | `.kasse-table tfoot tr td:first-child { font-weight:400 }` (`index.html:270`) widerspricht dem neuen Inline-`fontWeight:700`. Inline gewinnt (Total **ist** fett) — aber die Regel ist irreführend/tot. | Regel `index.html:270` auf `font-weight:700; color:var(--ink)` angleichen oder entfernen. |
| 6 | **Hintergrundfarben drucken — OK** | `-webkit-print-color-adjust: exact` ist gesetzt (`index.html:353`). | Kein Handlungsbedarf — als bestanden vermerkt. |
| 7 | **URLs hinter Links — N/A** | Druckinhalt enthält keine `<a href>` (Firma/Adresse/Telefon/E-Mail sind Plaintext). | Kein Bug; `a[href]::after`-Regel nicht nötig. |
| 8 | **A4-Breite gesetzt — OK** | `@page { size: A4 portrait; margin: 0 }` + `.app { padding: 14mm 13mm }` (`index.html:362,385`). Nutzbare Breite ~184 mm, Tabelle `width:100%`. | Kein Handlungsbedarf. |

---

## 🟢 Quick Wins (< 30 Min)

- **`maximum-scale=1.0` entfernen** (`index.html:5`) → Pinch-Zoom wieder erlauben (1 Wort löschen, behebt Layout-Bug #1).
- **`.card { overflow: visible !important }`** im `@media print` ergänzen → behebt mehrseitiges PDF-Clipping (PDF #1, das wichtigste Druck-Risiko).
- **`<html lang>` dynamisch setzen**: im bestehenden Sprach-Effect (`index.html:1820`) zusätzlich `document.documentElement.lang = state.settings.lang;` — analog zur Title-Aktualisierung.
- **`--ink-3` leicht abdunkeln** (z.B. #8C8A82 → ~#6E6C64) → erreicht 4.5:1 für den vielen Kleintext (Layout #2/#3). Ein-Token-Änderung, wirkt app-weit.
- **`--accent` minimal abdunkeln** (#C25C2E → ~#B5511F) → 4.5:1 für Kategorie-Chips/Kicker (Layout #6).
- **`aria-label`** an Navigations-Pfeile (`index.html:1282/1294`) und Print/Add-Buttons (haben `title`, aber `aria-label` ist zuverlässiger).
- **tfoot-Regel `index.html:270`** auf `font-weight:700` angleichen → entfernt toten/widersprüchlichen Stil (PDF #5).
- **Druck-Tabellenschrift auf 13 px** (`index.html:375`) → ≥10 pt Lesbarkeit (PDF #2).
- **Leeres Custom-Kategorie-„—"** statt `--ink-4` (1.76:1) ein dunkleres Grau geben (Layout #10).

---

## Beobachtungen / Architektur (kein akuter Bug)

- **Babel-im-Browser + unpkg-CDN zur Laufzeit** (`index.html:394`–`396`): Konsole warnt durchgehend „precompile for production". JSX wird bei **jedem** Laden im Browser kompiliert → langsamerer Erststart (v.a. Mobile) und Laufzeit-Abhängigkeit von unpkg. Der Service-Worker cached zwar nach dem ersten Besuch, aber der **allererste** Aufruf braucht das CDN. Für eine produktive Garagen-App ggf. einmalig vorkompilieren/bündeln.
- **Kein Dark-Mode** (bewusst): Bei `prefers-color-scheme: dark` bleibt die App hell (cremefarbenes Token-Set, native Inputs explizit hell gesetzt) — **nichts bricht**, aber `color-scheme` ist nicht deklariert, daher könnten native Picker-Innereien (Datums-Spinner) auf manchen Browsern dunkles Chrome zeigen.
- **Hardcodierte Hex-Werte** bei den Einnahme/Ausgabe-Knöpfen im Sheet (#f0fdf4, #dcfce7, #86efac, #fca5a5, #fef2f2, #fee2e2) statt Tokens — konsistent mit dem Light-Design, aber nicht token-basiert (relevant, falls je ein Theming kommt).

---

## Acceptance Criteria
- [x] Alle drei Sektionen befüllt
- [x] Jeder Befund hat eine Datei-/Selector-Referenz
- [x] `qa-report.md` im Projektroot gespeichert

## Progress
- ✅ **Sektion 1 (Visuelles Layout)** — 10 Befunde (0 render-brechend; Schwerpunkt Kontrast & A11y)
- ✅ **Sektion 2 (PDF/Print)** — 5 Befunde + 3 bestanden/N-A
- ✅ **Sektion 3 (Funktional)** — Konsole sauber (nur Babel-Dev-Warnung); keine `<p>`/`<img>`-Strukturfehler; CDN/Dark-Mode/Hardcoded-Farben als Beobachtungen vermerkt
