# ADR 0020 — Arquitectura de cumplimiento LFPDPPP

**Estado:** Aceptado
**Fecha:** 2026-05-18
**Autor:** Jesús Moreno
**Milestone:** M7

## Contexto

[ADR 0019](0019-flip-no-persistence.md) levanta la restricción "no persistencia" para el subset de imágenes consented en M7. Eso abre superficie legal bajo el régimen mexicano de protección de datos personales:

- **Ley Federal de Protección de Datos Personales en Posesión de los Particulares (LFPDPPP)**, vigente desde 2010, jurisdicción aplicable porque (a) el responsable es persona física mexicana (Jesús Moreno), (b) los titulares pueden ser personas físicas en territorio mexicano, (c) los datos personales recogidos incluyen email (al darse) e imágenes que pueden contener placas vehiculares, rostros, o ubicación EXIF (categorizables como datos sensibles).
- **Autoridad supervisora:** INAI (Instituto Nacional de Transparencia, Acceso a la Información y Protección de Datos Personales).
- **Sanciones potenciales:** multas administrativas + reclamos individuales por daños.

GDPR no aplica estrictamente (no hay establecimiento en UE) pero las prácticas son compatibles. LFPDPPP es más estricta en algunos puntos (consentimiento expreso para datos sensibles).

## Decisión

Tres pilares de cumplimiento, todos implementados como parte de M7:

### Pilar 1 — Aviso de privacidad accesible y específico

- **Página `/privacidad`** (ruta SPA, no PDF) con las 8 secciones obligatorias LFPDPPP art. 16:
  1. Identidad y domicilio del responsable.
  2. Datos personales recabados.
  3. Datos personales sensibles (declaración explícita: imágenes pueden contener placas/rostros/EXIF GPS).
  4. Finalidades (una sola: re-entrenamiento del modelo).
  5. Transferencias (email a Resend Inc. en USA, declarada).
  6. Derechos ARCO + procedimiento para ejercerlos.
  7. Medios para limitar uso (no marcar consent + withdraw endpoint).
  8. Cambios al aviso (versionado + re-consent si cambio sustantivo).
- **Texto en español**, escrito por el operador (no plantilla autogenerada).
- **Versionado:** cada contribución almacena `consent_version` (string `"privacy-v1"`); cambios sustantivos suben la versión y disparan re-consent.
- **Linkeado desde tres lugares:** checkbox de consent, footer global, cada email transaccional (D7 spec M7).

### Pilar 2 — Privacy by design en el procesamiento

- **EXIF strip antes de persistir.** `ImageOps.exif_transpose` para honrar orientación visual, luego nueva imagen sin EXIF (D5 spec M7). El sha256 se calcula sobre los bytes post-strip — el archivo persistido nunca contiene GPS ni serial de dispositivo.
- **Resize a 1024 px** (lado mayor, Lanczos) y **recompress JPEG q=85** unificado — minimización del dato persistido manteniendo utilidad para training.
- **No reconocimiento facial, no OCR de placas.** El pipeline CV detecta tipos de daño (`dent`, `scratch`, etc.) — no identifica personas ni vehículos individuales.
- **IP hash con sal diaria UTC** — `sha256(ip + "|" + YYYY-MM-DD)`. Permite forensics intra-day para rate limit; impide tracking longitudinal.
- **Email en tabla separada `subscribers`** con `opaque_id` UUID4 para URLs de unsubscribe. No exposición de email en logs ni headers.

### Pilar 3 — Derechos ARCO operativos

- **`A`cceso + `R`ectificación:** vía email a `jesusmcyc@gmail.com` con el `request_id` de la contribución (capturado en el toast post-upload). Procesado manualmente en ≤20 días hábiles (LFPDPPP).
- **`C`ancelación:** endpoint público `POST /api/privacy/withdraw/{opaque_id}` idempotente. Para contribuidores con email registrado, el opaque_id viene en cada email transaccional. Para los anónimos (sin email), la cancelación es vía email manual al operador.
- **`O`posición:** equivalente a no marcar el checkbox de consent. El visitante puede usar el demo sin contribuir.
- **Hard delete del archivo físico** en el endpoint de withdraw (`os.unlink`). LFPDPPP exige cancelación efectiva — soft delete con flag no califica si el dato sigue accesible.
- **Idempotencia:** segunda llamada al mismo opaque_id devuelve `{status: "already_withdrawn"}`. No errores en doble-click.

## Consecuencias

### Positivas

- **Defensa frente a un reclamo INAI realista.** Aviso completo + ARCO operativo + minimización demostrable = posición sólida si llega un reclamo.
- **Confianza visible para visitantes de LinkedIn.** Aviso en español accesible desde el footer comunica seriedad operativa.
- **EXIF strip protege también al operador.** Aunque el visitante sube voluntariamente, el operador no quiere almacenar GPS coords ni serials de dispositivos.
- **Tabla `subscribers` separada simplifica auditoría.** Ver "todas las contribuciones de este email" es un SELECT JOIN; minimizar es DROP ROW.

### Negativas

- **Mantenimiento del aviso en cada cambio sustantivo.** Si la finalidad se amplía (ej. "compartir el dataset con un tercero"), versión sube + re-consent. Costo operativo no nulo.
- **Withdraw endpoint público es atacable.** Mitigación: el opaque_id es UUID4 (impracticable de adivinar). Riesgo residual aceptable.
- **EXIF strip elimina información que podría ser útil para training futuro** (ej. metadata de cámara, lente, fecha). Trade-off explícito a favor de privacidad.

### Neutrales

- **El aviso de privacidad es responsabilidad humana del operador.** Texto legal sensible requiere firma humana; el draft inicial pasa por revisión antes de remover el marker `<!-- DRAFT -->`.
- **No hay representante en territorio mexicano** — no requerido para persona física residente en México.
- **No hay registro formal ante INAI** — LFPDPPP no exige registro previo (a diferencia de algunos regímenes europeos).

## Alternativas consideradas

1. **Plantilla legal autogenerada (Iubenda, TermsFeed).** Rechazado: $5-15/mes recurrente, plantillas a menudo más amplias que el uso real (declaran data sharing con terceros que nunca ocurre), generan distrust en lectores cuidadosos.
2. **Aviso de privacidad como PDF embebido.** Rechazado: no versionable con diff por línea, requiere abrir en visor externo, menos accesible móvil.
3. **Soft delete con purge job a los 30 días** en lugar de hard delete inmediato. Rechazado — LFPDPPP "cancelación" no admite delay arbitrario sin justificación legítima.
4. **Skip de derecho de oposición** (asumir que "no marcar checkbox" es suficiente). Rechazado por completitud — declarar el medio explícitamente protege legalmente.
5. **Auto-blur de rostros y placas con CV pre-persist.** Postergado a futuras iteraciones — el consent expreso + EXIF strip + privacy notice cubren el caso. Auto-blur añade complejidad sin reducir el riesgo legal en proporción.

## Referencias

- design doc — D4 (aviso), D5 (privacy by design), D9 (withdrawal).
- [ADR 0019](0019-flip-no-persistence.md) — flip de "no persistencia" que abre esta superficie.
- [ADR 0022](0022-email-resend-transactional.md) — transferencia del email a Resend USA (declarada en aviso).
- **LFPDPPP texto oficial:** <https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf>
- **INAI:** <https://home.inai.org.mx/>
- **Reglamento LFPDPPP:** <https://www.diputados.gob.mx/LeyesBiblio/regley/Reg_LFPDPPP.pdf>
