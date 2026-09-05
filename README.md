# BORDER SENTINEL — AI-Based Fake Identity & Document Screening System

An end-to-end Django platform that assists border security personnel by
automatically analyzing identity and travel documents, detecting tampering /
forgery, validating information against rules and security databases, and
verifying the traveller's face — then fusing everything into a 0-100 risk
score with a recommended action.

```
Module 1  OCR Extraction        Tesseract visual-zone + ICAO 9303 MRZ parser
                                with checksum-driven OCR self-correction
Module 2  Document Validation   15+ rules: checksums, date logic, formats,
                                cross-field consistency, watchlist/blacklist,
                                duplicate-identity detection
Module 3  Tampering Detection   Forensic suite: Error Level Analysis, noise
                                residual maps, copy-move search, metadata
                                fingerprints, portrait seam/splice analysis,
                                text-region recompression statistics
Module 4  Face Verification     OpenCV YuNet detector + SFace 128-d recogniser
                                (ONNX), classical histogram fallback
Risk engine                     Weighted fusion + policy gates + audit chain
```

## Quick start

```bash
cd border_sentinel
bash setup.sh                      # venv + deps + tesseract + ONNX models + migrate + seed
.venv/bin/python manage.py runserver 0.0.0.0:8000
```

`setup.sh` builds a project virtualenv (`.venv`) so it also works on
externally-managed Pythons (Kali/Debian/Ubuntu, PEP 668); on Windows/Git-Bash
it uses `.venv/Scripts/` automatically.  It then seeds:

