# Music Rights Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source platform for tracking composition and master recording rights, calculating royalty splits, and automating distribution to rights holders.

Music Rights Management is a mid-market rights administration platform built for independent labels, music publishers, and artist managers. It unifies catalog metadata, ownership splits, PRO registration, royalty statement ingestion, and payment calculation into a single system -- replacing the patchwork of spreadsheets, disconnected SaaS tools, and manual processes that dominate the independent music sector today.

---

## Why Music Rights Management?

- **Enterprise tools are priced out of reach.** Existing platforms like DISCO and Reprtoir target mid-to-large operations with commercial SaaS pricing, leaving smaller independents and self-publishing songwriters without affordable, full-featured tooling.
- **No single tool covers the full workflow.** DISCO excels at catalog marketing but has basic PRO registration. Reprtoir handles publishing administration well but lacks sync licensing and label workflows. Stem automates revenue splits but cannot manage contracts or multi-territory rights. Users are forced to stitch together multiple paid products.
- **Statement ingestion is fragile.** DSP and distributor royalty statements arrive in highly heterogeneous CSV and Excel formats. Incumbents handle common formats but require manual mapping for proprietary distributor layouts, creating bottlenecks at the most error-prone step in the pipeline.
- **Split management lacks auditability.** When ownership shares change -- due to reversion clauses, option exercises, or dispute resolution -- most tools overwrite historical records or require manual version tracking, making audits and dispute resolution difficult.
- **AI-generated music is creating new attribution complexity.** As AI-generated content grows, rights management must accommodate non-human creators, training data attributions, and fractional ownership at scale -- a need no incumbent currently addresses.

---

## Key Features

### Dual-Rights Catalog

- Separate but linked records for compositions (publishing) and master recordings
- Per-record ownership splits with territory and use-type specificity
- Linked relationships between compositions and their associated master recordings
- Catalog browser with search and filter by artist, label, release date, territory, and rights type

### Split Management & Royalty Calculation

- Version-controlled split sheets that preserve full history when ownership shares change
- Automatic recalculation of payable shares when splits are revised
- Rule-based royalty calculation engine applying per-territory, per-use-type, and per-rate-card schedules
- Currency conversion with daily FX rate feeds and historical rate preservation for auditing

### Statement Ingestion

- Configurable parser pipeline for major DSP and distributor CSV/Excel formats
- Period reconciliation and currency normalisation across heterogeneous statement sources
- Anomaly detection flagging potential under-reporting or missing DSP sources

### PRO Registration & Contract Management

- Structured CWR (Common Works Registration) export to ASCAP, BMI, SESAC, PRS, SOCAN, and other societies
- Per-society registration status tracking with acknowledgement monitoring
- Contract storage for recording agreements, co-publishing deals, sub-publishing agreements, and sync licences
- Option period and reversion date alerts

### Audit, Reporting & Access Control

- Immutable audit trail of all ownership changes, payment runs, and statement adjustments
- Revenue dashboards by territory, use type, catalog, and period
- Role-based access control: songwriters, artist managers, label accounting staff, and publishers each get appropriate visibility

---

## AI-Native Advantage

AI capabilities address the highest-friction parts of rights administration. Automated statement anomaly detection compares ingested DSP data against streaming metrics to flag under-reporting before it becomes a dispute. NLP-powered contract analysis extracts key terms, option periods, and reversion triggers from uploaded agreements, reducing manual data entry. Royalty forecasting projects future earnings by work based on streaming trajectory and historical collection patterns, helping rights holders plan cash flow. Sync licensing matching uses mood, tempo, and genre analysis to connect catalog tracks with incoming licensing briefs.

---

## Tech Stack & Deployment

- **Data model:** A graph-like structure suits the many-to-many relationships between works, recordings, rights holders, and territories better than a flat relational schema.
- **Statement ingestion:** A configurable transformation pipeline (ETL service) handles heterogeneous file formats without hard-coded importers.
- **Standards:** CWR (CISAC standard) for PRO registration; DDEX for catalog and statement data exchange. Both are publicly documented and patent-free.
- **Payment rails:** Integration with Stripe Connect, Tipalti, or ACH/SEPA for multi-currency disbursements to international rights holders.
- **Deployment:** Self-hosted or cloud-hosted. Role-based access control is a first-class concern given the multi-stakeholder nature of rights data.

---

## Market Context

The global music publishing market was valued at approximately USD 6 billion in 2024, with streaming revenue projected to reach roughly USD 10 billion by 2030. Independent labels and publishers are underserved by enterprise platforms sized for major-label volumes and budgets. A mid-market platform that automates statement ingestion, split calculations, and PRO registration -- without six-figure implementation costs -- addresses a credible gap, further expanded by sync licensing growth and the rise of AI-generated music with complex attribution requirements.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
