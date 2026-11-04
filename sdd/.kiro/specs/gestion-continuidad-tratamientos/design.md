# Design Document: Gestión de Continuidad de Tratamientos Odontológicos

## Overview

Este documento describe la arquitectura técnica, modelo de datos y componentes para implementar el feature "Gestión de Continuidad de Tratamientos Odontológicos" en DentiFlow.

**Objetivo**: Permitir que la asistente gestione el ciclo completo de tratamientos multi-sesión, desde el registro del paciente hasta la identificación automática de tratamientos que requieren seguimiento.

---


## Correctness Properties

### CP1: Integridad de Sesiones
- Property: sesiones_completadas nunca debe exceder total_sesiones_planificadas
- Enforcement: CHECK constraint + validación en aplicación
- Verification: TEST que intente actualizar sesiones_completadas > total

### CP2: Estado Consistente de Tratamientos
- Property: Si sesiones_completadas = total_sesiones, estado debe ser 'finalizado'
- Enforcement: Trigger automático al actualizar sesiones
- Verification: Query que encuentre inconsistencias

### CP3: Citas Futuras Validas
- Property: fecha_hora_fin > fecha_hora_inicio
- Enforcement: CHECK constraint
- Verification: TEST que intente crear cita inválida

### CP4: RLS Enforcement
- Property: Datos solo accesibles por usuarios autorizados
- Enforcement: RLS policies en PostgreSQL
- Verification: TEST login como diferentes roles

## Error Handling

### Errores de Validación
- Teléfono inválido → Mostrar mensaje "Formato de teléfono inválido (8-15 dígitos)"
- Fecha pasada → "No puede programar cita en fecha pasada"
- Sesiones > Total → "No puede completar más sesiones de las planificadas"

### Errores de Base de Datos
- Foreign key violation → "Paciente o tratamiento no encontrado"
- RLS violation → "No tiene permisos para acceder a este recurso"
- Connection error → Reintentar automáticamente, mostrar modal

### Errores de Lógica de Negocio
- Tratamiento sin sesiones → Prevenir eliminación
- Paciente con tratamientos activos → No permitir eliminar paciente

## Testing Strategy

### Pruebas Unitarias
- Funciones de cálculo (progreso de tratamiento)
- Validación de formatos (teléfono, email)
- Transformación de datos

### Pruebas de Integración
- CRUD completo de tratamientos
- Detección automática de tratamientos pendientes
- Actualización de sesiones y progreso

### Pruebas E2E
- Flujo completo: Registrar paciente → Crear tratamiento → Completar sesión → Ver alerta
- Cambios de estado (en_curso → finalizado)
- RLS (acceso por rol)
## Architecture

### 2.1 Stack Tecnológico

\\\
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (React + TS)                    │
│  ├─ Pages (Pacientes, Citas, Tratamientos, Seguimientos)   │
│  ├─ Components (Formularios, Tablas, Alertas)              │
│  └─ Hooks (usePacientes, useTratamientos, etc.)            │
└────────────────────────┬────────────────────────────────────┘
                         │ (HTTPS REST API)
┌────────────────────────▼────────────────────────────────────┐
│            SUPABASE (Backend as a Service)                  │
│  ├─ Supabase Auth (Autenticación)                          │
│  ├─ PostgreSQL Database                                     │
│  │  ├─ pacientes                                            │
│  │  ├─ citas                                                │
│  │  ├─ tratamientos                                         │
│  │  ├─ sesiones_tratamientos                               │
│  │  └─ seguimientos_pacientes                              │
│  ├─ RLS Policies (Row Level Security)                      │
│  ├─ Real-time Subscriptions                                │
│  └─ Edge Functions (Lógica del servidor)                   │
└─────────────────────────────────────────────────────────────┘
`

### 2.2 Flujo de Datos

\\\
Usuario (Asistente/Doctora)
        ↓
React Component
        ↓
Supabase Client (TypeScript SDK)
        ↓
Supabase API
        ↓
PostgreSQL Database + RLS
        ↓
Datos filtrados por rol
`

---

## Data Models

### 3.1 Tablas Principales

#### **Tabla: pacientes**

\\\sql
CREATE TABLE pacientes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  nombre_completo TEXT NOT NULL,
  telefono TEXT NOT NULL CONSTRAINT pacientes_telefono_check CHECK (telefono ~ '^[0-9]{8,15}$'),
  telefono_whatsapp TEXT CONSTRAINT pacientes_whatsapp_check CHECK (telefono_whatsapp IS NULL OR telefono_whatsapp ~ '^[0-9]{8,15}$'),
  correo_electronico TEXT,
  acepta_comunicaciones_whatsapp BOOLEAN DEFAULT false,
  fecha_registro DATE DEFAULT CURRENT_DATE,
  estado TEXT DEFAULT 'activo' CONSTRAINT pacientes_estado_check CHECK (estado IN ('activo', 'inactivo')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  CONSTRAINT pacientes_nombre_minlength CHECK (LENGTH(nombre_completo) >= 2)
);

