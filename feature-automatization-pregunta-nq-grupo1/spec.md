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

# HU-1: Orquestar envío automático de preguntas faltantes hacia NQ

## Contexto

Actualmente Lumeria registra preguntas faltantes durante el proceso de generación de materiales académicos. Sin embargo, el abastecimiento de estas preguntas requiere intervención manual de docentes mediante el flujo tradicional de generación en NQ, generando retrasos operativos y dependencia manual para iniciar la generación.

Se requiere automatizar el envío de estos faltantes hacia NQ reutilizando la integración backend existente para iniciar automáticamente el proceso de generación de preguntas.

---

## Valor

Reduce el tiempo de abastecimiento de preguntas faltantes eliminando la intervención manual inicial y permitiendo iniciar automáticamente la generación de nuevas preguntas desde el backlog de faltantes.

---

## Descripción

El sistema debe identificar automáticamente los registros de preguntas faltantes pendientes, organizarlos por prioridad temporal y enviarlos automáticamente hacia NQ utilizando la integración backend existente.

Cada solicitud deberá enviarse en bloques de hasta 5 preguntas respetando el flujo actual de generación definido por NQ y procesando los registros bajo una cola FIFO basada en la fecha de generación del material original.

---

## Rol

Administrador

---

## Alcance

* Registro de preguntas faltantes existentes en Lumeria
* Integración backend API con NQ
* Envío automático de solicitudes de generación
* Procesamiento FIFO de registros pendientes

---

## Criterios de Aceptación

### 1. Ejecución automática del envío

**WHEN** existan registros de preguntas faltantes pendientes de procesamiento
**THE SYSTEM SHALL** iniciar automáticamente el proceso de envío hacia NQ sin requerir intervención manual.

---

### 2. Procesamiento por orden de prioridad

**WHEN** existan múltiples registros pendientes de envío
**THE SYSTEM SHALL** procesarlos en orden cronológico desde el material más antiguo al más reciente.

---

### 3. Regla de cola FIFO

**WHILE** existan múltiples materiales pendientes de procesamiento
**THE SYSTEM SHALL** mantener una cola FIFO utilizando como referencia la fecha de generación del material asociado al faltante.

---

### 4. Validación de cursos habilitados

**WHEN** el sistema prepare información para enviar a NQ
**THE SYSTEM SHALL** enviar únicamente registros pertenecientes a cursos habilitados para esta integración.

Cursos habilitados iniciales:

* Álgebra
* Aritmética
* Trigonometría
* Química

---

### 5. Validación de atributos enviados

**WHEN** se genere una solicitud hacia NQ
**THE SYSTEM SHALL** enviar únicamente los atributos requeridos por la API de integración.

Atributos enviados:

* Curso
* Tema
* Subtema
* Nivel

---

### 6. División por bloques

**WHEN** un registro de faltante requiera generación de múltiples preguntas
**THE SYSTEM SHALL** dividir automáticamente la solicitud en bloques máximos de 5 preguntas por envío respetando la restricción operativa actual de NQ.

---

### 7. Consumo de API existente

**WHEN** el sistema ejecute un envío automático
**THE SYSTEM SHALL** utilizar la API backend actualmente disponible en NQ sin modificar el flujo interno de generación existente.

---

### 8. Manejo de error de integración

**IF** la API de NQ no responde correctamente
**THEN THE SYSTEM SHALL** registrar el intento fallido y mantener el registro en estado pendiente para su posterior reprocesamiento.

---
# HU-2: Validar duplicidad y almacenar preguntas generadas provenientes de NQ

## Contexto

Una vez que NQ genera automáticamente nuevas preguntas a partir de solicitudes enviadas por Lumeria, es necesario validar que estas preguntas no existan previamente dentro del banco de preguntas.

Actualmente Lumeria ya cuenta con una funcionalidad de validación de duplicidad, por lo que se requiere integrarla automáticamente al flujo para asegurar que únicamente preguntas válidas continúen hacia la tabla temporal de revisión.

---

## Valor

Evita el ingreso de preguntas duplicadas al flujo de revisión docente asegurando calidad del banco de preguntas y reduciendo reprocesamientos manuales.

---

## Descripción

El sistema debe recibir automáticamente las preguntas generadas por NQ, ejecutar validación de duplicidad utilizando el mecanismo existente en Lumeria y almacenar únicamente las preguntas válidas en la tabla temporal de revisión.

Si durante la validación se detectan preguntas duplicadas, el sistema deberá solicitar automáticamente nuevas preguntas a NQ hasta completar la cantidad requerida originalmente.

---

## Rol

Administrador

---

## Alcance

* Recepción automática de preguntas generadas desde NQ
* Validación automática de duplicidad en Lumeria
* Reenvío automático a NQ en caso de duplicados
* Almacenamiento en tabla temporal existente

---

## Criterios de Aceptación

### 1. Recepción automática de preguntas

**WHEN** NQ retorne preguntas generadas
**THE SYSTEM SHALL** recibir automáticamente la respuesta e iniciar el flujo de validación interna.

---

### 2. Validación automática de duplicidad

**WHEN** se reciban preguntas generadas desde NQ
**THE SYSTEM SHALL** ejecutar automáticamente la validación de duplicidad utilizando la lógica actualmente existente en Lumeria.

---

### 3. Descarte automático de duplicados

**IF** una pregunta recibida es identificada como duplicada
**THEN THE SYSTEM SHALL** excluir automáticamente dicha pregunta del proceso de almacenamiento.

---

### 4. Solicitud automática de reposición

**IF** durante la validación existan preguntas descartadas por duplicidad
**THEN THE SYSTEM SHALL** generar automáticamente una nueva solicitud hacia NQ solicitando únicamente la cantidad faltante hasta completar el volumen originalmente requerido.

---

### 5. Reenvío bajo regla de bloques

**WHEN** se solicite reposición de preguntas descartadas
**THE SYSTEM SHALL** enviar solicitudes respetando el límite máximo de 5 preguntas por transacción definido por el flujo actual de NQ.

---

### 6. Almacenamiento de preguntas válidas

**WHEN** una pregunta supere exitosamente la validación de duplicidad
**THE SYSTEM SHALL** almacenarla automáticamente en la tabla temporal actualmente utilizada para revisión docente.

---

### 7. Conservación del flujo actual

**WHILE** las preguntas permanezcan en estado pendiente de revisión
**THE SYSTEM SHALL** mantener el flujo actual existente de revisión docente previo al ingreso definitivo al banco de preguntas.

---

### 8. Reintento automático hasta completar cantidad requerida

**WHILE** la cantidad de preguntas válidas almacenadas sea menor a la cantidad originalmente solicitada
**THE SYSTEM SHALL** continuar solicitando automáticamente nuevas preguntas a NQ hasta completar el total requerido.


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
* Inserción automática de preguntas generadas en la tabla temporal de revisión docente (banco IA, previo a banco Lumeria).
* Validación de integridad y control de duplicados.

## FUERA (Explícito)

* Generación manual de preguntas por parte del usuario.
* Edición manual del contenido generado por NQ dentro de este flujo.
* Generación de preguntas contextualizadas o enriquecidas con contenido adicional.
* Gestión manual de aprobaciones antes de insertar preguntas en el banco.
