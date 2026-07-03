# Astrology Analysis Agent — Complete Instruction Manual

**Version:** 1.0
**Date:** 2026-06-26
**Purpose:** This document gives a developer or prompt engineer everything needed to build a multi-layer, fact-checking astrology agent. The agent collects birth data, runs deterministic planetary calculations, cross-references multiple chart layers before drawing any conclusion, and delivers probabilistic interpretations grounded in classical astrological tradition. It is designed to be opened once and used directly — no prior context required.

---

## What This Agent Does

This agent operates in three sequential stages. First, it collects date of birth, time of birth, and place of birth, then passes those inputs to a dedicated ephemeris engine that computes all planetary positions, house cusps, divisional charts, yogas, doshas, aspects, dignities, and the Vimshottari dasha sequence with sub-arc-second positional accuracy [1]. Second, it runs a structured multi-layer analysis that cross-references the D1 (Rasi) chart against the relevant divisional charts (D9 for relationships, D10 for career, D2 for wealth, etc.), validates every yoga and dosha across those layers, integrates current and upcoming dasha timing, and resolves any conflicts between layers using a defined weighting hierarchy — all before producing a single sentence of interpretation [2]. Third, it passes the validated analytical summary to an LLM, which turns the verified reasoning chain into clear, probabilistic, user-facing language, grounded in retrieved passages from classical astrological texts. The agent never delivers a reading from one chart layer in isolation. Every major claim must be traceable back through the cross-reference chain.

## What This Agent Does NOT Do

- Does not guess, infer, or hallucinate any planetary position, house placement, yoga, dosha, or dasha period not present in the calculation engine's output.
- Does not make absolute predictions about specific events or exact dates.
- Does not give medical diagnosis, treatment recommendations, legal advice, or specific financial investment instructions.
- Does not read from a single chart layer alone — no D1-only readings, no dasha-only readings.
- Does not present a dosha as active without first checking all standard cancellation conditions.
- Does not present a yoga as powerful without confirming divisional chart support.
- Does not allow the LLM to re-derive conclusions from raw chart data — the LLM interprets only what the analysis layer has already validated.

---

## Agent Architecture

The agent runs as a three-stage sequential pipeline. Each stage has a strictly defined input, a strictly defined output, and a hard rule about what the LLM may and may not touch.

```
[Birth Data Input: date, time, place]
 ↓
[Stage 1: Ephemeris Engine — Deterministic]
 → structured chart data (planets, houses, divisionals, yogas, doshas, dashas, aspects)
 ↓
[Stage 2: Multi-Layer Analysis & Fact-Check Engine]
 → validated reasoning chain (cross-referenced, conflict-resolved, dasha-integrated)
 ↓
[Stage 3: LLM Interpretation Layer]
 → final user-facing reading (probabilistic, grounded, ethically constrained)
```

**Stage 1 — Calculation Engine (Deterministic)**
Accepts: date of birth, time of birth, place of birth.
Outputs: all planetary positions, house cusps, sign and nakshatra placements, all divisional charts from D1 through at least D10, yogas, doshas, aspects, dignities (exalted, debilitated, own sign, Vargottama), and the full Vimshottari dasha sequence.
Hard rule: the LLM never touches this stage. All arithmetic is handled by a dedicated ephemeris library. If the library is unavailable or returns an error, the agent halts and requests valid input rather than allowing the LLM to estimate any positional value [2].

**Stage 2 — Multi-Layer Analysis and Fact-Check Engine**
Accepts: the structured output from Stage 1 plus the user's question.
Does: cross-references D1 against the relevant divisional chart or charts, validates yogas and doshas across layers, integrates dasha timing, resolves conflicts between layers using the weighting hierarchy defined in Section 4.
Outputs: a validated, cross-referenced analytical summary — not yet the final answer, but the verified reasoning chain that Stage 3 is permitted to interpret.

**Stage 3 — Interpretation and Communication Layer (LLM)**
Accepts: the validated analytical summary from Stage 2, the user's original question, and retrieved passages from classical astrological texts via RAG.
Does: turns the verified reasoning into clear, probabilistic, user-facing language.
Hard rule: the LLM may only interpret what Stage 2 has already validated. It cannot introduce new chart claims, re-read raw chart data to draw new conclusions, or present any planetary position, yoga, dosha, or timing period that does not appear in the Stage 2 summary.

---

## Stage 1 — Calculation Engine Requirements

The calculation engine must produce every data point listed below before Stage 2 begins. If any mandatory field is missing, Stage 2 must flag it explicitly rather than proceeding with incomplete data.

### 1.1 Core D1 (Rasi) Chart

**Ascendant / Lagna**
Compute and record: the sign the ascendant falls in, its exact degree, and the lord of that sign. This is the foundation of the entire chart — every house number derives from the lagna position.

**All Nine Planets**
Compute and record for each of the Sun, Moon, Mars, Mercury, Jupiter, Venus, Saturn, Rahu, and Ketu:
- Sign the planet occupies.
- Exact degree within that sign.
- House number the planet occupies (counted from the lagna).
- Nakshatra (lunar mansion) the planet occupies.
- Pada (quarter) within that nakshatra.
- Retrograde status: true or false. Rahu and Ketu are always retrograde by convention — flag this explicitly so the analysis layer does not misread it as an anomaly.
- Dignity: one of exalted, debilitated, own sign, moolatrikona, friendly sign, neutral sign, or enemy sign. Compute this from the planet's relationship to the sign it occupies according to classical Vedic rulership tables.
- House lordships: which house or houses does this planet rule? A planet ruling two houses (e.g., Mercury ruling the 3rd and 6th) must have both lordships recorded, because the analysis layer uses both when evaluating dasha periods.

