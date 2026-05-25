# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Music Rights Management · Candidate #441 · Generated: 2026-05-25

---

## Overview

This model combines the strengths of normalized relational tables for stable, well-defined entities with PostgreSQL JSONB columns for data that is inherently variable, semi-structured, or evolves frequently. The key insight driving this design is that music rights data splits naturally into two categories:

**Stable, well-structured data** (best served by normalized columns):
- Core entities: works, recordings, releases, parties
- Industry identifiers: ISWC, ISRC, IPI, UPC
- Financial records: royalty calculations, payment runs, FX rates
- System data: users, roles, audit entries

**Variable, semi-structured data** (best served by JSONB):
- Territory-specific split configurations (vary per deal, per society, per time period)
- Statement line metadata (different DSPs report different fields)
- Contract terms (infinite variation across deal types)
- CWR/DDEX message payloads (complex nested structures defined by external standards)
- Parser configurations (per-source column mappings)
- Anomaly detection results (evolving ML model outputs)

By using JSONB for the variable parts, we avoid the "table explosion" problem of the fully normalized model (where territory splits, use-type overrides, and parser mappings each need their own junction tables) while retaining relational integrity for the core domain.

---

## Schema Design

### Module 1: Catalog (Relational Core + JSONB Metadata)

#### works

```sql
CREATE TABLE works (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               TEXT NOT NULL,
    iswc                VARCHAR(15),
    work_type           VARCHAR(50) NOT NULL DEFAULT 'MusicalWork',
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    -- JSONB for variable/extensible metadata
    metadata            JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    metadata schema:
    {
        "alternate_titles": ["Title Variant 1", "Title Variant 2"],
        "language_code": "en",
        "duration_seconds": 214,
        "genre": "Pop",
        "sub_genre": "Synth Pop",
        "mood": ["melancholic", "atmospheric"],
        "tempo_bpm": 120,
        "key_signature": "A minor",
        "ai_contribution": {
            "is_ai": false,
            "ai_percentage": 0,
            "ai_tool": null,
            "human_elements": ["melody", "lyrics", "arrangement"]
        },
        "creation_date": "2026-01-10",
        "registration_notes": "Co-written during Nashville session",
        "custom_fields": {
            "internal_catalog_code": "PUB-2026-0042",
            "priority": "high"
        }
    }
    */

    -- CWR/DDEX export cache
    cwr_data            JSONB,
    /*
    cwr_data schema:
    {
        "cwr_version": "2.2",
        "nwr_record": {
            "work_title": "Midnight Rain",
            "language_code": "EN",
            "duration": "000334",
            "recorded_indicator": "Y",
            "text_music_relationship": "MTX",
            "composite_type": null,
            "version_type": "ORI",
            "music_arrangement": "ORI",
            "lyric_adaptation": "ORI"
        },
        "alt_records": [
            {"alternate_title": "Midnight Rain (Acoustic)", "title_type": "AT"}
        ]
    }
    */

    CONSTRAINT chk_iswc_format CHECK (
        iswc IS NULL OR iswc ~ '^T-[0-9]{9}-[0-9]$'
    )
);

CREATE UNIQUE INDEX idx_works_iswc ON works (iswc) WHERE iswc IS NOT NULL;
CREATE INDEX idx_works_status ON works (status);
CREATE INDEX idx_works_title ON works USING gin (title gin_trgm_ops);
CREATE INDEX idx_works_metadata ON works USING gin (metadata);
CREATE INDEX idx_works_genre ON works ((metadata->>'genre'));
CREATE INDEX idx_works_ai ON works ((metadata->'ai_contribution'->>'is_ai'))
    WHERE (metadata->'ai_contribution'->>'is_ai') = 'true';
```

#### recordings

```sql
CREATE TABLE recordings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title               TEXT NOT NULL,
    isrc                VARCHAR(12),
    recording_type      VARCHAR(30) NOT NULL DEFAULT 'SoundRecording',
    primary_artist_id   UUID REFERENCES parties(id),
    label_id            UUID REFERENCES parties(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    -- JSONB for variable recording metadata
    metadata            JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    metadata schema:
    {
        "duration_seconds": 214,
        "recording_date": "2026-01-10",
        "release_date": "2026-03-15",
        "catalog_number": "IND-2026-001",
        "audio_format": "WAV",
        "sample_rate": 48000,
        "bit_depth": 24,
        "is_ai_generated": false,
        "studio": "Blackbird Studio, Nashville",
        "producer": "Producer Name",
        "mixer": "Mixer Name",
        "mastering_engineer": "Engineer Name",
        "credits": [
            {"name": "Session Musician", "role": "guitar", "ipn": "12345678"}
        ],
        "ddex_data": {
            "ern_version": "4.3",
            "pline": "(P) 2026 Independent Records",
            "cline": "(C) 2026 Independent Records",
            "parental_warning": "NotExplicit"
        }
    }
    */

    -- DDEX ERN message cache for distribution
    ddex_ern_data       JSONB,

    CONSTRAINT chk_isrc_format CHECK (
        isrc IS NULL OR isrc ~ '^[A-Z]{2}[A-Z0-9]{3}[0-9]{7}$'
    )
);

CREATE UNIQUE INDEX idx_recordings_isrc ON recordings (isrc) WHERE isrc IS NOT NULL;
CREATE INDEX idx_recordings_artist ON recordings (primary_artist_id);
CREATE INDEX idx_recordings_label ON recordings (label_id);
CREATE INDEX idx_recordings_metadata ON recordings USING gin (metadata);
```

#### work_recordings

