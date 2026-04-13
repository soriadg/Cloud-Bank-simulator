# Documentación Técnica: Sistema Financiero de Dinero Electrónico (SFDE)

Este documento proporciona una visión técnica detallada de la arquitectura, flujos de datos y componentes del sistema distribuido de microservicios para la gestión de dinero electrónico.

---

## 1. Arquitectura del Sistema

El sistema sigue un patrón de **Microservicios Event-Driven** (orientado a eventos) utilizando **Google Cloud Pub/Sub** como bus de mensajes central.

### Diagrama de Arquitectura (Mermaid)

```mermaid
graph TD
    subgraph "Capa de Usuario"
        UI[Web Interface - Puerto 8085]
        CS[Client Simulator]
    end

    subgraph "Capa de Aplicación (Microservicios REST)"
        Auth[AuthService - Puerto 8081]
        Account[AccountService - Puerto 8080]
        Report[ReportService - Puerto 8084]
    end

    subgraph "Infraestructura de Mensajería (Pub/Sub)"
        Topic_TX[Topic: tx-events]
        Topic_Conf[Topic: tx-confirmed]
    end

    subgraph "Capa de Procesamiento Asíncrono"
        TX_Service[TransactionService]
        Audit_Service[AuditService]
    end

    subgraph "Capa de Datos"
        DB[(Google Cloud SQL - PostgreSQL)]
    end

    subgraph "Monitoreo"
        CPU[CPU Monitor - Lanterna TUI]
    end

    %% Relaciones
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
    
    CPU -.->|Monitorea| Auth
    CPU -.->|Monitorea| Account
    CPU -.->|Monitorea| TX_Service
```

---

## 2. Flujo de una Transacción

Este diagrama muestra el ciclo de vida de una operación financiera (ej. transferencia) desde que el usuario la solicita hasta que se confirma y audita.

```mermaid
sequenceDiagram
    participant U as Usuario / Simulator
    participant AC as AccountService
    participant PS as Google Cloud Pub/Sub
    participant TX as TransactionService
    participant DB as PostgreSQL (Cloud SQL)
    participant AU as AuditService

    U->>AC: POST /api/accounts/transfer
    Note over AC: Valida saldo y genera evento
    AC->>PS: Publish event to 'tx-events'
    AC-->>U: 202 Accepted (Transaction ID)

    PS->>TX: Push/Pull event
    Note over TX: Procesa lógica de negocio (Suma/Resta)
    TX->>DB: UPDATE balances (Atomicity)
    TX->>PS: Publish confirmation to 'tx-confirmed'

    PS->>AU: Push/Pull confirmation
    AU->>DB: INSERT into audit_logs
    Note over AU: Registro histórico inmutable
```

---

## 3. Descripción de Componentes

### 3.1 AuthService (Puerto 8081)
Responsable de la seguridad perimetral.
- **Tecnología:** Spring Security + JJWT.
- **Funciones:** Registro de usuarios, login, generación y validación de tokens JWT.
- **Seguridad:** Uso de BCrypt para el hashing de contraseñas.

### 3.2 AccountService (Puerto 8080)
El "Entry Point" para todas las operaciones de cuenta.
- **Operaciones:** Consultar saldo, depósitos, retiros y transferencias.
- **Rol:** Actúa como productor de eventos. No modifica los balances directamente en operaciones críticas, sino que delega al `TransactionService` vía Pub/Sub para garantizar escalabilidad.

### 3.3 TransactionService (Asíncrono)
El motor de procesamiento central.
- **Mecánica:** Escucha el tópico `tx-events`.
- **Garantías:** Asegura la consistencia eventual y el procesamiento exacto de las transacciones.
- **Escalabilidad:** Puede tener múltiples réplicas para procesar ráfagas de transacciones.

### 3.4 AuditService (Asíncrono)
Garantiza la trazabilidad total del sistema.
- **Mecánica:** Escucha el tópico `tx-confirmed`.
- **Función:** Registra cada transacción finalizada en una tabla de auditoría inmutable para fines de reporte y seguridad.

### 3.5 ReportService (Puerto 8084)
Capa de lectura optimizada para el panel de administración.
- **Funciones:** Generación de métricas agregadas (saldo total del sistema, número de transacciones por minuto, balances por usuario).

---

## 4. Modelo de Datos (ER)

El sistema utiliza una base de datos relacional PostgreSQL con el siguiente esquema simplificado:

```mermaid
erDiagram
    USUARIO ||--o{ CUENTA : "posee"
    CUENTA ||--o{ TRANSACCION : "genera"
    TRANSACCION ||--|| AUDITORIA : "registra"

    USUARIO {
        string curp PK
        string nombre
        string password
        string rol
    }

    CUENTA {
        int id PK
        string curp FK
        double saldo_banco
        double saldo_billetera
    }

    TRANSACCION {
        uuid id PK
        int cuenta_origen
        int cuenta_destino
        double monto
        timestamp fecha
    }
```

---

## 5. Monitoreo y Herramientas

### CPU Monitor (Lanterna TUI)
Aplicación de consola que utiliza la librería `OSHI` para obtener métricas de hardware en tiempo real. Permite visualizar la carga de trabajo de los diferentes microservicios distribuidos.

### Client Simulator
Herramienta de pruebas de carga que permite parametrizar:
- `n`: Número de clientes.
- `h`: Hilos concurrentes.
- `p`: Presupuesto inicial por cliente.
- `t`: Transacciones por minuto deseadas.

---

## 6. Comandos de Gestión

| Acción | Script |
|--------|--------|
| Levantar todo | `./run-all-services.sh` |
| Detener todo | `./stop-all-services.sh` |
| Verificar estado | `./check-services.sh` |
| Pruebas manuales | `GUIA_PRUEBAS_MANUAL.md` |

---
*Documentación generada automáticamente por Gemini CLI para el Proyecto Final de Sistemas Distribuidos.*
