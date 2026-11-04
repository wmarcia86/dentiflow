# Requirements Document

## Introduction

DentiFlow es un sistema de gestión administrativa para un consultorio dental pequeño. Este feature resuelve uno de los problemas principales del consultorio: el **abandono de tratamientos multisesión** (sin porcentaje de abandono cuantificado en el problema original). El abandono ocurre cuando pacientes dejan incompletos tratamientos multisesión (endodoncias, ortodoncia, etc.) a menudo porque experimentan mejoría temporal y no entienden la importancia de completar todas las sesiones. Este feature "Gestión de Continuidad de Tratamientos Odontológicos" permite que la asistente del consultorio registre pacientes, programe citas, administre tratamientos de varias sesiones e identifique automóticamente a los pacientes que necesitan seguimiento para completar sus tratamientos. **Nota importante**: Este feature se enfoca en prevenir el abandono de tratamientos. La prevención de inasistencias a citas (pacientes que no llegan a citas programadas) es responsabilidad de SPEC-002 (Gestión de Confirmación de Citas).

El sistema está diseñado para ser utilizado principalmente por la asistente del consultorio, quien gestiona todo el flujo operativo, y consultado por la doctora para revisar información y reportes.


## Assumptions and Dependencies

### Authentication Prerequisites

This feature assumes the following authentication and authorization infrastructure is already in place:

1. **Supabase Auth Configured**: Email/password authentication is configured and operational
2. **Test Users Available**: Two test users exist for development and testing:
   - sistente@dentiflow.com (rol: asistente) - Full read/write access
   - doctora@dentiflow.com (rol: doctora) - Read-only access to queries and reports
3. **usuarios Table**: A usuarios table exists with the following structure:
   - id (UUID, references auth.users)
   - 
ombre_completo (TEXT)
   - correo_electronico (TEXT)
   - 
ol (TEXT: 'doctora' or 'asistente')
   - ctivo (BOOLEAN)
   - echa_creacion (TIMESTAMPTZ)
   - echa_actualizacion (TIMESTAMPTZ)
4. **RLS Policies Active**: Row Level Security policies are implemented on all tables to enforce role-based access control
5. **Protected Routes**: All application routes require valid authentication before accessing patient or treatment data
6. **Session Management**: Supabase handles session tokens, refresh tokens, and session expiration (default 8 hours)

### Out of Scope for This Feature

The following authentication-related functionality is explicitly excluded from this feature:

- ? User registration flows and account creation UI
- ? Password reset and recovery workflows
- ? Multi-factor authentication (MFA)
- ? Session management UI (logout, session timeout warnings)
- ? User profile management (change password, update email)
- ? User administration (create/edit/delete users)
- ? Login page design and implementation

### External Dependencies

1. **Supabase Service**: This feature depends on Supabase being available and configured with:
   - PostgreSQL database (version 14+)
   - Supabase Auth service
   - Supabase Realtime (for future enhancements)
2. **Timezone Configuration**: Database and application must use America/Managua timezone
3. **Network Connectivity**: Internet connection required for Supabase API access

### Technical Constraints

1. All database object names (tables, columns, functions) must be in Spanish using snake_case
2. All timestamps must be stored as TIMESTAMPTZ with timezone America/Managua
3. All entity identifiers must use UUID v4 format
4. Frontend must be built with React + TypeScript + Tailwind CSS
5. Backend logic must use Supabase Edge Functions when server-side processing is required
## Glossary

- **Sistema**: El sistema DentiFlow de gestión administrativa dental
- **Asistente**: Usuario principal que gestiona el flujo operativo del consultorio (registro de pacientes, citas, tratamientos y seguimientos)
- **Doctora**: Usuario que consulta información y reportes sobre pacientes y tratamientos
- **Paciente**: Persona que recibe servicios odontológicos en el consultorio
- **Tratamiento**: Plan de atención odontológica que puede requerir una o múltiples sesiones (Endodoncia, Ortodoncia, Limpieza, Otros)
- **Sesion**: Cada una de las consultas individuales que conforman un tratamiento
- **Cita**: Agendamiento de una fecha y hora especófica para realizar una sesión de tratamiento
- **Seguimiento**: Intento de contacto documentado con un paciente para programar la continuación de su tratamiento
- **Alerta_de_Seguimiento**: Indicador visible que identifica tratamientos que requieren atención por tener sesiones pendientes sin citas programadas
- **Tratamiento_en_Riesgo**: Tratamiento con sesiones pendientes que no tiene una próxima cita programada
- **Base_de_Datos**: Sistema de almacenamiento PostgreSQL gestionado por Supabase
- **RLS**: Row Level Security, mecanismo de seguridad de PostgreSQL para controlar acceso a datos

## Requirements

### Requirement 1: Registro de Pacientes

**User Story:** As an Asistente, I want to register new patients with their basic contact information, so that I can track their treatments and communicate with them for follow-ups.

#### Acceptance Criteria

