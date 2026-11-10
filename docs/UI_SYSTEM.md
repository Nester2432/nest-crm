# Nest CRM — Sistema de Extensiones de Interfaz (UI Slots)

**Versión del documento**: 2.1.0\
**Fecha de actualización**: 2026-11-09\
**Estado**: Especificación de arquitectura objetivo\
**Clasificación general**: **[OBJETIVO APROBADO / NO IMPLEMENTADA]**\

---

## 1. Convenciones de Clasificación del Documento

- **[ACTUAL COMPROBADA]**: Funcionalidad implementada y verificada en el código.
- **[OBJETIVO APROBADO]**: Decisión arquitectónica aprobada, en especificación o planificación formal.
- **[PENDIENTE DE DECISIÓN]**: Propuesta técnica sujeta a análisis.
- **[NO IMPLEMENTADA]**: Diseñada conceptualmente pero sin código en el repositorio.

---

## 2. Estado Real vs. Arquitectura Objetivo

Para evitar expectativas erróneas sobre el código existente:

### 2.1 Lo que Existe Actualmente en el Código [ACTUAL COMPROBADA]:
1. **Navegación Modular Básica:** En `src/components/layout/Sidebar.tsx`, los elementos declarados en `manifiesto.nav` de los módulos activos se concatenan al final de tres secciones fijas hardcodeadas (`Operaciones`, `Productividad`, `Inteligencia`).
2. **Páginas Aisladas en `/m/[modulo]`:** Los módulos pueden renderizar pantallas completas bajo su propio prefijo de ruta.
3. **No existen puntos de anclaje de UI:** No existe en el código actual ningún mecanismo para inyectar widgets en el dashboard, bloques en fichas de contactos o deals, ni paneles laterales en el chat.

### 2.2 Arquitectura Objetivo de UI Slots [OBJETIVO APROBADO / NO IMPLEMENTADA]:
El **Sistema de UI Slots** es la arquitectura diseñada para permitir que módulos funcionales y plantillas sectoriales extiendan visualmente las pantallas maestras del núcleo **sin modificar el código fuente de los componentes en `src/components/`**.

---

## 3. Inventario de los Siete Slots de Interfaz Oficiales

Se formalizan taxativamente los siguientes siete puntos de anclaje (*UI Slots*) dentro del CRM:

```
┌────────────────────────────────────────────────────────────────────────┐
│                              APP SHELL                                 │
│  ┌────────────────────┬─────────────────────────────────────────────┐  │
│  │  sidebar.sections  │             dashboard.widgets               │  │
│  │                    │                                             │  │
│  │  - Operaciones     │  ┌──────────────┐  ┌─────────────────────┐  │  │
│  │  - Productividad   │  │ Core KPIs    │  │ Módulo Widget (ej.  │  │  │
│  │  - Inteligencia    │  │              │  │ Visitas Pendientes) │  │  │
│  │  - [Slots Módulos] │  └──────────────┘  └─────────────────────┘  │  │
│  │                    ├─────────────────────────────────────────────┤  │
│  │                    │              Vistas de Detalle              │  │
│  │                    │  ┌───────────────────────┬───────────────┐  │  │
│  │                    │  │ contact.detail / tabs │ conversation. │  │  │
│  │                    │  │ deal.detail / tabs    │ sidebar       │  │  │
│  │                    │  │ (Ficha enriquecida)   │ (Panel extra) │  │  │
│  │                    │  └───────────────────────┴───────────────┘  │  │
│  └────────────────────┴─────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

| Identificador de Slot | Ubicación en la Aplicación | Propósito Funcional | Props Inyectadas por el Núcleo |
|---|---|---|---|
| `dashboard.widgets` | Pantalla principal (`/`, `src/app/page.tsx`). | Tarjetas de métricas, accesos directos o listas resumidas de un módulo. | `{ member, dateRange }` |
| `sidebar.sections` | Menú lateral (`src/components/layout/Sidebar.tsx`). | Secciones y enlaces de navegación hacia las vistas principales de los módulos. | `{ member, pathname }` |
| `contact.detail` | Columna principal de la ficha del contacto (`/contacts/[id]`). | Bloques de información sectorial vinculada al prospecto (ej. requerimiento de inmueble). | `{ contact, member }` |
| `contact.tabs` | Pestañas de la ficha del contacto (`/contacts/[id]`). | Subvistas dedicadas de gestión profunda (ej. historial de visitas del prospecto). | `{ contact, member }` |
| `deal.detail` | Vista de oportunidad comercial (`/pipeline` modal o página de deal). | Atributos comerciales específicos del nicho (ej. propiedad asociada, tasación). | `{ deal, contact, member }` |
| `deal.tabs` | Pestañas de la ficha de oportunidad comercial. | Vistas relacionales del negocio (ej. cotizaciones generadas, contratos). | `{ deal, member }` |
| `conversation.sidebar` | Panel lateral derecho en la bandeja de chat (`/conversations/[id]`). | Herramientas operativas durante la conversación (ej. buscador de catálogo o agenda directa). | `{ conversation, contact, member }` |

---

## 4. Contratos de Componentes para Slots [OBJETIVO APROBADO]

Todo componente exportado por un módulo para ser utilizado en un UI Slot deberá ajustarse a contratos tipados:

```typescript
import type { ComponentType } from "react";
import type { TeamMember } from "@/lib/team";

