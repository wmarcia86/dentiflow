---
inclusion: auto
name: dentiflow-context
description: Contexto general del proyecto DentiFlow - Sistema de gestión para consultorio dental
---

# Contexto del Proyecto: DentiFlow

## Descripción General

**DentiFlow** es una aplicación web para gestión administrativa de un consultorio dental pequeño dirigido por la Dra. Laura, quien trabaja con una asistente.

## Problemas que Resuelve

1. **Inasistencias a Citas**: Aproximadamente el 30% de los pacientes no asisten a sus citas, ocasionando espacios desaprovechados.
2. **Abandono de Tratamientos**: Pacientes dejan incompletos tratamientos multi-sesión (endodoncia, ortodoncia) cuando mejoran temporalmente.
3. **Carga Operativa**: La asistente dedica horas diarias enviando recordatorios manuales por WhatsApp.

## Usuarios del Sistema

### Doctora (Dra. Laura)
- Consulta información de pacientes y tratamientos
- Revisa reportes y métricas
- Configura reglas de automatización
- Administra usuarios

### Asistente
- Registra pacientes y citas
- Gestiona tratamientos y sesiones
- Realiza seguimiento a pacientes
- Registra intentos de contacto
- Consulta alertas y tareas pendientes
- **SPEC-002**: Gestiona confirmación de citas (NEW)
- **SPEC-002**: Registra gestiones de citas (NEW)

## Stack Tecnológico

### Frontend
- **React** con TypeScript
- **Tailwind CSS** para estilos
- Diseño responsive (desktop, tablet, móvil)

### Backend y Persistencia
- **Supabase** (PostgreSQL + Auth + Edge Functions)
- **Row Level Security (RLS)** para protección de datos
- Autenticación basada en roles

### Integraciones Externas (Fases posteriores)
- Google Calendar API (OAuth 2.0)
- WhatsApp Business Cloud API

## Convenciones de Base de Datos

**IMPORTANTE**: Todos los nombres de tablas, columnas, funciones y objetos de base de datos deben estar en **español**.

### Nomenclatura
- **Formato**: \snake_case\
- **Tablas**: Nombres en plural (ej: \pacientes\, \citas\, \	ratamientos\)
- **Columnas**: Descriptivas en español (ej: \
ombre_completo\, \echa_creacion\)
- **Claves primarias**: \id\ (tipo UUID)
- **Claves foráneas**: \{tabla_singular}_id\ (ej: \paciente_id\, \	ratamiento_id\)

### Tipos de Datos Estándar
- **Identificadores**: \UUID\ (con \gen_random_uuid()\)
- **Fechas con hora**: \TIMESTAMPTZ\ (con zona horaria)
- **Fechas sin hora**: \DATE\
- **Booleanos**: \BOOLEAN\
- **Textos**: \TEXT\ (preferir sobre VARCHAR)
- **Estados/enums**: \TEXT\ con restricción \CHECK\ o tipo \ENUM\
- **Configuraciones flexibles**: \JSONB\

### Campos Obligatorios
Todas las tablas deben incluir:
\\\sql
created_at TIMESTAMPTZ DEFAULT NOW(),
updated_at TIMESTAMPTZ DEFAULT NOW()
\\\

### Triggers Automáticos
Implementar trigger para actualizar \updated_at\:
\\\sql
CREATE TRIGGER set_updated_at
BEFORE UPDATE ON nombre_tabla
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
\\\

### Políticas RLS
- Aplicar políticas de seguridad a nivel de fila
- Los usuarios solo acceden a datos de su consultorio
- Roles: \doctora\, \sistente\

## Zona Horaria

**Default**: \America/Managua\ (Nicaragua)

Siempre usar \TIMESTAMPTZ\ para almacenar fechas/horas con zona horaria.

## Modelo de Datos

### SPEC-001: Gestión de Continuidad de Tratamientos

**Tablas principales:**
- \pacientes\ - Información de contacto
- \citas\ - Citas programadas
- \	ratamientos\ - Tratamientos multi-sesión
- \sesiones_tratamientos\ - Sesiones individuales
- \seguimientos_pacientes\ - Registro de intentos de contacto

### SPEC-002: Gestión de Confirmación de Citas (NEW)

#### Tabla Nueva: \gestiones_citas\

Auditoría completa de intentos de confirmación de citas.

\\\sql
CREATE TABLE gestiones_citas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Relaciones
  cita_id UUID NOT NULL REFERENCES citas(id) ON DELETE CASCADE,
  paciente_id UUID NOT NULL REFERENCES pacientes(id) ON DELETE CASCADE,
  
  -- Datos de gestión
  fecha_gestion TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  medio_contacto TEXT NOT NULL CHECK (medio_contacto IN ('telefono', 'whatsapp', 'presencial', 'correo')),
  resultado TEXT NOT NULL CHECK (resultado IN ('contacto_exitoso', 'sin_respuesta', 'numero_invalido', 'rechaza', 'reagenda', 'cancela')),
  observaciones TEXT,
  
  -- Control
  retry_count INTEGER NOT NULL DEFAULT 0,
  
  -- Timestamps
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Índices
CREATE INDEX idx_gestiones_citas_cita ON gestiones_citas(cita_id);
CREATE INDEX idx_gestiones_citas_paciente ON gestiones_citas(paciente_id);
CREATE INDEX idx_gestiones_citas_fecha ON gestiones_citas(fecha_gestion);
CREATE INDEX idx_gestiones_citas_resultado ON gestiones_citas(resultado);

