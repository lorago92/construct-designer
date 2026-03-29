# SeqMind — TODO Roadmap

_Last updated: March 29, 2026_

---

## 🔴 PRIORITY 1 — MVP Launch Blockers

### 1. Smart Insertion Logic + Position Constraints
**Status:** Not started
**Complexity:** High — requires careful UX strategy

**The challenge:** Multi-part constructs mean "N-term" and "C-term" are relative to the WHOLE assembly, not individual parts. A signal peptide should go at the global N-terminus, but a tag could go at C-term of a specific protein OR at the global C-term.

**Cases to handle:**
- Signal added when construct is empty → becomes first part (N-term)
- Signal added when construct has parts → prompt: "Add to N-terminus of construct?" or "Add before [specific part]?"
- Tag added → prompt: "Add to C-terminus of construct?" or "Add after [specific part]?"
- Linker added → prompt: "Insert between [part A] and [part B]?" (show dropdown of junctions)
- Conjugation added → similar to tags, usually C-term but context-dependent

**Strategy options:**
- A) Auto-place with smart defaults + ability to drag-reorder after
- B) Always ask via a small popup: "Where should this go?" with visual construct preview
- C) Hybrid: auto-place obvious cases (signal → N-term if nothing there), ask for ambiguous ones

**Decision needed:** Which strategy? Probably B or C.

