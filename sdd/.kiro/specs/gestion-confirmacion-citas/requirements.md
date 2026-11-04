# Requirements Document

## Introduction

### Gestión de Confirmación de Citas Odontológicas

DentiFlow es un sistema de gestión administrativa para un consultorio dental pequeño. El problema que resuelve este feature es el alto porcentaje de inasistencias a las citas programadas, debido a que no se confirman las citas con los pacientes antes de la fecha programada. Este feature "Gestión de Confirmación de Citas Odontológicas" permite que la asistente del consultorio identifique las citas pendientes de confirmación, registre las respuestas de los pacientes, gestione cancelaciones y reprogramaciones, y documente el historial de gestiones realizadas.

El sistema está diseñado para ser utilizado principalmente por la asistente del consultorio, quien realiza todas las acciones operativas de confirmación, cancelación y reprogramación de citas, y consultado por la doctora para revisar el estado de las citas confirmadas y el historial de gestiones.

**Dependencia**: Este feature depende completamente de SPEC-001 (Gestión de Continuidad de Tratamientos Odontológicos). Reutiliza las tablas pacientes, citas y tratamientos, y extiende la funcionalidad con gestiones_citas para documentar el proceso de confirmación.

---

## Assumptions and Dependencies

### Dependencias de SPEC-001

This feature assumes the following from SPEC-001 are already implemented and operational:

1. **Tabla pacientes**: Existe con estructura completa (id, nombre_completo, telefono, telefono_whatsapp, correo_electronico, acepta_comunicaciones_whatsapp, etc.)
2. **Tabla citas**: Existe con campos (id, paciente_id, fecha_hora_inicio, fecha_hora_fin, estado: programada/completada/cancelada/no_asistio)
3. **Tabla tratamientos**: Existe con campos (id, paciente_id, tipo, estado, sesiones_completadas, total_sesiones_planificadas)
4. **Supabase Auth**: Configurado con usuarios asistente y doctora
5. **RLS Policies**: Implementadas en tablas existentes para control de acceso por rol
6. **Timezone**: America/Managua configurada en base de datos y aplicación

### Authentication Prerequisites

This feature assumes the following authentication and authorization infrastructure:

1. **Usuarios Autenticados**: Sesiones activas de Asistente y Doctora
2. **Rol Asistente**: Usuario con permisos para crear, leer, actualizar gestiones_citas
3. **Rol Doctora**: Usuario con permisos para leer gestiones_citas (consulta solamente)

### Out of Scope for This Feature

The following functionality is explicitly excluded:

- ❌ Integración con Google Calendar o sincronización de calendarios
- ❌ Envío automático de WhatsApp, SMS o correos electrónicos
- ❌ Integración con plataformas de mensajería externas
- ❌ Recordatorios automáticos a pacientes
- ❌ Notificaciones push o alertas en tiempo real
- ❌ Facturación o gestión de pagos
- ❌ Expedientes clínicos o historial médico
- ❌ Reportes avanzados o analytics
- ❌ Gestión de recursos (salas, equipos, personal)
- ❌ Integración con sistemas de terceros

### External Dependencies

1. **Supabase Service**: Base de datos PostgreSQL operacional en Supabase
2. **Timezone Configuration**: Database y aplicación deben usar America/Managua
3. **Network Connectivity**: Conexión a internet requerida para acceso a Supabase

### Technical Constraints

1. Todos los nombres de objetos de base de datos (tablas, columnas, funciones) deben estar en español usando snake_case
2. Todas las marcas de tiempo deben almacenarse como TIMESTAMPTZ con zona horaria America/Managua
3. Todos los identificadores de entidades deben usar formato UUID v4
4. El frontend debe construirse con React + TypeScript + Tailwind CSS
5. Las convenciones de nombres deben ser consistentes con SPEC-001

---

## Glossary

