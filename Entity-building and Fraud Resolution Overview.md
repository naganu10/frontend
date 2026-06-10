# Entity Building, Entity Resolution & Fraud Detection — Overview


---

## 1. Why this document

The pre-investigation pipeline reads every document attached to a claim, pulls out the "things" mentioned inside them (people, vehicles, phone numbers, workshops, identifiers, etc.), connects those things to the same things seen in *previous* claims, and uses that connected picture to decide how likely the current claim is to be fraudulent.

This document explains, in plain terms:

1. How we **build entities** out of the raw extracted documents.
2. How we **resolve entities** — i.e. how we recognise that "S. A. Asher" in today's claim is the same person as "Smit Atul Asher" stored from a previous claim.
3. How the **knowledge graph** is built and how we **compare a new claim against history** to detect fraud rings.
4. How the **graph score** is computed and how it folds into the final fraud score.

The intent is to give a complete walk-through without diving into Cypher or Python code.

---

## 2. The big picture in one paragraph

For every claim, the pipeline first extracts structured fields and free-text "facts" from each document. From those, it identifies real-world objects (a person, a phone, a vehicle, a workshop, a location, an identifier — Aadhaar/PAN/policy no./FIR no./etc.) and tags each with a **role** (claimant, insured, third-party, driver, witness, deceased, hospital, etc.). These tagged objects are pushed into a **Neo4j knowledge graph** that already holds every entity from every past claim. The graph automatically links the new claim to any past claim that shares an entity. A scoring step then asks: "How many past claims share entities with this one? How strong are those links? Is the shared entity a 'hub' that connects many unrelated claims (a hallmark of a fraud ring)?" The answer becomes the **graph score (0–100)**, which is fused with three other signals — rules, similarity, contradictions — to produce the final **fraud score (0–100)** and risk category (NO_RISK / LOW / MEDIUM / HIGH).

---

## 3. What we mean by "an entity"

An entity is a real-world object that may appear in one or more documents and may also appear in past claims. The current system recognises these entity types:

| Type | Examples |
|---|---|
| **PERSON** | Claimant, insured, driver, deceased, witness, doctor, police IO, advocate, employee |
| **VEHICLE** | Claimant's vehicle, third-party vehicle, by registration number |
| **PHONE** | Mobile / landline / contact numbers |
| **EMAIL** | Email addresses |
| **IDENTIFIER** | Aadhaar, PAN, GSTIN, policy number, claim number, FIR number, driving licence, chassis/engine number, bank account, IFSC, RC number |
| **ORGANIZATION** | Hospital, employer, police station, insurer, court, issuing authority |
| **WORKSHOP** | Garages / repair shops (a sub-type of organisation, tracked separately because workshops drive most repair-shop fraud rings) |
| **LOCATION** | Accident site, incident site, addresses |
| **DATE** | Accident date, incident date, DOB, policy start/end dates |
| **AMOUNT** | Claim amount, estimate, total, premium |
| **GENERIC_IDENTIFIER** | Catch-all for any structured-looking value that doesn't fit one of the above (so we don't silently drop signal) |

Every entity also carries a **role** — what part it plays in *this* claim. The role matters a lot for fraud detection: the same vehicle registration appearing twice in one claim, once as `claimant_vehicle` and once as `third_party_vehicle`, is a real signal. Two different vehicles appearing — both as `claimant_vehicle` — is also a signal. Without roles, both situations would look the same.

---

## 4. Stage 1 — Where the entities come from

Entities are not extracted by a single dedicated pass. They are *built up* from two parallel sources that the document-processing stage has already produced:

### 4.1 The "facts" source

When each document is processed by the LLM, alongside the structured fields the LLM also emits a list of free-text **facts**, e.g. *"The insured Mr. Suresh Kumar (PAN ABCDE1234F) was driving MH-12-AB-3456 on 15 Jan 2025…"* Each fact carries an `entities[]` array with the entities the LLM identified inside that sentence (PERSON: "Suresh Kumar", IDENTIFIER/pan: "ABCDE1234F", VEHICLE: "MH-12-AB-3456", DATE: "2025-01-15").

