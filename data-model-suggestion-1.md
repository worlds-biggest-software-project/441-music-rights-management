# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Music Rights Management · Candidate #441 · Generated: 2026-05-25

---

## Overview

This model uses a fully normalized relational schema in PostgreSQL, treating the music rights domain as a set of well-defined entities connected by foreign keys and junction tables. Every concept -- works, recordings, rights holders, splits, territories, statements, royalties -- gets its own table with strict referential integrity. This approach prioritizes data consistency, auditability, and alignment with industry standards (CWR, DDEX, ISO identifiers).

The schema is organized into six logical modules:
1. **Catalog** -- compositions, recordings, releases, and their metadata
2. **Parties** -- rights holders, publishers, PROs, and their identifiers
3. **Rights & Splits** -- ownership shares with territory/use-type granularity and version history
4. **Contracts** -- agreements, deal terms, option periods, and reversion clauses
5. **Statements & Royalties** -- ingested DSP data, royalty calculations, and payment runs
6. **System** -- audit trail, users, roles, FX rates, and configuration

---

## Module 1: Catalog

### works (Musical Compositions)

```sql
CREATE TABLE works (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               TEXT NOT NULL,
    title_normalized    TEXT NOT NULL,  -- lowercase, accent-stripped for search
    iswc                VARCHAR(15),   -- T-XXXXXXXXX-C format (ISO 15707)
    alternate_titles    TEXT[],         -- array of alternate/translated titles
    language_code       VARCHAR(5),    -- ISO 639-1 language of lyrics
    duration_seconds    INTEGER,
    genre               VARCHAR(100),
    sub_genre           VARCHAR(100),
    work_type           VARCHAR(50) NOT NULL DEFAULT 'MusicalWork',
        -- MusicalWork, LyricWork, ArrangementWork, AdaptedWork
    ai_contribution     BOOLEAN NOT NULL DEFAULT FALSE,
    ai_contribution_pct NUMERIC(5,2),  -- percentage of AI contribution if applicable
    creation_date       DATE,
    registration_date   DATE,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, registered, disputed, withdrawn
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    CONSTRAINT chk_iswc_format CHECK (
        iswc IS NULL OR iswc ~ '^T-[0-9]{9}-[0-9]$'
    ),
    CONSTRAINT chk_ai_pct CHECK (
        ai_contribution_pct IS NULL OR (ai_contribution_pct >= 0 AND ai_contribution_pct <= 100)
    )
);

CREATE UNIQUE INDEX idx_works_iswc ON works (iswc) WHERE iswc IS NOT NULL;
CREATE INDEX idx_works_title_normalized ON works USING gin (title_normalized gin_trgm_ops);
CREATE INDEX idx_works_status ON works (status);
CREATE INDEX idx_works_created_at ON works (created_at);
```

### recordings (Master Sound Recordings)

```sql
CREATE TABLE recordings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               TEXT NOT NULL,
    title_normalized    TEXT NOT NULL,
    isrc                VARCHAR(12),   -- ISO 3901 format: CC-XXX-YY-NNNNN
    duration_seconds    INTEGER,
    recording_date      DATE,
    release_date        DATE,
    recording_type      VARCHAR(30) NOT NULL DEFAULT 'SoundRecording',
        -- SoundRecording, MusicVideo, Remix, LiveRecording, Remaster
    primary_artist_id   UUID REFERENCES parties(id),
    label_id            UUID REFERENCES parties(id),
    catalog_number      VARCHAR(50),
    audio_format        VARCHAR(20),   -- WAV, FLAC, MP3, AAC
    sample_rate         INTEGER,       -- 44100, 48000, 96000
    bit_depth           INTEGER,       -- 16, 24, 32
    is_ai_generated     BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    CONSTRAINT chk_isrc_format CHECK (
        isrc IS NULL OR isrc ~ '^[A-Z]{2}[A-Z0-9]{3}[0-9]{7}$'
    )
);

CREATE UNIQUE INDEX idx_recordings_isrc ON recordings (isrc) WHERE isrc IS NOT NULL;
CREATE INDEX idx_recordings_title ON recordings USING gin (title_normalized gin_trgm_ops);
CREATE INDEX idx_recordings_artist ON recordings (primary_artist_id);
CREATE INDEX idx_recordings_label ON recordings (label_id);
```

### work_recordings (Link Works to Recordings -- many-to-many)

```sql
CREATE TABLE work_recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id         UUID NOT NULL REFERENCES works(id) ON DELETE CASCADE,
    recording_id    UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    is_primary      BOOLEAN NOT NULL DEFAULT TRUE,  -- primary vs. sample/interpolation
    relationship    VARCHAR(30) NOT NULL DEFAULT 'performance',
        -- performance, cover, sample, interpolation, remix, medley
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_work_recording UNIQUE (work_id, recording_id)
);

CREATE INDEX idx_wr_work ON work_recordings (work_id);
CREATE INDEX idx_wr_recording ON work_recordings (recording_id);
```