- **Sistema**: El sistema DentiFlow de gestión administrativa dental
- **Asistente**: Usuario principal que gestiona confirmaciones, cancelaciones y reprogramaciones de citas
- **Doctora**: Usuario que consulta información sobre citas confirmadas e historial de gestiones
- **Paciente**: Persona que recibe servicios odontológicos y tiene citas programadas
- **Cita**: Agendamiento de una fecha y hora específica para una consulta odontológica
- **Cita_Pendiente_de_Confirmacion**: Cita con estado 'programada' que aún no ha sido confirmada por el paciente
- **Gestion_de_Cita**: Intento de contacto documentado con un paciente para confirmar, cancelar o reprogramar una cita
- **Confirmacion**: Respuesta positiva del paciente indicando que asistirá a la cita
- **Cancelacion**: Solicitud del paciente para cancelar la cita programada
- **Reprogramacion**: Cambio de fecha y/o hora de una cita programada
- **Tipo_de_Gestion**: Categoría de la gestión realizada (confirmacion, cancelacion, reprogramacion)
- **Respuesta_Paciente**: Reacción del paciente ante el intento de contacto (confirmada, cancelada, reprogramada, sin_respuesta, numero_invalido, paciente_rechaza)
- **Historial_de_Gestiones**: Registro completo de todas las gestiones realizadas en una cita
- **Base_de_Datos**: Sistema de almacenamiento PostgreSQL gestionado por Supabase
- **RLS**: Row Level Security, mecanismo de seguridad de PostgreSQL para control de acceso a datos

---

## Requirements

### Requirement 1: Visualización de Citas Programadas

**User Story:** As an Asistente, I want to view all scheduled appointments with their associated patient information, so that I can have a complete overview of the daily and weekly schedule.

#### Acceptance Criteria

1. WHEN the Asistente navigates to the appointment view, THE Sistema SHALL display all Cita records with estado 'programada', 'completada', 'cancelada', or 'no_asistio'
2. WHEN the Asistente filters by a specific date range, THE Sistema SHALL return only Cita records within that range sorted by fecha_hora_inicio in ascending order
3. THE Sistema SHALL display for each Cita: paciente nombre, fecha_hora, estado, and link to associated Tratamiento if it exists
4. WHEN the Asistente searches by patient name, THE Sistema SHALL return all Cita records for that patient within 500 milliseconds
5. WHEN a Cita is clicked, THE Sistema SHALL display detailed information including paciente contact information (telefono, telefono_whatsapp, correo_electronico)

### Requirement 2: Identificación Automática de Citas Pendientes de Confirmación

**User Story:** As an Asistente, I want the system to automatically identify appointments that need confirmation, so that I can prioritize which appointments to contact patients about.

#### Acceptance Criteria

1. WHEN a Cita is created with estado 'programada', THE Sistema SHALL automatically mark it as necesita_confirmacion = true
2. WHEN the Asistente views the appointment list, THE Sistema SHALL display a visual indicator (badge, color, icon) for all Cita records with necesita_confirmacion = true
3. WHEN a Cita fecha_hora_inicio is within 48 hours from current time AND necesita_confirmacion = true, THE Sistema SHALL prioritize it in the default view by sorting to the top
4. THE Sistema SHALL calculate the count of Cita records with necesita_confirmacion = true and display it in a summary widget
5. WHEN the Asistente filters by 'necesita_confirmacion', THE Sistema SHALL return only Cita records where necesita_confirmacion = true sorted by fecha_hora_inicio in ascending order

### Requirement 3: Registro de Confirmaciones de Citas

**User Story:** As an Asistente, I want to record when a patient confirms their appointment, so that I can track confirmation status and reduce no-shows.

#### Acceptance Criteria

