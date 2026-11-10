# Nest CRM — Estrategia de Personalización y Protección de Cambios

**Versión del documento**: 2.1.0\
**Fecha de actualización**: 2026-11-09\
**Estado**: Documento rector de la personalización de instalaciones\

---

## 1. Convenciones de Clasificación del Documento

- **[ACTUAL COMPROBADA]**: Funcionalidad implementada y verificada en el código.
- **[OBJETIVO APROBADO]**: Decisión arquitectónica aprobada, en especificación o planificación formal.
- **[PENDIENTE DE DECISIÓN]**: Propuesta técnica sujeta a análisis.
- **[NO IMPLEMENTADA]**: Diseñada conceptualmente pero sin código en el repositorio.

---

## 2. Misión de la Personalización Sostenible

La personalización sostenible en Nest CRM garantiza que una empresa cliente pueda adaptar su identidad, procesos y vocabulario de negocio **sin que dichas modificaciones se pierdan al actualizar el núcleo o la plantilla sectorial**, y sin requerir la modificación del código fuente del producto (`src/`).

```
┌─────────────────────────────────────────────────────────────┐
│                 ACTUALIZACIÓN DEL NÚCLEO                    │
│      (Nuevas funciones, parches de seguridad, optimización) │
└──────────────────────────────┬──────────────────────────────┘
                               │ Fluye sin conflictos
                               ▼
┌─────────────────────────────────────────────────────────────┐
│            PERSONALIZACIÓN DE LA INSTALACIÓN                │
│    ✓ Marca propia (Logo, colores corporativos)              │
│    ✓ Nombres de etapas del pipeline y orden particular      │
│    ✓ Campos adicionales propios del negocio                 │
│    ✓ Instrucciones y tono personalizado del agente IA       │
│    ✓ Módulos específicos habilitados                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Mecanismos Existentes en el Código [ACTUAL COMPROBADA]

El núcleo actual ya incorpora patrones arquitectónicos diseñados para proteger la personalización:

### 3.1 Claves Técnicas Inmutables vs. Nombres Visuales [ACTUAL COMPROBADA]
- En las etapas del pipeline (`pipeline_stages`), existe una separación entre:
  1. `key`: Identificador técnico inmutable (ej. `nuevo`, `calificado`, `visita_agendada`).
  2. `name`: Texto descriptivo visible para el usuario en el tablero Kanban.
- **Regla:** El usuario puede renombrar libremente la etiqueta visual de una etapa. Las automatizaciones y las herramientas del agente de IA (`mover_etapa`) continúan funcionando de forma inalterada porque operan sobre la columna `key`.

### 3.2 Contenedor Semiestructurado para Datos Dinámicos (`custom_fields`) [ACTUAL COMPROBADA]
- En la tabla `contacts`, los campos personalizados definidos por la empresa o por los módulos activos no requieren sentencias `ALTER TABLE` relacionales; se persisten en la columna semiestructurada `custom_fields` de tipo `JSONB`.
- Esto garantiza que nuevas versiones del núcleo nunca colisionen con columnas creadas arbitrariamente por clientes.

### 3.3 Variables CSS Dinámicas en Runtime [ACTUAL COMPROBADA]
- En `src/lib/config/index.ts`, la función `cssVariablesMarca()` calcula los colores primarios y del sidebar en tiempo de ejecución (`SSR`) y los inyecta en el bloque `<head>` del layout raíz, evitando flash visual y sin modificar archivos CSS estáticos.

---

## 4. Protección de Personalizaciones por Procedencia [OBJETIVO APROBADO / NO IMPLEMENTADA]

Para que una instalación pueda actualizar la versión de su plantilla sectorial o del núcleo sin destruir cambios particulares:

### 4.1 Identificación por Procedencia (*Origin Tracking*)
- La configuración de una instalación no requiere dividirse físicamente en múltiples tablas de base de datos; basta con estructurar los registros de `crm_settings` asociando a cada propiedad una etiqueta de procedencia:
  - `origin: "core"`: Valor base inmutable.
  - `origin: "template"`: Valor sugerido por la plantilla sectorial.
  - `origin: "client"`: Valor modificado o creado explícitamente por el administrador de la empresa.

### 4.2 Reglas de Reconciliación y Límites de la Inmunidad [OBJETIVO APROBADO / NO IMPLEMENTADA]

> **Principio Rector:** Cuando se aplica una actualización de plantilla o del núcleo, los registros con `origin: "client"` **no se sobrescriben automáticamente por valores por defecto**.

Sin embargo, **preservar un valor personalizado no garantiza por sí solo que sea compatible con la nueva versión**:
1. **Límites de la Inmunidad Automática:**
   - Si una plantilla actualizada modifica la lógica del flujo comercial, los tipos de datos o las herramientas del agente, mantener un valor antiguo del cliente puede generar incompatibilidades semánticas o fallos de validación.
   - Por tanto, el sistema **no asume compatibilidad ciega**: ante discrepancias estructurales, suspende la aplicación automática y exige **revisión y confirmación administrativa**.
2. **Campos Eliminados o Renombrados por una Nueva Versión:**
   - Si una nueva versión de la plantilla retira un campo que la empresa utilizaba, o lo renombra, el sistema **no elimina los datos existentes** en `contacts.custom_fields`.
   - El campo se mantiene en modo "preservado/legado" accesible en la ficha, pero se notifica al administrador para que decida si lo mapea a un campo nuevo o lo conserva como atributo histórico.
3. **Cambios Incompatibles de Tipo de Datos:**
   - Si un campo personalizado definido previamente como `texto` es redefinido por una nueva plantilla como `numero` o `seleccion` con opciones cerradas, la actualización relacional directa provocaría errores de validación.
   - El sistema detecta la incompatibilidad de tipos, mantiene el valor existente en formato texto y solicita al administrador un procedimiento de **migración de datos asistida** o la asignación de una clave técnica alternativa.
4. **Etapas Utilizadas por Oportunidades Abiertas:**
   - Si una actualización de plantilla retira o reestructura etapas del embudo comercial, pero existen oportunidades (`deals`) activas en ellas, **se bloquea cualquier eliminación de la etapa**.
   - El administrador debe realizar un mapeo guiado transfiriendo los deals abiertos hacia las nuevas etapas antes de consolidar el cambio.
5. **Puntos de Control y Respaldo Previo:**
   - Ante cualquier operación de actualización que involucre reestructuración de esquemas o reconciliación de conflictos, el sistema exige generar un punto de respaldo de la base de datos y confirmación administrativa explícita de los cambios propuestos.

> [!NOTE]
> **Estado de Implementación:** El algoritmo formal de reconciliación por procedencia, detección de incompatibilidades semánticas y resolución guiada de conflictos es un **[OBJETIVO APROBADO / NO IMPLEMENTADA]**. En el código actual comprobado, la configuración es estática y se define en archivos locales (`config/cliente.ts`).

---

## 5. Matriz de Resolución de Cambios y Conflictos

| Escenario de Cambio | Comportamiento Técnico Previsto | Clasificación | Acción Requerida |
|---|---|---|---|
| **Renombrar una etapa comercial** | Se modifica `pipeline_stages.name`. La columna inmutable `key` no cambia. | **[ACTUAL COMPROBADA]** | Ninguna. Automatizaciones y tools de IA siguen operando con la `key`. |
| **Añadir campos de contacto extra** | Se registran en la configuración activa y se guardan dentro de `contacts.custom_fields`. | **[ACTUAL COMPROBADA]** | Ninguna alteración relacional requerida. |
| **Actualizar paleta cromática** | Las variables CSS (`--primary`, `--sidebar`) se inyectan dinámicamente en el layout. | **[ACTUAL COMPROBADA]** | Se inyecta en tiempo de ejecución sin tocar `globals.css`. |
| **Actualización de plantilla con nueva etapa** | El sistema detecta la nueva `key` técnica y la inserta al pipeline sin alterar el orden de las existentes. | **[OBJETIVO APROBADO]** | El administrador revisa y posiciona la nueva etapa en el tablero. |
| **Desactivar etapa con deals abiertos** | El sistema bloquea el borrado físico para evitar pérdida de oportunidades comerciales. | **[ACTUAL COMPROBADA]** | Reasignación previa de las operaciones comerciales a otra etapa. |
| **Cambio de tipo en campo existente (ej. texto a número)** | El validador detecta incompatibilidad de esquema en datos históricos de `custom_fields`. | **[OBJETIVO APROBADO / NO IMPLEMENTADA]** | Revisión administrativa y ejecución de script de migración/normalización de datos. |
| **Campo retirado por nueva versión de plantilla** | Los datos históricos en `custom_fields` se preservan; el campo pasa a estado legado. | **[OBJETIVO APROBADO / NO IMPLEMENTADA]** | Confirmación administrativa para mapeo o archivo del campo. |
| **Colisión de claves entre módulo y campo propio** | El validador de configuración detecta la duplicidad de `clave` y emite error descriptivo. | **[ACTUAL COMPROBADA]** | El implementador renombra la clave técnica conflictiva. |
