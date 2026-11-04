# 🏗️ DentiFlow - Arquitectura Completa

## Vista General del Sistema

\\\
┌─────────────────────────────────────────────────────────────────┐
│                     DentiFlow System Architecture                │
└─────────────────────────────────────────────────────────────────┘

                          USUARIOS
                     ┌──────────────┐
                     │ Asistente    │
                     │ (read/write) │
                     └──────┬───────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
    ┌────▼─────┐    ┌──────▼──────┐   ┌──────▼──────┐
    │Registrar │    │Crear        │   │Seguimiento │
    │Pacientes │    │Tratamientos │   │Pacientes   │
    └────┬─────┘    └──────┬──────┘   └──────┬──────┘
         │                 │                 │
         └─────────────────┼─────────────────┘
                          │
                ┌─────────▼──────────┐
                │  React Frontend    │
                │  (TypeScript +     │
                │   Tailwind CSS)    │
                └─────────┬──────────┘
                          │ HTTPS API
         ┌────────────────▼─────────────────┐
         │    Supabase Backend              │
         │  ┌─────────────────────────────┐ │
         │  │ Supabase Auth (RLS + JWT)  │ │
         │  │ - Asistente role           │ │
         │  │ - Doctora role             │ │
         │  └─────────────────────────────┘ │
         │  ┌─────────────────────────────┐ │
         │  │ PostgreSQL Database         │ │
         │  │ ┌──────────────────────────┐│ │
         │  │ │ Tablas:                  ││ │
         │  │ │ - pacientes              ││ │
         │  │ │ - citas                  ││ │
         │  │ │ - tratamientos           ││ │
         │  │ │ - sesiones_tratamientos  ││ │
         │  │ │ - seguimientos_pacientes ││ │
         │  │ └──────────────────────────┘│ │
         │  │ - RLS Policies (Seguridad) │ │
         │  │ - Triggers (updated_at)    │ │
         │  │ - Constraints (Validación) │ │
         │  └─────────────────────────────┘ │
         │  ┌─────────────────────────────┐ │
         │  │ Edge Functions              │ │
         │  │ - Detectar tratamientos    │ │
         │  │   pendientes de seguimiento │ │
         │  └─────────────────────────────┘ │
         └────────────────────────────────────┘
\\\

## Flujo de Datos Principal

\\\
CREAR PACIENTE → CREAR TRATAMIENTO (N sesiones)
                           ↓
        COMPLETAR SESIONES (1, 2, 3, ...)
                           ↓
        Sistema incrementa automáticamente:
        - sesiones_completadas += 1
        - fecha_ultima_sesion = hoy
        - si último → estado='finalizado'
                           ↓
        ¿Tiene sesiones pendientes sin cita futura?
                    SÍ ↓ NO ↓
                       ↓   (esperar siguiente sesión)
            ALERTA VISIBLE
        "Tratamiento pendiente seguimiento"
                       ↓
        ASISTENTE REGISTRA SEGUIMIENTO
        - Medio: WhatsApp, Teléfono, Presencial
        - Resultado: Contacto exitoso, Sin respuesta...
        - Observaciones: Notas del contacto
                       ↓
        PACIENTE PROGRAMA PRÓXIMA SESIÓN
        → estado vuelve a 'en_curso'
        → alerta se oculta
                       ↓
        CICLO CONTINÚA hasta finalizar tratamiento
\\\

## Modelo de Datos

\\\sql
pacientes (1:N) → citas (1:N) → sesiones_tratamientos
         ↓
      tratamientos (1:N) → sesiones_tratamientos
         ↓
   seguimientos_pacientes

Constraints:
- sesiones_completadas <= total_sesiones_planificadas
- fecha_hora_fin > fecha_hora_inicio
- numero_sesion <= total_sesiones_planificadas
\\\

## Estados de Tratamiento

\\\
┌────────────────────────────────────────────┐
│ Máquina de Estados de Tratamiento          │
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

## Componentes React

\\\
App
├── Header (Logo, Usuario, Logout)
├── Sidebar (Navegación)
└── Router
    ├── PacientesPage
    │   ├── ListaPacientes (tabla, búsqueda)
    │   ├── FormularioPaciente (modal)
    │   └── DetallePaciente
    │
    ├── TratamientosPage
    │   ├── ListaTratamientos (progreso visual)
    │   ├── FormularioTratamiento (modal)
    │   └── DetalleTratamiento (con sesiones)
    │
    ├── SesionesPage
    │   ├── ListaSesiones
    │   └── BotónMarcarCompletada
    │
    ├── AlertasPage (Panel Alertas)
    │   └── TarjetaAlerta (click → Seguimiento)
    │
    └── SeguimientosPage
        ├── FormularioSeguimiento
        └── HistorialSeguimientos (con filtros)

Componentes Comunes:
- Header, Sidebar, Badge, LoadingSpinner, ErrorMessage
\\\

## Timeline de Implementación

\\\
SEMANA 1
├─ Días 1-2 (8h): Base de Datos (Wave 1)
│  └─ 5 tablas + triggers + RLS
├─ Días 2-4 (12h): Backend API (Wave 2)
│  └─ 6 CRUD + queries
└─ Día 5 (4h): Inicio Frontend (Wave 3)
   └─ Componentes comunes + estructura

SEMANA 2
├─ Días 1-3 (16h): Frontend principal (Wave 3)
│  └─ 6 páginas principales
├─ Días 3-4 (8h): Testing (Wave 4)
│  └─ Unit + Integration + E2E
└─ Día 5 (2h): Documentación (Wave 5)
   └─ Guía de usuario

Total: 50 horas ≈ 2 semanas (10 días × 5 horas/día)
Equipo: Backend Lead + Backend Dev + Frontend Lead + QA
\\\

## Criterios de Aceptación Visualizados

\\\
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
\\\

## Matriz de Seguridad (RLS)

\\\
        │ SELECT │ INSERT │ UPDATE │ DELETE │ Notas
────────┼────────┼────────┼────────┼────────┼──────────────
Asistente│   ✅   │   ✅   │   ✅   │   ❌   │ Permite lectura+escritura
Doctora │   ✅   │   ❌   │   ❌   │   ❌   │ Solo lectura
Anónimo │   ❌   │   ❌   │   ❌   │   ❌   │ Sin acceso

Aplicado a:
- pacientes
- citas
- tratamientos
- sesiones_tratamientos
- seguimientos_pacientes
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

HERRAMIENTAS:
├─ Git (versionamiento)
├─ Vite (build frontend)
├─ TypeScript (tipado estático)
└─ Supabase CLI (desarrollo local)

DEPLOYMENT:
├─ Frontend: Vercel o Netlify
├─ Backend: Supabase hosted
└─ Database: Supabase PostgreSQL
\\\

## Próximas Fases (Roadmap Futuro)

\\\
Fase 1: ✅ COMPLETADA
- Gestión de Continuidad de Tratamientos

Fase 2: ⏳ PENDIENTE (Integración)
- Sincronización Google Calendar
- Recordatorios automáticos

Fase 3: ⏳ PENDIENTE (Automatización)
- WhatsApp Business API
- Confirmaciones automáticas
- Seguimientos automáticos

Fase 4: ⏳ PENDIENTE (Analytics)
- Dashboard con KPIs
- Reportes exportables
- Análisis de tendencias
\\\

---

Este diagrama representa la arquitectura **completa**, **validada** y **especificada** para DentiFlow.