### releases

```sql
CREATE TABLE releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           TEXT NOT NULL,
    upc             VARCHAR(14),    -- UPC/EAN barcode
    release_type    VARCHAR(30) NOT NULL DEFAULT 'single',
        -- single, EP, album, compilation, boxset
    release_date    DATE,
    label_id        UUID REFERENCES parties(id),
    catalog_number  VARCHAR(50),
    territory_code  VARCHAR(5),     -- ISO 3166-1 alpha-2 for release territory
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_upc_format CHECK (
        upc IS NULL OR upc ~ '^[0-9]{12,14}$'
    )
);

CREATE UNIQUE INDEX idx_releases_upc ON releases (upc) WHERE upc IS NOT NULL;
```

### release_tracks

```sql
CREATE TABLE release_tracks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    release_id      UUID NOT NULL REFERENCES releases(id) ON DELETE CASCADE,
    recording_id    UUID NOT NULL REFERENCES recordings(id),
    disc_number     SMALLINT NOT NULL DEFAULT 1,
    track_number    SMALLINT NOT NULL,
    side            VARCHAR(1),  -- A/B for vinyl

    CONSTRAINT uq_release_track UNIQUE (release_id, disc_number, track_number)
);

CREATE INDEX idx_rt_release ON release_tracks (release_id);
CREATE INDEX idx_rt_recording ON release_tracks (recording_id);
```

---

## Module 2: Parties

### parties (Rights Holders, Publishers, Labels, PROs)

```sql
CREATE TABLE parties (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_type          VARCHAR(30) NOT NULL,
        -- songwriter, composer, lyricist, arranger, performer, publisher,
        -- sub_publisher, administrator, label, distributor, pro, dsp
    legal_name          TEXT NOT NULL,
    display_name        TEXT,
    ipi_name_number     VARCHAR(11),   -- IPI Name Number (CISAC)
    ipi_base_number     VARCHAR(13),   -- IPI Base Number
    ipn                 VARCHAR(8),    -- International Performer Number
    isni                VARCHAR(19),   -- ISNI (ISO 27729)
    mbid                UUID,          -- MusicBrainz Identifier
    tax_id              VARCHAR(30),   -- encrypted at rest
    email               VARCHAR(255),
    phone               VARCHAR(30),
    country_code        VARCHAR(2),    -- ISO 3166-1 alpha-2
    address_line_1      TEXT,
    address_line_2      TEXT,
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    pro_affiliation_id  UUID REFERENCES parties(id),  -- which PRO they belong to
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_ipi_format CHECK (
        ipi_name_number IS NULL OR ipi_name_number ~ '^[0-9]{9,11}$'
    )
);

CREATE UNIQUE INDEX idx_parties_ipi ON parties (ipi_name_number)
    WHERE ipi_name_number IS NOT NULL;
CREATE INDEX idx_parties_type ON parties (party_type);
CREATE INDEX idx_parties_name ON parties USING gin (legal_name gin_trgm_ops);
```

### party_identifiers (Multiple external identifiers per party)

```sql
CREATE TABLE party_identifiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id        UUID NOT NULL REFERENCES parties(id) ON DELETE CASCADE,
    identifier_type VARCHAR(30) NOT NULL,
        -- ipi_name, ipi_base, ipn, isni, mbid, pro_member_id, tax_id, custom
    identifier_value TEXT NOT NULL,
    issuing_org     VARCHAR(100),
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_party_identifier UNIQUE (party_id, identifier_type, identifier_value)
);

CREATE INDEX idx_pi_party ON party_identifiers (party_id);
CREATE INDEX idx_pi_lookup ON party_identifiers (identifier_type, identifier_value);
```

### party_bank_accounts (Payment details)

```sql
CREATE TABLE party_bank_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id        UUID NOT NULL REFERENCES parties(id) ON DELETE CASCADE,
    account_name    TEXT NOT NULL,
    currency_code   VARCHAR(3) NOT NULL,  -- ISO 4217
    bank_name       TEXT,
    iban            VARCHAR(34),          -- encrypted at rest
    swift_bic       VARCHAR(11),
    routing_number  VARCHAR(9),           -- US ACH
    account_number  TEXT,                 -- encrypted at rest
    payment_method  VARCHAR(30) NOT NULL DEFAULT 'bank_transfer',
        -- bank_transfer, paypal, stripe_connect, tipalti, check
    payment_ref     TEXT,                 -- PayPal email, Stripe account ID, etc.
    is_primary      BOOLEAN NOT NULL DEFAULT TRUE,
    is_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pba_party ON party_bank_accounts (party_id);
```

---

## Module 3: Rights & Splits

### work_shares (Ownership splits for compositions)

