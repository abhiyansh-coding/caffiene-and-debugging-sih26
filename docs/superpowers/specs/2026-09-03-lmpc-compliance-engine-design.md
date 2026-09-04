# Design — LMPC Compliance Engine (SIH 2026, PS 26034)

**Date:** 2026-09-03
**Problem Statement:** SIH26034 — Software System to check compliance of Packaged
Commodities under Legal Metrology (Packaged Commodities) Rules, 2011 by scanning
products, images and labels.
**Team:** Caffiene and Debugging
**Project name:** none yet — the SIH portal asks for an *idea title*, not a brand, so this stays undecided until the team picks one.

---

## 1. What we are building, in one paragraph

A **compliance engine** for the Legal Metrology (Packaged Commodities) Rules, 2011,
plus **two clients** that feed it. The engine takes structured facts about a package
and returns findings, each citing a specific rule, the measured value, and the required
threshold. Client 1 is a **field scanner**: an inspector photographs a package with a
standard ID-1 card in frame, and the system recovers real-world scale, measures the
declarations in millimetres, and produces an annotated image plus a draft inspection
report. Client 2 is a **listing checker**: a browser extension that reads an e-commerce
product page and runs the same rules against the same engine. The engine never sees an
image or a web page; the clients never decide what is legal.

## 2. Scope

### In scope for v1
- Rule pack covering ~15 rules and 6 exemption predicates (§6, §7)
- Scale recovery from an ISO/IEC 7810 ID-1 reference card
- Millimetre measurement of glyph height, letter width, clear space, and contrast
- VLM-based field extraction from label photographs
- Annotated result image and draft PDF inspection report
- Two-screen web UI (upload/capture → result)
- Browser extension reading product listing pages

### Explicitly out of scope for v1
| Deferred | Why | When |
|---|---|---|
| Native mobile app (Expo/React Native) | PS says "web **and/or** mobile". We ship a PWA instead — see §8.1. A native shell would not improve a single measurement | Only if December brings a real driver: background sync, torch control, hardware scanners |
| User accounts / RBAC | PS requires it; wins nothing in the internal round | Grand Finale build |
| Inspection dashboard & history | Same | Grand Finale build |
| Cylindrical surface unwrapping | Planar homography fails on bottles; real work | Stretch goal |
| eMaap integration | No public API exists | Export schema only; never claim integration |
| Automated crawling of marketplaces | Platforms block scrapers; a dead demo is worse than none | Never — the extension reads pages the officer already opened |
| Editable (DOCX) report export | PS asks for it; PDF proves the point | Grand Finale build |

## 3. Architecture

```
                       ┌──────────────────────────┐
  photo + ID-1 card ──►│ scale.py                 │ mm/px + confidence
                       │  card detect, homography │
                       └───────────┬──────────────┘
                                   ▼
                       ┌──────────────────────────┐
                       │ extract.py               │ declarations as text
                       │  VLM read + CV boxes     │ + glyph bounding boxes
                       └───────────┬──────────────┘
                                   ▼
                       ┌──────────────────────────┐
                       │ measure.py               │ heights, widths, clear
                       │  px → mm, contrast, PDP  │ space, contrast ratios
                       └───────────┬──────────────┘
                                   ▼
   listing DOM ───────────────► PackageFacts
                                   │
                       ┌───────────▼──────────────┐
                       │ engine.py + rules/*.yaml │
                       │  applicability, checks   │
                       └───────────┬──────────────┘
                                   ▼
                              Finding[]
                                   │
                       ┌───────────▼──────────────┐
                       │ report.py                │ annotated image + PDF
                       └──────────────────────────┘
```

**The load-bearing boundary:** `extract` reads, `engine` decides. A vision model is
never asked whether something is legal — only what the label says. Every legal
conclusion comes from a deterministic evaluator over a declarative rule pack. This is
what makes findings auditable, and it is the answer to "what if the AI hallucinates?"
A wrong result traces to exactly one of two places: a misread field, or a misapplied rule.

## 4. Data contract

