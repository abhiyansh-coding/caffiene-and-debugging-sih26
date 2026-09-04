# Legal Metrology domain notes — the parts that matter for PS 26034

Source: LMPC Rules 2011 (as amended to 2021), Legal Metrology Act 2009,
LMPC Amendment Rules 2026 (Feb 2026 + Second Amendment Apr 2026).

## 1. The mandatory declarations (Rule 6(1))

| # | Declaration | Rule | Notes |
|---|---|---|---|
| 1 | Name & address of manufacturer / packer / importer | 6(1)(a) | If no "manufactured by"/"packed by" qualifier, the named entity is *presumed* the manufacturer. Brand owner listed as "marketer" is liable. **Explanation III: for food articles this clause is displaced by FSSAI**. |
| 2 | Country of origin | 6(1)(aa) | Imported products only |
| 3 | Common/generic name of commodity | 6(1)(b) | Multi-product packs: name + qty of each |
| 4 | Net quantity in standard units | 6(1)(c) | Weight/measure, or number if sold by count |
| 5 | Month & year of manufacture | 6(1)(d) | Amended 31-10-2021 (was "manufacture/pack/import") |
| 5b | Best before / use by date | 6(1)(da) | Only if it can become unfit for consumption |
| 6 | Retail sale price | 6(1)(e) | Must read as **MRP inclusive of all taxes, in Indian currency** |
| 7 | Consumer care details | 6(2) | Name, address, **phone number AND e-mail** of complaint contact |
| 8 | Unit sale price | 6(1)(ll) | Prescribed format: `Rs__ per g` (<1 kg) / `per kg` (≥1 kg) / `per ml` (<1 L) / `per litre` (≥1 L) / `per cm` (<1 m) / `per metre` (≥1 m) / `per number` |
| 9 | Dimensions | 6(1)(f) | Only where size is relevant |

Special marks:
- **"GM"** at top of PDP for genetically modified food — Rule 6(7)
- **Red/brown dot** (non-veg origin) or **green dot** (veg origin) at top of PDP for
  soap, shampoo, toothpaste, cosmetics & toiletries — Rule 6(8)

Sticker rules — Rule 6(3)/(4):
- Stickers **may not** be used to alter or make a mandatory declaration…
- …**except** a sticker showing a *revised lower MRP*, which must not cover the original MRP.
- Stickers are fine for non-mandatory info.
→ *A sticker sitting on top of the MRP is itself a detectable violation.*

## 2. The geometric rules — this is where the real engineering is

### Rule 7(2) + Table-I — minimum letter/numeral height, in millimetres

| PDP area A (cm²) | Min height (mm) | Min height if blown/formed/moulded (mm) |
|---|---|---|
| A ≤ 50 | 1.0 | 1.5 |
| 50 < A ≤ 100 | 1.5 | 3.0 |
| 100 < A ≤ 500 | 2.5 | 4.0 |
| 500 < A ≤ 2500 | 4.0 | 6.0 |
| 2500 < A | 6.0 | 6.0 |

**The threshold depends on the physical area of the package.** You cannot check this
from pixels alone — you need real-world scale recovery. This is the hard problem, and
it is the reason most teams will silently skip this requirement.

### Rule 7(4) — how PDP area is computed
- Rectangular package: height × width of the display side
- Cylindrical / near-cylindrical: **40% of (height × circumference)**
- Any other shape: **40% of total surface area**
- Excludes top, bottom, can flanges, bottle shoulders and necks

### Rule 7(3) — letter width ≥ ⅓ of its height
Exceptions: numeral `1`, letters `i`, `I`, `l`.

### Rule 8(1) proviso — mandatory clear space around the net quantity declaration
The area surrounding the quantity declaration must be **free from printed information**:
- above and below: ≥ **1×** the height of the numeral
- left and right: ≥ **2×** the height of the numeral

This is a pure computer-vision geometry check and it is visually spectacular to
demonstrate — draw the required clear-space rectangle, highlight what intrudes into it.

### Rule 9 — manner of declaration
- 9(1)(a) legible and prominent
- 9(1)(b) **MRP and net quantity numerals must contrast conspicuously with the
  background** → measurable as a contrast ratio. Exempt if blown/formed/moulded on
  glass or plastic.
- 9(2) **no declaration may require reading through a liquid** in the package
- 9(3) an outer container/wrapper must repeat all declarations unless they're visible through it
- Rule 31(2): in advertisements, net quantity font size must equal the MRP font size