-- Trigger
CREATE TRIGGER set_updated_at_gestiones_citas
BEFORE UPDATE ON gestiones_citas
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();

-- RLS Policy: Asistente puede leer/escribir todas
CREATE POLICY asistente_gestiones_all ON gestiones_citas
  FOR ALL
  USING (auth.jwt() ->> 'rol' = 'asistente')
  WITH CHECK (auth.jwt() ->> 'rol' = 'asistente');

-- RLS Policy: Doctora solo puede leer
CREATE POLICY doctora_gestiones_read ON gestiones_citas
  FOR SELECT
  USING (auth.jwt() ->> 'rol' = 'doctora');
\\\

#### Extensión de Tabla: \citas\

Agregar campos de confirmación a tabla \citas\:

\\\sql
ALTER TABLE citas ADD COLUMN (
  necesita_confirmacion BOOLEAN DEFAULT TRUE,
  confirmacion_estado TEXT DEFAULT 'pendiente' CHECK (confirmacion_estado IN ('pendiente', 'confirmada', 'cancelada', 'reprogramada')),
  fecha_confirmacion TIMESTAMPTZ,
  fecha_programada_original TIMESTAMPTZ,
  motivo_cancelacion TEXT,
  retry_count INTEGER DEFAULT 0
);

-- Índices
CREATE INDEX idx_citas_confirmacion_estado ON citas(confirmacion_estado);
CREATE INDEX idx_citas_necesita_confirmacion ON citas(necesita_confirmacion) WHERE necesita_confirmacion = TRUE;

-- Constraint: fecha de confirmación no puede ser antes de la fecha programada
ALTER TABLE citas ADD CONSTRAINT fecha_confirmacion_gte_programada
  CHECK (fecha_confirmacion IS NULL OR fecha_confirmacion >= fecha_hora_inicio);
\\\

### Estados de Confirmación de Citas

Máquina de estados para confirmación:

\\\
Estado Inicial: pendiente
  ├─ Asistente intenta contactar
  │  ├─ Paciente confirma → confirmada
  │  ├─ Paciente cancela → cancelada (se registra motivo)
  │  ├─ Paciente reagenda → reprogramada (nueva fecha)
  │  └─ Sin respuesta → sigue pendiente (retry_count ++)
  │
  ├─ Cita por suceder → alerta urgente
  └─ Cita pasada → se marca como no_asistio en tabla citas

Estados finales:
- confirmada: Paciente confirmó asistencia
- cancelada: Paciente canceló cita
- reprogramada: Paciente la movió a otra fecha
\\\

### Auditoría de Gestiones

Cada intento de confirmación se registra en \gestiones_citas\:
- **Fecha del intento**: \echa_gestion\
- **Medio utilizado**: \medio_contacto\ (telefono, whatsapp, presencial, correo)
- **Resultado**: \esultado\ (contacto_exitoso, sin_respuesta, numero_invalido, rechaza, reagenda, cancela)
- **Observaciones**: \observaciones\ (notas de la asistente)
- **Contador de reintentos**: \etry_count\

Ejemplo de auditoría:
\\\
Cita: Juan Pérez - 2024-11-15 10:00
├─ 2024-11-13 09:00 - WhatsApp - sin_respuesta - retry_count=1
├─ 2024-11-13 14:30 - Teléfono - contacto_exitoso - "Confirmó asistencia"
└─ confirmacion_estado = confirmada
\\\

## Diseño de Interfaz