```sql
CREATE TABLE work_shares (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(id) ON DELETE CASCADE,
    party_id            UUID NOT NULL REFERENCES parties(id),
    role                VARCHAR(30) NOT NULL,
        -- composer, lyricist, arranger, author, adapter, translator,
        -- original_publisher, sub_publisher, administrator
    performance_share   NUMERIC(7,4) NOT NULL DEFAULT 0,   -- % of performance rights
    mechanical_share    NUMERIC(7,4) NOT NULL DEFAULT 0,    -- % of mechanical rights
    sync_share          NUMERIC(7,4) NOT NULL DEFAULT 0,    -- % of sync licensing rights
    print_share         NUMERIC(7,4) NOT NULL DEFAULT 0,    -- % of print rights
    controlled          BOOLEAN NOT NULL DEFAULT FALSE,      -- is this party controlled by us?
    publisher_party_id  UUID REFERENCES parties(id),         -- which publisher controls this writer
    version             INTEGER NOT NULL DEFAULT 1,
    effective_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to        DATE,                                -- NULL = currently active
    superseded_by       UUID REFERENCES work_shares(id),     -- points to newer version
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    CONSTRAINT chk_share_range CHECK (
        performance_share >= 0 AND performance_share <= 100 AND
        mechanical_share  >= 0 AND mechanical_share  <= 100 AND
        sync_share        >= 0 AND sync_share        <= 100 AND
        print_share       >= 0 AND print_share       <= 100
    )
);

CREATE INDEX idx_ws_work ON work_shares (work_id);
CREATE INDEX idx_ws_party ON work_shares (party_id);
CREATE INDEX idx_ws_active ON work_shares (work_id, effective_from, effective_to);
```

### work_share_territories (Territory-specific split overrides)

```sql
CREATE TABLE work_share_territories (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_share_id       UUID NOT NULL REFERENCES work_shares(id) ON DELETE CASCADE,
    territory_code      VARCHAR(4) NOT NULL,   -- ISO 3166-1 alpha-2 or CISAC territory code
    include_exclude     VARCHAR(7) NOT NULL DEFAULT 'include',  -- include or exclude
    performance_share   NUMERIC(7,4),  -- override; NULL = use parent share
    mechanical_share    NUMERIC(7,4),
    sync_share          NUMERIC(7,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_wst UNIQUE (work_share_id, territory_code)
);

CREATE INDEX idx_wst_share ON work_share_territories (work_share_id);
CREATE INDEX idx_wst_territory ON work_share_territories (territory_code);
```

### recording_shares (Ownership splits for master recordings)

```sql
CREATE TABLE recording_shares (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_id        UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    party_id            UUID NOT NULL REFERENCES parties(id),
    role                VARCHAR(30) NOT NULL,
        -- featured_artist, producer, label, mixer, remixer, session_musician
    master_share        NUMERIC(7,4) NOT NULL DEFAULT 0,     -- % of master revenue
    neighboring_share   NUMERIC(7,4) NOT NULL DEFAULT 0,     -- % of neighboring rights
    version             INTEGER NOT NULL DEFAULT 1,
    effective_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to        DATE,
    superseded_by       UUID REFERENCES recording_shares(id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    CONSTRAINT chk_rec_share_range CHECK (
        master_share      >= 0 AND master_share      <= 100 AND
        neighboring_share >= 0 AND neighboring_share <= 100
    )
);

CREATE INDEX idx_rs_recording ON recording_shares (recording_id);
CREATE INDEX idx_rs_party ON recording_shares (party_id);
CREATE INDEX idx_rs_active ON recording_shares (recording_id, effective_from, effective_to);
```

### recording_share_territories

```sql
CREATE TABLE recording_share_territories (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_share_id  UUID NOT NULL REFERENCES recording_shares(id) ON DELETE CASCADE,
    territory_code      VARCHAR(4) NOT NULL,
    include_exclude     VARCHAR(7) NOT NULL DEFAULT 'include',
    master_share        NUMERIC(7,4),     -- override; NULL = use parent
    neighboring_share   NUMERIC(7,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_rst UNIQUE (recording_share_id, territory_code)
);

CREATE INDEX idx_rst_share ON recording_share_territories (recording_share_id);
```

---

## Module 4: Contracts

### contracts

