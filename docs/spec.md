# Stoffwerkstatt – Specification (MVP)

> Place this file at `docs/spec.md`. Every issue links here. This spec is the result of a completed grill session; do **not** re-run the interview. Ask only about gaps or contradictions you find.

- **Status:** Approved for MVP planning
- **Owner:** Felix Müllner (solo, hobby project, unpaid)
- **License:** MIT (code). Data: only free/open sources, attributed (see §11).
- **Run branch:** _(Claude Code: record the active `run/...` branch here)_

---

## 1. What this is

A small, free, German-language web app that helps students understand chemical substances. A student types a formula (e.g. `H2SO4`) or a name (e.g. `Schwefelsäure`) and sees:

1. a drawing of the substance,
2. what kind of bonding holds it together (covalent / ionic / metallic),
3. a very short info card about the substance and each element in it.

The app is named **Stoffwerkstatt** (not "Molekül…") on purpose: salt and metals are not molecules, and that distinction is itself something the app teaches.

## 2. Users and context

- **Users:** students of an Austrian agricultural secondary school, ages ~14–19 (five-year program with Matura).
- **Do not name the school** anywhere public: not in code, UI, README, docs, issues, commits or PRs. The school has not agreed to be named. Adding the name later is the owner's decision.
- **Assume** they are not enthusiastic about chemistry. Every screen must be understandable to the youngest, least advanced student.
- **Content focus:** substances students actually meet in agriculture, food and the barn (fertilizer, lime, slurry gases, lactic acid in silage, …), plus core basics.
- **Primary device:** phone (phone-first layout; must also look clean on a laptop).
- **Language:** German only. Code, comments, tests and docs in English.
- **No domain expert is available.** Neither the owner nor anyone reachable checks the chemistry. All chemistry decisions in this spec were made during planning and are listed in §14 (Assumptions). Content correctness is protected by sources + automated checks (§11, §12).

## 3. Scope

### 3.1 MVP (in scope)

- Search by **formula** (incl. parentheses) and by **German name** (case-insensitive, incl. common names).
- Curated **collection of 50 substances** (§9) and **16 element cards** (§10).
- Result screen: drawing (2D default, 3D toggle for molecules), substance class label, bonds colored by bond type, substance card, element chips → element card.
- Isomer hint for ambiguous formulas.
- Friendly capitalization hints ("Meintest du …?") and a "not in the collection yet" state.
- Static site on GitHub Pages, automatic deploy from `main` via CI.
- "Fehler gefunden?" link → GitHub Issues.
- Small "Über" page with sources and licenses.

### 3.2 Backlog (explicitly out of MVP)

- Suggestion feature: enter n elements → app suggests possible substances, connected to the visualizer.
- Search history, favorites, user accounts, any backend.
- PWA (installable, offline).
- English / language switch.
- Input of ions (`NH4+`, `SO4^2-`) and hydrates (`CuSO4·5H2O`).
- Live PubChem fallback for substances outside the collection (structure only, no info card).
- Lewis structures (lone pairs).
- More substances.

### 3.3 Extensibility requirements (apply to MVP code)

The MVP must not block the backlog. Concretely:

- **All data access goes through a repository interface** (§7.2). The UI never knows whether data comes from static JSON or a future API/backend.
- **All content lives in data files**, not in components.
- **Every substance has its own URL** (hash route), so history/favorites/sharing can be added later.
- **Every third-party library sits behind a small adapter** owned by us (§7.3).
- **All UI strings go through one strings module** (German only for now), so a language switch can be added without touching components.

## 4. Glossary (seed for `context.md`)

| Term (code) | German UI term | Definition | Avoid |
|---|---|---|---|
| Substance | Stoff | Anything in the collection: molecule, ionic compound, metal or network solid | "molecule" as a generic word, "compound" (excludes elements) |
| SubstanceClass | Stoffart | One of `molecule`, `ionic`, `metal`, `network` | "type" (ambiguous) |
| Molecule | Molekül | Discrete particle of covalently bonded atoms (H₂O, O₂) | – |
| IonicCompound | Ionenverbindung | Lattice of ions (NaCl, NH₄NO₃) | "salt molecule" |
| Metal | Metall | Lattice of metal atoms in an electron sea (Fe) | – |
| NetworkSolid | Netzwerk | Giant covalent structure (SiO₂) | – |
| BondType | Bindungsart | Per bond: `covalent-nonpolar`, `covalent-polar`, `ionic`, `metallic` | – |
| Collection | Sammlung | The curated set of 50 substances | "database" |
| SubstanceCard | Stoffkarte | Info about the substance (occurrence, rarity, uses) | – |
| ElementCard | Elementkarte | Info about one element | – |
| ElementChip | – | Small tappable element button below the drawing | – |
| Isomer | Isomer | Different substance with the same element counts | – |
| ElementCounts | Summenformel | Map element → count; the canonical key for formula matching | "Hill string" in UI |
| Rarity | Seltenheit | 1–5 dot score + comparison sentence | – |

