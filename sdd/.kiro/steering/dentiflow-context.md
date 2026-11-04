---
inclusion: auto
name: dentiflow-context
description: Contexto general del proyecto DentiFlow - Sistema de gesti?n para consultorio dental
---

# Contexto del Proyecto: DentiFlow

## Descripci?n General

**DentiFlow** es una aplicaci?n web para gesti?n administrativa de un consultorio dental peque?o dirigido por la Dra. Laura, quien trabaja con una asistente.

## Problemas del Consultorio

El consultorio dental enfrenta tres problemas operativos cr?ticos que afectan la viabilidad del negocio y la continuidad de la atenci?n:

### 1. Inasistencias a Citas (~30%)
**Definición**: Pacientes no llegan a citas ya programadas sin avisar o avisar con poca anticipaci?n.
**Impacto**: 
- Espacios de agenda desaprovechados
- Ingresos perdidos (~2-3 citas no asistidas por semana)
- Doctora con tiempo inactivo

**Causa ra?z**: Los pacientes no confirman su intenci?n de asistir antes de la fecha programada, por lo que la asistente no sabe si pueden ir o no.

**Soluci?n (SPEC-002)**:
- Gesti?n de confirmaci?n de citas: Asistente contacta a los pacientes 2-3 d?as antes para confirmar asistencia
- Registro de cancelaciones y reprogramaciones
- Historial completo de intentos de contacto

---

### 2. Abandono de Tratamientos
**Definición**: Pacientes dejan incompletos tratamientos multi-sesi?n (endodoncia, ortodoncia, implantes) sin informar y no responden a contactos posteriores.

**Impacto**:
- Dientes sin tratamiento completo • problemas de salud bucal
- Tratamientos que nunca se cierran en registros
- Pacientes que podr?an pagos futuros nunca retornan
- Ingresos perdidos (algunos tratamientos abandonados a mitad del proceso)

**Causa ra?z**: 
- Mejor?a temporal: Despu?s de 1-2 sesiones, los s?ntomas (dolor) disminuyen • paciente asume que est• "curado"
- Falta de comunicación: No se explica claramente por qu• es cr?tico completar todas las sesiones
- Falta de seguimiento: Doctora y asistente no tienen alertas sobre pacientes que no han programado siguiente sesi?n

**Soluci?n (SPEC-001)**:
- Control automatizado de sesiones: Sistema detecta cuando un tratamiento tiene sesiones pendientes sin citas futuras
- Alertas de seguimiento: Asistente recibe notificaciones sobre tratamientos "en riesgo"
- Documentaci?n de intentos de contacto: Registro completo de seguimientos

---

### 3. Carga Operativa Manual de la Asistente
**Definición**: La asistente dedica m?ltiples horas diarias (2-3h) enviando recordatorios manuales por WhatsApp sin herramienta sistematizada.

**Impacto**:
- Esfuerzo manual ineficiente
- Falta de registro sistem?tico (no se sabe qu• se contact• y qu• no)
- Recordatorios inconsistentes o incompletitos
- Tiempo que podr?a dedicarse a otras tareas administrativas

**Causa ra?z**: No existe sistema para:
- Identificar qu• citas necesitan confirmaci?n
- Registrar intentos de contacto
- Documentar respuestas de pacientes
- Generar alertas de seguimiento

**Soluci?n (SPEC-001 + SPEC-002)**:
- Vista centralizada de citas programadas
- Identificaci?n autom?tica de citas pendientes de confirmaci?n
- Registro estructurado de gestiones de citas
- Alertas de seguimiento autom?ticas
- Hist?ricos completos para auditor?a

---

### Diferenciaci?n: Inasistencias vs Abandono

Es cr?tico entender la diferencia, porque cada problema requiere una soluci?n distinta:

| Aspecto | **Inasistencias** | **Abandono** |
|--------|-------------------|-------------|
| **Cu?ndo ocurre** | El d?a de la cita programada: Paciente no llega | Entre sesiones: Paciente no programa siguientes sesiones |
| **S?ntoma** | Cita con estado "no_asistio" o ausencia sin aviso | Tratamiento con sesiones incompletas sin citas futuras |
| **Causa** | Olvido, cambio de planes del paciente, emergencias | Creencia de "estar curado", falta de comprensi?n de importancia |
| **Responsabilidad** | **SPEC-002**: Gesti?n de Confirmación de Citas | **SPEC-001**: Gesti?n de Continuidad de Tratamientos |
| **Soluci?n** | Confirmar cita con paciente 2-3 d?as antes | Detectar sesiones incompletas y hacer seguimiento |
| **Resultado esperado** | Reducir inasistencias; línea base aproximada: 30 % de citas | Mejorar continuidad; abandono sin línea base cuantificada |

