# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Music Rights Management · Candidate #441 · Generated: 2026-05-25

---

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the music rights domain. Every state change -- a split update, a statement ingestion, a royalty calculation, a payment disbursement -- is captured as an immutable event in a central event store. The current state of any entity is derived by replaying its event stream. Separate read-model projections are maintained for queries, dashboards, and reporting.

This architecture is particularly well-suited to music rights management because:

1. **Ownership changes must be auditable.** When a publisher's share reverts from 50% to 0% due to a contractual reversion clause, the entire history of how shares evolved must be preserved. Event sourcing makes this automatic -- there is no separate audit log to maintain; the event stream *is* the audit trail.

2. **Royalty calculations are temporal.** A stream earned revenue in January 2026 under one split arrangement, but the split changed in February. The calculation engine must apply the correct shares for each period. Event sourcing naturally handles this by replaying events up to each point in time.

3. **Disputes require state reconstruction.** When a songwriter disputes their royalty statement, the system must be able to reconstruct exactly what data was available, what splits were in effect, and what FX rates were used at the time of each calculation. Event replay provides this without maintaining parallel snapshot tables.

4. **Statement ingestion is inherently event-driven.** DSP statements arrive as batches of usage events. Modeling them as events from the start eliminates the impedance mismatch between ingestion and storage.

---

## Event Store Schema

The event store is the single source of truth. All events are stored in a single append-only table partitioned by aggregate type.

### Core Event Store

```sql
-- The single source of truth for all state changes
CREATE TABLE event_store (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type      VARCHAR(50) NOT NULL,
        -- Work, Recording, Release, Party, Contract,
        -- WorkShare, RecordingShare, StatementBatch,
        -- RoyaltyRun, PaymentRun, ProRegistration
    aggregate_id        UUID NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    event_version       INTEGER NOT NULL,       -- per-aggregate sequence number
    event_data          JSONB NOT NULL,          -- the event payload
    metadata            JSONB NOT NULL DEFAULT '{}',
        -- actor_id, actor_ip, correlation_id, causation_id,
        -- source_system, idempotency_key
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_aggregate_version UNIQUE (aggregate_type, aggregate_id, event_version)
) PARTITION BY LIST (aggregate_type);

-- Partitions for each aggregate type (keeps event streams co-located)
CREATE TABLE events_work PARTITION OF event_store FOR VALUES IN ('Work');
CREATE TABLE events_recording PARTITION OF event_store FOR VALUES IN ('Recording');
CREATE TABLE events_release PARTITION OF event_store FOR VALUES IN ('Release');
CREATE TABLE events_party PARTITION OF event_store FOR VALUES IN ('Party');
CREATE TABLE events_contract PARTITION OF event_store FOR VALUES IN ('Contract');
CREATE TABLE events_work_share PARTITION OF event_store FOR VALUES IN ('WorkShare');
CREATE TABLE events_recording_share PARTITION OF event_store FOR VALUES IN ('RecordingShare');
CREATE TABLE events_statement_batch PARTITION OF event_store FOR VALUES IN ('StatementBatch');
CREATE TABLE events_royalty_run PARTITION OF event_store FOR VALUES IN ('RoyaltyRun');
CREATE TABLE events_payment_run PARTITION OF event_store FOR VALUES IN ('PaymentRun');
CREATE TABLE events_pro_registration PARTITION OF event_store FOR VALUES IN ('ProRegistration');

-- Indexes for efficient event replay and querying
CREATE INDEX idx_es_aggregate ON event_store (aggregate_type, aggregate_id, event_version);
CREATE INDEX idx_es_event_type ON event_store (event_type);
CREATE INDEX idx_es_created_at ON event_store (created_at);
CREATE INDEX idx_es_correlation ON event_store USING gin ((metadata->'correlation_id'));
CREATE INDEX idx_es_causation ON event_store USING gin ((metadata->'causation_id'));
```

### Aggregate Snapshots (Performance Optimization)

```sql
-- Periodic snapshots to avoid replaying long event streams
CREATE TABLE aggregate_snapshots (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type      VARCHAR(50) NOT NULL,
    aggregate_id        UUID NOT NULL,
    snapshot_version    INTEGER NOT NULL,   -- the event_version at snapshot time
    snapshot_data       JSONB NOT NULL,     -- serialized aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_snapshot UNIQUE (aggregate_type, aggregate_id, snapshot_version)
);

CREATE INDEX idx_snap_aggregate ON aggregate_snapshots (aggregate_type, aggregate_id, snapshot_version DESC);
```

### Event Subscriptions (For projections and process managers)

