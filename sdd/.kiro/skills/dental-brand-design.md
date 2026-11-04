---
name: dental-brand-design
description: Guía de diseño de marca y estilo visual para DentiFlow
---

# Skill: Diseño de Marca DentiFlow

## Identidad de Marca

### Nombre
**DentiFlow** - Representa el flujo continuo y sin fricciones en la gestión del consultorio dental.

### Personalidad de Marca
- **Profesional**: Transmite confianza y seriedad en el cuidado dental
- **Eficiente**: Refleja automatización y optimización de procesos
- **Accesible**: Interfaz amigable, no intimidante
- **Moderna**: Tecnología actual sin perder calidez humana

### Valores
- Precisión en el seguimiento de tratamientos
- Cuidado al paciente a través de recordatorios oportunos
- Eficiencia operativa para el personal del consultorio
- Transparencia en la información

---

## Paleta de Colores

### Colores Primarios

#### Azul Dental (Primary)
```css
--dental-blue-50:  #EFF6FF;
--dental-blue-100: #DBEAFE;
--dental-blue-200: #BFDBFE;
--dental-blue-300: #93C5FD;
--dental-blue-400: #60A5FA;
--dental-blue-500: #3B82F6; /* Principal */
--dental-blue-600: #2563EB;
--dental-blue-700: #1D4ED8;
--dental-blue-800: #1E40AF;
--dental-blue-900: #1E3A8A;
```

**Uso**: 
- Botones primarios
- Encabezados principales
- Links
- Iconos de navegación activa

#### Turquesa Clínico (Accent)
```css
--dental-cyan-50:  #ECFEFF;
--dental-cyan-100: #CFFAFE;
--dental-cyan-200: #A5F3FC;
--dental-cyan-300: #67E8F9;
--dental-cyan-400: #22D3EE;
--dental-cyan-500: #06B6D4; /* Acento */
--dental-cyan-600: #0891B2;
--dental-cyan-700: #0E7490;
--dental-cyan-800: #155E75;
--dental-cyan-900: #164E63;
```

**Uso**: 
- Alertas informativas
- Badges de estado "En curso"
- Decoración de cards importantes

### Colores de Estado

#### Éxito / Confirmado
```css
--success-50:  #F0FDF4;
--success-500: #10B981; /* Verde confirmación */
--success-700: #047857;
```

**Uso**: Estados confirmados, sesiones completadas, tratamientos finalizados

#### Advertencia / Pendiente
```css
--warning-50:  #FFFBEB;
--warning-500: #F59E0B; /* Ámbar */
--warning-700: #B45309;
```

**Uso**: Citas pendientes de confirmación, seguimientos pendientes

#### Error / Crítico
```css
--error-50:  #FEF2F2;
--error-500: #EF4444; /* Rojo */
--error-700: #B91C1C;
```

**Uso**: Citas canceladas, alertas críticas, tratamientos abandonados

#### Alerta Urgente
```css
--alert-500: #DC2626; /* Rojo intenso */
--alert-700: #991B1B;
```

**Uso**: Tratamientos sin seguimiento >30 días, recordatorios fallidos

### Colores Neutrales

```css
--gray-50:  #F9FAFB; /* Fondos suaves */
--gray-100: #F3F4F6; /* Fondos de cards */
--gray-200: #E5E7EB; /* Bordes sutiles */
--gray-300: #D1D5DB; /* Bordes normales */
--gray-400: #9CA3AF; /* Texto placeholder */
--gray-500: #6B7280; /* Texto secundario */
--gray-600: #4B5563; /* Texto normal */
--gray-700: #374151; /* Texto importante */
--gray-800: #1F2937; /* Texto encabezados */
--gray-900: #111827; /* Texto principal */
```

---

## Tipografía

### Fuente Principal
**Inter** (Sistema: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto)

```css
font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', 
             Roboto, 'Helvetica Neue', Arial, sans-serif;
```

**Características**: Moderna, legible, excelente en pantallas, profesional

### Escala Tipográfica

