# 🏗️ DentiFlow - Arquitectura Completa

## Vista General del Sistema

\\\
┌─────────────────────────────────────────────────────────────────┐
│                     DentiFlow System Architecture                │
│                  (SPEC-001 + SPEC-002 Completo)                 │
└─────────────────────────────────────────────────────────────────┘

                          USUARIOS
                     ┌──────────────┐
                     │ Asistente    │
                     │ (read/write) │
                     └──────┬───────┘
                            │
         ┌──────────────────┼──────────────────┬──────────────┐
         │                  │                  │              │
    ┌────▼─────┐    ┌──────▼──────┐   ┌──────▼──────┐  ┌────▼──────┐
    │Registrar │    │Crear        │   │Seguimiento │  │Confirmar  │
    │Pacientes │    │Tratamientos │   │Pacientes   │  │Citas      │
    └────┬─────┘    └──────┬──────┘   └──────┬──────┘  └────┬──────┘
         │                 │                 │              │
         └─────────────────┼─────────────────┼──────────────┘
                           │                 │
                ┌──────────▼─────────────────▼──────┐
                │  React Frontend (TypeScript +     │
                │   Tailwind CSS)                    │
                │                                    │
                │ Módulos:                           │
                │ - Pacientes (SPEC-001)             │
                │ - Tratamientos (SPEC-001)          │
                │ - Sesiones (SPEC-001)              │
                │ - Alertas (SPEC-001)               │
                │ - Seguimientos (SPEC-001)          │
                │ - Confirmaciones (SPEC-002)        │
                │ - Gestiones (SPEC-002)             │
                └──────────┬────────────────────────┘
                           │ HTTPS REST API
         ┌─────────────────▼──────────────────┐
         │    Supabase Backend                │
         │  ┌──────────────────────────────┐  │
         │  │ Supabase Auth (RLS + JWT)    │  │
         │  │ - Asistente role (full)      │  │
         │  │ - Doctora role (read-only)   │  │
         │  └──────────────────────────────┘  │
         │  ┌──────────────────────────────┐  │
         │  │ PostgreSQL Database          │  │
         │  │ ┌────────────────────────────┤  │
         │  │ │ SPEC-001 Tablas:           │  │
         │  │ │ - pacientes                │  │
         │  │ │ - citas                    │  │
         │  │ │ - tratamientos             │  │
         │  │ │ - sesiones_tratamientos    │  │
         │  │ │ - seguimientos_pacientes   │  │
         │  │ ├────────────────────────────┤  │
         │  │ │ SPEC-002 Tablas (Nueva):   │  │
         │  │ │ - gestiones_citas (NEW)    │  │
         │  │ │ - citas (extendida)        │  │
         │  │ ├────────────────────────────┤  │
         │  │ │ RLS Policies (Seguridad)   │  │
         │  │ │ Triggers (updated_at)      │  │
         │  │ │ Constraints (Validación)   │  │
         │  │ └────────────────────────────┘  │
         │  ├──────────────────────────────┐  │
         │  │ Edge Functions               │  │
         │  │ - Detectar tratamientos     │  │
         │  │   pendientes de seguimiento  │  │
         │  │ - Citas pendientes de conf.  │  │
         │  └──────────────────────────────┘  │
         └──────────────────────────────────┘
\\\

## Flujo de Datos Principal (SPEC-001 + SPEC-002)

\\\
SPEC-001: GESTIÓN DE CONTINUIDAD DE TRATAMIENTOS
═══════════════════════════════════════════════════

CREAR PACIENTE → CREAR TRATAMIENTO (N sesiones) → CREAR CITA
                           ↓                         ↓
                COMPLETAR SESIONES            CONFIRMAR CITA (SPEC-002)
                (1, 2, 3, ...)                      ↓
                           ↓              ┌─────────┴──────────┐
        Sistema incrementa:          Confirmada         Cancelada
        - sesiones_completadas       (estado=            (estado=
        - fecha_ultima_sesion         confirmada)         cancelada)
        - si último →                     ↓                  ↓
          estado='finalizado'      Retrasa seguimiento  Crear nueva cita
                       ↓
        ¿Tiene sesiones pendientes
         sin cita futura?
                    SÍ ↓ NO
                       ↓
            ALERTA VISIBLE
        "Tratamiento pendiente
         seguimiento"
                       ↓
        ASISTENTE REGISTRA SEGUIMIENTO (SPEC-001)
        - Medio: WhatsApp, Teléfono, Presencial
        - Resultado: Contacto exitoso, Sin respuesta...
        - Observaciones: Notas del contacto
                       ↓
        PACIENTE PROGRAMA PRÓXIMA SESIÓN
        → estado vuelve a 'en_curso'
        → alerta se oculta

