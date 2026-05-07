# Standards & API Reference

> Project: Music Rights Management · Candidate #441 · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO 3901 — International Standard Recording Code (ISRC)**
- **URL:** https://www.iso.org/standard/64795.html
- The ISRC is a 12-character alphanumeric identifier permanently assigned to a specific sound recording or music video recording. Originally codified in 1986 and updated in 2001, it is the universal identifier used by PROs, DSPs, labels, and distributors to track revenue from sales and streams. Every rights management system must be able to store, validate, and look up ISRCs to reconcile incoming royalty statements against catalog records.

**ISO 15707 — International Standard Musical Work Code (ISWC)**
- **URL:** https://www.iso.org/standard/28780.html
- The ISWC is an 11-character identifier (format: T-XXXXXXXXX-C) assigned to a musical composition (work), distinct from its recordings. A composition has exactly one ISWC regardless of how many recordings (ISRCs) exist of it. Rights management systems must link ISWCs to composition records and surface them during PRO registration workflows, where societies use ISWCs to deduplicate work registrations across publishers.

**ISO 13818 / ISO 14496 — MPEG Audio/Video Standards**
- **URL:** https://www.iso.org/standard/75929.html
- Underlying encoding standards for audio deliverables distributed to DSPs. Relevant when a rights management system also handles asset delivery alongside metadata; delivery pipelines typically accept MP3 (MPEG-1 Audio Layer III), AAC (ISO 14496-3), and FLAC.

**ISO/IEC 27001 — Information Security Management**
- **URL:** https://www.iso.org/standard/27001
- The baseline information security standard for any SaaS platform handling commercially sensitive catalog data, royalty financials, and artist personal data. Relevant for enterprise contracts with major publishers and labels that often require ISO 27001 certification or equivalent SOC 2 attestation.

---

### W3C & IETF Standards

**RFC 7519 — JSON Web Tokens (JWT)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7519
- JWTs are the standard bearer token format used by OAuth 2.0 and OpenID Connect flows. Apple Music's developer API requires JWT for authentication; many music rights platforms issue JWTs for API session tokens. Essential for any API authentication layer in a music rights system.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization protocol for delegated access. Spotify's Web API, Apple Music, and most modern music platform APIs use OAuth 2.0. A music rights system exposing an API should implement OAuth 2.0 for third-party integrations (e.g. allowing DSPs or label ERPs to connect).

**RFC 7231 — HTTP/1.1 Semantics**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7231
- Foundational semantics for RESTful APIs. Platforms such as Revelator, MusicBrainz, and Tuned Global all expose REST APIs and conform to RFC 7231 conventions for status codes, content negotiation (Accept headers for JSON vs. XML), and method semantics.

**RFC 8288 — Web Linking**
- **URL:** https://datatracker.ietf.org/doc/html/rfc8288
- The standard for `Link` response headers used in pagination and resource discovery in REST APIs. Relevant for browse/search APIs returning large catalogs of recordings or works.