```sql
-- Tracks where each projection/subscriber has read up to
CREATE TABLE event_subscriptions (
    subscription_id     VARCHAR(100) PRIMARY KEY,
        -- "projection:active_catalog", "projection:royalty_dashboard",
        -- "process:statement_pipeline", "process:payment_disbursement"
    last_event_id       UUID REFERENCES event_store(event_id),
    last_processed_at   TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    checkpoint_data     JSONB,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalog

### Work Aggregate Events

```jsonc
// WorkCreated
{
    "event_type": "WorkCreated",
    "event_data": {
        "title": "Midnight Rain",
        "iswc": "T-345246800-1",
        "language_code": "en",
        "duration_seconds": 214,
        "work_type": "MusicalWork",
        "genre": "Pop",
        "ai_contribution": false,
        "alternate_titles": ["Midnight Rain (Acoustic Version)"]
    }
}

// WorkTitleUpdated
{
    "event_type": "WorkTitleUpdated",
    "event_data": {
        "previous_title": "Midnight Rain",
        "new_title": "Midnight Rain (feat. Artist B)",
        "reason": "featured_artist_added"
    }
}

// WorkISWCAssigned
{
    "event_type": "WorkISWCAssigned",
    "event_data": {
        "iswc": "T-345246800-1",
        "assigned_by": "CISAC",
        "assignment_date": "2026-03-15"
    }
}

// WorkWithdrawn
{
    "event_type": "WorkWithdrawn",
    "event_data": {
        "reason": "duplicate_registration",
        "superseded_by_work_id": "a1b2c3d4-...",
        "withdrawal_date": "2026-04-01"
    }
}

// WorkRecordingLinked
{
    "event_type": "WorkRecordingLinked",
    "event_data": {
        "recording_id": "r9e8f7d6-...",
        "relationship": "performance",
        "is_primary": true
    }
}
```

### WorkShare Aggregate Events

```jsonc
// WorkShareCreated -- a party's ownership share in a work is established
{
    "event_type": "WorkShareCreated",
    "event_data": {
        "work_id": "w1a2b3c4-...",
        "party_id": "p5e6f7g8-...",
        "role": "composer",
        "performance_share": 25.00,
        "mechanical_share": 25.00,
        "sync_share": 25.00,
        "print_share": 25.00,
        "controlled": true,
        "publisher_party_id": "pub123-...",
        "effective_from": "2026-01-01",
        "territories": [
            {"code": "US", "include_exclude": "include"},
            {"code": "GB", "include_exclude": "include"}
        ]
    }
}

// WorkShareRevised -- ownership percentage changes
{
    "event_type": "WorkShareRevised",
    "event_data": {
        "work_id": "w1a2b3c4-...",
        "party_id": "p5e6f7g8-...",
        "previous_performance_share": 25.00,
        "new_performance_share": 12.50,
        "previous_mechanical_share": 25.00,
        "new_mechanical_share": 12.50,
        "reason": "sub_publishing_agreement",
        "contract_id": "c9d8e7f6-...",
        "effective_from": "2026-06-01"
    }
}

// WorkShareTerritoryOverrideAdded
{
    "event_type": "WorkShareTerritoryOverrideAdded",
    "event_data": {
        "work_id": "w1a2b3c4-...",
        "party_id": "p5e6f7g8-...",
        "territory_code": "DE",
        "performance_share_override": 30.00,
        "mechanical_share_override": 30.00,
        "reason": "german_sub_publisher_deal"
    }
}

// WorkShareReverted -- rights revert to original holder
{
    "event_type": "WorkShareReverted",
    "event_data": {
        "work_id": "w1a2b3c4-...",
        "reverting_party_id": "pub456-...",
        "receiving_party_id": "p5e6f7g8-...",
        "reversion_trigger": "contract_expiry",
        "contract_id": "c9d8e7f6-...",
        "shares_reverted": {
            "performance": 25.00,
            "mechanical": 25.00,
            "sync": 25.00
        },
        "effective_from": "2026-12-31"
    }
}
```

### Recording Aggregate Events

```jsonc
// RecordingCreated
{
    "event_type": "RecordingCreated",
    "event_data": {
        "title": "Midnight Rain",
        "isrc": "USRC12345678",
        "duration_seconds": 214,
        "recording_type": "SoundRecording",
        "primary_artist_id": "art789-...",
        "label_id": "lab456-...",
        "recording_date": "2026-01-10",
        "audio_format": "WAV",
        "sample_rate": 48000,
        "bit_depth": 24
    }
}

// RecordingShareCreated
{
    "event_type": "RecordingShareCreated",
    "event_data": {
        "recording_id": "r9e8f7d6-...",
        "party_id": "art789-...",
        "role": "featured_artist",
        "master_share": 15.00,
        "neighboring_share": 50.00,
        "effective_from": "2026-01-10"
    }
}