1. WHEN the Asistente creates a Gestion_de_Cita with tipo_gestion = 'confirmacion' and respuesta_paciente = 'confirmada', THE Sistema SHALL create a gestion record linked to the Cita
2. THE Sistema SHALL update the associated Cita necesita_confirmacion = false when confirmación is recorded
3. WHEN a confirmation is recorded, THE Sistema SHALL record fecha_gestion with current timestamp in timezone America/Managua
4. WHEN the Asistente provides observaciones during confirmation (e.g., "paciente confirmó vía llamada a las 3pm"), THE Sistema SHALL store the complete text
5. WHEN a Cita has a Gestion_de_Cita with respuesta_paciente = 'confirmada', THEN the Cita estado SHALL NOT automatically change to 'cancelada' or 'no_asistio' without explicit action

### Requirement 4: Gestión de Cancelaciones de Citas

**User Story:** As an Asistente, I want to record when a patient cancels their appointment, so that I can update the schedule and reallocate time to other patients.

#### Acceptance Criteria

1. WHEN the Asistente creates a Gestion_de_Cita with tipo_gestion = 'cancelacion' and respuesta_paciente = 'cancelada', THE Sistema SHALL create a gestion record linked to the Cita
2. WHEN a cancellation is recorded, THE Sistema SHALL update the associated Cita estado = 'cancelada' and fecha_cancelacion = current timestamp
3. WHEN the Asistente provides motivo_cancelacion (e.g., "paciente enfermo"), THE Sistema SHALL store the reason for audit purposes
4. WHEN a cancellation is recorded, THE Sistema SHALL record the date and time of cancellation in timezone America/Managua
5. WHEN a Cita is cancelled, the Tratamiento associated with the Paciente SHALL remain unchanged (no automatic status updates)

### Requirement 5: Reprogramación de Citas

**User Story:** As an Asistente, I want to reschedule appointments to different dates and times, so that I can accommodate patient availability and maintain continuity of treatment sessions.

#### Acceptance Criteria

1. WHEN the Asistente creates a Gestion_de_Cita with tipo_gestion = 'reprogramacion', THE Sistema SHALL allow specifying nueva_fecha_hora for the appointment
2. WHEN a reprogramacion is recorded, THE Sistema SHALL create a new Cita record with the nueva_fecha_hora and update the previous Cita estado = 'cancelada'
3. WHEN a reprogramacion occurs, THE Sistema SHALL create two Gestion_de_Cita records: one for the cancellation of the original and one for the creation of the new appointment
4. WHEN the Asistente enters motivo_reprogramacion (e.g., "paciente no puede asistir el día indicado"), THE Sistema SHALL store the reason for documentation
5. WHEN reprogramming, THE Sistema SHALL validate that nueva_fecha_hora is not in the past relative to the current date in timezone America/Managua

### Requirement 6: Historial Completo de Gestiones de Cita

**User Story:** As an Asistente, I want to view the complete history of confirmations, cancellations, and reschedulings for each appointment, so that I can understand the interaction history with patients and make informed decisions.

#### Acceptance Criteria

1. WHEN the Asistente views historial for a Cita, THE Sistema SHALL display all Gestion_de_Cita records linked to that Cita sorted by fecha_gestion in descending order
2. FOR each Gestion_de_Cita, THE Sistema SHALL display: tipo_gestion, respuesta_paciente, fecha_gestion, observaciones, and usuario_id (who created the record)
3. WHEN the Asistente filters gestiones by tipo_gestion, THE Sistema SHALL return only records matching that type (confirmacion, cancelacion, reprogramacion)
4. WHEN the Asistente clicks on a Gestion_de_Cita record, THE Sistema SHALL display full details including motivo, observaciones, and complete audit trail
5. WHEN querying gestion history for a specific Cita, THE Sistema SHALL return results within 500 milliseconds

### Requirement 7: Estados de Confirmación de Cita

**User Story:** As an Asistente, I want to track different confirmation statuses for each appointment, so that I can distinguish between confirmed, cancelled, rescheduled, and unconfirmed appointments.

#### Acceptance Criteria

