# Implementation Plan: Gestión de Confirmación de Citas Odontológicas

## Overview

Este plan de implementación describe todas las tareas necesarias para desarrollar el feature SPEC-002 "Gestión de Confirmación de Citas Odontológicas" en DentiFlow. 

**Enfoque**: Implementar funcionalidad backend (BD + APIs) en las primeras waves, seguido por componentes frontend, testing y documentación. Cada wave es independiente y puede ejecutarse en paralelo dentro de esa wave.

**Distribución horaria estimada**:
- Wave 0: Setup (2h)
- Wave 1: Database & Migrations (4h)
- Wave 2: Backend APIs (8h)
- Wave 3: Frontend Componentes (16h)
- Wave 4: Property Tests (6h)
- Wave 5: Documentación (2h)
- **Total: 38 horas**

---

## Tasks

### Phase 0: Setup y Verificación

- [ ] 0.1 Verificar conexión a Supabase y sincronizar con SPEC-001
  - Confirmar acceso a proyecto DentiFlow en Supabase
  - Verificar tablas pacientes, citas, tratamientos existen
  - Confirmar Supabase Auth con usuarios asistente@dentiflow.com y doctora@dentiflow.com
  - Crear archivo .env.local con credenciales VITE_SUPABASE_URL y VITE_SUPABASE_ANON_KEY
  - _Requirements: 1.1, 2.1_

- [ ] 0.2 Crear rama de trabajo y estructura de directorios
  - Git checkout -b feature/spec-002-confirmacion-citas
  - Crear directorios: src/features/confirmacion-citas/components, src/features/confirmacion-citas/hooks, src/features/confirmacion-citas/types
  - Crear archivo src/features/confirmacion-citas/types.ts para tipos TypeScript
  - Confirmar estructura con ls -la
  - _Requirements: Ninguno específico_

---

### Phase 1: Database & Migrations

- [ ] 1.1 Extender tabla `citas` con campos de confirmación
  - Crear migration: CREATE TABLE gestiones_citas (ver design.md)
  - Agregar columnas necesita_confirmacion, confirmacion_estado, fecha_confirmacion, fecha_cancelacion, retry_count a tabla citas
  - Crear índices: idx_citas_necesita_confirmacion, idx_citas_confirmacion_estado
  - Ejecutar migration en Supabase
  - Verificar con SELECT * FROM information_schema.columns WHERE table_name='citas'
  - _Requirements: 2.1, 7.1_

- [ ] 1.2 Crear tabla `gestiones_citas` con relaciones
  - Crear tabla gestiones_citas con campos: id, cita_id, tipo_gestion, respuesta_paciente, motivo, observaciones, usuario_id, nueva_fecha_hora, nueva_cita_id, fecha_gestion, created_at, updated_at
  - Agregar CHECK constraints para tipo_gestion y respuesta_paciente (ver design.md)
  - Crear índices: idx_gestiones_citas_cita_id, idx_gestiones_citas_tipo_gestion, idx_gestiones_citas_usuario_id, idx_gestiones_citas_respuesta, idx_gestiones_citas_fecha
  - Agregar FOREIGN KEY constraints: cita_id -> citas(id), usuario_id -> auth.users(id)
  - Ejecutar en Supabase SQL Editor
  - Verificar con SELECT * FROM gestiones_citas LIMIT 1
  - _Requirements: 3.1, 4.1, 6.1, 10.1_

- [ ] 1.3 Implementar RLS Policies para gestiones_citas
  - Crear policy: "Asistente puede ver todas las gestiones_citas" (SELECT para rol asistente)
  - Crear policy: "Asistente puede crear gestiones_citas" (INSERT para rol asistente, validando usuario_id = auth.uid())
  - Crear policy: "Asistente puede actualizar gestiones_citas creadas por ellos" (UPDATE para usuario_id = auth.uid())
  - Crear policy: "Doctora puede ver todas las gestiones_citas (lectura)" (SELECT para rol doctora, denegar INSERT/UPDATE/DELETE)
  - Crear policy para actualizar confirmacion_estado en citas (solo Asistente)
  - Habilitar RLS en tabla gestiones_citas: ALTER TABLE gestiones_citas ENABLE ROW LEVEL SECURITY
  - Verificar policies en Supabase con SELECT * FROM pg_policies WHERE tablename='gestiones_citas'
  - _Requirements: 8.1, 8.2, 8.3, 8.4_

