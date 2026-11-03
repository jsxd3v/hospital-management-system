# Architecture

## System overview

```mermaid
flowchart TB
    subgraph Client
        SPA[React 19 + TypeScript SPA]
        TV[Waiting-room display<br/>no sign-in, numbers only]
        V[Public document verification<br/>QR code]
    end

    subgraph Server
        NG[Nginx<br/>TLS, static files]
        GU[Gunicorn]
        DJ[Django 5.2 + DRF<br/>service layer, permissions, audit]
        TM[systemd timers<br/>reminders, backups, partitions, host checks]
    end

    subgraph Data
        PG[(PostgreSQL 16<br/>constraints, triggers, partitions)]
        FS[(Patient documents<br/>outside web root)]
        BK[(Encrypted backups<br/>object storage)]
    end

    SPA --> NG
    TV --> NG
    V --> NG
    NG --> GU --> DJ
    DJ --> PG
    DJ --> FS
    TM --> DJ
    TM --> BK
    DJ -. email / SMS .-> EXT[Email and SMS providers]
```

## Backend modules

The Django backend has 23 apps, each with its own models, services, permissions and tests:

```
accounts      audit         departments   patients      scheduling
clinical      diagnostics   pharmacy      inpatient     emergency
billing       notifications documents     purchasing    nursing
theatre       queues        certificates  printouts     chat
reports       core
```

Business rules live in a **service layer** (`services.py`). Access rules are kept in separate
`permissions.py` and `scoping.py` modules so each role only reaches the patients it should.

## Layered enforcement

```mermaid
flowchart LR
    A[UI<br/>role-aware screens] --> B[API permissions<br/>per role and scope]
    B --> C[Service layer<br/>business rules, locking]
    C --> D[Database<br/>constraints, triggers, role grants]
```

Each important rule is checked in more than one layer. For example, a signed consultation is refused by the
service layer **and** by a database trigger, so even a bug in application code can't change it.

## Patient visit flow

```mermaid
sequenceDiagram
    participant R as Reception
    participant D as Doctor
    participant L as Lab
    participant P as Pharmacy
    participant B as Billing
    R->>R: Register patient and book appointment
    R->>D: Check in
    D->>D: Vitals, notes, diagnosis
    D->>L: Order lab tests
    L-->>D: Release results
    D->>P: Sign prescription
    P->>P: Dispense (earliest expiry first)
    P-->>B: Charges raised automatically
    B->>B: Issue invoice, take payment, print receipt
```

## Deployment pipeline

```mermaid
flowchart LR
    PUSH[git push main] --> BT[Backend tests<br/>ruff, mypy, pytest]
    PUSH --> FT[Frontend tests<br/>ESLint, tsc, Vitest]
    PUSH --> DT[Deploy script tests]
    BT & FT & DT --> E2E[Playwright E2E]
    E2E --> PKG[Package release]
    PKG --> DEP[Deploy, smoke test,<br/>auto-rollback on failure]
```
