# Test Cases — Automatización de Preguntas NQ

> Plan de pruebas para la orquestación asíncrona de reposición de stock académico e integración con NQ.
> Cobertura: HU-1 (8 ACs), HU-2 (9 ACs), Casos Borde, NFRs, Constitution, Plan.
> Incluye mitigaciones para 14 roturas detectadas en grilling (concurrencia en encolado, PARTIAL, idempotencia extendida, sanitización, ciclos de reposición, FIFO con reintentos).

---

## Resumen Ejecutivo

### ¿Qué se prueba?

Se valida la funcionalidad completa de automatización de reposición de preguntas académicas mediante integración con NQ, cubriendo dos historias de usuario (HU-1 y HU-2), sus 16 criterios de aceptación, 5 casos borde, los 3 requisitos no funcionales, y los artículos aplicables de la Constitución del proyecto (seguridad, idempotencia, trazabilidad). Los 31 casos de prueba (TC-01 a TC-31) diseñados verifican:

- **Orquestación de envío (HU-1):** Detección automática de stock insuficiente, división de solicitudes en bloques de 5 preguntas, procesamiento FIFO por fecha de generación, validación de cursos habilitados (Álgebra, Aritmética, Trigonometría, Química), construcción de payload con atributos requeridos (curso, tema, subtema, nivel).
- **Validación y almacenamiento (HU-2):** Recepción automática de respuestas NQ, validación de duplicidad usando lógica existente de Lumeria, descarte automático de duplicados, solicitud de reposición por preguntas faltantes (máximo 3 ciclos), almacenamiento en tabla temporal de revisión docente, conservación del flujo actual de revisión.
- **Casos borde:** Consolidación acumulativa de registros repetidos para una misma combinación curso-tema-subtema-nivel, subtemas sin nivel configurado, ciclos de reposición con estado PARTIAL, concurrencia de workers, cursos no habilitados.
- **Manejo de errores:** HTTP 500 de NQ con backoff y reintentos, respuestas HTTP 200 con body inválido o incompleto, fallo de inserción en BD con rollback total y sin registros huérfanos, NQ devolviendo más preguntas de las solicitadas.
- **Concurrencia y seguridad:** Doble encolado por requests HTTP simultáneos (condición de carrera), secreto `ODISEO_KEY` nunca expuesto en logs ni stack traces, sanitización de contenido HTML/scripts proveniente de NQ.
- **Cualidades transversales:** Máquina de estados completa (`PENDING → PROCESSING → PARTIAL/COMPLETED/FAILED`), idempotencia del `GenerationJob` en todos los estados terminales y activos, campos de trazabilidad poblados correctamente al finalizar, división en bloques durante reposición, detección de duplicados intra-lote.
- **Recuperación y resiliencia:** Stale job recovery mediante comando programado `faltantes:recover-stale` (PROCESSING zombies > 30 min), NQ devuelve menos preguntas de las solicitadas con éxito parcial + reposición, contador `reposition_cycles` incrementado en errores NQ, máximo 3 reintentos por ciclo, caché de respuesta NQ en Redis (TTL 24h) ante fallo de BD para evitar doble consumo de créditos.

### Enfoque de las pruebas

Las pruebas se estructuran con un enfoque **basado en riesgos y trazabilidad**:

- Cada caso de prueba sigue el formato **Given/When/Then/Verification** con resultados medibles (estados en BD, conteo de registros, invocaciones a `NQClient::post`, contenido de logs) sin ambigüedad.
- La cobertura se organiza por **capas de testing**: unitaria (lógica de detección de stock, validación estructural, reglas de negocio — 100% de reglas), integración (comunicación API NQ, colas Redis, persistencia, FIFO) y seguridad (protección de secretos, ofuscación de logs, sanitización de entrada externa).
- Se utiliza **mocking de la API NQ** en entornos aislados para validar reintentos, manejo de errores 5xx y ciclos de reposición sin consumir créditos reales.
- Las **14 roturas detectadas durante sesiones de grilling** entre QA, Tech Lead y Product Owner fueron mitigadas con casos de prueba específicos (ver matriz de trazabilidad), asegurando que defectos potenciales de diseño queden cubiertos antes de la implementación.
- La **matriz de trazabilidad** (sección 9) mapea cada requisito (16 ACs), caso borde, NFR, artículo de la Constitución y rotura detectada a sus TC correspondientes, garantizando que ningún criterio de aceptación quede sin verificación.
- El plan sigue los lineamientos de la **Constitución del proyecto (Art. 6 — Testing Standards)**, que exige pruebas automatizadas, cobertura de escenarios críticos de negocio, integraciones externas con casos de éxito y error, y cobertura de casos borde y concurrencia.

### Qué quedó sin cubrir

Aunque los 27 casos de prueba cubren exhaustivamente los requisitos funcionales, de negocio y las roturas detectadas, se identifican las siguientes áreas sin cobertura explícita en este plan:

