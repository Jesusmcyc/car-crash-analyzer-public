# ADR 0024 — Consent flow: default-on con opt-out visible

**Estado:** Aceptado
**Fecha:** 2026-05-18
**Autor:** Jesús Moreno
**Milestone:** M7.1 (UI polish post-M7)

## Contexto

[ADR 0019](0019-flip-no-persistence.md) abrió la persistencia condicional al `consent` expreso. [ADR 0020](0020-lfpdppp-compliance-architecture.md) implementó el flow M7 inicial: checkbox **opt-in default-off** ("Acepto que mi imagen se guarde…"). Spec M7 D1 cementó esa decisión como "más segura legalmente" en su momento.

Post-deploy + smoke prod, el operador identificó tres issues con el modelo opt-in default-off:

1. **Fricción para el visitante objetivo (LinkedIn/portfolio).** La página entrega valor con el análisis; el checkbox separado del flujo principal añade un paso cognitivo extra.
2. **Tasa de donación esperada baja (~5-15% según UX benchmarks).** El dataset crece despacio → poco data para fine-tune en ciclos siguientes.
3. **El copy del checkbox** ("Acepto que mi imagen se guarde para mejorar el modelo") leía como compromiso explícito; muchos visitantes lo skipean por cautela aunque el demo es low-risk.

El operador propuso cambiar a un modelo donde "al subir la imagen el usuario está de acuerdo en donarla" (consent implícito por uso). Tras revisión:

- **Consent implícito puro** (sin banner) no cumple LFPDPPP art. 8 para datos que pueden ser sensibles (imágenes con placas, rostros, EXIF GPS).
- **Consent expreso por acción afirmativa** (banner + click del botón de upload) sí cumple, siempre que el banner sea inequívoco y el visitante tenga opción de oposición visible (LFPDPPP art. 8 y 11).

## Decisión

Migrar al modelo **default-on con opt-out visible**:

1. **Banner prominente** sobre el dropzone declarando que al subir la imagen se analiza **y se guarda** para mejorar el modelo. Link al aviso de privacidad inline.
2. **Email opcional** disponible (enabled) por default; útil para notificación post-training.
3. **Checkbox opt-out sutil** al final del bloque: *"No guardar mi imagen — solo analizar"*. Si el visitante lo marca antes de subir, el backend recibe `consent=false` y descarta la imagen tras el análisis (cero persistencia).
4. **El acto de subir** = consent expreso por acción afirmativa (asumiendo opt-out unchecked).

El backend no cambia: el contrato `POST /api/analyze` ya acepta `consent: bool = False` como query field. Solo el frontend cambia el default a `consent=true` y el componente UI a banner + opt-out.

## Consecuencias

### Positivas

- **Tasa de donación esperada ~80-95%.** El visitante que solo quiere probar el demo (no donar) puede hacerlo con un click adicional; el resto contribuye por flujo natural.
- **Dataset crece más rápido** → ciclos de fine-tune más frecuentes → mejora del modelo más rápida (visible en LinkedIn posts sobre "v2 entrenó con N imágenes donadas").
- **UX más limpia** — banner único en lugar de checkbox + label legalese.
- **Email field enabled por default** sin perjuicio de UX: el visitante simplemente no lo llena si no quiere ser notificado.

### Negativas

- **Trade-off LFPDPPP comparado con M7 original.** Pasamos de "consentimiento expreso por checkbox activo" a "consentimiento expreso por acción afirmativa con banner". El segundo es jurisprudencia INAI aceptada (clicks como acción afirmativa para datos no-sensibles + banner declarando uso) **siempre que el banner sea inequívoco y el opt-out esté visible**. Ambos requisitos se cumplen. **Si un revisor INAI cuestionara el cambio, el aviso de privacidad sec 7 actualizado explica el modelo en lenguaje claro.**
- **Visitantes que no leen el banner** podrían contribuir sin notar — la "carga de leer" se traslada al visitante. Mitigación: el banner es grande, sin scroll, con verbo claro ("se guarda"). Withdraw endpoint sigue disponible para retiro inmediato.
- **El aviso de privacidad sec 7 cambia** — versionado vía `CONSENT_VERSION` no se incrementa porque el aviso describe el mismo procesamiento; solo cambia el medio de expresar el consent. Documentado aquí + en el commit.

### Neutrales

- **Backward compat del API:** `POST /api/analyze` sigue aceptando `consent: bool = False` como opcional. El frontend antiguo (M7 con checkbox opt-in) seguiría funcionando contra el backend nuevo sin cambios.
- **Withdraw endpoint** (`POST /api/privacy/withdraw/{opaque_id}`) sigue siendo el mecanismo principal de cancelación ARCO.
- **EXIF strip + resize + recompress** (ADR 0020) sigue aplicando — privacy by design preservada.

## Alternativas consideradas

1. **Status quo (M7 original) — opt-in default-off con copy más warm.** Rechazado: el operador validó que la fricción cognitiva del checkbox separado es el bloqueador real, no solo el copy.
2. **Consent implícito puro (banner sin opt-out).** Rechazado: LFPDPPP art. 11 requiere un medio simple de oposición para todos los modelos de consent, y el spec D9 ya documentó que pre-checked / sin opt-out es violación directa.
3. **Doble botón de upload ("Solo analizar" vs "Analizar y donar").** Rechazado por introducir decision paralysis y aumentar la complejidad del UI por una decisión que pocas personas toman conscientemente al primer uso.

## Riesgos / mitigaciones

- **Riesgo:** Visitante reclama ante INAI argumentando que no se dio cuenta de que su imagen se guardaba. **Mitigación:** banner en español claro + link al aviso siempre visible + opt-out visible en el mismo viewport + withdraw endpoint operativo + procedimiento ARCO vía email documentado. Posición defensiva sólida.
- **Riesgo:** Tasa de donación demasiado alta saturó disco. **Mitigación:** monitor mensual del volumen Coolify (ya documentado en ADR 0021); rate limit 10 contribuciones/IP/día (ya existente).
- **Riesgo:** El cambio confunde a usuarios que ya conocían el flow M7 original. **Mitigación:** aviso de privacidad sec 7 actualizado declara el modelo activo; visitantes recurrentes (improbable en portfolio) ven la diferencia inmediata.

## Referencias

- [ADR 0019](0019-flip-no-persistence.md) — flip de "no persistencia" original.
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — arquitectura LFPDPPP base.
- design doc sec D1 — modelo opt-in original superseded por este ADR.
- `frontend/src/components/upload/ConsentBlock.tsx` — implementación.
- `frontend/src/pages/PrivacyPage.tsx` sec 7 — aviso de privacidad actualizado.
- **LFPDPPP texto oficial:** <https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf>