Ambos problemas se atacan simult?neamente pero con estrategias, herramientas y equipo responsable distintos.

---

## Problemas que Resuelve (Referencia - Expandido Arriba)

1. **Inasistencias a Citas**: Ver "Inasistencias a Citas (~30%)" arriba
2. **Abandono de Tratamientos**: Ver "Abandono de Tratamientos" arriba
3. **Carga Operativa**: Ver "Carga Operativa Manual de la Asistente" arriba

## Usuarios del Sistema

### Doctora (Dra. Laura)
- Consulta informaci?n de pacientes y tratamientos
- Revisa reportes y m?tricas
- Configura reglas de automatizaci?n
- Administra usuarios

### Asistente
- Registra pacientes y citas
- Gestiona tratamientos y sesiones
- Realiza seguimiento a pacientes
- Registra intentos de contacto
- Consulta alertas y tareas pendientes
- **SPEC-002**: Gestiona confirmaci?n de citas (NEW)
- **SPEC-002**: Registra gestiones de citas (NEW)

## Stack Tecnol?gico

### Frontend
- **React** con TypeScript
- **Tailwind CSS** para estilos
- Diseño responsive (desktop, tablet, m?vil)

### Backend y Persistencia
- **Supabase** (PostgreSQL + Auth + Edge Functions)
- **Row Level Security (RLS)** para protecci?n de datos
- Autenticaci?n basada en roles

### Integraciones Externas (Fases posteriores)
- Google Calendar API (OAuth 2.0)
- WhatsApp Business Cloud API

## Convenciones de Base de Datos

**IMPORTANTE**: Todos los nombres de tablas, columnas, funciones y objetos de base de datos deben estar en **espa?ol**.

### Nomenclatura
- **Formato**: `snake_case`
- **Tablas**: Nombres en plural (ej.: `pacientes`, `citas`, `tratamientos`)
- **Columnas**: Descriptivas en español (ej.: `nombre_completo`, `fecha_creacion`)
ombre_completo\, \echa_creacion\)
- **Claves primarias**: `id` de tipo UUID
- **Claves foráneas**: `{tabla_singular}_id` (ej.: `paciente_id`, `tratamiento_id`)

### Tipos de Datos Est?ndar
- **Identificadores**: `UUID` con `gen_random_uuid()`
- **Fechas con hora**: `TIMESTAMPTZ` (con zona horaria)
- **Fechas sin hora**: `DATE`
- **Booleanos**: `BOOLEAN`
- **Textos**: `TEXT` (preferir sobre VARCHAR)
- **Estados/enums**: `TEXT` con restricción `CHECK` o tipo `ENUM`
- **Configuraciones flexibles**: `JSONB`

### Campos Obligatorios
Todas las tablas deben incluir:
```sql
created_at TIMESTAMPTZ DEFAULT NOW(),
updated_at TIMESTAMPTZ DEFAULT NOW()
```
### Triggers Autom?ticos
Implementar trigger para actualizar `updated_at`:
```sql
CREATE TRIGGER set_updated_at
BEFORE UPDATE ON nombre_tabla
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```
### Pol?ticas RLS
- Aplicar pol?ticas de seguridad a nivel de fila
- Los usuarios autorizados acceden a datos del único consultorio contemplado en esta versión
- Roles: `doctora`, `asistente`

## Autenticación y autorización (fuente confiable de roles)

Los roles se almacenan en **`auth.users.raw_app_meta_data`**, gestionado únicamente mediante backend administrativo y privilegios de servidor. **Nunca usar `raw_user_meta_data`/`user_metadata` para autorización** porque el usuario puede modificarlo. Los valores autorizados son `asistente` y `doctora`.

Ejemplo de claim confiable incluido en el JWT:

```json
{ "app_metadata": { "rol": "asistente" } }
```

Todas las políticas RLS deben comprobar:

```sql
(auth.jwt() -> 'app_metadata' ->> 'rol') = 'asistente'
-- o para consultas de doctora:
(auth.jwt() -> 'app_metadata' ->> 'rol') = 'doctora'
```