- **Pruebas de carga/volumen:** No existen TC para escenarios de alto volumen (miles de faltantes pendientes simultáneos) ni pruebas de estrés sobre la cola Redis. El plan de riesgos menciona "alto volumen de faltantes pendientes" como riesgo identificado (plan.md:177), y la constitución exige al menos un TC de carga/volumen (constitution.md:82). Se creará TC-32 en Fase 2.
- **Timeout de NQ como error independiente:** El plan técnico menciona "timeout" junto con HTTP 5xx como errores de NQ, pero TC-11 solo cubre explícitamente HTTP 500. No hay un TC específico para `connection timeout` o `read timeout` del cliente HTTP hacia NQ. La constitución exige cobertura de timeout (constitution.md:89). Se creará TC-29 en Fase 2.
- **Rate limiting de NQ (HTTP 429):** No se contempla el escenario donde NQ responde con HTTP 429 (Too Many Requests) ni cómo el sistema debe reaccionar (backoff, respetar header `Retry-After`). Se creará TC-31 en Fase 2.
- **HTTP 4xx → FAILED inmediato:** La constitución exige cobertura de 4xx como error independiente con FAILED sin reintento (constitution.md:90). TC-11 solo cubre 5xx. Se creará TC-28 en Fase 2.
- **Comportamiento del sistema ante 401/403:** TC-18 verifica que `ODISEO_KEY` no se expone en logs durante un 401, pero no existe un TC que defina el comportamiento completo del sistema ante credenciales inválidas o expiradas (transición de estado, reintentos, registro de auditoría). Esto se incluirá dentro de TC-28 (Fase 2).
- **Stale job en estado PARTIAL:** TC-23 cubre la recuperación de trabajos zombies en estado PROCESSING (> 30 min), pero no contempla explícitamente un worker que muere mientras el faltante está en PARTIAL. El scheduled command `faltantes:recover-stale` solo actúa sobre `PROCESSING`.
- **Indisponibilidad de Redis:** TC-27 asume Redis disponible para cachear respuestas de NQ ante fallo de BD. La constitución exige cobertura de indisponibilidad de Redis (cola) (constitution.md:76). No existe TC que cubra qué sucede si Redis no está disponible durante la operación de caché o durante la operación normal de la cola. Se creará TC-30 en Fase 2.
- **Notificación/alertas de estados FAILED:** No existen TC para verificar que el sistema notifique o alerte (email, dashboard, webhook) cuando un faltante transita a FAILED (ej. `max_reposition_cycles_exceeded`, `curso no habilitado`). La constitución exige que las condiciones de fallo permanente sean detectables y trazables por el equipo de operaciones (constitution.md:96-97).
- **Migración de datos preexistentes:** No se cubre el comportamiento del sistema con registros de faltantes creados antes de la activación de esta funcionalidad. La constitución exige definir y probar el comportamiento con datos preexistentes (constitution.md:101). ¿Deben procesarse retroactivamente o solo aplica a nuevos registros? Se creará TC-33 en Fase 2.
- **Rotura-11 y Rotura-12 sin definir:** El documento afirma 14 roturas detectadas en grilling, pero solo 12 están identificadas en la matriz. Rotura-11 y Rotura-12 requieren definición por parte del equipo (QA + Tech Lead + PO). Ver notas al final del documento.

---

## 1. Test Strategy

### 1.1 Test Scope

| Tipo | Alcance | Objetivo |
|------|---------|----------|
| Unit | Lógica de detección de stock, validación estructural, reglas de negocio | 100% de reglas |
| Integration | Comunicación API NQ, colas Redis, persistencia, FIFO | Escenarios felices + fallos externos |
| Security | Protección de secretos, ofuscación de logs, sanitización de entrada externa | Invariantes de la Constitución |

### 1.2 Test Environment

Las pruebas se ejecutan en entornos aislados. La API de NQ se simula (mocks) para validar reintentos y manejo de errores 5xx sin consumir créditos reales.

---

## 2. Happy Path — HU-1 (Orquestación de envío)

### TC-01 · Detección de stock insuficiente

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-1, AC-1 |

**Given:** Un subtema con stock = 2 (threshold mínimo = 5) durante la generación de material.
**When:** Se solicita la generación y el sistema detecta que `question_count < threshold`.
**Then:** Se registra un faltante con `{curso, tema, subtema, nivel, estado = PENDING}`. Se encola un `GenerationJob` en Redis.

---

### TC-02 · División en bloques de 5 preguntas

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-1, AC-6 / Plan §1 (delay fijo configurable entre requests) |

**Given:** Un registro de faltante que requiere 12 preguntas para completar stock.
**When:** El `GenerationJob` inicia el procesamiento y la API de NQ tiene límite de 5 preguntas por petición.
**Then:** El sistema divide la solicitud en 3 peticiones: bloque 1 (5), bloque 2 (5), bloque 3 (2). Se respeta el límite de 5 por transacción. Entre cada petición se aplica el delay fijo configurable definido en el plan técnico.
**Verificación:** `NQClient::post` invocado exactamente 3 veces con `quantity = 5, 5, 2` respectivamente. Timestamps de invocación (`t1`, `t2`, `t3`) cumplen `t2 - t1 ≥ delay_configurado` y `t3 - t2 ≥ delay_configurado`. `SELECT COUNT(*) FROM nq_api_logs WHERE faltante_id = F1` = 3.