```sql
CREATE TABLE work_recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id         UUID NOT NULL REFERENCES works(id) ON DELETE CASCADE,
    recording_id    UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    relationship    VARCHAR(30) NOT NULL DEFAULT 'performance',
    is_primary      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_work_recording UNIQUE (work_id, recording_id)
);

CREATE INDEX idx_wr_work ON work_recordings (work_id);
CREATE INDEX idx_wr_recording ON work_recordings (recording_id);
```

#### releases

```sql
CREATE TABLE releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           TEXT NOT NULL,
    upc             VARCHAR(14),
    release_type    VARCHAR(30) NOT NULL DEFAULT 'single',
    release_date    DATE,
    label_id        UUID REFERENCES parties(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Variable release metadata
    metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    {
        "catalog_number": "IND-2026-001",
        "territory_code": "WW",
        "artwork_url": "s3://catalog/artwork/ind-2026-001.jpg",
        "genre": "Pop",
        "sub_genre": "Indie Pop",
        "label_copy": "A debut collection exploring themes of...",
        "ddex_release_data": { ... }
    }
    */

    CONSTRAINT chk_upc CHECK (upc IS NULL OR upc ~ '^[0-9]{12,14}$')
);

CREATE UNIQUE INDEX idx_releases_upc ON releases (upc) WHERE upc IS NOT NULL;
```

#### release_tracks

```sql
CREATE TABLE release_tracks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    release_id      UUID NOT NULL REFERENCES releases(id) ON DELETE CASCADE,
    recording_id    UUID NOT NULL REFERENCES recordings(id),
    disc_number     SMALLINT NOT NULL DEFAULT 1,
    track_number    SMALLINT NOT NULL,

    CONSTRAINT uq_release_track UNIQUE (release_id, disc_number, track_number)
);
```

---

### Module 2: Parties (Relational Core + JSONB Identifiers)

#### parties

```sql
CREATE TABLE parties (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_type          VARCHAR(30) NOT NULL,
    legal_name          TEXT NOT NULL,
    display_name        TEXT,
    ipi_name_number     VARCHAR(11),
    country_code        VARCHAR(2),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    pro_affiliation_id  UUID REFERENCES parties(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- All identifiers in a single JSONB column
    identifiers         JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    {
        "ipi_base_number": "I-000000123-4",
        "ipn": "12345678",
        "isni": "0000 0001 2345 6789",
        "mbid": "a1b2c3d4-e5f6-...",
        "pro_member_ids": {
            "ASCAP": "ASC-123456",
            "BMI": "BMI-789012",
            "PRS": "PRS-345678"
        },
        "tax_ids": {
            "US": {"type": "SSN", "value": "encrypted:..."},
            "GB": {"type": "UTR", "value": "encrypted:..."}
        }
    }
    */

    -- Contact and payment details
    contact_info        JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    {
        "email": "artist@example.com",
        "phone": "+1-555-0123",
        "address": {
            "line_1": "123 Music Row",
            "city": "Nashville",
            "state": "TN",
            "postal_code": "37203",
            "country": "US"
        },
        "bank_accounts": [
            {
                "account_name": "Artist A",
                "currency": "USD",
                "bank_name": "First Bank",
                "routing_number": "encrypted:...",
                "account_number": "encrypted:...",
                "is_primary": true,
                "is_verified": true
            },
            {
                "account_name": "Artist A Ltd",
                "currency": "GBP",
                "bank_name": "Barclays",
                "iban": "encrypted:...",
                "swift_bic": "BARCGB22",
                "is_primary": false,
                "is_verified": true
            }
        ],
        "payment_preferences": {
            "preferred_method": "bank_transfer",
            "minimum_payout": 50.00,
            "stripe_connect_id": "acct_1abc...",
            "paypal_email": null
        }
    }
    */

    CONSTRAINT chk_ipi_format CHECK (
        ipi_name_number IS NULL OR ipi_name_number ~ '^[0-9]{9,11}$'
    )
);

CREATE UNIQUE INDEX idx_parties_ipi ON parties (ipi_name_number)
    WHERE ipi_name_number IS NOT NULL;
CREATE INDEX idx_parties_type ON parties (party_type);
CREATE INDEX idx_parties_name ON parties USING gin (legal_name gin_trgm_ops);
CREATE INDEX idx_parties_identifiers ON parties USING gin (identifiers);
CREATE INDEX idx_parties_pro_members ON parties USING gin ((identifiers->'pro_member_ids'));
```

---

### Module 3: Rights & Splits (Hybrid Approach)

This is where the hybrid model shines. The core split record uses relational columns for the primary share percentages and standard fields, while territory-specific overrides and use-type variations are stored as JSONB -- eliminating the need for separate junction tables that would otherwise explode in row count.

#### work_shares