La interfaz puede utilizar `user.app_metadata?.rol` para ocultar acciones, pero **solo RLS/backend impone permisos**. Las funciones RPC obtienen la identidad mediante `auth.uid()`, nunca confían en IDs/roles arbitrarios enviados desde el cliente. No exponer la clave `service_role` al navegador. Requiere renovar el JWT tras cambiar roles.

**Alcance de seguridad:** las políticas aquí definidas separan roles dentro de un solo consultorio. Si se amplía a múltiples consultorios, añadir `consultorio_id` y aislamiento por organización como requisito independiente; no afirmar aislamiento multi-tenant en esta versión.

---


### Perfiles de usuario y concurrencia de la agenda

SPEC-001 debe crear `public.usuarios` como tabla de perfiles vinculada a `auth.users(id)`; tendrá `nombre_completo`, `correo_electronico`, `activo`, `created_at` y `updated_at`. La lectura por `asistente` y `doctora` se protege mediante RLS, mientras la creación y edición de perfiles queda reservada al backend administrativo. **No usar el perfil como fuente de autorización**: los roles de acceso provienen exclusivamente de `app_metadata` protegido.

Las tablas de SPEC-001 conceden operaciones SELECT a ambos roles e INSERT/UPDATE a la asistente mediante políticas separadas; no crean `FOR ALL` ni permiten DELETE desde el frontend. La tabla `citas` aplica una exclusión `citas_sin_solapamiento_paciente` para impedir citas programadas superpuestas del mismo paciente. El límite de reprogramación de SPEC-002 es **30 días** y cada operación genera **una única** gestión histórica.

## Zona Horaria

**Zona horaria de visualización**: `America/Managua` (Nicaragua)

Siempre usar `TIMESTAMPTZ` para almacenar fechas/horas con zona horaria.

## Modelo de Datos

### SPEC-001: Gestión de Continuidad de Tratamientos

**Tablas principales:**
- `pacientes` - Información de contacto
- `citas` - Citas programadas
- `tratamientos` - Tratamientos de múltiples sesiones
- `sesiones_tratamientos` - Sesiones individuales
- `seguimientos_pacientes` - Registro de intentos de contacto

### SPEC-002: Gestión de Confirmación de Citas (NEW)

#### Tabla nueva: `gestiones_citas`

La estructura **única y canónica** se comparte con SPEC-002/design.md. `paciente_id` se obtiene a través de `citas.paciente_id`; `retry_count` existe en `citas` y no en `gestiones_citas`. Las gestiones son inmutables para el usuario, sin eliminación en cascada; conservar auditoría.

```sql
CREATE TABLE gestiones_citas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Relación con cita
  cita_id UUID NOT NULL REFERENCES citas(id) ON DELETE RESTRICT,
  
  -- Tipo de gestión
  tipo_gestion TEXT NOT NULL 
    CONSTRAINT gestiones_citas_tipo_check 
    CHECK (tipo_gestion IN ('contacto', 'confirmacion', 'cancelacion', 'reprogramacion')),
  
  -- Respuesta del paciente
  respuesta_paciente TEXT NOT NULL 
    CONSTRAINT gestiones_citas_respuesta_check 
    CHECK (respuesta_paciente IN (
      'confirmada', 'cancelada', 'reprogramada', 'sin_respuesta', 
      'numero_invalido', 'paciente_rechaza'
    )),
  
  -- Detalles de la gestión
  motivo TEXT,
  observaciones TEXT,
  medio_contacto TEXT CHECK (medio_contacto IS NULL OR medio_contacto IN
    ('telefono','presencial','correo','otro','whatsapp_manual')),
  CONSTRAINT gestiones_tipo_respuesta_compatible CHECK (
    (tipo_gestion = 'contacto' AND respuesta_paciente IN
      ('sin_respuesta','numero_invalido','paciente_rechaza')) OR
    (tipo_gestion = 'confirmacion' AND respuesta_paciente = 'confirmada') OR
    (tipo_gestion = 'cancelacion' AND respuesta_paciente = 'cancelada') OR
    (tipo_gestion = 'reprogramacion' AND respuesta_paciente = 'reprogramada')
  ),
  usuario_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE RESTRICT,
  
  -- Para reprogramaciones
  nueva_fecha_hora TIMESTAMPTZ,
  nueva_cita_id UUID REFERENCES citas(id) ON DELETE SET NULL,
  
  -- Auditoría
  fecha_gestion TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Índices
CREATE INDEX idx_gestiones_citas_cita_id ON gestiones_citas(cita_id);
CREATE INDEX idx_gestiones_citas_tipo_gestion ON gestiones_citas(tipo_gestion);
CREATE INDEX idx_gestiones_citas_usuario_id ON gestiones_citas(usuario_id);
CREATE INDEX idx_gestiones_citas_respuesta ON gestiones_citas(respuesta_paciente);
CREATE INDEX idx_gestiones_citas_fecha ON gestiones_citas(fecha_gestion);

ALTER TABLE gestiones_citas ENABLE ROW LEVEL SECURITY;
-- Aplicar exclusivamente las políticas granulares de SPEC-002/design.md.
-- No crear políticas FOR ALL; podrían abrir UPDATE/DELETE y comprometer la auditoría.
-- SELECT: asistente y doctora. INSERT/UPDATE/DELETE: prohibidos desde el cliente.
-- Toda nueva gestión debe generarse por RPC transaccional con autorización segura.
```