---

### TC-03 · Orden de procesamiento FIFO por fecha de generación + reintento conserva timestamp

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-1, AC-2, AC-3 |

**Given:** Tres faltantes pendientes con fechas de generación de material: `F1 (10:00)`, `F2 (10:05)`, `F3 (10:10)`. `F1` falla con `HTTP 500` y vuelve a `PENDING`.
**When:** El worker asíncrono consume la cola FIFO tras el reintento de `F1`.
**Then:** `F1` conserva su timestamp original (`10:00`). El orden de procesamiento es `F1 → F2 → F3`, con `F1` al frente por ser el más antiguo. Verificación: `SELECT faltante_id, MIN(timestamp) FROM process_traces WHERE estado = 'PENDING' GROUP BY faltante_id ORDER BY MIN(timestamp) ASC` retorna `['F1','F2','F3']`. `NQClient::post` recibe llamadas en orden `F1, F2, F3`.

---

### TC-04 · Payload con atributos requeridos

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-1, AC-5 |

**Given:** Un faltante con `{curso = Álgebra, tema = Ecuaciones, subtema = Lineales, nivel = 1}`.
**When:** El sistema construye el payload para enviar a NQ.
**Then:** El payload contiene exactamente `{curso, tema, subtema, nivel}`. No se envían atributos adicionales ni faltan los requeridos.

---

## 3. Happy Path — HU-2 (Validación y almacenamiento)

### TC-05 · Recepción, validación de duplicidad y almacenamiento en tabla temporal

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-2, AC-1, AC-2, AC-3, AC-6, AC-7 |

**Given:** NQ responde con 5 preguntas válidas y no duplicadas. El validador de duplicidad de Lumeria está operativo.
**When:** El sistema recibe la respuesta de NQ y ejecuta validación de duplicidad.
**Then:** Las 5 preguntas se insertan en la tabla temporal de revisión docente (banco de preguntas IA, previo a banco Lumeria). Las preguntas **no** van directo al banco final, permanecen en flujo de revisión docente existente. Verificación: `COUNT(*) FROM preguntas_temporal WHERE faltante_id = F1 AND source = 'NQ'` = 5.

---

## 4. Edge Cases

### TC-06 · Consolidación acumulativa de registros repetidos

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Spec — Caso Borde (múltiples faltantes misma combinación) / Plan §1 (acumulativos) |

**Given:** Se generan 3 registros de faltante para la misma combinación `(Álgebra, Ecuaciones, Lineales, Nivel 1)` con `requested_quantity = 5` cada uno, antes de ser procesados.
**When:** El worker asíncrono identifica los registros pendientes para esa combinación.
**Then:** El sistema suma las cantidades: `requested_quantity total = 15`. Se envía a NQ una solicitud consolidada con `quantity = 15` (dividida en 3 bloques de 5 por TC-02), no 3 solicitudes independientes de 5. Verificación: `COUNT(DISTINCT NQClient::post para combinación X)` = 3 bloques agrupados bajo un mismo proceso, no 9 llamadas independientes.

---

### TC-07 · Subtema sin nivel configurado

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Spec — Caso Borde (inconsistencias en configuración académica) |

**Given:** Un faltante cuyo subtema no tiene nivel asociado en la estructura académica del banco.
**When:** `GenerationJob` intenta construir el payload `{curso, tema, subtema, nivel}`.
**Then:** El sistema detecta que `nivel = NULL` y bloquea el envío a NQ. El faltante transita a `FAILED` con motivo: `subtema sin nivel configurado`. No se consume crédito de NQ. Verificación: `NQClient::post` no es invocado. `faltantes.estado = 'FAILED'`.

---

### TC-08 · Reposición automática por duplicados + estado PARTIAL + límite de 3 ciclos

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-2, AC-4, AC-5, AC-8 / Plan §2 (estado PARTIAL) |

**Given:** NQ genera 5 preguntas. El validador detecta 2 duplicadas. `requested_quantity = 5`, `generated_quantity = 3`.
**When:** El sistema descarta las 2 duplicadas y verifica que `generated_quantity (3) < requested_quantity (5)`.
**Then:** 
- El faltante transita a `PARTIAL`. 
- El sistema envía automáticamente solicitud de reposición a NQ por las 2 preguntas faltantes, respetando límite de 5 por bloque.
- Si la reposición devuelve 2 válidas → `generated_quantity = 5` → `COMPLETED`.
- Si la reposición vuelve a tener duplicados → nuevo ciclo de reposición.
- **Máximo 3 ciclos de reposición.** Si tras 3 ciclos `generated_quantity < requested_quantity`, transita a `FAILED` con motivo: `max_reposition_cycles_exceeded`.
- Verificación: `reposition_cycles` en metadata del faltante ≤ 3.

---

### TC-09 · Concurrencia — worker no procesa faltante en PROCESSING ni PARTIAL

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | NFR-3 / Spec — Caso Borde (reposición en curso) / Constitution Art. 3 |