Both clients produce the same structure. Fields are `None` when absent — absence is a
finding, not an error.

```python
@dataclass
class Declaration:
    value: str | None
    source: Literal["label", "listing"]
    bbox: tuple[int, int, int, int] | None   # pixels, label only
    height_mm: float | None                   # measured glyph height
    width_ratio: float | None                 # mean width / height
    contrast_ratio: float | None              # vs local background
    confidence: float

@dataclass
class PackageFacts:
    manufacturer: Declaration
    country_of_origin: Declaration
    commodity_name: Declaration
    net_quantity: Declaration
    manufacture_date: Declaration
    mrp: Declaration
    consumer_care: Declaration
    unit_sale_price: Declaration

    net_qty_value: float | None                # parsed magnitude
    net_qty_unit: str | None                   # parsed SI unit
    commodity_class: str | None                # food | cosmetic | tobacco | drug | fast_food | thread | other
    packer_type: str | None                    # manufacturer | restaurant | importer — drives Rule 26(b)
    is_imported: bool
    package_shape: Literal["rectangular", "cylindrical", "other"] | None
    pdp_area_cm2: float | None
    is_moulded: bool                           # blown/formed/moulded → Table-I col 3
    clear_space_mm: dict | None                # {above, below, left, right}
    mm_per_px: float | None
    scale_confidence: float
    packed_on: date | None                     # drives temporal rule selection

@dataclass
class Finding:
    rule_id: str
    citation: str                              # "Rule 7(2) Table-I, LMPC Rules 2011"
    verdict: Literal["pass", "fail", "not_applicable", "undetermined"]
    measured: str | None                       # "2.1 mm"
    required: str | None                       # "2.5 mm"
    severity: Literal["major", "minor"]
    reason: str
```

`undetermined` is a first-class verdict. When scale confidence is too low to measure,
the system reports "measurement required", never a violation. An enforcement tool that
overclaims is worthless in a compounding proceeding.

## 5. Rule pack format

Rules are data, not code, for two reasons: the rules changed in 2021, February 2026 and
April 2026 with provisions landing 1 July 2027; and a package must be judged against
**the law as it stood on its date of packing**.

```yaml
- id: LMPC-7-2
  citation: "Rule 7(2) read with Table-I, LMPC Rules 2011"
  title: Minimum height of numerals and letters
  effective_from: 2011-04-01
  effective_to: null
  applies_when:
    all:
      - not_exempt
      - has: pdp_area_cm2
      - has: scale
  check: threshold_table
  table: table_I
  measure: height_mm
  applies_to_fields: [net_quantity, mrp, manufacture_date, consumer_care]
  severity: major

tables:
  table_I:
    key: pdp_area_cm2
    column: is_moulded
    bands:
      - { max: 50,   normal: 1.0, moulded: 1.5 }
      - { max: 100,  normal: 1.5, moulded: 3.0 }
      - { max: 500,  normal: 2.5, moulded: 4.0 }
      - { max: 2500, normal: 4.0, moulded: 6.0 }
      - { max: null, normal: 6.0, moulded: 6.0 }
```

The evaluator selects rules where `effective_from <= packed_on < effective_to`, tests
`applies_when`, runs `check`, and emits a `Finding`. Adding a rule when the law changes
is a YAML edit, not a code change.

### v1 rule list

**Presence — both clients**
| id | Rule | Check |
|---|---|---|
| LMPC-6-1-a | 6(1)(a) | Manufacturer/packer/importer name **and complete address including PIN** (Expl. I) |
| LMPC-6-1-aa | 6(1)(aa) | Country of origin — imported goods only |
| LMPC-6-1-b | 6(1)(b) | Common or generic name of commodity |
| LMPC-6-1-c | 6(1)(c) | Net quantity in standard units |
| LMPC-6-1-d | 6(1)(d) | Month and year of manufacture |
| LMPC-6-1-e | 6(1)(e) | Retail sale price |
| LMPC-6-2 | 6(2) | Consumer care: name, address, **phone and e-mail** |
| LMPC-6-1-ll | 6(1)(ll) | Unit sale price present |

