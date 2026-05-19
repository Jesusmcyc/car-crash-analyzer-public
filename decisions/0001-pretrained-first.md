# ADR 0001 — Pretrained-first: el pipeline se construye sobre modelos preentrenados antes de cualquier fine-tune

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

El proyecto Car Crash Analyzer (architecture doc) tiene dos componentes ML core: un detector de daños y un segmentador. El roadmap (sec 12) contempla fine-tunear RT-DETRv2 sobre el dataset CarDD en una fase posterior (Fase 4 / M5).

La pregunta arquitectónica al iniciar el proyecto es: **¿se construye el pipeline directamente con pesos fine-tuneados, o se ensambla primero con modelos preentrenados y se sustituyen los pesos después?**

## Decisión

**El pipeline (M0–M4) corre exclusivamente con modelos preentrenados.** El fine-tune con CarDD ocurre en M5 como hito paralelo, y los pesos resultantes se sueltan al backend mediante una variable de entorno (`DETECTOR_WEIGHTS`) sin tocar código de aplicación.

## Consecuencias

### Positivas

- **Trabajo paralelo:** mientras se construye y despliega el pipeline (M0–M4), el fine-tune puede prepararse en notebook aparte sin bloquearse mutuamente.
- **Riesgo aislado:** los problemas de entrenamiento (convergencia, overfitting, pérdida de mAP en clases raras) no contaminan el debugging de la app.
- **Fallback obvio:** si el fine-tune no mejora la baseline preentrenada, el sistema funciona igual con los pesos originales. El "downside" del fine-tune es cero.
- **Reproducibilidad para el revisor:** alguien puede levantar el demo sin tener acceso al checkpoint custom, solo con pesos públicos.

### Negativas

- **Detecciones M0–M4 no son específicas de CarDD.** RT-DETRv2 preentrenado en COCO no conoce categorías como `crack` o `broken_lamp` directamente; en M2 se mapean clases COCO genéricas a las categorías del dominio mediante un dict (es una aproximación; M5 lo arregla).
- **Métricas de M2/M3 no son finales.** No se debe presentar el demo M0–M4 con afirmaciones de precisión hasta que M5 cierre.

## Alternativas consideradas

1. **Esperar a tener pesos fine-tuneados antes de empezar el backend.** Rechazada — bloquea M0–M4 por días/semanas según ritmo de entrenamiento, y deja el riesgo de deploy sin descubrir.
2. **Entrenar desde cero.** Fuera de alcance.

## Referencias

- architecture doc sec 12 (Roadmap).
- design doc sec 3 → M0–M5.