**Given:** Faltante `F1` en estado `PENDING`. Dos workers consumen la misma cola simultáneamente.
**When:** Worker A ejecuta `UPDATE faltantes SET estado = 'PROCESSING' WHERE id = F1 AND estado = 'PENDING'` → 1 fila afectada. Worker B ejecuta la misma query → 0 filas afectadas.
**Then:** Worker B hace early return (no lanza excepción, no reencola, no llama a NQ). `F1` es procesado una sola vez. Verificación: `NQClient::post` invocado exactamente 1 vez para `F1`. **PARTIAL también es bloqueante**: mismo comportamiento si `F1.estado = 'PARTIAL'` — el UPDATE condicional con `WHERE estado IN ('PENDING', 'PARTIAL')` falla y el worker aborta.

---

### TC-10 · Curso no habilitado bloquea el envío

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-1, AC-4 |

**Given:** Un faltante pertenece al curso `Geometría`, el cual no está en la lista de cursos habilitados (`Álgebra, Aritmética, Trigonometría, Química`).
**When:** El sistema valida el curso antes de enviar a NQ.
**Then:** El envío se bloquea. El faltante transita a `FAILED` con motivo: `curso no habilitado para integración NQ`. No se consume crédito. Verificación: `NQClient::post` no es invocado. `faltantes.estado = 'FAILED'`.

---

## 5. Error Scenarios

### TC-11 · NQ HTTP 500 — registro permanece PENDING con backoff

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-1, AC-8 |

**Given:** La API de NQ responde `HTTP 500` en cada intento.
**When:** El `GenerationJob` envía la solicitud y recibe `500`.
**Then:** El sistema ejecuta reintentos con backoff incremental. Si todos fallan, el registro **permanece en estado `PENDING`** (no transita a `FAILED`) con `retry_count` incrementado y conservando su timestamp original de generación. El intento fallido queda registrado en logs. Verificación: `faltantes.estado = 'PENDING'`, `faltantes.retry_count > 0`.

---

### TC-12 · NQ responde HTTP 200 con body inválido o incompleto

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | Spec — Caso Borde (respuesta incompleta o inválida de NQ) |

**Given:** NQ responde `HTTP 200` pero el body está vacío (`{}`) o no contiene el campo `questions`.
**When:** El sistema intenta parsear la respuesta.
**Then:** El sistema detecta que la respuesta no cumple el schema esperado. El registro permanece en `PENDING`. Se registra error con motivo: `respuesta NQ inválida`. No se insertan preguntas. Verificación: `questions.count` sin cambio. Log contiene entry con `reason = 'invalid_nq_response'`.

---

### TC-13 · Falla de BD durante inserción — rollback total y sin huérfanos

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Constitution Art. 7 / Plan §2 (integridad) |

**Given:** NQ responde con 5 preguntas válidas, pero `pgsql_master` lanza excepción durante el `INSERT` en la tabla temporal.
**When:** La transacción de inserción falla.
**Then:** Rollback total. Cero preguntas persistidas (no hay huérfanos). Las preguntas recibidas de NQ **se descartan** (no se almacenan en caché). El faltante permanece en `PENDING` para reprocesamiento. En el reintento, el sistema solicita nuevas preguntas a NQ (implica nuevo consumo de créditos). El error se registra con `trace_id`, `timestamp` y `faltante_id`. Verificación: `COUNT(*) FROM preguntas_temporal WHERE faltante_id = F1` = 0. `faltantes.estado = 'PENDING'`.

---

### TC-14 · NQ devuelve más preguntas de las solicitadas

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | Constitution Art. 2 (sanitización de entrada externa) |

**Given:** Bloque de 2 preguntas enviado a NQ.
**When:** NQ responde con 5 preguntas (ignorando el límite solicitado de 2).
**Then:** El sistema acepta únicamente las primeras 2 preguntas de la respuesta o descarta el excedente. No se insertan más preguntas de las solicitadas para ese bloque. Verificación: `COUNT(*) FROM preguntas_temporal WHERE bloque_id = B3` ≤ 2.

---

## 6. Concurrencia y seguridad en encolado

### TC-15 · Doble encolado por requests HTTP simultáneos

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | HU-1, AC-1 / NFR-3 |

**Given:** Dos requests HTTP simultáneos generan material para el mismo subtema `S1` con stock insuficiente.
**When:** Ambos requests ejecutan la detección de faltantes en el mismo milisegundo (antes de que el primer INSERT sea visible para el segundo).
**Then:** El sistema detecta la condición de carrera a nivel de aplicación. Se crea exactamente 1 registro de faltante para `(curso, tema, subtema, nivel)` — ya sea mediante restricción UNIQUE en BD con upsert, o lock atómico a nivel de aplicación. Se encola exactamente 1 `GenerationJob`. Verificación: `COUNT(*) FROM faltantes WHERE subtema_id = S1 AND estado = 'PENDING'` = 1. `LLEN generate_nq_questions` incrementa en 1, no en 2.

---

## 7. Non-Functional & Cross-Cutting

### TC-16 · Máquina de estados — todos los ciclos de vida

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Constitution Art. 7 / Plan §2 |