**W3C Verifiable Credentials Data Model**
- **URL:** https://www.w3.org/TR/vc-data-model/
- An emerging standard for cryptographically verifiable assertions of rights ownership, provenance, and identity. Relevant to next-generation rights attribution systems (e.g. linking an ISWC-registered composition to a specific songwriter's decentralized identity) and blockchain-adjacent ownership verification use cases.

---

### Data Model & API Specifications

**DDEX ERN (Electronic Release Notification) — v4.3**
- **URL:** https://ddex.net/standards/ · Knowledge base: https://kb.ddex.net
- The primary XML message suite for communicating release metadata and audio/video assets from record companies and distributors to DSPs. ERN 4.3 is the current operational profile. Rights management systems must generate or parse ERN messages when integrating with distribution pipelines. Implementation is available under a free DDEX Implementation Licence.

**DDEX MWDR (Musical Work Data and Rights)**
- **URL:** https://kb.ddex.net/implementing-each-standard/musical-work-data-and-rights-communication-(mwdr)/
- The DDEX standard for music publishers to communicate composition metadata and rights data (ownership splits, territory restrictions, PRO affiliations) to DSPs. The Musical Work Right-Share Notification (MWN) sub-standard enables publishers to notify societies of rights splits; it explicitly references CWR as the legacy equivalent.

**DDEX RDR (Recording Data and Rights)**
- **URL:** https://ddex.net/standards/recording-data-and-rights/
- Covers metadata about sound recordings, music videos, performer credits, and usage/sales data linked to master recording rights. Relevant for master rights tracking modules within a rights management platform.

**DDEX DSR (Digital Sales Reporting)**
- **URL:** https://ddex.net/standards/
- The message format DSPs use to report sales and streaming data back to labels and distributors. A rights management platform's statement ingestion layer must parse DSR messages (alongside proprietary CSV/XLS formats) to receive earnings data.

**DDEX MEAD (Media Enrichment and Description)**
- **URL:** https://kb.ddex.net/implementing-each-standard/media-enrichment-and-description-(mead)/mead-explained/
- Enables record companies and distributors to communicate non-core metadata (editorial descriptions, mood tags, genre classifications) to DSPs to enhance user experience. Relevant for catalog management features.

**CISAC CWR (Common Works Registration) — v2.2**
- **URL:** https://members.cisac.org/CisacPortal/cisacDownloadFile.do?docId=37081
- The globally recognised file format for bulk registration and revision of musical works with performing rights societies. A publisher creates a single CWR file and submits it to multiple CISAC member societies worldwide; societies return CWR acknowledgement files. Current operational version is 2.2. Any publishing administration feature must support CWR 2.2 export. Open-source parsing libraries exist: weso/CWR-DataApi (Python) and aporia-records/APORIA-Works-Registration (PHP).
- GitHub libraries: https://github.com/weso/CWR-DataApi · https://github.com/aporia-records/APORIA-Works-Registration

**Open Music Initiative (OMI) MVI 1.0**
- **URL:** https://open-music.org/our-api · GitHub: https://github.com/omi · Apiary docs: https://docs.omi01.apiary.io
- An open-source HTTP API specification developed by Berklee/MIT (2017, regularly updated) that defines how to link sound recordings to musical compositions and their creators/rights holders for royalty discovery. Built on a federated search model connecting to MusicBrainz, Wikipedia, and other data sources. Relevant as a standard for interoperability between independent rights management tools.

**OpenAPI Specification (OAS) 3.1**
- **URL:** https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for documenting REST APIs. Platforms such as Tuned Global (Swagger/OpenAPI interfaces), LabelGrid, and Revelator publish machine-readable API descriptions. A music rights management platform should publish an OpenAPI 3.1 spec to support third-party integrations, SDK generation, and contract testing.

**MusicBrainz Data Model**
- **URL:** https://musicbrainz.org/doc/MusicBrainz_API · Wiki: https://wiki.musicbrainz.org/MusicBrainz_API
- MusicBrainz defines 13 canonical music entities (artist, recording, release, release-group, work, label, etc.) with globally unique MBIDs (36-character UUIDs). The MusicBrainz data model is a widely adopted open reference for music metadata schemas. Rights management systems that import or validate catalog metadata should support MBID lookups and ISRCs/ISWCs as secondary identifiers.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7636
- The standard combination for secure API access from public/native clients without client secrets. Spotify's Web API, AllTrack's partner APIs, and most modern music platform APIs mandate OAuth 2.0 with PKCE for third-party authentication flows.

**OpenID Connect 1.0**
- **URL:** https://openid.net/connect/
- Identity layer on top of OAuth 2.0 providing standard user identity tokens. Relevant for multi-tenant SaaS rights platforms where labels, publishers, and songwriters authenticate via federated identity providers (Google, Microsoft).

**OWASP API Security Top 10**
- **URL:** https://owasp.org/www-project-api-security/
- The authoritative checklist for securing REST and GraphQL APIs. Critical for rights management platforms that expose financial data: broken object-level authorization, excessive data exposure, and mass assignment vulnerabilities are high-priority risks when different users (songwriters, label staff, publishers) share the same API surface with different permission scopes.

**NIST SP 800-63B — Digital Identity Guidelines**
- **URL:** https://pages.nist.gov/800-63-3/sp800-63b.html
- Authentication assurance levels (AAL1–AAL3) for SaaS platforms. Enterprise publisher and label clients frequently require MFA (AAL2) and may require phishing-resistant authentication (AAL3 / hardware keys) for financial disbursement workflows.

**GDPR (EU) 2016/679**
- **URL:** https://gdpr-info.eu/
- Applies to any platform storing personal data of EU-resident artists, songwriters, or rights holders. Music rights systems collect name, address, banking details, tax IDs, and royalty history — all personal data subject to GDPR. Requires data minimisation, right-to-erasure workflows, and data processing agreements with third-party integrations (DSPs, payment processors).

---

### MCP Server Specifications

Model Context Protocol (MCP) servers are relevant if the platform exposes AI-assistant capabilities for rights query answering, contract summarisation, or anomaly detection on royalty statements.

**Model Context Protocol (MCP)**
- **URL:** https://modelcontextprotocol.io/
- An open protocol for connecting AI models (LLMs) to external data sources and tools via a standardised client-server interface. A music rights management platform could expose an MCP server providing tools such as: `lookup_work(iswc)`, `get_royalty_splits(work_id, period)`, `search_catalog(query)`, and `summarize_contract(contract_id)` — enabling AI assistants to answer rights queries without requiring direct database access.

---

## Similar Products — Developer Documentation & APIs

### MusicBrainz
- **Description:** Open-source collaborative music encyclopedia providing a free, publicly accessible REST API for music metadata including artists, recordings, releases, works, and labels. The canonical open reference for music entity models.
- **API Documentation:** https://musicbrainz.org/doc/MusicBrainz_API
- **SDKs/Libraries:** JavaScript: https://github.com/Borewit/musicbrainz-api · Emacs Lisp: https://github.com/emacsmirror/musicbrainz
- **Developer Guide:** https://musicbrainz.org/doc/Developer_Resources · Community: https://community.metabrainz.org
- **Standards:** REST/JSON and XML; supports ISRC and ISWC identifier lookups; 1 req/sec rate limit; OAuth for write operations
- **Authentication:** Anonymous for read; OAuth 2.0 or HTTP Digest for write/submission

### Revelator
- **Description:** End-to-end platform for music distribution, rights management, royalty processing, reporting, and analytics; provides a REST API for labels and distributors to manage their full music business operations programmatically.
- **API Documentation:** https://api-docs.revelator.com · Getting started: https://developers.revelator.com/
- **Sections:** Catalog Management, Distribution, Rights (`/accounting/contract/*`), Royalty Tokens, Reporting
- **Developer Guide:** https://api-docs.revelator.com/v2/en/getting-started
- **Standards:** REST/JSON; OpenAPI-documented endpoints; paginated list endpoints with filtering and sorting
- **Authentication:** API key (client secret) for standard access; parent account access tokens for contract/rights management endpoints

### AllTrack (Performing Rights Organization)
- **Description:** US digital PRO offering a suite of APIs for creator-centric platforms (streaming services, UGC platforms, distributors, publishing admins) to enable in-platform PRO registration, songwriter profile authentication, work registration, and real-time performing rights clearance.
- **API Documentation:** https://www.alltrack.com/partner-with-alltrack/
- **Key Capabilities:** In-platform PRO membership registration, IPI number issuance, work registration sync, real-time performing rights clearance, songwriter credit display, webhook notifications for registration status updates
- **Standards:** REST; webhook-based event notifications
- **Authentication:** Not publicly documented; partner agreement required

### Tuned Global
- **Description:** B2B music technology provider offering 500+ APIs across metadata, streaming services, authentication, and rights management, used by music services and distributors to power catalog delivery, metadata, and licensing compliance.
- **API Documentation:** https://www.tunedglobal.com/streaming-services/streaming-music-api-for-apps
- **Standards:** REST/JSON; Swagger/OpenAPI interfaces; supports music catalog ingestion with rights metadata and territory-level licensing rules
- **Authentication:** Not publicly documented; commercial agreement required

### LabelGrid
- **Description:** Full-stack label management platform with comprehensive API coverage of catalog ingestion, release scheduling, DSP delivery, royalty accounting, analytics, and payment processing.
- **API Documentation:** https://labelgrid.com/features/white-label-and-api/
- **Standards:** REST; interactive documentation with request examples, response schemas, and error codes per endpoint
- **Authentication:** API key; commercial agreement required

### Revelator Rights API (sub-section)
- **Description:** Within the Revelator platform, the Rights module (`/accounting/contract/*`) models rights splits at account, label, artist, release, or track level with hierarchical precedence. Supports percentage-based revenue shares, territory and DSP scope filtering, and delivery type segmentation.
- **API Documentation:** https://api-docs.revelator.com/en/rights/
- **Key Endpoints:** `POST /accounting/contract/save`, `GET /accounting/contracts`, `GET /accounting/contracts/{contractId}`
- **Standards:** REST/JSON; paginated responses
- **Authentication:** Parent account access token

### Open Music Initiative (OMI)
- **Description:** Nonprofit open-source API specification (Berklee/MIT) for linking sound recordings to musical compositions and rights holders to facilitate royalty discovery; built on a federated search model.
- **API Documentation:** https://open-music.org/our-api · Apiary: https://docs.omi01.apiary.io
- **SDKs/Libraries:** GitHub: https://github.com/omi · Reference implementation: https://github.com/COALAIP/omi-mvi-1.0
- **Developer Guide:** https://open-music.org/our-open-protocols
- **Standards:** REST/HTTP; MVI 1.0 specification; federated queries to MusicBrainz and Wikipedia
- **Authentication:** Open specification; implementation-dependent

### Spotify Web API
- **Description:** Comprehensive REST API providing access to Spotify's music catalog, streaming, playlist management, and metadata. Relevant as a reference implementation of OAuth 2.0 music API authentication and as a data source for usage/stream data in rights systems.
- **API Documentation:** https://developer.spotify.com/documentation/web-api
- **Developer Guide:** https://developer.spotify.com/documentation/web-api/concepts/authorization
- **Standards:** REST/JSON; OAuth 2.0 with Authorization Code + PKCE; OpenAPI-documented; access tokens expire after 3600 seconds
- **Authentication:** OAuth 2.0 (Authorization Code, Client Credentials, Implicit Grant); scopes-based access

### Apple Music API
- **Description:** REST API for accessing the Apple Music catalog, managing user libraries, and streaming. Used as a reference for JWT-based authentication in music APIs and as an integration point for usage reporting.
- **API Documentation:** https://developer.apple.com/documentation/applemusicapi
- **Standards:** REST/JSON; JWT authentication (MusicKit JWT)
- **Authentication:** JWT signed with a MusicKit private key issued through the Apple Developer Program

### BMAT Music Operating System
- **Description:** Music monitoring and rights management platform for broadcasters, digital services, collecting societies, publishers, and labels. Provides automated reporting, usage tracking, and rights attribution across broadcast and digital channels.
- **API Documentation:** https://www.bmat.com/ (commercial partner access)
- **Standards:** REST; commercial API with partner agreements required
- **Authentication:** Not publicly documented; partner agreement required

---

## Notes

**Fragmented Standards Landscape**
The music industry's metadata standards have historically been maintained in silos: DDEX covers the label/DSP supply chain, CISAC CWR covers publishing/PRO registration, and no single unified standard bridges both. The DDEX MWDR standard (specifically the MWN sub-standard) is the most current attempt at harmonisation, though CWR 2.2 remains the operational standard for PRO submissions and is not yet superseded.

**ISRC vs. ISWC Confusion**
A significant source of data errors in rights systems is conflating recording identifiers (ISRC, assigned per recording) with work identifiers (ISWC, assigned per composition). A robust data model must maintain the many-to-one relationship from ISRCs to ISWCs and ensure both are surfaced correctly in export pipelines to PROs and DSPs.

**Emerging: AI-Generated Works Registration**
As of October 2025, ASCAP, BMI, and SOCAN have aligned policies permitting registration of partially AI-generated musical compositions, provided human authorship elements are present. Systems must accommodate a new "AI contribution" field in work metadata to comply with PRO registration requirements.

**Emerging: Blockchain / Tokenised Rights**
Several platforms (Revelator's Royalty Tokens feature, COALAIP) are experimenting with on-chain representations of rights shares. No ISO or W3C standard has been ratified for this yet; the W3C Verifiable Credentials Data Model is the closest applicable standard. This is an evolving area to monitor but is not yet a production requirement for mainstream rights management systems.

**PRO APIs are Limited**
ASCAP, BMI, PRS, and most collecting societies do not offer public or partner APIs for programmatic work registration. CWR file upload via SFTP remains the dominant integration method. AllTrack is a notable exception with its 2024 REST API launch; other PROs are expected to follow. Rights management platforms should abstract the registration workflow to support CWR export now and REST-based registration later.