SPEC-002: GESTIÓN DE CONFIRMACIÓN DE CITAS
═══════════════════════════════════════════════════

CITA PROGRAMADA (sin confirmación)
        ↓
ASISTENTE VE EN PANEL "CONFIRMACIONES"
        ↓
┌───────────────────────────────────┐
│ INTENTA CONTACTAR PACIENTE        │
│ (Teléfono, WhatsApp, etc.)        │
└───────────────────────────────────┘
        ↓
┌───────────────────────────────────────┐
│ ¿RESPONDE PACIENTE?                  │
│ ├─ SÍ → Confirma cita               │
│ ├─ NO → Registra intento fallido    │
│ └─ Cancela → Cancela cita           │
└───────────────────────────────────────┘
        ↓
REGISTRA GESTIÓN EN "gestiones_citas"
- Fecha del intento
- Medio utilizado
- Resultado
- Observaciones
        ↓
CITA ESTADO ACTUALIZADO
- confirmada → Sigue programada
- cancelada → Se marca cancelada
- reprogramada → Se mueve de fecha
        ↓
HISTORIAL DISPONIBLE
(Auditoría completa de todos los intentos)
\\\

## Modelo de Datos Completo

\\\sql
--- SPEC-001 ---
pacientes (1:N) → citas (1:N) → sesiones_tratamientos
         ↓
      tratamientos (1:N) → sesiones_tratamientos
         ↓
   seguimientos_pacientes

--- SPEC-002 (Extensión) ---
        ↓
   gestiones_citas (NEW) ← pacientes (1:N)
                       ↑
                     citas (1:N)

Constraints SPEC-001:
- sesiones_completadas <= total_sesiones_planificadas
- fecha_hora_fin > fecha_hora_inicio
- numero_sesion <= total_sesiones_planificadas

Constraints SPEC-002:
- confirmacion_estado IN ('pendiente', 'confirmada', 'cancelada', 'reprogramada')
- retry_count >= 0
- fecha_confirmacion >= fecha_programa (si confirmada)
\\\

## Estados de Tratamiento (SPEC-001)

\\\
┌────────────────────────────────────────────┐
│ Máquina de Estados de Tratamiento          │
│ (SPEC-001: Gestión de Continuidad)         │
└────────────────────────────────────────────┘

       ┌─────────────────┐
       │ pendiente_      │
       │ iniciar         │
       │ (inicial)       │
       └────────┬────────┘
                │ (crear primera sesión)
       ┌────────▼────────┐
       │ en_curso        │◄─────────────┐
       │ (sesiones < N)  │              │
       └────────┬────────┘              │
                │                       │
       (sesiones < N &&              (programar
        sin cita futura)              próxima
                │                      cita)
       ┌────────▼──────────────┐      │
       │ pendiente_            │──────┘
       │ seguimiento           │
       │ (alerta visible)      │
       └────────┬──────────────┘
                │
    (completar última sesión)
                │
       ┌────────▼──────────┐
       │ finalizado        │
       │ (todas sesiones)  │
       └───────────────────┘

Otros estados:
- suspendido: Paciente solicita pausa
- abandonado: Sin contacto > 30 días
\\\

## Estados de Confirmación de Citas (SPEC-002)

\\\
┌────────────────────────────────────────────┐
│ Máquina de Estados de Confirmación         │
│ (SPEC-002: Gestión de Confirmación)        │
└────────────────────────────────────────────┘

       ┌──────────────────┐
       │ pendiente        │
       │ confirmacion     │
       │ (inicial)        │
       └────────┬─────────┘
                │
     (asistente intenta contactar)
                │
       ┌────────┴──────────┐
       │                   │
   ┌───▼──────┐    ┌──────▼──┐    ┌───────────┐
   │confirmada│    │cancelada │    │reprogramada
   │(paciente │    │(paciente │    │(nueva fecha)
   │confirma) │    │cancela)  │    │
   └──────────┘    └──────────┘    └───────────┘
       ↓                 ↓              ↓
   Cita mantiene   Cita cancelada   Cita con
   programación    se registra      fecha nueva
       
Auditoría completa en gestiones_citas:
- Cada intento se registra
- Medio utilizado (teléfono, whatsapp, presencial)
- Resultado (contacto_exitoso, sin_respuesta, rechaza)
- Observaciones del asistente
\\\

## Componentes React

