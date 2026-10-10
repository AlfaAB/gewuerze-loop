# Gewürz-Radar: wöchentliche Claude Code Routine

Ein wöchentlicher, unbeaufsichtigter Claude-Code-Loop, der den Markt beobachtet, aus
dem ein Gewürzunternehmen lebt. Er:

1. recherchiert mit 4 parallelen Research-Subagenten die Woche in der Gewürzwelt
   (Markt, Recht, Rohstoff, Produkt),
2. wählt daraus die 3 wichtigsten Entwicklungen aus plus 0–3 kleinere "Signale vom
   Rand", ohne Wiederholungen aus den letzten Wochen; dafür sorgt eine State-Datei,
3. baut daraus eine HTML-Email,
4. legt dieselbe Ausgabe als Daten für die Website ab (`data/weeks/<KW>.json`),
5. prüft Auswahl und Dateien vor dem Versand (Quellen, Dubletten, Plausibilität),
6. verschickt die Email über den **Gmail-Connector**,
7. und committet alles ins Repo. Mit diesem Push ist die Ausgabe auch auf der Website.

Der Zwilling dieses Repos ist [ki-news-loop](https://github.com/AlfaAB/ki-news-loop).
Beide liefern dieselbe Website-Struktur, nur zu einem anderen Thema.

## Die Perspektive

Alles hier ist aus der Sicht eines deutschen Betriebs geschrieben, der **eigene
Mischungen entwickelt und herstellt** und Produkte auf den Markt bringt: als Zulieferer
für Lebensmittelhersteller und Gastronomie (B2B) und als Marke im Handel (B2C).
Wettbewerb, Handel und Recht werden deutsch und EU-weit betrachtet, Rohstoffe weltweit.

Daraus folgt das Auswahlkriterium: Ein Thema kommt hinein, wenn es für diesen Betrieb
eine **Folge** hat (Pflicht, Frist, Kosten, Risiko) oder eine **Idee** hergibt (ein
Produkt, eine Mischung, eine Vermarktung). Jeder Eintrag beantwortet beides. Themen
ohne beides fliegen raus, auch wenn sie interessant sind — und ein Thema muss nicht das
Wort "Gewürz" enthalten, um wichtig zu sein.

## Die vier Bereiche

| Bereich | Frage | Inhalt |
|---|---|---|
| `markt` | Wer bewegt sich? | Neue Firmen und Marken, Übernahmen, Listungen, Eigenmarken, Launches der Wettbewerber, Marktzahlen |
| `regularien` | Was müssen wir einhalten? | Verpackungsrecht, Kennzeichnung, Grenzwerte, Rückrufe und RASFF, Bio und Zertifizierung, Leitsätze |
| `rohstoff` | Was kostet es, kriegen wir es? | Ernten, Preise, Verfügbarkeit, Herkunftsländer, Verfälschung, Logistik |
| `produkt` | Was will der Markt als Nächstes? | Geschmackstrends, Produktkonzepte, Messeneuheiten, Konsumstudien |

Auf der Website sind das die vier Sektoren des Radars.

## Die Beobachtungsliste

`config/watchlist.json` ist der Ort, an dem die Ausrichtung gesteuert wird, **ohne
Prompts anzufassen**. Die Research-Agenten klopfen jede Woche zusätzlich zur freien
Suche ab, was dort steht: Wettbewerber, Verbände und Behörden, Messen, Dauerthemen und
die Rohstoffe, die im Blick bleiben sollen.

Die mitgelieferte Liste ist ein **Vorschlag zum Überschreiben**. Streichen, was nicht
interessiert, ergänzen, was fehlt, besonders die eigenen Wettbewerber. Je genauer die
Liste, desto brauchbarer die Ausgabe.

## Wie die Website an die Daten kommt

Die Website liegt auf Antons eigenem Server und lädt ihre Daten beim Aufruf direkt aus
diesem Repo:

```
https://raw.githubusercontent.com/alfaab/gewuerze-loop/main/data/index.json
https://raw.githubusercontent.com/alfaab/gewuerze-loop/main/data/weeks/<KW>.json
```

Es gibt keinen Build, kein Deployment und keinen Job auf dem Server. Sobald die Routine
gepusht hat, zeigt die Website die neue Woche, spätestens fünf Minuten später; so lange
hält GitHub die Dateien zwischengespeichert. Ist GitHub nicht erreichbar, fällt die
Website auf den Ordner `gewuerze/data/` auf dem Server zurück und weist darauf hin.

Damit das funktioniert, **muss dieses Repo öffentlich bleiben.**

## Enthaltene Dateien

```
.claude/agents/research-agent.md      Subagent: Recherche (4x parallel, je ein Bereich)
.claude/agents/synthesis-agent.md     Subagent: Auswahl Top 3 + Signale + Dedupe
.claude/agents/email-agent.md         Subagent: HTML-Email bauen (nur Inline-Styles)
.claude/agents/verification-agent.md  Subagent: Qualitätsprüfung vor Versand
config/topic.json                     Kategorien und Einordnungen des Themas
config/watchlist.json                 Wer und was jede Woche abgeklopft wird
scripts/publish_week.mjs              Schreibt die Website-Daten aus der Synthese
ROUTINE_PROMPT.md                     Der Prompt fürs "Instructions"-Feld der Routine
state/history.json                    Verlauf behandelter Themen (Dedupe)
state/last_run.json                   Schutz vor Doppel-Versand + Retry-Erkennung
email/output/<KW>.html                Die verschickte Email-Fassung je Woche
data/weeks/<KW>.json                  Eine Ausgabe im Datenformat der Website
data/index.json                       Verzeichnis aller Wochen, liest die Website zuerst
tmp/                                  Zwischenstände eines Laufs, nicht im Repo
```

## Konfiguration

### Umgebungsvariablen (in der Cloud Environment)

| Variable | Wert | Zweck |
|---|---|---|
| `EMAIL_TO` | Empfängeradresse | Empfänger der Wochen-Email |
| `SITE_URL` | `https://anton-blochwitz.de/gewuerze/` | Die Email verlinkt auf `<SITE_URL>#/woche/<KW>` |

`SITE_URL` endet mit `/`. Ist sie nicht gesetzt, entfallen die Website-Links in der
Email ersatzlos, der Rest läuft normal weiter.

> **Eigene Environment nötig.** Der KI-Radar hat seine eigene `SITE_URL`. Beide
> Routinen können sich deshalb keine Cloud Environment teilen.

### Connectors

Der **Gmail-Connector** muss verbunden und für die Routine aktiviert sein. Ohne ihn
bricht der Lauf in Schritt 6 ab und markiert `state/last_run.json` als `failed`, damit
ein späterer Lauf es erneut versucht, ohne neu zu recherchieren.

### Routine

Auf [claude.ai/code/routines](https://claude.ai/code/routines):

- **Prompt**: kompletter Inhalt aus `ROUTINE_PROMPT.md` (zwischen den Markierungen)
- **Repository**: dieses Repo
- **Trigger**: Schedule, wöchentlich mittwochs (Zeitzone Europe/Berlin)

> **Wichtig:** Wird der Prompt in den Routine-Einstellungen geändert, muss
> `ROUTINE_PROMPT.md` mitgezogen werden, und umgekehrt.

## Das Datenformat

`scripts/publish_week.mjs` erzeugt es aus der Ausgabe des synthesis-agent und prüft
dabei alle Pflichtfelder. Ist etwas ungültig, schreibt das Skript **nichts** und nennt
die fehlerhaften Felder. Welche Kategorien und Einordnungen gültig sind, steht in
`config/topic.json`; dadurch ist das Skript dasselbe wie im KI-Repo.

Von Hand, etwa um eine Woche zu reparieren:

```bash
node scripts/publish_week.mjs --week 2026-W42 --date 2026-10-14 --input tmp/synthesis.json
node scripts/publish_week.mjs --index-only
```

Die zweite Zeile baut nur das Verzeichnis neu, zum Beispiel nachdem eine Woche gelöscht
wurde.

## Zustand verstehen (`state/`)

### `state/last_run.json`

Steuert Doppelversand-Sperre und Retry:

- `last_sent_iso_week` == aktuelle Woche **und** `last_status: "success"`
  → der Lauf bricht sofort ab und verschickt nichts.
- `last_attempted_iso_week` == aktuelle Woche **und** `last_status: "failed"`
  **und** `email/output/<KW>.html` existiert
  → nur der Versand wird wiederholt, ohne neue Recherche.

> **Einen Extra-Versand erzwingen:** `last_sent_iso_week` auf `null` setzen (oder auf
> eine ältere Woche) und committen.

### `state/history.json`

Steuert die Deduplizierung:

- `dedupe_window_weeks` (Standard `8`): wie viele Wochen zurück ein Thema als
  schon behandelt gilt
- `max_history_entries` (Standard `30`): ältere Einträge werden entfernt; die Dateien
  unter `data/weeks/` bleiben als Archiv der Website erhalten
- **Pro Kalenderwoche genau ein Eintrag.**

Eine echte Fortsetzung darf erneut auftauchen: ein Gesetz, das vom Entwurf in Kraft
tritt, eine Übernahme, die freigegeben wird, ein Preis, der eine neue Schwelle reißt.
Dieselbe Nachricht aus einer zweiten Publikation nicht.

## Was sich leicht ändern lässt

- **Wettbewerber und Dauerthemen**: `config/watchlist.json`
- **Bereiche und Einordnungen**: `config/topic.json` — die Website liest ihre Sektoren
  aus ihrer eigenen Themendatei, beides muss zusammenpassen
- **Dedupe-Fenster**: `dedupe_window_weeks` in `state/history.json`
- **Quellenbasis der Recherche**: `.claude/agents/research-agent.md`
- **Tag und Uhrzeit**: direkt in der Routine unter Repeats
- **Ton und Aufbau der Email**: `.claude/agents/email-agent.md`

## Bezahlschranken

Die wichtigste Fachpresse dieser Branche ist überwiegend kostenpflichtig. Deshalb gilt
durchgängig: Was nur als Anriss lesbar war, wird **nicht** ausgeschmückt. Stattdessen
sucht der Agent die zugrunde liegende Pressemitteilung oder lässt das Thema fallen. Ein
dünnerer, aber belegter Wochenüberblick ist mehr wert als ein voller, der rät.