**Given:** Cuatro escenarios: (a) flujo exitoso sin duplicados, (b) flujo con reposición por duplicados, (c) error permanente, (d) curso deshabilitado en NQ.
**When:** Se ejecuta cada escenario de principio a fin.
**Then:**
- **(a) Sin duplicados:** `PENDING → PROCESSING → COMPLETED`. No existe `PENDING → COMPLETED` sin `PROCESSING`.
- **(b) Con reposición:** `PENDING → PROCESSING → PARTIAL → PROCESSING → COMPLETED`. No existe `PARTIAL → COMPLETED` sin `PROCESSING` intermedio.
- **(c) Error permanente:** `PENDING → PROCESSING → FAILED`. No existe `PENDING → FAILED` sin `PROCESSING`.
- **(d) Curso deshabilitado en NQ:** `PENDING → PROCESSING → CANCELLED`. No existe `PENDING → CANCELLED` sin `PROCESSING`. `CANCELLED` es estado terminal (no se re-procesa automáticamente).
- Verificación para todos: `SELECT estado FROM process_traces WHERE faltante_id = F1 ORDER BY timestamp ASC`. Estados consecutivos con timestamps incrementales (`t1 < t2 < t3`).

---

### TC-17 · Idempotencia del GenerationJob en todos los estados terminales y activos

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Constitution Art. 3 |

**Given:** Cinco escenarios de faltante ya procesado: `F_COMPLETED`, `F_FAILED`, `F_CANCELLED`, `F_PARTIAL`, `F_PROCESSING`.
**When:** Se intenta ejecutar un segundo `GenerationJob` para cada uno (por duplicado en cola o reintento espurio).
**Then:**
- `F_COMPLETED`: Job B detecta `estado = COMPLETED` → early return. `NQClient::post` no invocado.
- `F_FAILED`: Job B detecta `estado = FAILED` → early return. No se reintenta automáticamente.
- `F_CANCELLED`: Job B detecta `estado = CANCELLED` → early return. No se reintenta automáticamente.
- `F_PARTIAL`: Job B detecta `estado = PARTIAL` → early return (PARTIAL es bloqueante, TC-09).
- `F_PROCESSING`: Job B detecta `estado = PROCESSING` → early return (PROCESSING es bloqueante, TC-09).
- En ningún caso se duplica `generated_quantity` ni se envían solicitudes adicionales a NQ. Verificación: `NQClient::post` no invocado para ninguno. `generated_quantity` sin cambio.

---

### TC-18 · ODISEO_KEY no expuesta en logs — todos los niveles de log (Parte A: ofuscación de secretos)

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Constitution Art. 2 |

> **Nota:** Este TC cubre únicamente la ofuscación del secreto en logs y stack traces. El comportamiento completo del sistema ante HTTP 401/403 (transición de estado, reintentos, registro de auditoría) se cubrirá en TC-28 (Fase 2).

**Given:** `GenerationJob` ejecuta flujo exitoso (`HTTP 200`) y también falla con error `HTTP 401`. `ODISEO_KEY` está configurada en `.env`.
**When:** El sistema registra logs en ambos escenarios (info en éxito, error en fallo). Se inspeccionan `storage/logs/laravel.log`, stack traces de Sentry y headers de requests logueados.
**Then:** En **ningún** nivel de log (debug, info, error) aparece el valor real de `ODISEO_KEY`. Verificación: `grep -r "<valor real de ODISEO_KEY>" storage/logs/` retorna 0 líneas. `grep -ri "authorization" storage/logs/` no contiene el valor del token. Stack traces no contienen headers de autorización. El constructor de `NQClient` no loguea la key al cargarla de configuración.

---

### TC-19 · Campos de trazabilidad poblados correctamente al finalizar

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | Plan §2 (Información sugerida) |

**Given:** Un faltante se procesa completamente con 1 ciclo de reposición (2 duplicados requirieron 1 reposición exitosa).
**When:** El proceso finaliza en `COMPLETED`.
**Then:** Verificación vía `SELECT` sobre el registro del faltante:
- `requested_quantity` = cantidad original solicitada (ej: 5).
- `generated_quantity` = total de preguntas válidas insertadas en tabla temporal (ej: 5 = 3 iniciales + 2 reposición).
- `pending_quantity` = `requested - generated` = 0.
- `processed_at` = timestamp de cuando transita a `COMPLETED`, con valor no nulo.
- `retry_count` = número de reintentos por error NQ (independiente de ciclos de reposición).
- `reposition_cycles` = 1 (cantidad de ciclos de reposición ejecutados).

---

### TC-20 · División en bloques durante reposición con cantidad pendiente > 5

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | HU-2, AC-5, AC-8 |

**Given:** `requested_quantity = 15`. NQ generó 5 preguntas pero las 5 eran duplicadas. `pending_quantity = 15`.
**When:** El sistema dispara reposición por las 15 preguntas faltantes.
**Then:** El sistema divide la reposición en 3 bloques de 5 (5+5+5), respetando el límite máximo por transacción. No intenta enviar un solo bloque de 15. Verificación: `NQClient::post` invocado 3 veces durante la reposición, cada una con `quantity = 5`.

