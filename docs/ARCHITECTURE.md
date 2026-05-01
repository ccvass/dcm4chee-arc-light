# Architecture — dcm4chee-arc-light 5.35.0

DICOM medical imaging archive built on Jakarta EE / WildFly.

## Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Language | Java | 17+ |
| App Server | WildFly | 39.0.1.Final |
| ORM | Hibernate | 6.6.40.Final |
| REST | JAX-RS (RESTEasy) | WildFly-provided |
| Auth | Keycloak (optional) | 25.0.6 |
| Frontend | Angular + Material | 20.x |
| Config | LDAP (DICOM Part 15) | — |
| DICOM lib | dcm4che | 5.35.0 |

Supported databases: PostgreSQL (default), MySQL, MariaDB, Oracle, SQL Server, DB2, Firebird, H2.

## Layered Architecture

```
  +-----------------------+
  |   Angular UI (ui2)    |   Browser
  +-----------+-----------+
              |
  +-----------v-----------+
  |  REST Endpoints (-rs) |   JAX-RS @Path resources
  |  QIDO / STOW / WADO  |   30 modules in WAR
  +-----------+-----------+
              |
  +-----------v-----------+
  |  Business Services    |   @ApplicationScoped CDI beans
  |  Store / Query / Ret  |   Interface + Impl pattern
  +-----------+-----------+
              |
  +-----------v-----------+
  |  EJB Layer            |   @Stateless, container-managed TX
  |  StoreServiceEJB etc  |   @PersistenceContext
  +-----------+-----------+
              |
  +-----------v-----------+
  |  JPA Entities         |   32 @Entity classes
  |  Patient > Study >    |   DICOM Information Model
  |  Series > Instance    |
  +-----------+-----------+
              |
  +-----------v-----------+
  |  Database             |   PostgreSQL / MySQL / Oracle / ...
  +---+---------------+---+
      |               |
  +---v---+     +-----v-----+
  | LDAP  |     |  Storage   |
  | Config|     | FS/S3/Cloud|
  +-------+     +-----------+
```

## Module Categories (130 modules)

### Foundation (shared by nearly everything)

- `dcm4chee-arc-conf` — Configuration model (ArchiveDeviceExtension)
- `dcm4chee-arc-entity` — JPA entities (Patient, Study, Series, Instance, Location, Task)
- `dcm4chee-arc-event` — CDI event model
- `dcm4chee-arc-service` — Core archive lifecycle (@Singleton @Startup)
- `dcm4chee-arc-keycloak` — Keycloak integration wrapper

### DICOM Services (SCP/SCU pattern)

Each DICOM service follows the pattern: `-scp` (inbound) / `-scu` (outbound) / core EJB.

| Service | SCP | SCU | Core | REST |
|---------|-----|-----|------|------|
| C-STORE | store-scp | store-scu | store | stow |
| C-FIND | query-scp | query-scu | query | qido |
| C-MOVE/GET | retrieve-scp | retrieve-scu | retrieve | wado |
| MPPS | mpps-scp | mpps-scu | mpps | — |
| MWL | mwl-scp | — | — | mwl-rs |
| UPS | ups-scp | — | ups | ups-rs |

### Storage Backends (pluggable SPI)

- `storage` — Abstract SPI
- `storage-filesystem` — Local filesystem
- `storage-aws-s3` — AWS S3
- `storage-cloud` — jclouds (multi-cloud)
- `storage-ejb` — DB operations for storage metadata

### Export Framework (pluggable providers)

- `export` — SPI + scheduler
- `export-dicom` / `export-stow` / `export-wado` / `export-storage` — Transport
- `export-fhir` / `export-xdsi` / `export-dcm2hl7` — Protocol conversion

### Interoperability

- `hl7` / `hl7-rs` / `hl7-psu` — HL7 v2 messaging
- `fhir-client` / `fhir-rs` / `fhir-util` — FHIR R4
- `pdq-*` (6 modules) — Patient Demographics Query (DB, DICOM, FHIR, HL7, X-Road)

## Core Dependency Flow

```
  conf ──> entity ──> event
    |         |
    v         v
  service ──> code, coerce, keycloak
    |
    v
  store ──> query ──> query-util
    |                    |
    v                    v
  retrieve ──> storage, compress
    |
    v
  wado, qido, stow (REST layer)
    |
    v
  WAR ──> EAR (deployment)
```

Key coupling: `store` depends on `query` for MWL matching during ingest.

## Data Model (DICOM Information Model)

```
  Patient (1)
    |
    +-- Study (N)
          |
          +-- Series (N)
                |
                +-- Instance (N)
                      |
                      +-- Location (N)  [storage references]
```

Supporting entities: Task (unified queue), MWLItem, MPPS, UPS, CodeEntity,
PatientID, PersonName, StgCmtResult, Subscription.

## Deployment

```
  WildFly 39
    |
    +-- dcm4chee-arc.ear
    |     +-- dcm4chee-arc.war        (REST endpoints)
    |     +-- dcm4chee-arc-ui2.war    (Angular UI)
    |     +-- ~85 EJB/JAR modules
    |
    +-- LDAP (OpenLDAP/ApacheDS)      (configuration)
    +-- Database (PostgreSQL/...)      (data)
    +-- Keycloak (optional)           (authentication)
    +-- Storage (filesystem/S3/cloud) (DICOM files)
```

Build profiles:
- `-Ddb=psql|mysql|oracle|h2|...` — Database selection
- `-Dsecure=ui|all` — Enable Keycloak authentication

## Cross-Cutting Concerns

- **CDI Events**: 69 event producers, 53 observers — loose coupling for audit, HL7 PSU, IAN, prefetch
- **Audit**: DICOM audit trail via `dcm4chee-arc-audit` (ATNA/IHE)
- **Metrics**: Micrometer-based metrics via `dcm4chee-arc-metrics`
- **CORS**: Configurable via `ACCESS_CONTROL_ALLOW_ORIGIN` env var
- **Security Headers**: X-Content-Type-Options, X-Frame-Options, CSP, HSTS