## 5. User-facing behavior

### 5.1 Start screen

- App name, one-line tagline (e.g. "Tippe eine Formel oder einen Namen ein").
- One search box, autofocus on desktop (not on phone, to avoid the keyboard popping up).
- 3–4 example buttons: `H₂O`, `NaCl`, `CO₂`, `Fe` (one per substance class except network).
- Footer: "Über" · "Fehler gefunden?".

### 5.2 Search input rules

1. **Normalize:** trim; remove inner spaces; convert subscript digits (`₂` → `2`) and the middle dot is rejected (hydrates out of scope).
2. **Formula path** (input consists only of letters, digits and `()`):
   - Parse **case-sensitively** with the formula adapter → `ElementCounts`.
   - Match `ElementCounts` against all substances and their isomers. Different notations of the same formula match (`C2H5OH` = `C2H6O` → Ethanol).
3. **Name path** (always runs too): case-insensitive match against German name and common names, with normalization `ä→ae, ö→oe, ü→ue, ß→ss` applied to both sides. Exact match wins; otherwise prefix match.
4. **Live suggestions** while typing: up to 5 matches from both paths. Enter or tapping a suggestion opens the result.
5. **Capitalization hint:** if the formula path finds nothing (parse error or no match), generate all valid element tokenizations of the input with different capitalization and offer those that are in the collection:
   - `h2so4` → "Meintest du **H₂SO₄** (Schwefelsäure)?"
   - `Co` → parses as cobalt, not in collection → "Co ist das Element Cobalt (nicht in der Sammlung). Meintest du **CO** (Kohlenstoffmonoxid)? Groß- und Kleinschreibung zählt in der Chemie."
6. **Not found:** "Noch nicht in der Sammlung" + the example buttons. No error styling, no blame.
7. If the formula matches the main entry **and** isomers exist, open the main entry (the most common one) and show the isomer hint (§5.4).

### 5.3 Result screen (top to bottom)

1. **Header:** formula with real subscripts, German name, substance class label as a colored badge (e.g. "Ionenverbindung").
2. **Drawing** (§6). Toggle "2D / 3D" only for molecules.
3. **One-line explanation of the bonding**, e.g. "Ionenverbindung: Ammonium-Ionen und Nitrat-Ionen ziehen sich an. Innerhalb der Ionen halten Atombindungen zusammen."
4. **Substance card:** Vorkommen · Seltenheit · Verwendung · source link(s).
5. **Element chips:** one per element; tap → element card (bottom sheet on phone, side panel on wide screens). Tapping an atom in the drawing opens the same card.
6. **Isomer hint** if applicable (§5.4).
7. Back to search (search box stays reachable at the top).

URL: `#/stoff/<id>` (e.g. `#/stoff/schwefelsaeure`). Direct links and browser back/forward must work.

### 5.4 Isomers

- Shown as a small section "Gleiche Formel, anderer Stoff:" with name + small 2D drawing per isomer.
- Isomers have **no** info card in the MVP and are not standalone collection entries.
- Isomer data: name, `ElementCounts`, structure (SMILES) — fetched like main structures.

### 5.5 Element card

- Symbol, German name, atomic number.
- Vorkommen · Seltenheit (dots + comparison) · Verwendung · Elektronegativität (Pauling) · source link(s).

### 5.6 Rarity display

- 5-dot scale (●●●○○) **plus** a relatable comparison sentence underneath ("Seltener als Gold", "Macht fast die Hälfte der Erdkruste aus").
- **Elements:** dots derived from abundance in the Earth's crust (mass fraction), with documented override (see §14):

  | Dots | Crustal abundance |
  |---|---|
  | ●●●●● | ≥ 1 % |
  | ●●●●○ | 0.01 % – < 1 % |
  | ●●●○○ | 1 ppm – < 100 ppm |
  | ●●○○○ | 0.01 ppm – < 1 ppm |
  | ●○○○○ | < 0.01 ppm |