- [ ] 1.4 Crear vistas SQL para queries principales
  - Crear vista v_citas_pendientes_confirmacion (Q1 del design)
  - Crear vista v_gestiones_por_cita (Q2 del design)
  - Crear vista v_estadisticas_confirmacion (Q3 del design)
  - Crear vista v_motivos_cancelacion (Q4 del design)
  - Crear vista v_citas_multiples_intentos_fallidos (Q5 del design)
  - Ejecutar en SQL Editor de Supabase
  - Verificar que cada vista retorna datos correctos
  - _Requirements: 9.1, 9.2, 9.3_

---

### Phase 2: Backend APIs (Edge Functions o Hooks)

- [ ] 2.1 Implementar función: crear Gestion_de_Cita (confirmación)
  - Crear Supabase Edge Function o hook TypeScript: createGestionConfirmacion
  - Input: { cita_id: UUID, observaciones: string, usuario_id: UUID }
  - Validar: cita_id existe, observaciones.length > 0
  - Lógica: INSERT INTO gestiones_citas con tipo_gestion='confirmacion', respuesta_paciente='confirmada'
  - Lógica: UPDATE citas SET necesita_confirmacion=false, confirmacion_estado='confirmada', fecha_confirmacion=NOW()
  - Output: { success: true, gestion_id: UUID } o { error: string }
  - Testear manualmente desde panel Supabase
  - _Requirements: 3.1, 3.2, 3.3_

- [ ] 2.2 Implementar función: crear Gestion_de_Cita (cancelación)
  - Crear función: createGestionCancelacion
  - Input: { cita_id: UUID, motivo: string, observaciones?: string, usuario_id: UUID }
  - Validar: cita_id existe, motivo no vacío
  - Lógica: INSERT INTO gestiones_citas con tipo_gestion='cancelacion', respuesta_paciente='cancelada'
  - Lógica: UPDATE citas SET estado='cancelada', confirmacion_estado='cancelada', fecha_cancelacion=NOW()
  - Lógica: UPDATE citas SET retry_count = retry_count + 1 si respuesta fue 'sin_respuesta'
  - Output: { success: true, gestion_id: UUID } o { error: string }
  - _Requirements: 4.1, 4.2, 4.3_

- [ ] 2.3 Implementar función: crear Gestion_de_Cita (reprogramación)
  - Crear función: createGestionReprogramacion
  - Input: { cita_id: UUID, nueva_fecha_hora: TIMESTAMPTZ, motivo: string, usuario_id: UUID }
  - Validar: cita_id existe, nueva_fecha_hora > NOW(), nueva_fecha_hora no en pasado
  - Lógica: Crear NEW Cita con estado='programada', confirmacion_estado='pendiente'
  - Lógica: UPDATE Cita original a estado='cancelada'
  - Lógica: INSERT INTO gestiones_citas (2 registros): uno para cancelación, uno para creación de nueva cita
  - Lógica: Actualizar nueva_cita_id en registro de gestión
  - Output: { success: true, nueva_cita_id: UUID } o { error: string }
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

- [ ] 2.4 Implementar función: obtener Citas Pendientes de Confirmación
  - Crear función: getCitasPendientes
  - Input: { filtro?: 'todas' | 'proximas_48h', ordenar?: 'fecha_asc' | 'fecha_desc' | 'reintentos', usuarioId: UUID }
  - Query: SELECT from v_citas_pendientes_confirmacion con filtros aplicados
  - Validar: usuario tiene rol 'asistente' o 'doctora' (lectura)
  - Implementar paginación: LIMIT 20 OFFSET {page*20}
  - Output: { citas: Cita[], total: number } o { error: string }
  - _Requirements: 2.1, 2.2, 2.3_

- [ ] 2.5 Implementar función: obtener Historial de Gestiones
  - Crear función: getHistorialGestiones
  - Input: { cita_id: UUID, usuarioId: UUID }
  - Query: SELECT from v_gestiones_por_cita (Q2)
  - Validar: usuario tiene permisos de lectura
  - Output: { gestiones: GestionCita[] } o { error: string }
  - Verificar que retorna todas las gestiones sin omisiones
  - _Requirements: 6.1, 6.2, 6.3, 6.4_