```sql
CREATE TABLE work_shares (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(id) ON DELETE CASCADE,
    party_id            UUID NOT NULL REFERENCES parties(id),
    role                VARCHAR(30) NOT NULL,

    -- Primary share percentages (relational -- most queried)
    performance_share   NUMERIC(7,4) NOT NULL DEFAULT 0,
    mechanical_share    NUMERIC(7,4) NOT NULL DEFAULT 0,
    sync_share          NUMERIC(7,4) NOT NULL DEFAULT 0,
    print_share         NUMERIC(7,4) NOT NULL DEFAULT 0,
    controlled          BOOLEAN NOT NULL DEFAULT FALSE,
    publisher_party_id  UUID REFERENCES parties(id),

    -- Versioning (relational for efficient temporal queries)
    version             INTEGER NOT NULL DEFAULT 1,
    effective_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to        DATE,
    superseded_by       UUID REFERENCES work_shares(id),

    -- Territory overrides, use-type specifics, and collection details (JSONB)
    territory_config    JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    territory_config schema:
    {
        "default_territories": "worldwide",    -- or array of codes
        "excluded_territories": ["CN", "RU"],
        "territory_overrides": {
            "US": {
                "performance_share": 30.00,
                "mechanical_share": 30.00,
                "collection_society": "ASCAP",
                "society_work_id": "ASC-890123"
            },
            "GB": {
                "performance_share": 25.00,
                "mechanical_share": 25.00,
                "collection_society": "PRS",
                "sub_publisher_id": "subpub-uk-..."
            },
            "DE": {
                "performance_share": 20.00,
                "mechanical_share": 20.00,
                "collection_society": "GEMA",
                "sub_publisher_id": "subpub-de-...",
                "sub_publisher_share": 15.00
            }
        },
        "use_type_overrides": {
            "sync": {
                "share": 50.00,
                "notes": "Higher sync share per co-pub agreement"
            },
            "user_generated_content": {
                "share": 10.00,
                "notes": "Reduced UGC rate per contract addendum"
            }
        },
        "collection_config": {
            "mechanical_collection": "at_source",
            "performance_collection": "at_source",
            "direct_deal_territories": ["US", "CA", "GB"]
        }
    }
    */

    -- CWR export data for this share (pre-computed)
    cwr_share_data      JSONB,
    /*
    {
        "spu_record": {
            "publisher_name": "Independent Music Publishing",
            "publisher_ipi": "00123456789",
            "publisher_capacity": "E",
            "pr_ownership_share": "05000",
            "mr_ownership_share": "05000"
        },
        "spt_records": [
            {"territory": "US", "pr_collection_share": "05000", "mr_collection_share": "05000"},
            {"territory": "GB", "pr_collection_share": "02500", "mr_collection_share": "02500"}
        ]
    }
    */

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
CREATE INDEX idx_ws_active ON work_shares (work_id)
    WHERE effective_to IS NULL AND superseded_by IS NULL;
CREATE INDEX idx_ws_territory ON work_shares USING gin (territory_config);
CREATE INDEX idx_ws_territory_overrides ON work_shares
    USING gin ((territory_config->'territory_overrides'));
```

#### recording_shares

```sql
CREATE TABLE recording_shares (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recording_id        UUID NOT NULL REFERENCES recordings(id) ON DELETE CASCADE,
    party_id            UUID NOT NULL REFERENCES parties(id),
    role                VARCHAR(30) NOT NULL,

    -- Primary shares (relational)
    master_share        NUMERIC(7,4) NOT NULL DEFAULT 0,
    neighboring_share   NUMERIC(7,4) NOT NULL DEFAULT 0,

    -- Versioning
    version             INTEGER NOT NULL DEFAULT 1,
    effective_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to        DATE,
    superseded_by       UUID REFERENCES recording_shares(id),

    -- Territory and use-type configuration (JSONB)
    territory_config    JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    Same structure as work_shares.territory_config but with
    master_share and neighboring_share instead of performance/mechanical.
    */

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
CREATE INDEX idx_rs_active ON recording_shares (recording_id)
    WHERE effective_to IS NULL AND superseded_by IS NULL;
```

---

### Module 4: Contracts (JSONB-Heavy)

Contracts are the most variable entity in the domain. Each contract type has radically different terms and structures. The hybrid approach uses relational columns for the universal fields (type, dates, status) and JSONB for the type-specific terms.

#### contracts