SPEC-002/design.md contiene las políticas canónicas de solo lectura y las RPC transaccionales de escritura. No agregar políticas adicionales que permitan editar o insertar el historial desde el frontend. La creación de gestiones se vincula al usuario autenticado.

#### Extensión de `citas`

Utilizar exclusivamente las columnas y restricciones definidas en **SPEC-002/design.md → Extensiones a Tabla `citas`**; no mantener una segunda migración SQL divergente aquí. Incluye `necesita_confirmacion`, `confirmacion_estado`, fechas de confirmación/cancelación/reprogramación y `retry_count`. Una confirmación puede registrarse antes de la cita.

### Estados de Confirmación de Citas

M?quina de estados para confirmaci?n:

```
Estado Inicial: pendiente
  +- Asistente intenta contactar
  •  +- Paciente confirma • confirmada
  •  +- Paciente cancela • cancelada (se registra motivo)
  •  +- Paciente reagenda • reprogramada (nueva fecha)
  •  +- Sin respuesta • sigue pendiente (retry_count ++)
  ?
  +- Cita por suceder • alerta urgente
  +- Cita pasada • se marca como no_asistio en tabla citas

Estados finales:
- confirmada: Paciente confirm• asistencia
- cancelada: Paciente cancel• cita
- reprogramada: Paciente la movi• a otra fecha
```
### Auditor?a de Gestiones

Cada intento de confirmaci?n se registra en `gestiones_citas`:
- **Fecha del intento**: \echa_gestion\
- **Medio utilizado**: \medio_contacto\ (telefono, whatsapp, presencial, correo)
- **Resultado**: \
esultado\ (contacto_exitoso, sin_respuesta, numero_invalido, rechaza, reagenda, cancela)
- **Observaciones**: \observaciones\ (notas de la asistente)
- **Contador de reintentos**: \
etry_count\

Ejemplo de auditor?a:
```
Cita: Juan P?rez - 2024-11-15 10:00
+- 2024-11-13 09:00 - WhatsApp - sin_respuesta - retry_count=1
+- 2024-11-13 14:30 - Tel?fono - contacto_exitoso - "Confirm• asistencia"
+- confirmacion_estado = confirmada
```
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
- Iconograf?a sencilla y consistente
- Tarjetas informativas (cards) para indicadores
- Tablas limpias y f?ciles de consultar
- Formularios con validaci?n clara

### Idioma
**Espa?ol** en todos los textos, etiquetas, botones, notificaciones y mensajes.

## Restricciones y Alcance

### Dentro del Alcance
- • Gesti?n administrativa de pacientes (datos de contacto)
- • Programaci?n y seguimiento de citas
- • Control de tratamientos multi-sesi?n
- • Alertas y seguimiento de pacientes
- • **SPEC-002**: Gesti?n de confirmaci?n de citas (manual)
- • **SPEC-002**: Auditor?a de intentos de contacto
- • Automatización de recordatorios
- • Reportes e indicadores

### Fuera del Alcance
- • **NO** expedientes cl?nicos completos
- • **NO** diagn?sticos m?dicos
- • **NO** antecedentes m?dicos detallados
- • **NO** prescripciones o recetas
- • **NO** facturaci?n o pagos
- • **NO** inventario de medicamentos
- • **NO** im?genes m?dicas (radiograf?as)
- • **NO** Confirmación autom?tica (solo manual en SPEC-002)
- • **NO** Envío autom?tico de WhatsApp (solo registro de intentos)

**Raz?n**: DentiFlow es una herramienta **administrativa**, no un sistema de historia cl?nica electr?nica.

## Prioridades de Implementaci?n