---

### TC-21 · Sanitización de contenido proveniente de NQ

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Constitution Art. 2 (validar y sanitizar datos de entrada externa) |

**Given:** NQ responde con una pregunta cuyo `enunciado` contiene `<script>alert(1)</script>` o `opciones` con caracteres de control Unicode.
**When:** El sistema recibe la respuesta de NQ y la procesa antes de insertar en la tabla temporal.
**Then:** El contenido se sanitiza antes de la inserción: tags HTML escapados (`&lt;script&gt;`), caracteres de control removidos o normalizados. La pregunta se inserta sanitizada, no con el payload crudo de NQ. Verificación: `SELECT enunciado FROM preguntas_temporal WHERE id = P1` no contiene `<script>` sin escapar.

---

### TC-22 · NQ devuelve preguntas duplicadas dentro del mismo lote

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | HU-2, AC-2, AC-3 |

**Given:** NQ responde con 5 preguntas en un mismo bloque. 2 de ellas son idénticas entre sí (mismo `content_hash` dentro del lote).
**When:** El validador de duplicidad procesa el lote.
**Then:** La primera ocurrencia se inserta. La segunda se descarta como duplicada (aunque no existiera previamente en la BD). Se dispara reposición por la pregunta duplicada. Verificación: `COUNT(*) FROM preguntas_temporal WHERE bloque_id = B1` = 4 (5 - 1 duplicada interna). `rejected_questions` contiene 1 registro con `reason = 'duplicate_in_batch'`.

---

## 8. Recovery & Resilience

### TC-23 · Stale job recovery — scheduled command libera PROCESSING zombies

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | NFR-3 / Constitution Art. 3 |

**Given:** Faltante `F1` en estado `PROCESSING` hace más de 30 minutos (worker crash por OOM, deploy, timeout).
**When:** Scheduled command `faltantes:recover-stale` se ejecuta (cada 5 minutos).
**Then:** El comando ejecuta `UPDATE faltantes SET estado = 'PENDING' WHERE estado = 'PROCESSING' AND updated_at < NOW() - INTERVAL '30 minutes'`. `F1` vuelve a `PENDING` sin modificar `retry_count`. El worker pickup normal lo retoma.
**Verification:** `SELECT estado FROM faltantes WHERE id = F1` → `PENDING`. `NQClient::post` invocado 1 vez post-recovery (no antes). Log contiene entry con `action = 'stale_recovery'` y `faltante_id = F1`.

---

### TC-24 · NQ devuelve menos preguntas de las solicitadas — éxito parcial + reposición

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | HU-2, AC-4 |

**Given:** Bloque de 5 preguntas enviado a NQ. NQ responde `HTTP 200` con 3 preguntas válidas (sin error).
**When:** Sistema recibe las 3 preguntas, las valida estructuralmente y de duplicidad.
**Then:** Las 3 preguntas se insertan en tabla temporal. `generated_quantity += 3`. `pending_quantity` se reduce en 3. Como `pending_quantity > 0`, el sistema dispara reposición por las 2 preguntas faltantes. No se trata como error.
**Verification:** `generated_quantity = 3` tras el bloque. `NQClient::post` invocado 1 vez (bloque original) + 1 vez (reposición por 2). `reposition_cycles = 1`.

---

### TC-25 · PARTIAL + error NQ en reposición incrementa reposition_cycles

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | Plan §1 (reposición con 3 ciclos máx) |

**Given:** Faltante `F1` en `PARTIAL` con `reposition_cycles = 1`, `pending_quantity = 2`.
**When:** La reposición envía solicitud a NQ y recibe `HTTP 500`.
**Then:** `reposition_cycles` se incrementa a 2 (el ciclo fallido cuenta contra el límite de 3). El registro vuelve a `PENDING` conservando `reposition_cycles = 2`. Si en el siguiente procesamiento el error persiste → `reposition_cycles = 3` → `FAILED` con motivo `max_reposition_cycles_exceeded`.
**Verification:** `SELECT reposition_cycles FROM faltantes WHERE id = F1` = 2 tras el error. En estado `PENDING` con `reposition_cycles` intacto.

---

### TC-26 · Máximo 3 reintentos por ciclo para error NQ 5xx

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | HU-1, AC-8 / Plan §4 (reintentos) |

**Given:** NQ responde `HTTP 500` en cada intento dentro de un mismo ciclo de procesamiento.
**When:** Worker inicia procesamiento → intento 1 (500) → intento 2 (500) → intento 3 (500).
**Then:** Tras 3 reintentos fallidos en este ciclo, el sistema NO reintenta una 4ª vez. El registro vuelve a `PENDING` con `retry_count = 3`. El worker pickup en el siguiente ciclo comenzará de nuevo con 3 reintentos frescos.
**Verification:** `NQClient::post` invocado exactamente 3 veces en este ciclo. `retry_count = 3`. Estado = `PENDING`. No hay un 4º intento en el mismo ciclo.

---