/** Props base comunes a todos los slots */
export interface BaseSlotProps {
  member: TeamMember;
}

/** Props para widgets del dashboard */
export interface DashboardWidgetProps extends BaseSlotProps {
  config?: Record<string, unknown>;
}

/** Props para slots contextuales a un contacto */
export interface ContactSlotProps extends BaseSlotProps {
  contactId: string;
  contactData: {
    name: string;
    email?: string | null;
    phone?: string | null;
    customFields: Record<string, unknown>;
  };
}

/** Props para slots contextuales a una oportunidad comercial */
export interface DealSlotProps extends BaseSlotProps {
  dealId: string;
  dealData: {
    title: string;
    value: number;
    stageId: string;
    contactId: string;
  };
}

/** Props para el menú lateral (sidebar.sections) */
export interface SidebarSectionProps extends BaseSlotProps {
  pathname: string;
}

/** Props para el panel lateral de conversación */
export interface ConversationSidebarProps extends BaseSlotProps {
  conversationId: string;
  contactId: string;
  channel: "whatsapp" | "instagram" | "facebook";
}

/**
 * Mapa canónico que vincula cada identificador oficial de Slot con su interfaz de props.
 * Garantiza que cada componente declare y reciba exclusivamente las props correspondientes a su anclaje.
 */
export interface SlotPropsMap {
  "dashboard.widgets": DashboardWidgetProps;
  "sidebar.sections": SidebarSectionProps;
  "contact.detail": ContactSlotProps;
  "contact.tabs": ContactSlotProps;
  "deal.detail": DealSlotProps;
  "deal.tabs": DealSlotProps;
  "conversation.sidebar": ConversationSidebarProps;
}

export type SlotId = keyof SlotPropsMap;

/**
 * Representación conceptual fuertemente tipada de una extensión registrada.
 * Parametrizada por el identificador del Slot para evitar 'ComponentType<any>'.
 */
export interface RegisteredSlotExtension<S extends SlotId = SlotId> {
  id: string;                 // ej. "propiedades.buscador_rapido"
  slot: S;                    // Identificador exacto del slot
  title: string;              // Título para pestañas, encabezados o accesibilidad
  component: ComponentType<SlotPropsMap[S]>; // Componente tipado con las props exactas del slot
  rolesPermitidos?: ("admin" | "asesor")[];
  tamanoDefault?: "sm" | "md" | "lg" | "full";
}

/** Unión discriminada para colecciones heterogéneas de extensiones */
export type AnyRegisteredSlotExtension = {
  [K in SlotId]: RegisteredSlotExtension<K>;
}[SlotId];
```

> [!NOTE]
> **Decisión Técnica Pendiente sobre Inferencia de Colecciones Heterogéneas [PENDIENTE DE DECISIÓN]:**\
> Si bien `RegisteredSlotExtension<S>` y `AnyRegisteredSlotExtension` resuelven formalmente la asociación estricta entre slot y props a nivel conceptual, la implementación de un despachador genérico sin pérdida de inferencia requerirá una evaluación técnica específica durante la Fase 3 del Roadmap para validar la compatibilidad con Server Components de Next.js y empaquetado dinámico sin recurrir a tipos inseguros.

---

## 5. Principios de Seguridad y Aislamiento en UI Slots [OBJETIVO APROBADO]

La implementación de UI Slots deberá respetar estrictamente las siguientes garantías técnicas:

1. **La Autorización en Servidor es Innegociable:**
   - El filtrado visual por rol en el frontend (`rolesPermitidos`) es una optimización de experiencia de usuario, **no una barrera de seguridad**.
   - Todo endpoint de API (`/api/m/[modulo]/*`) o Server Action que alimente un componente de slot **debe verificar obligatoriamente la sesión y los permisos en el servidor** (`requireMember()`, `requireAdmin()`, `ownerFilter()`).
2. **`Suspense` no Reemplaza a `ErrorBoundary`:**
   - `Suspense` gestiona exclusivamente estados de carga asíncrona (*fallbacks* y *skeletons*).
   - Cada slot debe estar obligatoriamente encapsulado en un **`ErrorBoundary` dedicado** para que, si un componente de módulo arroja una excepción no controlada en tiempo de ejecución, el fallo quede confinado a ese widget y **no derribe la pantalla principal del CRM**.
3. **Soberanía del Núcleo en el Renderizado:**
   - El núcleo decide cuándo, cómo y dónde se renderiza cada slot.
   - Los módulos no pueden manipular el DOM fuera del contenedor de su slot ni interferir con la navegación global.
4. **Prohibición de Código Arbitrario:**
   - Los módulos no pueden inyectar scripts, cadenas ejecutables (`eval`) ni componentes dinámicos no validados desde la base de datos o archivos de configuración. Todo componente debe estar fuertemente tipado e importado desde el código compilado del repositorio.