1. WHEN a Cita is created, THE Sistema SHALL set confirmacion_estado = 'pendiente' by default
2. WHEN a confirmation Gestion is recorded, THE Sistema SHALL update confirmacion_estado = 'confirmada' and registrar fecha_confirmacion
3. WHEN a cancellation Gestion is recorded, THE Sistema SHALL update confirmacion_estado = 'cancelada' and registrar fecha_cancelacion
4. WHEN a rescheduling Gestion is recorded, THE Sistema SHALL update confirmacion_estado = 'reprogramada' and create a new Cita with confirmacion_estado = 'pendiente'
5. THE Sistema SHALL allow querying Cita records by confirmacion_estado to generate reports on confirmation rates

### Requirement 8: Documentación de Contactos Fallidos

**User Story:** As an Asistente, I want to record when contact attempts fail (invalid numbers, no answer), so that I can track engagement issues and know which appointments need follow-up.

#### Acceptance Criteria

1. WHEN the Asistente creates a Gestion_de_Cita with respuesta_paciente = 'sin_respuesta', THE Sistema SHALL record the failed contact attempt
2. WHEN the Asistente creates a Gestion_de_Cita with respuesta_paciente = 'numero_invalido', THE Sistema SHALL mark the contact method as unreliable and suggest using alternative contact information
3. WHEN the Asistente creates a Gestion_de_Cita with respuesta_paciente = 'paciente_rechaza', THE Sistema SHALL record the explicit rejection with observaciones
4. WHEN multiple failed contact attempts occur (≥2) for the same Cita, THE Sistema SHALL display a warning indicator suggesting alternative contact methods
5. THE Sistema SHALL store retry_count for each Cita tracking the number of contact attempts made

### Requirement 9: Reportes de Confirmación de Citas

**User Story:** As a Doctora, I want to review confirmation statistics and reports, so that I can understand attendance patterns and identify process improvements.

#### Acceptance Criteria

1. WHEN the Doctora queries for confirmation statistics, THE Sistema SHALL display: total Cita records, confirmadas count, canceladas count, reprogramadas count, y pendientes count
2. WHEN the Doctora filters by date range, THE Sistema SHALL return statistics for Cita records within that range
3. WHEN the Doctora queries for confirmation rate (%), THE Sistema SHALL calculate: (confirmadas / total) * 100 with precision to 2 decimal places
4. WHEN the Doctora queries for cancellation reasons, THE Sistema SHALL group Gestion_de_Cita records by motivo_cancelacion and display frequency count
5. WHEN the Doctora exports report data, THE Sistema SHALL include: Cita details, Gestion history, confirmation status, and timestamps in CSV or PDF format

### Requirement 10: Validación e Integridad de Datos de Gestiones

**User Story:** As a Doctora, I want to ensure that all appointment confirmations, cancellations, and reschedulings are accurately recorded with complete audit trails, so that I can trust the reliability of the system.

#### Acceptance Criteria

1. WHEN the Asistente attempts to create a Gestion_de_Cita with a cita_id that does not exist, THE Sistema SHALL reject the operation and display an error message
2. WHEN the Asistente attempts to create a Gestion_de_Cita with tipo_gestion not in (confirmacion, cancelacion, reprogramacion), THE Sistema SHALL reject the operation
3. WHEN the Asistente attempts to create a Gestion_de_Cita with respuesta_paciente not in (confirmada, cancelada, reprogramada, sin_respuesta, numero_invalido, paciente_rechaza), THE Sistema SHALL reject the operation
4. THE Sistema SHALL enforce that every Gestion_de_Cita record includes usuario_id (who created it) and fecha_gestion (when it was created)
5. WHEN querying Gestion_de_Cita records, THE Sistema SHALL enforce RLS policies to ensure only authorized users can view and modify records

---

## Out of Scope

- User registration and account creation
- Manual calendar synchronization or export to Google Calendar
- Automatic SMS or WhatsApp integration
- Patient notification system
- Advanced business intelligence or predictive analytics
- Multi-language support beyond Spanish
- Mobile app for patients
- Video conferencing integration
- Electronic signature or consent management

</content>