- [ ] 2.6 Implementar función: obtener Estadísticas de Confirmación
  - Crear función: getEstadisticasConfirmacion
  - Input: { fecha_inicio: DATE, fecha_fin: DATE, usuarioId: UUID }
  - Query: SELECT from v_estadisticas_confirmacion (Q3)
  - Calcular: total_citas, confirmadas, canceladas, reprogramadas, pendientes, tasa_confirmacion_pct
  - Output: { stats: EstadisticasConfirmacion } o { error: string }
  - _Requirements: 9.1, 9.2, 9.3_

- [ ] 2.7 Implementar función: obtener Motivos de Cancelación
  - Crear función: getMotivoCancelacion
  - Input: { fecha_inicio: DATE, fecha_fin: DATE, usuarioId: UUID }
  - Query: SELECT from v_motivos_cancelacion (Q4)
  - Output: { motivos: { motivo: string, frecuencia: number, porcentaje: number }[] } o { error: string }
  - _Requirements: 9.4_

- [ ] 2.8 Implementar validaciones en Edge Functions
  - Validar todos los campos requeridos no sean null/empty
  - Validar tipo_gestion está en ('confirmacion', 'cancelacion', 'reprogramacion')
  - Validar respuesta_paciente está en valores permitidos
  - Validar fechas no estén en el pasado
  - Validar referencias externas (cita_id, usuario_id, paciente_id) existen
  - Retornar errores descriptivos para cada validación fallida
  - _Requirements: 10.1, 10.2, 10.3_

---

### Phase 3: Frontend Components

- [ ] 3.1 Crear tipos TypeScript para Gestiones
  - Crear archivo src/features/confirmacion-citas/types.ts
  - Definir tipos: GestionCita, CrearGestionDTO, EstadisticasConfirmacion, ConfirmacionEstado
  - Definir enums: TipoGestion, RespuestaPaciente, ConfirmacionEstado
  - Exportar tipos en src/index.ts
  - Validar con TypeScript compiler (tsc --noEmit)
  - _Requirements: Ninguno específico_

- [ ] 3.2 Crear custom hook: useGestiones
  - Crear archivo src/features/confirmacion-citas/hooks/useGestiones.ts
  - Implementar: createGestionConfirmacion, createGestionCancelacion, createGestionReprogramacion
  - Implementar: getHistorialGestiones
  - Implementar manejo de errores y estados (loading, error, success)
  - Usar Supabase client para llamadas a Edge Functions
  - _Requirements: 3.1, 4.1, 5.1, 6.1_

- [ ] 3.3 Crear custom hook: useCitasPendientes
  - Crear archivo src/features/confirmacion-citas/hooks/useCitasPendientes.ts
  - Implementar: getCitasPendientes con filtros y paginación
  - Implementar: useEffect con fetch inicial
  - Implementar: refresh function
  - Usar React Query para caching y sincronización
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

- [ ] 3.4 Crear componente: ListaCitasPendientes
  - Crear archivo src/features/confirmacion-citas/components/ListaCitasPendientes.tsx
  - Implementar tabla con columnas: Paciente, Fecha/Hora, Estado, Intentos, Acciones
  - Agregar filtros: todas, próximas 48h, confirmadas, canceladas
  - Agregar ordenamiento: por fecha, por reintentos
  - Implementar badges: "Pendiente", "Confirmada", "Cancelada", "Reprogramada"
  - Implementar ícono de advertencia para retry_count >= 2
  - Agregar botones de acción rápida: Confirmar, Cancelar, Reprogramar
  - Implementar contador de citas pendientes en header
  - Agregar paginación (20 registros/página)
  - _Requirements: 1.1, 1.2, 1.5, 2.3, 2.4, 2.5_