// RecordingISRCAssigned
{
    "event_type": "RecordingISRCAssigned",
    "event_data": {
        "isrc": "USRC12345678",
        "assigned_by": "label_self_assigned"
    }
}
```

### StatementBatch Aggregate Events

```jsonc
// StatementBatchReceived -- a new DSP/distributor statement file arrives
{
    "event_type": "StatementBatchReceived",
    "event_data": {
        "source_name": "Spotify",
        "source_type": "dsp",
        "file_name": "spotify_q1_2026.csv",
        "file_hash": "sha256:abc123...",
        "file_format": "csv",
        "statement_period_start": "2026-01-01",
        "statement_period_end": "2026-03-31",
        "original_currency": "USD",
        "line_count": 45230
    }
}

// StatementLineParsed -- a single line item extracted from the file
{
    "event_type": "StatementLineParsed",
    "event_data": {
        "line_number": 1,
        "reported_isrc": "USRC12345678",
        "reported_title": "Midnight Rain",
        "reported_artist": "Artist A",
        "territory_code": "US",
        "use_type": "stream",
        "units": 15234,
        "unit_rate": 0.00385,
        "gross_amount": 58.65,
        "currency_code": "USD",
        "usage_period_start": "2026-01-01",
        "usage_period_end": "2026-01-31"
    }
}

// StatementLineMatched -- line matched to catalog recording/work
{
    "event_type": "StatementLineMatched",
    "event_data": {
        "line_number": 1,
        "matched_recording_id": "r9e8f7d6-...",
        "matched_work_id": "w1a2b3c4-...",
        "match_method": "isrc",
        "match_confidence": 100.00
    }
}

// StatementLineAnomalyDetected
{
    "event_type": "StatementLineAnomalyDetected",
    "event_data": {
        "line_number": 42,
        "anomaly_type": "under_reporting",
        "anomaly_details": "Stream count 50% below expected based on chart position",
        "expected_units": 30000,
        "reported_units": 15000,
        "severity": "warning"
    }
}

// StatementBatchValidated
{
    "event_type": "StatementBatchValidated",
    "event_data": {
        "total_lines": 45230,
        "matched_lines": 44100,
        "unmatched_lines": 1130,
        "total_amount": 125430.75,
        "anomaly_count": 12
    }
}
```

### Contract Aggregate Events

```jsonc
// ContractCreated
{
    "event_type": "ContractCreated",
    "event_data": {
        "contract_type": "co_publishing",
        "title": "Artist A Co-Publishing Deal 2026",
        "parties": [
            {"party_id": "pub123-...", "role": "publisher"},
            {"party_id": "p5e6f7g8-...", "role": "writer"}
        ],
        "effective_date": "2026-01-01",
        "expiry_date": "2031-12-31",
        "governing_law": "New York, USA",
        "currency_code": "USD",
        "advance_amount": 50000.00,
        "territories": ["US", "CA", "GB", "AU"]
    }
}

// ContractOptionExercised
{
    "event_type": "ContractOptionExercised",
    "event_data": {
        "period_number": 2,
        "exercise_date": "2027-11-15",
        "new_expiry_date": "2032-12-31",
        "additional_advance": 25000.00,
        "exercised_by": "pub123-..."
    }
}

// ContractReversionTriggered
{
    "event_type": "ContractReversionTriggered",
    "event_data": {
        "trigger_type": "out_of_print",
        "trigger_date": "2028-06-01",
        "reversion_scope": "full",
        "affected_works": ["w1a2b3c4-...", "w2b3c4d5-..."],
        "reverting_party_id": "pub123-...",
        "receiving_party_id": "p5e6f7g8-..."
    }
}
```

### RoyaltyRun Aggregate Events

```jsonc
// RoyaltyRunStarted
{
    "event_type": "RoyaltyRunStarted",
    "event_data": {
        "statement_batch_ids": ["sb1-...", "sb2-...", "sb3-..."],
        "calculation_period_start": "2026-01-01",
        "calculation_period_end": "2026-03-31",
        "settlement_currency": "USD",
        "fx_rate_date": "2026-04-01"
    }
}

// RoyaltyLineCalculated -- one calculation per statement line per rights holder
{
    "event_type": "RoyaltyLineCalculated",
    "event_data": {
        "statement_line_ref": {"batch_id": "sb1-...", "line_number": 1},
        "party_id": "p5e6f7g8-...",
        "work_share_version": 3,
        "rights_type": "performance",
        "gross_amount": 58.65,
        "original_currency": "USD",
        "fx_rate": 1.0,
        "converted_amount": 58.65,
        "share_percentage": 25.00,
        "royalty_amount": 14.66,
        "admin_fee_pct": 15.00,
        "admin_fee_amount": 2.20,
        "withholding_tax_pct": 0,
        "withholding_amount": 0,
        "net_amount": 12.46,
        "recoupable": true,
        "recouped_amount": 12.46,
        "payable_amount": 0.00,
        "contract_id": "c9d8e7f6-...",
        "remaining_advance": 49987.54
    }
}