```css
/* Encabezados */
.text-h1 { font-size: 2.5rem; font-weight: 700; line-height: 1.2; }  /* 40px */
.text-h2 { font-size: 2rem;   font-weight: 700; line-height: 1.3; }  /* 32px */
.text-h3 { font-size: 1.5rem; font-weight: 600; line-height: 1.4; }  /* 24px */
.text-h4 { font-size: 1.25rem; font-weight: 600; line-height: 1.4; } /* 20px */
.text-h5 { font-size: 1.125rem; font-weight: 600; line-height: 1.5; }/* 18px */

/* Cuerpo */
.text-body-lg { font-size: 1.125rem; line-height: 1.75; }   /* 18px */
.text-body    { font-size: 1rem;     line-height: 1.5; }    /* 16px */
.text-body-sm { font-size: 0.875rem; line-height: 1.5; }    /* 14px */
.text-caption { font-size: 0.75rem;  line-height: 1.5; }    /* 12px */
```

### Pesos de Fuente
- **Regular (400)**: Texto normal
- **Medium (500)**: Labels, menú
- **Semibold (600)**: Subtítulos, énfasis
- **Bold (700)**: Títulos principales

---

## Componentes UI

### Cards

```tsx
// Card estándar
<div className="bg-white rounded-lg shadow-sm border border-gray-200 p-6">
  {/* Contenido */}
</div>

// Card con estado
<div className="bg-white rounded-lg shadow-sm border-l-4 border-l-blue-500 p-6">
  {/* Contenido destacado */}
</div>

// Card de alerta
<div className="bg-red-50 rounded-lg border border-red-200 p-4">
  {/* Alerta */}
</div>
```

### Botones

```tsx
// Primario
<button className="bg-blue-600 hover:bg-blue-700 text-white font-medium px-4 py-2 rounded-lg transition-colors">
  Acción Principal
</button>

// Secundario
<button className="bg-gray-100 hover:bg-gray-200 text-gray-700 font-medium px-4 py-2 rounded-lg transition-colors">
  Acción Secundaria
</button>

// Peligro
<button className="bg-red-600 hover:bg-red-700 text-white font-medium px-4 py-2 rounded-lg transition-colors">
  Eliminar
</button>
```

### Badges de Estado

```tsx
// Confirmado
<span className="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800">
  Confirmada
</span>

// Pendiente
<span className="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-yellow-100 text-yellow-800">
  Pendiente
</span>

// Cancelada
<span className="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-red-100 text-red-800">
  Cancelada
</span>
```

### Iconografía

**Librería recomendada**: Heroicons (by Tailwind)

```tsx
import { 
  UserIcon,           // Pacientes
  CalendarIcon,       // Citas
  ClipboardListIcon,  // Tratamientos
  BellAlertIcon,      // Alertas
  PhoneIcon,          // Seguimientos
  CheckCircleIcon,    // Completado
  ExclamationCircleIcon // Alerta
} from '@heroicons/react/24/outline';
```

**Tamaños**:
- Navegación: 24px (h-6 w-6)
- Botones: 20px (h-5 w-5)
- Inline: 16px (h-4 w-4)

---

## Layout y Espaciado

### Espaciado Base
Usar escala de Tailwind (múltiplos de 4px):

```css
/* Espaciado interno de componentes */
padding: 16px (p-4)  /* Componentes pequeños */
padding: 24px (p-6)  /* Cards, modales */
padding: 32px (p-8)  /* Contenedores principales */

/* Espaciado entre elementos */
gap: 8px  (gap-2)  /* Elementos relacionados */
gap: 16px (gap-4)  /* Grupos de elementos */
gap: 24px (gap-6)  /* Secciones */
```

### Estructura de Navegación

```
┌─────────────────────────────────────────────────────┐
│ Header: Logo + Usuario                              │ 60px
├─────────────┬───────────────────────────────────────┤
│             │                                       │
│  Sidebar    │  Contenido Principal                  │
│  (240px)    │                                       │
│             │                                       │
│  - Dashboard│  [Breadcrumbs]                        │
│  - Pacientes│                                       │
│  - Citas    │  <Contenido de la vista>              │
│  - Tratam.  │                                       │
│  - Seguim.  │                                       │
│  - Reportes │                                       │
│             │                                       │
└─────────────┴───────────────────────────────────────┘
```

### Responsive Breakpoints

```css
/* Mobile */
@media (max-width: 640px) {
  /* Sidebar oculto, menú hamburguesa */
}

/* Tablet */
@media (min-width: 768px) {
  /* Sidebar colapsable */
}

/* Desktop */
@media (min-width: 1024px) {
  /* Sidebar fijo visible */
}
```

---

## Patrones de Interacción

### Estados de Carga