* a **watchlist** entry (`VIKRAM SINGH`) and a **blacklisted document** (`B4444444`)
* a synthetic demo corpus in `demo_assets/generated/` (all imagery is
  computer/AI generated — no real person's document):

The documents follow realistic Indian layouts with bilingual
Hindi/English field labels (like the real ones): a two-column passport data
page (type/code/passport-no header, surname / given names / nationality /
place of birth / date of birth / sex / place of issue / issue & expiry
dates, ghost portrait, signature + TD3 MRZ), a visa sticker with the
reference-style header, visa number top-right, special endorsement lines and
a Type-B (MRV-B) machine readable zone (no immigration stamp), an
Aadhaar-style ID card, and a smart DL card (`AN01 20130003278`, `(AN)(NT)`
badges, chip, photo right, blood group, `Son/Daughter/Wife of` relative
name). Hindi labels are rendered with the bundled
`screening/fonts/NotoSansDevanagari-Regular.ttf`; on machines without the
file the generator falls back to English-only labels and extraction keeps
working (labels are placed so an eng-only Tesseract reads them cleanly).

| File | Scenario | Expected outcome |
|---|---|---|
| `passport_clean.jpg` + `subject_a_live_capture.jpg` | genuine traveller | **LOW** 1 (face match 0.82) |
| `passport_swapped_photo.jpg` + `subject_a_live_capture.jpg` | photo substitution / impersonation | **HIGH** 70 (face 0.18, biometric gate) |
| `passport_tampered_dob.jpg` | DOB altered in visual zone | **MEDIUM** 45 (visual-vs-MRZ consistency gate) |
| `passport_watchlist.jpg` | watchlisted traveller (`VIKRAM SINGH`) | **HIGH** 60 (watchlist gate) |
| `passport_blacklisted.jpg` | lost/stolen number `B4444444` | **HIGH** 75 (blacklist gate) |
| `passport_edited_metadata.jpg` | file processed in editing software | **MEDIUM** 30 (Module-3 metadata fingerprint) |
| `visa_clean.jpg` | visa sticker, MRV-B MRZ supplies validity | **LOW** 9 |
| `id_clean.jpg` | Aadhaar card (no printed expiry → warning only) | **LOW** 14 |
| `license_clean.jpg` | driving licence `AN01 20130003278` | **LOW** 10 |

Re-running `python manage.py seed_demo` clears previous screening records so
each demo starts from a clean audit trail.

## Web console

* `/` dashboard — volumes, risk mix, average processing time
* `/screen/` — upload a document (+ optional live capture), full report
* `/record/<id>/` — per-module report: extracted fields & MRZ, every rule
  check with PASS/FAIL, per-detector tamper bars, face similarity, risk
  breakdown, analyst overlay image, and the hash-chained audit trail
* `/records/` — searchable/filterable case list
* `/watchlist/` — manage person watchlist and lost/stolen document blacklist

## JSON API

```
POST /api/screen/            multipart: document, [live_photo], [doc_type]
GET  /api/records/?level=HIGH
GET  /api/records/<id>/      includes audit trail + chain validity
GET  /api/watchlist/         POST to add entries
GET  /api/health/            face backend + counters
```

Example:

```bash
curl -F document=@demo_assets/generated/passport_clean.jpg \
     -F live_photo=@demo_assets/subject_a_live_capture.jpg \
     http://localhost:8000/api/screen/
```

## How the risk score works

| Component | Weight | Source |
|---|---|---|
| OCR confidence | 10 | mean Tesseract confidence (visual + MRZ) |
| Validation failures | 20 | 10 pts per failed critical, 4 per warning |
| Tampering likelihood | 35 | Module-3 forensic fusion |
| Watchlist hit | 15 | person watchlist |
| Face verification | 20 | similarity vs threshold (uncertainty penalty when no live capture) |

Policy gates (border-control doctrine): any failed critical check ⇒ ≥ MEDIUM;
blacklisted document ⇒ ≥ 75; biometric mismatch ⇒ ≥ 70; watchlist hit ⇒ ≥ 60.
Bands: `<30` LOW (clear), `<60` MEDIUM (secondary inspection), else HIGH
(hold & refer to forgery unit).

## Engineering notes & honest limitations

* **MRZ first**: the ICAO parser repairs common OCR glyph confusions
  (O/0, I/1, S/5 …) via checksums and *flags* every repair; when the visual
  zone OCR is weak, the MRZ is the authoritative channel. Genuinely divergent
  visual-vs-MRZ values are kept so the consistency rules can flag tampering.
* **Module 3 is deliberately conservative.** Identity documents are covered
  in print (guilloche, stamps, MRZ glyphs), and our measurements showed that
  naive ELA / copy-move thresholds saturate on legitimate artwork or alias
  with the JPEG block-grid phase. The suite therefore reports only robust
  signals (editing-software metadata, extreme recompression tails, seam
  geometry, large-scale cloning) and renders suspect-region heatmaps for the
  analyst. Subtle pixel forgeries in the demo are caught by the logical
  gates (Module 2) and biometrics (Module 4) — the same division of labour as
  real checkpoints. A production upgrade path is deep forensic models
  (e.g. CAT-Net / MVSS-Net) behind the same detector interface.
* **Face backend**: YuNet + SFace cosine threshold 0.363 (OpenCV zoo
  documented value); validated in this repo at 0.81 genuine vs 0.18 imposter.
* Demo mode only: `DEBUG=True`, permissive hosts, SQLite. For production set
  real secrets, Postgres, auth on the API, and run OCR/forensics in workers.

## Tests

```bash
python manage.py test screening        # 28 tests: MRZ, rules, tamper, faces,
                                       # pipeline gates, web + API flows
```

## Running natively on Windows

Install Python 3.10+ and Tesseract-OCR (the UB-Mannheim installer puts it in
`C:\Program Files\Tesseract-OCR`).  The engine auto-detects that path (or the
x86 variant) and also honours a `TESSERACT_CMD` environment variable, so no
PATH change is required; if you prefer PATH:
`setx PATH "%PATH%;C:\Program Files\Tesseract-OCR"` and reopen the terminal.

```
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
.venv\Scripts\python manage.py migrate
.venv\Scripts\python manage.py seed_demo
.venv\Scripts\python manage.py runserver
```

## Database migrations

The initial schema ships as `screening/migrations/0001_initial.py`
(fully commented: `ScreeningRecord` case table, hash-chained `AuditEvent`
trail, `WatchlistEntry`, `ExpiredDocument`).  `setup.sh` applies it via
`python manage.py migrate`.  After changing `screening/models.py`:

```bash
python manage.py makemigrations screening   # generate the next migration
python manage.py migrate                    # apply it
python manage.py makemigrations --check     # CI guard: no missing migrations
```

## Layout

```
border_sentinel/
  config/            Django settings/urls
  screening/
    mrz.py           ICAO 9303 TD1/TD2/TD3 parsing + OCR self-correction
    ocr_engine.py    Tesseract visual-zone extraction + doc classifier
    rules.py         validation rule engine
    tamper.py        forensic detector suite
    faces.py         YuNet/SFace biometrics (+ classical fallback)
    scoring.py       risk fusion + policy gates
    synthetic.py     synthetic document generator + forgery operations
    pipeline.py      orchestration, persistence, audit chain
    migrations/      commented schema migrations (0001_initial)
    management/      seed_demo command (watchlist, blacklist, demo corpus)
    api.py / views.py / templates/ / static/
demo_assets/         AI-generated portraits + generated demo documents
```