- **Substances:** curated dots + sentence. Substances that barely occur naturally show **"kommt in der Natur kaum vor – wird industriell hergestellt"** instead of dots (this is how the app answers "artificial" substances).

### 5.7 Global UI rules

- Minimal, calm, lots of whitespace, one font, no decoration without meaning.
- **Bond-type colors** are used consistently everywhere (drawing, badges, legend). Colors are defined once as CSS variables.
- **Color is never the only signal:** every bond type also has a label and/or line pattern (accessibility, color-blindness).
- Dark mode follows the system setting (`prefers-color-scheme`).
- Touch targets ≥ 44 px. Readable at 320 px width.
- No tracking, no cookies, no analytics, no external requests at runtime.

## 6. Visualization

| Substance class | 2D (default) | 3D toggle | Notes |
|---|---|---|---|
| Molecule | Structural formula from SMILES | Ball-and-stick, rotatable | Atoms tappable → element card |
| Ionic compound | Ions with charges side by side (formula unit), covalent bonds inside polyatomic ions | – | Plus a small **schematic** lattice illustration labeled "schematisch" |
| Metal | Shared "atom cores in electron sea" illustration with the element symbol | – | One reusable SVG component |
| Network solid | Schematic network excerpt | – | SiO₂ only in MVP |

- Each bond is colored by `BondType`. A small legend appears below the drawing.
- 3D code loads **only** when the user taps "3D" (dynamic import).
- Lattice / electron-sea / network illustrations are our own SVG components (a few templates, not one per substance).

## 7. Architecture

### 7.1 Stack

- **Vite + React + TypeScript** (strict mode), npm, current Node LTS (pinned in `.nvmrc` / devcontainer).
- **Styling:** plain CSS with CSS Modules; design tokens (colors incl. bond-type colors, spacing, dark mode) as CSS custom properties in one file. No CSS framework, no component library.
- **Routing:** hash-based (GitHub Pages has no SPA fallback).
- **Hosting:** GitHub Pages, deployed by GitHub Actions.

### 7.2 Layers

```
src/
  domain/          pure TypeScript, no React, no I/O – fully unit-tested
    formula/       parse adapter usage, ElementCounts, capitalization variants
    search/        formula + name search, normalization, ranking
    bonding/       ΔEN helper, labels/explanations from stored data
  data/
    repository.ts  interface: getSubstance(id), listSubstances(), search(query),
                   getElement(symbol), …
    static/        StaticJsonRepository – implements the interface from bundled JSON
  adapters/        thin wrappers around third-party libraries (§7.3)
  ui/              React components and screens
  strings/de.ts    all UI strings
data/
  substances.json  curated: ids, names, common names, formula, class, card texts, sources
  elements.json    curated: card texts, rarity, sources; facts (EN, abundance)
  structures/      generated by script (§8.2), committed
scripts/
  fetch-structures.ts
```

Rule: `ui/` may import `domain/`, `data/repository.ts` (interface) and `adapters/`; it must never import `data/static/` or a third-party chemistry library directly. Wiring happens in one place (app entry).

### 7.3 Third-party libraries (each behind an adapter)

| Job | Library | License | Adapter |
|---|---|---|---|
| Formula parsing | chemparse | MIT | `adapters/formulaParser.ts` (fallback candidate: mf-parser, MIT) |
| 2D drawing | SmilesDrawer | MIT | `adapters/drawing2d.ts` |
| 3D view | 3Dmol.js | BSD-3-Clause | `adapters/viewer3d.ts` (lazy-loaded) |
| Schema validation | zod (or equivalent) | MIT | used in data loading + tests |

Before installing any of these, verify license and maintenance status (last release, open issues) and list them in the PR under "New dependencies".

## 8. Data

### 8.1 Data model (TypeScript, indicative)