```tsx
// Skeleton loader para cards
<div className="animate-pulse">
  <div className="h-4 bg-gray-200 rounded w-3/4 mb-2"></div>
  <div className="h-4 bg-gray-200 rounded w-1/2"></div>
</div>
```

### Feedback de Acciones

```tsx
// Toast de éxito
<div className="bg-green-50 border-l-4 border-green-500 p-4">
  <div className="flex items-center">
    <CheckCircleIcon className="h-5 w-5 text-green-500 mr-2" />
    <p className="text-green-800">Paciente registrado exitosamente</p>
  </div>
</div>
```

### Confirmaciones

```tsx
// Modal de confirmación
<div className="bg-white rounded-lg p-6 max-w-md">
  <h3 className="text-lg font-semibold text-gray-900 mb-2">
    ¿Cancelar cita?
  </h3>
  <p className="text-gray-600 mb-4">
    Esta acción notificará al paciente.
  </p>
  <div className="flex gap-3 justify-end">
    <button className="px-4 py-2 text-gray-700 hover:bg-gray-100 rounded-lg">
      No, volver
    </button>
    <button className="px-4 py-2 bg-red-600 text-white hover:bg-red-700 rounded-lg">
      Sí, cancelar cita
    </button>
  </div>
</div>
```

---

## Estados Visuales de Tratamientos

### En Curso
- **Color**: Azul (`bg-blue-50`, `border-blue-500`)
- **Ícono**: ClipboardDocumentCheckIcon
- **Mensaje**: "Sesión X de Y completadas"

### Pendiente Seguimiento
- **Color**: Amarillo (`bg-yellow-50`, `border-yellow-500`)
- **Ícono**: ExclamationTriangleIcon
- **Mensaje**: "Sin próxima cita programada"

### Abandonado
- **Color**: Rojo (`bg-red-50`, `border-red-500`)
- **Ícono**: XCircleIcon
- **Mensaje**: "Sin contacto desde [fecha]"

### Finalizado
- **Color**: Verde (`bg-green-50`, `border-green-500`)
- **Ícono**: CheckCircleIcon
- **Mensaje**: "Tratamiento completado"

---

## Principios de Diseño

### 1. Claridad sobre Creatividad
- Priorizar legibilidad y funcionalidad
- Evitar animaciones excesivas
- Textos directos y concisos

### 2. Jerarquía Visual Clara
- Usar tamaño, peso y color para establecer importancia
- Información crítica siempre visible
- Reducir ruido visual

### 3. Feedback Inmediato
- Confirmar cada acción del usuario
- Estados de carga visibles
- Mensajes de error específicos y útiles

### 4. Accesibilidad
- Contraste mínimo 4.5:1 para texto
- Tamaños de botón ≥44px táctiles
- Navegación por teclado
- Labels descriptivos en formularios

### 5. Consistencia
- Mismos patrones en toda la aplicación
- Componentes reutilizables
- Terminología uniforme

---

## Ejemplos de Aplicación

### Dashboard Card
```tsx
<div className="bg-white rounded-lg shadow-sm border border-gray-200 p-6">
  <div className="flex items-center justify-between mb-4">
    <h3 className="text-lg font-semibold text-gray-900">
      Citas de Hoy
    </h3>
    <CalendarIcon className="h-6 w-6 text-blue-500" />
  </div>
  <div className="text-3xl font-bold text-gray-900 mb-2">
    12
  </div>
  <p className="text-sm text-gray-600">
    8 confirmadas, 4 pendientes
  </p>
</div>
```

### Alerta de Seguimiento
```tsx
<div className="bg-yellow-50 border-l-4 border-yellow-500 p-4 mb-4">
  <div className="flex items-start">
    <ExclamationTriangleIcon className="h-5 w-5 text-yellow-600 mt-0.5 mr-3" />
    <div className="flex-1">
      <h4 className="text-sm font-semibold text-yellow-900 mb-1">
        Paciente requiere seguimiento
      </h4>
      <p className="text-sm text-yellow-800">
        Juan Pérez tiene una endodoncia con 2 sesiones pendientes y sin cita programada.
      </p>
      <button className="mt-2 text-sm font-medium text-yellow-900 hover:text-yellow-700">
        Registrar seguimiento →
      </button>
    </div>
  </div>
</div>
```

---

Esta skill debe aplicarse consistentemente en todos los componentes UI de DentiFlow para mantener una identidad de marca profesional y coherente.