// RoyaltyRunCompleted
{
    "event_type": "RoyaltyRunCompleted",
    "event_data": {
        "total_lines_calculated": 176920,
        "total_parties_affected": 342,
        "total_gross": 125430.75,
        "total_royalties": 125430.75,
        "total_payable": 89234.50,
        "total_recouped": 36196.25,
        "calculation_duration_ms": 45230
    }
}
```

### PaymentRun Aggregate Events

```jsonc
// PaymentRunCreated
{
    "event_type": "PaymentRunCreated",
    "event_data": {
        "royalty_run_id": "rr1-...",
        "period_label": "Q1 2026",
        "settlement_currency": "USD",
        "total_payees": 245,
        "total_amount": 89234.50
    }
}

// PaymentDisbursed
{
    "event_type": "PaymentDisbursed",
    "event_data": {
        "party_id": "p5e6f7g8-...",
        "amount": 1250.00,
        "currency_code": "USD",
        "payment_method": "stripe_connect",
        "payment_reference": "pi_3abc123...",
        "bank_account_last4": "7890"
    }
}

// PaymentFailed
{
    "event_type": "PaymentFailed",
    "event_data": {
        "party_id": "p5e6f7g8-...",
        "amount": 1250.00,
        "payment_method": "stripe_connect",
        "error_code": "insufficient_funds",
        "error_message": "Bank account could not accept transfer",
        "retry_eligible": true
    }
}

// PaymentRunCompleted
{
    "event_type": "PaymentRunCompleted",
    "event_data": {
        "total_disbursed": 88500.00,
        "total_failed": 734.50,
        "successful_payments": 240,
        "failed_payments": 5,
        "completion_time": "2026-04-15T14:30:00Z"
    }
}
```

### ProRegistration Aggregate Events

```jsonc
// ProRegistrationSubmitted
{
    "event_type": "ProRegistrationSubmitted",
    "event_data": {
        "work_id": "w1a2b3c4-...",
        "pro_name": "ASCAP",
        "pro_party_id": "pro_ascap-...",
        "registration_type": "NWR",
        "cwr_version": "2.2",
        "submission_file_path": "cwr/2026/ascap_batch_042.cwr",
        "transaction_id": "00000042",
        "shares_submitted": [
            {"party_id": "p5e6f7g8-...", "role": "composer", "performance_share": 50.00},
            {"party_id": "pub123-...", "role": "original_publisher", "performance_share": 50.00}
        ]
    }
}

// ProRegistrationAcknowledged
{
    "event_type": "ProRegistrationAcknowledged",
    "event_data": {
        "pro_name": "ASCAP",
        "society_work_id": "ASCAP-890123456",
        "acknowledgement_type": "accepted",
        "acknowledgement_date": "2026-04-20",
        "acknowledgement_file_path": "cwr/ack/ascap_ack_042.cwr"
    }
}

// ProRegistrationRejected
{
    "event_type": "ProRegistrationRejected",
    "event_data": {
        "pro_name": "BMI",
        "rejection_reason": "duplicate_work",
        "conflicting_work_id": "BMI-456789012",
        "rejection_date": "2026-04-22"
    }
}
```

---

## Read Model Projections

Read models are materialized views optimized for specific query patterns. They are rebuilt by replaying events from the event store and kept current by processing new events as they arrive.

### Projection 1: Active Catalog (for catalog browsing and search)

```sql
CREATE TABLE rm_catalog_works (
    work_id             UUID PRIMARY KEY,
    title               TEXT NOT NULL,
    iswc                VARCHAR(15),
    language_code       VARCHAR(5),
    duration_seconds    INTEGER,
    genre               VARCHAR(100),
    work_type           VARCHAR(50),
    ai_contribution     BOOLEAN,
    status              VARCHAR(30),
    recording_count     INTEGER DEFAULT 0,
    share_holder_count  INTEGER DEFAULT 0,
    total_performance_share NUMERIC(7,2),
    total_mechanical_share  NUMERIC(7,2),
    created_at          TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ,
    last_event_version  INTEGER NOT NULL,
    
    -- Denormalized for search
    primary_writers     TEXT[],      -- array of writer names
    primary_publishers  TEXT[],      -- array of publisher names
    recording_isrcs     TEXT[],      -- array of linked ISRCs
    territories         TEXT[],      -- array of territory codes with active shares
    search_text         TSVECTOR     -- full-text search vector
);

