# Nest CRM — Sistema de Módulos Funcionales (Modules)

**Versión del documento**: 2.1.0\
**Fecha de actualización**: 2026-11-09\
**Estado**: Documento rector del sistema de módulos\
**Ubicación de referencia**: `modules/` y `src/lib/modules/`\

---

## 1. Convenciones de Clasificación del Documento

- **[ACTUAL COMPROBADA]**: Funcionalidad o arquitectura implementada y comprobada en el código.
- **[OBJETIVO APROBADO]**: Decisión arquitectónica aprobada, en especificación o planificación formal.
- **[PENDIENTE DE DECISIÓN]**: Propuesta técnica sujeta a análisis.
- **[NO IMPLEMENTADA]**: Diseñada conceptualmente pero sin código en el repositorio.

---

## 2. Concepto y Rol de los Módulos

Un **Módulo Funcional** en Nest CRM es un paquete autocontenido de software que extiende las capacidades del sistema en una vertical específica (ej. catálogo de inmuebles en `modules/propiedades`, agenda de visitas presenciales en `modules/visitas`).

### Principios Fundamentales:
1. **Reutilización Intersectorial [ACTUAL COMPROBADA]:** Diferentes plantillas pueden requerir los mismos módulos (ej. `visitas` puede emplearse en inmobiliarias, talleres o consultorios).
2. **Aislamiento del Núcleo [ACTUAL COMPROBADA]:** Los módulos residen en `modules/<id>/` y no modifican archivos en `src/`.
3. **Preservación Estricta de Datos [ACTUAL COMPROBADA en comportamiento / OBJETIVO APROBADO en política]:** Desactivar un módulo **NUNCA elimina sus tablas, columnas ni registros históricos de la base de datos**.

---

## 3. Estado Real del Contrato de Módulos

Es imprescindible distinguir entre el contrato que existe en el código hoy y el contrato formalizado para la arquitectura objetivo:

### 3.1 Contrato Actual Comprobado en Código [ACTUAL COMPROBADA]
En `src/lib/modules/types.ts`, el sistema define actualmente dos interfaces:
- `ManifiestoModulo`: Metadatos del navegador (`nombre`, `titulo`, `descripcion`, `version`, `nav`, `campos`).
- `ModuloServidor`: Capacidades del servidor (`manifiesto`, `paginas`, `api`, `tools`, `seed`).

### 3.2 Contrato Objetivo Unificado `ModuleDefinition` [OBJETIVO APROBADO]
Evolución formalizada para dotar a los módulos de gestión de dependencias transitivas, grafo acíclico, capacidades explícitas y metadatos de persistencia:

```typescript
export interface ModuleDefinition {
  /** Identificador técnico único en snake_case (ej. "propiedades", "visitas") */
  id: string;

  /** Nombre comercial legible en la interfaz */
  name: string;

  /** Descripción concisa de la funcionalidad aportada */
  description: string;

  /** Versión semántica del módulo (ej. "1.1.0") */
  version: string;

  /** Compatibilidad semver con el núcleo */
  compatibility: {
    coreMinVersion: string; // ej. ">=2.1.0"
    coreMaxVersion?: string;
  };

  /** Dependencias entre módulos */
  dependencies: {
    /** Módulos que deben estar previamente activos en la instalación */
    required: string[];
    /** Módulos con los que se integra opcionalmente si están presentes */
    optional: string[];
  };

  /** Capacidades declaradas explícitamente */
  capabilities: {
    hasPages: boolean;         // Expone páginas bajo /m/[id]/*
    hasApi: boolean;           // Expone endpoints bajo /api/m/[id]/*
    hasAgentTools: boolean;    // Registra herramientas para el LLM
    hasUiSlots: boolean;       // Exporta componentes para UI slots
    hasAutomations: boolean;   // Registra disparadores o acciones
    hasReports: boolean;       // Aporta métricas a la sección de reportes
  };

  /** Gestión de persistencia y esquema propio */
  database: {
    /** Directorio relativo de scripts SQL */
    migrationsDir: string;
    /** Versión secuencial del esquema del módulo */
    schemaVersion: number;
    /** Tablas pertenecientes al módulo */
    tables: string[];
  };
}
```

---

## 4. Gestión de Dependencias entre Módulos [OBJETIVO APROBADO / NO IMPLEMENTADA]

El contrato `ModuleDefinition` introduce la gestión formal de dependencias:

1. **Dependencias Directas:**
   - Si el módulo `visitas` declara `dependencies.required: ["propiedades"]`, el CRM no permitirá activar `visitas` a menos que `propiedades` esté activo en la instalación.
2. **Dependencias Transitivas y Detección de Ciclos (DAG):**
   - El validador de módulos evalúa el grafo de dependencias como un **Grafo Acíclico Dirigido (*Directed Acyclic Graph - DAG*)**.
   - Si se detecta una dependencia circular (ej. Módulo A requiere Módulo B, y Módulo B requiere Módulo A), la activación se rechaza inmediatamente con error descriptivo.
3. **Orden Secuencial de Activación y Migración:**
   - Los módulos se activan y sus migraciones SQL se aplican respetando estrictamente el orden topológico del DAG (primero las dependencias base, luego los módulos dependientes).

---

## 5. Estructura Interna de un Módulo [ACTUAL COMPROBADA]

