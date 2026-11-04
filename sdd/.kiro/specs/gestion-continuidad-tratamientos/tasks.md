# Implementation Plan: Gestión de Continuidad de Tratamientos Odontológicos

## Overview

Plan de implementación del feature "Gestión de Continuidad de Tratamientos Odontológicos" con 23 tareas organizadas en 5 waves.

**Total**: 50 horas | **Duración**: 2 semanas | **Equipo**: 4 personas

## Task Dependency Graph

```json
{
  "waves": [
    {
      "wave": 1,
      "title": "Database",
      "tasks": ["TASK-DB-001", "TASK-DB-002", "TASK-DB-003", "TASK-DB-004", "TASK-DB-005", "TASK-DB-006", "TASK-DB-007"],
      "hours": 8
    },
    {
      "wave": 2,
      "title": "API",
      "tasks": ["TASK-API-001", "TASK-API-002", "TASK-API-003", "TASK-API-004", "TASK-API-005", "TASK-API-006"],
      "hours": 12
    },
    {
      "wave": 3,
      "title": "Frontend",
      "tasks": ["TASK-UI-001", "TASK-UI-002", "TASK-UI-003", "TASK-UI-004", "TASK-UI-005", "TASK-UI-006"],
      "hours": 20
    },
    {
      "wave": 4,
      "title": "Testing",
      "tasks": ["TASK-TEST-001", "TASK-TEST-002", "TASK-TEST-003"],
      "hours": 8
    },
    {
      "wave": 5,
      "title": "Documentation",
      "tasks": ["TASK-DOC-001"],
      "hours": 2
    }
  ]
}
```

## Tasks

### Prerrequisitos de seguridad e integridad (Wave 1)

- [ ] Crear `public.usuarios` (1:1 con `auth.users`) antes de SPEC-002; permitir SELECT a ambos roles, sin INSERT/UPDATE/DELETE desde el cliente. Verificar carga segura de perfiles mediante backend administrativo.
- [ ] Aplicar políticas RLS separadas por operación (SELECT / INSERT / UPDATE); no usar `FOR ALL` ni conceder DELETE directo en las cinco tablas de SPEC-001.
- [ ] Aplicar la extensión `btree_gist` y la exclusión `citas_sin_solapamiento_paciente` después de validar datos existentes. Probar dos inserciones concurrentes para el mismo paciente.
- [ ] Probar autorización para asistente, doctora, usuario sin rol y anónimo con JWT reales.

### Validaciones adicionales obligatorias (Wave 1–4)

- [ ] Registrar campos `fecha_suspension` y `fecha_abandono` y comprobar las transiciones administrativas.
- [ ] Crear la restricción UNIQUE(tratamiento_id, numero_sesion), validar rango y paciente de una cita asociada a una sesión.
- [ ] Implementar la recalculación **transaccional** de sesiones completadas; eliminar incrementos del frontend.
- [ ] Crear la vista `v_tratamientos_sin_cita_futura` con `security_invoker=true` a partir de la consulta de diseño y probar RLS.
- [ ] Comprobar la exclusión de citas con estados programada, completada y no_asistio; cancelada no bloquea el horario.
- [ ] Ejecutar pruebas de concurrencia, reintentos y consistencia de alertas por **tratamiento**, no solo por paciente.

### Database Tasks (Wave 1)

#### 1. [ ] TASK-DB-001: Create pacientes table
- [ ] 1. Tabla con campos id, nombre_completo, telefono, correo, whatsapp, estado
- [ ] 2. Constraints: telefono 8-15 dígitos, nombre >= 2 chars
- [ ] 3. Índices en nombre, telefono, estado
- [ ] 4. Default estado='activo', fecha_registro=hoy

#### 2. [ ] TASK-DB-002: Create citas table
- [ ] 1. FK a pacientes, constraint fecha_hora_fin > fecha_hora_inicio
- [ ] 2. Estados: programada, completada, cancelada, no_asistio
- [ ] 3. Índices en paciente_id, fecha_hora_inicio, estado

#### 3. [ ] TASK-DB-003: Create tratamientos table
- [ ] 1. FK a pacientes, constraint sesiones_completadas <= total_sesiones
- [ ] 2. 6 estados válidos: pendiente_iniciar, en_curso, pendiente_seguimiento, finalizado, suspendido, abandonado
- [ ] 3. Índices en paciente_id, estado, fecha_ultima_sesion

#### 4. [ ] TASK-DB-004: Create sesiones_tratamientos table
- [ ] 1. FKs a tratamientos y citas (opcional)
- [ ] 2. Constraint numero_sesion <= total_sesiones
- [ ] 3. Estados: pendiente, completada, cancelada, reprogramada

#### 5. [ ] TASK-DB-005: Create seguimientos_pacientes table
- [ ] 1. FKs a pacientes y tratamientos
- [ ] 2. medio_contacto: telefono, whatsapp, presencial, correo
- [ ] 3. resultado: contacto_exitoso, sin_respuesta, numero_invalido, paciente_rechaza