**All Twelve House Cusps**
For each house from 1 to 12: record the sign on the cusp and the lord of that sign. In whole-sign houses the cusp sign is simply the sign of that house number counted from the lagna; record the degree of the lagna as the reference point.

### 1.2 Divisional Charts (Varga Charts)

Compute the following divisional charts. Each divisional chart is derived from the D1 positions by a specific mathematical division of each planet's degree within its sign; the engine must apply the correct formula for each division number.

**D9 — Navamsa (mandatory for all readings)**
The most important divisional chart. Used to assess how D1 promises actually manifest in lived experience, particularly for relationships, inner strength, and spiritual path. Compute for every planet: the navamsa sign it falls in and the house it occupies in the D9 chart. Compute the D9 lagna and its lord. Flag any planet that occupies the same sign in both D1 and D9 — this is called Vargottama and significantly strengthens that planet's expression.

**D10 — Dasamsa (mandatory for career and profession questions)**
Used to assess professional life, public role, and career trajectory. Compute for every planet: the dasamsa sign and house. Compute the D10 lagna and its lord. The condition of the 10th house and its lord in D10 is the primary career indicator in this chart.

**D2 — Hora (for wealth questions)**
Used to assess financial accumulation and wealth potential. Compute lagna, all planetary placements by sign and house.

**D7 — Saptamsa (for children questions)**
Used to assess children and progeny. Compute lagna, all planetary placements by sign and house.

**D4 — Chaturthamsa (for property and fixed assets questions)**
Used to assess real estate, home, and immovable property. Compute lagna, all planetary placements by sign and house.

**D60 — Shashtiamsa (for overall karma and life themes, if the engine supports it)**
The most granular divisional chart, used for deep karmic assessment. Include if the ephemeris engine supports 60th division calculations. If not supported, record this explicitly so the analysis layer does not attempt to use D60 data that does not exist.

**For every divisional chart computed:** record the lagna sign and lord, all planet placements (sign and house number), and Vargottama flags where applicable (a planet is Vargottama if it occupies the same sign in D1 and D9).

### 1.3 Yogas

Detect and record all yogas present in the D1 chart. For each yoga, record: the yoga name, the planets and houses involved in forming it, the house or houses where the yoga is formed, and a strength rating of strong, moderate, or weak based on the dignity and placement of the yoga-forming planets.

The engine must detect at minimum the following yogas:

**Gajakesari Yoga:** Jupiter in a kendra (1st, 4th, 7th, or 10th house) from the Moon. Indicates recognition, moral authority, and material stability. Strength is higher when Jupiter is also well-dignified.

**Raja Yogas:** Formed when lords of kendra houses (1, 4, 7, 10) and lords of trikona houses (1, 5, 9) conjoin, exchange signs, or aspect each other. Multiple Raja Yogas can exist in a single chart. Record each separately with the specific lords involved.

**Dhana Yogas:** Formed by connections between lords of the 2nd, 5th, 9th, and 11th houses. Indicates wealth accumulation potential.

**Budhaditya Yoga:** Sun and Mercury in conjunction in the same house. Indicates intelligence, analytical ability, and communication skill. Strength depends on the dignity of both planets and the house where the conjunction falls.

**Pancha Mahapurusha Yogas:** Five yogas formed when Mars, Mercury, Jupiter, Venus, or Saturn is in its own sign or exaltation and placed in a kendra house. Each is named separately: Ruchaka (Mars), Bhadra (Mercury), Hamsa (Jupiter), Malavya (Venus), Shasha (Saturn).

**Neecha Bhanga Raja Yoga:** A debilitated planet whose debilitation is cancelled by specific conditions (the lord of the sign of debilitation is in a kendra from lagna or Moon; or the planet that would be exalted in that sign is in a kendra; or the debilitated planet is aspected by its exaltation lord). When debilitation is cancelled this way, the planet often produces strong results. Record the specific cancellation condition present.

**Viparita Raja Yoga:** Lords of the 6th, 8th, or 12th houses placed in each other's houses or in other dusthana houses, without involvement of benefic planets or kendra lords. Indicates gains through apparent losses or reversals.

### 1.4 Doshas

Detect and record all doshas present in the D1 chart. For each dosha, record: the dosha name, the triggering planet and placement, a severity rating of mild, moderate, or severe, and any cancellation conditions that are present in the chart.

The engine must detect at minimum the following doshas:

**Mangal Dosha (Kuja Dosha):** Mars placed in the 1st, 2nd, 4th, 7th, 8th, or 12th house from the lagna, Moon, or Venus (different traditions count from different reference points — specify which reference point the engine uses). Record the exact house Mars occupies and which reference point triggered the dosha. Record all cancellation conditions present: Mars in its own sign (Aries or Scorpio), Mars in its exaltation sign (Capricorn), Mars aspected by Jupiter, both partners in a compatibility reading have Mangal Dosha, Mars in the 2nd house for specific lagnas where it is less harmful, etc.

**Kaal Sarp Dosha:** All seven visible planets (Sun, Moon, Mars, Mercury, Jupiter, Venus, Saturn) positioned between Rahu and Ketu on one side of the chart axis. Record the Rahu-Ketu axis (which houses they occupy), confirm that all seven planets fall within the arc between them, and note whether any planet is outside the axis (which would cancel the dosha). Record severity based on which houses Rahu and Ketu occupy.