Todo módulo en `modules/<id>/` respeta la siguiente organización de archivos:

```
modules/<id>/
├── manifiesto.ts       # Metadatos del módulo (seguro para el navegador)
├── servidor.tsx        # Páginas, APIs, tools de IA y seed (exclusivo servidor)
├── tabla.ts            # Esquemas de base de datos en Drizzle ORM
├── tipos.ts            # Definiciones de TypeScript internas del módulo
├── componentes/        # Componentes React propios de las vistas del módulo
├── migrations/         # Scripts de migración SQL secuenciales (0000_*.sql)
└── README.md           # Documentación técnica y variables de entorno
```

### Separación Estricta Navegador / Servidor [ACTUAL COMPROBADA]:
- `manifiesto.ts`: No importa código de base de datos, secretos ni librerías pesadas. Es consumido de forma segura por el cliente.
- `servidor.tsx`: Contiene `import "server-only"`. Resuelve peticiones de base de datos, consultas Drizzle y llamadas LLM.

---

## 6. Ciclo de Vida: Activación, Desactivación y Persistencia

### 6.1 Activación de un Módulo

| Etapa | Estado Actual Comprobado [ACTUAL COMPROBADA] | Estado Objetivo Aprobado [OBJETIVO APROBADO / NO IMPLEMENTADA] |
|---|---|---|
| **Mecanismo / Disparador** | Edición manual de `config/modulos.json` (`"activos": [...]`). Requiere reconstrucción de artefacto (`build`). | Acción interactiva desde el panel de administración del CRM (`POST /api/settings/modules/[id]/toggle`) en tiempo de ejecución. |
| **Generación de Registro** | `scripts/generar-registro-modulos.mjs` genera imports estáticos en tiempo de compilación. | Registro dinámico basado en catálogo compilado con conmutación en runtime. |
| **Validación de Código** | Comprueba la existencia física de `manifiesto.ts` y `servidor.tsx`. | Comprueba que el módulo esté presente en el artefacto de ejecución y compatible con el núcleo. |
| **Validación de Dependencias** | No implementada. | Validación estricta del grafo acíclico dirigido (DAG) de dependencias (`dependencies.required`). |
| **Migraciones SQL** | `scripts/migrar.mjs` aplica `modules/<activo>/migrations/*.sql` durante el arranque. | Ejecución transaccional automática de migraciones pendientes del módulo al activarlo. |
| **Persistencia de Estado** | Archivo estático versionado `config/modulos.json`. | Base de datos de la instalación (`crm_settings.modules_active_list`). |

> [!IMPORTANT]
> **Aclaración sobre la activación en el CRM:**
> La activación y desactivación dinámica en tiempo de ejecución desde la interfaz web es un **[OBJETIVO APROBADO / NO IMPLEMENTADA]**. En la arquitectura actual comprobada, no existe conmutación en caliente; la inclusión de un módulo requiere su presencia en `config/modulos.json`, la ejecución del generador de registro y el build o reinicio del proceso Next.js.

### 6.2 Desactivación de un Módulo y Conservación de Datos

> [!CAUTION]
> **REGLA ARQUITECTÓNICA INNEGOCIABLE:**\
> **La desactivación de un módulo NUNCA elimina sus tablas, columnas ni datos de la base de datos.**

#### A. Comportamiento Actual Comprobado [ACTUAL COMPROBADA]:
1. **Procedimiento de Desactivación:** Se elimina el identificador del módulo de la lista `"activos"` en `config/modulos.json`.
2. **Efecto tras Recompilación / Reinicio:**
   - `scripts/generar-registro-modulos.mjs` excluye el módulo de `registro.generated.ts` y `registro-cliente.generated.ts`.
   - Al no formar parte del registro compilado, sus rutas en `/m/[id]/*` y `/api/m/[id]/*` son interceptadas por el despachador dinámico respondiendo **404 Not Found**.
   - El módulo no se incluye en el menú de navegación (`Sidebar.tsx`).
   - Sus herramientas de IA no son registradas en el agente conversacional.
   - `scripts/migrar.mjs` deja de listar o aplicar migraciones del módulo en arranques posteriores.
3. **Persistencia Garantizada:** Las tablas PostgreSQL asociadas al módulo (ej. `propiedades`, `visitas`) y todos sus registros permanecen **estrictamente intactos** en la base de datos.
4. **Limitación Actual:** Las rutas y componentes **no se desactivan de forma inmediata en caliente sin recompilación o reinicio**, dado que la resolución de módulos activos depende del registro generado estáticamente.

#### B. Comportamiento Objetivo Aprobado [OBJETIVO APROBADO / NO IMPLEMENTADA]:
1. **Desactivación en Runtime:** El administrador conmuta el estado del módulo desde la interfaz del CRM sin requerir reconstrucción de la imagen Docker ni reinicio de la instancia.
2. **Ocultamiento Inmediato:** El despachador dinámico (`/m/*`, `/api/m/*`), el agente de IA y los UI Slots consultan el estado activo en base de datos (`crm_settings`) y ocultan inmediatamente las interfaces y herramientas.
3. **Reactivación Controlada:** Al volver a activar el módulo, el sistema valida que las dependencias continúen cumpliéndose y restablece el acceso a los datos históricos sin pérdida de información ni re-migración innecesaria.