```sql
CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_type       VARCHAR(50) NOT NULL,
        -- recording_agreement, co_publishing, sub_publishing, admin_agreement,
        -- sync_licence, master_use_licence, distribution_agreement,
        -- songwriter_agreement, producer_agreement
    title               TEXT NOT NULL,
    contract_number     VARCHAR(50),
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, active, expired, terminated, disputed
    effective_date      DATE,
    expiry_date         DATE,
    auto_renew          BOOLEAN NOT NULL DEFAULT FALSE,
    renewal_term_months INTEGER,
    governing_law       VARCHAR(100),    -- jurisdiction
    currency_code       VARCHAR(3),      -- ISO 4217 primary currency
    advance_amount      NUMERIC(14,2),
    advance_currency    VARCHAR(3),
    advance_recouped    BOOLEAN NOT NULL DEFAULT FALSE,
    notes               TEXT,
    document_url        TEXT,            -- S3/GCS path to signed document
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id)
);

CREATE INDEX idx_contracts_type ON contracts (contract_type);
CREATE INDEX idx_contracts_status ON contracts (status);
CREATE INDEX idx_contracts_expiry ON contracts (expiry_date);
```

### contract_parties

```sql
CREATE TABLE contract_parties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    party_id        UUID NOT NULL REFERENCES parties(id),
    role            VARCHAR(30) NOT NULL,
        -- licensor, licensee, publisher, writer, artist, label, administrator
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_contract_party UNIQUE (contract_id, party_id, role)
);

CREATE INDEX idx_cp_contract ON contract_parties (contract_id);
CREATE INDEX idx_cp_party ON contract_parties (party_id);
```

### contract_works (Which works/recordings are covered by a contract)

```sql
CREATE TABLE contract_works (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    work_id         UUID REFERENCES works(id),
    recording_id    UUID REFERENCES recordings(id),
    scope           VARCHAR(30) NOT NULL DEFAULT 'specific',
        -- specific, catalog, future_works
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT chk_contract_target CHECK (
        work_id IS NOT NULL OR recording_id IS NOT NULL OR scope = 'future_works'
    )
);

CREATE INDEX idx_cw_contract ON contract_works (contract_id);
CREATE INDEX idx_cw_work ON contract_works (work_id);
CREATE INDEX idx_cw_recording ON contract_works (recording_id);
```

### contract_territories

```sql
CREATE TABLE contract_territories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    territory_code  VARCHAR(4) NOT NULL,
    include_exclude VARCHAR(7) NOT NULL DEFAULT 'include',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_ct UNIQUE (contract_id, territory_code)
);

CREATE INDEX idx_ct_contract ON contract_territories (contract_id);
```

### contract_option_periods

```sql
CREATE TABLE contract_option_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    period_number   SMALLINT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    exercise_deadline DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, exercised, declined, expired
    advance_amount  NUMERIC(14,2),
    advance_currency VARCHAR(3),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_option_period UNIQUE (contract_id, period_number)
);

CREATE INDEX idx_cop_contract ON contract_option_periods (contract_id);
CREATE INDEX idx_cop_deadline ON contract_option_periods (exercise_deadline)
    WHERE status = 'pending';
```

### contract_reversion_triggers

```sql
CREATE TABLE contract_reversion_triggers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id) ON DELETE CASCADE,
    trigger_type    VARCHAR(50) NOT NULL,
        -- date_based, recoupment, sales_threshold, out_of_print, custom
    trigger_date    DATE,
    trigger_condition TEXT,   -- human-readable description
    reversion_scope VARCHAR(30) NOT NULL DEFAULT 'full',
        -- full, partial, territory_specific
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, triggered, waived
    triggered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_reversion UNIQUE (contract_id, trigger_type, trigger_date)
);

CREATE INDEX idx_crt_contract ON contract_reversion_triggers (contract_id);
CREATE INDEX idx_crt_pending ON contract_reversion_triggers (trigger_date)
    WHERE status = 'pending';
```

---

## Module 5: Statements & Royalties

### statement_batches (Ingested statement files)

```sql
CREATE TABLE statement_batches (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type         VARCHAR(30) NOT NULL,
        -- dsp, distributor, pro, publisher, sub_publisher
    source_name         VARCHAR(100) NOT NULL,  -- "Spotify", "DistroKid", "ASCAP"
    source_party_id     UUID REFERENCES parties(id),
    file_name           TEXT NOT NULL,
    file_format         VARCHAR(20) NOT NULL,   -- csv, xlsx, tsv, ddex_dsr, custom
    file_hash           VARCHAR(64),            -- SHA-256 for dedup
    statement_period    DATERANGE NOT NULL,      -- [start, end)
    original_currency   VARCHAR(3) NOT NULL,    -- ISO 4217
    total_amount        NUMERIC(16,4),          -- sum of all line items in original currency
    line_count          INTEGER,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, parsing, parsed, validated, calculated, error, rejected
    error_message       TEXT,
    ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ,
    ingested_by         UUID REFERENCES users(id)
);

CREATE INDEX idx_sb_source ON statement_batches (source_name);
CREATE INDEX idx_sb_status ON statement_batches (status);
CREATE INDEX idx_sb_period ON statement_batches USING gist (statement_period);
CREATE UNIQUE INDEX idx_sb_file_hash ON statement_batches (file_hash) WHERE file_hash IS NOT NULL;
```

### statement_lines (Individual line items from statements)