1. WHEN the Asistente submits a new patient form with nombre_completo, telefono y datos administrativos opcionales, THE Sistema SHALL create a new Paciente record in the Base_de_Datos
2. THE Sistema SHALL validate that telefono contains only numeric characters and has between 8 and 15 digits
3. THE Sistema SHALL validate that whatsapp contains only numeric characters and has between 8 and 15 digits
4. WHEN the Asistente searches for a patient by nombre or telefono, THE Sistema SHALL return all matching Paciente records within 500 milliseconds
5. THE Sistema SHALL require that nombre has at least 2 characters and consentimiento_comunicacion is explicitly set to true or false

### Requirement 2: Gestión Básica de Citas

**User Story:** As an Asistente, I want to schedule appointments for patient treatment sessions, so that I can organize the daily schedule and link appointments to specific treatment sessions.

#### Acceptance Criteria

1. WHEN the Asistente creates a Cita with fecha, hora, paciente_id, and estado, THE Sistema SHALL store the Cita in the Base_de_Datos with timezone America/Managua
2. THE Sistema SHALL validate that fecha is not in the past relative to the current date in timezone America/Managua
3. THE Sistema SHALL allow estado values of programada, completada, cancelada, or no_asistio
4. WHEN the Asistente associates a Cita with a Sesion, THE Sistema SHALL store the cita_id reference in the associated Sesion record (sesiones_tratamientos.cita_id); citas does not contain sesion_id
5. WHEN the Asistente queries Cita records for a specific date range, THE Sistema SHALL return all matching appointments sorted by fecha and hora

### Requirement 3: Registro de Tratamientos Multi-Sesión

**User Story:** As an Asistente, I want to register multi-session treatments with a defined number of planned sessions, so that I can track treatment progress and identify when treatments are incomplete.

#### Acceptance Criteria

1. WHEN the Asistente creates a Tratamiento with tipo, total_sesiones_planificadas, and paciente_id, THE Sistema SHALL create the Tratamiento record with estado set to pendiente_iniciar and sesiones_completadas set to 0
2. THE Sistema SHALL validate that total_sesiones_planificadas is a positive integer between 1 and 100
3. THE Sistema SHALL allow tipo values of Endodoncia, Ortodoncia, Limpieza, or Otros
4. THE Sistema SHALL allow estado values of pendiente_iniciar, en_curso, pendiente_seguimiento, finalizado, suspendido, or abandonado
5. WHEN the Asistente creates a Tratamiento, THE Sistema SHALL record fecha_creacion with the current timestamp in timezone America/Managua

### Requirement 4: Control de Sesiones de Tratamiento

**User Story:** As an Asistente, I want to register completed and pending treatment sessions, so that I can track detailed progress for each multi-session treatment.

#### Acceptance Criteria

1. WHEN the Asistente creates a Sesion with numero_sesion, tratamiento_id, and fecha_programada, THE Sistema SHALL create the Sesion record with estado set to pendiente
2. THE Sistema SHALL validate that numero_sesion is a positive integer and does not exceed the total_sesiones_planificadas of the associated Tratamiento
3. THE Sistema SHALL allow estado values of pendiente, completada, cancelada, or reprogramada
4. WHEN the Asistente marks a Sesion as completada by setting fecha_realizacion, THE Sistema SHALL increment sesiones_completadas in the associated Tratamiento by 1 and set fecha_ultima_sesion to fecha_realizacion
5. IF sesiones_completadas equals total_sesiones_planificadas after marking a Sesion as completada, THEN THE Sistema SHALL update the Tratamiento estado to finalizado and set fecha_finalizacion to the current timestamp in timezone America/Managua

### Requirement 5: Detección Automótica de Tratamientos en Riesgo

**User Story:** As an Asistente, I want the system to automatically identify treatments that need follow-up, so that I can proactively contact patients before they abandon their incomplete treatments.

#### Acceptance Criteria

1. WHEN a Tratamiento has sesiones_completadas less than total_sesiones_planificadas AND has no Cita with estado programada and fecha in the future, THE Sistema SHALL update the Tratamiento estado to pendiente_seguimiento
2. WHEN a Tratamiento estado is updated to pendiente_seguimiento, THE Sistema SHALL create an Alerta_de_Seguimiento visible to the Asistente
3. THE Sistema SHALL execute the automatic detection process within 2 seconds after any Sesion estado change or Cita creation or cancellation
4. WHEN the Asistente queries for Tratamiento records with estado pendiente_seguimiento, THE Sistema SHALL return all matching treatments sorted by fecha_ultima_sesion in ascending order
5. WHEN a new Cita with estado programada is created for a Tratamiento with estado pendiente_seguimiento, THE Sistema SHALL update the Tratamiento estado to en_curso and hide the associated Alerta_de_Seguimiento

### Requirement 6: Registro de Seguimientos de Contacto

**User Story:** As an Asistente, I want to document my contact attempts with patients who have incomplete treatments, so that I can maintain a history of follow-up efforts and coordinate next actions.

#### Acceptance Criteria

