# Test Cases — Automatización de Preguntas NQ

> Plan de pruebas para la orquestación asíncrona de reposición de stock académico e integración con NQ.
> Cobertura: HU-1 (8 ACs), HU-2 (8 ACs), Casos Borde, NFRs, Constitution, Plan.
> Incluye mitigaciones para 14 roturas detectadas en grilling (concurrencia en encolado, PARTIAL, idempotencia extendida, sanitización, ciclos de reposición, FIFO con reintentos).

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
| Traces To | HU-1, AC-6 |

**Given:** Un registro de faltante que requiere 12 preguntas para completar stock.
**When:** El `GenerationJob` inicia el procesamiento y la API de NQ tiene límite de 5 preguntas por petición.
**Then:** El sistema divide la solicitud en 3 peticiones: bloque 1 (5), bloque 2 (5), bloque 3 (2). Se respeta el límite de 5 por transacción.

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

**Given:** Tres escenarios: (a) flujo exitoso sin duplicados, (b) flujo con reposición por duplicados, (c) error permanente.
**When:** Se ejecuta cada escenario de principio a fin.
**Then:**
- **(a) Sin duplicados:** `PENDING → PROCESSING → COMPLETED`. No existe `PENDING → COMPLETED` sin `PROCESSING`.
- **(b) Con reposición:** `PENDING → PROCESSING → PARTIAL → PROCESSING → COMPLETED`. No existe `PARTIAL → COMPLETED` sin `PROCESSING` intermedio.
- **(c) Error permanente:** `PENDING → PROCESSING → FAILED`. No existe `PENDING → FAILED` sin `PROCESSING`.
- Verificación para todos: `SELECT estado FROM process_traces WHERE faltante_id = F1 ORDER BY timestamp ASC`. Estados consecutivos con timestamps incrementales (`t1 < t2 < t3`).

---

### TC-17 · Idempotencia del GenerationJob en todos los estados terminales y activos

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Constitution Art. 3 |

**Given:** Cuatro escenarios de faltante ya procesado: `F_COMPLETED`, `F_FAILED`, `F_PARTIAL`, `F_PROCESSING`.
**When:** Se intenta ejecutar un segundo `GenerationJob` para cada uno (por duplicado en cola o reintento espurio).
**Then:**
- `F_COMPLETED`: Job B detecta `estado = COMPLETED` → early return. `NQClient::post` no invocado.
- `F_FAILED`: Job B detecta `estado = FAILED` → early return. No se reintenta automáticamente.
- `F_PARTIAL`: Job B detecta `estado = PARTIAL` → early return (PARTIAL es bloqueante, TC-09).
- `F_PROCESSING`: Job B detecta `estado = PROCESSING` → early return (PROCESSING es bloqueante, TC-09).
- En ningún caso se duplica `generated_quantity` ni se envían solicitudes adicionales a NQ. Verificación: `NQClient::post` no invocado para ninguno. `generated_quantity` sin cambio.

---

### TC-18 · ODISEO_KEY no expuesta en logs — todos los niveles de log

| Atributo | Valor |
|----------|-------|
| Prioridad | P1 |
| Traces To | Constitution Art. 2 |

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

## 9. Matriz de Trazabilidad

| Requisito | TC asociado | Estado |
|-----------|-------------|--------|
| HU-1 AC-1 — Ejecución automática | TC-01, TC-15 | ✅ |
| HU-1 AC-2 — FIFO cronológico | TC-03 | ✅ |
| HU-1 AC-3 — Cola FIFO | TC-03 | ✅ |
| HU-1 AC-4 — Cursos habilitados | TC-04, TC-10 | ✅ |
| HU-1 AC-5 — Atributos payload | TC-04 | ✅ |
| HU-1 AC-6 — División por bloques | TC-02, TC-20 | ✅ |
| HU-1 AC-7 — API existente NQ | TC-01, TC-02, TC-05 | ✅ |
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
| Rotura-5 — Descarte respuesta NQ en fallo BD | TC-13 | ✅ |
| Rotura-6 — Idempotencia extendida | TC-17 | ✅ |
| Rotura-7 — FIFO conserva timestamp | TC-03 | ✅ |
| Rotura-8 — ODISEO_KEY en todos los logs | TC-18 | ✅ |
| Rotura-9 — División reposición > 5 | TC-20 | ✅ |
| Rotura-10 — Ciclos de estado alternos | TC-16 | ✅ |
| Rotura-14 — Sanitización contenido NQ | TC-21 | ✅ |
| Rotura-4 — Stale job recovery | TC-23 | ✅ |
| NQ devuelve menos preguntas de las solicitadas | TC-24 | ✅ |
| PARTIAL + error NQ incrementa reposition_cycles | TC-25 | ✅ |
| Máximo 3 reintentos por ciclo NQ 5xx | TC-26 | ✅ |
| Cache respuesta NQ en fallo BD | TC-27 | ✅ |

---

## Notas para el equipo

- **Rotura-4 (stale jobs):** Mitigado mediante scheduled command `faltantes:recover-stale`. Comando programado (cron) ejecutado cada 5 minutos detecta faltantes en `PROCESSING` con `updated_at > 30 min` y los devuelve a `PENDING` (ver TC-23).
- **Rotura-13 (HTTP response al usuario):** El endpoint de generación de material es preexistente y no forma parte de esta implementación.

---

## Vigencia

Este plan de pruebas entra en vigor inmediatamente. Todo TC debe ser verificable con resultados medibles (estados en BD, conteo de registros, HTTP status codes, contenido de logs). El artefacto debe actualizarse si se modifican los ACs del spec o los estados del plan.
