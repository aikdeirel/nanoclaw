---
name: news-summary
description: >
  Zusammenfassen von Nachrichtenartikeln, die der Nutzer teilt. Immer verwenden wenn der Nutzer
  eine Artikel-URL schickt, nach einem "TL;DR" fragt, einen Artikel "zusammenfassen" oder
  "erklären" möchte, oder Phrasen wie "was steht da drin", "fass das zusammen", "summarize this",
  "what's this about", "tldr" oder ähnliches verwendet. Auch triggern wenn der Nutzer einen
  Link teilt und eine kurze Einschätzung oder Kontext will, auch ohne explizite Bitte ums
  Zusammenfassen. Nicht verwenden für allgemeine Fragen zu Themen ohne konkreten Artikel.
---

# News Summary Skill

## Purpose
Liefere strukturierte Artikel-Zusammenfassungen in ~30 Sekunden Lesezeit. Kernpunkte,
Kontext, Bias-Check, offene Fragen. Output immer auf Deutsch, unabhängig von der
Sprache des Artikels.

## Workflow

### Step 1: URL klassifizieren & fetchen

Vor dem Fetch: URL-Typ bestimmen.

| URL-Typ | Vorgehen |
|---|---|
| Normaler Artikel | `webFetch` direkt |
| Paywall-Verdacht (NYT, FAZ, Spiegel+, FT, etc.) | Fetch versuchen, Partial-Content nutzen wenn vorhanden |
| Twitter/X, Threads, Mastodon | Nicht unterstützt → Nutzer informieren |
| YouTube, Podcast | Nicht unterstützt → Nutzer informieren |
| PDF | Nicht unterstützt → Nutzer informieren |
| Reddit | Fetch versuchen, oft öffentlich zugänglich |

**Bei Paywall:** Wenn Fetch nur Teaser/Intro liefert, klar markieren: "⚠️ Nur Teilzugriff — Zusammenfassung basiert auf verfügbarem Content." Keine Inhalte erfinden.

**Bei Fetch-Fehler:** Nutzer bitten, Artikeltext direkt einzufügen oder andere URL zu versuchen.

### Step 2: Artikel analysieren

- Publikationsdatum und Quelle identifizieren
- Artikeltyp bestimmen
- Bias/Perspektive prüfen
- Was fehlt oder wird nicht erwähnt?
- Einordnung in größere laufende Entwicklungen prüfen
- Sprache des Artikels notieren (Output bleibt immer Deutsch)

### Step 3: Summary strukturieren

#### 🎯 TL;DR
1-2 Sätze, Kernaussage.

#### 📰 Quick Facts
- **Quelle:** [Name + kurze Einschätzung der Publikation]
- **Datum:** [heute / gestern / konkretes Datum]
- **Typ:** [🔥 Breaking / 💡 Analyse / 📊 Daten / 💬 Meinung / 🔍 Investigation]
- **Warum jetzt relevant:** Ein Satz.

#### 📋 Kernpunkte
3-5 Bullets, wichtigstes zuerst. Je 1-2 Sätze. Überraschende Wendungen oder unerwartete
Details explizit hervorheben.

#### 🔄 Größerer Kontext
*(Weglassen wenn Standalone-Artikel ohne laufende Story)*
2-3 Sätze: Einordnung in Gesamtentwicklung.

#### ⚠️ Perspektive & Lücken
- **Bias:** Faktisch/ausgewogen oder meinungslastig? Welche Perspektive dominiert?
- **Nicht erwähnt:** Wichtige Gegenargumente oder Aspekte die fehlen.
2-3 Sätze insgesamt.

#### ❓ Offene Fragen
1-2 Fragen die der Artikel aufwirft aber nicht beantwortet.

---

#### 📚 Kontext & Erklärungen
*(Nur bei komplexen Fachbegriffen, unbekannten Akteuren oder Hintergrundwissen das zum
Verständnis nötig ist. Weglassen wenn der Artikel selbsterklärend ist.)*

- **[Begriff/Akteur]:** Kurze Erklärung (2-3 Sätze)
- **Quellen:** Link für Vertiefung

## Stilregeln

- Gesamte Summary in ~30 Sekunden lesbar
- Neutraler Ton in der Summary selbst
- Externe Einschätzungen klar als solche markieren, nie als Artikelinhalt ausgeben
- Output immer Deutsch, auch wenn Artikel auf Englisch, Französisch etc.
- Keine unnötigen Füllsätze

## Nicht unterstützte Formate

Bei folgenden Inhalten klar kommunizieren dass der Skill nicht anwendbar ist:
- Social Media Posts (Twitter/X, Instagram, TikTok)
- Videos und Podcasts (YouTube, Spotify etc.)
- PDFs ohne Text-Layer
- Artikel hinter Login-Walls ohne Partial-Content

Alternativen anbieten: "Füge den Text direkt ein, dann kann ich zusammenfassen."