```sql
CREATE TABLE statement_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id            UUID NOT NULL REFERENCES statement_batches(id) ON DELETE CASCADE,
    line_number         INTEGER NOT NULL,
    
    -- Matching identifiers (as provided by the source)
    reported_isrc       VARCHAR(20),
    reported_iswc       VARCHAR(20),
    reported_upc        VARCHAR(20),
    reported_title      TEXT,
    reported_artist     TEXT,
    reported_writer     TEXT,
    
    -- Resolved references (set during reconciliation)
    matched_recording_id UUID REFERENCES recordings(id),
    matched_work_id      UUID REFERENCES works(id),
    match_confidence     NUMERIC(5,2),   -- 0-100% match confidence
    match_method         VARCHAR(30),    -- isrc, iswc, title_artist, manual
    
    -- Financial data
    territory_code      VARCHAR(4),
    use_type            VARCHAR(50) NOT NULL,
        -- stream, download, ringtone, broadcast_radio, broadcast_tv,
        -- public_performance, sync_fee, mechanical, neighboring,
        -- user_generated_content, social_media, fitness, gaming
    units               BIGINT,          -- streams, downloads, plays
    unit_rate           NUMERIC(12,8),   -- per-unit rate from DSP
    gross_amount        NUMERIC(14,4) NOT NULL,
    currency_code       VARCHAR(3) NOT NULL,
    usage_period_start  DATE,
    usage_period_end    DATE,
    sub_source          VARCHAR(100),    -- sub-platform or tier
    
    -- Processing
    status              VARCHAR(20) NOT NULL DEFAULT 'unmatched',
        -- unmatched, matched, disputed, excluded
    anomaly_flags       TEXT[],          -- array of anomaly codes
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_stmt_line UNIQUE (batch_id, line_number)
);

CREATE INDEX idx_sl_batch ON statement_lines (batch_id);
CREATE INDEX idx_sl_isrc ON statement_lines (reported_isrc) WHERE reported_isrc IS NOT NULL;
CREATE INDEX idx_sl_iswc ON statement_lines (reported_iswc) WHERE reported_iswc IS NOT NULL;
CREATE INDEX idx_sl_recording ON statement_lines (matched_recording_id);
CREATE INDEX idx_sl_work ON statement_lines (matched_work_id);
CREATE INDEX idx_sl_territory ON statement_lines (territory_code);
CREATE INDEX idx_sl_use_type ON statement_lines (use_type);
CREATE INDEX idx_sl_status ON statement_lines (status);
```

### fx_rates (Daily foreign exchange rates)

```sql
CREATE TABLE fx_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_date       DATE NOT NULL,
    from_currency   VARCHAR(3) NOT NULL,  -- ISO 4217
    to_currency     VARCHAR(3) NOT NULL,  -- ISO 4217
    rate            NUMERIC(16,8) NOT NULL,
    source          VARCHAR(50) NOT NULL,  -- "ecb", "openexchangerates", "manual"
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_fx_rate UNIQUE (rate_date, from_currency, to_currency)
);

CREATE INDEX idx_fx_date ON fx_rates (rate_date);
CREATE INDEX idx_fx_pair ON fx_rates (from_currency, to_currency);
```

### royalty_calculations (Calculated royalty amounts per rights holder)

```sql
CREATE TABLE royalty_calculations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_line_id   UUID NOT NULL REFERENCES statement_lines(id),
    work_share_id       UUID REFERENCES work_shares(id),
    recording_share_id  UUID REFERENCES recording_shares(id),
    party_id            UUID NOT NULL REFERENCES parties(id),
    
    -- Calculation inputs
    gross_amount        NUMERIC(14,4) NOT NULL,
    original_currency   VARCHAR(3) NOT NULL,
    fx_rate_id          UUID REFERENCES fx_rates(id),
    converted_amount    NUMERIC(14,4) NOT NULL,  -- in settlement currency
    settlement_currency VARCHAR(3) NOT NULL,
    
    -- Calculation outputs
    share_percentage    NUMERIC(7,4) NOT NULL,
    rights_type         VARCHAR(30) NOT NULL,
        -- performance, mechanical, sync, master, neighboring, print
    royalty_amount      NUMERIC(14,4) NOT NULL,  -- party's calculated share
    
    -- Deductions
    admin_fee_pct       NUMERIC(5,2) DEFAULT 0,
    admin_fee_amount    NUMERIC(14,4) DEFAULT 0,
    withholding_tax_pct NUMERIC(5,2) DEFAULT 0,
    withholding_amount  NUMERIC(14,4) DEFAULT 0,
    net_amount          NUMERIC(14,4) NOT NULL,   -- final payable amount
    
    -- Recoupment
    contract_id         UUID REFERENCES contracts(id),
    recoupable          BOOLEAN NOT NULL DEFAULT FALSE,
    recouped_amount     NUMERIC(14,4) DEFAULT 0,
    payable_amount      NUMERIC(14,4) NOT NULL,   -- net minus recouped
    
    calculation_period  DATERANGE,
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    calculated_by       VARCHAR(50),  -- "engine_v1", "manual_override"

    CONSTRAINT chk_calc_target CHECK (
        work_share_id IS NOT NULL OR recording_share_id IS NOT NULL
    )
);

CREATE INDEX idx_rc_stmt_line ON royalty_calculations (statement_line_id);
CREATE INDEX idx_rc_party ON royalty_calculations (party_id);
CREATE INDEX idx_rc_period ON royalty_calculations USING gist (calculation_period);
CREATE INDEX idx_rc_work_share ON royalty_calculations (work_share_id);
CREATE INDEX idx_rc_recording_share ON royalty_calculations (recording_share_id);
```