```sql
CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_type       VARCHAR(50) NOT NULL,
    title               TEXT NOT NULL,
    contract_number     VARCHAR(50),
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    effective_date      DATE,
    expiry_date         DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    -- Parties involved (JSONB array)
    parties             JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*
    [
        {"party_id": "pub123-...", "role": "publisher", "name": "Indie Publishing"},
        {"party_id": "wrt456-...", "role": "writer", "name": "Songwriter A"}
    ]
    */

    -- Territory scope (JSONB)
    territories         JSONB NOT NULL DEFAULT '{"scope": "worldwide"}'::jsonb,
    /*
    {"scope": "worldwide"} OR
    {"scope": "specific", "include": ["US", "CA", "GB", "AU"]} OR
    {"scope": "worldwide_except", "exclude": ["CN", "RU"]}
    */

    -- Works/recordings covered (JSONB)
    covered_works       JSONB NOT NULL DEFAULT '{"scope": "specific", "items": []}'::jsonb,
    /*
    {"scope": "specific", "items": [
        {"work_id": "w1-...", "title": "Midnight Rain"},
        {"work_id": "w2-...", "title": "Dawn"}
    ]} OR
    {"scope": "catalog", "label": "Indie Records"} OR
    {"scope": "future_works", "from_date": "2026-01-01"}
    */

    -- Contract-type-specific terms (JSONB -- the core flexibility)
    terms               JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    -- For co-publishing agreements:
    {
        "advance": {
            "amount": 50000.00,
            "currency": "USD",
            "recoupable": true,
            "recouped_to_date": 12500.00,
            "recoupment_rate": 100
        },
        "royalty_rates": {
            "mechanical": {
                "writer_share": 50.00,
                "publisher_share": 50.00,
                "co_publisher_split": {
                    "publisher_a": 75.00,
                    "publisher_b": 25.00
                }
            },
            "performance": {
                "writer_share": 50.00,
                "publisher_share": 50.00
            },
            "sync": {
                "split": 50.00,
                "minimum_fee": 5000.00,
                "approval_required": true
            }
        },
        "option_periods": [
            {
                "period_number": 1,
                "start_date": "2026-01-01",
                "end_date": "2027-12-31",
                "exercise_deadline": "2027-09-30",
                "status": "exercised",
                "advance": 25000.00
            },
            {
                "period_number": 2,
                "start_date": "2028-01-01",
                "end_date": "2029-12-31",
                "exercise_deadline": "2029-09-30",
                "status": "pending",
                "advance": 30000.00
            }
        ],
        "reversion_clauses": [
            {
                "trigger_type": "out_of_print",
                "condition": "If no commercial release available for 12 consecutive months",
                "reversion_scope": "full",
                "notice_period_days": 90
            },
            {
                "trigger_type": "recoupment",
                "condition": "Full reversion after advance fully recouped",
                "reversion_scope": "partial",
                "retained_rights": ["sync"]
            }
        ],
        "auto_renew": false,
        "governing_law": "New York, USA",
        "delivery_commitment": {
            "minimum_works": 12,
            "delivered_works": 5,
            "period": "per_option_period"
        }
    }

    -- For sync licences:
    {
        "licence_fee": {
            "amount": 15000.00,
            "currency": "USD",
            "paid": true,
            "paid_date": "2026-04-15"
        },
        "usage": {
            "media_type": "film",
            "production_title": "Summer Nights",
            "scene_description": "End credits",
            "duration_seconds": 90,
            "use_type": "background"
        },
        "scope": {
            "territories": "worldwide",
            "term": "in_perpetuity",
            "platforms": ["theatrical", "streaming", "broadcast"]
        },
        "master_licence_required": true,
        "master_licence_party_id": "lab789-..."
    }
    */

    -- AI-extracted contract insights
    ai_analysis         JSONB,
    /*
    {
        "extracted_at": "2026-03-15T10:00:00Z",
        "model_version": "contract-analyzer-v2",
        "key_dates": [
            {"type": "option_deadline", "date": "2027-09-30", "confidence": 0.95},
            {"type": "expiry", "date": "2031-12-31", "confidence": 0.98}
        ],
        "key_terms": [
            {"term": "advance", "value": "$50,000", "confidence": 0.92},
            {"term": "territory", "value": "Worldwide", "confidence": 0.88}
        ],
        "risk_flags": [
            {"flag": "no_audit_clause", "severity": "medium"}
        ]
    }
    */

    -- Document storage reference
    documents           JSONB DEFAULT '[]'::jsonb
    /*
    [
        {
            "file_name": "co-pub-agreement-signed.pdf",
            "file_path": "s3://contracts/2026/co-pub-001.pdf",
            "file_size": 245000,
            "uploaded_at": "2026-01-15",
            "document_type": "signed_agreement"
        }
    ]
    */
);

CREATE INDEX idx_contracts_type ON contracts (contract_type);
CREATE INDEX idx_contracts_status ON contracts (status);
CREATE INDEX idx_contracts_expiry ON contracts (expiry_date);
CREATE INDEX idx_contracts_parties ON contracts USING gin (parties);
CREATE INDEX idx_contracts_terms ON contracts USING gin (terms);
CREATE INDEX idx_contracts_territories ON contracts USING gin (territories);
CREATE INDEX idx_contracts_option_deadlines ON contracts
    USING gin ((terms->'option_periods'));
```

---

### Module 5: Statement Ingestion (JSONB-Native)

Statement ingestion is the most heterogeneous part of the system. Every DSP, distributor, and PRO sends statements in different formats with different columns. The hybrid model uses relational columns for the universal financial fields and JSONB for the source-specific raw data.

#### statement_sources (Parser configuration per DSP/distributor)

```sql
CREATE TABLE statement_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_name         VARCHAR(100) NOT NULL UNIQUE,
    source_type         VARCHAR(30) NOT NULL,
    party_id            UUID REFERENCES parties(id),

    -- Parser configuration stored as JSONB
    parser_config       JSONB NOT NULL,
    /*
    {
        "file_format": "csv",
        "encoding": "utf-8",
        "delimiter": ",",
        "has_header": true,
        "skip_rows": 0,
        "column_mapping": {
            "isrc": {"column": "ISRC", "transform": "strip_dashes"},
            "title": {"column": "Track Title", "transform": "trim"},
            "artist": {"column": "Artist Name", "transform": "trim"},
            "territory": {"column": "Country Code", "transform": "iso_alpha2"},
            "use_type": {"column": "Sale Type",
                         "value_map": {
                             "Stream": "stream",
                             "Download": "download",
                             "Ad Supported Stream": "stream_ad",
                             "Premium Stream": "stream_premium"
                         }},
            "units": {"column": "Quantity", "transform": "to_integer"},
            "unit_rate": {"column": "Per Unit Rate", "transform": "to_decimal"},
            "gross_amount": {"column": "Net Amount", "transform": "to_decimal"},
            "currency": {"column": "Currency", "transform": "uppercase"},
            "period_start": {"column": "Start Date", "format": "YYYY-MM-DD"},
            "period_end": {"column": "End Date", "format": "YYYY-MM-DD"}
        },
        "validation_rules": [
            {"field": "gross_amount", "rule": "non_negative"},
            {"field": "units", "rule": "non_negative"},
            {"field": "territory", "rule": "valid_iso_country"}
        ],
        "dedup_keys": ["isrc", "territory", "use_type", "period_start"],
        "currency_default": "USD",
        "notes": "Spotify's standard quarterly statement format as of Q1 2026"
    }
    */

    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### statement_batches

```sql
CREATE TABLE statement_batches (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id           UUID NOT NULL REFERENCES statement_sources(id),
    file_name           TEXT NOT NULL,
    file_hash           VARCHAR(64),
    statement_period    DATERANGE NOT NULL,
    original_currency   VARCHAR(3) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ,
    ingested_by         UUID REFERENCES users(id),

    -- Processing summary (JSONB)
    processing_stats    JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    {
        "total_lines": 45230,
        "parsed_lines": 45230,
        "matched_lines": 44100,
        "unmatched_lines": 1130,
        "duplicate_lines": 0,
        "error_lines": 0,
        "total_amount": 125430.75,
        "anomaly_count": 12,
        "parse_duration_ms": 3400,
        "match_duration_ms": 12500,
        "errors": [
            {"line": 3042, "error": "Invalid currency code: XXX"},
            {"line": 7891, "error": "Negative amount: -5.23"}
        ]
    }
    */

    -- Anomaly detection results
    anomaly_report      JSONB
    /*
    {
        "generated_at": "2026-04-02T10:30:00Z",
        "model_version": "anomaly-v3",
        "summary": {
            "total_anomalies": 12,
            "critical": 2,
            "warning": 7,
            "info": 3
        },
        "anomalies": [
            {
                "type": "under_reporting",
                "work_id": "w1-...",
                "territory": "US",
                "expected_streams": 50000,
                "reported_streams": 25000,
                "deviation_pct": -50.0,
                "severity": "critical",
                "similar_period_avg": 48000
            },
            {
                "type": "missing_territory",
                "territory": "GB",
                "note": "No UK streams reported; historically 15% of volume",
                "severity": "warning"
            }
        ]
    }
    */
);