These are pulled in directly. Every such entity is tagged with `source = "fact"` and back-linked to the originating document and fact row so an investigator can trace it.

### 4.2 The "fields" source

In parallel, every document has its declared **Fields Spec** — a per-document-type list of canonical fields (e.g. an FIR has `complainant_name`, `accused_name`, `vehicle_number`, `fir_number`, `incident_date`, `police_station_name`, etc.). A generic walker (the **field walker**) iterates the spec and pulls each populated value out as an entity. Each canonical key maps to an entity type and a role via a lookup table:

- `claimant_name` → PERSON / claimant
- `tp_vehicle_number` → VEHICLE / third_party_vehicle
- `aadhaar_number` → IDENTIFIER / aadhaar
- `workshop_name` → WORKSHOP
- `accident_location` → LOCATION / accident_site
- `hospital_name` → ORGANIZATION / hospital
- …and so on across ~150 canonical keys.

Anything not in the lookup table can still be admitted as a `GENERIC_IDENTIFIER` if the value "looks structured" (alphanumeric, length ≥ 5, contains a digit). This catches BAGIC-specific IDs we haven't pre-mapped without admitting free-text noise.

Entities from this path are tagged with `source = "field"` and a back-link to the originating field row.

### 4.3 Why two sources?

The two sources catch different things and act as a safety net for each other:

- **Fields are deterministic** — if the document type has `aadhaar_number` declared and the value is populated, we will *always* get that entity, regardless of LLM mood.
- **Facts are flexible** — the LLM can surface entities mentioned in narrative paragraphs that aren't covered by any structured field (e.g. *"the local mechanic Ravi at Shree Auto Works took the vehicle for repair"* — Ravi and Shree Auto Works might not be in the Fields Spec for that doc, but the LLM picks them up).

After both lists are gathered, they are merged.

---

## 5. Stage 2 — Cleaning and merging within the same claim

Before pushing anything to the graph, the system does several clean-up passes on the merged entity list — all within the boundary of the *current* claim:

### 5.1 Type normalisation (catch typos)

The fact-extraction LLM occasionally emits typo'd entity types like `VEHCILE` or `PERSN`. A small edit-distance check coerces these to the closest canonical type (`VEHICLE`, `PERSON`). Anything too far off becomes `GENERIC_IDENTIFIER` so we don't pollute the graph with one-off mis-spelled labels that would silently break cross-claim matching.

### 5.2 Value normalisation (collapse format variants)

Each entity value is passed through a per-type normaliser so that the same real-world object looks the same regardless of how it was written:

- **PHONE** — strip non-digits, drop country code: `+91 98765-43210` → `9876543210`
- **VEHICLE** — strip spaces and dashes, uppercase: `MH 12 AB 3456` and `mh-12-ab-3456` both → `MH12AB3456`
- **PERSON** — Title Case, collapse honorifics (`Mr.`, `Dr.`, etc.)
- **AMOUNT** — strip currency symbols and Indian thousand separators: `Rs. 1,25,000` → `125000`
- **DATE** — normalise to ISO `YYYY-MM-DD`
- **EMAIL** — lowercase, trim
- **LOCATION** — collapse whitespace, expand abbreviations (`Rd` → `Road`)
- **IDENTIFIER** — dispatched by sub-role: Aadhaar is normalised differently from PAN, which is normalised differently from GSTIN, etc. This stops a 10-digit policy number and a 10-digit bank account from collapsing onto the same node.

### 5.3 Name initial-expansion pre-pass

Within the same claim, a person can appear as both `Smit Atul Asher` (in the Aadhaar/Policy doc) and `S. A. Asher` (in the FIR). A deterministic pre-pass groups PERSON entities by their initials (`SAA`) and collapses each group to the longest form. This is "deterministic" because it does not depend on the LLM — even if the LLM is unavailable, this fix-up still runs.

### 5.4 Deduplication with role merge