- [ ] 3.5 Crear componente: ModalConfirmacionCita
  - Crear archivo src/features/confirmacion-citas/components/ModalConfirmacionCita.tsx
  - Mostrar información del paciente (nombre, teléfono, WhatsApp)
  - Mostrar fecha/hora de la cita
  - Campo textarea para observaciones (mínimo 10 caracteres)
  - Botones: "Confirmar", "Cancelar"
  - Validación: observaciones requeridas, no vacías
  - Callback onConfirm que llama useGestiones.createGestionConfirmacion
  - Mostrar loading y error states
  - Feedback de éxito: toast "Cita confirmada a las [hora]"
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

- [ ] 3.6 Crear componente: ModalCancelacionCita
  - Crear archivo src/features/confirmacion-citas/components/ModalCancelacionCita.tsx
  - Mostrar información de la cita
  - Dropdown para seleccionar motivo: "Paciente enfermo", "Emergencia", "Cambio de planes", "Otros"
  - Campo textarea para observaciones adicionales (opcional)
  - Checkbox de confirmación: "¿Está seguro de cancelar?"
  - Botones: "Cancelar Cita", "Cerrar"
  - Callback onCancel que llama useGestiones.createGestionCancelacion
  - Feedback: toast "Cita cancelada"
  - Opción: "Ver otras citas disponibles" (para reprogramación)
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

- [ ] 3.7 Crear componente: ModalReprogramacionCita
  - Crear archivo src/features/confirmacion-citas/components/ModalReprogramacionCita.tsx
  - Mostrar fecha/hora actual de la cita
  - Date picker: Rango mínimo hoy, máximo 30 días adelante
  - Time picker: Bloques de 30min, horario 9am-6pm
  - Validar nueva_fecha_hora no esté en el pasado
  - Dropdown para motivo: "Paciente no puede", "Conflicto agenda", "Solicitud paciente", "Otros"
  - Botones: "Reprogramar", "Cerrar"
  - Callback onReschedule que llama useGestiones.createGestionReprogramacion
  - Feedback: toast "Cita reprogramada a [fecha/hora]"
  - Mostrar "Próximos horarios disponibles" (sugerencia)
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

- [ ] 3.8 Crear componente: HistorialGestiones
  - Crear archivo src/features/confirmacion-citas/components/HistorialGestiones.tsx
  - Timeline vertical mostrando todas las gestiones (más recientes arriba)
  - Para cada gestión: fecha/hora, tipo (ícono + color), respuesta, usuario, observaciones
  - Colores por tipo: Verde (confirmada), Rojo (cancelada), Amarillo (reprogramada), Gris (fallida)
  - Filtro por tipo_gestion
  - Búsqueda por observaciones
  - Modo compacto: últimas 3 gestiones
  - Modo detallado: todas con búsqueda
  - Estadísticas: Total intentos, confirmada sí/no, última acción
  - Botón "Exportar como JSON"
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [ ] 3.9 Crear página: ConfirmacionesCitas
  - Crear archivo src/pages/confirmacion-citas.tsx o src/features/confirmacion-citas/pages/ConfirmacionesCitas.tsx
  - Integrar ListaCitasPendientes como componente principal
  - Agregar header con título "Confirmación de Citas"
  - Agregar widgets de resumen: "Pendientes", "Confirmadas", "Canceladas", "Reprogramadas"
  - Agregar filtros y controles de vista
  - Manejar modales: ModalConfirmacionCita, ModalCancelacionCita, ModalReprogramacionCita
  - State para cita seleccionada y tipo de modal
  - Funciones: handleSelectCita, handleConfirmar, handleCancelar, handleReprogramar
  - _Requirements: 1.1, 1.2, 2.1, 2.2, 2.3, 2.4, 2.5_

- [ ] 3.10 Crear página: ReportesConfirmacion
  - Crear archivo src/pages/reportes-confirmacion.tsx
  - Mostrar estadísticas de confirmación (Q3)
  - Mostrar motivos de cancelación (Q4)
  - Date range picker para filtrar por período
  - Gráficos: Pie chart (confirmadas vs canceladas), Bar chart (motivos)
  - Tabla de estadísticas con: total, confirmadas, canceladas, reprogramadas, % confirmación
  - Botón "Exportar a CSV"
  - Acceso restringido a rol Doctora
  - _Requirements: 9.1, 9.2, 9.3, 9.4_

