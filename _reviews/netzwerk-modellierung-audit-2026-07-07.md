# Audit: Netzwerk-Modellierung — Wachstum, Churn, Vergütungsplan

**Datum:** 2026-07-07
**Scope:** Simulationskern und LifePlus-Vergütung, gelesen im Ist-Stand `main`:

- [packages/simulator-core/src/tree-generator.ts](packages/simulator-core/src/tree-generator.ts) — Wachstum, Carry, Churn, Reattachment, Shopper-Counts
- [packages/simulator-core/src/simulation.ts](packages/simulator-core/src/simulation.ts) — `runSimulation`, Jahres-/Quartals-Aggregation
- [packages/simulator-core/src/person-tree.ts](packages/simulator-core/src/person-tree.ts) — `personTreeToNetworkSnapshot`, Level-/Bein-Ableitung
- [packages/product-lifeplus/src/tree-compensation.ts](packages/product-lifeplus/src/tree-compensation.ts) — Phasen-Provision, Rang-States, QGV
- [packages/product-lifeplus/src/ranks.ts](packages/product-lifeplus/src/ranks.ts), [constants.ts](packages/product-lifeplus/src/constants.ts), [payout-slots.ts](packages/product-lifeplus/src/payout-slots.ts) — Rang-/Slot-Regeln
- [packages/product-lifeplus/src/plan.ts](packages/product-lifeplus/src/plan.ts) — Churn-Kandidaten
- [simulator-app/src/App.tsx](simulator-app/src/App.tsx) — Parameter-Verdrahtung

**Regel-Referenz (Soll):** [_doc/growth_models/02-Wachstums-und-Churn-Regeln.md](_doc/growth_models/02-Wachstums-und-Churn-Regeln.md), [01-Zielarchitektur.md](_doc/growth_models/01-Zielarchitektur.md).