\\\
App
├── Header (Logo, Usuario, Logout)
├── Sidebar (Navegación)
└── Router
    │
    ├─ SPEC-001 Módulos:
    │  ├── PacientesPage
    │  │   ├── ListaPacientes (tabla, búsqueda)
    │  │   ├── FormularioPaciente (modal)
    │  │   └── DetallePaciente
    │  │
    │  ├── TratamientosPage
    │  │   ├── ListaTratamientos (progreso visual)
    │  │   ├── FormularioTratamiento (modal)
    │  │   └── DetalleTratamiento (con sesiones)
    │  │
    │  ├── SesionesPage
    │  │   ├── ListaSesiones
    │  │   └── BotónMarcarCompletada
    │  │
    │  ├── AlertasPage (Panel Alertas)
    │  │   └── TarjetaAlerta (click → Seguimiento)
    │  │
    │  └── SeguimientosPage
    │      ├── FormularioSeguimiento
    │      └── HistorialSeguimientos (con filtros)
    │
    └─ SPEC-002 Módulos:
       ├── ConfirmacionesPage (NEW)
       │   ├── ListaCitasPendientes
       │   ├── ModalConfirmacion
       │   └── RegistroGestión
       │
       └── GestionesPage (NEW)
           ├── HistorialGestiones
           ├── FiltrosPorEstado
           └── DetallesGestión

Componentes Comunes:
- Header, Sidebar, Badge, LoadingSpinner, ErrorMessage
- TablaGenerica, ModalGenerico, FormularioGenerico
\\\

## Timeline de Implementación (Completo)

\\\
FASE 1: SPEC-001 (Semanas 1-2)
═════════════════════════════════

SEMANA 1
├─ Días 1-2 (8h): Wave 1 - Base de Datos
│  └─ 5 tablas + triggers + RLS
├─ Días 2-4 (12h): Wave 2 - Backend API
│  └─ 6 CRUD + queries
└─ Día 5 (4h): Wave 3 - Inicio Frontend
   └─ Componentes comunes + estructura

SEMANA 2
├─ Días 1-3 (16h): Wave 3 - Frontend principal
│  └─ 6 páginas (Pacientes, Citas, Tratamientos, Sesiones, Alertas, Seguimientos)
├─ Días 3-4 (8h): Wave 4 - Testing
│  └─ Unit + Integration + E2E
└─ Día 5 (2h): Wave 5 - Documentación
   └─ Guía de usuario

Subtotal SPEC-001: 50 horas

FASE 2: SPEC-002 (Semanas 3-4)
═════════════════════════════════
[Requiere SPEC-001 completada]

SEMANA 3
├─ Día 1 (2h): Wave 0 - Setup
│  └─ Verificación de SPEC-001, rama de desarrollo
├─ Días 1-2 (4h): Wave 1 - Base de Datos
│  └─ Nueva tabla gestiones_citas + extensiones a citas
├─ Días 2-4 (8h): Wave 2 - Backend API
│  └─ CRUD gestiones + queries de confirmaciones
└─ Día 5 (4h): Wave 3 - Inicio Frontend
   └─ Componentes de confirmación

SEMANA 4
├─ Días 1-3 (12h): Wave 3 - Frontend principal
│  └─ 2 páginas (Confirmaciones, Historial Gestiones)
├─ Días 3-4 (6h): Wave 4 - Property Tests
│  └─ PBT para integridad de confirmaciones
└─ Día 5 (2h): Wave 5 - Documentación
   └─ Guía de gestión de confirmaciones

Subtotal SPEC-002: 38 horas

TOTAL: 88 horas ≈ 4 semanas (20 días × 4.4 horas/día)
Equipo: Backend Lead + Backend Dev + Frontend Lead + QA
\\\

## Criterios de Aceptación Visualizados

\\\
SPEC-001 - GESTIÓN DE CONTINUIDAD
═════════════════════════════════════

CA1: Crear Tratamiento de 3 Sesiones
     Input: Tipo=Endodoncia, Sesiones=3
     Output: ✅ estado='pendiente_iniciar', sesiones_completadas=0

CA2: Completar Sesión #2
     Before: sesiones_completadas=1
     After:  ✅ sesiones_completadas=2, fecha_ultima_sesion=hoy

CA3: Detectar Tratamiento en Riesgo
     Condición: sesiones < total && NO cita futura
     Output: ✅ estado='pendiente_seguimiento', alerta visible

CA4: Registrar Seguimiento
     Input: medio=whatsapp, resultado=sin_respuesta, obs=...
     Output: ✅ Guardado en BD, alerta sigue visible

CA5: Completar Último Sesión
     Input: Marcar sesión #3 completada
     Output: ✅ estado='finalizado', fecha_finalizacion=hoy
             ✅ Alerta se oculta