#### 6. [ ] TASK-DB-006: Create triggers
- [ ] 1. Función update_updated_at_column() para todas las tablas
- [ ] 2. Aplicar a 5 tablas principales

#### 7. [ ] TASK-DB-007: Create RLS policies
- [ ] 1. Habilitar RLS en 5 tablas
- [ ] 2. Asistente: SELECT/INSERT/UPDATE; Doctora: solo SELECT. DELETE denegado.
- [ ] 3. Bloquear no autenticados

### API Tasks (Wave 2)

#### 8. [ ] TASK-API-001: CRUD Pacientes
- [ ] 1. CREATE, READ, UPDATE; sin DELETE físico directo
- [ ] 2. SEARCH by nombre/telefono < 500ms
- [ ] 3. Validar constraints

#### 9. [ ] TASK-API-002: CRUD Citas
- [ ] 1. CREATE (validar fecha no pasada)
- [ ] 2. READ, UPDATE y cancelación administrativa (sin DELETE físico)

#### 10. [ ] TASK-API-003: CRUD Tratamientos
- [ ] 1. CREATE (validar 1-100 sesiones)
- [ ] 2. READ (incluir sesiones relacionadas)
- [ ] 3. Query pendientes_seguimiento < 1s

#### 11. [ ] TASK-API-004: CRUD Sesiones
- [ ] 1. CREATE, UPDATE, READ
- [ ] 2. Marcar completada: recalcular progreso automáticamente desde la base de datos; no incrementar en cliente
- [ ] 3. Auto-finalizar si última sesion

#### 12. [ ] TASK-API-005: Detect Pending Treatments
- [ ] 1. Query: sesiones < total AND sin cita futura
- [ ] 2. Retornar con info de paciente y progreso

#### 13. [ ] TASK-API-006: CRUD Seguimientos
- [ ] 1. CREATE, READ, UPDATE
- [ ] 2. Validar medio y resultado

### Frontend Tasks (Wave 3)

#### 14. [ ] TASK-UI-001: Common Components
- [ ] 1. Header, Sidebar, Badge, Spinner, ErrorMessage
- [ ] 2. Colores DentiFlow (azul #3B82F6, turquesa #06B6D4)

#### 15. [ ] TASK-UI-002: Patients Page
- [ ] 1. CRUD con tabla listable
- [ ] 2. Búsqueda real-time
- [ ] 3. Modal de nuevo paciente

#### 16. [ ] TASK-UI-003: Treatments Page
- [ ] 1. Lista de tratamientos con progreso visual
- [ ] 2. Crear tratamiento con validación
- [ ] 3. Ver detalles y cambiar estado

#### 17. [ ] TASK-UI-004: Sessions Page
- [ ] 1. Marcar sesiones completadas
- [ ] 2. Validar no exceder total
- [ ] 3. Mostrar progreso automático

#### 18. [ ] TASK-UI-005: Alerts Panel
- [ ] 1. Mostrar tratamientos pendientes de seguimiento
- [ ] 2. Auto-refresh cada 5 minutos
- [ ] 3. Acceso rápido a formulario seguimiento

#### 19. [ ] TASK-UI-006: Follow-ups Page
- [ ] 1. Formulario crear seguimiento
- [ ] 2. Histórico con filtros
- [ ] 3. Registrar medio, resultado, observaciones

### Testing Tasks (Wave 4)

#### 20. [ ] TASK-TEST-001: Unit Tests
- [ ] 1. Validación teléfono
- [ ] 2. Cálculo progreso
- [ ] 3. Detección pendientes
- [ ] 4. Transiciones de estado

#### 21. [ ] TASK-TEST-002: Integration Tests
- [ ] 1. CRUD pacientes
- [ ] 2. Flujo tratamiento completo
- [ ] 3. Detección automática alertas
- [ ] 4. Registro seguimiento

#### 22. [ ] TASK-TEST-003: E2E Tests
- [ ] 1. Flujo completo: login → paciente → tratamiento → alertas → seguimiento

### Documentation Tasks (Wave 5)

#### 23. [ ] TASK-DOC-001: User Documentation
- [ ] 1. Guías para cada funcionalidad principal
- [ ] 2. Solución de problemas

## Notes

### Key Assumptions

- Supabase Auth ya configurado con usuarios de prueba
- Timezone: America/Managua
- React 18+, TypeScript, Tailwind CSS
- Supabase Client SDK

### Risks

- Database schema changes → Backup antes de Wave 1
- RLS policies security → Review antes de deploy
- Performance → Test con 10k+ records
- Frontend complexity → Build components library first

### Success Criteria

- Unit tests > 80% coverage
- E2E flujo completo exitoso
- Alertas < 1s response
- UI < 2s load
- RLS enforce role access
- Zero defects en testing