```ts
type SubstanceClass = 'molecule' | 'ionic' | 'metal' | 'network';
type BondType = 'covalent-nonpolar' | 'covalent-polar' | 'ionic' | 'metallic';
type ElementCounts = Record<string, number>;

interface Source { label: string; url: string; }

interface RarityInfo {
  dots: 1 | 2 | 3 | 4 | 5 | null;   // null = "kommt in der Natur kaum vor"
  comparison: string;               // German sentence
}

interface Substance {
  id: string;                 // slug, e.g. "schwefelsaeure"
  formula: string;            // display formula as commonly written, e.g. "H2SO4", "Ca(OH)2"
  elementCounts: ElementCounts;
  name: string;               // German name
  commonNames: string[];      // e.g. ["Löschkalk"]
  substanceClass: SubstanceClass;
  bondingSummary: string;     // one German sentence (§5.3)
  occurrence: string;         // Vorkommen, ≤ 25 words
  rarity: RarityInfo;
  uses: string;               // Verwendung, ≤ 25 words
  sources: Source[];          // ≥ 1
  isomers: Isomer[];          // may be empty
  isExample?: boolean;        // shown on start screen
}

interface Isomer { name: string; elementCounts: ElementCounts; }

interface ElementInfo {
  symbol: string; name: string; atomicNumber: number;
  electronegativity: number | null;   // Pauling
  crustalAbundance: number;           // mass fraction
  rarity: RarityInfo;
  rarityOverrideReason?: string;
  occurrence: string; uses: string;   // ≤ 25 words each
  sources: Source[];
}

interface Structure {        // generated, per substance id and per isomer
  smiles: string;
  atoms: { id: number; element: string; charge?: number; x?: number; y?: number; z?: number }[];
  bonds: { from: number; to: number; order: number; type: BondType }[];
  pubchemCid?: number;
}
```

### 8.2 Structure data pipeline

- `scripts/fetch-structures.ts` is run **manually** (never in CI). It reads the collection, fetches SMILES and 3D coordinates from PubChem (PUG-REST), assigns `BondType` per bond (from stored classification + ΔEN, see §14), and writes `data/structures/*.json`.
- Output is committed; changes are reviewed as diffs in PRs.
- CI and the live app never contact PubChem.
- Metals, lattice and network illustrations do not need PubChem data.

### 8.3 Content rules (info texts)

- Texts are drafted by Claude Code from **open sources** (PubChem, Wikidata, and other openly licensed/public-domain sources), then reviewed by the owner for tone and readability only.
- **Every** substance and element card has ≥ 1 source URL. No source → no text.
- **Never invent a chemistry fact.** If a fact cannot be sourced, leave it out and flag it in the PR.
- Do **not** copy text from Wikipedia (CC BY-SA). Facts may be used; wording must be original.
- German, short sentences, plain language (target reading level: a 14-year-old). Each field ≤ 25 words. Neutral, friendly tone, no jokes that age badly.
- Agriculture/food relevance first where true (e.g. Ammoniumnitrat → Dünger).

## 9. The collection (50 substances)

Columns: display formula · German name (common names) · class. Isomers in the last column are shown as hints only.

