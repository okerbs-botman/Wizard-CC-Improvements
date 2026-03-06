### Fase 1 — Inicialización

/migration-init
Crea worktree + estructura de extras.

### Fase 2 — Generación de casos de uso

/generate-use-cases '<DelphiName>' from '<DelphiModulePath>' as '<AxCloudName>'
Un contexto limpio lee exclusivamente el Delphi + consulta la DB real. Genera:

useCases_UI.md — Checklist manual para el tester
useCases_API.json — Tests automatizables con payloads HTTP
Ejemplo:


/generate-use-cases 'RegistracionComprobantesIIBB' from 'IV/IV24' as 'RegistracionComprobantesIIBB'


### Fase 3 — Migración

/abm-data-model '<DelphiName>' from '<DelphiModulePath>' as '<AxCloudName>'
/abm-controls '<DelphiName>' from '<DelphiModulePath>' as '<AxCloudName>'
/abm-services '<DelphiName>' from '<DelphiModulePath>' as '<AxCloudName>'
/abm-tests <AxCloudName>
Cada vez que Claude edita un .cs, el hook determinístico corre automáticamente y le avisa si usó var, desordenó visibilidad, olvidó braces, etc. Claude corrige en el momento sin que intervengas.

Ejemplo:


/abm-data-model 'RegistracionComprobantesIIBB' from 'IV/IV24' as 'RegistracionComprobantesIIBB'
/abm-controls 'RegistracionComprobantesIIBB' from 'IV/IV24' as 'RegistracionComprobantesIIBB'
/abm-services 'RegistracionComprobantesIIBB' from 'IV/IV24' as 'RegistracionComprobantesIIBB'
/abm-tests RegistracionComprobantesIIBB


### Fase 4 — Verificación automática

/run-use-cases <AxCloudName>
Con el backend levantado:

Lee el useCases_API.json
Ejecuta setup SQL → HTTP calls → teardown SQL por grupo
Compara status codes y mensajes de error
Genera testResults.md con PASS/FAIL
Si hay FAILs → propone plan de corrección identificando qué archivo C# tiene el bug
Corrige → re-ejecuta → loop hasta que pase todo
En paralelo, el tester humano recorre useCases_UI.md para lo que no se puede probar por API.

Ejemplo:


/run-use-cases RegistracionComprobantesIIBB


### Fase 5 — Commit
Checklists + commit cuando todo pasa.

Resumen visual

                    /generate-use-cases (contexto limpio)
                              │
                    ┌─────────┴─────────┐
                    │                   │
              useCases_API.json    useCases_UI.md
                    │                   │
                    │                   ▼
                    │            Tester humano
                    │
    /migrate-abm ───┤
    (hook corrige    │
     estilo en cada  │
     Edit de .cs)    │
                    │
                    ▼
              /run-use-cases
              (API automático)
                    │
               ¿FAIL? ──→ Corrige C# ──→ Re-ejecuta
                    │
                    ▼
                 Commit
