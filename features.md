# Music Rights Management — Feature & Functionality Survey

> Candidate #441 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| DISCO | SaaS | Commercial | https://get.disco.ac |
| Reprtoir | SaaS | Commercial | https://www.reprtoir.com |
| Stem | SaaS | Commercial | https://stem.is |
| Contracko | SaaS | Commercial | https://contracko.com |
| Songtrust | SaaS | Commercial (publisher admin) | https://songtrust.com |

## Feature Analysis by Solution

### DISCO

**Core features**
- All-in-one platform for labels, publishers, distributors, and songwriters centralising catalog metadata, ownership splits, contracts, and royalty data in a single workspace
- Dual-rights catalog management: separate but linked records for compositions and masters, each with their own ownership splits and territory restrictions
- Royalty accounting: automated calculation of payable amounts to each rights holder from ingested statement data
- Contract storage: recording agreements, co-publishing deals, and sync licences with renewal and option-period alerts
- Collaboration tools: share catalog previews and track access with external parties during sync licensing pitches

**Differentiating features**
- Unified workspace covering both rights administration and catalog marketing (streaming quality control, pitch tools, label copy management)
- Audio player and metadata editor built in — a catalog management and music review tool as well as a rights platform
- Multi-stakeholder access: songwriters, managers, labels, and publishers each get role-appropriate views of the same underlying data

**UX patterns**
- Catalog browser with search and filter by artist, label, release date, territory, and rights type
- Deal room interface for sharing album or track subsets with sync licensing prospects
- Royalty dashboard showing period-by-period income by source and territory

**Integration points**
- DSP statement import (Spotify, Apple Music, YouTube via Linkfire/Soundcharts aggregators)
- Distributor connections for delivery and reporting
- PRO data import for performance royalty reconciliation

**Known gaps**
- PRO registration workflow is basic; complex multi-society registration requires manual steps
- Statement ingestion handles common DSP formats but proprietary distributor formats require manual mapping
- Not designed for scale of major-label operations (tens of thousands of tracks); better suited to mid-size independents

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Reprtoir

**Core features**
- Cloud-based catalog metadata management for publishers and labels with PRO registration workflow
- Split sheet management: version-controlled ownership splits with history preservation when splits are revised
- Royalty tracking from DSP statements and collecting society distributions
- Contract management: storage of publishing agreements, co-publishing deals, and licensing contracts with alert dates for option periods and reversions
- Works registration: structured export of works to ASCAP, BMI, SESAC, PRS, and SOCAN with registration status tracking

**Differentiating features**
- PRO registration workflow is more structured than most competitors, with status tracking per society and jurisdiction
- Split version history: immutable record of how splits have changed over time for audit and dispute resolution
- Publisher-specific design: feature prioritisation reflects publishing administration workflows rather than label or distributor needs

**UX patterns**
- Works registration dashboard showing registration status per work per society
- Split sheet editor with contributor invitations for collaborative confirmation of ownership shares
- Revenue report by work, by period, and by source type (mechanical, performance, sync, master)

**Integration points**
- CSV and DDEX import for statement ingestion
- API for programmatic catalog management
- PRO portal data export in society-specific formats

**Known gaps**
- Royalty calculation engine handles mechanical and performance royalties but complex multi-territory calculations require manual validation
- No integrated audio player or sync licensing pitch tools
- Limited integration with non-publishing workflows (label distribution, artist management)

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Stem

**Core features**
- Revenue splits automation: when income arrives from any source, Stem automatically calculates and pays each contributor their agreed percentage
- Split management: configurable revenue splits across collaborators with no platform fee deducted from creator earnings (creators keep 100% of their stated split)
- Direct deposit payouts to contributors via bank transfer with transaction history
- Publishing administration: basic performance royalty collection through affiliated societies
- Reporting: per-platform and per-period revenue breakdown for each split recipient

**Differentiating features**
- Fastest path to automated revenue splitting for independent artists and small labels
- No minimum balance threshold for payouts; contributors receive funds as they arrive
- Transparent reporting visible to all split recipients without requiring a separate account

**UX patterns**
- Project setup wizard: define contributors, set splits, connect distribution, receive and distribute income automatically
- Contributor dashboard showing cumulative earnings and per-period breakdowns
- Payout history with transfer confirmation timestamps