| # | Formula | Name (common names) | Class | Isomers shown |
|---|---|---|---|---|
| **Basics & gases** |||||
| 1 | H2O | Wasser | molecule | |
| 2 | O2 | Sauerstoff | molecule | |
| 3 | N2 | Stickstoff | molecule | |
| 4 | H2 | Wasserstoff | molecule | |
| 5 | CO2 | Kohlenstoffdioxid (Kohlendioxid) | molecule | |
| 6 | CO | Kohlenstoffmonoxid (Kohlenmonoxid) | molecule | |
| 7 | CH4 | Methan | molecule | |
| 8 | NH3 | Ammoniak | molecule | |
| **Metals** |||||
| 9 | Fe | Eisen | metal | |
| 10 | Cu | Kupfer | metal | |
| 11 | Al | Aluminium | metal | |
| 12 | Zn | Zink | metal | |
| 13 | Mg | Magnesium | metal | |
| **Fertilizer & soil** |||||
| 14 | NH4NO3 | Ammoniumnitrat | ionic | |
| 15 | KNO3 | Kaliumnitrat (Kalisalpeter) | ionic | |
| 16 | KCl | Kaliumchlorid | ionic | |
| 17 | K2SO4 | Kaliumsulfat | ionic | |
| 18 | (NH4)2SO4 | Ammoniumsulfat | ionic | |
| 19 | CaCO3 | Calciumcarbonat (Kalk, Kalkstein) | ionic | |
| 20 | CaO | Calciumoxid (Branntkalk) | ionic | |
| 21 | Ca(OH)2 | Calciumhydroxid (Löschkalk) | ionic | |
| 22 | Ca3(PO4)2 | Calciumphosphat | ionic | |
| 23 | MgSO4 | Magnesiumsulfat (Bittersalz) | ionic | |
| 24 | CO(NH2)2 | Harnstoff | molecule | |
| **Acids & bases** |||||
| 25 | H2SO4 | Schwefelsäure | molecule | |
| 26 | HCl | Chlorwasserstoff (Salzsäure) | molecule | |
| 27 | HNO3 | Salpetersäure | molecule | |
| 28 | H3PO4 | Phosphorsäure | molecule | |
| 29 | NaOH | Natriumhydroxid (Ätznatron) | ionic | |
| 30 | KOH | Kaliumhydroxid (Ätzkali) | ionic | |
| **Salts & minerals** |||||
| 31 | NaCl | Natriumchlorid (Kochsalz) | ionic | |
| 32 | NaHCO3 | Natriumhydrogencarbonat (Natron) | ionic | |
| 33 | CaSO4 | Calciumsulfat (Gips) | ionic | |
| 34 | CuSO4 | Kupfersulfat | ionic | |
| 35 | SiO2 | Siliciumdioxid (Quarz, Sand) | network | |
| 36 | Fe2O3 | Eisen(III)-oxid (Rost) | ionic | |
| **Food & biology** |||||
| 37 | C6H12O6 | Glucose (Traubenzucker) | molecule | Fructose, Galactose |
| 38 | C12H22O11 | Saccharose (Haushaltszucker) | molecule | Lactose, Maltose |
| 39 | C2H5OH | Ethanol (Alkohol) | molecule | Dimethylether |
| 40 | CH3COOH | Essigsäure | molecule | Ameisensäuremethylester |
| 41 | C3H6O3 | Milchsäure | molecule | Glycerinaldehyd |
| 42 | C6H8O7 | Citronensäure | molecule | |
| 43 | C6H8O6 | Ascorbinsäure (Vitamin C) | molecule | |
| 44 | C3H8O3 | Glycerin | molecule | |
| 45 | H2O2 | Wasserstoffperoxid | molecule | |
| **Environment & barn** |||||
| 46 | N2O | Distickstoffmonoxid (Lachgas) | molecule | |
| 47 | NO2 | Stickstoffdioxid | molecule | |
| 48 | SO2 | Schwefeldioxid | molecule | |
| 49 | O3 | Ozon | molecule | |
| 50 | H2S | Schwefelwasserstoff | molecule | |

Start-screen examples (`isExample`): Wasser, Natriumchlorid, Kohlenstoffdioxid, Eisen.

The owner may strike or swap entries at any time; tests must not hard-code the count 50 except via the data file.

## 10. Elements (16)

H, C, N, O, Na, Mg, Al, Si, P, S, Cl, K, Ca, Fe, Cu, Zn.

A test asserts that every element appearing in any substance (incl. isomers) has an element card, and vice versa (no orphan cards).

## 11. Sources, licenses, attribution

- **PubChem** (NIH): structures, SMILES, identifiers. Attribute on the "Über" page.
- **Wikidata** (CC0): names, element facts.
- Element facts (electronegativity, crustal abundance) are facts; record the source per value or per dataset.
- The "Über" page lists: purpose, MIT license, data sources with links, third-party library licenses, "Fehler gefunden?" link, and the note that content is AI-drafted from cited sources and not reviewed by a chemist.
- No Wikipedia text (see §8.3).

## 12. Quality and testing

### 12.1 Tooling (Phase 0)

- TypeScript strict, ESLint, Prettier, Vitest, Playwright, `npm audit` (fail on high/critical in production deps), Dependabot (npm + GitHub Actions), secret scanning (gitleaks), all in CI.

### 12.2 TDD policy

- **Test-first (red-green-refactor)** for all logic in `domain/` and `data/`: parser adapter, search, normalization, capitalization hints, isomer matching, rarity banding, repository.
- **Visual UI** (layout, drawings, styling) is a TDD exemption: test behavior after the fact (component tests where useful, Playwright for flows), check looks by eye. Declare as `TDD exemption: visual UI`.
- Tests are never changed just to make them pass.

### 12.3 Data tests (must exist)

- Every substance validates against the schema (zod).
- Every `formula` parses and its parsed `ElementCounts` equals the stored `elementCounts`.
- Every substance has a structure file (molecules, ionic) or an illustration mapping (metal, network).
- Every bond in every structure has a `BondType`.
- Every substance and element card has ≥ 1 source and non-empty `occurrence` and `uses` within the word limit.
- Ids, names and common names are unique (after normalization).
- No two collection entries share `ElementCounts` (would make formula search ambiguous); isomers are not counted.
- Element cards ↔ elements used: exact match (§10).