## 3. Exemptions — where naive checkers destroy their own credibility (Rule 26)

Nothing in these rules applies to:
- (a) net weight/measure **≤ 10 g or 10 ml** — *except tobacco and tobacco products*
- (b) fast food packed by a restaurant/hotel
- (c) scheduled & non-scheduled formulations under DPCO 2013 — *but medical devices
  declared as drugs get no exemption*
- (e) thread sold in coil to handloom weavers

Plus per-declaration carve-outs in Rule 6(1) provisos:
- No month/year needed on: bidis, incense sticks, 14.2 kg / 5 kg domestic LPG from a PSU
- No MRP needed on: bidi, domestic LPG under the Administered Price Mechanism
- Alcoholic beverages: State Excise law governs MRP
- Essential commodities with a notified price: that price applies
- Rule 7(5): for packages ≤ 10 cc, a mark identifying the manufacturer suffices

Also: for **food articles**, Rule 6(1)(a) is displaced by FSSAI; for **cosmetics**,
Drugs & Cosmetics Rules 1945 govern the date declaration. A checker that flags a 5 g
sachet or applies the wrong statute to a food package will be dismissed by any real
inspector on the first demo.

## 4. E-commerce — the live, current pain point

- **Rule 6(10)**: an e-commerce entity must display all mandatory declarations
  (except month/year of manufacture) on the platform. Marketplace models shift
  responsibility to the seller/manufacturer *only if* the platform is a passive
  intermediary and does due diligence under the IT Act 2000.
- **LMPC Amendment Rules 2026** (Feb 2026, second amendment Apr 2026): new obligations
  around **Country of Origin** display for imported products on listing pages, with a
  transition period — key provisions effective **1 July 2027**. Platforms must capture
  origin at SKU level at seller onboarding and display it before checkout.
- Enforcement problem: listings change constantly, dynamic pricing, many sellers per
  SKU. Inspectors have no tooling for this at all.

## 5. Enforcement reality — who uses this and what they need

- Enforcement is by **State Legal Metrology Departments** — Controller of Legal
  Metrology, Legal Metrology Officers / Inspectors.
- **eMaap**, the National Legal Metrology Portal, launched **4 December 2024** by the
  Department of Consumer Affairs — unifies state LM departments, licensing,
  verification/stamping, registrations, appeals into one national system.
  *Our repository/dashboard should be positioned as feeding eMaap, not as a rival silo.*
- Outcome of a finding is **compounding or prosecution**, not a "score":

  | Offence | Compounding — retailer/wholesaler | Compounding — manufacturer/importer |
  |---|---|---|
  | Contravention of s.29 | ₹2,000 | ₹10,000 |
  | Contravention of s.36(1) | ₹5,000 | ₹25,000 |
  | Contravention of s.36(2) | ₹10,000 | ₹50,000 |

- Rule 32: contravening rules 27/28 (registration) → ₹4,000 fine; any other rule with
  no specified punishment → ₹5,000 fine.
- Rule 27: manufacturers/packers/importers must **register** with the Director or
  Controller (₹500 fee). A registration number on the pack can be cross-checked.

**Implication:** the deliverable of an inspection is *evidence that survives challenge*.
Every flag must cite rule + sub-rule, show measured value vs required threshold, and be
confirmed by a human officer. The system proposes; the officer adjudicates.

## 6. Why the rule set must be data, not code

The rules changed in 2021, Feb 2026, and Apr 2026, with provisions landing on
1 July 2027. Two consequences:
1. Compliance logic belongs in a **versioned, declarative rule pack** a legal officer
   can edit — not in if-statements.
2. A package must be judged against **the law as it stood on its date of packing**.
   Penalising a package made before an amendment commenced is legally wrong.

## References
- LMPC Rules 2011 full text: https://legalmetrologymh.in/public/temp/368/02d3c4fef3045bc21d90ba000a28357e.pdf
- Legal Metrology Act & Rules (DoCA): https://consumeraffairs.gov.in/pages/legal-metrology-act
- eMaap portal background: https://www.indialaw.in/blog/civil/emaap-national-legal-metrology-portal/
- Amendment Rules 2026 analysis: https://www.mondaq.com/india/dodd-frank-consumer-protection-act/1806934/
- India's Evolving Metrology Ecosystem (PIB, May 2026): https://static.pib.gov.in/WriteReadData/specificdocs/documents/2026/may/doc2026520873201.pdf