Finally, entities are de-duplicated on `(type, normalised value)`. Critically, when the same `(type, value)` pair appears with different roles — e.g. a person who is both `claimant` and `driver` — we **keep all the distinct roles** in a `roles` list rather than discarding all but the first. The graph layer uses this list to detect cross-claim links where the same person plays different roles per claim.

When the same person/object appears once from the facts source and once from the fields source, the **field-sourced** version wins as the canonical record because the Fields Spec gives a more reliable role than the LLM's free-text guess.

---

## 6. Stage 3 — Resolution against the existing knowledge graph

At this point we have a clean list of entities for the current claim. Now we need to decide, for each one: **is this an entity we already know about from a past claim, or is it new?**

This is "entity resolution" and it happens in two layers.

### 6.1 Layer A — LLM soft-match (variant resolution)

For each candidate, we look up the existing Neo4j graph for any node of the same type with a similar value, then send a short comparison to the LLM with a strict matching policy. The LLM is asked to decide, per pair, whether the candidate is canonically the same as an existing graph node. Example:

- Candidate: `PERSON / "S. A. Asher"`
- Existing graph node: `PERSON / "Smit Atul Asher"`
- → LLM returns `alias_of = "Smit Atul Asher"` ⇒ we use the existing node.

To stop the LLM from inventing a non-existent canonical node, two hard guards apply:

- **Membership guard:** the `alias_of` value must already exist in the graph-nodes list we showed the LLM.
- **Type guard:** the candidate's type and the matched node's type must be identical (a PHONE cannot be aliased to a PERSON).

If the LLM call fails twice (parse error, timeout, etc.), the system enters **degraded mode** — it skips LLM resolution and only uses the deterministic Cypher match below. The final fraud-score report carries a flag so reviewers know graph signal was partially degraded for this claim.

### 6.2 Layer B — Cypher exact-match upsert (atomic)

After any LLM-suggested aliases are applied, we run a single-roundtrip MERGE in Neo4j for each entity:

> *"Find a node of this type with this exact value. If it exists, return its id. If it doesn't, create it with a fresh UUID and return that id."*

This is atomic — two parallel claim processes cannot accidentally create two duplicate nodes for the same value.

For PERSON specifically, the upsert has an additional pre-step: if the candidate carries initials (e.g. `SAA`) and shares overlapping words with exactly one existing PERSON node with the same initials, that existing node is reused. This catches `S. A. Asher` → `Smit Atul Asher` even without the LLM running.

### 6.3 Two edges per match

Once an entity node is resolved (matched or freshly created), two relationships are written from the current Claim node:

1. **A legacy typed edge** — e.g. `(Claim)-[:CLAIMANT]->(PERSON)`, `(Claim)-[:HAS_PHONE]->(PHONE)`, `(Claim)-[:INVOLVES_VEHICLE]->(VEHICLE)`, `(Claim)-[:REPAIRED_AT]->(WORKSHOP)`, `(Claim)-[:OCCURRED_AT]->(LOCATION)`. These were the original edges and existing fraud-ring queries depend on them.
2. **A generic `HAS_ENTITY` edge** carrying the full `roles` list as a property — e.g. `(Claim)-[:HAS_ENTITY {roles: ["claimant", "driver"]}]->(PERSON)`. This is additive and is what allows newer queries to find cross-claim links where the same entity plays *different* roles in different claims.

---

## 7. Stage 4 — Comparing the current claim against past claims (the graph)

This is the heart of fraud detection by entity. Once the Stage-3 writes are done, the graph automatically connects the current claim to any past claim that shares an entity. We then run three separate analyses on this connected structure.

### 7.1 Connected claims (direct neighbourhood)

> *"Starting from the current Claim node, walk outward up to 4 hops along the meaningful edge types and collect every other Claim node we can reach. For each, record the distance (1–4 hops) and the chain of edge types we traversed."*

Edge types walked: `CLAIMANT`, `HAS_PHONE`, `INVOLVES_VEHICLE`, `INVOLVES_PERSON`, `REPAIRED_AT`, `OCCURRED_AT`, `HAS_ENTITY`.