**Format / semantics**
| id | Rule | Check |
|---|---|---|
| LMPC-6-1-e-fmt | 6(1)(e) | MRP must read as maximum retail price **inclusive of all taxes**, in ₹ |
| LMPC-6-1-c-unit | 6(1)(c) | Net quantity uses a legal SI unit |
| LMPC-6-1-ll-fmt | 6(1)(ll) | Unit price band matches net quantity: `per g` under 1 kg, `per kg` at or above; likewise ml/litre, cm/metre |
| LMPC-6-1-d-sane | 6(1)(d) | Date parses and is not in the future |

**Geometry — field scanner only**
| id | Rule | Check |
|---|---|---|
| LMPC-7-2 | 7(2) + Table-I | Glyph height in mm against the PDP-area band |
| LMPC-7-3 | 7(3) | Letter width ≥ ⅓ height, excluding `1`, `i`, `I`, `l` |
| LMPC-8-1 | 8(1) proviso | Clear space around net quantity: ≥1× numeral height above and below, ≥2× left and right |
| LMPC-9-1-b | 9(1)(b) | MRP and net quantity contrast conspicuously with background; waived when moulded on glass or plastic |

PDP area per Rule 7(4): rectangular = height × width of the display face; cylindrical =
40% of (height × circumference); other = 40% of total surface. Excludes tops, bottoms,
can flanges, bottle shoulders and necks.

## 6. Exemptions

Evaluated **before** any check. A false positive on an exempt package destroys
credibility with a real inspector faster than a missed violation.

| id | Source | Effect |
|---|---|---|
| EX-26-a | Rule 26(a) | Net quantity ≤ 10 g or 10 ml → all rules exempt — **except tobacco products** |
| EX-26-b | Rule 26(b) | Fast food packed by a restaurant or hotel → exempt |
| EX-26-c | Rule 26(c) | DPCO 2013 formulations → exempt, **but not medical devices declared as drugs** |
| EX-6-1-A | 6(1) proviso A | Bidis, incense sticks, PSU domestic LPG (14.2/5 kg) → no date declaration required |
| EX-6-1-C | 6(1) proviso C | Bidi, LPG under Administered Price Mechanism → no MRP required |
| EX-FSSAI | 6(1)(a) Expl. III | Food articles → Rule 6(1)(a) displaced by FSSAI. Report as *governed by FSSAI*, **not** as a violation |
| EX-7-5 | Rule 7(5) | Package ≤ 10 cc → identifying mark suffices |

## 7. Scale recovery

**Method.** Detect an ISO/IEC 7810 ID-1 card (85.60 × 53.98 mm) — any Indian debit,
credit or PVC Aadhaar card. Contour detection, quadrilateral fit, aspect-ratio filter,
then a homography from the four corners to a rectified plane. Output: millimetres per
pixel, plus a confidence value.

**Why a card and not AR depth.** Depth APIs are platform-specific and unreliable on
mid-range Android. A physical scale in frame is standard evidentiary photography
practice — it makes the evidence stronger, not weaker, and it is ~150 lines of OpenCV
with no ML.

**Failure modes we must handle honestly:**
- *Non-coplanar card* — a card lying on the table beside an upright box yields a wrong
  scale. Mitigation: the capture screen instructs the officer to hold the card flat
  against the same face as the declarations; we detect large plane disagreement and
  refuse rather than guess.
- *Curved surfaces* — a planar homography is invalid on bottles. v1 returns
  `undetermined` for cylindrical packages on geometry rules. Unwrapping is a stretch goal.
- *No card in frame* — geometry rules return `undetermined`; presence and format rules
  still run.

**Accuracy target:** glyph height within **±0.2 mm** of a caliper measurement on a flat
package. This is the number we must be able to defend on stage.

## 8. Interface

Two screens, two artifacts. Effort goes into the artifacts.

**Screens:** capture/upload, and result. That is all for v1.

