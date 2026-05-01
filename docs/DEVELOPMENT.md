# Developer Setup Guide

## Prerequisites

- JDK 17+ (Temurin recommended)
- Maven 3.9+ (or use included `./mvnw` wrapper)
- Node.js 22+ and Yarn 1.22+ (for Angular UI)
- Docker (for LDAP and database)
- Git

## Quick Start (H2 — no external DB)

```bash
git clone https://github.com/dcm4che/dcm4chee-arc-light.git
cd dcm4chee-arc-light
./mvnw install -Ddb=h2
```

This builds all 130 modules with the embedded H2 database profile.

## Build with PostgreSQL

```bash
./mvnw install -Ddb=psql
```

Other database options: `mysql`, `oracle`, `sqlserver`, `db2`, `firebird`, `h2`.

## Build with Security (Keycloak)

```bash
# Secured UI only
./mvnw install -Ddb=psql -Dsecure=ui

# Secured UI + REST endpoints
./mvnw install -Ddb=psql -Dsecure=all
```

## Frontend Development

```bash
cd dcm4chee-arc-ui2
yarn install
yarn start          # ng serve on http://localhost:4200
```

The Angular app proxies API requests to the WildFly backend.

## IDE Setup (IntelliJ IDEA)

1. Open the root `pom.xml` as a Maven project
2. Set Project SDK to JDK 17
3. Maven settings: add `-Ddb=psql` to Maven Runner VM options
4. For WildFly integration:
   - Install the WildFly plugin
   - Configure a local WildFly 39.0.1.Final server
   - Deploy the EAR artifact from `dcm4chee-arc-ear/target/`

## IDE Setup (Eclipse)

1. Import as Existing Maven Projects
2. Select the root `pom.xml`
3. Eclipse will import all 130 modules
4. Configure WildFly Tools for deployment

## Running a Development Instance

A full development instance requires:

1. **LDAP server** — Configuration store (DICOM Part 15)
2. **Database** — PostgreSQL, MySQL, or H2
3. **WildFly** — Application server
4. **Keycloak** (optional) — Authentication

See the [dcm4chee-arc-light wiki](https://github.com/dcm4che/dcm4chee-arc-light/wiki)
for detailed deployment instructions with Docker Compose.

## Remote Debugging

```bash
# Start WildFly with debug port
./standalone.sh --debug 8787

# IntelliJ: Run > Edit Configurations > Remote JVM Debug
# Host: localhost, Port: 8787
```

## DICOM Traffic Inspection

Use dcm4che CLI tools for testing:

```bash
# Send a DICOM file (C-STORE)
storescu -c DCM4CHEE@localhost:11112 /path/to/dicom/file.dcm

# Query studies (C-FIND)
findscu -c DCM4CHEE@localhost:11112 -L STUDY -m PatientName=DOE*

# Retrieve study (C-MOVE)
movescu -c DCM4CHEE@localhost:11112 --dest STORESCP -m StudyInstanceUID=1.2.3
```

## Project Structure

```
dcm4chee-arc-light/
├── pom.xml                    # Parent POM (130 modules)
├── dcm4chee-arc-conf/         # Configuration model
├── dcm4chee-arc-entity/       # JPA entities
├── dcm4chee-arc-store/        # C-STORE / STOW-RS
├── dcm4chee-arc-query/        # C-FIND / QIDO-RS
├── dcm4chee-arc-retrieve/     # C-MOVE / WADO-RS
├── dcm4chee-arc-wado/         # WADO-RS REST
├── dcm4chee-arc-stow/         # STOW-RS REST
├── dcm4chee-arc-qido/         # QIDO-RS REST
├── dcm4chee-arc-war/          # WAR (all REST endpoints)
├── dcm4chee-arc-ear/          # EAR (deployment)
├── dcm4chee-arc-ui2/          # Angular frontend
├── docs/                      # Documentation
│   ├── ARCHITECTURE.md
│   ├── swagger/               # OpenAPI specs
│   └── dbschema-5.x/          # SchemaSpy output
└── .github/workflows/ci.yml   # CI pipeline
```

See [docs/ARCHITECTURE.md](ARCHITECTURE.md) for the full module map.

## Common Issues

**Build fails with missing dcm4che artifacts**: The dcm4che 5.35.0 library
may not be on Maven Central yet. Add the dcm4che Maven repository:

```xml
<repository>
    <id>dcm4che</id>
    <url>https://www.dcm4che.org/maven2</url>
</repository>
```

**Frontend build fails**: Ensure Node.js 22+ and Yarn 1.22+ are installed.
Run `yarn install` in `dcm4chee-arc-ui2/` before the Maven build.

**Database-specific build errors**: Ensure you pass the correct `-Ddb=` flag.
The entity module uses database-specific JPA mapping files.
