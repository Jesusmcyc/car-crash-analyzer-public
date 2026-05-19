# ADR 0022 — Email transaccional: Resend + tabla subscribers separada

**Estado:** Aceptado
**Fecha:** 2026-05-18
**Autor:** Jesús Moreno
**Milestone:** M7

## Contexto

M7 spec D7 requiere enviar email automático al donante cuando su imagen contribuya a una nueva versión del modelo (decisión 7a del operador, ver memoria `project-m7-decisions`). Requisitos derivados:

- **Volumen esperado:** 3-12 emails/año (cadencia esperada de v2/v3/v4 del modelo) × N donantes con email registrado. Realista: <500 emails/año.
- **Transaccional, no marketing:** un email por cada release del modelo, no newsletter.
- **Deliverability razonable:** los donantes deben recibir el email en inbox, no spam.
- **Unsubscribe trivial:** requisito legal LFPDPPP (derecho de cancelación) + CAN-SPAM (USA, donde está Resend).
- **Costo bajo:** free tier suficiente; no comprometer el demo con costos recurrentes.

Cuatro providers evaluados: **Resend**, Postmark, SendGrid, Mailgun. También considerado Listmonk (self-hosted) y direct SMTP via Gmail.

## Decisión

**Resend** como provider transaccional + **tabla `subscribers` separada** de `contributions` en SQLite.

### Provider: Resend

- **Free tier:** 3,000 emails/mes, 100/día — sobra para el volumen esperado.
- **API simple:** `resend.Emails.send({from, to, subject, text, html})`. Python SDK official (`pip install resend`).
- **Sender inicial:** `onboarding@resend.dev` (Resend default, no requiere DNS). Plan B documentado: `@cca.imb-central.tech` con DNS verification (SPF + DKIM TXT records, 24-48h propagation).
- **API key en env var** `RESEND_API_KEY`. Si no se setea, fallback automático a `LocalLogEmailSender` que loggea pero no envía (útil para dev local + tests).

### Schema: tabla `subscribers` separada

```sql
CREATE TABLE subscribers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    opaque_id TEXT NOT NULL UNIQUE,    -- UUID4 para URLs de unsubscribe
    subscribed_at TEXT NOT NULL,
    unsubscribed_at TEXT,
    unsubscribe_reason TEXT
);

ALTER TABLE contributions ADD COLUMN subscriber_id INTEGER REFERENCES subscribers(id);
```

**Email en plain text en DB privada.** No hashed, no encrypted at rest. La defensa real es el control de acceso al volumen Coolify; encriptación adicional desplaza el problema sin reducir el riesgo proporcionalmente para portfolio scale.

**Disclosure en aviso de privacidad** (texto literal a publicar en `/privacidad`):

> Tu email se almacena en plain text en un volumen privado del operador. Lo usamos solo para notificarte cuando una nueva versión del modelo entrene con tu imagen. No te suscribimos a newsletter ni compartimos con terceros. Puedes retirar el consentimiento en cualquier momento.

### Pipeline de envío

Single-shot manual (`scripts/notify_contributors.py`) — el operador ejecuta tras deploy de v2/v3/v4 con el `model_version` como arg:

```bash
python scripts/notify_contributors.py --model-version=rtdetr-cardd-v2
python scripts/notify_contributors.py --model-version=rtdetr-cardd-v2 --dry-run  # preview sin enviar
```

Query interno: `SELECT email, opaque_id FROM subscribers JOIN contributions ON ... JOIN training_contributions ON ... WHERE training_runs.model_version = ? AND subscribers.unsubscribed_at IS NULL` — dedup automático por email.

## Consecuencias

### Positivas