**Artifact 1 — annotated result image.** Rectified photo showing: the reference card
outlined with `scale locked · 0.081 mm/px`; each declaration boxed and tagged with its
rule; failures called out in red with measured vs required (`Net Qty — 2.1 mm measured,
2.5 mm required · Rule 7(2) Table-I`); the Rule 8 clear-space rectangle drawn around the
net quantity with intruding print highlighted.

**Artifact 2 — draft inspection report (PDF).** Findings table with a rule citation per
row, the applicable compounding amount from the Rule 32A table, photo evidence, and an
officer signature block. Marked throughout as a **system-generated draft for officer
confirmation** — the officer adjudicates, the system proposes.

### 8.1 Mobile: PWA, not native

The PS asks for a "user-friendly **web and/or mobile**-based software application", so
web alone satisfies the requirement. But this is a field tool — an officer standing in a
shop holding a packet — so it must be genuinely good on a phone.

`<input type="file" capture="environment">` opens the phone's **native camera app**:
full sensor resolution, real autofocus, a familiar interface. For measurement accuracy
that beats any in-app camera we could build in a hackathon.

What remains is perception, and a `manifest.json` plus a small service worker (~30 lines)
closes it: home-screen icon, fullscreen launch, no browser chrome, offline capable.

**Why not Expo/React Native.** A second language and codebase, EAS builds, and signing —
the most likely way two developers fail to finish. And a native shell would not make a
single measurement more accurate; the camera is the same camera, and all the
differentiation lives in `scale.py` and the rule pack.

**Deck line:** *installs to the officer's phone like a native app, no app store, no
device management — deployable to an entire state inspectorate by sending a link.*
State IT departments hate MDM rollouts; this is an advantage, not a compromise.

**Stack:** FastAPI serving one hand-written HTML page. No React, no npm, no build step.
Full visual control, and it runs from a laptop with no internet.

**Offline constraint.** VLM extraction needs the network; college demo wifi cannot be
trusted. Extraction results for demo products are cached to disk, and all geometry runs
locally in OpenCV. If the network dies mid-demo, the impressive half still works.

## 9. Milestones

| When | Deliverable |
|---|---|
| Internal round (days away) | Scale + measure + engine working on 3–5 real products; two-screen UI; annotated image; draft PDF; ~15-rule pack; pitch deck |
| 20 Sep 2026 | SIH portal submission: idea title, description, 6-slide PDF on the official template |
| Sep–Dec 2026 | Browser extension; dashboard and inspection history; RBAC; DOCX export; cylindrical unwrapping; wider rule coverage |
| Grand Finale, Dec 2026 | Full prototype |

## 10. Risks and honest ceilings

| Risk | Reality | Mitigation |
|---|---|---|
| OCR quality on Indian packaging | Multilingual, glossy, low contrast, curved — Tesseract will embarrass us | VLM for reading; classical CV for geometry; cache demo extractions |
| Non-coplanar reference card | Silently wrong measurements — the worst failure mode | Detect plane disagreement, refuse rather than guess |
| Curved packages | Planar homography invalid | `undetermined`, stated openly |
| Scope creep into dashboards | The classic way hackathon teams finish nothing | Everything in §2's deferred table stays deferred |
| Overclaiming eMaap | A Ministry judge will probe it | Say "export-ready", never "integrated" |
| Two developers | The binding constraint | One language, one repo, no services, no build pipeline |

## 11. Acceptance criteria for the internal round

1. Photograph a real product bought from a shop, with an ID-1 card held against the
   label face. The system recovers scale and measures MRP glyph height within ±0.2 mm
   of a caliper reading.
2. The system correctly cites the Table-I band for the measured PDP area and returns
   the right verdict.
3. On a package with print intruding into the net-quantity clear space, Rule 8(1) fails
   and the annotated image shows the required rectangle and the intrusion.
4. On a package under 10 g, the system reports **exempt under Rule 26(a)** and raises no
   violations.
5. With the network disconnected, geometry findings still render.
6. The draft PDF opens and cites rule numbers matching the on-screen findings.