CREATE INDEX idx_sb_source ON statement_batches (source_id);
CREATE INDEX idx_sb_status ON statement_batches (status);
CREATE INDEX idx_sb_period ON statement_batches USING gist (statement_period);
CREATE UNIQUE INDEX idx_sb_file_hash ON statement_batches (file_hash) WHERE file_hash IS NOT NULL;
```

#### statement_lines

```sql
CREATE TABLE statement_lines (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id            UUID NOT NULL REFERENCES statement_batches(id) ON DELETE CASCADE,
    line_number         INTEGER NOT NULL,

    -- Universal financial fields (relational -- heavily queried/aggregated)
    territory_code      VARCHAR(4),
    use_type            VARCHAR(50) NOT NULL,
    units               BIGINT,
    gross_amount        NUMERIC(14,4) NOT NULL,
    currency_code       VARCHAR(3) NOT NULL,

    -- Catalog matching results (relational for JOINs)
    matched_recording_id UUID REFERENCES recordings(id),
    matched_work_id      UUID REFERENCES works(id),
    match_confidence     NUMERIC(5,2),
    match_method         VARCHAR(30),
    status              VARCHAR(20) NOT NULL DEFAULT 'unmatched',

    -- Raw source data preserved exactly as received (JSONB)
    raw_data            JSONB NOT NULL,
    /*
    The complete original row from the source file, preserving all
    source-specific fields that don't map to our standard columns:
    {
        "ISRC": "USRC12345678",
        "Track Title": "Midnight Rain",
        "Artist Name": "Artist A",
        "Album Title": "Debut Album",
        "Label": "Independent Records",
        "Country Code": "US",
        "Sale Type": "Premium Stream",
        "Quantity": 15234,
        "Per Unit Rate": 0.00385,
        "Net Amount": 58.65,
        "Currency": "USD",
        "Start Date": "2026-01-01",
        "End Date": "2026-01-31",
        "Store": "Spotify",
        "Subscription Type": "Premium",
        "Content Type": "Audio"
    }
    */

    -- Parsed identifiers from raw data
    parsed_identifiers  JSONB DEFAULT '{}'::jsonb,
    /*
    {
        "isrc": "USRC12345678",
        "upc": "012345678901",
        "title": "Midnight Rain",
        "artist": "Artist A",
        "album": "Debut Album"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_stmt_line UNIQUE (batch_id, line_number)
);

CREATE INDEX idx_sl_batch ON statement_lines (batch_id);
CREATE INDEX idx_sl_recording ON statement_lines (matched_recording_id);
CREATE INDEX idx_sl_work ON statement_lines (matched_work_id);
CREATE INDEX idx_sl_territory ON statement_lines (territory_code);
CREATE INDEX idx_sl_use_type ON statement_lines (use_type);
CREATE INDEX idx_sl_status ON statement_lines (status);
CREATE INDEX idx_sl_parsed_isrc ON statement_lines
    ((parsed_identifiers->>'isrc'))
    WHERE parsed_identifiers->>'isrc' IS NOT NULL;
CREATE INDEX idx_sl_raw ON statement_lines USING gin (raw_data);
```

---

### Module 6: Royalty Calculations & Payments

Royalty calculations use relational columns because financial data demands strict typing, precision, and JOIN performance.

#### fx_rates

```sql
CREATE TABLE fx_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_date       DATE NOT NULL,
    from_currency   VARCHAR(3) NOT NULL,
    to_currency     VARCHAR(3) NOT NULL,
    rate            NUMERIC(16,8) NOT NULL,
    source          VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_fx_rate UNIQUE (rate_date, from_currency, to_currency)
);

CREATE INDEX idx_fx_date ON fx_rates (rate_date);
```

#### royalty_calculations

```sql
CREATE TABLE royalty_calculations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_line_id   UUID NOT NULL REFERENCES statement_lines(id),
    work_share_id       UUID REFERENCES work_shares(id),
    recording_share_id  UUID REFERENCES recording_shares(id),
    party_id            UUID NOT NULL REFERENCES parties(id),

    -- All financial fields relational for precision and aggregation
    gross_amount        NUMERIC(14,4) NOT NULL,
    original_currency   VARCHAR(3) NOT NULL,
    fx_rate_id          UUID REFERENCES fx_rates(id),
    converted_amount    NUMERIC(14,4) NOT NULL,
    settlement_currency VARCHAR(3) NOT NULL,
    share_percentage    NUMERIC(7,4) NOT NULL,
    rights_type         VARCHAR(30) NOT NULL,
    royalty_amount      NUMERIC(14,4) NOT NULL,
    admin_fee_pct       NUMERIC(5,2) DEFAULT 0,
    admin_fee_amount    NUMERIC(14,4) DEFAULT 0,
    withholding_tax_pct NUMERIC(5,2) DEFAULT 0,
    withholding_amount  NUMERIC(14,4) DEFAULT 0,
    net_amount          NUMERIC(14,4) NOT NULL,
    recoupable          BOOLEAN NOT NULL DEFAULT FALSE,
    recouped_amount     NUMERIC(14,4) DEFAULT 0,
    payable_amount      NUMERIC(14,4) NOT NULL,
    contract_id         UUID REFERENCES contracts(id),
    calculation_period  DATERANGE,
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Calculation audit trail (JSONB -- captures the exact inputs used)
    calculation_inputs  JSONB NOT NULL DEFAULT '{}'::jsonb
    /*
    {
        "share_version": 3,
        "share_effective_from": "2026-01-01",
        "territory_override_applied": true,
        "territory_override": {"performance_share": 30.00},
        "fx_rate_date": "2026-04-01",
        "fx_rate_value": 0.8125,
        "contract_terms": {
            "admin_fee": 15.00,
            "advance_remaining": 37500.00,
            "recoupment_rate": 100
        },
        "calculation_engine_version": "v2.1.0"
    }
    */
);