- **Free tier sobra.** 3,000/mes vs <500/año esperado = 60× margen. Free tier permanente mientras no escale; al $20/mes si fuera necesario.
- **DX moderna.** Resend tiene mejor SDK Python que SendGrid v1 (deprecated) o v2 (clunky). Postmark es competitivo pero requiere setup más extenso.
- **Tabla separada permite minimización.** Si una contribución no tiene email, no entra a `subscribers`. Múltiples contribuciones del mismo email se enlazan a un solo `subscriber.id`, no duplican.
- **Opaque ID = UUIDs robustos para unsubscribe.** No exponen email ni ID interno en URLs ni logs.
- **Fallback automático a `LocalLogEmailSender`** sin `RESEND_API_KEY`. Tests no requieren mocks elaborados ni internet.
- **Dry-run mode** en el script: el operador puede ver "a quién mandaría" antes de mandar.

### Negativas

- **Dependencia de un tercero (Resend Inc., USA).** Si Resend desactiva la cuenta o cambia precios, hay que migrar. Mitigación: el código abstrae `EmailSender` Protocol; swap a Postmark/SendGrid es ~30 líneas de código.
- **Transferencia internacional de datos.** El email se transfiere a USA (servidores Resend). Declarado en aviso de privacidad LFPDPPP — no es non-issue, pero la base legal es consentimiento expreso del titular en el flujo de upload.
- **Sender por default genérico** (`onboarding@resend.dev`) puede triggear filtros antispam en algunos clientes. Upgrade a sender custom `@cca.imb-central.tech` queda como tarea operativa post-M7 si la deliverability se vuelve issue.
- **Plain text de emails es lectura humana** — accesible para el operador con SSH al droplet. Por diseño (necesario para el send), no por accidente.

### Neutrales

- **Resend cuenta del operador** requiere setup manual (one-time): signup + API key + paste en Coolify UI. Documentado en plan T15.
- **Tests no usan red real.** `LocalLogEmailSender` cubre paths de éxito + fallo conceptual. Smoke en prod con la API key real es el test de integración real.
- **El email se envía solo cuando el operador trigger el script.** No cron, no auto-send post-training. Mantiene control humano del momento.

## Alternativas consideradas

### Postmark

- Más caro ($15/mes free → $15/mes 10k emails) pero excelente deliverability + setup más maduro.
- Rechazado para portfolio scale — Resend free tier basta.
- **Reconsiderar como plan B** si Resend falla.

### SendGrid

- Free tier 100/día (similar a Resend free tier).
- SDK Python tiene historia de versionado quebradizo (v1 deprecated, v2 clunky).
- Rechazado por DX inferior a Resend para el mismo precio.

### Mailgun

- Cambió a 30-day paid trial en 2023, no hay free tier persistente.
- Rechazado por costo.

### Listmonk self-hosted

- Mailing list manager open-source.
- Rechazado: añade servicio Coolify, requiere SMTP propio, overkill para 3-12 disparos al año.
- **Reconsiderar** si el volumen escala a múltiples newsletters segmentados (no es el caso).

### Direct SMTP via Gmail (App Password)

- Gratis, simple.
- Rechazado: Gmail rate-limita SMTP outbound, no es para uso transaccional en escala, deliverability poco predecible (DKIM/SPF dependiente del envío personal). No-go profesional.

### Email en plain text vs encrypted at rest

- **Plain text** (decisión): mantenimiento mínimo, el envío requiere el texto plano, encriptación adicional desplaza el problema a key management.
- **`cryptography.Fernet` con key en env var**: añade rotación de keys + recovery si el operador pierde la key. Costo operativo > beneficio para portfolio scale.
- **Hash de email** (`bcrypt`/`argon2`): rompe el envío proactivo — no se puede recuperar el email desde el hash. Solo serviría para deduplicación, que ya cubrimos con `UNIQUE(email)`.

## Referencias

- design doc — D3 (email handling), D7 (notification mechanism).
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — declaración de transferencia internacional a USA.
- [ADR 0021](0021-storage-sqlite-volume.md) — tabla `subscribers` en schema SQLite.
- **Resend Python SDK:** <https://resend.com/docs/sdks/python>
- **Postmark (plan B):** <https://postmarkapp.com>
- **CAN-SPAM (USA):** <https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business>