### Paleta de Colores
- **Principal**: Azul suave / azul turquesa (#3B82F6, #06B6D4)
- **Secundario**: Blanco (#FFFFFF)
- **Fondos**: Gris muy claro (#F9FAFB, #F3F4F6)
- **Estados**:
  - Confirmado: Verde (#10B981)
  - Pendiente: Amarillo/Naranja (#F59E0B)
  - Cancelado: Rojo (#EF4444)
  - Alerta: Rojo intenso (#DC2626)

### Estilo Visual
- Moderno, profesional, limpio y minimalista
- Iconografía sencilla y consistente
- Tarjetas informativas (cards) para indicadores
- Tablas limpias y fáciles de consultar
- Formularios con validación clara

### Idioma
**Español** en todos los textos, etiquetas, botones, notificaciones y mensajes.

## Restricciones y Alcance

### Dentro del Alcance
- ✅ Gestión administrativa de pacientes (datos de contacto)
- ✅ Programación y seguimiento de citas
- ✅ Control de tratamientos multi-sesión
- ✅ Alertas y seguimiento de pacientes
- ✅ **SPEC-002**: Gestión de confirmación de citas (manual)
- ✅ **SPEC-002**: Auditoría de intentos de contacto
- ✅ Automatización de recordatorios
- ✅ Reportes e indicadores

### Fuera del Alcance
- ❌ **NO** expedientes clínicos completos
- ❌ **NO** diagnósticos médicos
- ❌ **NO** antecedentes médicos detallados
- ❌ **NO** prescripciones o recetas
- ❌ **NO** facturación o pagos
- ❌ **NO** inventario de medicamentos
- ❌ **NO** imágenes médicas (radiografías)
- ❌ **NO** Confirmación automática (solo manual en SPEC-002)
- ❌ **NO** Envío automático de WhatsApp (solo registro de intentos)

**Razón**: DentiFlow es una herramienta **administrativa**, no un sistema de historia clínica electrónica.

## Prioridades de Implementación

### Fase 1: Funcionalidad Principal (MVP) - SPEC-001
1. Autenticación y roles
2. CRUD de Pacientes
3. CRUD de Citas
4. Gestión de Tratamientos multi-sesión
5. Control de Sesiones
6. Sistema de Alertas de seguimiento
7. Estimación: 50 horas

### Fase 2: Gestión de Confirmación - SPEC-002
1. Base de datos: Tabla \gestiones_citas\ + extensiones a \citas\
2. API: CRUD de gestiones, queries de confirmaciones pendientes
3. Frontend: Página de confirmaciones, historial de gestiones
4. Testing: Property tests de integridad de confirmaciones
5. Estimación: 38 horas
6. **Dependencia**: Requiere SPEC-001 completada

### Fase 3: Automatización WhatsApp (Futura)
- Integración con WhatsApp Business
- Recordatorios automáticos (48h, 24h antes)
- Confirmaciones automáticas por WhatsApp

### Fase 4: Integraciones y Reportes (Futura)
- Sincronización con Google Calendar
- Dashboard con KPIs
- Reportes avanzados
- Exportación de datos

## Principios de Desarrollo

### Seguridad
- Autenticación obligatoria para todas las rutas
- Autorización basada en roles (RBAC)
- RLS en todas las tablas sensibles
- Tokens y credenciales solo en backend
- Validación de datos en frontend y backend

### Usabilidad
- La asistente debe poder realizar operaciones en ≤3 clics
- Feedback claro en todas las acciones
- Estados de carga visibles
- Mensajes de error descriptivos
- Formularios con validación en tiempo real

### Mantenibilidad
- Componentes reutilizables
- Separación de concerns (UI / Lógica / Datos)
- Código documentado
- Nombres descriptivos en español
- Funciones pequeñas y cohesivas

### Performance
- Cargas iniciales < 2 segundos
- Consultas optimizadas con índices
- Paginación en listados grandes
- Lazy loading de componentes

## Datos de Prueba (Seed Data)

Para facilitar el desarrollo, crear datos de prueba representativos:

### Usuarios
- Dra. Laura (rol: doctora)
- María González (rol: asistente)

### Pacientes (mínimo 10)
Ejemplos:
- Juan Pérez (tratamiento activo, cita pendiente de confirmar)
- Ana Martínez (tratamiento pendiente seguimiento, cita confirmada)
- Carlos López (sin tratamientos, cita cancelada)
- María Rodríguez (tratamiento finalizado, cita reprogramada)

### Tratamientos
Tipos comunes:
- Endodoncia (2-3 sesiones)
- Ortodoncia (12-24 sesiones)
- Limpieza dental (1 sesión)
- Implante dental (3-4 sesiones)
- Corona/Puente (2-3 sesiones)

### Estados de Tratamiento
- \pendiente_iniciar\: Registrado sin sesiones
- \en_curso\: Con sesiones en progreso
- \pendiente_seguimiento\: Sin próxima cita programada
- \inalizado\: Todas las sesiones completadas
- \suspendido\: Pausado por solicitud del paciente
- \bandonado\: Sin contacto >30 días

### Citas de Prueba (SPEC-002)
- Cita confirmada: Juan Pérez - 2024-11-15
- Cita pendiente: Ana Martínez - 2024-11-16
- Cita cancelada: Carlos López - 2024-11-17
- Cita reprogramada: María Rodríguez - 2024-11-18

## Referencias y Documentación

### Supabase
- Auth: https://supabase.com/docs/guides/auth
- RLS: https://supabase.com/docs/guides/auth/row-level-security
- Edge Functions: https://supabase.com/docs/guides/functions

### Tailwind CSS
- Documentación: https://tailwindcss.com/docs
- UI Components: Usar Headless UI o Radix UI

### TypeScript
- Tipar todas las entidades de base de datos
- Usar interfaces para props de componentes
- Evitar \ny\, preferir \unknown\ si es necesario

## Notas Adicionales

Este contexto se aplica a todas las interacciones con agentes de IA trabajando en DentiFlow. Cualquier código generado debe respetar estas convenciones y restricciones.

**SPEC-002 Update**: Este documento ahora incluye especificaciones completas para la tabla \gestiones_citas\ y extensiones a \citas\, proporcionando todo el contexto necesario para implementar la gestión de confirmación de citas.