CREATE INDEX idx_rc_party ON royalty_calculations (party_id);
CREATE INDEX idx_rc_stmt_line ON royalty_calculations (statement_line_id);
CREATE INDEX idx_rc_period ON royalty_calculations USING gist (calculation_period);
```

#### payment_runs

```sql
CREATE TABLE payment_runs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_date            DATE NOT NULL,
    period_label        VARCHAR(50),
    settlement_currency VARCHAR(3) NOT NULL,
    total_amount        NUMERIC(16,4) NOT NULL,
    total_payees        INTEGER NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID REFERENCES users(id),

    -- Payment summary (JSONB)
    summary             JSONB DEFAULT '{}'::jsonb
    /*
    {
        "by_payment_method": {
            "stripe_connect": {"count": 180, "amount": 65000.00},
            "bank_transfer": {"count": 45, "amount": 20000.00},
            "paypal": {"count": 15, "amount": 3500.00}
        },
        "by_currency": {
            "USD": {"count": 150, "amount": 52000.00},
            "GBP": {"count": 50, "amount": 18500.00},
            "EUR": {"count": 40, "amount": 15000.00}
        },
        "failed_payments": 5,
        "failed_amount": 734.50,
        "retry_eligible": 3
    }
    */
);

CREATE INDEX idx_pr_status ON payment_runs (status);
CREATE INDEX idx_pr_date ON payment_runs (run_date);
```

#### payment_run_items

```sql
CREATE TABLE payment_run_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_run_id      UUID NOT NULL REFERENCES payment_runs(id) ON DELETE CASCADE,
    party_id            UUID NOT NULL REFERENCES parties(id),
    total_amount        NUMERIC(14,4) NOT NULL,
    currency_code       VARCHAR(3) NOT NULL,
    payment_method      VARCHAR(30),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    sent_at             TIMESTAMPTZ,
    confirmed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Payment processor response data (JSONB)
    payment_details     JSONB DEFAULT '{}'::jsonb
    /*
    {
        "payment_reference": "pi_3abc123...",
        "processor": "stripe_connect",
        "stripe_account_id": "acct_1abc...",
        "transfer_id": "tr_xyz789...",
        "bank_last4": "7890",
        "settlement_date": "2026-04-17",
        "processor_fee": 2.50,
        "error_code": null,
        "error_message": null,
        "retry_count": 0
    }
    */

    -- Breakdown of what's being paid (JSONB summary)
    breakdown           JSONB DEFAULT '{}'::jsonb
    /*
    {
        "by_work": [
            {"work_id": "w1-...", "title": "Midnight Rain", "amount": 450.00},
            {"work_id": "w2-...", "title": "Dawn", "amount": 800.00}
        ],
        "by_rights_type": {
            "performance": 625.00,
            "mechanical": 375.00,
            "sync": 250.00
        },
        "by_territory": {
            "US": 800.00,
            "GB": 250.00,
            "DE": 200.00
        },
        "calculation_ids": ["rc1-...", "rc2-...", "rc3-..."]
    }
    */
);

CREATE INDEX idx_pri_run ON payment_run_items (payment_run_id);
CREATE INDEX idx_pri_party ON payment_run_items (party_id);
CREATE INDEX idx_pri_status ON payment_run_items (status);
```

---

### Module 7: PRO Registration

```sql
CREATE TABLE pro_registrations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_id             UUID NOT NULL REFERENCES works(id),
    pro_party_id        UUID NOT NULL REFERENCES parties(id),
    registration_type   VARCHAR(10) NOT NULL,
    cwr_version         VARCHAR(5) DEFAULT '2.2',
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    submission_date     DATE,
    acknowledgement_date DATE,
    society_work_id     VARCHAR(30),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Full CWR transaction data (JSONB)
    cwr_transaction     JSONB,
    /*
    {
        "transaction_id": "00000042",
        "nwr_record": { ... },
        "spu_records": [ ... ],
        "spt_records": [ ... ],
        "swr_records": [ ... ],
        "swt_records": [ ... ],
        "per_records": [ ... ],
        "rec_records": [ ... ]
    }
    */

    -- Acknowledgement data from the PRO (JSONB)
    acknowledgement     JSONB
    /*
    {
        "status": "accepted",
        "society_work_id": "ASCAP-890123456",
        "response_file": "cwr/ack/ascap_ack_042.cwr",
        "conflicts": [],
        "warnings": [
            "Writer IPI mismatch: submitted 00123456789, society has 00123456790"
        ]
    }
    */
);

