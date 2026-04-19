# Technology Stack

## Core Technologies

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Runtime | .NET 8 LTS | All services |
| APIs | ASP.NET Core Minimal APIs | KYC, Case Management, CBS Simulator |
| Workers | .NET Worker Services | Transaction Monitor, SAR Service |
| UI | Blazor Server | Portal |
| Database | MS SQL Server | Single instance, per-service schemas |
| Messaging | RabbitMQ + MassTransit | Event-driven pub/sub |
| ORM | Dapper | Lightweight DB access |
| Migrations | FluentMigrator | Code-first schemas |
| Encryption | AES-256 (.NET) | SAR content at rest |
| Auth | API Key middleware | Swappable for OAuth2/OIDC |
| Logging | OpenTelemetry + Seq | Structured logging, correlation IDs |
| Containers | Docker Compose + K8s | Both dev & prod |

## Design Goals

| Goal | Implementation |
|------|-----------------|
| Event-driven flows | RabbitMQ pub/sub across all flows |
| Service independence | Each service owns its DB tables |
| Swappable messaging | RabbitMQ → MSK/ServiceBus via interface |
| Swappable cloud DB | MS SQL → RDS/Azure SQL via connection string |
| K8s ready | Helm-deployable, same images as docker-compose |
| Audit trail | Every event in AuditLog table + Seq |
| Rule engine | Threshold + pattern rules in TXM |
| Auth flexibility | API Key → OAuth2/OIDC via middleware |