**Nicht reviewt:** UI-Visualisierung (Sunburst/RadialTree), Reality-Strategien-Interna (`simulator-realistic-growth`), Persistenz-Migration, `example-line.ts` (didaktischer Nebenpfad). Siehe [Nicht reviewt](#nicht-reviewt).

---

## Zusammenfassung

- **Regel-Konformität ist überwiegend gegeben.** Year-Offset, selektiver Rang-Churn, Shopper-als-Float, Phase-1-Kompression auf qualifizierte Upline und Reattachment sind so implementiert wie in `02-Wachstums-und-Churn-Regeln.md` beschrieben.
- **Eine dokumentierte Produktregel fehlt komplett im Code:** die Mindest-Churn-Regel (§3.3). Der Attrition-Slider fließt ungefiltert durch — genau das Szenario „aggressives Wachstum ohne Fluktuation" ist damit ungebremst möglich (**F-01, Hoch**).
- **Compressed-Mode verletzt eine Modell-Invariante.** Ab 5.000 expliziten Personen zählt der Code Beine über `weight`; das inflationiert `bronzeLegs`/`diamondLegs` und damit Rang und Phase-3-Provision in genau den großen Szenarien, die als Verkaufsargument dienen (**F-02, Hoch**).
- **Zwei stille Volumen-Anhebungen** verzerren die Provision nach oben: die effektive AV wird auf die Rang-Mindestanforderung hochgezogen (**F-03**), und Vergütungsraten haben zwei nicht-synchronisierte Wahrheitsquellen (**F-04**).
- **Wachstumsmechanik selbst ist sauber und strategie-neutral:** Die Reality-Strategie verteilt *wo* neue Member andocken, nicht *wie viele* — die Gesamtzahl hängt nur an `membersPerYear × duplicationRate × aktive Member`.

---

## Teil A — Konformität mit dem Vergütungsplan

Gegenüberstellung Soll (`02`/`constants.ts`) ↔ Ist (Code). Grün = konform, ⚠ = Abweichung/Finding.

| Regel | Soll | Ist im Code | Status |
|---|---|---|---|
| Year-Offset | Neue Member werben erst im Folgejahr | `sourceMembers`-Filter `joinedMonth < monthIndex`, [tree-generator.ts:192](packages/simulator-core/src/tree-generator.ts#L192) | ✓ |
| Attrition vor Wachstum | Churn zuerst, dann neue Member | Reihenfolge in `advanceYear`, [tree-generator.ts:181-197](packages/simulator-core/src/tree-generator.ts#L181-L197) | ✓ |
| Selektiver Rang-Churn | nur Member/Believer/Builder/Bronze | `churnableRanks`-Set, [plan.ts:16](packages/product-lifeplus/src/plan.ts#L16) | ✓ |
| Root churnt nie, rekrutiert weiter | §3.2 | Root nur als Sponsor in `applyMemberAttrition`, nie Opfer; recruit in `advanceYear` | ✓ |
| Shopper = Float, QGV-relevant, nicht AV | §2.1 | `shopperCount` als Zahl; `ownShopperQgv` in QGV, nicht in AV, [tree-compensation.ts:280-283](packages/product-lifeplus/src/tree-compensation.ts#L280-L283) | ✓ |
| Phase-1-Sätze 25/10/5 etc. | `PHASE1` | identisch, [constants.ts:3-7](packages/product-lifeplus/src/constants.ts#L3-L7) | ✓ |
| Phase-1-Kompression auf qualifizierte Upline | AV≥40, L3 QL≥3 | `findPhase1Receiver`, [tree-compensation.ts:418](packages/product-lifeplus/src/tree-compensation.ts#L418) | ✓ |
| Phase 2/3 ab Ebene 4 | Tiefenbonus | `upline.slice(3)`, [tree-compensation.ts:168](packages/product-lifeplus/src/tree-compensation.ts#L168) | ✓ |
| Rang-Treppe (Bronze→Diamond, n*Diamond) | `PHASE2/3_RANKS` | `determineRank`, [ranks.ts:32](packages/product-lifeplus/src/ranks.ts#L32) | ✓ |
| **Mindest-Churn nach `membersPerYear`** | §3.3 (10/18/25/30 %) | **nicht vorhanden** | ⚠ **F-01** |
| **Kompression verfälscht Rang/QGV/Provision nicht** | §8 Invariante | weight-basierte Bein-Zählung | ⚠ **F-02** |
| Effektive AV | „AV, die der Plan verlangt" | wird auf Rang-Minimum **angehoben**, verändert Bestellvolumen | ⚠ **F-03** |

**Kernbefund:** Die *Struktur* des Plans ist korrekt abgebildet. Die Abweichungen liegen in (a) einer fehlenden Dämpfungsregel und (b) Stellen, an denen aggregierte Performance-Kompression die exakte Fachlogik überschreibt.

---

## Teil B — Wie die Parameter das Wachstum steuern

Fünf Eingaben aus [App.tsx:199-207](simulator-app/src/App.tsx#L199-L207) fließen in `SimulatorInputs`. Ihr Zusammenspiel im Jahres-Schritt `advanceYear`:

```text
                       Jahr N  (advanceYear)
   ┌───────────────────────────────────────────────────────────┐
   │ 1. Churn (zuerst)                                          │
   │    Shopper:  count -= count × attritionRate                │
   │    Member:   floor(aktiveKinder × attritionRate + carry)   │
   │              nur Ränge {Member,Believer,Builder,Bronze}    │
   │                        │                                   │
   │ 2. sourceMembers = aktive Member mit joinedMonth < N       │  ← Year-Offset
   │                        │                                   │
   │ 3. Root:   + membersPerYear           (ohne duplication)   │
   │            + shoppersPerYear                               │
   │                        │                                   │
   │ 4. je Source-Member:                                       │
   │    + weight × membersPerYear × duplicationRate × wᵢ        │
   │    + weight × shoppersPerYear × duplicationRate × wᵢ       │
   │            gedeckelt auf  weight × maxDirect − aktiveKinder │
   └───────────────────────────────────────────────────────────┘
```

**Effektive Wachstumsrate.** Weil `sourceGrowthWeights` die Strategie-Gewichte so normiert, dass ihr gewichtetes Mittel = 1 ist ([tree-generator.ts:510-523](packages/simulator-core/src/tree-generator.ts#L510-L523)), gilt für die *Gesamtzahl* neuer Downline-Member pro Jahr:

```text
neue Downline-Member(N) ≈ membersPerYear × duplicationRate × Σ aktive Source-Member-weights
```

Daraus folgt für die Parameter:

| Parameter | Wirkung | Besonderheit |
|---|---|---|
| `membersPerYear` | linearer Multiplikator auf Rekrutierung; Root **und** Downline | Slider 0–36, Default 2 |
| `duplicationRate` | skaliert **nur die Downline** — Member *und* deren Shopper | Default `0` → Downline ist inert, nur Root wächst |
| `shoppersPerYear` | additiver Volumen-Zufluss; Root flat, Downline × dup | Volumen, keine Struktur |
| `attritionRate` | dämpft Member (selektiv) und Shopper (voll) | **kein Mindestwert** (F-01) |
| `maxDirectMembersPerMember` | deckelt Rekrutierung pro Sponsor (Default 29) | Überschuss wandert in `memberCarry`, geht nicht verloren |

Das effektive System ist ein gedeckeltes Exponentialwachstum mit Basis `≈ 1 + membersPerYear × duplicationRate × (1 − churn)`, um ein Jahr verzögert. **Wichtig für Erwartungsmanagement:** Bei Default `duplicationRate = 0` sieht der Nutzer nur lineares Root-Wachstum (max. 29 direkte Beine). Erst `duplicationRate > 0` startet die Kettenreaktion.

> **Strategie-Neutralität der Menge:** `person-tree-equal/-random/-momentum` ändern nur die *Verteilung* neuer Member auf bestehende Beine, nicht deren Anzahl. Momentum kann dadurch andere Endränge erzeugen (Konzentration in starken Beinen), aber nicht mehr Personen. Das ist konsistent mit `01-Zielarchitektur §5`.

---

## Teil C — Regeln für Hinzufügen und Abziehen von Members & Shoppern

```text
        MEMBER (diskrete Person)              SHOPPER (Float am Sponsor)
   ┌──────────────────────────────┐     ┌──────────────────────────────┐
ADD│ points = weight × mpy × dup×wᵢ │     │ count += weight × spy × dup×wᵢ │
   │ total  = points + memberCarry  │     │ (Root: += shoppersPerYear)    │
   │ whole  = floor(total)  ↑cap    │     │ Dezimalwerte erlaubt          │
   │ Rest → memberCarry             │     │ kein Carry                    │
   │ → ganze SimPerson (weight 1)   │     │ → nur Zahl, keine Person      │
   └──────────────────────────────┘     └──────────────────────────────┘
   ┌──────────────────────────────┐     ┌──────────────────────────────┐
RM │ pool = aktiveKinder×rate+carry │     │ lost = count × attritionRate  │
   │ whole = floor(pool)            │     │ count -= lost                 │
   │ Auswahl: neueste zuerst        │     │ (voller Float-Abzug)          │
   │ weight−1; bei 0 → inaktiv      │     │ kein Carry, keine Ganzzahl    │
   │ + Reattachment der Kinder      │     │ churnt bei jedem Sponsor      │
   └──────────────────────────────┘     └──────────────────────────────┘
```

**Member — Hinzufügen** ([tree-generator.ts:211-251](packages/simulator-core/src/tree-generator.ts#L211-L251), [`takeCarryGrowth`](packages/simulator-core/src/tree-generator.ts#L261)):
- Fraktionale Rekrutierung wird über `memberCarry` deterministisch akkumuliert (F1a-Carry). Aus zwei 0,5-Anteilen wird im Folgejahr ein ganzer Member.
- Deckel: `weight × maxDirect − aktiveKinder`. Überschuss bleibt als Carry liegen.
- Ab > 5.000 expliziten Personen entsteht statt Einzelpersonen ein **gewichteter Aggregat-Knoten** (`weight = count`) — bewusste Lockerung der „ganze Personen"-Garantie ([tree-generator.ts:302-323](packages/simulator-core/src/tree-generator.ts#L302-L323)).

**Member — Abziehen** ([`applyMemberAttrition`](packages/simulator-core/src/tree-generator.ts#L353)):
- Pro Sponsor: `floor(aktiveKinder × rate + memberAttritionCarry)`, Rest als Carry.
- Nur churnfähige Ränge (Vorjahresstatus). Auswahl deterministisch „neueste zuerst".
- Beim vollständigen Churn: `active=false`, aktive Kinder + `shopperCount` wandern zum Sponsor (**Reattachment**, [tree-generator.ts:462](packages/simulator-core/src/tree-generator.ts#L462)).

**Shopper — Hinzufügen/Abziehen** ([`addShopperGrowth`](packages/simulator-core/src/tree-generator.ts#L346), [`applyShopperAttrition`](packages/simulator-core/src/tree-generator.ts#L438)):
- Reiner Float. Wachstum und Churn ohne Carry, ohne Ganzzahl-Rundung. Churn trifft **jeden** Sponsor mit `shopperCount > 0`.

Alle drei Fragen des Auftrags sind damit beantwortet; die konkreten Fehler und Verbesserungen stehen unten.

---

## Findings

| Stufe | Bedeutung |
|---|---|
| **Kritisch** | Datenverlust, Sicherheitslücke, Produktionsausfall |
| **Hoch** | spürbare Auswirkung auf Ergebnis/Zahlen, klare technische Schuld |
| **Mittel** | Wartbarkeit/Korrektheit beeinträchtigt, kein akutes Problem |
| **Niedrig** | Stil, Konvention, Optimierung ohne Drängen |

### F-01 [Hoch] — Mindest-Churn-Regel (§3.3) ist nicht implementiert

**Datei:** [simulator-app/src/App.tsx:202](simulator-app/src/App.tsx#L202) → [tree-generator.ts:183-189](packages/simulator-core/src/tree-generator.ts#L183-L189)

**Beobachtung:**
`02-Wachstums-und-Churn-Regeln.md §3.3` definiert eine Mindest-Fluktuation abhängig von `membersPerYear` (≥2,0 → 10 %; ≥2,5 → 18 %; ≥3,0 → 25 %; ≥3,5 → 30 %). Im Code existiert dazu **nichts**. `attritionRate` wird in `App.tsx` als `attrition / 100` gesetzt und unverändert an `applyMemberAttrition`/`applyShopperAttrition` durchgereicht. Eine repo-weite Suche nach den Schwellenwerten findet nur einen gleichnamigen Opacity-Wert in einer Visualisierung.

**Auswirkung:**
Der Nutzer kann `membersPerYear = 4,5` mit `attritionRate = 0` kombinieren. Das ist exakt das in `01-Zielarchitektur §8` genannte „~3,6 Mio Netzwerk-Größe in Jahr 10"-Szenario — die Regel sollte genau das dämpfen. Zusätzlich verletzt es die Modell-Invariante „aggressive Wachstumsannahmen dürfen nicht zu unplausiblen Zahlen führen".

**Vorschlag:**
Eine reine Ableitung vor dem Simulationslauf, z. B. in `App.tsx` beim Bauen der `inputs` oder als `effectiveAttrition` im Product-Plan:

```ts
function minChurnFor(membersPerYear: number): number {
  if (membersPerYear >= 3.5) return 0.30;
  if (membersPerYear >= 3.0) return 0.25;
  if (membersPerYear >= 2.5) return 0.18;
  if (membersPerYear >= 2.0) return 0.10;
  return 0;
}
const attritionRate = Math.max(attrition / 100, minChurnFor(membersPerYear));
```

Falls die Regel bewusst verschoben ist (P3-Stresstest bricht sie ja absichtlich), sollte das im UI sichtbar sein — sonst wirkt der Slider bei niedrigen Werten „zu optimistisch".

**Aufwand:** S · **Risiko:** niedrig · **Offene Frage:** Soll der Floor hart greifen oder nur eine UI-Warnung sein?

---

### F-02 [Hoch] — Compressed-Mode inflationiert Bein- und Rangzählung

**Datei:** [tree-compensation.ts:284-295](packages/product-lifeplus/src/tree-compensation.ts#L284-L295)

**Beobachtung:**
`qualifiedLegs`, `bronzeLegs` und `diamondLegs` summieren `child.weight`:

```ts
const bronzeLegs = directMemberChildren.reduce((total, child, index) => {
  const childComputation = childComputations[index];
  return childComputation?.containsBronze ? total + child.weight : total;
}, 0);
```

Im Exact-Mode ist `weight = 1`, das ist korrekt. Im Compressed-Mode (> 5.000 Personen) hat ein Aggregat-Knoten `weight = count` (z. B. 50.000). Wird dieser eine Knoten als „bronze-haltig" erkannt, zählt er als **50.000 Bronze-Beine** — obwohl er strukturell *ein* aggregierter Zweig mit *einer* gemeinsamen Downline ist, nicht 50.000 unabhängige Beine, die alle zufällig Bronze erreichen.

**Auswirkung:**
`determineRank` gate-t Phase-3-Ränge über `diamondLegs`/`bronzeLegs` ([ranks.ts:53-82](packages/product-lifeplus/src/ranks.ts#L53-L82)). Inflationierte Bein-Zahlen lassen den Root in großen Szenarien trivial `n*Diamond` erreichen → **überhöhte Phase-3-Provision** genau in den Headline-Szenarien. Das verletzt die in `02 §8` geforderte Invariante „Kompression darf Rank/QGV/Provision nicht verfälschen", für die laut Doku Tests existieren *sollen*.

**Vorschlag:**
Bein-Qualität (`containsBronze/Diamond`) ist eine strukturelle Ja/Nein-Eigenschaft eines Beins und sollte **nicht** mit `weight` multipliziert werden. Kandidat: qualifizierende Beine als **Anzahl echter Beinlinien** zählen (Aggregat = 1 Bein-Linie, unabhängig von `weight`), während `qualifiedLegs` (reine Aktivitäts-Schwelle, max. 12) weiter über `weight` laufen darf. Vor der Änderung eine Referenz-Regressionsprobe (Exact vs. Compressed bei identischer Endgröße) als Test einfrieren.

**Aufwand:** M · **Risiko der Änderung:** mittel (berührt Rangbestimmung; Tests nötig)

---

### F-03 [Mittel] — Effektive AV hebt Bestellvolumen und QGV still an

**Datei:** [tree-compensation.ts:84-92](packages/product-lifeplus/src/tree-compensation.ts#L84-L92), [ranks.ts:93-136](packages/product-lifeplus/src/ranks.ts#L93-L136)

**Beobachtung:**
`determineEffectiveAV` gibt `max(av, requiredAV)` zurück — die AV wird auf das Minimum des höchsten strukturell erreichten Rangs **angehoben**. Anschließend ersetzt `calculateOrderPayouts` das Bestellvolumen durch diese effektive AV:

```ts
const effectiveOrder =
  order.kind === 'member_order'
    ? { ...order, volume: rankByPersonId.get(order.personId)?.av ?? order.volume }
    : order;
```

Bei Default `memberMonthlyVolume = 45` und einem Member, dessen Struktur Bronze (minAV 100) oder Gold/Diamond (minAV 150) erfüllt, springt das Bestellvolumen von 45 auf 100 bzw. 150 IP. Über `ownVolume = av × weight` ([tree-compensation.ts:314](packages/product-lifeplus/src/tree-compensation.ts#L314)) fließt diese Anhebung zusätzlich in das QGV der **gesamten Upline**.

**Auswirkung:**
Provisionsbasis und QGV werden gegenüber dem tatsächlich eingegebenen Verbrauch nach oben verzerrt. Das ist als Modellannahme („wer den Rang hält, muss entsprechend konsumieren") vertretbar, aber **nicht sichtbar** und selbstverstärkend (höhere AV → höheres QGV → höherer Rang → höhere AV). Bei Default-Volumen 45 greift der Effekt regelmäßig.

**Vorschlag:**
Entscheidung explizit machen und dokumentieren: entweder (a) effektive AV nur für die *Rang-Prüfung* verwenden, aber die *Provisionsbasis* auf dem echten `personalMonthlyVolume` lassen, oder (b) die Annahme im UI/Tooltip offenlegen („Rang-Halten setzt Mindest-AV voraus"). Als Annahme markieren, nicht als stille Rechenregel.

**Aufwand:** S–M · **Risiko:** mittel (verändert Zahlen; bewusst zu entscheiden)

---

### F-04 [Mittel] — Zwei nicht-synchronisierte Wahrheitsquellen für Vergütungsraten

**Datei:** [constants.ts:26-64](packages/product-lifeplus/src/constants.ts#L26-L64) vs. [payout-slots.ts:9-20](packages/product-lifeplus/src/payout-slots.ts#L9-L20)

**Beobachtung:**
Die Phase-2/3-Raten existieren doppelt: als `rate`/`incrementalRate`/`cumulativeRate` in `PHASE2_RANKS`/`PHASE3_RANKS` **und** als Slot-Raten in `PHASE2_SLOTS`/`PHASE3_SLOTS`. Die tatsächliche Auszahlung läuft ausschließlich über die Slot-Allokation ([tree-compensation.ts:169-223](packages/product-lifeplus/src/tree-compensation.ts#L169-L223)); `determineRank.phase2Rate`/`phase3Rate` werden für Payouts **nicht** verwendet. Beide Darstellungen stimmen heute überein (Diamond = 4 × 0,03 = 0,12), aber nur per Konstruktion.

**Auswirkung:**
Ändert jemand eine Rate nur an einer Stelle, driften Rang-Anzeige und Auszahlung auseinander, ohne dass ein Test zwangsläufig anschlägt.

**Vorschlag:**
Eine Quelle ableiten aus der anderen (z. B. `PHASE2_RANKS[i].rate == Σ der ersten i Slot-Raten` als Invariantentest), oder die ungenutzten `rate`-Felder entfernen und im Code kommentieren, dass Slots führen.

**Aufwand:** S · **Risiko:** niedrig

---

### F-05 [Niedrig] — `memberCarry` wächst bei dauerhaft gecapptem Sponsor unbegrenzt

**Datei:** [tree-generator.ts:261-274](packages/simulator-core/src/tree-generator.ts#L261-L274)

**Beobachtung:**
Ist ein Sponsor dauerhaft am `maxDirect`-Cap, ist `capacity = 0`, `newWhole = 0`, und der volle `points`-Betrag wandert jedes Jahr in `nextCarry`. Der Carry akkumuliert monoton, ohne je zu materialisieren.

**Auswirkung:**
Numerisch heute harmlos (Materialisierung bleibt auf `capacity` begrenzt). Aber wenn durch Churn *einmal* Kapazität frei wird, kann ein unrealistischer Nachhol-Burst entstehen, und der Carry-Wert selbst wird bedeutungslos groß.

**Vorschlag:**
Carry beim Cap deckeln (z. B. auf `membersPerYear`) oder nicht weiter akkumulieren, sobald `capacity == 0`.

**Aufwand:** S · **Risiko:** niedrig

---

### F-06 [Niedrig] — Direct-Cap-Default an vier Stellen dupliziert

**Datei:** [tree-generator.ts:5](packages/simulator-core/src/tree-generator.ts#L5) u. a.

**Beobachtung:**
`29` als Default für `maxDirectMembersPerMember` lebt laut `02 §1.4` an vier Stellen (tree-generator, simulation, App.tsx, product-`index.ts`). Das ist bereits als Konsolidierungs-TODO (18 Z8) dokumentiert, aber offen.

**Vorschlag:** Eine exportierte Konstante in `simulator-core`, alle Caller referenzieren sie.

**Aufwand:** S · **Risiko:** niedrig

---

### F-07 [Niedrig] — Irreführende Spec-Referenz im Kommentar

**Datei:** [tree-generator.ts:209-210](packages/simulator-core/src/tree-generator.ts#L209-L210)

**Beobachtung:**
Der Kommentar begründet Root-Rekrutierung ohne `duplicationRate` mit „(Root ist per Definition immer 100% aktiv, Spec §4.4)". `§4.4` in `02` ist aber die Tabellen-Spalte „DL". Gemeint ist `§3.2` (Root-Churn/Recruiting).

**Vorschlag:** Referenz auf `§3.2` korrigieren.

**Aufwand:** XS · **Risiko:** keins

---

## Empfohlene nächste Schritte

1. **F-01 zuerst** — die Mindest-Churn-Regel ist die einzige *fehlende* Produktregel und betrifft direkt die Plausibilität der Headline-Zahlen. Kleiner, isolierter Eingriff.
2. **F-02 als Regressionssockel** — vor jeder Änderung an der Compressed-Mode-Bein-Zählung einen Test „Exact vs. Compressed bei gleicher Endgröße liefert gleichen Rang" einfrieren (Invariante aus `02 §8`).
3. **F-03/F-04 als bewusste Entscheidung** — beides sind keine Tippfehler, sondern Modell-Entscheidungen, die dokumentiert statt still getroffen werden sollten. Ein ADR („Effektive AV: Rang-Prüfung vs. Provisionsbasis") wäre der richtige Ort.
4. **F-05–F-07** als Aufräum-Welle mitnehmen.

---

## Nicht reviewt

- **UI-Visualisierung** (`NetworkVisualizations.tsx`, `PersonTreeVisualizations.tsx`, Sunburst/RadialTree) — nur Datenzufluss gestreift, keine Render-Logik.
- **`simulator-realistic-growth`** (Dirichlet/Momentum-Interna) — nur die Normierungs-Schnittstelle `sourceWeights` betrachtet.
- **`example-line.ts`** — laut Doku didaktischer Nebenpfad, kein zweiter Berechnungspfad.
- **Persistenz-/Auth-Layer**, Quartals-Chart-Rendering, Zielbewertung (`simulator-goals`).
- **Laufende Tests** — die Findings sind aus dem Code abgeleitet, nicht durch Ausführen der Testsuite verifiziert.

---

## Annahmen und offene Fragen

- **(Annahme)** Mittlerer Detailgrad, Zielgruppe Reviewer/Architekt. Ich habe die genannten Kern-Dateien vollständig gelesen und die Regel-Doku `02`/`01` als Soll herangezogen; die restliche Codebasis nur gezielt (grep) berührt.
- **(Annahme)** Ablage in `_reviews/` folgt der bestehenden Repo-Konvention (dort liegen bereits `arc42-review-*` und `phase-2-3-followup-*`), nicht dem Agent-Default `docs/reviews/`.
- **Offen (F-01):** Ist die Mindest-Churn-Regel bewusst zurückgestellt oder schlicht nie umgesetzt worden? Das entscheidet, ob es ein Bug oder ein offener Task ist.
- **Offen (F-03):** Soll die effektive AV die Provisionsbasis erhöhen (aktuell) oder nur die Rang-Prüfung speisen?
- **Nicht abschließend prüfbar ohne LifePlus-PDF:** Die exakte Slot-Reihenfolge in Phase 3 (`1*/2*/3*Diamant` → Raten 0,03/0,03/0,02 positionsbasiert) ist gegen die Konstanten konsistent, aber gegen das Original-PDF nur über die Referenz-Tests (`reference-rank.test.ts`) abgesichert — nicht in diesem Review nachgerechnet.