1. WHEN the Asistente creates a Seguimiento with tratamiento_id, fecha_seguimiento, medio_contacto, resultado, and observaciones, THE Sistema SHALL store the Seguimiento record in the Base_de_Datos
2. THE Sistema SHALL allow medio_contacto values of telefono, whatsapp, presencial, or correo
3. THE Sistema SHALL allow resultado values of contacto_exitoso, sin_respuesta, numero_invalido, or paciente_rechaza
4. THE Sistema SHALL record fecha_seguimiento with the current timestamp in timezone America/Managua
5. WHEN the Asistente queries Seguimiento records for a specific Tratamiento, THE Sistema SHALL return all Seguimiento records sorted by fecha_seguimiento in descending order

### Requirement 7: Validación de Integridad de Datos

**User Story:** As a Doctora, I want the system to maintain data integrity across all entities, so that I can trust the accuracy of treatment progress reports and patient information.

#### Acceptance Criteria

1. WHEN the Asistente attempts to create a Tratamiento with a paciente_id that does not exist in the Base_de_Datos, THE Sistema SHALL reject the operation and return an error message
2. WHEN the Asistente attempts to create a Sesion with a tratamiento_id that does not exist in the Base_de_Datos, THE Sistema SHALL reject the operation and return an error message
3. WHEN the Asistente attempts to create a Cita with a paciente_id that does not exist in the Base_de_Datos, THE Sistema SHALL reject the operation and return an error message
4. WHEN the Asistente attempts to delete a Paciente that has associated Tratamiento records, THE Sistema SHALL reject the operation and return an error message indicating existing dependencies
5. THE Sistema SHALL enforce that sesiones_completadas never exceeds total_sesiones_planificadas for any Tratamiento record

### Requirement 8: Seguridad y Control de Acceso

**User Story:** As a Doctora, I want the system to enforce role-based access controls, so that only authorized users can modify patient and treatment data.

#### Acceptance Criteria

1. THE Sistema SHALL use RLS policies in the Base_de_Datos to restrict data access based on user authentication
2. WHEN an Asistente is authenticated, THE Sistema SHALL grant read and write access to Paciente, Tratamiento, Sesion, Cita, and Seguimiento records
3. WHEN a Doctora is authenticated, THE Sistema SHALL grant read-only access to Paciente, Tratamiento, Sesion, Cita, and Seguimiento records
4. WHEN a user is not authenticated, THE Sistema SHALL deny all access to Paciente, Tratamiento, Sesion, Cita, and Seguimiento records
5. THE Sistema SHALL registrar el usuario responsable y la fecha de las operaciones relevantes de seguimiento y citas; las operaciones sensibles de confirmación se auditarán en SPEC-002. Los instantes se almacenarán como TIMESTAMPTZ y se mostrarán en America/Managua

### Requirement 9: Consultas y Reportes Básicos

**User Story:** As a Doctora, I want to query treatment status and patient information, para revisar el avance y la continuidad administrativa de los tratamientos.

#### Acceptance Criteria

1. WHEN the Doctora or Asistente queries for all Tratamiento records with estado pendiente_seguimiento, THE Sistema SHALL return results within 1 second
2. WHEN the Doctora or Asistente queries for all Sesion records associated with a specific Tratamiento, THE Sistema SHALL return results sorted by numero_sesion in ascending order
3. WHEN the Doctora or Asistente queries for all Cita records for a specific date, THE Sistema SHALL return results sorted by fecha_hora_inicio in ascending order
4. WHEN the Doctora or Asistente queries for a specific Paciente by paciente_id, THE Sistema SHALL return the Paciente record along with a count of total Tratamiento records associated with that patient
5. WHEN the Doctora or Asistente queries for all Seguimiento records for a specific Paciente, THE Sistema SHALL return results grouped by Tratamiento and sorted by fecha_seguimiento in descending order

### Requirement 10: Actualización de Estados de Tratamiento

**User Story:** As an Asistente, I want to manually update treatment status when circumstances require it, so that I can mark treatments as suspended or abandoned based on patient decisions.

#### Acceptance Criteria

1. WHEN the Asistente updates a Tratamiento estado to suspendido, THE Sistema SHALL record the current timestamp as fecha_suspension in timezone America/Managua and hide any associated Alerta_de_Seguimiento
2. WHEN the Asistente updates a Tratamiento estado to abandonado, THE Sistema SHALL record the current timestamp as fecha_abandono in timezone America/Managua and hide any associated Alerta_de_Seguimiento
3. WHEN the Asistente updates a Tratamiento estado from suspendido or abandonado to en_curso, THE Sistema SHALL clear fecha_suspension or fecha_abandono respectively
4. THE Sistema SHALL prevent updating a Tratamiento estado to finalizado if sesiones_completadas is less than total_sesiones_planificadas
5. WHEN the Asistente updates a Tratamiento estado, THE Sistema SHALL validate that the new estado is one of the allowed values: pendiente_iniciar, en_curso, pendiente_seguimiento, finalizado, suspendido, or abandonado