The result is a list of past claims connected to this one, each labelled with how many hops away it is and which entity types form the chain. For example: *"Claim-A → CLAIMANT → Person-X → CLAIMANT → Claim-B"* tells us Claim A and Claim B share a claimant (1 entity between them, 2 hops).

### 7.2 Ring detection

> *"Is the current claim part of a cluster of two or more interconnected claims?"*

If the connected-claims walk found two or more past claims, the current claim is considered to be in a **fraud ring** (the term is used loosely — it just means a connected cluster; whether it's actually fraudulent is what the score is for).

For each ring, we also count how many of the ring members already have `fraud_confirmed = true` set on them from prior investigations. The ratio (confirmed-fraud claims ÷ total ring size) becomes the **fraud rate** — the strongest single signal in the graph layer. A new claim joining a ring that's 80% confirmed-fraud is far more concerning than one joining a ring that's 0% confirmed.

### 7.3 Hub-entity / shared-entity detection

> *"Are there any single entities — a phone, a workshop, an address — that link the current claim to many unrelated claims?"*

Two specific patterns are flagged here:

- **Shared phone** — if the same phone number is found on more than 2 claimants across the graph, that phone is treated as a suspicious hub.
- **High-volume workshop** — if the same workshop has handled more than 10 claims across the graph, it's flagged as a hub (typical of repair-shop-driven inflation rings).

These hub entities are returned alongside the ring, both as raw data and as a short LLM-generated narrative ("Phone 9876543210 is shared across 6 claimants spanning 3 different vehicles — strong indicator of an organised filer").

---

## 8. Stage 5 — Computing the graph score (0–100)

The graph score collapses everything from Stage 4 into a single number between 0 and 100. It is the contribution of *historical-comparison signal* to the final fraud score.

The score is built additively from four components:

| Component | What it represents | Maximum contribution |
|---|---|---|
| **Ring membership** | The current claim is part of a connected cluster of 2 or more past claims | **+50** |
| **Connected claims (per-link)** | For each past claim reachable in 1–4 hops, add a base value that decays with distance (15 at 1 hop, 10 at 2 hops, 5 at 3 hops, 0 at 4 hops), **multiplied by the weakest edge strength along that path** (see below) | Uncapped, but distance decay limits it in practice |
| **Connection count** | Raw count of suspicious connections × 4 | **+20** |
| **Shared / hub entities** | Distinct hub entities × 5 | **+15** |

The total is then capped at 100.

### 8.1 Why edge-strength weighting matters

Not every shared link is equal evidence of fraud. A shared **phone number** is almost always meaningful — different claims sharing a contact number is suspicious. A shared **workshop**, on the other hand, is usually coincidental — popular workshops handle hundreds of unrelated claims a year. Without weighting, a 10-claim "ring" linked only by a common workshop would score the same as a 10-claim ring linked by a shared claimant phone — which would over-flag legitimate workshops and dilute the signal.

The current weights are:

| Edge type | Strength | Reasoning |
|---|---:|---|
| `CLAIMANT` | 1.0 | Same person filing multiple claims — strongest signal |
| `HAS_PHONE` | 1.0 | Same phone across claimants — almost always meaningful |
| `INVOLVES_VEHICLE` | 0.9 | Same vehicle — meaningful but vehicles do change owners |
| `INVOLVES_PERSON` | 0.9 | Same person involved (driver, witness, etc.) |
| `HAS_ENTITY` | 0.5 | Generic role-aware edge (catch-all) |
| `OCCURRED_AT` | 0.4 | Shared location — many claims happen in the same area |
| `REPAIRED_AT` | 0.3 | Shared workshop — high baseline traffic |

A path's strength is the **weakest edge along it** (multiplicative bottleneck), not the average. A two-hop chain `claimant → workshop → claimant` is bottlenecked by the workshop and contributes weakly, even though both endpoints are claimants.

### 8.2 De-duplication of hub entities

The same person playing two roles in the same claim (e.g. claimant *and* driver) would otherwise produce two edges and inflate the shared-entity count. Hub entities are therefore de-duplicated by `id` (or by `value` when no id) before counting.

### 8.3 Worked example

Suppose the current claim:

- Belongs to a ring of 3 past claims → **+50** (ring membership).
- Is connected to one past claim 1 hop away via a shared phone (`HAS_PHONE` strength 1.0): `15 × 1.0 = 15` → **+15**.
- Is connected to a second past claim 2 hops away via a shared workshop (`REPAIRED_AT` strength 0.3): `10 × 0.3 = 3` → **+3**.
- Has 2 suspicious connections total: `2 × 4 = 8` → **+8**.
- One hub entity (the shared phone): `1 × 5 = 5` → **+5**.

Total graph score: **50 + 15 + 3 + 8 + 5 = 81**.

---

## 9. How the graph score becomes the fraud verdict

The graph score is one of four signals. They are fused with configurable weights to produce the final fraud score:

| Signal | Default weight | What it measures |
|---|---:|---|
| **Rule score** | 30% | Business-rule triggers (CSV rules + system rules — e.g. policy-inception-vs-incident-date, claim-amount-vs-IDV, weather mismatches) |
| **Graph score** | 25% | Historical comparison via the knowledge graph (this document) |
| **Similarity score** | 20% | Vector-similarity search against past claim descriptions, with extra weight on past claims confirmed as fraud |
| **Consistency score** | 15% | Contradictions across the documents of the current claim (e.g. date mismatch between FIR and claim form) |
| **ML / LLM holistic score** | 10% | Overall LLM judgement of the claim |

Two extra rules apply on top of the weighted average:

- If **any single signal is above 80**, the final score is lifted to at least `85% × that signal` — so a single very strong signal can't be diluted by weak ones.
- If **three or more signals are above 60**, the final score is multiplied by 1.15 — multiple corroborating signals compound.

The final score is mapped to a risk category and a recommended action:

| Final score | Risk category | Recommended action (OD claims) |
|---|---|---|
| 0–24 | NO_RISK | WAIVER (or DESKTOP for high-value claims) |
| 25–44 | LOW | WAIVER (or DESKTOP for high-value claims) |
| 45–69 | MEDIUM | DESKTOP (or FIELD for high-value claims) |
| 70–100 | HIGH | FIELD |

All TP (Third-Party) claims default to FIELD regardless of score.

The thresholds and weights are stored in configuration so the finance / SIU teams can re-tune them without a code release.

---

## 10. What this gives the business

In effect, every new claim is automatically scored against the entire history of past claims along three axes:

1. **Is this claim's claimant / driver / vehicle / phone / workshop / etc. someone we've seen before?** (Entity resolution + connected-claims walk.)
2. **Is the current claim part of a wider cluster of related claims, and are any of those clusters already known to be fraudulent?** (Ring detection + fraud-rate.)
3. **Is any single entity acting as a "hub" linking many unrelated claims together?** (Hub-entity detection — typically phones and workshops.)

The result is a single number (0–100) and a risk category that determines whether the claim can be settled directly, sent for desktop investigation, or escalated to a field investigator — with every signal traceable back to the specific document, field, fact, and graph node that produced it.

---

## 11. Glossary

- **Entity** — A real-world object (person, vehicle, phone, identifier, etc.) extracted from one or more documents.
- **Role** — What part an entity plays in the claim (claimant, insured, third-party, driver, witness, etc.).
- **Entity resolution** — Recognising that two differently-written values refer to the same real-world object (e.g. `S. A. Asher` ↔ `Smit Atul Asher`).
- **Knowledge graph** — A Neo4j database of every claim, entity, and relationship between them.
- **Ring** — A connected cluster of two or more claims that share entities.
- **Fraud rate** — Within a ring, the share of claims already confirmed as fraudulent in prior investigations.
- **Hub entity** — A single entity (typically a phone or workshop) that connects an unusually large number of otherwise-unrelated claims.
- **Graph score** — A 0–100 score representing the strength of historical-comparison signal for this claim.
- **Fraud score** — The final 0–100 score combining graph score with rule, similarity, consistency, and ML signals.
- **Edge strength** — A weight (0–1) on each relationship type capturing how strong a fraud signal that link is. The weakest edge along a path determines that path's overall strength.