### 12.4 Unit test examples (non-exhaustive)

- `H2SO4` → Schwefelsäure; `Ca(OH)2` → Calciumhydroxid; `C2H6O` and `C2H5OH` → Ethanol (+ isomer Dimethylether).
- `schwefelsaeure`, `Schwefelsäure`, `SCHWEFELSÄURE` → Schwefelsäure; `kochsalz` → Natriumchlorid.
- `h2so4` → hint H₂SO₄; `Co` → cobalt explanation + hint CO; `xyz` → not found.

### 12.5 End-to-end (Playwright, phone viewport + one desktop run)

1. Type `H2SO4` → result shows name, class badge, drawing.
2. Type `kochsalz` → Natriumchlorid result.
3. Type `co` → capitalization hint → tap → CO result.
4. On a result, tap an element chip → element card opens; tap an atom in the drawing → same.
5. Toggle 3D on a molecule → 3D view renders (canvas present).
6. Unknown input → "Noch nicht in der Sammlung" with example buttons.
7. Deep link `#/stoff/wasser` opens directly; browser back returns to search.

### 12.6 Non-functional targets

- Lighthouse (mobile) ≥ 90 for performance and accessibility on the result page.
- Initial JS (excluding the lazy 3D chunk) ≲ 200 KB gzipped.
- Works in current Chrome, Safari (iOS) and Firefox.

## 13. CI/CD

- **On every PR** (into `main` and `run/**`): typecheck, lint, format check, unit/data tests, Playwright, `npm audit`, gitleaks, production build.
- **On merge into `main`:** same checks, then deploy to GitHub Pages (official `actions/deploy-pages`). Deploys happen only if all checks pass. The owner's PR review into `main` is the gate.
- The Pages base path (`/<repo-name>/`) is configured in Vite.

## 14. Assumptions and chemistry decisions (no expert review)

Made during planning; change only with owner approval and flag in the PR.

1. **Bond classification is stored, not computed.** Each substance's class and each bond's type are curated. The electronegativity difference (Pauling) is shown as the explanation, using the common school thresholds: ΔEN < 0.4 nonpolar covalent, 0.4–1.7 polar covalent, > 1.7 ionic. Where the stored classification and the ΔEN rule disagree, the stored one wins and the explanation says the rule is a rule of thumb.
2. **Metal + non-metal compounds are treated as ionic** at school level, including the oxides Fe₂O₃ and CaO.
3. **HCl** is the molecule hydrogen chloride; "Salzsäure" (its solution in water) is a common name that maps to it.
4. **CaSO₄** is listed as calcium sulfate; gypsum ("Gips") is strictly the dihydrate. The card text mentions that Gips contains bound water; hydrate input stays out of scope.
5. **Nitrogen rarity:** crustal abundance would rate N as rare, though it makes up most of the air. Use a rarity override with a reason (e.g. dots from atmospheric abundance, comparison "Macht rund drei Viertel der Luft aus"). Same mechanism for any similar case.
6. **Lattice, electron-sea and network drawings are schematic**, not crystallographically accurate, and are labeled "schematisch".
7. **3D view only for molecules** in the MVP.
8. **Polyatomic ions** (NH₄⁺, NO₃⁻, SO₄²⁻, PO₄³⁻, OH⁻, CO₃²⁻, HCO₃⁻) are drawn with covalent bonds inside and ionic attraction between ions.
9. **"Most common" isomer** = the collection entry; isomers are hints only.

## 15. Definition of Done (MVP)

- All substances in the collection are findable by formula (incl. parentheses and alternative notations) and by German/common name.
- Each shows a drawing (2D; 3D toggle for molecules), the class badge, bonds colored by type with a legend, and a one-line bonding explanation.
- Substance card and all 16 element cards show Vorkommen, Seltenheit (dots + comparison) and Verwendung, each with ≥ 1 source link.
- Capitalization hints and the "not in the collection yet" state work as in §5.2.
- Isomer hints work for the entries in §9.
- Phone-first, German UI, automatic dark mode, accessibility rules of §5.7 met.
- All tests in §12 exist and pass in CI; non-functional targets met.
- Live on GitHub Pages via CI, MIT license, "Über" page and "Fehler gefunden?" link present.

## 16. Open questions

- Is an imprint/"Offenlegung" needed for a private, non-commercial Austrian website? Owner to check before sharing the URL with the school.
- Final GitHub repo name (affects the Pages URL and Vite base path).
