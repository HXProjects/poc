## FLOW A, B, C: Core Business Workflows

**Overview:** The three main system flows showing how customer data flows through the compliance system. Flow B assesses customer risk, Flow A monitors transactions against rules, and Flow C escalates suspicious activity into cases and generates SARs.

- **Flow B**: KYC officer assesses customer risk profile and stores risk classification
- **Flow A**: Transactions analyzed against rule engine (amount thresholds, velocity, cross-border, etc.)
- **Flow C**: Rules triggered → alerts created → cases escalated → SAR generation initiated

```mermaid
sequenceDiagram
    participant UI as 🖥️ Portal
    participant CBS as 🏦 CBS Simulator
    participant KYC as 👤 KYC Service
    participant TXM as ⚙️ Tx Monitor
    participant MQ as 🐇 RabbitMQ
    participant CMS as 📁 Case Mgmt
    participant SAR as 📄 SAR Service
    participant DB as 🗄️ AML DB

    Note over UI,DB: ── FLOW B: KYC Risk Assessment ──
    UI->>KYC: POST /api/risk-assessments\n{customerId: "uuid"} [API Key]
    KYC->>CBS: GET /customers/{customerId}
    CBS-->>KYC: {cbsRiskCategory, customerSince}
    KYC->>KYC: Calculate AML risk score\nfrom CBS category + rules
    KYC->>DB: UPSERT RiskProfiles\n{customerId, riskScore, riskLevel}
    KYC->>MQ: Publish CustomerRiskAssessed\n{customerId, riskLevel, riskScore}
    MQ-->>TXM: Load risk context for customer

    Note over UI,DB: ── FLOW A: Transaction Monitoring ──
    UI->>TXM: POST /api/transactions\n{customerId, amount, currency,\nfromAccountRef, toAccountRef} [API Key]
    TXM->>DB: SELECT RiskProfiles WHERE customerId
    TXM->>TXM: Apply Rule Engine\n• Amount threshold\n• Velocity check\n• Cross-border flag\n• Risk score weighting
    TXM->>DB: INSERT Transactions\n(IDs + refs only)
    TXM->>MQ: Publish TransactionAnalyzed

    alt Rules Triggered
        TXM->>MQ: Publish AlertTriggered\n{alertId, transactionId,\ncustomerId, ruleName, severity}
        MQ-->>CMS: Consume AlertTriggered
        CMS->>DB: INSERT Alert + Case\n(customerId ref only)
        CMS->>MQ: Publish CaseCreated

        Note over UI,DB: ── FLOW C: Case + SAR ──
        MQ-->>SAR: Consume CaseCreated
        SAR->>CBS: GET /customers/{customerId}
        CBS-->>SAR: Full PII (for SAR only)
        SAR->>SAR: Build + Encrypt SAR draft
        SAR->>DB: INSERT SarDraft\n(encrypted content)
        SAR->>MQ: Publish SarDraftCreated
    end
```

---

## FLOW D: Analyst Case Review

**Overview:** How compliance analysts interact with the system to review and investigate flagged cases. Demonstrates the **fetch-on-demand pattern** where customer names are retrieved from CBS only when needed for display, ensuring PII is not stored in the AML database.

**Key Pattern:** AML DB stores only IDs and transaction metadata. Customer names are fetched from CBS at display time for the analyst's UI.

```mermaid
sequenceDiagram
    participant ANA as 🧑 AML Analyst
    participant UI as 🖥️ Blazor Portal
    participant CMS as 📁 Case Management
    participant DB as 🗄️ AML Database
    participant CBS as 🏦 CBS Simulator

    Note over ANA,CBS: ── FLOW D: Case Review & Enrichment ──

    ANA->>UI: Open Case list
    UI->>CMS: GET /api/cases [API Key]
    CMS->>DB: SELECT cases\n(customerId, alertId, status only)
    DB-->>CMS: [{caseId, customerId: "uuid", status}]

    loop For each case (or batch)
        CMS->>CBS: GET /api/customers/{customerId}
        CBS-->>CMS: {fullName, riskCategory}
    end

    CMS-->>UI: Enriched case view\n(name resolved, not stored)
    UI-->>ANA: Case list with names shown
```

---

## FLOW E (Deep Dive): SAR Generation & Encryption

**Overview:** Detailed technical flow for SAR generation, showing how sensitive customer data is handled securely during SAR document creation. This is the **implementation detail** of FLOW C, highlighting encryption at rest and just-in-time PII retrieval.

**Security Architecture Shown:**
- PII fetched from CBS only when building the SAR
- SAR content encrypted with AES-256 before storage
- Only encrypted blob stored in AML database
- No plaintext sensitive data in logs or database

```mermaid
sequenceDiagram
    participant CMS as 📁 Case Management
    participant MQ as 🐇 RabbitMQ
    participant SAR as 📄 SAR Service
    participant CBS as 🏦 CBS Simulator
    participant DB as 🗄️ AML Database

    Note over CMS,DB: ── FLOW E (Deep Dive): SAR Generation & Encryption ──

    CMS->>MQ: Publish CaseCreated {caseId, customerId}
    MQ-->>SAR: Consume CaseCreated

    SAR->>DB: SELECT case + alert details\n(no PII — IDs only)
    SAR->>CBS: GET /customers/{customerId}
    CBS-->>SAR: Full customer PII (for SAR only)

    SAR->>SAR: Build SAR document\nwith customer details
    SAR->>SAR: AES-256 Encrypt SAR content\n(key from env/k8s secret)
    SAR->>DB: INSERT SarDraft\n{caseId, customerId,\nencryptedContent, status}
    SAR->>MQ: Publish SarDraftCreated

    Note over SAR: PII fetched just-in-time\nfor SAR generation only
    Note over SAR: AES-256 encryption\nat rest
    Note over DB: Only encrypted blob stored\nin database
```