SPEC-002 - GESTIÓN DE CONFIRMACIÓN DE CITAS
═════════════════════════════════════════════

CA6: Ver Citas Pendientes de Confirmación
     Input: Asistente abre panel "Confirmaciones"
     Output: ✅ Lista de citas sin confirmar en próximos 7 días

CA7: Confirmar Cita
     Input: Asistente marca cita como confirmada
     Output: ✅ Estado = 'confirmada', fecha_confirmacion = hoy

CA8: Registrar Intento Fallido
     Input: Asistente marca "sin respuesta" + observaciones
     Output: ✅ Gestión registrada, retry_count incrementa

CA9: Cancelar Cita
     Input: Asistente cancela cita
     Output: ✅ Estado = 'cancelada', se registra en gestiones_citas

CA10: Ver Historial de Gestiones
      Input: Asistente abre historial de cita X
      Output: ✅ Todos los intentos de confirmación (media, resultados, fechas)
\\\

## Matriz de Seguridad (RLS)

\\\
        │ SELECT │ INSERT │ UPDATE │ DELETE │ Notas
────────┼────────┼────────┼────────┼────────┼──────────────────────
Asistente│   ✅   │   ✅   │   ✅   │   ❌   │ Lectura + escritura completa
Doctora │   ✅   │   ❌   │   ❌   │   ❌   │ Solo lectura (reportes)
Anónimo │   ❌   │   ❌   │   ❌   │   ❌   │ Sin acceso

Aplicado a todas las tablas (SPEC-001 + SPEC-002):
- pacientes
- citas
- tratamientos
- sesiones_tratamientos
- seguimientos_pacientes
- gestiones_citas (NEW)
\\\

## Tecnologías Stack

\\\
FRONTEND:
├─ React 18+
├─ TypeScript 5+
├─ Tailwind CSS 3+
├─ Heroicons (iconografía)
└─ Supabase Client SDK

BACKEND:
├─ Supabase (BaaS)
├─ PostgreSQL 14+
├─ JWT (autenticación)
├─ RLS (autorización)
└─ Edge Functions (procesamiento)

TESTING:
├─ Vitest (unit tests)
├─ Testing Library (component tests)
├─ Playwright (E2E)
└─ Property-based testing (especificaciones)

HERRAMIENTAS:
├─ Git (versionamiento)
├─ Vite (build frontend)
├─ TypeScript (tipado estático)
├─ Supabase CLI (desarrollo local)
└─ Kiro (Spec Driven Development)

DEPLOYMENT:
├─ Frontend: Vercel o Netlify
├─ Backend: Supabase hosted
└─ Database: Supabase PostgreSQL (America/Managua)
\\\

## Próximas Fases (Roadmap Futuro)

\\\
Fase 1: ✅ ESPECIFICADA (Implementación pendiente)
- Gestión de Continuidad de Tratamientos (SPEC-001)
- Estimación: 50 horas

Fase 2: ✅ ESPECIFICADA (Implementación pendiente)
- Gestión de Confirmación de Citas (SPEC-002)
- Estimación: 38 horas
- Dependencia: SPEC-001

Fase 3: ⏳ DISEÑO FUTURO (Integración)
- Sincronización Google Calendar
- Recordatorios automáticos 48h/24h
- Detección de cambios bidireccionales

Fase 4: ⏳ DISEÑO FUTURO (Automatización)
- WhatsApp Business API
- Confirmaciones automáticas
- Seguimientos automáticos

Fase 5: ⏳ DISEÑO FUTURO (Analytics)
- Dashboard con KPIs
- Reportes exportables
- Análisis de tendencias
- Predicción de abandonos
\\\

---

## Resumen Ejecutivo de Arquitectura

| Aspecto | SPEC-001 | SPEC-002 | Total |
|---------|----------|----------|-------|
| **Tablas BD** | 5 | 1 (+ extensiones) | 6 |
| **Componentes React** | 6 páginas | 2 páginas | 8 páginas |
| **Horas Estimadas** | 50h | 38h | 88h |
| **Semanas** | 2 | 2 | 4 |
| **Roles Soportados** | Asistente + Doctora | Asistente + Doctora | 2 roles |
| **RLS Policies** | 5 tablas | 1 tabla nueva | 6 tablas |
| **Testing** | Unit + Integration + E2E | PBT + E2E | Completo |

**Esta arquitectura está completamente especificada, validada y lista para implementación.**

---

Este documento representa la arquitectura **COMPLETA**, **VALIDADA** y **ESPECIFICADA** para DentiFlow con ambas features (SPEC-001 + SPEC-002).
