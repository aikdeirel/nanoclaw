---
name: software-research
description: >
  Software, Tools oder Libraries recherchieren, analysieren und bewerten. Immer verwenden wenn
  der Nutzer eine URL (GitHub, Produktseite, Docs) teilt und fragt "wie funktioniert das",
  "was ist das", "ist das sinnvoll", "lohnt sich das", "was sind Alternativen", oder ähnliches.
  Auch triggern wenn Kai einen Toolnamen nennt und eine ehrliche Einschätzung will, oder wenn
  ein Vergleich zwischen mehreren Tools angefragt wird. Auch bei Fragen wie "haben die ein
  Pricing", "wer steckt dahinter", "ist das production-ready", "was speichern die". Kurz:
  immer wenn es um die Evaluation von Software geht, egal ob mit URL oder ohne.
---

# Software Research Skill

## Purpose

Liefere eine ehrliche, praxisorientierte Analyse von Software-Produkten. Kein Marketing-Sprech,
keine diplomatische Weichspülung. Kai ist Senior Engineer mit über 10 Jahren Erfahrung - er
braucht Substanz, keine Oberfläche.

Drei Szenarien:
1. **Freie Exploration** - Nutzer teilt URL/Name, keine konkreten Fragen → vollständige Analyse
2. **Gezielte Fragen** - Nutzer hat spezifische Fragen → diese beantworten + relevante Lücken proaktiv füllen
3. **Vergleich** - Mehrere Tools gegenüberstellen → strukturierter Head-to-Head

---

## Workflow

### Step 1: Quellen fetchen

**Pflicht-Quellen:**
- GitHub README (bei Open Source) - direkt fetchen, nicht raten
- Offizielle Produktseite oder Docs
- Pricing-Seite falls nicht im README

**Bei Bedarf:**
- Externe Abhängigkeiten eigenständig recherchieren wenn sie zentral für das Produkt sind
- Alternativen suchen (etablierte und rising) - immer relevant, nie weglassen
- News/recent commits/last activity prüfen bei Reifegrad-Fragen

Nicht nach Permission fragen - einfach fetchen.

### Step 2: Analyse

Bevor geantwortet wird, intern klären:

**Wer steckt dahinter?**
- Firma, Forschungsgruppe, Einzelperson?
- Funding/Backing vorhanden?
- Wie lange gibt es das Projekt? Wie viele Commits, Stars, Contributors?
- Geografische Einordnung wo relevant (Data Sovereignty etc.)

**Was passiert wirklich? (Top-Down)**
Nicht die Feature-Liste aufzählen, sondern den Ablauf erklären: Was passiert wenn ein User
das Tool benutzt, von Anfang bis Ende? Welche Komponenten sind involviert, in welcher
Reihenfolge, mit welchen Entscheidungen dazwischen? Externe Services als vollwertige
Bestandteile des Flows behandeln, nicht als Fußnote.

**Datenflüsse (immer detailliert)**
Für jede Kategorie die zutrifft klären:
- Welche Daten verlassen die eigene Infrastruktur, wohin, wann?
- Was wird persistiert, wie lange, wo?
- Was läuft lokal vs. remote?
- Welche externen Services bekommen was zu sehen?
Das ist kein optionaler Abschnitt - Datenflüsse haben bei jeder Software Relevanz.

**Kosten**
- Pricing-Modell vollständig verstehen: Pro Request? Pro Token? Subscription? Open Core?
- Hidden costs: Wenn externer Service Pflicht ist, dessen Kosten miteinrechnen
- Konkrete Beispielkosten wenn möglich

**Maturity / Production-Readiness**
- Research-Paper-Demo vs. Production-Tool klar unterscheiden
- Test coverage, Issues, letzte Commits
- Welche Firmen nutzen es already in Production?

**Abhängigkeiten & Lock-in**
- Wenn Tool X Pflicht-Dependency auf Service Y hat, ist das ein zentraler Fakt
- Was passiert wenn der externe Service wegfällt, teurer wird, oder die API bricht?

### Step 3: Output strukturieren

#### Wenn gezielte Fragen gestellt wurden:
Fragen direkt und vollständig beantworten, dann "Was du nicht gefragt hast aber wissen solltest:" ergänzen wenn relevant.

#### Wenn freie Exploration:

**Was ist es wirklich** *(1-2 Sätze, technisch konkret, kein Marketing)*

**Wie es funktioniert** *(Top-Down: was passiert von User-Aktion bis Ergebnis, inkl. externer Services)*

**Datenflüsse** *(was verlässt die eigene Infra, wohin, was wird gespeichert - immer vollständig)*

**Kosten** *(vollständig, inkl. versteckte Kosten durch Dependencies)*

**Reifegrad** *(ehrliche Einschätzung: Research-Demo / Alpha / Stabil / Production-proven)*

**Das nicht Offensichtliche** *(Haken, Widersprüche, was die README verschweigt)*

**Alternativen** *(etablierte UND rising, mit klarer Einschätzung wann welche besser ist)*

**Urteil** *(direkte Empfehlung für Kais konkreten Use Case falls erkennbar, sonst generell)*

---

## Stilregeln

- Keine Weichspülung. "Das ist ein 4-Star Research-Projekt" ist korrekt wenn es so ist.
- Externe Dependencies immer vollständig durchdenken - nicht nur das Hauptprodukt
- Wenn etwas unklar ist: nachforschen, nicht raten
- Konkret über abstrakt: Zahlen, Beispielkosten, konkrete Szenarien statt vage Beschreibungen
- Kein "Das kommt drauf an" ohne direkt danach zu sagen worauf es ankommt

## Was NICHT passiert

- Nicht die README paraphrasieren und das Analyse nennen
- Nicht bei jeder Aussage "laut README..." - einmal fetchen, dann eigenständig urteilen
- Nicht alle möglichen Fragen stellen bevor gefetcht wird - erst researchen, dann antworten
- Datenflüsse nicht in einem Satz abhandeln oder unter einem anderen Punkt verstecken