**Position constraints (NEW):** Add `position` metadata to `parts_library.json` entries:
- `"position": "n-term"` — SUMO, MBP, GST (these are N-terminal solubility fusions; SUMO literally won't cleave C-terminally because ULP1 recognizes the 3D fold)
- `"position": "c-term"` — LPETG/sortase tag, most conjugation tags
- `"position": "either"` — His6, FLAG, Strep-tag II, etc.
- Warn on placement violation: "⚠ SUMO is N-terminal only — ULP1 protease cannot cleave C-terminal SUMO fusions"
- Natural pairing with this item since both are about intelligent part placement

---

### 2. Save / Load Projects
**Status:** Not started
**Complexity:** Medium

**Options:**
- **File-based:** Export/import `.seqmind.json` files (works offline, no server needed)
- **localStorage:** Auto-save current session (risk: browser clears it)
- **Both:** Auto-save to localStorage + manual export/import to file

**Data to persist:**
- `AppState.construct.parts` (with all `_inserted`, `_deleted`, `mutations`, etc.)
- `AppState.expandedParts`
- `AppState.uniprotCache` (so offline reload works)
- Construct name (see item #14)
- Version number for forward compatibility

---

### 3. DNA Sequence Export
**Status:** Not started  
**Complexity:** Medium-High

**What's needed:**
- Reverse-translate assembled protein sequence → DNA
- Codon optimization for target organism (human, CHO, E. coli, yeast, insect)
- Use validated codon usage tables (Kazusa, GenScript algorithm, or similar)
- Export as plain DNA, GenBank format, and ideally SnapGene (.dna) format

**Codon optimization engines to evaluate:**
- GenScript OptimumGene algorithm (industry gold standard)
- IDT Codon Optimization Tool
- Kazusa codon usage database (public, well-validated)
- JCat (Java Codon Adaptation Tool — academic)
- Custom implementation using organism-specific codon frequency tables

**Key question:** Build our own from codon tables, or integrate with an existing API?

---

### 4. GenBank / SnapGene Export
**Status:** Not started  
**Complexity:** Medium

**GenBank format (.gb):**
- Header: LOCUS, DEFINITION, ACCESSION, FEATURES
- Feature annotations for each part (signal, linker, tag, protein domains)
- Mutations annotated as /note
- DNA sequence (requires codon optimization first, or protein-only GenBank)

**SnapGene format (.dna):**
- Binary format, but SnapGene Viewer is free
- Would require understanding their file spec or using protein-only annotation

**Minimum viable:** GenBank protein format with annotated features. DNA version depends on item #3.

---

### 5. Abs 0.1% + pI Calculation (stats bar bundle)
**Status:** Not started  
**Complexity:** Low

**Abs 0.1%:**
- Formula: Abs 0.1% = (Ext. coeff. at 280nm) / (MW in Daltons) × 10
- We already calculate Ext. coeff. (ox/red) and MW. Just need the division and display.

**pI (isoelectric point) — NEW:**
- Every biochemist expects this alongside MW and ext. coeff.
- Needed for choosing ion exchange chromatography conditions (critical for purification design).
- Algorithm: iterative Henderson-Hasselbalch over ionizable groups (D, E, C, Y, H, K, R + N-term + C-term). Well-established pKa tables (Lehninger, EMBOSS, ExPASy).
- Display in stats bar: `pI: 6.8`
- Implementation: bisection search on pH 0–14 where net charge = 0. ~20 lines of code.

**Disulfide bond count — NEW:**
- Already counting Cys for extinction coefficient. Display `floor(nC/2)` potential disulfide pairs alongside ext. coeff.
- One line of code. Show as: `Cys: 6 (3 potential S-S)`

---

### 6. Mobile / Tablet Responsiveness
**Status:** Partial (basic @media query at 900px exists)
**Complexity:** Medium

**What needs work:**
- Construct strip: vertical layout on mobile instead of horizontal
- Engineering panel: slide-out drawer or bottom sheet instead of side panel
- Sequence popups: full-width on small screens
- Touch-friendly: larger hit targets for residue clicks
- Zone 1 sequence: horizontal scroll instead of wrap on narrow screens

---

### 7. Internal Protease Site Scanning
**Status:** Not started  
**Complexity:** Low-Medium  
**NEW — Scientific safety feature**

**The problem:** User adds a TEV-cleavable linker. Their target protein contains ENLYFQ at position 234. Protein gets cleaved at both sites during purification. Weeks of wasted lab time.

**Implementation:**
- When a cleavable linker is added to the construct, scan all other parts for the cleavage motif
- Motifs to scan (already in `parts_library.json`): `ENLYFQ` (TEV), `LEVLFQ` (HRV 3C), `LVPR` (Thrombin), `IEGR` (Factor Xa), `DDDDK` (Enterokinase)
- Also scan when a new protein is loaded into a construct that already has a cleavable linker
- Display: warning banner in Zone 1 / stats area: "⚠ Internal TEV site found at position 234 of EGFR_HUMAN. Your construct contains a TEV-cleavable linker — both sites will be cleaved."
- Optionally highlight the internal site in the sequence view

**Complexity is low because:** the recognition sequences are short fixed strings, already defined in our library. It's a `string.indexOf()` scan.

---

### 8. N-Glycosylation Sequon Scanning
**Status:** Not started  
**Complexity:** Low  
**NEW — Scientific safety feature**

**The problem:** N-linked glycosylation at N-X-S/T (where X≠P) dramatically affects protein folding, stability, solubility, and immunogenicity in mammalian and insect expression systems. Users need to know where these sites are.

**Implementation:**
- Regex scan: `/N[^P][ST]/g` on the assembled sequence
- Display as count in stats bar: `N-glyc sites: 3`
- Highlight sequons on hover/click in the sequence view (similar to domain highlighting)
- Context-dependent: most relevant for mammalian (HEK/CHO) and insect (Sf9/Hi5) systems. Could show a note: "N-glycosylation occurs in mammalian and insect cells but not in E. coli"
- Also flag sequons that were introduced or destroyed by mutations

**Future extension:** O-glycosylation prediction (requires ML, much harder — defer to P3).

---

## 🟡 PRIORITY 2 — High Value Features

### 9. Construct Validation Warnings ("Lint")
**Status:** Not started  
**Complexity:** Medium  
**NEW — Catches novice mistakes before they hit the bench**

**Warnings to implement (ordered by impact):**
- Signal peptide not at N-terminus: "⚠ Signal peptides must be at the N-terminus to function"
- Missing initiator Met: "⚠ Construct does not start with Met — add M or a signal peptide"
- Tag without adjacent linker: "💡 Consider adding a linker between [His6] and [EGFR] for better accessibility"
- Duplicate signal peptides: "⚠ Construct has 2 signal peptides — only the N-terminal one will be used"
- Cleavable linker without a tag: "💡 You added a TEV site but no purification tag — did you mean to add one?"
- SUMO/MBP/GST at C-terminus (ties into item #1 position constraints)

**UX:** Non-blocking warning banner below the construct bar in Zone 1. Yellow for suggestions, red for likely errors. Dismissible per-warning.

---

### 10. Cleavable Linker + Tag Pairing Guidance
**Status:** Not started  
**Complexity:** Low  
**NEW — Educational guardrail for novice users**

**The problem:** Novice users add His6-GGGGS-Protein instead of His6-TEV-GGGGS-Protein. The protein works fine, but you can't remove the tag later for crystallography, in vivo studies, or downstream processing.

**Implementation:**
- When a purification tag is adjacent to a protein with only a flexible linker (no cleavable linker) between them, show a contextual hint: "💡 Consider adding a cleavable linker (e.g., TEV) between [His6] and [Protein] for optional tag removal"
- Don't block — just suggest. Experienced users know when they want a permanent tag.
- Pairs well with the Guided Builder (existing item #17): the wizard can ask "Do you want to be able to remove the tag later?"

---

### 11. FASTA Header with Mutations
**Status:** Not started  
**Complexity:** Low  
**NEW — Trust and traceability**

**Current:** `>EGFR_HUMAN | 1210 aa | SeqMind`
**Proposed:** `>EGFR_HUMAN | IgK-leader–EGFR(K745R,L858R)–TEV–His6 | 1248 aa | SeqMind`

**Implementation:**
- List all parts in order: `signal–protein(mutations)–linker–tag`
- Mutations formatted as standard nomenclature: `K745R,L858R`
- Include insertion/deletion summary if present: `+2ins,-1del`
- Users share these FASTAs with collaborators and gene synthesis vendors — the header IS the construct documentation

---

### 12. N-End Rule / N-Degron Warnings
**Status:** Not started  
**Complexity:** Low  
**NEW — Protein stability awareness**

**The problem:** After signal peptide cleavage or tag removal, the exposed N-terminal residue determines protein half-life via the N-end rule pathway. In E. coli: R, K, L, F, Y, W = minutes half-life. In mammalian: same plus N, Q (deamidated by NTAN1/NTAQ1).

**Implementation:**
- Identify the first residue after each cleavable site in the construct
- Cross-reference against N-degron tables for the selected expression system
- Warning: "⚠ After TEV cleavage, the exposed N-terminal residue is Leu — half-life ~2 min in E. coli (N-end rule)"
- Reference: Varshavsky (2011) for the canonical table

---

### 13. Construct Naming
**Status:** Not started  
**Complexity:** Low  
**NEW — Basic usability**

**Implementation:**
- Editable text field at top of Zone 1: "Untitled construct" → click to rename
- Auto-suggest based on parts: "IgK–EGFR–TEV–His6"
- Feeds into: FASTA header (item #11), save/load filename (item #2), URL sharing (item #19)
- Stored in `AppState.construct.name`

---

### 14. BLAST for Custom Sequences
**Status:** Not started  
**Complexity:** High — UX design critical

**The question:** When a user pastes a custom sequence, should we BLAST it before or after adding to the construct?

**Option A — BLAST before adding (popup):**
- User pastes sequence → "Analyzing..." → popup shows: "This looks like Human IgG1 Fc region (98% identity to P01857 residues 100-330). Classify as: [Protein] [Tag] [Custom]"
- Pro: Clean data from the start
- Con: Slower workflow, requires network

**Option B — BLAST after adding (inline suggestion):**
- Sequence added immediately as "Custom"
- Background BLAST runs → notification appears: "This sequence matches P01857. [Link to UniProt] [Reclassify]"
- Pro: Fast workflow, non-blocking
- Con: Might need to restructure the part after classification

**Option C — Dedicated BLAST section:**
- Separate tab/tool: "Identify Sequence"
- User pastes → gets full analysis → then clicks "Add to construct" with proper classification
- Pro: Clean separation of concerns
- Con: Extra step

**Recommendation:** Option B with Option C as a secondary tool. Fast for experts, thorough for newcomers.

**Technical:** NCBI BLAST API (free, rate-limited) or UniProt sequence search endpoint.

---

### 15. Restriction Site Analysis (DNA level)
**Status:** Not started  
**Complexity:** Medium (depends on DNA export)
**Prerequisite:** Item #3 (DNA sequence)

**Features:**
- Scan assembled DNA for common restriction sites
- Highlight sites on the construct map
- Filter by enzyme (NEB catalog)
- Flag sites that would cut within coding regions
- "Silent mutation" suggestions to remove unwanted sites

---

### 16. Trust & Source Attribution Banner
**Status:** Not started  
**Complexity:** Low

**Implementation:**
- Footer or subtle banner: "Sequence data from UniProt (Swiss-Prot reviewed) • Signal/tag/linker sequences validated against primary literature"
- Link to our validation methodology
- Show UniProt accessions inline where applicable (already partially done)
- "Last library update: March 2026" timestamp

---

### 17. Guided Construct Builder ("Build it in 30 seconds")
**Status:** Not started  
**Complexity:** Medium-High

**Concept:** A step-by-step wizard with button choices:

```
Step 1: What are you building?
  [Antibody] [Fusion protein] [Reporter] [Vaccine antigen] [Other]

Step 2: Expression system?
  [Mammalian (HEK/CHO)] [E. coli] [Insect] [Yeast]

Step 3: What's your target protein?
  [Search UniProt: ___________]

Step 4: Do you need a purification tag?
  [His6] [Twin-Strep] [FLAG] [None] [Other]

Step 5: Signal peptide?
  [Auto-select for your system] [Choose manually] [None]

→ [Build Construct]
```

**Result:** Pre-assembled construct with recommended parts, ready for customization.

**Alternative:** ChatBot-style interface with natural language: "I want to express human IL-6 in CHO cells with a His tag"

---

## 🟢 PRIORITY 3 — Differentiators

### 18. ESMFold / AlphaFold Structure Preview
**Status:** Not started  
**Complexity:** High

**Options:**
- ESMFold API (Meta, free, fast, works for custom/modified sequences)
- AlphaFold DB (DeepMind, pre-computed for known UniProt entries only)
- Hybrid: AlphaFold for known proteins, ESMFold for custom/assembled constructs

**What to show:**
- 3D structure viewer (Mol*, NGL, or 3Dmol.js)
- Color by part type (signal=green, protein=cyan, tag=purple)
- Highlight mutations in red
- Confidence coloring (pLDDT)

**Challenge:** Assembled multi-part constructs may not fold predictably. ESMFold handles single chains well but linker/tag effects are unpredictable. Need to set user expectations.

---

### 19. Collaboration / Sharing via URL
**Status:** Not started  
**Complexity:** Medium-High

**Options:**
- **URL hash encoding:** Compress construct state into URL (works for small constructs, no server needed)
- **GitHub Gist integration:** Save as JSON to Gist, share link
- **Firebase/Supabase backend:** Full project sharing, but adds server dependency

**Decision deferred.**

---

### 20. S-tag Addition
**Status:** Not started  
**Complexity:** Trivial (data entry only)  
**NEW**

Add to `parts_library.json`:
```json
{"name": "S-tag", "sequence": "KETAAAKFERQHMDS", "category": "detection", "use": "S-protein interaction detection — binds S-protein fragment of RNase A"}
```

15 aa, detection category. Five-minute addition.

---

### 21. Expression System Metadata on Tags
**Status:** Not started  
**Complexity:** Low  
**NEW**

Add `system: ["mammalian", "bacterial", "insect", "yeast"]` to tag entries in `parts_library.json`, matching the format already used on signal peptides. Enables filtering tags by expression system in the Engineering panel.

**Examples:**
- His6: all systems
- Twin-Strep: mammalian, bacterial (Strep-Tactin available for both)
- SUMO: bacterial (ULP1 endogenous in yeast — would cleave prematurely)
- GST: bacterial primarily

---

### 22. DNA vs Protein Paste Detection
**Status:** Not started  
**Complexity:** Low  
**NEW**

**Implementation:**
- When pasted sequence is >80% ATCGN characters → suggest: "This looks like a DNA sequence. [Translate to protein] [Keep as-is]"
- Offer 3-frame or 6-frame translation preview
- Auto-detect ORFs if possible

---

### 23. MW Monoisotopic Toggle
**Status:** Not started  
**Complexity:** Low  
**NEW**

Add a small toggle near MW display: **Avg** / **Mono**. Mass spec users need monoisotopic mass for intact protein analysis. Use standard monoisotopic residue masses (NIST values). Low priority since most users work with average mass.

---

## ✅ COMPLETED

### Session 3 — Ground Truth Integrity Fixes

- [x] **Bug fix: deletion revert used naive position math** — protein segment deletion revert (edit log ✕ button) used `d.origPos - seg.start` instead of `origPosToLocalIdx(seg, d.origPos)`. When insertions and deletions coexisted in the same segment, reverting a deletion re-inserted the AA at the wrong position. Fixed in both protein segment and non-protein part paths.
- [x] **Bug fix: deletion revert didn't shift `_inserted` indices** — after re-inserting a deleted residue, `_inserted` indices at or after the restore position were not shifted +1. Subsequent edits could corrupt insertion tracking. Now fixed.
- [x] **Bug fix: deletion revert removed from `_deleted` by array index instead of identity** — used `splice(di, 1)` where `di` was captured at render time, potentially stale if multiple reverts happened between renders. Now uses `splice(indexOf(d), 1)` and removes from `_deleted` *before* computing restore position so `origPosToLocalIdx` sees the correct state.
- [x] **Bug fix: sequence editor `editorOrigPos` ignored `_deleted`** — position labels in the sequence editor only accounted for insertions, not deletions. When a segment had prior deletions, displayed position numbers were off by the number of preceding deletions. Now mirrors `localIdxToOrigPos` logic exactly (walk sorted `_deleted` positions and increment past each one). Data integrity was not affected (downstream processing used the correct function), but displayed positions could mislead users into editing the wrong residue via range inputs.

### Sessions 1 & 2

#### Parts Library (54 validated entries)
- [x] Signals: 22 entries (12 mammalian, 5 bacterial, 2 insect, 3 yeast)
- [x] Linkers: 11 entries (6 repeatable + 5 cleavable)
- [x] Conjugation: 8 entries (3 WT + 4 engineered ⚡ + 1 synthetic ⚡)
- [x] Tags: 13 entries (5 purification + 4 detection + 4 solubility)
- [x] All sequences cross-validated against UniProt / primary literature
- [x] ⚡ icons with instant tooltips for engineered sequences

#### Edit Tracking System (Bulletproof)
- [x] `localIdxToOrigPos` / `origPosToLocalIdx` — account for `_inserted` AND `_deleted`
- [x] 7 rules enforced across ALL edit paths (26 tests × 13 runs = 338/338 pass)
- [x] Ground truth never modified — mutations are overlay-only
- [x] Insertions shift existing `_inserted` indices (all 7 sites fixed)
- [x] All virtual segments include `_deleted`
- [x] Zero naive position patterns remaining
- [x] Multi-segment region edits (Subcellular Location / Zone 3) fully working

#### UI / UX
- [x] Residue hover tooltip (AA + position) in Zone 1, expanded cards, and non-domain parts
- [x] Domain highlighting on hover in expanded protein cards
- [x] Protein colors distinct from engineering colors
- [x] 70/30 layout split (Construct Assembly / Engineering)
- [x] Expanded card padding optimized (recovered 70px width)
- [x] Duplicate end-number suppression in sequence editor
- [x] Contextual card hints ("Click residues to edit" / "Click to expand · N aa")
- [x] Domain toggle preserves scroll position
- [x] Instant ⚡ tooltips (no browser delay)

#### Sequence Corrections Applied
- [x] Twin-Strep: 24→28 aa (missing GGSA in linker)
- [x] V5 tag: 12→14 aa (missing terminal ST)
- [x] GB1: MQ→T at position 2, removed artificial M (55 aa)
- [x] TEV site: ENLYFQS→ENLYFQG
- [x] HRV 3C: fixed cleavage description
- [x] CD33 signal: added initiator M
- [x] tPA signal: 23→22 aa (removed extra S)
- [x] HSA signal: 16→18 aa (added missing LF)
- [x] Influenza HA: V→A at position 14 (correct UniProt strain)
- [x] Removed: 2A peptides (4), hinge regions (2), calmodulin-binding tag (1)
- [x] Added: GST (218 aa), MBP (370 aa), 7 new signal peptides

---

## Architecture Notes

**Current stack:** Single `index.html` (~1550 lines) + `data/parts_library.json` (58 entries)
**Hosting:** GitHub Pages at `lorago92.github.io/construct-designer/`
**Dependencies:** None (vanilla JS, no framework, no build step)
**Network:** UniProt REST API (live), AlphaFold API (live)

**When to consider splitting:**
- If file exceeds ~2500 lines → split into modules
- If adding backend features (save/share) → consider Supabase or Firebase
- If adding DNA tools → may need a Web Worker for heavy computation

---

_SeqMind — Protein Intelligence Platform_
_© 2026_