**Integration points**
- Distribution platforms (DistroKid, TuneCore, CD Baby) for revenue aggregation
- Bank account and PayPal for direct disbursement
- Stripe-powered payment processing

**Known gaps**
- Not designed for complex multi-territory rights management or PRO registration at scale
- No contract management or deal administration features
- Royalty calculation is split-percentage based only; cannot handle complex royalty rate schedules or advances

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Dual-rights catalog: separate but linked records for compositions (publishing) and masters
- Split management: version-controlled ownership shares with territory and use-type specificity
- DSP and distributor statement ingestion with period reconciliation and currency conversion
- Royalty calculation engine applying per-territory, per-use-type rate schedules
- PRO registration workflow: structured work export and registration status tracking
- Contract storage with option period and reversion date alert system

### Differentiating Features
- Unified workspace covering rights administration, catalog marketing, and sync licensing
- Audio player and metadata editor embedded in the rights management tool (DISCO)
- Automated direct payouts to split recipients when income arrives (Stem)
- Multi-society registration workflow with per-society status tracking (Reprtoir)
- AI-generated rights management for AI-generated music with complex attribution requirements (emerging 2026 need)

### Underserved Areas / Opportunities
- Mid-market platform combining PRO registration, statement ingestion, automated royalty calculation, and split disbursement in one affordable product below enterprise pricing
- AI music attribution tracking: as AI-generated content grows, rights management must accommodate non-human creators, training data attributions, and fractional ownership at scale
- Sync licensing workflow: integrated pitch tools, usage monitoring, and licence fee collection alongside the rights management layer
- International royalty reclaim: tooling to identify and claim uncollected foreign performance royalties that collecting societies hold

### AI-Augmentation Candidates
- Automated statement anomaly detection: identifying under-reporting or missing DSP sources by comparing against streaming data
- NLP-powered contract analysis: extracting key terms, option periods, and reversion triggers from uploaded agreements
- Royalty forecasting: projecting future earnings by work based on streaming trajectory and historical collection patterns
- Sync licensing matching: AI matching catalog tracks to incoming licensing briefs based on mood, tempo, and genre analysis

## Legal & IP Summary

Music rights management is a complex legal domain spanning copyright law (varying by territory), collecting society regulations, and contractual frameworks. The platform itself does not generate protectable IP in the rights data it manages — ownership splits, royalty calculations, and contract terms are factual records belonging to the rights holders. Standard royalty calculation formulas (mechanical rate calculation, performance royalty allocation) are based on statutory and contractual rate structures, not proprietary algorithms. DSP reporting formats vary by platform and are not standardised; statement ingestion parsers must be developed independently for each major DSP's CSV format. PRO registration formats (CWR — Common Works Registration) are defined by the CISAC standard and are publicly documented. A new entrant can build a full rights management platform using CWR, DDEX, and standard database technologies without patent encumbrances. The primary legal obligation is ensuring that royalty payments are calculated correctly and paid in accordance with the underlying rights agreements — errors create contractual liability to rights holders.

## Recommended Feature Scope

**Must-have (MVP)**:
- Dual-rights catalog: separate composition and master records with linked relationships
- Split management: version-controlled ownership shares with territory and use-type granularity
- DSP statement ingestion: configurable parser for major DSP and distributor CSV formats with period and currency reconciliation
- Royalty calculation engine applying per-contract rate tables and generating payable amounts per rights holder
- PRO registration workflow: structured CWR export and registration status tracking per society
- Contract storage with option period and reversion date alerts

**Should-have (v1.1)**:
- Automated payment disbursement to split recipients via bank transfer and multi-currency payout rails (Tipalti, Stripe Connect)
- Daily FX rate feed with historical rate preservation for audit purposes
- Revenue analytics dashboard: income by territory, use type, catalog, and period
- Audit trail: immutable log of all split changes, payment runs, and statement adjustments
- Multi-stakeholder access: role-based views for songwriters, managers, labels, and publishers

**Nice-to-have (backlog)**:
- AI-powered statement anomaly detection flagging potential under-reporting by source
- Sync licensing workflow: pitch delivery, usage monitoring, and licence fee invoicing
- NLP contract analysis extracting key terms and alert dates from uploaded agreements
- AI music attribution module for AI-generated content with configurable ownership attribution models
- International royalty reclaim tracking: identifying uncollected foreign performance royalties for active pursuit
