---
name: generate-use-cases
description: Genera casos de uso/prueba para un proceso ABM migrado, analizando exclusivamente el codigo Delphi fuente y la base de datos. Produce un MD para tests manuales de UI y un JSON parseable para tests automatizados de API.
allowed-tools: Read, Glob, Grep, Agent, Write, mcp__botman-sql__query, mcp__botman-sql__describe_table, mcp__botman-sql__list_tables
user-invocable: true
---

# Generador de Casos de Uso para Migracion ABM

Genera casos de prueba exhaustivos para verificar que un proceso ABM fue migrado correctamente de Delphi a C#.

**Argumento:** `'<DelphiName>' from '<DelphiModulePath>' as '<AxCloudName>'`

Ejemplo: `/generate-use-cases 'RegistracionComprobantesIIBB' from 'IV/IV24' as 'RegistracionComprobantesIIBB'`

---

## Paso 0 — Resolver paths y leer contexto

1. **Resolver paths:**
   - Delphi source: `AxSource/Tango/Source/ProcesosTG/<DelphiModulePath>.PAS` (Legacy) o `AxSource/Astor/Source/<DelphiModulePath>/` (Astor)
   - Si existe `migration-support/extras/<AxCloudName>/reference_delphi/$2_delphiPaths.md`, leerlo para obtener los paths exactos
   - Extras output: `migration-support/extras/<AxCloudName>/reference_delphi/whole_process/UseCases/`

2. **Leer el archivo Delphi completo.** Este es tu unico source of truth para la logica de negocio.

3. **Si existe un worktree** `AxCloud_<AxCloudName>/`, leer SOLO la estructura C# para entender la API (nombres de endpoints, DTOs), pero NO basar las validaciones en el C# — siempre basarlas en el Delphi.

---

## Paso 1 — Analizar el Delphi

Lee el archivo Delphi completo y extrae TODAS las siguientes categorias de logica. Registra cada hallazgo con el numero de linea Delphi donde lo encontraste.

### 1.1 Campos y restricciones
- Calls a `AddFieldES`: extraer nombre del campo, formato (`'3U'`, `'20X'`, etc.), tipo (`EditES`, `NoEmptyES`, `SelectES`, etc.)
- Campos con `NoEmptyES` = requeridos
- Campos con `SelectES` = enum/domain con valores fijos
- Formato: primer digito = max length, letra = tipo (U=uppercase, X=alfanumerico, N=numerico)

### 1.2 Validaciones en CheckField / CheckES
- Validacion de duplicados: `FindKeyOnly`, `Search` + error
- Validacion de rango: comparaciones `Des > Has`
- Validacion de existencia en tabla relacionada: `Found`, `Search`
- Validacion de formato: numerico, alfanumerico, etc.

### 1.3 Validaciones en Save / BeforePost
- Validaciones antes de grabar: campos vacios, combinaciones logicas invalidas
- Validacion de fecha vs periodo cerrado
- Validacion de importe > 0
- Confirmaciones al usuario (`Confirm`)

### 1.4 Control de campos condicionales
- `ChangeAttrES` que cambian la editabilidad de campos segun valores de otros campos
- Campos que se habilitan/deshabilitan segun tipo, estado, etc.
- Campos que se limpian cuando se deshabilitan

### 1.5 Restricciones de eliminacion
- Busquedas en tablas relacionadas antes de eliminar (`Search` en `_IVAxx`, `_STAxx`, etc.)
- Mensajes de error especificos por cada tabla relacionada
- Confirmaciones de eliminacion

### 1.6 Restricciones de modificacion
- Campos que se vuelven no editables en modo modificacion
- Campos protegidos si el registro esta usado en otras tablas
- UK protection

### 1.7 Valores por defecto
- Asignaciones directas en modo Alta (`MnKey = 1`)
- Inicializacion de campos calculados
- SISTEMA_OR, fechas default, etc.

### 1.8 Lookups y filtros
- Filtros aplicados a lookups (ej: excluir ocasionales, excluir inhabilitados)
- Validaciones de existencia en tablas de lookup

### 1.9 Logica de copia
- Si existe una funcion de copiar registro, que campos se copian y cuales se limpian

### 1.10 Parametros del sistema
- Consultas a tablas de parametros (`IVA_PARAMETRO`, `STA_PARAMETRO`, etc.)
- Comportamiento que cambia segun parametros (ej: control de letra E/C/N)

---

## Paso 2 — Consultar la base de datos

Usa el MCP SQL para obtener datos reales que enriquezcan los tests:

1. **Describir la tabla principal** (`describe_table`) para verificar columnas, tipos, nullability
2. **Obtener datos de referencia** necesarios para los tests:
   - `SELECT TOP 5 * FROM <tabla_principal>` — registros existentes para tests de navegacion
   - Tablas de lookup referenciadas: `SELECT TOP 5 ID, descripcion FROM <tabla_lookup> WHERE habilitado = 'S'`
   - Parametros del sistema: `SELECT * FROM <modulo>_PARAMETRO` (solo columnas relevantes al proceso)
   - Fecha de cierre si aplica
   - Conteo de registros existentes
3. **Identificar IDs reales** para usar en los payloads de test (provincias, tipos, clientes, proveedores, etc.)

---

## Paso 3 — Generar los archivos de salida

### 3.1 Archivo: `useCases_UI.md`

Formato markdown legible para testing manual. Cada caso debe tener:

```markdown
# Casos de Prueba UI — <NombreProceso>

## Datos de referencia
<!-- Tabla con parametros actuales de la DB -->

---
## UI-XX — <Titulo descriptivo>

**Objetivo:** <Que se verifica>

**Preparacion SQL:** (si aplica)
```sql
<SQL de setup>
```

**Pasos:**
1. <Paso concreto>
2. <Paso concreto>
...

**Resultado esperado:** <Que debe pasar>

**Restaurar SQL:** (si aplica)
```sql
<SQL de teardown>
```

**Origen Delphi:** Linea XX — `<fragmento de codigo>`
```

#### Categorias obligatorias de tests UI:
1. **Apertura y navegacion** — Verificar que el ABM se abre y muestra datos
2. **Alta exitosa** — Un caso por cada tipo/combinacion principal
3. **Campos requeridos** — Un caso por cada campo obligatorio
4. **Validaciones de negocio** — Cada regla identificada en Paso 1
5. **Campos condicionales** — Cada campo que se habilita/deshabilita
6. **Modificacion** — Campos editables vs protegidos
7. **Eliminacion** — Bloqueo por FK y eliminacion exitosa
8. **Copia** — Si aplica
9. **Parametros** — Tests que requieren cambiar parametros del sistema

### 3.2 Archivo: `useCases_API.json`

Formato JSON parseable para el runner automatizado. Estructura:

```json
{
  "process": "<AxCloudName>",
  "module": "<ModuloAxCloud>",
  "baseUrl": "/api/<modulo>/<proceso>",
  "generatedFrom": "<path al archivo Delphi>",
  "generatedAt": "<fecha ISO>",
  "dbReferenceData": {
    "parametros": { "<param>": "<valor>" },
    "registrosExistentes": <count>,
    "lookupData": {
      "<tabla>": [{ "id": <n>, "descripcion": "<desc>" }]
    }
  },
  "groups": [
    {
      "id": "G1",
      "name": "<Nombre del grupo>",
      "description": "<Que valida este grupo>",
      "setupSql": "<SQL de preparacion o null>",
      "teardownSql": "<SQL de restauracion o null>",
      "tests": [
        {
          "id": "1.1",
          "description": "<Descripcion concisa>",
          "method": "POST",
          "endpoint": "/add",
          "payload": { },
          "expected": {
            "status": 201,
            "errorContains": null
          },
          "delphiReference": {
            "file": "<archivo.PAS>",
            "line": <numero>,
            "code": "<fragmento relevante>"
          }
        },
        {
          "id": "1.2",
          "description": "<Test de error>",
          "method": "POST",
          "endpoint": "/add",
          "payload": { },
          "expected": {
            "status": 400,
            "errorContains": "<fragmento del mensaje de error esperado>"
          },
          "delphiReference": {
            "file": "<archivo.PAS>",
            "line": <numero>,
            "code": "<fragmento relevante>"
          }
        }
      ]
    }
  ],
  "cleanup": {
    "description": "SQL para limpiar todos los registros creados por los tests",
    "sql": "<DELETE FROM ... WHERE OBSERVACIONES LIKE 'Test %'>"
  }
}
```

#### Grupos obligatorios en el JSON:

| ID | Grupo | Que testea |
|----|-------|------------|
| G1 | Altas exitosas | Un POST exitoso por cada combinacion valida principal |
| G2 | Campos requeridos | Un POST con cada campo requerido vacio/null |
| G3 | Validaciones de fecha | Fecha en periodo cerrado, fecha limite, fecha valida |
| G4 | Combinaciones logicas | Combinaciones invalidas de campos dependientes |
| G5 | Formato de campos | Numerico vs alfanumerico, max length |
| G6 | Parametros del sistema | Tests que requieren cambio de parametro (con setup/teardown SQL) |
| G7 | Duplicados | Alta base + intento de duplicado |
| G8 | Modificacion | PUT/modify con campos protegidos cambiados |
| G9 | Eliminacion | DELETE con y sin FK constraints |

> Omitir grupos que no aplican al proceso (ej: si no hay fecha de cierre, omitir G3).

#### Reglas para los payloads:
- Usar IDs reales obtenidos de la DB en Paso 2
- Cada test debe ser independiente (no depender del resultado de otro test del mismo grupo)
- Los tests dentro de un grupo con `setupSql` comparten ese setup
- Incluir `OBSERVACIONES` o campo equivalente con el ID del test para facilitar cleanup
- Los campos del payload deben usar los nombres del DTO C# si se conocen, o los nombres de la tabla SQL

---

## Paso 4 — Escribir los archivos

1. Crear el directorio si no existe:
   ```
   migration-support/extras/<AxCloudName>/reference_delphi/whole_process/UseCases/
   ```

2. Escribir:
   - `useCases_UI.md`
   - `useCases_API.json`

3. Mostrar un resumen al usuario:
   - Total de tests UI generados
   - Total de tests API generados
   - Grupos cubiertos
   - Validaciones del Delphi que NO pudieron traducirse a test API (solo UI)
   - Datos de referencia de la DB usados

---

## Reglas criticas

1. **SOLO basar las validaciones en el codigo Delphi.** El C# es util para saber nombres de endpoints/DTOs, pero la logica de negocio viene del Delphi.
2. **Cada test debe tener su `delphiReference`** con archivo, linea y fragmento de codigo. Si no podes identificar la linea exacta, indicar el bloque.
3. **No inventar validaciones.** Si no esta en el Delphi, no generar test para eso.
4. **Usar datos reales de la DB.** No inventar IDs de lookup, provincias, etc.
5. **Los tests API deben ser ejecutables sin intervencion humana** (excepto los que requieren setup SQL).
6. **Marcar claramente los tests que solo se pueden verificar en UI** (ej: foco de campo, habilitacion visual, confirmaciones modales).

---

Responde al usuario: $ARGUMENTS
