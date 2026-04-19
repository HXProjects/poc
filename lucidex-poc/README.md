lucidex-poc/
├── 📄 docker-compose.yml
├── 📄 docker-compose.override.yml
├── 📁 k8s/
│   ├── 📁 namespaces/          # lucidex, lucidex-infra namespace definitions
│   ├── 📁 infra/               # RabbitMQ, MSSQL, Seq manifests
│   ├── 📁 services/            # Per-service Deployment + Service YAMLs
│   └── 📁 config/              # ConfigMaps + Secrets
├── 📁 src/
│   ├── 📁 shared/
│   │   ├── 📁 Lucidex.Contracts/
│   │   │   ├── Events/         # TransactionSubmitted, Analyzed, AlertTriggered, etc.
│   │   │   ├── DTOs/           # No PII — CustomerId refs only
│   │   │   └── Interfaces/     # ICustomerDataSource (CBS integration)
│   │   ├── 📁 Lucidex.Domain/
│   │   │   ├── Entities/       # Transaction, Alert, Case, RiskProfile, SarDraft (no PII)
│   │   │   └── Rules/          # ITransactionRule, AmountThresholdRule, VelocityRule, etc.
│   │   └── 📁 Lucidex.Infrastructure/
│   │       ├── Persistence/    # DapperConnectionFactory, FluentMigrator
│   │       ├── Messaging/      # RabbitMqPublisher, RabbitMqConsumerBase
│   │       ├── Http/           # CbsHttpClient (typed HttpClient)
│   │       └── Security/       # ApiKeyMiddleware, SarEncryptionService (AES-256)
│   └── 📁 services/
│       ├── 📁 Lucidex.TransactionMonitor/   # .NET Worker Service (:8081)
│       ├── 📁 Lucidex.KycService/           # ASP.NET Core Web API (:8082)
│       ├── 📁 Lucidex.CaseManagement/       # ASP.NET Core Web API (:8083)
│       ├── 📁 Lucidex.SarService/           # .NET Worker Service (:8084)
│       ├── 📁 Lucidex.Portal/               # Blazor Server UI (:8080)
│       └── 📁 Lucidex.CbsSimulator/         # In-memory CBS mock (:8085)
├── 📁 tests/
│   ├── 📁 Lucidex.UnitTests/
│   │   ├── Rules/
│   │   └── Domain/
│   └── 📁 Lucidex.IntegrationTests/         # Testcontainers
└── 📄 Lucidex.sln


## Core Components

### 6 Service Containers:

**Lucidex Portal** (Blazor Server, :8080)  
UI for compliance officers to view transactions, cases, and SAR drafts

**Transaction Monitor** (.NET Worker Service, :8081)  
Analyzes transactions against rule engine, calculates risk scores, publishes alerts

**KYC / Risk Service** (.NET Web API, :8082)  
Fetches customer risk category from CBS, assesses risk levels, stores risk profiles

**Case Management** (.NET Web API, :8083)  
Creates and manages compliance cases triggered by alerts, enriches case data from CBS

**SAR Service** (.NET Worker Service, :8084)  
Generates Suspicious Activity Reports from cases, encrypts at rest

**CBS Simulator** (.NET Web API, :8085)  
In-memory mock of Core Banking System, provides customer PII on-demand

## CBS Simulator Architecture

**Overview:** The CBS Simulator is the source of truth for all customer PII (names, addresses, personnummer, account references). It's in-memory only and designed as a **perfect swap point for real CBS integration**.

See [CBSSimulator.md](NoteRoom/CBSSimulator.md) for:
- Complete REST endpoint documentation
- Integration patterns with each service
- Seed data details
- Production integration guide
- Security & privacy considerations

**Key Pattern:** PII is **never stored** in Lucidex. Services fetch from CBS only when needed:
- 🔑 **KYC** uses `cbsRiskCategory` for risk scoring
- 👁️ **Case Management** fetches names for analyst UI (display-time only)
- 🔒 **SAR Service** fetches PII for document generation, then encrypts before storage

## Three Business Flows
### FLOW A: Transaction Monitoring

User submits transaction via Portal → Transaction Monitor analyzes it → Rule engine checks thresholds/patterns → Publishes TransactionAnalyzed event
### FLOW B: KYC Risk Assessment

Officer assesses customer via Portal → KYC Service fetches risk category from CBS → Stores RiskProfile → Publishes CustomerRiskAssessed event
### FLOW C: Case & SAR

If rules trigger → Alert created → Case Management creates case → SAR Service generates encrypted SAR draft → Publishes SarDraftCreated

## Event-Driven Architecture

Services communicate via domain events published to RabbitMQ:
- **TransactionSubmitted** — New transaction received
- **TransactionAnalyzed** — Analysis complete with risk score
- **AlertTriggered** — Rules matched, alert generated
- **CustomerRiskAssessed** — Risk profile updated
- **CaseCreated** — Compliance case opened
- **SarDraftCreated** — SAR draft generated and encrypted

All events logged to **Seq** for structured monitoring and **AuditLog** table for compliance.

## Data Storage Pattern

| Data | Stored In | Notes |
|------|-----------|-------|
| **Transactions** | Lucidex DB | IDs + refs only, no PII |
| **Alerts** | Lucidex DB | CustomerId ref, rule metadata |
| **Cases** | Lucidex DB | CustomerId ref, status only |
| **RiskProfiles** | Lucidex DB | Risk scores, no PII |
| **SarDrafts** | Lucidex DB | AES-256 encrypted content |
| **Customer PII** | CBS Simulator | Names, addresses, personnummer |
| **Audit Events** | Seq + AuditLog | All domain events, pseudonymised |

## Authentication

All API calls require **API Key authentication** via middleware. Swappable for OAuth2/OIDC in production.