CREATE INDEX idx_rmcw_iswc ON rm_catalog_works (iswc) WHERE iswc IS NOT NULL;
CREATE INDEX idx_rmcw_search ON rm_catalog_works USING gin (search_text);
CREATE INDEX idx_rmcw_genre ON rm_catalog_works (genre);
CREATE INDEX idx_rmcw_status ON rm_catalog_works (status);
```

```sql
CREATE TABLE rm_catalog_recordings (
    recording_id        UUID PRIMARY KEY,
    title               TEXT NOT NULL,
    isrc                VARCHAR(12),
    duration_seconds    INTEGER,
    recording_type      VARCHAR(30),
    primary_artist_name TEXT,
    label_name          TEXT,
    release_date        DATE,
    status              VARCHAR(30),
    linked_work_ids     UUID[],
    linked_work_titles  TEXT[],
    last_event_version  INTEGER NOT NULL,
    search_text         TSVECTOR
);

CREATE INDEX idx_rmcr_isrc ON rm_catalog_recordings (isrc) WHERE isrc IS NOT NULL;
CREATE INDEX idx_rmcr_search ON rm_catalog_recordings USING gin (search_text);
CREATE INDEX idx_rmcr_artist ON rm_catalog_recordings (primary_artist_name);
```

### Projection 2: Current Shares (for rights administration)

```sql
CREATE TABLE rm_current_work_shares (
    id                  UUID PRIMARY KEY,
    work_id             UUID NOT NULL,
    work_title          TEXT NOT NULL,
    party_id            UUID NOT NULL,
    party_name          TEXT NOT NULL,
    party_type          VARCHAR(30),
    ipi_name_number     VARCHAR(11),
    role                VARCHAR(30) NOT NULL,
    performance_share   NUMERIC(7,4),
    mechanical_share    NUMERIC(7,4),
    sync_share          NUMERIC(7,4),
    print_share         NUMERIC(7,4),
    controlled          BOOLEAN,
    effective_from      DATE,
    version             INTEGER,
    last_event_version  INTEGER NOT NULL
);

CREATE INDEX idx_rmcws_work ON rm_current_work_shares (work_id);
CREATE INDEX idx_rmcws_party ON rm_current_work_shares (party_id);
```

```sql
-- Territory-specific overrides, denormalized
CREATE TABLE rm_current_work_share_territories (
    work_share_id       UUID NOT NULL,
    territory_code      VARCHAR(4) NOT NULL,
    performance_share   NUMERIC(7,4),
    mechanical_share    NUMERIC(7,4),
    sync_share          NUMERIC(7,4),
    PRIMARY KEY (work_share_id, territory_code)
);
```

### Projection 3: Royalty Dashboard (for financial reporting)

```sql
CREATE TABLE rm_royalty_summary (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id            UUID NOT NULL,
    party_name          TEXT NOT NULL,
    work_id             UUID,
    work_title          TEXT,
    recording_id        UUID,
    territory_code      VARCHAR(4),
    use_type            VARCHAR(50),
    rights_type         VARCHAR(30),
    period_year         SMALLINT NOT NULL,
    period_quarter      SMALLINT NOT NULL,
    period_month        SMALLINT,
    settlement_currency VARCHAR(3) NOT NULL,
    total_gross         NUMERIC(16,4) DEFAULT 0,
    total_royalties     NUMERIC(16,4) DEFAULT 0,
    total_admin_fees    NUMERIC(16,4) DEFAULT 0,
    total_withholding   NUMERIC(16,4) DEFAULT 0,
    total_net           NUMERIC(16,4) DEFAULT 0,
    total_recouped      NUMERIC(16,4) DEFAULT 0,
    total_payable       NUMERIC(16,4) DEFAULT 0,
    stream_count        BIGINT DEFAULT 0,
    download_count      BIGINT DEFAULT 0,
    line_count          INTEGER DEFAULT 0,
    last_event_version  INTEGER NOT NULL
);