- [ ] 3.11 Integrar rutas en navegación principal
  - Agregar ruta /confirmacion-citas → ConfirmacionesCitas (acceso Asistente)
  - Agregar ruta /reportes-confirmacion → ReportesConfirmacion (acceso Doctora)
  - Actualizar menú de navegación con links a nuevas páginas
  - Agregar protección de rutas por rol (middleware)
  - Verificar que rutas son accesibles desde browser
  - _Requirements: Ninguno específico_

---

### Phase 4: Property-Based Tests

- [ ] 4.1 Implementar property test: Confirmación actualiza estado
  - Crear archivo src/features/confirmacion-citas/__tests__/confirmacion.property.test.ts
  - Usar librería fast-check para generación de datos
  - Generar: UUID para cita, string para observaciones
  - Precondición: Cita existe con estado 'programada'
  - Acción: Llamar createGestionConfirmacion
  - Postcondición: confirmacion_estado='confirmada' Y necesita_confirmacion=false
  - Configurar 100 iteraciones
  - Tag: **Feature: gestion-confirmacion-citas, Property 1: Confirmación Actualiza Estado**
  - Validar: Requirements 3.1, 3.2, 7.2
  - _Requirements: 3.1, 3.2, 7.2_

- [ ] 4.2 Implementar property test: Cancelación cambia estado de cita
  - Crear test: cancelacion.property.test.ts
  - Generar: UUID cita, string motivo
  - Precondición: Cita existe
  - Acción: createGestionCancelacion
  - Postcondición: cita.estado='cancelada' Y cita.confirmacion_estado='cancelada'
  - 100 iteraciones
  - Tag: **Feature: gestion-confirmacion-citas, Property 2: Cancelación Cambia Estado**
  - _Requirements: 4.1, 4.2, 7.3_

- [ ] 4.3 Implementar property test: Reprogramación crea cita nueva
  - Crear test: reprogramacion.property.test.ts
  - Generar: UUID cita, TIMESTAMPTZ fecha futura, string motivo
  - Precondición: Cita existe, fecha futura válida
  - Acción: createGestionReprogramacion
  - Postcondición: Cita nueva creada con confirmacion_estado='pendiente' Y Cita original cancelada Y referencia en nueva_cita_id
  - 100 iteraciones
  - Tag: **Feature: gestion-confirmacion-citas, Property 3: Reprogramación Crea Cita**
  - _Requirements: 5.1, 5.2, 5.3_

- [ ] 4.4 Implementar property test: Historial íntegro y ordenado
  - Crear test: historial.property.test.ts
  - Generar: UUID cita con múltiples gestiones (1-10)
  - Precondición: Gestiones existen en BD
  - Acción: getHistorialGestiones
  - Postcondición: Retorna TODAS las gestiones, ordenadas DESC por fecha_gestion, sin omisiones
  - 50 iteraciones (menos porque cada una crea múltiples registros)
  - Tag: **Feature: gestion-confirmacion-citas, Property 4: Historial Íntegro**
  - _Requirements: 6.1, 6.4, 6.5_

- [ ] 4.5 Implementar property test: Validación integridad referencial
  - Crear test: integridad.property.test.ts
  - Generar: UUID aleatorio (no válido), string inválido para tipo_gestion
  - Acción: Intentar createGestion con datos inválidos
  - Postcondición: Operación rechazada, error retornado
  - 100 iteraciones
  - Tag: **Feature: gestion-confirmacion-citas, Property 5: Validación Integridad**
  - _Requirements: 10.1, 10.2, 10.3_

- [ ] 4.6 Implementar property test: Conteo de reintentos
  - Crear test: reintentos.property.test.ts
  - Generar: UUID cita, múltiples gestiones con respuesta_paciente='sin_respuesta' (1-5)
  - Precondición: Gestiones existen
  - Acción: Actualizar retry_count después de cada intento fallido
  - Postcondición: retry_count refleja número total de intentos fallidos
  - 100 iteraciones
  - Tag: **Feature: gestion-confirmacion-citas, Property 6: Conteo Reintentos**
  - _Requirements: 8.4, 8.5_

- [ ] 4.7 Ejecutar todos los property tests
  - Comando: npm test -- --testPathPattern=confirmacion-citas
  - Verificar que todos pasan
  - Revisar cobertura de código
  - Documentar resultados
  - _Requirements: Todos_

