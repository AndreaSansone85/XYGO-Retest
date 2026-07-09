# 🌹 XYGO Wind Rose

**A psychometric instrument for the visual assessment of gender identity and sexual orientation**

Developed by the [ENDOSEX Research Group](https://sessuologiamedica.com) · Università degli Studi di Roma Tor Vergata
Principal Investigator: Prof. Emmanuele A. Jannini

---

## Overview

XYGO is a self-administered web-based tool that captures gender identity and sexual orientation on two independent continuous scales, visualised as a compass rose (wind rose). Responses are encoded in a compact **XYGO notation** (e.g. `X; G+2; O-2`) suitable for statistical analysis and clinical reporting.

The repository contains two instruments:

| File | Purpose |
|---|---|
| `xygo_windrose_v5.html` | Standard administration — single session |
| `xygo_validation.html` | Test-retest validation — two sessions (T1 / T2) |
| `xygo_validation_webhook.gs` | Google Apps Script backend for data collection |

---

## Scales

### X / Y axis — Gender Identity (`G`)
A vertical scale ranging from **−3** (strongly male-perceived) to **+3** (strongly female-perceived), with **0** as a neutral/non-binary anchor. Participants may select one or more points; a *fluid* mode represents the selection as a continuous range.

### Orientation axis — Sexual Orientation (`O`)
A horizontal scale ranging from **−3** (exclusively heterosexual) to **+3** (exclusively homosexual). Same multi-point and fluid selection logic applies.

> Circle labels display absolute values (1, 2, 3) to avoid positive/negative framing bias. Signed values are preserved in the data payload and XYGO notation.

### XYGO Notation
```
<sex>; G<gender_score(s)>; O<orientation_score(s)>

Examples:
  X; G+3; O-3       →  AFAB, fully female-identified, exclusively heterosexual
  Y; G-1:+1; O+2    →  AMAB, fluid gender identity (−1 to +1), predominantly homosexual
  X; G+1,+3; O0     →  AFAB, two gender points selected, asexual orientation
```

---

## Questionnaire Flow

```
Splash screen
    │
    ▼
Sesso biologico  ←  età (obbligatoria) + selezione AFAB/AMAB
    │
    ▼
XYGO — Identità di genere  (scala verticale)
    │
    ▼
XYGO — Orientamento sessuale  (scala orizzontale)
    │
    ▼
Informazioni su di te  ←  etichetta di genere + orientamento autodichiarato
    │
    ▼
Done  ←  notazione XYGO + invio dati al webhook
```

### Gender identity labels
Options are dynamically generated based on biological sex:

**AMAB (Y):** Cisgender · Parzialmente cisgender · Agender · Senza genere · Parzialmente transgender · Transgender · Pangender · Gender fluid

**AFAB (X):** same labels, with male/female references appropriately swapped

### Sexual orientation options
Eterosessuale · Parzialmente eterosessuale · Omosessuale · Parzialmente omosessuale · Asessuale · Pansessuale · Bisessuale

---

## Validation Tool (`xygo_validation.html`)

Designed for **anonymous test-retest** administration. Participants complete the instrument at two time points (T1, T2) without revealing their identity.

### Token-based matching
At the end of T1, a **6-character alphanumeric token** is generated client-side (e.g. `XK4R2M`) and displayed prominently. The participant saves it and enters it at the start of T2. The token is the sole key for merging T1 and T2 records.

- Alphabet excludes visually ambiguous characters: `O`, `0`, `I`, `1`
- The token is never linked to any personally identifiable information
- Email addresses (see below) are stored in a **separate sheet** and deleted after the reminder is sent

### Optional email reminder
At the end of T1, participants may optionally provide an email address to receive a reminder after 10 days. The reminder email is bilingual (Italian / English) and includes the participant's token.

Email addresses are:
- stored in a dedicated `XYGO_Reminders` sheet, **separate** from psychometric data
- automatically deleted from the sheet immediately after the reminder is sent
- never written to the `XYGO_Validation` data sheet

### Merging T1 and T2 in R

```r
library(tidyverse)
library(googlesheets4)

df <- read_sheet("<spreadsheet_url>", sheet = "XYGO_Validation")

t1 <- df |> filter(timepoint == "T1")
t2 <- df |> filter(timepoint == "T2")

merged <- inner_join(t1, t2, by = "token", suffix = c("_t1", "_t2"))
```

---

## Data Structure

### `XYGO_Validation` sheet

| Column | Description |
|---|---|
| `id` | UUID generated server-side |
| `timestamp` | ISO 8601 datetime |
| `timepoint` | `"T1"` or `"T2"` |
| `token` | Merge key for test-retest |
| `lang` | `"it"` or `"en"` |
| `sex` | `"X"` (AFAB) or `"Y"` (AMAB) |
| `age` | Integer |
| `genderLabel` | Self-reported gender identity label |
| `orientLabel` | Self-reported sexual orientation label |
| `genderScores` | Signed comma-separated values (e.g. `+1,+3`) |
| `fluidGender` | `TRUE` / `FALSE` |
| `orientScores` | Signed comma-separated values (e.g. `-2`) |
| `fluidOrient` | `TRUE` / `FALSE` |
| `notation` | Full XYGO notation string |

### `XYGO_Reminders` sheet

| Column | Description |
|---|---|
| `token` | Links to `XYGO_Validation` (not to identity) |
| `email` | Participant email — deleted after reminder is sent |
| `lang` | Language for the reminder email |
| `timestamp_t1` | T1 completion datetime |
| `send_after` | `timestamp_t1 + 10 days` |
| `reminder_sent` | `TRUE` / `FALSE` |
| `sent_at` | Datetime of actual sending |

---

## Setup

### 1. Deploy the Google Apps Script backend

1. Create a new **Google Spreadsheet**
2. Go to **Extensions → Apps Script** and paste the contents of `xygo_validation_webhook.gs`
3. Save, then **Deploy → New deployment**:
   - Type: **Web App**
   - Execute as: **Me**
   - Who has access: **Anyone (anonymous)**
4. Copy the generated URL
5. In `xygo_validation.html`, replace the `WEBHOOK` constant with your URL
6. Run `createDailyTrigger()` **once** from the Apps Script editor to activate the daily reminder check

### 2. Deploy the HTML

Host `xygo_validation.html` as a static page (e.g. GitHub Pages, any web server). No build step required — the file is fully self-contained.

### 3. Verify

Run `testT1()` and `testT2()` from the Apps Script editor. Two rows with `token = "TST001"` should appear in the `XYGO_Validation` sheet. Delete them after verification.

---

## Technical Stack

- **Frontend:** React 18 + Babel standalone (single HTML file, no build step)
- **Fonts:** DM Serif Display, DM Sans (Google Fonts)
- **Backend:** Google Apps Script → Google Sheets
- **Deployment:** GitHub Pages (static)
- **Analysis:** R (`tidyverse`, `googlesheets4`)

---

## Ethics

This instrument was developed under approval of the **Comitato Etico, Università degli Studi di Roma Tor Vergata**.
Data collection is fully anonymous. Participants provide informed consent before proceeding.

---

## Citation

>  Di Cristofaro A, Jannini TB, Colonnello E, Limoncin E, Mollaioli D, Ciocca G, Sansone A, Jannini EA. XYGO: proposing a new holistic measure of gender identity and sexual orientation. Nat Rev Urol. 2025 May 13;22(6):387–405. doi:10.1038/s41585-025-01041-7 PubMed PMID: 40360732.


---

## License

© ENDOSEX Research Group — Università degli Studi di Roma Tor Vergata. All rights reserved.
For academic collaborations and licensing enquiries, contact the PI.
