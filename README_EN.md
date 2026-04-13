# Technical Documentation: Electronic Money Financial System (EMFS)

This document provides a detailed technical overview of the architecture, data flows, and components of the distributed microservices system for electronic money management.

---

## 1. System Architecture

The system follows an **Event-Driven Microservices** pattern using **Google Cloud Pub/Sub** as the central message bus.

### Architecture Diagram (Mermaid)

```mermaid
graph TD
    subgraph "User Layer"
        UI[Web Interface - Port 8085]
        CS[Client Simulator]
    end

    subgraph "Application Layer (REST Microservices)"
        Auth[AuthService - Port 8081]
        Account[AccountService - Port 8080]
        Report[ReportService - Port 8084]
    end

    subgraph "Messaging Infrastructure (Pub/Sub)"
        Topic_TX[Topic: tx-events]
        Topic_Conf[Topic: tx-confirmed]
    end

    subgraph "Asynchronous Processing Layer"
        TX_Service[TransactionService]
        Audit_Service[AuditService]
    end

    subgraph "Data Layer"
        DB[(Google Cloud SQL - PostgreSQL)]
    end

    subgraph "Monitoring"
        CPU[CPU Monitor - Lanterna TUI]
    end

    %% Relationships
    UI --> Auth
    UI --> Account
    UI --> Report
    CS --> Account
    
    Account --> Topic_TX
    Topic_TX --> TX_Service
    TX_Service --> DB
    TX_Service --> Topic_Conf
    
    Topic_Conf --> Audit_Service
    Audit_Service --> DB
    
    Report --> DB
    Auth --> DB
    Account --> DB
    
    CPU -.->|Monitors| Auth
    CPU -.->|Monitors| Account
    CPU -.->|Monitors| TX_Service
```

---

## 2. Transaction Lifecycle

This diagram illustrates the lifecycle of a financial operation (e.g., a transfer) from the user's request until it is confirmed and audited.

```mermaid
sequenceDiagram
    participant U as User / Simulator
    participant AC as AccountService
    participant PS as Google Cloud Pub/Sub
    participant TX as TransactionService
    participant DB as PostgreSQL (Cloud SQL)
    participant AU as AuditService

    U->>AC: POST /api/accounts/transfer
    Note over AC: Validates balance & triggers event
    AC->>PS: Publish event to 'tx-events'
    AC-->>U: 202 Accepted (Transaction ID)

    PS->>TX: Push/Pull event
    Note over TX: Business logic processing (Add/Subtract)
    TX->>DB: UPDATE balances (Atomicity)
    TX->>PS: Publish confirmation to 'tx-confirmed'

    PS->>AU: Push/Pull confirmation
    AU->>DB: INSERT into audit_logs
    Note over AU: Immutable historical record
```

---

## 3. Component Breakdown

### 3.1 AuthService (Port 8081)
Responsible for perimeter security.
- **Technology:** Spring Security + JJWT.
- **Core Functions:** User registration, login, JWT token generation, and validation.
- **Security:** BCrypt hashing for password storage.

### 3.2 AccountService (Port 8080)
The entry point for all account-related operations.
- **Operations:** Balance inquiry, deposits, withdrawals, and transfers.
- **Role:** Acts as an event producer. It does not modify balances directly for critical operations; instead, it delegates to `TransactionService` via Pub/Sub to ensure scalability and fault tolerance.

### 3.3 TransactionService (Asynchronous)
The central processing engine.
- **Mechanism:** Listens to the `tx-events` topic.
- **Guarantees:** Ensures eventual consistency and exactly-once processing of transactions.
- **Scalability:** Supports multiple replicas to handle high-traffic bursts.

### 3.4 AuditService (Asynchronous)
Ensures full system traceability.
- **Mechanism:** Listens to the `tx-confirmed` topic.
- **Function:** Logs every completed transaction into an immutable audit table for reporting and security auditing purposes.

### 3.5 ReportService (Port 8084)
A read-optimized layer for the administration panel.
- **Core Functions:** Aggregates metrics (total system balance, transactions per minute, user balance rankings).

---

## 4. Data Model (ER)

The system utilizes a PostgreSQL relational database with the following schema:

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : "owns"
    ACCOUNT ||--o{ TRANSACTION : "generates"
    TRANSACTION ||--|| AUDIT : "logs"

    USER {
        string curp PK
        string name
        string password
        string role
    }

    ACCOUNT {
        int id PK
        string curp FK
        double bank_balance
        double wallet_balance
    }

    TRANSACTION {
        uuid id PK
        int source_account
        int destination_account
        double amount
        timestamp created_at
    }
```

---

## 5. Monitoring & Tools

### CPU Monitor (Lanterna TUI)
A terminal-based application that uses the `OSHI` library to retrieve hardware metrics in real-time. It provides a visual dashboard of the workload across all distributed microservices.

### Client Simulator
A load testing tool designed to simulate high-concurrency environments:
- `n`: Number of clients.
- `h`: Concurrent threads.
- `p`: Initial budget per client.
- `t`: Targeted transactions per minute.

---

## 6. Management Commands

| Action | Script |
|--------|--------|
| Start All Services | `./run-all-services.sh` |
| Stop All Services | `./stop-all-services.sh` |
| Check Service Status | `./check-services.sh` |
| Manual Testing Guide | `GUIA_PRUEBAS_MANUAL.md` |

---
*Documentation automatically generated by Gemini CLI for the Distributed Systems Final Project.*