### payment_runs

```sql
CREATE TABLE payment_runs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_date            DATE NOT NULL,
    period_label        VARCHAR(50),       -- "Q1 2026", "January 2026"
    settlement_currency VARCHAR(3) NOT NULL,
    total_amount        NUMERIC(16,4) NOT NULL,
    total_payees        INTEGER NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, approved, processing, completed, failed, cancelled
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id)
);

CREATE INDEX idx_pr_status ON payment_runs (status);
CREATE INDEX idx_pr_date ON payment_runs (run_date);
```

### payment_run_items

```sql
CREATE TABLE payment_run_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_run_id      UUID NOT NULL REFERENCES payment_runs(id) ON DELETE CASCADE,
    party_id            UUID NOT NULL REFERENCES parties(id),
    bank_account_id     UUID REFERENCES party_bank_accounts(id),
    total_amount        NUMERIC(14,4) NOT NULL,
    currency_code       VARCHAR(3) NOT NULL,
    fx_rate_id          UUID REFERENCES fx_rates(id),
    payment_method      VARCHAR(30),
    payment_reference   VARCHAR(100),    -- external payment ID
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, processing, sent, confirmed, failed, returned
    sent_at             TIMESTAMPTZ,
    confirmed_at        TIMESTAMPTZ,
    error_message       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pri_run ON payment_run_items (payment_run_id);
CREATE INDEX idx_pri_party ON payment_run_items (party_id);
CREATE INDEX idx_pri_status ON payment_run_items (status);
```

### payment_run_calculations (Links calculations to payment items)

```sql
CREATE TABLE payment_run_calculations (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_run_item_id     UUID NOT NULL REFERENCES payment_run_items(id) ON DELETE CASCADE,
    royalty_calculation_id   UUID NOT NULL REFERENCES royalty_calculations(id),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_prc UNIQUE (payment_run_item_id, royalty_calculation_id)
);

CREATE INDEX idx_prc_item ON payment_run_calculations (payment_run_item_id);
CREATE INDEX idx_prc_calc ON payment_run_calculations (royalty_calculation_id);
```

---

## Module 6: PRO Registration

### pro_registrations (CWR submissions to collecting societies)

```sql
CREATE TABLE pro_registrations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(id),
    pro_party_id        UUID NOT NULL REFERENCES parties(id),  -- ASCAP, BMI, PRS etc.
    registration_type   VARCHAR(10) NOT NULL,  -- NWR (new), REV (revision)
    cwr_version         VARCHAR(5) DEFAULT '2.2',
    submission_file     TEXT,           -- path to CWR file
    submission_date     DATE,
    transaction_id      VARCHAR(20),    -- CWR transaction sequence
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, submitted, acknowledged, accepted, rejected, conflict
    acknowledgement_date DATE,
    acknowledgement_file TEXT,
    rejection_reason    TEXT,
    society_work_id     VARCHAR(30),    -- ID assigned by the society
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reg_work ON pro_registrations (work_id);
CREATE INDEX idx_reg_pro ON pro_registrations (pro_party_id);
CREATE INDEX idx_reg_status ON pro_registrations (status);
CREATE INDEX idx_reg_submission ON pro_registrations (submission_date);
```

---

## Module 7: System & Audit

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   TEXT,
    display_name    TEXT NOT NULL,
    party_id        UUID REFERENCES parties(id),   -- linked rights holder
    role            VARCHAR(30) NOT NULL DEFAULT 'viewer',
        -- admin, manager, accounting, publisher, writer, viewer
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_party ON users (party_id);
CREATE INDEX idx_users_role ON users (role);
```

### audit_log (Immutable audit trail)

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(50) NOT NULL,
        -- share_created, share_updated, share_superseded,
        -- statement_ingested, calculation_run, payment_approved,
        -- payment_sent, contract_created, contract_modified,
        -- registration_submitted, registration_acknowledged,
        -- user_login, permission_change
    entity_type     VARCHAR(50) NOT NULL,  -- works, recordings, work_shares, etc.
    entity_id       UUID NOT NULL,
    actor_id        UUID REFERENCES users(id),
    actor_ip        INET,
    old_values      JSONB,
    new_values      JSONB,
    metadata        JSONB,     -- additional context
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Append-only: no UPDATE or DELETE allowed (enforced via application + RLS)
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_event ON audit_log (event_type);
CREATE INDEX idx_audit_actor ON audit_log (actor_id);
CREATE INDEX idx_audit_time ON audit_log (created_at);
CREATE INDEX idx_audit_metadata ON audit_log USING gin (metadata);
```