Criterion 4 matters as much as criterion 1: it is the one that proves we read the law.

---

## 12. Stack

One language, one process, no services. Every choice below is justified by what it
*replaces*, because the binding constraint is two developers.

| Layer | Choice | What it replaces |
|---|---|---|
| Language | Python 3.12+ | A second language. OpenCV and the rule engine both want Python |
| Web server | FastAPI + Uvicorn | Django (ORM, admin, migrations we don't need) |
| Templating | Jinja2 (ships with FastAPI usage) | A JS framework |
| Frontend | One hand-written HTML page + a little CSS | React, Next.js, npm, a build step, a bundler |
| Computer vision | OpenCV (`opencv-python-headless`) + NumPy | Torch, PaddleOCR, EasyOCR |
| Text location | OpenCV adaptive threshold + morphological dilation | An OCR engine dependency |
| Glyph measurement | `cv2.connectedComponentsWithStats` | OCR bounding boxes, which are too loose to measure |
| Text reading | Anthropic Python SDK, `claude-opus-5` | Tesseract, which fails on real Indian packaging |
| Rule pack | PyYAML | A rules DSL, a database of rules, a config service |
| Report PDF | `@media print` stylesheet + browser print-to-PDF | ReportLab, WeasyPrint, headless Chrome |
| Persistence | **The filesystem** | Postgres, SQLAlchemy, Alembic, Docker |
| Deployment | `uvicorn` on a laptop | Kubernetes, Docker, a cloud account, venue wifi |

### Why there is no database in v1

The PS requires a "repository of scanned products and compliance history", and we will
build it — in December. For v1 an inspection is a directory:

```
inspections/2026-09-05T14-22-08-a3f9/
  original.jpg          # never mutated — this is the evidence
  rectified.jpg
  annotated.jpg
  facts.json
  findings.json
```

That is queryable with `glob` and readable by a human. When we genuinely need
querying and history, the answer is **`sqlite3` from the standard library**, not a
server. Postgres becomes correct only when there are concurrent writers, which a
hackathon prototype will never have.

### Why the reading pipeline is shaped the way it is

Vision models are excellent at *reading* and unreliable at *precise bounding boxes*.
Classical CV is the reverse. So we split them along that line:

1. **Locate** text regions with adaptive threshold + morphological dilation (OpenCV)
2. **Crop** each region and send the crops to `claude-opus-5` for reading
3. **Match** returned semantic fields (MRP, net quantity, …) back to their source regions
4. **Measure** the matched region with `connectedComponentsWithStats` for exact glyph extents

We never trust a model's coordinates and never ask an OCR engine to understand Hindi
packaging. Each tool does the half it is good at.

### Dependencies, complete

```
fastapi
uvicorn
jinja2
opencv-python-headless
numpy
pyyaml
anthropic
```

Seven. No torch, no OCR engine, no database driver, no node_modules.

### Repository layout

```
repository root
  engine/
    scale.py          # ID-1 card detection, homography, mm/px + confidence
    extract.py        # text-region location, VLM reading, field matching
    measure.py        # px → mm, glyph height/width, clear space, contrast
    engine.py         # rule pack evaluator
    report.py         # annotated image renderer
    rules/
      lmpc-2011.yaml  # the rule pack
      tables.yaml     # Table-I and the Rule 32A compounding table
  web/
    app.py            # FastAPI, two routes
    templates/
    static/
  extension/          # December — browser extension for listing pages
  tests/
    test_engine.py    # rule evaluation, exemptions, temporal selection
    test_scale.py     # measurement accuracy against known references
  inspections/        # runtime output, gitignored
  fixtures/           # demo product photos + cached VLM extractions
```

### API cost

A label scan is roughly 2,500 input tokens (image + prompt) and 500 output tokens.
On `claude-opus-5` at $5/MTok input and $25/MTok output that is **about ₹2 per scan**.
Demo extractions are cached to `fixtures/`, so repeated demo runs cost nothing and work
offline. For the December e-commerce sweep, listing checks are text-only and the Batch
API halves the cost again.