CREATE INDEX idx_rmrs_party ON rm_royalty_summary (party_id);
CREATE INDEX idx_rmrs_period ON rm_royalty_summary (period_year, period_quarter);
CREATE INDEX idx_rmrs_territory ON rm_royalty_summary (territory_code);
CREATE INDEX idx_rmrs_work ON rm_royalty_summary (work_id);
CREATE INDEX idx_rmrs_use_type ON rm_royalty_summary (use_type);
```

### Projection 4: Statement Pipeline Status (for operations monitoring)

```sql
CREATE TABLE rm_statement_pipeline (
    batch_id            UUID PRIMARY KEY,
    source_name         VARCHAR(100),
    source_type         VARCHAR(30),
    file_name           TEXT,
    statement_period    TEXT,        -- "2026-Q1"
    original_currency   VARCHAR(3),
    total_amount        NUMERIC(16,4),
    total_lines         INTEGER,
    matched_lines       INTEGER DEFAULT 0,
    unmatched_lines     INTEGER DEFAULT 0,
    anomaly_count       INTEGER DEFAULT 0,
    status              VARCHAR(30),
    ingested_at         TIMESTAMPTZ,
    validated_at        TIMESTAMPTZ,
    calculated_at       TIMESTAMPTZ,
    error_message       TEXT,
    last_event_version  INTEGER NOT NULL
);

CREATE INDEX idx_rmsp_status ON rm_statement_pipeline (status);
CREATE INDEX idx_rmsp_source ON rm_statement_pipeline (source_name);
```

### Projection 5: PRO Registration Status (for publishing admin)

```sql
CREATE TABLE rm_pro_registration_status (
    id                  UUID PRIMARY KEY,
    work_id             UUID NOT NULL,
    work_title          TEXT NOT NULL,
    iswc                VARCHAR(15),
    pro_name            VARCHAR(100) NOT NULL,
    registration_type   VARCHAR(10),
    submission_date     DATE,
    status              VARCHAR(30),
    society_work_id     VARCHAR(30),
    acknowledgement_date DATE,
    rejection_reason    TEXT,
    last_event_version  INTEGER NOT NULL
);

CREATE INDEX idx_rmprs_work ON rm_pro_registration_status (work_id);
CREATE INDEX idx_rmprs_status ON rm_pro_registration_status (status);
CREATE INDEX idx_rmprs_pro ON rm_pro_registration_status (pro_name);
```

### Projection 6: Contract Alerts (for deal management)

```sql
CREATE TABLE rm_contract_alerts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL,
    contract_title      TEXT NOT NULL,
    contract_type       VARCHAR(50),
    alert_type          VARCHAR(50) NOT NULL,
        -- option_deadline, expiry_approaching, reversion_pending,
        -- advance_recouped, renewal_due
    alert_date          DATE NOT NULL,
    party_names         TEXT[],
    details             TEXT,
    is_dismissed        BOOLEAN DEFAULT FALSE,
    last_event_version  INTEGER NOT NULL
);

CREATE INDEX idx_rmca_date ON rm_contract_alerts (alert_date) WHERE NOT is_dismissed;
CREATE INDEX idx_rmca_contract ON rm_contract_alerts (contract_id);
```

### Projection 7: Party Ledger (per-party financial history)

```sql
CREATE TABLE rm_party_ledger (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id            UUID NOT NULL,
    entry_date          DATE NOT NULL,
    entry_type          VARCHAR(30) NOT NULL,
        -- royalty_earned, admin_fee, withholding_tax, advance_recoupment,
        -- payment_disbursed, payment_failed, adjustment
    reference_type      VARCHAR(50),   -- royalty_run, payment_run, manual_adjustment
    reference_id        UUID,
    description         TEXT,
    debit_amount        NUMERIC(14,4) DEFAULT 0,
    credit_amount       NUMERIC(14,4) DEFAULT 0,
    balance             NUMERIC(14,4) NOT NULL,  -- running balance
    currency_code       VARCHAR(3) NOT NULL,
    last_event_version  INTEGER NOT NULL
);

CREATE INDEX idx_rmpl_party ON rm_party_ledger (party_id, entry_date);
CREATE INDEX idx_rmpl_reference ON rm_party_ledger (reference_type, reference_id);
```

---

## Process Managers (Sagas)

Process managers coordinate multi-step workflows by listening to events and issuing commands.

### Statement Ingestion Process

```
StatementBatchReceived
  -> parse file, emit StatementLineParsed for each row
     -> for each line, attempt catalog match
        -> emit StatementLineMatched or StatementLineUnmatched
           -> run anomaly detection
              -> emit StatementLineAnomalyDetected if flagged
                 -> when all lines processed, emit StatementBatchValidated
```

### Royalty Calculation Process

```
RoyaltyRunStarted
  -> for each matched statement line in the batch:
     -> look up active work/recording shares at the usage date
     -> look up applicable FX rate
     -> look up contract terms (admin fees, advances, withholding)
     -> calculate each rights holder's share
     -> emit RoyaltyLineCalculated
  -> when all lines done, emit RoyaltyRunCompleted