### territories (Reference table)

```sql
CREATE TABLE territories (
    code            VARCHAR(4) PRIMARY KEY,  -- ISO 3166-1 alpha-2 or CISAC territory
    name            TEXT NOT NULL,
    region          VARCHAR(50),
    cisac_code      VARCHAR(4),     -- CISAC numeric territory code
    currency_code   VARCHAR(3),     -- default currency for this territory
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);
```

### use_types (Reference table)

```sql
CREATE TABLE use_types (
    code            VARCHAR(50) PRIMARY KEY,
    name            TEXT NOT NULL,
    category        VARCHAR(30) NOT NULL,
        -- digital, broadcast, live, print, sync, other
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);
```

---

## Key Views

### v_active_work_shares (Current effective shares only)

```sql
CREATE VIEW v_active_work_shares AS
SELECT
    ws.*,
    p.legal_name AS party_name,
    p.party_type,
    p.ipi_name_number,
    w.title AS work_title,
    w.iswc
FROM work_shares ws
JOIN parties p ON p.id = ws.party_id
JOIN works w ON w.id = ws.work_id
WHERE ws.effective_to IS NULL
  AND ws.superseded_by IS NULL;
```

### v_work_share_totals (Validate shares sum to 100%)

```sql
CREATE VIEW v_work_share_totals AS
SELECT
    work_id,
    SUM(performance_share) AS total_performance,
    SUM(mechanical_share) AS total_mechanical,
    SUM(sync_share) AS total_sync,
    SUM(print_share) AS total_print,
    COUNT(*) AS share_count
FROM work_shares
WHERE effective_to IS NULL AND superseded_by IS NULL
GROUP BY work_id;
```

### v_royalty_summary (Revenue dashboard)

```sql
CREATE VIEW v_royalty_summary AS
SELECT
    rc.party_id,
    p.legal_name AS party_name,
    sl.territory_code,
    sl.use_type,
    rc.rights_type,
    rc.settlement_currency,
    date_trunc('month', lower(rc.calculation_period)) AS period_month,
    SUM(rc.gross_amount) AS total_gross,
    SUM(rc.royalty_amount) AS total_royalty,
    SUM(rc.net_amount) AS total_net,
    SUM(rc.payable_amount) AS total_payable,
    COUNT(*) AS line_count
FROM royalty_calculations rc
JOIN statement_lines sl ON sl.id = rc.statement_line_id
JOIN parties p ON p.id = rc.party_id
GROUP BY rc.party_id, p.legal_name, sl.territory_code, sl.use_type,
         rc.rights_type, rc.settlement_currency,
         date_trunc('month', lower(rc.calculation_period));
```

---

## Pros and Cons

### Pros

1. **Strong data integrity.** Foreign keys, check constraints, and unique indexes prevent orphaned records, invalid shares, and duplicate identifiers. The many-to-many relationships between works, recordings, parties, territories, and contracts are modeled with explicit junction tables, making every relationship queryable and enforceable.

2. **Standards alignment.** The schema directly maps to CWR record types (NWR/REV -> works, SPU/SPT -> work_shares with territories, SWR/SWT -> writer shares) and DDEX entity concepts (MusicalWork, SoundRecording, RightsController). Generating CWR files from this schema is a straightforward SELECT + format operation.

3. **Auditability.** Version-controlled shares (effective_from/effective_to, superseded_by) preserve full ownership history without overwriting records. The separate audit_log table captures every mutation with old/new values, satisfying the immutable audit trail requirement.

4. **Battle-tested tooling.** PostgreSQL's ecosystem provides mature tools for backup/restore (pg_dump, WAL archiving), replication (streaming replication, logical replication), monitoring (pg_stat), and migration (Flyway, Alembic, Prisma Migrate). Every ORM and framework supports PostgreSQL natively.

5. **Regulatory compliance.** DATERANGE types with GiST indexes enable efficient period-based queries for territory-specific rights validity. The schema can enforce GDPR data minimization through column-level encryption (pgcrypto) and row-level security policies.

6. **Query flexibility.** Complex royalty reports joining statements, shares, territories, and FX rates across multiple periods are standard SQL operations. No impedance mismatch with BI tools, reporting frameworks, or ad-hoc analysis.