### TC-27 · Cache de respuesta NQ ante fallo de BD — evita doble consumo de créditos

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | TC-13 / Plan §2 (integridad) |

**Given:** NQ responde con 5 preguntas válidas (`HTTP 200`). La transacción de inserción en tabla temporal falla (BD caída).
**When:** El sistema captura la excepción de BD y ejecuta rollback.
**Then:** El sistema persiste la respuesta de NQ en Redis asociada al `faltante_id` con TTL de 24h (`cache:nq_response:{faltante_id}`). El faltante vuelve a `PENDING`. En el reintento, el sistema verifica si existe respuesta cacheada antes de llamar a NQ. Si existe, reusa la respuesta sin consumir créditos NQ.
**Verification:** En el reintento, `NQClient::post` NO es invocado. Las 5 preguntas se insertan desde caché. `generated_quantity = 5`. `SELECT COUNT(*) FROM preguntas_temporal WHERE faltante_id = F1` = 5.

---

### TC-28 · NQ responde HTTP 429 (Rate Limiting) — backoff con Retry-After

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | NFR-5 / Spec §3 — HTTP 429 |

**Given:** La API de NQ responde `HTTP 429` con header `Retry-After: 30`.
**When:** El `GenerationJob` envía la solicitud y recibe `429`.
**Then:** El sistema NO intenta inmediatamente. Espera 30 segundos (valor del header `Retry-After`) y reintenta. Si tras 3 reintentos el error persiste, el registro vuelve a `PENDING` conservando su timestamp original.
**Verification:** `NQClient::post` invocado 3 veces con intervalos ≥ 30s entre cada uno. `faltantes.estado = 'PENDING'`, `retry_count = 3`. Log contiene entry con `reason = 'rate_limited'`.

---

### TC-29 · Connection / Read timeout hacia NQ

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | NFR-6 / Spec §3 — Timeouts |

**Given:** El cliente HTTP lanza excepción por `connection timeout` (NQ no responde en < 5s).
**When:** El `GenerationJob` envía la solicitud y captura el timeout.
**Then:** El sistema trata el timeout como error 5xx: reintenta con backoff hasta 3 veces. Si todos fallan, el registro vuelve a `PENDING` con `retry_count` incrementado y conservando su timestamp original.
**Verification:** `NQClient::post` invocado con backoff creciente (ej. 1s, 2s, 4s). `faltantes.estado = 'PENDING'`, `retry_count = 3`. Se prueba también con `read timeout` (respuesta parcial).

---

### TC-30 · Redis indisponible durante operación de cola

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | NFR-3 / Constitution Art. 6.3 — Resiliencia |

**Given:** Redis no está disponible (caída del servicio, timeout de conexión).
**When:** El sistema intenta encolar un `GenerationJob` o cachear respuesta de NQ.
**Then:** El job de encolado falla con excepción. El registro de faltante permanece en `PENDING`. En el caso de caché de respuesta NQ (TC-27), el sistema no persiste la respuesta en caché y en el reintento debe llamar a NQ nuevamente (consume nuevo crédito). Se registra alerta en logs para monitoreo vía Horizon.
**Verification:** `faltantes.estado = 'PENDING'` sin cambios. Log contiene entry con `reason = 'redis_unavailable'`.

---

### TC-31 · Carga/volumen — procesamiento FIFO con miles de registros pendientes

| Atributo | Valor |
|----------|-------|
| Prioridad | P2 |
| Traces To | NFR-3 / Constitution Art. 6.4 — Rendimiento |

**Given:** Existen 5000 registros de faltantes `PENDING` con fechas de generación distribuidas en 7 días.
**When:** El worker batch procesa la cola FIFO secuencialmente.
**Then:** Los registros se procesan en estricto orden cronológico sin pérdida de prioridad. El throughput se mantiene dentro de la latencia esperada (< 1s por registro, sin contar tiempo de NQ). No se observan timeouts de cola ni degradación en el orden FIFO.
**Verification:** `SELECT faltante_id FROM faltantes WHERE estado = 'COMPLETED' ORDER BY processed_at ASC` retorna los IDs en orden FIFO. El tiempo total de procesamiento de los 5000 registros es medible y documentado.

---

## 9. Matriz de Trazabilidad