CREATE INDEX idx_pacientes_nombre ON pacientes(nombre_completo);
CREATE INDEX idx_pacientes_telefono ON pacientes(telefono);
CREATE INDEX idx_pacientes_estado ON pacientes(estado);
\\\

#### **Tabla: citas**

\\\sql
CREATE TABLE citas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  paciente_id UUID NOT NULL REFERENCES pacientes(id) ON DELETE RESTRICT,
  fecha_hora_inicio TIMESTAMPTZ NOT NULL,
  fecha_hora_fin TIMESTAMPTZ,
  duracion_minutos INTEGER,
  estado TEXT DEFAULT 'programada' CONSTRAINT citas_estado_check 
    CHECK (estado IN ('programada', 'completada', 'cancelada', 'no_asistio')),
  motivo TEXT,
  observaciones_administrativas TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  CONSTRAINT citas_hora_valida CHECK (fecha_hora_fin IS NULL OR fecha_hora_fin > fecha_hora_inicio)
);

CREATE INDEX idx_citas_paciente_id ON citas(paciente_id);
CREATE INDEX idx_citas_fecha_hora ON citas(fecha_hora_inicio);
CREATE INDEX idx_citas_estado ON citas(estado);
\\\

#### **Tabla: tratamientos**

\\\sql
CREATE TABLE tratamientos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  paciente_id UUID NOT NULL REFERENCES pacientes(id) ON DELETE RESTRICT,
  tipo_tratamiento TEXT NOT NULL CONSTRAINT tratamientos_tipo_check 
    CHECK (tipo_tratamiento IN ('Endodoncia', 'Ortodoncia', 'Limpieza', 'Otros')),
  descripcion_administrativa TEXT,
  fecha_inicio DATE,
  total_sesiones_planificadas INTEGER NOT NULL CONSTRAINT tratamientos_sesiones_check CHECK (total_sesiones_planificadas >= 1 AND total_sesiones_planificadas <= 100),
  sesiones_completadas INTEGER DEFAULT 0 CONSTRAINT tratamientos_completadas_check CHECK (sesiones_completadas >= 0),
  fecha_ultima_sesion DATE,
  fecha_proxima_sesion DATE,
  fecha_finalizacion DATE,
  estado TEXT DEFAULT 'pendiente_iniciar' CONSTRAINT tratamientos_estado_check 
    CHECK (estado IN ('pendiente_iniciar', 'en_curso', 'pendiente_seguimiento', 'finalizado', 'suspendido', 'abandonado')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  CONSTRAINT tratamientos_sesiones_lógica CHECK (sesiones_completadas <= total_sesiones_planificadas)
);

CREATE INDEX idx_tratamientos_paciente_id ON tratamientos(paciente_id);
CREATE INDEX idx_tratamientos_estado ON tratamientos(estado);
CREATE INDEX idx_tratamientos_fecha_ultima ON tratamientos(fecha_ultima_sesion);
\\\

#### **Tabla: sesiones_tratamientos**

\\\sql
CREATE TABLE sesiones_tratamientos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tratamiento_id UUID NOT NULL REFERENCES tratamientos(id) ON DELETE RESTRICT,
  cita_id UUID REFERENCES citas(id) ON DELETE SET NULL,
  numero_sesion INTEGER NOT NULL CONSTRAINT sesiones_numero_check CHECK (numero_sesion >= 1),
  fecha_programada DATE,
  fecha_realizacion DATE,
  estado TEXT DEFAULT 'pendiente' CONSTRAINT sesiones_estado_check 
    CHECK (estado IN ('pendiente', 'completada', 'cancelada', 'reprogramada')),
  observacion_administrativa TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_sesiones_tratamiento_id ON sesiones_tratamientos(tratamiento_id);
CREATE INDEX idx_sesiones_numero ON sesiones_tratamientos(numero_sesion);
CREATE INDEX idx_sesiones_estado ON sesiones_tratamientos(estado);
\\\

#### **Tabla: seguimientos_pacientes**