---

### Phase 5: Documentación

- [ ] 5.1 Crear README.md para feature
  - Crear archivo src/features/confirmacion-citas/README.md
  - Sección: Overview (qué hace el feature)
  - Sección: Estructura de directorios
  - Sección: Componentes principales
  - Sección: Hooks disponibles
  - Sección: Tipos TypeScript
  - Sección: Ejemplos de uso
  - Sección: Testing (cómo ejecutar tests)
  - _Requirements: Documentación_

- [ ] 5.2 Documentar API de Edge Functions
  - Crear archivo SPEC-002-APIs.md
  - Documentar: createGestionConfirmacion, createGestionCancelacion, createGestionReprogramacion
  - Documentar: getCitasPendientes, getHistorialGestiones, getEstadisticasConfirmacion
  - Para cada API: propósito, input, output, errores, ejemplo
  - Incluir curl examples para testing manual
  - _Requirements: Documentación_

- [ ] 5.3 Actualizar INDICE.md del proyecto
  - Agregar sección: SPEC-002: Gestión de Confirmación de Citas
  - Listar archivos principales creados
  - Listar cambios a BD realizados
  - Listar componentes React nuevos
  - Listar rutas nuevas
  - Agregar enlace a requirements.md, design.md, tasks.md
  - _Requirements: Documentación_

- [ ] 5.4 Crear guía de usuario para Asistente
  - Crear archivo GUIA-ASISTENTE-CONFIRMACIONES.md
  - Paso a paso: Cómo confirmar cita
  - Paso a paso: Cómo cancelar cita
  - Paso a paso: Cómo reprogramar cita
  - Paso a paso: Cómo ver historial de gestiones
  - Captura de pantallas (cuando esté implementado)
  - Tips y trucos
  - Solución de problemas comunes
  - _Requirements: Documentación_

---

## Notes

### Consideraciones de Implementación

1. **Timezone**: Todas las operaciones deben usar `TIMESTAMPTZ` con timezone America/Managua. Verificar que Supabase está configurado correctamente.

2. **Reutilización de SPEC-001**: Las tablas pacientes, citas, tratamientos YA EXISTEN. Solo se modifica tabla citas y se crea gestiones_citas.

3. **RLS Security**: Las RLS policies son CRÍTICAS. Verificar que:
   - Asistente puede crear/actualizar gestiones_citas
   - Doctora solo puede LEER gestiones_citas
   - No hay filtrado accidental de datos

4. **Validación en Múltiples Capas**:
   - Validación en Edge Functions (server-side)
   - Validación en componentes React (client-side)
   - Validación en DB (CHECK constraints)

5. **Manejo de Errores**: Implementar try-catch en todas las funciones async. Mostrar mensajes de error amigables al usuario.

6. **Testing**: Las property tests son IMPORTANTES para verificar propiedades universales. Configurar mínimo 100 iteraciones.

7. **Wave Execution**: Las waves son independientes DENTRO DE LA WAVE, pero secuenciales ENTRE WAVES. Ejecutar Wave 0 → Wave 1 → Wave 2 → etc.

8. **Rollback Plan**: Si algo falla en Wave 1 (DB), crear script de rollback que elimine tabla gestiones_citas y revertir modificaciones a citas.

---

## Task Dependency Graph

```json
{
  "waves": [
    {
      "id": 0,
      "tasks": ["0.1", "0.2"]
    },
    {
      "id": 1,
      "tasks": ["1.1", "1.2", "1.3", "1.4"]
    },
    {
      "id": 2,
      "tasks": ["2.1", "2.2", "2.3", "2.4", "2.5", "2.6", "2.7", "2.8"]
    },
    {
      "id": 3,
      "tasks": ["3.1", "3.2", "3.3", "3.4", "3.5", "3.6", "3.7", "3.8", "3.9", "3.10", "3.11"]
    },
    {
      "id": 4,
      "tasks": ["4.1", "4.2", "4.3", "4.4", "4.5", "4.6", "4.7"]
    },
    {
      "id": 5,
      "tasks": ["5.1", "5.2", "5.3", "5.4"]
    }
  ]
}
```

</content>