| Requisito | TC asociado | Estado |
|-----------|-------------|--------|
| HU-1 AC-1 — Ejecución automática | TC-01, TC-15 | ✅ |
| HU-1 AC-2 — FIFO cronológico | TC-03 | ✅ |
| HU-1 AC-3 — Cola FIFO | TC-03 | ✅ |
| HU-1 AC-4 — Cursos habilitados | TC-04, TC-10 | ✅ |
| HU-1 AC-5 — Atributos payload | TC-04 | ✅ |
| HU-1 AC-6 — División por bloques | TC-02, TC-20 | ✅ |
| HU-1 AC-7 — API existente NQ | TC-01, TC-05 | ✅ |
| HU-1 AC-8 — Error → PENDING | TC-11 | ✅ |
| HU-2 AC-1 — Recepción automática | TC-05 | ✅ |
| HU-2 AC-2 — Validación duplicidad | TC-05, TC-08, TC-22 | ✅ |
| HU-2 AC-3 — Descarte duplicados | TC-05, TC-08, TC-22 | ✅ |
| HU-2 AC-4 — Reposición automática | TC-08 | ✅ |
| HU-2 AC-5 — Reenvío bajo bloques | TC-08, TC-20 | ✅ |
| HU-2 AC-6 — Tabla temporal | TC-05 | ✅ |
| HU-2 AC-7 — Conservación flujo actual | TC-05 | ✅ |
| HU-2 AC-8 — Reintento hasta completar | TC-08 | ✅ |
| Spec CB — Misma combinación repetida | TC-06 | ✅ |
| Spec CB — Reposición en curso | TC-09, TC-15 | ✅ |
| Spec CB — NQ devuelve duplicados | TC-05, TC-08, TC-22 | ✅ |
| Spec CB — NQ respuesta inválida | TC-12, TC-14 | ✅ |
| Spec CB — Inconsistencia académica | TC-07 | ✅ |
| NFR-1 — Procesamiento asíncrono | TC-01, TC-03, TC-11 | ✅ |
| NFR-2 — Integridad / no duplicados | TC-05, TC-08, TC-13, TC-22 | ✅ |
| NFR-3 — Procesamiento concurrente | TC-09, TC-15 | ✅ |
| Constitution Art. 2 — Seguridad (secretos, sanitización) | TC-18, TC-21 | ✅ |
| Constitution Art. 3 — Idempotencia, concurrencia | TC-17, TC-09 | ✅ |
| Constitution Art. 7 — Trazabilidad, logs, estados | TC-16, TC-13, TC-19 | ✅ |
| Plan §2 — Estado PARTIAL | TC-08, TC-16 | ✅ |
| Plan §2 — Campos trazabilidad | TC-19 | ✅ |
| Rotura-1 — Doble encolado concurrente | TC-15 | ✅ |
| Rotura-2 — PARTIAL bloqueante | TC-09 | ✅ |
| Rotura-3 — Límite 3 ciclos reposición | TC-08 | ✅ |
| Rotura-4 — Stale job recovery | TC-23 | ✅ |
| Rotura-5 — Descarte respuesta NQ en fallo BD | TC-13 | ✅ |
| Rotura-6 — Idempotencia extendida | TC-17 | ✅ |
| Rotura-7 — FIFO conserva timestamp | TC-03 | ✅ |
| Rotura-8 — ODISEO_KEY en todos los logs | TC-18 | ✅ |
| Rotura-9 — División reposición > 5 | TC-20 | ✅ |
| Rotura-10 — Ciclos de estado alternos | TC-16 | ✅ |
| Rotura-11 — Pendiente de definición | — | ⚠️ Sin TC |
| Rotura-12 — Pendiente de definición | — | ⚠️ Sin TC |
| Rotura-14 — Sanitización contenido NQ | TC-21 | ✅ |
| NQ devuelve menos preguntas de las solicitadas | TC-24 | ✅ |
| PARTIAL + error NQ incrementa reposition_cycles | TC-25 | ✅ |
| Máximo 3 reintentos por ciclo NQ 5xx | TC-26 | ✅ |
| Cache respuesta NQ en fallo BD | TC-27 | ✅ |

---

## Notas para el equipo

- **Rotura-4 (stale jobs):** Mitigado mediante scheduled command `faltantes:recover-stale`. Comando programado (cron) ejecutado cada 5 minutos detecta faltantes en `PROCESSING` con `updated_at > 30 min` y los devuelve a `PENDING` (ver TC-23).
- **Rotura-11 y Rotura-12:** Detectadas en sesión de grilling pero no definidas en la matriz de trazabilidad. El equipo (QA + Tech Lead + PO) debe definir ambos riesgos y asignarles TC correspondiente o documentar su exclusión justificada (similar a Rotura-13).
- **Rotura-13 (HTTP response al usuario):** El endpoint de generación de material es preexistente y no forma parte de esta implementación.
- **TC pendientes (Fase 2):** Se crearán TC-28 (HTTP 4xx → FAILED inmediato), TC-29 (timeout HTTP), TC-30 (Redis indisponible), TC-31 (HTTP 429 rate limiting), TC-32 (carga/volumen FIFO) y TC-33 (datos preexistentes / migración) para cubrir los gaps identificados en "Qué quedó sin cubrir" y satisfacer los requisitos de la constitución (Art. 6.3-6.7).
- **Decisiones pendientes (Fase 1):** Quedan 4 contradicciones entre spec/plan/test-cases que requieren resolución del equipo: (1) consolidación vs no consolidación de registros repetidos, (2) CANCELLED vs FAILED para curso no habilitado en NQ, (3) PENDING vs FAILED al agotar reintentos, (4) re-procesamiento automático de registros FAILED. Ver plan de acción.

---

## Vigencia

Este plan de pruebas entra en vigor inmediatamente. Todo TC debe ser verificable con resultados medibles (estados en BD, conteo de registros, HTTP status codes, contenido de logs). El artefacto debe actualizarse si se modifican los ACs del spec o los estados del plan.