### Cons

1. **Many-to-many complexity.** The heavily normalized design requires numerous JOINs for common queries (e.g., "show me all active shares for this work across all territories with their calculated royalties"). A single royalty dashboard query might join 6-8 tables. This is manageable with proper indexing but increases query complexity and maintenance burden.

2. **Territory explosion.** A work distributed in 200+ territories with different splits per territory creates hundreds of rows in work_share_territories for a single work. Multiplied across thousands of works, this table grows very large. CISAC territory codes add another dimension beyond ISO country codes.

3. **Rigid schema evolution.** Adding new rights types (e.g., a future "AI training data" right), new use types, or new royalty calculation models requires DDL changes and migration scripts. The normalized structure makes schema changes more surgical but also more disruptive than document-oriented approaches.

4. **Statement ingestion performance.** Bulk loading millions of statement lines per quarter with real-time matching against the catalog requires careful use of COPY, staging tables, and batch processing. The foreign key checks on matched_recording_id and matched_work_id can slow bulk inserts if not managed with deferred constraints.

5. **No native graph traversal.** Answering questions like "find all parties connected to this work through any chain of contracts, sub-publishing agreements, and administration deals" requires recursive CTEs, which are slower and harder to write than graph database traversals.

6. **Split validation complexity.** Ensuring that total shares across all parties for a work sum to exactly 100% (or 200% in the case of writer/publisher split convention) requires application-level validation or complex database triggers, since CHECK constraints cannot span multiple rows.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with `pg_trgm`, `btree_gist`, and `pgcrypto` extensions |
| **Connection pooling** | PgBouncer in transaction mode for high-concurrency API workloads |
| **Migrations** | Flyway or Alembic for version-controlled DDL changes |
| **Full-text search** | PostgreSQL `tsvector` + GIN indexes for catalog search; Elasticsearch if sub-50ms faceted search is needed |
| **Bulk ingestion** | PostgreSQL `COPY` with staging tables; deferred FK constraints during batch loads |
| **Encryption** | `pgcrypto` for column-level encryption of bank account numbers and tax IDs |
| **Row-level security** | PostgreSQL RLS policies to enforce per-tenant, per-role data visibility |
| **Backup** | WAL archiving to S3/GCS with point-in-time recovery (PITR) |
| **Monitoring** | pg_stat_statements + Prometheus/Grafana for query performance |
| **ORM** | Prisma, SQLAlchemy, or TypeORM depending on application language |

---

## Migration and Scaling Considerations

### Initial Deployment

For a mid-market platform managing thousands of works and tens of thousands of recordings, a single PostgreSQL instance with 8-16 GB RAM and SSD storage handles the workload comfortably. Statement ingestion of 1-5 million lines per quarter is well within single-node capacity.

### Scaling Path

1. **Read replicas** -- Add streaming replication replicas for dashboard queries and reporting workloads, keeping the primary for writes (statement ingestion, royalty calculations, payment processing).

2. **Table partitioning** -- Partition `statement_lines` and `royalty_calculations` by statement period (RANGE partitioning on the date column) once these tables exceed 50-100 million rows. This enables efficient partition pruning for period-based queries and simplifies data retention (drop old partitions).

3. **Materialized views** -- Pre-compute `v_royalty_summary` as a materialized view refreshed after each calculation run, avoiding repeated aggregation of millions of rows for dashboard queries.

4. **Connection pooling** -- PgBouncer is essential once concurrent API connections exceed ~100, as PostgreSQL's process-per-connection model does not scale gracefully beyond a few hundred connections.

5. **Statement ingestion pipeline** -- Move from synchronous ingestion to an async pipeline: files land in object storage, a worker picks them up, loads into staging tables via COPY, runs matching/validation, and promotes to production tables. This avoids blocking the API during large imports.

6. **Sharding** -- If the platform grows to serve major-label-scale catalogs (millions of works, billions of statement lines), consider Citus (distributed PostgreSQL) for horizontal sharding by tenant (label/publisher) or by work ID.

### Data Retention

Statement lines and royalty calculations should be retained for a minimum of 7 years (common audit requirement in music publishing). Partitioned tables make archival straightforward: detach old partitions, compress, and move to cold storage (S3 Glacier) while keeping the data queryable via foreign data wrappers if needed.

### Migration from Existing Systems

Most independent publishers are migrating from spreadsheets or disconnected tools. The schema supports bulk import via CSV-to-COPY pipelines. Key migration challenges:

- **Identifier cleanup** -- Many catalogs have missing or inconsistent ISRCs and ISWCs that need resolution before import.
- **Historical split reconstruction** -- If the source system overwrote splits rather than versioning them, historical share data may be unrecoverable.
- **Currency normalization** -- Historical royalty data from multiple sources may use different FX rates; the migration must decide whether to re-convert using consistent rates or preserve original conversions.