\\\sql
CREATE TABLE seguimientos_pacientes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  paciente_id UUID NOT NULL REFERENCES pacientes(id) ON DELETE RESTRICT,
  tratamiento_id UUID REFERENCES tratamientos(id) ON DELETE SET NULL,
  fecha_seguimiento TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  medio_contacto TEXT CONSTRAINT seguimientos_medio_check 
    CHECK (medio_contacto IN ('telefono', 'whatsapp', 'presencial', 'correo')),
  motivo_seguimiento TEXT,
  resultado TEXT CONSTRAINT seguimientos_resultado_check 
    CHECK (resultado IN ('contacto_exitoso', 'sin_respuesta', 'numero_invalido', 'paciente_rechaza')),
  proxima_fecha_seguimiento DATE,
  observacion_administrativa TEXT,
  estado TEXT DEFAULT 'pendiente' CONSTRAINT seguimientos_estado_check 
    CHECK (estado IN ('pendiente', 'completado', 'cancelado')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_seguimientos_paciente_id ON seguimientos_pacientes(paciente_id);
CREATE INDEX idx_seguimientos_tratamiento_id ON seguimientos_pacientes(tratamiento_id);
CREATE INDEX idx_seguimientos_estado ON seguimientos_pacientes(estado);
CREATE INDEX idx_seguimientos_fecha ON seguimientos_pacientes(fecha_seguimiento);
\\\

### 3.2 Triggers para Actualización Automática

\\\sql
-- Función para actualizar updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS \$\$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
\$\$ language 'plpgsql';

-- Aplicar trigger a todas las tablas
CREATE TRIGGER pacientes_updated_at BEFORE UPDATE ON pacientes
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER citas_updated_at BEFORE UPDATE ON citas
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER tratamientos_updated_at BEFORE UPDATE ON tratamientos
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER sesiones_tratamientos_updated_at BEFORE UPDATE ON sesiones_tratamientos
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER seguimientos_pacientes_updated_at BEFORE UPDATE ON seguimientos_pacientes
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
\\\

### 3.3 Políticas RLS (Row Level Security)

\\\sql
-- Habilitar RLS en todas las tablas
ALTER TABLE pacientes ENABLE ROW LEVEL SECURITY;
ALTER TABLE citas ENABLE ROW LEVEL SECURITY;
ALTER TABLE tratamientos ENABLE ROW LEVEL SECURITY;
ALTER TABLE sesiones_tratamientos ENABLE ROW LEVEL SECURITY;
ALTER TABLE seguimientos_pacientes ENABLE ROW LEVEL SECURITY;

-- POLICY para pacientes (Asistente: CRUD, Doctora: SELECT)
CREATE POLICY "Asistente puede leer y modificar pacientes" ON pacientes
  FOR ALL USING (auth.jwt() ->> 'role' = 'asistente');

CREATE POLICY "Doctora puede leer pacientes" ON pacientes
  FOR SELECT USING (auth.jwt() ->> 'role' = 'doctora');

-- POLICY para citas
CREATE POLICY "Asistente puede leer y modificar citas" ON citas
  FOR ALL USING (auth.jwt() ->> 'role' = 'asistente');

CREATE POLICY "Doctora puede leer citas" ON citas
  FOR SELECT USING (auth.jwt() ->> 'role' = 'doctora');

-- POLICY para tratamientos
CREATE POLICY "Asistente puede leer y modificar tratamientos" ON tratamientos
  FOR ALL USING (auth.jwt() ->> 'role' = 'asistente');

CREATE POLICY "Doctora puede leer tratamientos" ON tratamientos
  FOR SELECT USING (auth.jwt() ->> 'role' = 'doctora');

-- POLICY para sesiones_tratamientos
CREATE POLICY "Asistente puede leer y modificar sesiones" ON sesiones_tratamientos
  FOR ALL USING (auth.jwt() ->> 'role' = 'asistente');

CREATE POLICY "Doctora puede leer sesiones" ON sesiones_tratamientos
  FOR SELECT USING (auth.jwt() ->> 'role' = 'doctora');

-- POLICY para seguimientos_pacientes
CREATE POLICY "Asistente puede leer y modificar seguimientos" ON seguimientos_pacientes
  FOR ALL USING (auth.jwt() ->> 'role' = 'asistente');

CREATE POLICY "Doctora puede leer seguimientos" ON seguimientos_pacientes
  FOR SELECT USING (auth.jwt() ->> 'role' = 'doctora');
\\\

---

## 4. Queries Principales

### 4.1 Detección Automática de Tratamientos Pendientes de Seguimiento

\\\sql
-- Query: Identificar tratamientos que necesitan seguimiento
SELECT 
  t.id,
  t.paciente_id,
  p.nombre_completo,
  t.tipo_tratamiento,
  t.total_sesiones_planificadas,
  t.sesiones_completadas,
  t.fecha_ultima_sesion,
  CASE 
    WHEN t.fecha_ultima_sesion IS NULL THEN 0
    ELSE EXTRACT(DAY FROM NOW() - t.fecha_ultima_sesion::timestamp)
  END AS dias_sin_sesion
FROM tratamientos t
JOIN pacientes p ON t.paciente_id = p.id
WHERE t.sesiones_completadas < t.total_sesiones_planificadas
  AND t.estado = 'en_curso'
  AND NOT EXISTS (
    SELECT 1 FROM citas c 
    WHERE c.paciente_id = t.paciente_id 
      AND c.fecha_hora_inicio > NOW()
      AND c.estado != 'cancelada'
  )
ORDER BY t.fecha_ultima_sesion ASC NULLS FIRST;
\\\

### 4.2 Actualizar Estado de Tratamiento Después de Completar Sesión

\\\	ypescript
// TypeScript con Supabase
async function marcarSesionCompletada(sesionId: string, fechaRealizacion: string) {
  const { data: sesion, error: sesionError } = await supabase
    .from('sesiones_tratamientos')
    .select('tratamiento_id')
    .eq('id', sesionId)
    .single();

  if (sesionError) throw sesionError;

  // Actualizar sesión
  const { error: updateError } = await supabase
    .from('sesiones_tratamientos')
    .update({
      estado: 'completada',
      fecha_realizacion: fechaRealizacion,
    })
    .eq('id', sesionId);

  if (updateError) throw updateError;

  // Obtener tratamiento
  const { data: tratamiento } = await supabase
    .from('tratamientos')
    .select('total_sesiones_planificadas, sesiones_completadas')
    .eq('id', sesion.tratamiento_id)
    .single();

  // Incrementar sesiones completadas
  const nuevoTotal = tratamiento.sesiones_completadas + 1;
  const nuevoEstado = nuevoTotal === tratamiento.total_sesiones_planificadas 
    ? 'finalizado' 
    : 'en_curso';

  const { error: tratamientoError } = await supabase
    .from('tratamientos')
    .update({
      sesiones_completadas: nuevoTotal,
      fecha_ultima_sesion: fechaRealizacion,
      estado: nuevoEstado,
      fecha_finalizacion: nuevoEstado === 'finalizado' ? new Date().toISOString() : null,
    })
    .eq('id', sesion.tratamiento_id);

  if (tratamientoError) throw tratamientoError;

  return { sesionId, tratamientoId: sesion.tratamiento_id, nuevoEstado };
}
\\\

---

## Components and Interfaces

### 5.1 Estructura de Carpetas

\\\
src/
├── components/
│   ├── Pacientes/
│   │   ├── ListaPacientes.tsx
│   │   ├── FormularioPaciente.tsx
│   │   └── DetallePaciente.tsx
│   ├── Tratamientos/
│   │   ├── ListaTratamientos.tsx
│   │   ├── FormularioTratamiento.tsx
│   │   └── ProgressoTratamiento.tsx
│   ├── Sesiones/
│   │   ├── ListaSesiones.tsx
│   │   └── FormularioSesion.tsx
│   ├── Seguimientos/
│   │   ├── PanelAlertas.tsx
│   │   ├── FormularioSeguimiento.tsx
│   │   └── HistorialSeguimientos.tsx
│   └── Common/
│       ├── Header.tsx
│       ├── Sidebar.tsx
│       └── Badge.tsx
├── hooks/
│   ├── usePacientes.ts
│   ├── useTratamientos.ts
│   ├── useSesiones.ts
│   └── useSeguimientos.ts
├── types/
│   └── database.ts
└── pages/
    ├── PacientesPage.tsx
    ├── TratamientosPage.tsx
    └── SeguimientosPage.tsx
\\\

### 5.2 Componente Ejemplo: Panel de Alertas

\\\	ypescript
// src/components/Seguimientos/PanelAlertas.tsx
import React, { useEffect, useState } from 'react';
import { ExclamationTriangleIcon } from '@heroicons/react/24/outline';
import { supabase } from '@/lib/supabase';

interface TratamientoPendiente {
  id: string;
  paciente_id: string;
  nombre_paciente: string;
  tipo_tratamiento: string;
  sesiones_completadas: number;
  total_sesiones_planificadas: number;
  dias_sin_sesion: number;
}

export function PanelAlertas() {
  const [alertas, setAlertas] = useState<TratamientoPendiente[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    cargarAlertas();
  }, []);

  const cargarAlertas = async () => {
    const { data, error } = await supabase
      .from('tratamientos')
      .select(\
        id,
        paciente_id,
        pacientes(nombre_completo),
        tipo_tratamiento,
        sesiones_completadas,
        total_sesiones_planificadas,
        fecha_ultima_sesion
      \)
      .eq('estado', 'pendiente_seguimiento')
      .order('fecha_ultima_sesion', { ascending: true });

    if (!error && data) {
      setAlertas(data.map(t => ({
        id: t.id,
        paciente_id: t.paciente_id,
        nombre_paciente: t.pacientes.nombre_completo,
        tipo_tratamiento: t.tipo_tratamiento,
        sesiones_completadas: t.sesiones_completadas,
        total_sesiones_planificadas: t.total_sesiones_planificadas,
        dias_sin_sesion: Math.floor(
          (Date.now() - new Date(t.fecha_ultima_sesion).getTime()) / (1000 * 60 * 60 * 24)
        ),
      })));
    }
    setLoading(false);
  };

  return (
    <div className="bg-white rounded-lg shadow-sm border border-gray-200 p-6">
      <div className="flex items-center mb-4">
        <ExclamationTriangleIcon className="h-6 w-6 text-yellow-600 mr-2" />
        <h2 className="text-lg font-semibold text-gray-900">
          Tratamientos que Requieren Seguimiento
        </h2>
      </div>

      {loading ? (
        <p className="text-gray-500">Cargando alertas...</p>
      ) : alertas.length === 0 ? (
        <p className="text-green-600">Todos los tratamientos tienen seguimiento programado ✓</p>
      ) : (
        <div className="space-y-3">
          {alertas.map(alerta => (
            <div 
              key={alerta.id} 
              className="bg-yellow-50 border-l-4 border-yellow-500 p-4 rounded"
            >
              <p className="font-semibold text-yellow-900">
                {alerta.nombre_paciente}
              </p>
              <p className="text-sm text-yellow-800 mt-1">
                {alerta.tipo_tratamiento}: {alerta.sesiones_completadas}/{alerta.total_sesiones_planificadas} sesiones
              </p>
              <p className="text-xs text-yellow-700 mt-1">
                Sin sesión por {alerta.dias_sin_sesion} días
              </p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
\\\

---

## 6. Flujo de Usuario Principal

\\\mermaid
graph TD
    A["Asistente Abre App"] --> B["Selecciona 'Nuevo Paciente'"]
    B --> C["Registra Paciente<br/>(Nombre, Teléfono, WhatsApp)"]
    C --> D["Crea Tratamiento<br/>(Tipo: Endodoncia, 3 Sesiones)"]
    D --> E["Sistema crea 3 Sesiones"]
    E --> F["Programa Cita #1"]
    F --> G["Registra Sesión #1 Completada"]
    G --> H["Sistema incrementa progreso<br/>(1/3 sesiones)"]
    H --> I["Sin Cita Futura?"]
    I -->|SÍ| J["Sistema crea ALERTA<br/>(pendiente_seguimiento)"]
    I -->|NO| K["Esperar Sesión #2"]
    J --> L["Asistente ve alerta<br/>en Panel"]
    L --> M["Registra Seguimiento<br/>(contactó al paciente)"]
    M --> N["Paciente acepta cita"]
    N --> O["Programa Cita #2<br/>(estado = en_curso)"]
    O --> P["Continuar hasta finalizar..."]
    P --> Q["Última Sesión Completada"]
    Q --> R["Estado = FINALIZADO<br/>Alerta oculta"]
\\\

---

## 7. Consideraciones de Seguridad

### 7.1 Validación en Frontend
- Validar formato de teléfono antes de enviar
- Validar que sesiones_completadas ≤ total_sesiones
- Validar estados permitidos

### 7.2 Validación en Backend (PostgreSQL)
- CHECK constraints en todas las columnas críticas
- Foreign keys con ON DELETE RESTRICT
- Triggers para evitar inconsistencias

### 7.3 RLS Policies
- Asistente: puede leer y modificar todas las tablas
- Doctora: solo lectura (SELECT)
- Datos del consultorio aislados por usuario autenticado

---

## 8. Próximas Fases (Fuera de Alcance Actual)

1. **Integración Google Calendar**: Sincronizar citas con Google Calendar
2. **Automatización WhatsApp**: Enviar recordatorios y seguimientos automáticos
3. **Dashboard con KPIs**: Métricas de asistencia y abandonos
4. **Reportes Avanzados**: Exportar a Excel/CSV

---

## 9. Referencias

- Supabase Docs: https://supabase.com/docs
- PostgreSQL Check Constraints: https://www.postgresql.org/docs/current/ddl-constraints.html
- React Hook Form: https://react-hook-form.com/