```

### Payment Disbursement Process

```
PaymentRunCreated
  -> for each party with a positive payable balance:
     -> look up payment method and bank account
     -> initiate payment via payment rails (Stripe Connect / Tipalti)
     -> emit PaymentDisbursed or PaymentFailed
  -> when all payments processed, emit PaymentRunCompleted
```

### Reversion Monitor Process

```
ContractReversionTriggered
  -> identify all works covered by the contract
  -> for each work:
     -> emit WorkShareReverted (transferring shares back)
     -> emit ProRegistrationRequired (to update PRO records)
  -> emit ContractTerminated
```

---

## Command Handlers

Commands are the write side of CQRS. Each command validates business rules and, if valid, appends events to the event store.

```
RegisterWork(title, iswc, writers, publishers, shares, territories)
  -> validate ISWC format
  -> validate share totals
  -> validate territory codes
  -> append WorkCreated event
  -> for each writer/publisher: append WorkShareCreated event

ReviseWorkShares(work_id, new_shares, reason, effective_from)
  -> load current shares from event stream
  -> validate new totals
  -> append WorkShareRevised events

IngestStatement(file, source, format, period)
  -> validate file hash for dedup
  -> append StatementBatchReceived
  -> trigger parse pipeline

RunRoyaltyCalculation(batch_ids, period, settlement_currency, fx_date)
  -> validate all batches are in 'validated' status
  -> append RoyaltyRunStarted
  -> trigger calculation pipeline

ApprovePaymentRun(royalty_run_id, approver)
  -> validate run is complete
  -> validate approver has permission
  -> append PaymentRunCreated
  -> trigger disbursement pipeline
```

---

## Temporal Query Examples

One of the most powerful capabilities of event sourcing is temporal queries -- reconstructing state at any point in time.

### "What were the ownership shares for this work on January 15, 2026?"

```python
def get_shares_at_date(work_id: UUID, target_date: date) -> list[Share]:
    events = event_store.query(
        aggregate_type='WorkShare',
        aggregate_id__work_id=work_id,
        created_at__lte=target_date,
        order_by='event_version'
    )
    shares = {}
    for event in events:
        if event.event_type == 'WorkShareCreated':
            shares[event.data['party_id']] = event.data
        elif event.event_type == 'WorkShareRevised':
            if date.fromisoformat(event.data['effective_from']) <= target_date:
                shares[event.data['party_id']].update(event.data)
        elif event.event_type == 'WorkShareReverted':
            if date.fromisoformat(event.data['effective_from']) <= target_date:
                del shares[event.data['reverting_party_id']]
    return list(shares.values())
```

### "Reconstruct the exact calculation for this party's Q1 2026 royalty"

```python
def reconstruct_royalty(party_id: UUID, period: str) -> dict:
    # Replay all RoyaltyLineCalculated events for this party in this period
    events = event_store.query(
        aggregate_type='RoyaltyRun',
        event_type='RoyaltyLineCalculated',
        event_data__party_id=party_id,
        event_data__period=period
    )
    return {
        'lines': [e.data for e in events],
        'total_gross': sum(e.data['gross_amount'] for e in events),
        'total_payable': sum(e.data['payable_amount'] for e in events),
        'share_versions_used': set(e.data['work_share_version'] for e in events)
    }
