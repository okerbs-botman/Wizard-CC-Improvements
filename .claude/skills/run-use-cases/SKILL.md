---
name: run-use-cases
description: Ejecuta los casos de prueba API generados por generate-use-cases contra el backend corriendo localmente. Reporta resultados, modifica la DB para cubrir todos los escenarios, e indica que tests fallaron para que se corrija la migracion.
allowed-tools: Read, Bash, Write, Edit, mcp__botman-sql__query, mcp__botman-sql__describe_table
user-invocable: true
---

# Runner de Casos de Uso API

Ejecuta los tests API del archivo `useCases_API.json` contra el backend corriendo localmente y genera un reporte de resultados.

**Argumento:** `<AxCloudName>` [con `baseUrl=<url_base>`]

Ejemplo: `/run-use-cases RegistracionComprobantesIIBB`
Ejemplo con URL custom: `/run-use-cases RegistracionComprobantesIIBB con baseUrl=http://localhost:5000`

---

## Paso 0 — Preparacion

1. **Leer el archivo de tests:**
   ```
   migration-support/extras/<AxCloudName>/reference_delphi/whole_process/UseCases/useCases_API.json
   ```
   Si no existe, indicar al usuario que primero debe ejecutar `/generate-use-cases`.

2. **Determinar la URL base:**
   - Si el usuario la proporciono: usar esa
   - Si no: usar `http://localhost:5168` como default
   - Verificar que el backend responde: `curl -s -o /dev/null -w "%{http_code}" <baseUrl>/health` o endpoint similar

3. **Leer el JSON completo** y mostrar al usuario un resumen:
   - Cantidad de grupos
   - Cantidad total de tests
   - Grupos que requieren setup SQL
   - Preguntar confirmacion para continuar

---

## Paso 1 — Ejecutar los tests grupo por grupo

Para cada grupo en el JSON:

### 1.1 Setup SQL
Si el grupo tiene `setupSql`:
- Ejecutar el SQL via MCP SQL tool
- Verificar que se ejecuto correctamente
- Esperar 1 segundo para que la DB se estabilice

### 1.2 Ejecutar cada test del grupo

Para cada test, ejecutar via `curl`:

```bash
# Para POST (add)
curl -s -w "\n%{http_code}" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '<payload_json>' \
  "<baseUrl><endpoint>"

# Para DELETE
curl -s -w "\n%{http_code}" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '<payload_json>' \
  "<baseUrl><endpoint>"
```

### 1.3 Evaluar resultado

Para cada test, comparar:

| Campo esperado | Verificacion |
|---------------|-------------|
| `expected.status` | HTTP status code debe coincidir |
| `expected.errorContains` | Si no es null, el body de respuesta debe contener ese texto |

Registrar resultado como:
- **PASS** — Status y mensaje coinciden con lo esperado
- **FAIL** — Status o mensaje no coinciden
- **ERROR** — No se pudo ejecutar el request (backend caido, timeout, etc.)

Para cada resultado, guardar:
- Test ID y descripcion
- Status HTTP recibido vs esperado
- Body de respuesta (truncado a 500 chars)
- Referencia Delphi del test

### 1.4 Teardown SQL
Si el grupo tiene `teardownSql`:
- Ejecutar el SQL via MCP SQL tool
- Verificar que se ejecuto correctamente

---

## Paso 2 — Generar reporte

Escribir el archivo de resultados en:
```
migration-support/extras/<AxCloudName>/reference_delphi/whole_process/UseCases/testResults.md
```

Formato:

```markdown
# Resultados de Tests API — <NombreProceso>

**Ejecutado:** <fecha y hora>
**Backend:** <baseUrl>
**Total:** XX tests | YY PASS | ZZ FAIL | WW ERROR

## Resumen

| Estado | Cantidad | Porcentaje |
|--------|----------|------------|
| PASS   | YY       | XX%        |
| FAIL   | ZZ       | XX%        |
| ERROR  | WW       | XX%        |

---

## Tests FALLIDOS

### [FAIL] Test X.Y — <Descripcion>

- **Esperado:** Status <code>, error contiene "<texto>"
- **Recibido:** Status <code>
- **Respuesta:** <body truncado>
- **Origen Delphi:** <archivo>:<linea> — `<codigo>`
- **Diagnostico:** <explicacion breve de que puede estar fallando>

---

## Tests EXITOSOS

### [PASS] Test X.Y — <Descripcion>
(listado compacto)

---

## Tests con ERROR

### [ERROR] Test X.Y — <Descripcion>
- **Error:** <descripcion del error>
```

---

## Paso 3 — Analisis y plan de correccion

Si hay tests FALLIDOS:

1. **Agrupar fallos por categoria:**
   - Validaciones faltantes (el C# no valida algo que el Delphi si)
   - Validaciones incorrectas (el C# valida diferente al Delphi)
   - Errores de formato/mapeo (nombres de campo, tipos de dato)
   - Problemas de configuracion (endpoint mal, DTO incompleto)

2. **Para cada fallo, identificar el archivo C# probable** donde deberia estar la validacion:
   - Buscar en `AxCloud_<AxCloudName>/AxCloud/modules/` los archivos del proceso
   - Identificar si es un problema de Helper, Abm, Domain, o Controls

3. **Presentar al usuario un plan de correccion:**
   ```
   Se encontraron X tests fallidos. Plan de correccion:

   1. [FAIL 2.1] Validacion de provincia requerida
      → Falta SetRequiredExpression para ID_PROVINCIA en <Archivo>Abm.cs:SetEntityDomain()
      → Delphi: IV24.PAS:156 — if ID_PROVINCIA = 0 then ErrMsg(...)

   2. [FAIL 4.1] Ventas sin cliente
      → Falta validacion en <Archivo>Helper.cs:ValidarClienteProveedor()
      → Delphi: IV24.PAS:203 — if (Tipo = 'V') and (Cliente = 0) then...

   ¿Procedo a corregir?
   ```

4. **Si el usuario aprueba**, leer los archivos C# relevantes y aplicar las correcciones necesarias siguiendo las guias de migracion.

5. **Despues de corregir**, preguntar si quiere re-ejecutar los tests para verificar.

---

## Paso 4 — Cleanup

Si el JSON tiene seccion `cleanup`:
- Preguntar al usuario si desea limpiar los registros creados por los tests
- Si confirma, ejecutar el SQL de cleanup via MCP

---

## Reglas criticas

1. **No modificar el JSON de tests.** Si un test falla, el problema esta en el C#, no en el test.
2. **Ejecutar tests secuencialmente dentro de cada grupo** (pueden depender del setup SQL del grupo).
3. **Ejecutar grupos secuencialmente** (teardown de uno puede afectar setup del siguiente).
4. **Siempre hacer teardown** incluso si hay tests fallidos en el grupo.
5. **No corregir codigo sin aprobacion del usuario.**
6. **Si el backend no responde**, abortar y pedir al usuario que lo levante.
7. **Guardar siempre el reporte** incluso si todos los tests pasan (sirve como evidencia).

---

Responde al usuario: $ARGUMENTS