CREATE INDEX idx_reg_work ON pro_registrations (work_id);
CREATE INDEX idx_reg_pro ON pro_registrations (pro_party_id);
CREATE INDEX idx_reg_status ON pro_registrations (status);
```

---

### Module 8: System & Audit

#### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   TEXT,
    display_name    TEXT NOT NULL,
    party_id        UUID REFERENCES parties(id),
    role            VARCHAR(30) NOT NULL DEFAULT 'viewer',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    preferences     JSONB DEFAULT '{}'::jsonb
    /*
    {
        "default_currency": "USD",
        "timezone": "America/New_York",
        "notification_preferences": {
            "email_statement_ingested": true,
            "email_payment_sent": true,
            "email_option_deadline": true
        },
        "dashboard_config": {
            "default_period": "quarter",
            "default_territory": "all"
        }
    }
    */
);

CREATE INDEX idx_users_party ON users (party_id);
```

#### audit_log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    actor_id        UUID REFERENCES users(id),
    actor_ip        INET,

    -- Change data as JSONB (flexible for any entity type)
    changes         JSONB NOT NULL,
    /*
    {
        "old": {"performance_share": 25.00, "effective_to": null},
        "new": {"performance_share": 12.50, "effective_to": "2026-05-31"},
        "reason": "sub_publishing_agreement",
        "contract_id": "c9d8e7f6-..."
    }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_event ON audit_log (event_type);
CREATE INDEX idx_audit_actor ON audit_log (actor_id);
CREATE INDEX idx_audit_time ON audit_log (created_at);
CREATE INDEX idx_audit_changes ON audit_log USING gin (changes);
```

---

## Helper Functions

### Resolve territory-specific share for royalty calculation

```sql
CREATE OR REPLACE FUNCTION get_effective_share(
    p_work_share_id UUID,
    p_territory VARCHAR(4),
    p_rights_type VARCHAR(30)
) RETURNS NUMERIC AS $$
DECLARE
    v_share NUMERIC;
    v_override NUMERIC;
    v_config JSONB;
BEGIN
    -- Get base share and territory config
    SELECT
        CASE p_rights_type
            WHEN 'performance' THEN ws.performance_share
            WHEN 'mechanical' THEN ws.mechanical_share
            WHEN 'sync' THEN ws.sync_share
            WHEN 'print' THEN ws.print_share
        END,
        ws.territory_config
    INTO v_share, v_config
    FROM work_shares ws
    WHERE ws.id = p_work_share_id;

    -- Check for territory-specific override
    v_override := (
        v_config->'territory_overrides'->p_territory->>
        (p_rights_type || '_share')
    )::NUMERIC;

    IF v_override IS NOT NULL THEN
        RETURN v_override;
    END IF;

    -- Check for use-type override
    v_override := (
        v_config->'use_type_overrides'->p_rights_type->>'share'
    )::NUMERIC;

    IF v_override IS NOT NULL THEN
        RETURN v_override;
    END IF;

    -- Check if territory is excluded
    IF v_config->'excluded_territories' ? p_territory THEN
        RETURN 0;
    END IF;

    RETURN v_share;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

### Validate JSONB contract terms against expected schema

```sql
CREATE OR REPLACE FUNCTION validate_contract_terms(
    p_contract_type VARCHAR(50),
    p_terms JSONB
) RETURNS BOOLEAN AS $$
BEGIN
    CASE p_contract_type
        WHEN 'co_publishing' THEN
            RETURN p_terms ? 'royalty_rates'
               AND p_terms->'royalty_rates' ? 'mechanical'
               AND p_terms->'royalty_rates' ? 'performance';
        WHEN 'sync_licence' THEN
            RETURN p_terms ? 'licence_fee'
               AND p_terms ? 'usage'
               AND p_terms ? 'scope';
        WHEN 'recording_agreement' THEN
            RETURN p_terms ? 'royalty_rates'
               AND p_terms ? 'delivery_commitment';
        ELSE
            RETURN TRUE;  -- unknown types pass by default
    END CASE;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

---

## Pros and Cons

### Pros

1. **Best balance of structure and flexibility.** Core financial data (shares, amounts, FX rates) gets strict typing and relational integrity. Variable data (territory overrides, parser configs, contract terms, raw statement data) gets JSONB flexibility. This avoids both the rigidity of full normalization and the chaos of pure document storage.

2. **Dramatically fewer tables.** The fully normalized model (Suggestion 1) requires separate tables for work_share_territories, recording_share_territories, contract_territories, contract_works, contract_parties, contract_option_periods, contract_reversion_triggers, and parser configurations. The hybrid model collapses all of these into JSONB columns, reducing the table count from ~35 to ~20 while preserving all the same data.

3. **Parser configuration as data.** Statement source configurations are stored as JSONB documents, meaning new DSP formats can be added by inserting a row rather than writing code. The parser engine reads the column_mapping, applies transformations, and produces standardized output. This directly addresses the README's requirement for "configurable parser pipeline for major DSP and distributor CSV/Excel formats."

4. **Raw data preservation.** Storing the raw statement line data as JSONB alongside the normalized fields provides complete traceability. When a rights holder disputes a royalty amount, the system can show both the standardized calculation and the exact raw data from the DSP statement.

5. **CWR/DDEX data caching.** Pre-computed CWR and DDEX message data stored as JSONB enables fast export without complex real-time assembly. The structured external format maps naturally to JSON intermediate representation.

6. **Still relational where it counts.** Foreign keys between works, recordings, shares, statement_lines, and royalty_calculations are standard relational constraints. The royalty calculation engine operates on typed NUMERIC columns with proper decimal precision. Financial aggregations use standard SQL SUM/GROUP BY.

7. **PostgreSQL-native.** No additional database engines needed. PostgreSQL's JSONB support includes GIN indexing, containment queries (@>), existence checks (?), and path expressions (->>), providing efficient access to structured data within JSONB columns.

### Cons

1. **JSONB data is not schema-enforced.** There is no built-in mechanism to prevent a user from storing `"performance_share": "banana"` in the territory_config JSONB column. Application-level validation (or CHECK constraints calling validator functions) is required. This shifts part of the data integrity burden from the database to the application layer.

2. **JSONB query performance has limits.** While GIN indexes make containment queries fast, complex aggregations across JSONB fields (e.g., "sum the territory-specific performance shares across all works for publisher X") are significantly slower than the equivalent query against a normalized column. If territory-level reporting is a high-frequency use case, a materialized view may be needed.

3. **Migration complexity.** Evolving the JSONB schema within a column requires data migration scripts that parse and transform existing JSONB documents. These migrations are harder to write, test, and roll back than standard ALTER TABLE operations on relational columns.

4. **ORM impedance mismatch.** Most ORMs handle JSONB columns as opaque blobs rather than structured objects. Application code must manually serialize/deserialize JSONB fields, validate their structure, and handle missing keys gracefully. This adds boilerplate compared to purely relational models where the ORM handles all type mapping.

5. **Audit trail complexity for JSONB changes.** When a territory override is added to the territory_config JSONB column, the audit log must capture a JSON diff rather than simple old/new column values. Computing meaningful diffs of nested JSONB structures is more complex than diffing flat relational columns.

6. **Risk of JSONB creep.** Without discipline, developers may default to putting everything in JSONB because "it's easier than adding a column." Over time this can result in core business data being trapped in JSONB where it should be a proper relational column with constraints and indexes.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with JSONB, GIN indexes, pg_trgm, pgcrypto |
| **JSONB validation** | Application-layer JSON Schema validation (ajv for Node.js, jsonschema for Python) + PostgreSQL CHECK constraints calling PL/pgSQL validator functions for critical fields |
| **ORM** | Prisma (with `Json` type support) or SQLAlchemy (with `JSONB` column type) |
| **Migration** | Flyway or Alembic; use custom migration scripts for JSONB schema evolution |
| **Statement parser** | Custom ETL service that reads parser_config JSONB and applies column mappings dynamically |
| **Search** | PostgreSQL GIN indexes on JSONB for structured queries; pg_trgm for fuzzy text matching; consider Elasticsearch only if faceted catalog search at scale is needed |
| **JSONB diff** | Use PostgreSQL `jsonb_diff` functions or application-level diff libraries for audit trail generation |
| **Backup** | WAL archiving with PITR; JSONB data is included in standard pg_dump |

---

## Migration and Scaling Considerations

### Initial Deployment

The hybrid model runs on a single PostgreSQL instance. No additional database engines are needed. The parser_config JSONB approach means new DSP formats can be supported by inserting a row in statement_sources rather than deploying code. This significantly reduces time-to-market for supporting new statement formats.

### From Spreadsheets

Most mid-market publishers migrating from spreadsheets will benefit from the flexibility of JSONB. Their existing data often has inconsistent structures (some works have territory splits, others don't; some contracts have option periods, others are one-time sync licenses). The JSONB columns accommodate this variance without requiring the publisher to restructure all their data before migration.

### Scaling

1. **JSONB column size monitoring.** Use `pg_column_size()` to monitor the size of JSONB columns. If territory_config documents grow beyond 10KB per row (hundreds of territory overrides), consider extracting the most-queried fields into dedicated relational columns.

2. **Partial indexes on JSONB paths.** For high-frequency query patterns, create partial indexes on specific JSONB paths rather than full GIN indexes. Example: an index on `(territory_config->'territory_overrides'->'US')` is faster and smaller than a full GIN index on the entire territory_config column.

3. **Materialized views for JSONB aggregation.** Pre-compute territory-level summaries that require JSONB traversal as materialized views, refreshed after share changes.

4. **Statement line partitioning.** Partition statement_lines by batch period (monthly or quarterly) using RANGE partitioning. This keeps the raw_data JSONB storage manageable and enables efficient partition pruning.

5. **TOAST compression.** PostgreSQL automatically compresses JSONB values larger than ~2KB using TOAST (The Oversized-Attribute Storage Technique). For very large contract terms documents, this provides transparent compression without application changes.

### JSONB Schema Evolution Strategy

Establish a versioning convention within JSONB documents:

```json
{
    "_schema_version": 2,
    "royalty_rates": { ... }
}
```

When the schema evolves:
1. Increment `_schema_version` in new documents.
2. Write a migration function that upgrades v1 documents to v2 on read (lazy migration) or in batch (eager migration).
3. Application code checks `_schema_version` and applies the appropriate parser.
4. Old documents remain valid and readable; they are upgraded when next written.
