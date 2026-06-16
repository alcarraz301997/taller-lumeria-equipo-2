# Spec - Automatización de Preguntas NQ en Segundo Plano

## Resumen ejecutivo

Se implementará un flujo automatizado de reabastecimiento del banco de preguntas de Lumeria mediante integración con NQ. Actualmente, cuando durante la generación de material académico no existen suficientes preguntas disponibles, el sistema registra preguntas faltantes asociadas a tema, subtema y nivel. Sin embargo, dichos registros solo cumplen una función de trazabilidad y no activan procesos de reposición. La nueva funcionalidad utilizará estos registros como insumo para generar solicitudes automáticas en segundo plano hacia NQ, permitiendo generar nuevas preguntas e incorporarlas al banco de forma automática. Esto garantizará una mayor disponibilidad de contenido, reducirá la intervención manual y mejorará la capacidad de respuesta ante futuras solicitudes de generación.

---

# 1. Contexto de negocio (qué y por qué)

## Problema que resuelve

Actualmente, cuando un usuario genera material académico y el sistema no encuentra suficientes preguntas disponibles en el banco, Lumeria registra internamente preguntas faltantes según tema, subtema y nivel. Sin embargo, este registro únicamente sirve como trazabilidad operativa y no activa ningún proceso automático de reposición, lo que genera dependencia de procesos manuales posteriores para recuperar stock de preguntas.

## Por qué ahora / a quién impacta

Esta limitación impacta directamente la disponibilidad futura del banco de preguntas, ya que la falta de reposición automática genera acumulación progresiva de faltantes y reduce la capacidad del sistema para responder eficientemente a futuras generaciones de material.

La automatización permitirá mantener un banco constantemente abastecido, reduciendo carga operativa interna y mejorando la escalabilidad del ecosistema de generación académica.

---

# 2. User Stories y Criterios de Aceptación

## US-1 (P1)

**Como** sistema de generación de materiales,
**quiero** procesar automáticamente los registros de preguntas faltantes,
**para** reabastecer el banco de preguntas mediante integración con NQ.

### AC-1.1 Registro estructurado de faltantes

**Dado** que durante la generación de material no existen suficientes preguntas disponibles para completar la solicitud,

**Cuando** el sistema detecta la ausencia de preguntas requeridas,

**Entonces** debe registrar obligatoriamente un faltante almacenando tema, subtema y nivel asociado como estructura de reposición pendiente.

### AC-1.2 Procesamiento automático en segundo plano

**Dado** que existe un registro de faltante pendiente generado por el sistema,

**Cuando** el proceso automático de sincronización identifica dicho registro,

**Entonces** debe enviar una solicitud automática hacia NQ incluyendo como payload el tema, subtema y nivel correspondiente al faltante registrado.

### AC-1.3 Reabastecimiento automático del banco

**Dado** que NQ responde exitosamente con preguntas generadas,

**Cuando** el sistema recibe la respuesta de generación,

**Entonces** debe insertar automáticamente las nuevas preguntas dentro del banco validando integridad estructural y evitando duplicidad de contenido.

---

# 3. Requisitos No Funcionales (NFR)

### NFR-1

El proceso de envío de solicitudes hacia NQ debe ejecutarse de forma asíncrona sin afectar el rendimiento del proceso principal de generación de material.

### NFR-2

La inserción de nuevas preguntas en el banco debe validar reglas de integridad de datos para evitar registros duplicados dentro del repositorio de preguntas.

### NFR-3

El sistema debe soportar procesamiento concurrente de múltiples registros de faltantes pendientes sin bloquear operaciones simultáneas de generación.

---

# 4. Casos Borde

* El mismo tema, subtema y nivel generan múltiples faltantes antes de ser procesados.
* Existe una reposición en curso para el mismo faltante.
* NQ devuelve preguntas ya existentes dentro del banco.
* NQ devuelve una respuesta incompleta o inválida.
* Un faltante no puede procesarse por inconsistencias en la configuración académica.

---

# 5. Assumptions

Se asume que toda estructura académica del banco mantiene una relación obligatoria entre subtema y nivel, permitiendo construir correctamente el payload enviado a NQ.

Si esta relación no existe o presenta inconsistencias, el sistema no podrá procesar correctamente la generación automática y deberá bloquear el envío del registro.

---

# 6. NEEDS_CLARIFICATION

1. ¿Cuántas preguntas debe solicitar el sistema a NQ por cada registro de faltante detectado?
2. ¿El sistema debe consolidar registros repetidos antes de enviar solicitudes a NQ o procesarlos individualmente?
3. ¿El procesamiento hacia NQ debe ejecutarse inmediatamente después del registro o mediante procesamiento batch programado?

---

# 7. Scope

## DENTRO

* Registro automático de preguntas faltantes durante generación de material.
* Lectura automática de registros pendientes de reposición.
* Envío automático de solicitudes a NQ utilizando tema, subtema y nivel.
* Procesamiento asíncrono en segundo plano.
* Inserción automática de preguntas generadas dentro del banco.
* Validación de integridad y control de duplicados.

## FUERA (Explícito)

* Generación manual de preguntas por parte del usuario.
* Edición manual del contenido generado por NQ dentro de este flujo.
* Generación de preguntas contextualizadas o enriquecidas con contenido adicional.
* Gestión manual de aprobaciones antes de insertar preguntas en el banco.
