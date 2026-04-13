# Guía de Configuración: Google Cloud Pub/Sub

Esta guía detalla el proceso paso a paso para configurar la infraestructura de mensajería necesaria para el **Sistema Financiero de Dinero Electrónico (SFDE)**.

---

## 1. Requisitos Previos

1.  **Cuenta de Google Cloud:** Tener acceso a la consola de GCP.
2.  **Google Cloud SDK (gcloud):** Instalado y configurado en tu máquina local.
3.  **Habilitar la API:** Asegúrate de que la API de Pub/Sub esté habilitada para tu proyecto.

---

## 2. Preparación del Entorno

Abre una terminal y ejecuta los siguientes comandos para autenticarte y seleccionar tu proyecto:

```bash
# Iniciar sesión en Google Cloud
gcloud auth login

# Configurar el ID del proyecto
gcloud config set project sistemafinancierodistribuido
```

---

## 3. Configuración de Tópicos (Topics)

Los tópicos son los canales donde los microservicios publican mensajes (eventos).

| Tópico | Responsable | Uso |
| :--- | :--- | :--- |
| `tx-events` | `AccountService` | Recibe solicitudes de depósitos, retiros y transferencias. |
| `tx-confirmed` | `TransactionService` | Recibe la confirmación de que el saldo fue actualizado exitosamente. |

### Comandos de Creación:
```bash
gcloud pubsub topics create tx-events
gcloud pubsub topics create tx-confirmed
```

---

## 4. Configuración de Suscripciones (Subscriptions)

Las suscripciones permiten que los microservicios "consumidores" reciban los mensajes de los tópicos.

| Suscripción | Tópico Origen | Consumidor |
| :--- | :--- | :--- |
| `tx-events-sub` | `tx-events` | `TransactionService` |
| `tx-confirmed-audit-sub` | `tx-confirmed` | `AuditService` |

### Comandos de Creación:
```bash
# Suscripción para el procesador de transacciones
gcloud pubsub subscriptions create tx-events-sub --topic=tx-events

# Suscripción para el auditor del sistema
gcloud pubsub subscriptions create tx-confirmed-audit-sub --topic=tx-confirmed
```

---

## 5. Configuración de Credenciales Locales

Para que tus microservicios Java puedan conectarse a GCP desde tu máquina local, necesitas generar una llave de cuenta de servicio (Service Account Key).

1.  Crea una cuenta de servicio:
    ```bash
    gcloud iam service-accounts create sfd-local-dev
    ```
2.  Asígnale permisos de Editor de Pub/Sub:
    ```bash
    gcloud projects add-iam-policy-binding sistemafinancierodistribuido \
        --member="serviceAccount:sfd-local-dev@sistemafinancierodistribuido.iam.gserviceaccount.com" \
        --role="roles/pubsub.editor"
    ```
3.  Genera el archivo JSON de la llave:
    ```bash
    gcloud iam service-accounts keys create gcp-credentials.json \
        --iam-account=sfd-local-dev@sistemafinancierodistribuido.iam.gserviceaccount.com
    ```
4.  **IMPORTANTE:** Configura la variable de entorno en tu terminal antes de correr los servicios:
    ```bash
    export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/gcp-credentials.json"
    ```

---

## 6. Verificación (Pruebas de Conexión)

Puedes probar que la infraestructura funciona publicando un mensaje de prueba manualmente:

```bash
# Publicar mensaje manual
gcloud pubsub topics publish tx-events --message='{"id": "test-1", "monto": 100}'

# Intentar jalar el mensaje (Pull)
gcloud pubsub subscriptions pull tx-events-sub --auto-ack
```

---
*Esta guía es esencial para que la comunicación asíncrona entre microservicios funcione correctamente.*