### Fase 1: Funcionalidad Principal (MVP) - SPEC-001
1. Autenticaci?n y roles
2. CRUD de Pacientes
3. CRUD de Citas
4. Gesti?n de Tratamientos multi-sesi?n
5. Control de Sesiones
6. Sistema de Alertas de seguimiento
7. Estimaci?n: 50 horas

### Fase 2: Gesti?n de Confirmación - SPEC-002
1. Base de datos: Tabla `gestiones_citas` + extensiones a `citas`
2. API: CRUD de gestiones, queries de confirmaciones pendientes
3. Frontend: P?gina de confirmaciones, historial de gestiones
4. Testing: Property tests de integridad de confirmaciones
5. Estimaci?n: 38 horas
6. **Dependencia**: Requiere SPEC-001 completada

### Fase 3: Automatización WhatsApp (Futura)
- Integraci?n con WhatsApp Business
- Recordatorios autom?ticos (48h, 24h antes)
- Confirmaciones autom?ticas por WhatsApp

### Fase 4: Integraciones y Reportes (Futura)
- Sincronización con Google Calendar
- Dashboard con KPIs
- Reportes avanzados
- Exportaci?n de datos

## Principios de Desarrollo

### Seguridad
- Autenticaci?n obligatoria para todas las rutas
- Autorizaci?n basada en roles (RBAC)
- RLS en todas las tablas sensibles
- Tokens y credenciales solo en backend
- Validaci?n de datos en frontend y backend

### Usabilidad
- La asistente debe poder realizar operaciones en =3 clics
- Feedback claro en todas las acciones
- Estados de carga visibles
- Mensajes de error descriptivos
- Formularios con validaci?n en tiempo real

### Mantenibilidad
- Componentes reutilizables
- Separaci?n de concerns (UI / L?gica / Datos)
- C?digo documentado
- Nombres descriptivos en espa?ol
- Funciones peque?as y cohesivas

### Performance
- Cargas iniciales < 2 segundos
- Consultas optimizadas con ?ndices
- Paginaci?n en listados grandes
- Lazy loading de componentes

## Datos de Prueba (Seed Data)

Para facilitar el desarrollo, crear datos de prueba representativos:

### Usuarios
- Dra. Laura (rol: doctora)
- Mar?a Gonz?lez (rol: asistente)

### Pacientes (m?nimo 10)
Ejemplos:
- Juan P?rez (tratamiento activo, cita pendiente de confirmar)
- Ana Mart?nez (tratamiento pendiente seguimiento, cita confirmada)
- Carlos L?pez (sin tratamientos, cita cancelada)
- Mar?a Rodr?guez (tratamiento finalizado, cita reprogramada)

### Tratamientos
Tipos comunes:
- Endodoncia (2-3 sesiones)
- Ortodoncia (12-24 sesiones)
- Limpieza dental (1 sesi?n)
- Implante dental (3-4 sesiones)
- Corona/Puente (2-3 sesiones)

### Estados de Tratamiento
- \pendiente_iniciar\: Registrado sin sesiones
- \en_curso\: Con sesiones en progreso
- \pendiente_seguimiento\: Sin pr?xima cita programada
- \inalizado\: Todas las sesiones completadas
- \suspendido\: Pausado por solicitud del paciente
- \bandonado\: Sin contacto >30 d?as

### Citas de Prueba (SPEC-002)
- Cita confirmada: Juan P?rez - 2024-11-15
- Cita pendiente: Ana Mart?nez - 2024-11-16
- Cita cancelada: Carlos L?pez - 2024-11-17
- Cita reprogramada: Mar?a Rodr?guez - 2024-11-18

## Referencias y Documentaci?n

### Supabase
- Auth: https://supabase.com/docs/guides/auth
- RLS: https://supabase.com/docs/guides/auth/row-level-security
- Edge Functions: https://supabase.com/docs/guides/functions

### Tailwind CSS
- Documentaci?n: https://tailwindcss.com/docs
- UI Components: Usar Headless UI o Radix UI

### TypeScript
- Tipar todas las entidades de base de datos
- Usar interfaces para props de componentes
- Evitar \ny\, preferir \unknown\ si es necesario

## Notas Adicionales

Este contexto se aplica a todas las interacciones con agentes de IA trabajando en DentiFlow. Cualquier c?digo generado debe respetar estas convenciones y restricciones.

**SPEC-002 Update**: Este documento ahora incluye especificaciones completas para la tabla `gestiones_citas` y extensiones a `citas`, proporcionando todo el contexto necesario para implementar la gesti?n de confirmaci?n de citas.


