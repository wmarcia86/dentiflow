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

### Database Tasks (Wave 1)

#### 1. [ ] TASK-DB-001: Create pacientes table
- [x] 1. Tabla con campos id, nombre_completo, telefono, correo, whatsapp, estado
- [x] 2. Constraints: telefono 8-15 dígitos, nombre >= 2 chars
- [x] 3. Índices en nombre, telefono, estado
- [x] 4. Default estado='activo', fecha_registro=hoy

#### 2. [ ] TASK-DB-002: Create citas table
- [x] 1. FK a pacientes, constraint fecha_hora_fin > fecha_hora_inicio
- [x] 2. Estados: programada, completada, cancelada, no_asistio
- [x] 3. Índices en paciente_id, fecha_hora_inicio, estado

#### 3. [ ] TASK-DB-003: Create tratamientos table
- [x] 1. FK a pacientes, constraint sesiones_completadas <= total_sesiones
- [x] 2. 6 estados válidos: pendiente_iniciar, en_curso, pendiente_seguimiento, finalizado, suspendido, abandonado
- [x] 3. Índices en paciente_id, estado, fecha_ultima_sesion

#### 4. [ ] TASK-DB-004: Create sesiones_tratamientos table
- [x] 1. FKs a tratamientos y citas (opcional)
- [x] 2. Constraint numero_sesion <= total_sesiones
- [x] 3. Estados: pendiente, completada, cancelada, reprogramada

#### 5. [ ] TASK-DB-005: Create seguimientos_pacientes table
- [x] 1. FKs a pacientes y tratamientos
- [x] 2. medio_contacto: telefono, whatsapp, presencial, correo
- [x] 3. resultado: exitoso, sin_respuesta, numero_invalido, rechaza

#### 6. [ ] TASK-DB-006: Create triggers
- [x] 1. Función update_updated_at_column() para todas las tablas
- [x] 2. Aplicar a 5 tablas principales

#### 7. [ ] TASK-DB-007: Create RLS policies
- [x] 1. Habilitar RLS en 5 tablas
- [x] 2. Asistente: read/write, Doctora: read-only
- [x] 3. Bloquear no autenticados

### API Tasks (Wave 2)

#### 8. [ ] TASK-API-001: CRUD Pacientes
- [x] 1. CREATE, READ, UPDATE, DELETE
- [x] 2. SEARCH by nombre/telefono < 500ms
- [x] 3. Validar constraints

#### 9. [ ] TASK-API-002: CRUD Citas
- [x] 1. CREATE (validar fecha no pasada)
- [x] 2. READ, UPDATE, DELETE (soft delete)

#### 10. [ ] TASK-API-003: CRUD Tratamientos
- [x] 1. CREATE (validar 1-100 sesiones)
- [x] 2. READ (incluir sesiones relacionadas)
- [x] 3. Query pendientes_seguimiento < 1s

#### 11. [ ] TASK-API-004: CRUD Sesiones
- [x] 1. CREATE, UPDATE, READ
- [x] 2. Marcar completada: incrementar progreso
- [x] 3. Auto-finalizar si última sesion

#### 12. [ ] TASK-API-005: Detect Pending Treatments
- [x] 1. Query: sesiones < total AND sin cita futura
- [x] 2. Retornar con info de paciente y progreso

#### 13. [ ] TASK-API-006: CRUD Seguimientos
- [x] 1. CREATE, READ, UPDATE
- [x] 2. Validar medio y resultado

### Frontend Tasks (Wave 3)

#### 14. [ ] TASK-UI-001: Common Components
- [x] 1. Header, Sidebar, Badge, Spinner, ErrorMessage
- [x] 2. Colores DentiFlow (azul #3B82F6, turquesa #06B6D4)

#### 15. [ ] TASK-UI-002: Patients Page
- [x] 1. CRUD con tabla listable
- [x] 2. Búsqueda real-time
- [x] 3. Modal de nuevo paciente

#### 16. [ ] TASK-UI-003: Treatments Page
- [x] 1. Lista de tratamientos con progreso visual
- [x] 2. Crear tratamiento con validación
- [x] 3. Ver detalles y cambiar estado

#### 17. [ ] TASK-UI-004: Sessions Page
- [x] 1. Marcar sesiones completadas
- [x] 2. Validar no exceder total
- [x] 3. Mostrar progreso automático

#### 18. [ ] TASK-UI-005: Alerts Panel
- [x] 1. Mostrar tratamientos pendientes de seguimiento
- [x] 2. Auto-refresh cada 5 minutos
- [x] 3. Acceso rápido a formulario seguimiento

#### 19. [ ] TASK-UI-006: Follow-ups Page
- [x] 1. Formulario crear seguimiento
- [x] 2. Histórico con filtros
- [x] 3. Registrar medio, resultado, observaciones

### Testing Tasks (Wave 4)

#### 20. [ ] TASK-TEST-001: Unit Tests
- [x] 1. Validación teléfono
- [x] 2. Cálculo progreso
- [x] 3. Detección pendientes
- [x] 4. Transiciones de estado

#### 21. [ ] TASK-TEST-002: Integration Tests
- [x] 1. CRUD pacientes
- [x] 2. Flujo tratamiento completo
- [x] 3. Detección automática alertas
- [x] 4. Registro seguimiento

#### 22. [ ] TASK-TEST-003: E2E Tests
- [x] 1. Flujo completo: login → paciente → tratamiento → alertas → seguimiento

### Documentation Tasks (Wave 5)

#### 23. [ ] TASK-DOC-001: User Documentation
- [x] 1. Guías para cada funcionalidad principal
- [x] 2. Solución de problemas

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