```

---

## Pros and Cons

### Pros

1. **Perfect audit trail by construction.** Every state change is an immutable event. There is no separate audit_log table to maintain -- the event store *is* the complete history. Regulatory audits, dispute resolution, and forensic analysis can reconstruct the exact state of any entity at any point in time. This directly addresses the README's requirement for an "immutable audit trail of all ownership changes, payment runs, and statement adjustments."

2. **Natural fit for temporal ownership.** Music rights are inherently temporal -- splits change, contracts expire, territories are added or removed. Event sourcing handles this natively. Questions like "what shares were in effect for this work in Germany in February 2026?" are answered by replaying events rather than querying complex date-range tables.

3. **Decoupled read and write models.** The CQRS separation means the catalog search projection can be optimized for full-text search (denormalized, with tsvectors), the royalty dashboard projection can be pre-aggregated for fast rendering, and the statement pipeline projection can be tailored for operational monitoring -- all without compromising the integrity of the write model.

4. **Natural alignment with statement ingestion.** DSP statements are fundamentally events: "X streams happened in territory Y during period Z." Modeling them as events eliminates the ETL impedance mismatch. The statement ingestion pipeline becomes a stream of events flowing through matching, validation, and calculation stages.

5. **Replay and reprocessing.** If a royalty calculation bug is discovered, the entire calculation can be replayed with corrected logic against the same input events. This is far safer than updating rows in place, which risks data loss.

6. **Supports event-driven integrations.** External systems (payment processors, PRO submission services, notification systems) can subscribe to specific event types and react asynchronously. This loose coupling simplifies integration architecture.

### Cons

1. **Operational complexity.** Event sourcing requires maintaining the event store, snapshot management, projection rebuilds, and subscription checkpoints. This is significantly more infrastructure than a simple CRUD application with PostgreSQL. The team needs experience with event-driven architecture.

2. **Eventual consistency in read models.** Projections are updated asynchronously after events are written. A user who creates a work and immediately searches for it might not find it in the catalog projection yet. This requires either synchronous projection updates (defeating some CQRS benefits) or UI-level optimistic updates.

3. **Event schema evolution.** As the domain model evolves, event schemas change. The system must handle upcasting (transforming old events to new schemas during replay) without corrupting historical data. This is a non-trivial engineering challenge that requires careful versioning strategy from day one.

4. **Storage growth.** Every state change is preserved forever. A work with 50 share revisions over 10 years has 50+ events rather than a single row. Statement ingestion generates millions of events per quarter. Without aggressive snapshotting and archival, the event store grows very large.

5. **Complex debugging.** When a royalty calculation is wrong, diagnosing the issue requires replaying the event stream and understanding which events contributed to the incorrect state. This is more complex than inspecting a row in a relational table.

6. **Query limitations.** Ad-hoc queries against the event store are awkward -- the data is stored as JSONB payloads, not in typed columns. Any new query pattern requires building a new projection. The initial development of projections is slower than writing SQL queries against normalized tables.

7. **Testing complexity.** Unit tests must work with event streams rather than database fixtures. Integration tests require setting up event store infrastructure. The test surface area is larger because both the event store and all projections must be verified.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event store** | PostgreSQL with JSONB (leverage existing PostgreSQL expertise); EventStoreDB if the team has event sourcing experience |
| **Message broker** | Apache Kafka or Amazon Kinesis for event streaming between services; RabbitMQ for simpler deployments |
| **Projection engine** | Custom projector service reading from the event store; or Kafka Streams / Flink for streaming projections |
| **Snapshot storage** | PostgreSQL (same database as event store, separate table) |
| **Read model database** | PostgreSQL for most projections; Elasticsearch for full-text catalog search |
| **Framework** | Axon Framework (Java), EventSourcing (Python), or Marten (.NET) for event sourcing primitives |
| **Serialization** | JSON with schema registry for event versioning; Avro or Protobuf for high-throughput scenarios |
| **Monitoring** | OpenTelemetry for distributed tracing; Prometheus for event store metrics (lag, throughput, projection health) |
| **Testing** | Event-driven test fixtures; projection verification suites; Testcontainers for integration tests |

---

## Migration and Scaling Considerations

### Initial Deployment

Start with PostgreSQL as both event store and read model database. This minimizes operational overhead while the team learns event sourcing patterns. Use a single-process projector that reads events and updates projections synchronously. Introduce Kafka only when the system needs multi-consumer event streaming.

### Migration from Existing Data

Migrating existing catalog data into an event-sourced system requires generating "synthetic events" that represent the initial state:

1. For each existing work, generate a `WorkCreated` event with its current metadata.
2. For each current split, generate a `WorkShareCreated` event.
3. For historical splits (if available), generate a sequence of `WorkShareRevised` events in chronological order.
4. Import all historical statements as `StatementBatchReceived` + `StatementLineParsed` events.

These synthetic events are marked with a metadata flag `"synthetic": true` so they can be distinguished from organic events during analysis.

### Scaling Path

1. **Partition the event store** by aggregate type (already done in the schema above) and by time range when individual partitions grow large.

2. **Snapshot frequently** for aggregates with long event streams. A work with 100+ share revision events should have a snapshot every 20 events to keep replay time under 50ms.

3. **Scale projections independently.** The royalty dashboard projection handles the most data volume (millions of calculation events). It can run on a dedicated read replica or a separate database instance.

4. **Introduce Kafka** when the system has multiple consumers for the same events (e.g., the royalty calculation engine, the anomaly detection service, and the notification service all need to process statement events independently).

5. **Archive old events** to cold storage (S3/GCS) after 7 years, keeping only snapshots in the active event store. Events can be restored for replay if needed for historical audits.

### Handling Event Store Growth

For a mid-market platform:
- ~10,000 works = ~50,000 work-related events (creation + share changes)
- ~50,000 recordings = ~100,000 recording events
- ~5 million statement lines/quarter = ~15 million statement events/quarter
- ~5 million royalty calculations/quarter = ~5 million calculation events/quarter

At ~20 million events/quarter, the event store grows by approximately 10-20 GB/quarter (with JSONB compression). This is manageable with PostgreSQL table partitioning by date range and aggregate type.