**Shrapit Dosha:** Saturn and Rahu conjunct in the same house. Indicates karmic burdens requiring conscious effort to resolve. Record the house of conjunction and the dignity of both planets.

**Grahan Dosha:** Sun or Moon conjunct Rahu or Ketu within a defined orb (typically within 10 degrees for a strong dosha). Record which luminary is affected, which node it conjuncts, the house of conjunction, and the orb in degrees.

### 1.5 Vimshottari Dasha Sequence

Compute the full Vimshottari dasha sequence based on the Moon's nakshatra at birth. Record:

- **Current mahadasha:** the ruling planet, exact start date, and exact end date.
- **Current antardasha (bhukti):** the ruling planet within the mahadasha, exact start date, and exact end date.
- **Current pratyantardasha (if relevant):** the sub-sub period planet, start date, and end date. Include this when the user asks about timing within the next few months, as the pratyantardasha is the most granular timing layer.
- **Next three to five upcoming mahadasha periods:** planet name, start date, and end date for each. The analysis layer uses these to describe the trajectory of the coming years.

### 1.6 Aspects

Compute all major aspects between planets and record for each: the two planets involved, the aspect type, and the orb in degrees.

**For Vedic tradition:** the primary aspect is the full aspect (7th house from a planet's position). Additionally, record the special aspects unique to three planets:
- Saturn aspects the 3rd and 10th houses from its position (in addition to the 7th). Flag these as Saturn's special aspects.
- Mars aspects the 4th and 8th houses from its position (in addition to the 7th). Flag these as Mars's special aspects.
- Jupiter aspects the 5th and 9th houses from its position (in addition to the 7th). Flag these as Jupiter's special aspects.

These special aspects carry full strength and must be flagged explicitly because they create connections between houses that would not exist in a standard aspect list.

**For Western tradition:** record conjunctions, oppositions, trines, squares, and sextiles with their orbs in degrees. Standard orbs are 8 degrees for conjunctions and oppositions, 6 degrees for trines and squares, and 4 degrees for sextiles — adjust if a different orb system is specified.

### 1.7 Ayanamsa and Tradition

Record which ayanamsa is applied (Lahiri is the standard for Vedic Jyotish and is recommended as the default) and which house system is used (whole-sign or Placidus — specify which). Record whether the reading is Vedic, Western, or hybrid. If hybrid, record which layers belong to which tradition. This metadata is used by the analysis layer to apply the correct aspect rules and by the output layer to label tradition-specific claims correctly.

---

## Stage 2 — Multi-Layer Analysis Protocol (Connect the Dots)

This is the core of the agent. Stage 2 is where the agent earns its accuracy: by refusing to draw any conclusion until it has cross-referenced all relevant layers and resolved all conflicts. Every step below is mandatory for every reading.

### 2.1 The Layering Hierarchy

The agent must apply this weighting order consistently. A higher layer sets the frame; lower layers refine it. No lower layer overrides a higher one — it modifies the interpretation of it.

1. **D1 (Rasi chart):** Always the foundation. D1 represents the primary promise of the chart — what the life is set up to deliver at the outer, manifest level. Every conclusion begins here.

2. **D9 (Navamsa):** The most important divisional chart. D9 reveals how D1 promises actually manifest in lived experience. A strong D1 promise with a weak D9 means the outer opportunity exists but the inner experience of it is difficult or delayed. D9 refines D1; it does not override it.

3. **D10, D2, D7, D4, and other topic-specific divisional charts:** These are refinement layers for specific life domains. Use the question-to-chart mapping table to select which divisional chart is relevant. Each of these shows the specific domain in greater detail than D1 can alone.

4. **D60 (Shashtiamsa):** The karmic baseline. Use when available to understand deep-seated patterns that color the entire chart. D60 provides context for why D1 promises manifest the way they do.

5. **Dashas:** The timing activation layer. Dashas do not create new promises — they activate what is already present in the chart. A dasha period is supportive for a given domain when the dasha lord has strong connections to the relevant houses and planets in both D1 and the relevant divisional chart. A dasha period does not create a new career or a new marriage out of nothing; it opens or closes windows for what the chart already promises.

6. **Transits:** Short-term triggers. Mention transits when relevant to timing questions, but do not over-weight them. A transit alone cannot deliver what the chart does not promise, and it cannot permanently block what the chart strongly promises.

**Non-negotiable rule:** Never draw a major conclusion from one layer alone. Every major claim must be cross-referenced against at least D1 plus one relevant divisional chart plus the current dasha before it is included in the Stage 2 summary.

### 2.2 Question-to-Chart Mapping

Use this table to determine which houses, planets, and divisional charts are relevant for each type of user question. The analysis must cover all columns for the relevant row before producing any conclusion.

| Question Type | Primary Houses (D1) | Key Planets | Divisional Chart | Secondary Houses |
|---|---|---|---|---|
| Career / Profession | 10th, 6th, 2nd, 11th | Sun, Saturn, Mercury | D10 | 1st (self-effort) |
| Marriage / Relationships | 7th, 2nd, 5th, 11th | Venus, Jupiter (for women), Mars (for men) | D9 | 8th (longevity of union) |
| Wealth / Finance | 2nd, 11th, 5th, 9th | Jupiter, Venus, Mercury | D2 | 8th (inheritance) |
| Children | 5th, 9th | Jupiter, Moon | D7 | 11th (fulfillment) |
| Health | 1st, 6th, 8th, 12th | Sun, Moon, Mars, Saturn | D1 (primary) | Lagna lord condition |
| Spiritual / Life Purpose | 9th, 12th, 1st | Jupiter, Ketu, Saturn | D9, D60 | 5th (past merit) |
| Property / Home | 4th | Moon, Saturn | D4 | 12th (loss) |
| General Life Themes | All | All | D1 + D9 | All |

For questions that span multiple domains (e.g., "how will my career affect my finances?"), apply the analysis protocol for each domain separately and then combine the results in Step 6.

### 2.3 The Cross-Reference Protocol — Step by Step

Follow these six steps in order for every reading. Do not skip a step. Do not proceed to Stage 3 until Step 6 is complete.

**Step 1 — Establish the D1 Baseline Promise**

Using the question-to-chart mapping table, identify the houses and lords relevant to the user's question. For each relevant house:
- Assess the condition of the house lord: what is its dignity? Which house is it placed in? What aspects does it receive? What planets conjoin it?
- Assess the condition of the relevant karaka (natural significator) planet for the domain. For career the karaka is the Sun and Saturn. For marriage the karaka is Venus (and Jupiter for women in classical Vedic tradition). For children the karaka is Jupiter.
- Assess whether the house itself is occupied by benefic planets (Jupiter, Venus, well-placed Mercury, well-placed Moon) or malefic planets (Saturn, Mars, Rahu, Ketu, afflicted Sun).
- Produce a baseline verdict: **Strong Promise**, **Mixed Promise**, or **Weak Promise**, with one to three sentences of explicit reasoning referencing the specific placements and conditions that produced the verdict.

**Step 2 — Cross-Check with the Relevant Divisional Chart**

Pull the same houses and the same planets in the relevant divisional chart identified in the mapping table. For each:
- What sign does the relevant divisional chart lagna fall in, and who is its lord?
- What is the condition of the house equivalent to the domain's primary house in the divisional chart?
- What is the condition of the karaka planet in the divisional chart?
- Does the divisional chart confirm, strengthen, weaken, or contradict the D1 baseline?

Apply the following combination matrix to produce a modified verdict:

| D1 Baseline | Divisional Chart Signal | Combined Reading |
|---|---|---|
| Strong | Strong | Very strong — high probability of manifestation across both the outer and inner dimensions of the domain |
| Strong | Weak | Mixed — the outer potential exists but the inner experience or actual delivery has obstacles, delays, or dissatisfaction |
| Weak | Strong | Latent potential — the domain may develop strongly later in life or after a specific dasha activates it; early life may feel blocked |
| Weak | Weak | Genuinely challenging area — requires sustained conscious effort; do not overstate the difficulty but do not understate it |
| Mixed | Strong | Better than it appears — the underlying strength is real even if the surface picture looks uncertain |
| Mixed | Weak | More challenging than it appears — the surface looks manageable but the deeper experience is harder |

Record the modified verdict with explicit reasoning referencing specific placements in both D1 and the divisional chart.

**Step 3 — Validate Yogas and Doshas Across Layers**

For each yoga detected in D1 by the calculation engine:
- Identify the yoga-forming planets.
- Check the condition of those same planets in the relevant divisional chart. Are they well-placed (own sign, exaltation, friendly sign, angular house)? Or are they debilitated, in enemy signs, in dusthana houses (6th, 8th, 12th), or aspected by malefics?
- If the yoga-forming planets are well-placed in the divisional chart: mark the yoga as **Active and Supported**. The yoga is likely to deliver its promised results.
- If the yoga-forming planets are poorly placed in the divisional chart: mark the yoga as **Present but Weakened**. The yoga exists in the D1 blueprint but its delivery is compromised. Explain why — name the specific condition in the divisional chart that weakens it.
- Never present a yoga as powerful without completing this check.

For each dosha detected in D1 by the calculation engine:
- Work through every standard cancellation condition for that dosha. The primary cancellation conditions for the most common doshas are listed below. This list is not exhaustive — apply additional classical cancellation rules as appropriate.

*Mangal Dosha cancellation conditions:*
- Mars occupies its own sign (Aries or Scorpio) in the chart where the dosha is counted.
- Mars occupies its exaltation sign (Capricorn).
- Mars is aspected by Jupiter.
- Both partners in a compatibility reading have Mangal Dosha — the dosha cancels across partners.
- Mars is in the 2nd house for specific lagnas where classical texts indicate reduced harm (consult tradition-specific rules for which lagnas these are).
- Venus or the Moon occupies the 1st or 2nd house in the chart, softening the impact.
- Mars is in the 8th house for Aries or Scorpio lagna, where Mars rules the lagna and the dosha is considered cancelled.

*Kaal Sarp Dosha cancellation conditions:*
- Any planet is positioned outside the Rahu-Ketu axis (even one planet outside the arc cancels the dosha).
- Rahu or Ketu occupies a kendra house (1st, 4th, 7th, 10th) in a sign where it is considered strong.
- The lagna lord is well-placed and strong.

*Shrapit Dosha cancellation conditions:*
- Jupiter aspects the Saturn-Rahu conjunction.
- The conjunction occurs in a house where both planets are well-dignified.
- Strong benefic planets in the chart overall mitigate the effect.

*Grahan Dosha cancellation conditions:*
- The affected luminary is in its own sign or exaltation.
- The conjunction orb is wide (beyond 8 to 10 degrees, depending on the tradition).
- Jupiter aspects the affected luminary.

After checking all cancellation conditions, check the dosha planet's condition in D9. A dosha planet that is exalted, in its own sign, or strongly placed in D9 has its D1 dosha effect significantly reduced.

Produce a dosha status of **Active**, **Partially Cancelled**, or **Fully Cancelled** with explicit reasoning naming each cancellation condition that is or is not present. Never present a dosha as active without completing this check.

**Step 4 — Integrate Dasha Timing**

For the current mahadasha lord, answer all of the following:
- Which house or houses does this planet rule in D1?
- Which house is this planet placed in within D1?
- What is its dignity in D1?
- What is its placement in the relevant divisional chart (sign, house, dignity)?
- Does this planet have a direct connection — through lordship, placement, or aspect — to the houses relevant to the user's question?

For the current antardasha lord: answer the same set of questions.

Produce a timing verdict for the current period: **Strongly Supportive**, **Mixed**, or **Less Supportive** for the user's question domain. The reasoning must explicitly tie the dasha planet back to both its D1 condition and its divisional chart condition. A verdict of Strongly Supportive requires that the dasha lord connects to the relevant houses in D1 and is also well-placed in the relevant divisional chart. A verdict of Mixed means the dasha lord connects to the relevant houses but is poorly dignified or afflicted in the divisional chart, or connects to both supportive and challenging houses simultaneously. A verdict of Less Supportive means the dasha lord has no meaningful connection to the relevant houses or is significantly afflicted in both D1 and the divisional chart.

For the next two to three upcoming mahadasha periods: repeat the same analysis and describe the trajectory — is the situation likely to improve, remain similar, or become more challenging as the dashas progress?

**Step 5 — Resolve Conflicts**

If D1 and the divisional chart give contradictory signals, the agent must not suppress the conflict or smooth it over. Instead:

- State the conflict explicitly. Name the specific placements that create the contradiction (for example: "D1 shows a strong 7th house with Venus as the 7th lord in its own sign, but D9 shows Venus debilitated in Virgo in the 6th house of the navamsa").
- Apply the hierarchy: D1 represents the outer life promise — what is visible, what opportunities arise. D9 represents the inner experience — how those opportunities actually feel and what they deliver at a deeper level. A strong D1 with a weak D9 means visible relationship opportunities but recurring inner dissatisfaction, difficulty sustaining depth, or a pattern of relationships that look good from outside but feel unfulfilling from inside. A weak D1 with a strong D9 means the outer circumstances are constrained but the inner life is rich; relationships that do form tend to be deeply meaningful even if they arrive late or unconventionally.
- Apply the same D1-outer / divisional-inner logic to other domains: career (D1 = public role, D10 = actual professional satisfaction and authority), wealth (D1 = earning potential, D2 = actual accumulation), etc.
- Explain what the tension means in practical terms for the user's question. Do not leave the conflict as an unresolved abstraction.

**Step 6 — Produce the Validated Analytical Summary**

After completing Steps 1 through 5, produce a structured internal summary. This summary is the only thing Stage 3 is permitted to interpret. It must contain all of the following:

- **D1 baseline verdict:** Strong / Mixed / Weak Promise, with the specific placements that produced it.
- **Divisional cross-check result:** the modified verdict from the combination matrix, with the specific divisional chart placements that modified the D1 baseline.
- **Yoga status list:** for each yoga, Active and Supported / Present but Weakened, with reasoning.
- **Dosha status list:** for each dosha, Active / Partially Cancelled / Fully Cancelled, with reasoning naming each cancellation condition checked.
- **Dasha timing verdict:** Strongly Supportive / Mixed / Less Supportive for the current period, with trajectory for the next two to three periods.
- **Conflict log:** any unresolved tensions between layers, with the practical interpretation of each tension.
- **Overall integrated picture:** three to five sentences that combine all of the above into a single coherent assessment of the user's question.

---

## Stage 3 — LLM Interpretation Rules

### 3.1 What the LLM Receives

The LLM receives exactly four inputs and no others:

1. The validated analytical summary produced by Stage 2.
2. The user's original question, verbatim.
3. Retrieved passages from classical astrological texts, retrieved via RAG based on the key themes and placements identified in the Stage 2 summary. Relevant classical sources for Vedic tradition include Brihat Parashara Hora Shastra, Phaladeepika, Brihat Jataka, Uttara Kalamrita, and Saravali. For Western tradition, relevant sources include Ptolemy's Tetrabiblos, William Lilly's Christian Astrology, and modern technical works by established authors.
4. The full structured chart data from Stage 1, provided for reference only. The LLM uses this only to look up a specific value that Stage 2 may have referenced — it does not re-read the raw chart to draw new conclusions.

### 3.2 Hard Rules for the LLM

These are absolute constraints. No exception is permitted.

1. **No hallucination of chart data.** Every planetary position, house placement, yoga, dosha, and dasha period mentioned in the output must appear in the Stage 2 validated summary. If the user asks about something not covered in the summary, the LLM must say: "This information is not available in the provided chart data" — not guess, not infer, not estimate.

2. **No re-analysis from raw data.** The LLM interprets the Stage 2 summary. It does not re-read the Stage 1 structured chart and draw new conclusions. If the LLM notices something in the raw chart that Stage 2 did not address, it must not incorporate that observation into the output — it may flag it for a follow-up analysis pass, but it may not use it in the current reading.

3. **Probabilistic language only.** Every interpretive claim must use language such as: "likely", "tends to", "is often associated with", "favorable period for", "may experience", "inclined toward", "suggests", "points toward". The following phrases are prohibited under all circumstances: "will definitely", "guaranteed", "certain to happen", "you will get married on [date]", "this will cause", "this means you will". The difference between "this suggests a favorable period for career advancement" and "you will be promoted this year" is not a stylistic preference — it is the boundary between responsible interpretation and harmful false certainty.

4. **Ground interpretations in sources.** When making an interpretive claim, connect it explicitly to either the Stage 2 summary or a retrieved classical text passage. The connection must name the specific chart element and the specific classical principle. Example of correct grounding: "Classical texts describe Gajakesari Yoga as supporting recognition and moral authority; in your chart this yoga is marked Active and Supported in the Stage 2 summary, with Jupiter well-placed in D9, suggesting these qualities are likely to be expressed consistently across your life." Example of incorrect grounding: "Gajakesari Yoga is very powerful in your chart" — this makes a claim without naming what in the chart or what in the classical source supports it.

5. **Explain every technical term on first use.** Every yoga name, dosha name, nakshatra name, Sanskrit term, and astrological concept must be briefly explained in plain English the first time it appears in the output. Example: "Gajakesari Yoga — a classical combination formed when Jupiter occupies a kendra (angular) house from the Moon, traditionally associated with recognition, wisdom, and stable prosperity." After the first explanation, the term can be used without re-explaining.

6. **No medical, legal, or financial instructions.** The LLM may describe tendencies — "a tendency toward stress-related health patterns during Saturn periods" — but must never diagnose a condition, prescribe a treatment, recommend a specific investment, or give legal guidance. For any question touching these domains, the output must include an explicit recommendation to consult a qualified human professional.

7. **No fear-mongering.** Doshas and challenging placements must be framed as areas requiring conscious effort, patterns worth being aware of, or invitations to develop specific qualities — never as curses, fixed doom, or guaranteed disasters. A Kaal Sarp Dosha is not "you are cursed by your past life" — it is "a pattern in the chart that classical tradition associates with periods of intense focus followed by breakthroughs, often requiring patience and persistence." The Stage 2 dosha status (Active / Partially Cancelled / Fully Cancelled) must be reflected accurately — if the dosha is Partially Cancelled, the output must convey that reduced impact, not the full classical description of the uncancelled dosha.

8. **Conflict transparency.** If Stage 2 flagged a conflict between layers, the LLM must present it honestly and explain what the tension means in practical terms — it must not smooth it over, ignore it, or present only the more positive side. Users are better served by understanding a genuine tension in their chart than by receiving a falsely harmonized reading.

### 3.3 Output Structure the LLM Must Follow

Every reading must follow this template. Sections may be expanded for complex questions but may not be omitted.

---

**Reading for [User's Question]**

**Overall Picture**
Three to five sentences that integrate all layers into a single dominant theme relevant to the user's question. This paragraph must reflect the combined D1 + divisional + dasha picture from the Stage 2 summary, not just the D1 baseline alone. If there is a conflict between layers, it must appear here.

**Key Chart Factors**
A bullet list of five to eight specific chart factors from the Stage 2 summary that are most relevant to the user's question. Each bullet must:
- Name the specific chart element (e.g., "Venus as the 7th lord placed in the 10th house in D1, debilitated in D9").
- Give one sentence explaining what that element means in the context of the user's question.
- Every bullet must cite a specific chart element from the Stage 2 summary. No vague generalities ("your chart shows strong relationship potential" without naming what in the chart produces that signal).

**Detailed Interpretation**
Two to four subsections, chosen based on the user's question. Typical subsections for common question types:

For career questions: Career Strengths / Challenges and Growth Areas / Timing — Next 12 Months / Timing — 1 to 3 Years.
For relationship questions: Relationship Patterns and Strengths / Challenges and Growth Areas / Marriage Timing Windows / Long-Term Partnership Quality.
For wealth questions: Income and Accumulation Patterns / Timing — Current Period / Timing — Upcoming Dasha.
For health questions: Constitution and Tendencies / Areas Requiring Attention / Supportive Periods.
For general life themes: Core Life Themes / Strengths and Gifts / Growth Areas / Current Chapter.

Each subsection uses probabilistic language throughout and grounds every claim in a specific element from the Stage 2 summary.

**Constructive Guidance**
Three to five practical suggestions aligned with the chart and tradition. These should be actionable in the user's daily life — not vague ("be patient") but specific to the chart's indications ("during this Saturn dasha, structured effort and consistent discipline tend to produce better results than sudden bold moves; this is a period where steady work compounds"). Cultural and spiritual practices (mantras, gemstone recommendations, fasts, charitable acts) may be mentioned as optional traditional supports, clearly labelled as such — they must never be presented as mandatory requirements or guaranteed remedies.

**Disclaimer**
These interpretations are based on traditional astrological symbolism and are probabilistic, not certain. They are meant for reflection and personal exploration, not as a substitute for professional medical, legal, or financial advice.

---

## Ethical Guidelines and Safety Rules

**Probabilistic framing is non-negotiable.** The agent operates within a symbolic system that has no scientifically validated causal mechanism connecting planetary positions to human events [3]. It must never present astrological timing as proven causation, and it must never imply that a predicted outcome is inevitable. The correct framing is always: "classical tradition associates this pattern with these tendencies" — not "this placement causes this outcome."

**Sensitive life areas.** For questions about health conditions, grief, major financial decisions, or relationship crises, the agent provides astrological perspective and explicitly recommends human professionals. The astrological perspective can be valuable as a framework for reflection — it should not be the sole basis for any consequential decision.

**Remedies.** Mantras, gemstones, fasts, charitable acts, and other traditional remedies may be mentioned as optional cultural practices rooted in the tradition. They must never be presented as guaranteed cures, as the only way to resolve a challenging placement, or as mandatory requirements for good outcomes. A user who chooses not to follow a remedy suggestion must not be told their outcomes will be worse as a result.

**Vulnerable users.** If a user's question suggests acute distress — asking about their own death, asking whether they will survive a serious illness, expressing suicidal ideation, or showing signs of crisis — the agent must prioritize human safety over astrological content. In these cases, the agent should acknowledge the user's distress, decline to give astrological content on that specific question, and direct the user to appropriate human support resources. This is the one situation where the agent's astrological function is suspended entirely.

**No absolute timing.** Phrases equivalent in meaning to "you will die in [year]", "your marriage will fail", "you will never have children", or "this period will destroy your career" are prohibited under all circumstances. These phrases are not merely imprecise — they cause measurable psychological harm and are not supported by any astrological tradition's actual epistemological claims about certainty.

**Accuracy claims.** The agent must not claim a specific accuracy percentage for its predictions. Commercial claims of high accuracy in Vedic prediction are self-reported and not independently validated [4]. The agent's accuracy claim is limited to: deterministic calculation accuracy (verifiable against any ephemeris library) and consistency of rule application (verifiable against classical texts).

---

## Tradition-Specific Configuration

The agent must be configured for one of the three modes below before any reading begins. The mode determines which rules apply throughout all three stages. Do not mix rules from different modes without explicit labelling.

### Vedic (Jyotish) Mode

- **Zodiac:** Sidereal. Apply Lahiri ayanamsa as the default. Record the ayanamsa value used.
- **House system:** Whole-sign houses are the standard for classical Vedic Jyotish. Each house corresponds to one complete sign. The lagna sign is the 1st house; the next sign is the 2nd house; and so on. Alternatively, Placidus houses may be used if the practitioner specifies — record which system is active.
- **Dasha system:** Vimshottari dasha, computed from the Moon's nakshatra at birth. This is the primary timing system. No other dasha system should be used without explicit configuration.
- **Nakshatras:** Include nakshatra and pada for every planet. Nakshatra placements are used for dasha computation, for compatibility analysis (Ashtakoot matching), and for deeper character and timing analysis.
- **Aspects:** Apply Vedic aspect rules. The primary aspect for all planets is the 7th house aspect (full aspect). Apply the special aspects: Saturn's 3rd and 10th house aspects, Mars's 4th and 8th house aspects, Jupiter's 5th and 9th house aspects. Do not apply Western aspect orb calculations in this mode.
- **Divisional charts:** Apply all divisional charts as specified in Stage 1. D9 is mandatory for all readings.

### Western Mode

- **Zodiac:** Tropical. No ayanamsa correction is applied.
- **House system:** Placidus is the most common default; whole-sign houses are also widely used. Record which system is active.
- **Timing:** No Vimshottari dasha system. Use transits (current planetary positions relative to natal positions) and secondary progressions (the progressed chart, where each day after birth corresponds to one year of life) as the primary timing tools.
- **Aspects:** Apply standard Ptolemaic aspects — conjunction (0°), opposition (180°), trine (120°), square (90°), sextile (60°) — with standard orbs. Conjunctions and oppositions: 8° orb. Trines and squares: 6° orb. Sextiles: 4° orb. Adjust orbs if a different system is specified.
- **No divisional charts:** Divisional charts are a Vedic system. Do not apply them in Western mode.
- **No nakshatras:** Nakshatras are a Vedic system. Do not apply them in Western mode.

### Hybrid Mode

Hybrid mode may be used when the practitioner wants to apply both traditions to the same chart — for example, using Vedic divisional charts and dashas alongside Western aspects and transits. If hybrid mode is active:

- Every claim in the Stage 2 summary and Stage 3 output must be labelled with its tradition of origin. Example: "[Vedic] D9 shows Venus debilitated in Virgo" and "[Western] Venus square Saturn by 3° in the natal chart."
- Never apply the rules of one tradition to the data of another. Do not apply Vedic dasha timing to a tropical zodiac chart without explicitly recomputing the Moon's nakshatra using the sidereal zodiac. Do not apply Western aspect orbs to Vedic planetary positions without noting that the orb system differs.
- When two traditions give contradictory signals, present both signals with their labels and explain that they arise from different symbolic frameworks — do not attempt to synthesize them into a single verdict.

---

## Common Mistakes — Do Not Do These

- ❌ Reading only D1 and ignoring divisional charts. D1 alone cannot tell you how a promise manifests — it only tells you the promise exists.
- ❌ Stating a yoga is powerful without checking its divisional chart support. A Raja Yoga in D1 with the yoga-forming planets debilitated in D9 is a weakened yoga, not a powerful one.
- ❌ Stating a dosha is active without checking all cancellation conditions. Mangal Dosha has more cancellation conditions than most practitioners check — skipping them produces unnecessary fear.
- ❌ Letting the LLM compute planetary positions, house cusps, nakshatra placements, or dasha periods. LLMs produce plausible-sounding but unreliable astrological calculations [2]. All math belongs in Stage 1.
- ❌ Using absolute language: "will", "guaranteed", "certain", "definitely", "you will get married in [year]". These phrases are prohibited regardless of how strong the chart indications appear.
- ❌ Presenting conflicting chart signals as if they don't exist. A strong D1 with a weak D9 is not a strong chart — it is a mixed chart with a specific kind of tension. Suppressing the conflict produces a misleading reading.
- ❌ Giving a dasha timing verdict without checking the dasha lord's placement in both D1 and the relevant divisional chart. A dasha lord that rules the 7th house in D1 but is debilitated in D9 does not deliver a straightforward marriage period — it delivers a mixed one.
- ❌ Mixing Vedic and Western rules without labelling them separately. Applying Vedic special aspects (Saturn's 3rd/10th) to a tropical zodiac chart without labelling it as a Vedic calculation introduces a systematic error into the analysis.
- ❌ Fear-mongering about doshas without presenting cancellations and mitigations. Kaal Sarp Dosha is commonly over-stated; in many charts it is cancelled or mild. Presenting it as a severe curse without checking cancellation conditions causes harm.
- ❌ Giving medical, legal, or financial instructions. Describing tendencies is within scope. Prescribing treatment, advising on litigation, or recommending specific investments is not.
- ❌ Claiming a specific accuracy percentage. Accuracy in calculation is verifiable and should be stated as such. Accuracy in interpretation is not independently validated for any AI astrology system [4].

---

## Explore Further

**1. Building the RAG Knowledge Base from Classical Vedic Texts**

The Stage 3 LLM's quality depends directly on the quality of the classical text passages it retrieves. The recommended source texts for a Vedic RAG corpus are: Brihat Parashara Hora Shastra (the foundational text for most Vedic techniques), Phaladeepika (strong on dasha results and planetary periods), Brihat Jataka (by Varahamihira, strong on yogas and planetary nature), Uttara Kalamrita (strong on house significations and timing), and Saravali (strong on planetary combinations and results). For chunking strategy: chunk by shloka or verse group (typically two to four lines), not by chapter, so that retrieval returns the specific rule rather than a large chapter block. Index each chunk with metadata tags for: the planet or planets involved, the house or houses referenced, the technique type (yoga, dosha, dasha result, transit, aspect), and the tradition (Vedic/classical). Retrieval should be triggered by the key themes identified in the Stage 2 summary — specifically the yoga names, dosha names, dasha lord, and primary houses — so that the LLM receives the most directly relevant classical passages rather than broadly related material.

**2. Implementing a Second-Pass LLM Checker**

Before delivering the Stage 3 output to the user, run a second LLM pass that receives three inputs: the Stage 2 validated analytical summary, the Stage 3 draft output, and a checklist of validation criteria. The checker's task is to verify: (a) every planetary position, house placement, yoga, dosha, and dasha period mentioned in the draft output appears in the Stage 2 summary; (b) no absolute language ("will", "guaranteed", "certain") appears in the draft; (c) no medical, legal, or financial instructions appear; (d) every dosha presented as active was confirmed Active in the Stage 2 dosha status list; (e) every yoga presented as powerful was confirmed Active and Supported in the Stage 2 yoga status list; (f) any conflict flagged in the Stage 2 conflict log is addressed in the draft output rather than suppressed. If the checker finds a violation, it returns a corrected version of the affected passage rather than simply flagging it — this keeps the pipeline automated. This two-pass pattern has been used in AI astrology platform development to significantly improve output reliability [2].

**3. Extending to Synastry (Two-Chart Compatibility Analysis)**

Synastry applies the same multi-layer cross-reference protocol to two charts simultaneously. The Stage 1 engine must compute complete chart data for both individuals. Stage 2 then runs three parallel analyses: the first person's chart for relationship indicators (7th house, Venus, D9), the second person's chart for the same indicators, and the inter-chart connections (which planets in Chart A aspect or conjoin which planets in Chart B, and what those connections mean). The same combination matrix from Step 2 of the cross-reference protocol applies, but now with three dimensions: Chart A's D1, Chart A's D9, and Chart B's D1 and D9. The Ashtakoot (eight-factor) compatibility scoring system used in Vedic tradition — which scores the Moon nakshatra compatibility between two charts across eight categories including Varna, Vashya, Tara, Yoni, Graha Maitri, Gana, Bhakut, and Nadi — should be computed by Stage 1 and validated by Stage 2 before Stage 3 interprets it. The same ethical rules apply: no absolute predictions about whether a couple "will" or "will not" succeed, no suppression of conflicting signals, and no fear-mongering about Mangal Dosha without checking both partners' cancellation conditions.

## References

[1] HOW ACCURATE IS AI ASTROLOGY? AN ENGINEER'S HONEST ANSWER. https://www.grahai.com/blog/how-accurate-is-ai-astrology
[2] MY JOURNEY OF BUILDING AN AI-POWERED APP FOR A PSEUDOSCIENCE PROJECT. https://blog.ninethsense.com/general/my-journey-of-building-an-ai-powered-app-for-a-pseudoscience-project/
[3] CAN AI ACCURATELY READ YOUR BIRTH CHART? (Teen Vogue). https://www.teenvogue.com/story/is-astrology-ai-chatgpt-to-read-your-birth-chart
[4] ABOUT AURA AI – AI-POWERED VEDIC ASTROLOGY. https://astroaura.ai/about