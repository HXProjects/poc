# System Architecture

## Overview

Lucidex is a **6-service event-driven compliance system** built on .NET 8 with RabbitMQ messaging, MS SQL Server persistence, and strict PII isolation.

## Services (6 Containers)

### Application Services

**🖥️ Lucidex Portal** — Blazor Server UI (:8080)  
Compliance officer interface for case review, transaction monitoring, SAR submission

**⚙️ Transaction Monitor** — .NET Worker Service (:8081)  
Asynchronous transaction analysis with rule engine (amount thresholds, velocity, cross-border checks)

**👤 KYC / Risk Service** — ASP.NET Core Web API (:8082)  
Customer risk profiling via CBS integration, stores RiskProfiles in Lucidex DB

**📁 Case Management** — ASP.NET Core Web API (:8083)  
Compliance case CRUD, alert aggregation, analyst-facing API

**📄 SAR Service** — .NET Worker Service (:8084)  
Suspicious Activity Report generation with AES-256 encryption

**🏦 CBS Simulator** — ASP.NET Core Web API (:8085)  
In-memory mock of Core Banking System, provides all customer PII on-demand  
*See [CBSSimulator.md](CBSSimulator.md) for detailed documentation*

### Infrastructure (3 Containers)

**🐇 RabbitMQ** (:5672 / :15672)  
Event broker for async pub/sub messaging

**🗄️ MS SQL Server** (:1433)  
Single database instance with per-service schemas

**🔍 Seq** (:5341 / :8090)  
Structured logging aggregator with correlation ID tracking

## Communication Flow

```
Portal (HTTP + API Key Auth)
    ↓
    ├→ Transaction Monitor (analysis)
    ├→ KYC Service (risk profiles)
    ├→ Case Management (case CRUD)
    └→ SAR Service (report generation)

        ↓ (publish events)
    
    RabbitMQ
        ↓ (subscribe)
    
    ├→ Case Management (AlertTriggered)
    ├→ SAR Service (CaseCreated)
    └→ Seq (all events)

CBS Simulator (external PII source)
    ↑ (on-demand fetch)
    ├← KYC Service (cbsRiskCategory)
    ├← Case Management (customer names for display)
    └← SAR Service (full PII for document generation)
```

## Data Architecture

**Lucidex DB stores:**
- Transactions (IDs + refs only, no PII)
- Alerts (metadata + rules only)
- Cases (status + investigation notes)
- RiskProfiles (scores + categories)
- SarDrafts (encrypted content)
- AuditLog (pseudonymised events)

**CBS Simulator (external) stores:**
- Customer names, addresses, personnummer
- Account references (IBANs)
- Risk categories
- Customer since date

## Related Documentation

- **[FlowDiagram.md](FlowDiagram.md)** — 5 business process flows (A-E)
- **[Database.md](Database.md)** — ER diagram with schema details
- **[DomainEvents.md](DomainEvents.md)** — Event class hierarchy
- **[Solution.md](Solution.md)** — Project structure diagram
- **[ComponentDiagram.md](ComponentDiagram.md)** — System topology
- **[Kubernetes.md](Kubernetes.md)** — K8s deployment structure
- **[CBSSimulator.md](CBSSimulator.md)** — CBS integration & swap
- **[Technologies.md](Technologies.md)** — Tech stack & design goals