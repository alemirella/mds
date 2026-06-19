# Informe técnico integral del sistema SGOHA

## a. Análisis SonarQube

### Concepto del apartado

SonarQube permite medir de forma integral la calidad de código (bugs, smells, duplicación, deuda técnica, seguridad estática y Quality Gate) sobre `frontend` y `backend`, usando cobertura LCOV como entrada.

### Subapartados(concepto + propósito + logro)

| Subapartado | Qué hace | Qué se hizo en SGOHA | Logro |
| ---------- | -------- | -------------------- | ----- |
| Configuración | Define alcance de análisis | `sonar-project.properties` validado | ✅ Fuentes/tests/cobertura correctamente mapeadas |
| Entorno local | Habilita Sonar reproducible | `docker-compose.sonar.yml` con Postgres | ✅ Stack local operativo |
| Ejecución | Corre análisis estático real | Scanner dockerizado contra `localhost:9000` | ✅ Reporte subido a Sonar |
| Métricas | Entrega KPIs de calidad | API de `measures`, `issues`, `quality gate` consultadas | ✅ Métricas trazables y verificadas |
| Integración | Inserta control en pipeline | `sonar.yml` condicionado por secretos | ⚙️ Listo para CI continuo |

### Configuración validada

- Archivo: [`sonar-project.properties`](../sonar-project.properties)
- Fuentes: `frontend/src`, `backend/src`
- Tests: `tests`, `cypress`
- Cobertura: `tests/reports/coverage/*/lcov.info`
- Exclusiones: `node_modules`, `dist`, `build`, `coverage`, artefactos y evidencias

### Entorno local reproducible

- Archivo: [`docker-compose.sonar.yml`](../docker-compose.sonar.yml)
- Servicios: SonarQube Community + PostgreSQL

### Evidencia técnica local disponible para Sonar

| Insumo | Estado |
| ------ | ------ |
| `frontend-quality.txt` | ✅ Generado |
| `backend-quality.txt` | ✅ Generado |
| `coverage-summary.md` | ✅ Generado (30,3 % líneas) |
| LCOV por suite | ✅ Generado |
| `SONARQUBE_LOCAL_EXECUTION.md` | ✅ Registro de ejecución real y métricas API |

### Verificación explícita de ejecución SonarQube (real)

| Verificación | Resultado real | Diagnóstico |
| ------------ | -------------- | ----------- |
| `docker compose -f docker-compose.sonar.yml up -d` | servicios levantados | ✅ SonarQube y PostgreSQL iniciados |
| `docker compose -f docker-compose.sonar.yml ps` | `sgoha-sonar-db` healthy / `sgoha-sonarqube` up | ✅ Stack operativo |
| `curl http://localhost:9000/api/system/status` | `status=UP` | ✅ API de SonarQube disponible |
| `docker run ... sonarsource/sonar-scanner-cli ...` | `ANALYSIS SUCCESSFUL` | ✅ Análisis ejecutado sobre `sgoha` |
| `api/qualitygates/project_status?projectKey=sgoha` | `status: OK` | ✅ Quality Gate consultado |
| Workflow `.github/workflows/sonar.yml` | condicionado por `SONAR_TOKEN` | ⚙️ En CI depende de secretos |

### Estado Sonar al cierre

- ✅ **Ejecución local completada** con análisis real y resultados consultables en dashboard local.
- ⚙️ **Ejecución CI preparada**; en GitHub seguirá dependiendo de `SONAR_TOKEN`/`SONAR_HOST_URL` o `SONAR_ORGANIZATION`.

### Métricas Sonar (matriz del apartado)

| Métrica | Resultado inicial | Resultado posterior | Estado | Interpretación | Evidencia |
| ------- | ----------------: | ------------------: | ------ | -------------- | --------- |
| Quality Gate | No medido | `OK` | 🟢 Conforme | El proyecto pasa el gate configurado en Sonar | API `project_status` |
| Bugs | No medido | 4 | 🟡 Observación | Defectos de confiabilidad por priorizar | API `measures` |
| Vulnerabilities | No medido | 3 | 🟠 Riesgo medio | Hallazgos de seguridad en código a corregir | API `measures` |
| Security Hotspots | No medido | 7 | 🟡 Revisión | Requiere validación manual en panel | API `measures` |
| Code Smells | No medido | 777 | 🟠 Deuda técnica | Alto volumen de mantenibilidad | API `measures` |
| Duplicated Lines Density | No medido | 1,4 % | 🟢 Controlado | Duplicación baja relativa | API `measures` |
| Coverage (Sonar) | No medido | 15,5 % | 🟠 Riesgo alto | Cobertura efectiva en fuentes analizadas aún baja | API `measures` + LCOV |
| Reliability Rating | No medido | 4.0 (D) | 🟠 Riesgo medio | Debe mejorar corrección de bugs | API `measures` |
| Security Rating | No medido | 5.0 (E) | 🔴 Riesgo alto | Priorizar vulnerabilidades/hotspots | API `measures` |
| Maintainability Rating | No medido | 1.0 (A) | 🟢 Conforme | Deuda relativa mantenible pese a smells | API `measures` |
| Technical Debt (`sqale_index`) | No medido | 3794 | 🟡 Observación | Deuda acumulada a planificar por sprint | API `measures` |
| Cognitive Complexity | No medido | 1555 | 🟠 Riesgo medio | Complejidad alta en módulos extensos | API `measures` |
| Issues totales | No medido | 784 | 🟡 Observación | Requiere estrategia de priorización por severidad | API `issues/search` |

### Evidencias

Ver detalle completo en [`Capturas`](./COVERAGE_ANALYSIS.md).

### Conclusión

Se logró el ciclo completo del apartado Sonar: **configuración validada, stack local levantado, análisis ejecutado y métricas obtenidas**.  
Resultado final del apartado: ✅ SonarQube funcional en local y listo para continuidad en CI con secretos.

---

## b. Interpretación de métricas

### Concepto del apartado

No se trata de listar números aislados, sino de explicar su **impacto técnico real en SGOHA**: estabilidad funcional, riesgo de regresión, seguridad operativa y mantenibilidad.

### Subapartados (concepto + propósito + logro)

| Subapartado | Qué evalúa | Resultado logrado |
| ---------- | ---------- | ----------------- |
| Calidad | Bugs/smells/complejidad/deuda | ✅ Interpretación conectada a módulos críticos |
| Confiabilidad | tests, cobertura, ramas | ✅ Riesgo funcional cuantificado para matrícula/horarios |
| Seguridad | vuln deps + análisis estático | ✅ Riesgo técnico priorizado por severidad |
| Accesibilidad | violaciones automáticas + checklist | ✅ Estado mixto (automatizado + humano) documentado |
| Usabilidad | instrumento SUS y lectura de puntajes | ✅ Infraestructura de medición completada |
| CI/CD | salud del pipeline | ✅ Trazabilidad técnica en workflows |

### Lectura técnica de indicadores principales

| Métrica | Resultado | Interpretación aplicada a SGOHA |
| ------- | --------- | -------------------------------- |
| Pruebas ejecutadas | 208 | 🟢 Base de regresión sólida para flujos clave |
| Cobertura líneas | 30,3 % | 🟠 Riesgo medio-alto en módulos complejos (CSP/horarios) |
| ESLint frontend | 0 errores | 🟢 Mantenibilidad mínima garantizada para CI |
| Audit backend | 0 vulnerabilidades | 🟢 Cadena backend saneada tras fix `qs` |
| Audit frontend | hallazgos vigentes | 🟡 Riesgo en seguimiento, sin ocultar resultados |
| A11y automática | configurada/ejecutable | 🟡 Debe complementarse con validación humana |

### Interpretación de negocio académico

- Una cobertura baja en matrícula/restricciones puede habilitar escenarios de inscripción inválida.
- Un control de lint estable reduce errores triviales en formularios y rutas.
- Un backend con dependencias saneadas disminuye exposición a fallos de disponibilidad.
- El Quality Gate en `OK` confirma capacidad de control, pero los 784 issues y ratings D/E muestran deuda de calidad y seguridad que debe priorizarse.

### Evidencias

Ver detalle completo en [`Capturas`](./COVERAGE_ANALYSIS.md).

### Conclusión

Se logró interpretar métricas con enfoque técnico y de negocio académico: ya no solo se reportan números, sino el impacto directo en matrícula, horarios, estabilidad y riesgo operacional del sistema.

---

## c. Análisis OWASP

### Concepto del apartado

Evaluar la postura de seguridad de SGOHA frente a riesgos OWASP (acceso, autenticación, misconfiguración, dependencias, excepciones y trazabilidad), combinando revisión de código y evidencia de ejecución.

### Subapartados (concepto + propósito + logro)

| Subapartado | Qué hace | Logro en SGOHA |
| ---------- | -------- | -------------- |
| Superficie de ataque | Identifica vectores por capa | ✅ Frontend, backend y CI cubiertos |
| Controles preventivos | Endurece app antes de explotar | ✅ Helmet/rate-limit/RBAC activos |
| Cadena de suministro | Reduce riesgo de dependencias | ✅ `qs` corregido; frontend en seguimiento |
| Seguridad automatizada | Escaneo estático/dinámico continuo | ⚙️ CodeQL y ZAP definidos |
| Riesgo residual | Define qué queda abierto | ✅ Matriz con estado técnico y acción |

### Qué se implementó y qué cubre

| Control | Qué hace en SGOHA | Estado |
| ------- | ----------------- | ------ |
| `helmet` | Endurece cabeceras HTTP | ✅ Corregido |
| `loginRateLimiter` / `apiRateLimiter` | Mitiga abuso y fuerza bruta | ✅ Corregido |
| JWT + RBAC | Restringe acceso por rol | 🟢 Conforme |
| bcrypt y exclusión de password | Protección de credenciales | 🟢 Conforme |
| npm audit + matriz OWASP | Control de cadena de suministro | 🟡 Seguimiento activo |
| CodeQL workflow | Análisis estático en GitHub | ⚙️ Automatizado |
| ZAP workflow | DAST baseline en CI | ⚙️ Automatizado |

### Aplicación real al sistema

- Se redujo exposición a ataques de autenticación y misconfiguración.
- Se resolvió el hallazgo moderado de `qs` en backend.
- Se documentaron riesgos residuales reales (sin ocultarlos ni falsearlos).

### Evidencias

Ver detalle completo en [`Capturas`](./COVERAGE_ANALYSIS.md).

### Conclusión

Se consolidó una base DevSecOps funcional: controles preventivos en backend, auditoría de dependencias, escaneo estático y flujo DAST reproducible. El logro principal es pasar de seguridad declarativa a seguridad operativa verificable.

---

## d. Validación WCAG

### Concepto del apartado

Verificar que la interfaz cumpla criterios de accesibilidad con objetivo AA, tanto en pruebas automatizadas (axe/Lighthouse) como en revisión humana guiada.

### Subapartados (concepto + propósito + logro)

| Subapartado | Qué hace | Logro en SGOHA |
| ---------- | -------- | -------------- |
| Perceptible | Comprensión visual y semántica | ✅ `lang`, estructura y reportes definidos |
| Operable | Uso por teclado/foco/controles | ⚙️ Cobertura automática + checklist manual |
| Comprensible | Mensajes y consistencia | ✅ Patrones de validación documentados |
| Robusto | Compatibilidad con asistencia | ⚙️ ARIA y pruebas axe configuradas |
| Evidencia | Prueba reproducible | ✅ Suite Cypress a11y por pantallas |

### Automatización implementada

| Elemento | Aporte técnico | Estado |
| -------- | -------------- | ------ |
| `lang="es"` | Mejora lectura de tecnologías asistivas | ✅ Corregido |
| Cypress + `cypress-axe` | Detección automática de violaciones | ⚙️ Configurado |
| Specs a11y por módulo | Cobertura de pantallas críticas por rol | ⚙️ Configurado |
| `lighthouserc.json` | Auditoría reproducible de accesibilidad | ⚙️ Configurado |
| Checklist manual | Validación de teclado/foco/lector/zoom | 🧑‍💻 Requiere validación humana |

### Resultado del apartado

- La automatización está implementada y lista.
- La conformidad completa depende de la validación manual y evidencia capturable.

### Evidencias

Ver detalle completo en [`Capturas`](./COVERAGE_ANALYSIS.md).

### Conclusión

Se alcanzó una validación híbrida madura (automatización + protocolo manual). El logro real es contar con pruebas accesibles repetibles por módulo y una ruta clara para cerrar cumplimiento AA con evidencia humana.

---

## e. Análisis SUS

### Concepto del apartado

SUS mide usabilidad percibida con un instrumento estandarizado (10 preguntas, escala 1–5). En este proyecto se implementó el sistema completo de captura/cálculo, evitando simular usuarios reales.

### Subapartados (concepto + propósito + logro)

| Subapartado | Qué hace | Logro en SGOHA |
| ---------- | -------- | -------------- |
| Instrumento | Define preguntas y escala | ✅ Cuestionario formal en español |
| Datos | Estandariza captura | ✅ CSV plantilla y anonimización |
| Cálculo | Automatiza fórmula SUS | ✅ Script genera JSON y Markdown |
| Piloto | Valida metodología sin falsear resultados | ✅ Ejemplo demostrativo explícito |
| Aplicación real | Ejecuta con participantes | 🧑‍💻 Protocolo listo para campo |

### Qué quedó completo

| Entregable | Descripción | Estado |
| ---------- | ----------- | ------ |
| Cuestionario SUS | Instrumento formal en español | ✅ |
| Plantilla CSV | Captura anonimizada por participante/rol | ✅ |
| Script de cálculo | Cálculo individual, promedio, mediana y salida MD/JSON | ✅ |
| Piloto metodológico | Ejemplo matemático declarado como demostrativo | ✅ |
| Protocolo real | Sesión con participantes y trazabilidad | ✅ |

### Qué requiere validación humana

- Aplicación con participantes reales (mínimo recomendado por protocolo).
- Carga de `sus-responses.csv` real y ejecución del cálculo final institucional.

### Evidencias

Ver detalle completo en [`Capturas`](./COVERAGE_ANALYSIS.md).

### Conclusión

Se completó el sistema de medición SUS de extremo a extremo (instrumento, datos, cálculo, protocolo e interpretación). Queda únicamente la aplicación con usuarios reales, que es una validación humana posterior y no una brecha técnica del expediente.

---

## Integración CI/CD

### Concepto del apartado

Conectar calidad, seguridad, accesibilidad y trazabilidad en pipelines automáticos para evitar validaciones manuales aisladas.


| Workflow | Función actual |
| -------- | -------------- |
| `ci.yml` | Lint, build, tests, cobertura, audit, a11y |
| `security.yml` | npm audit, secret scan, ZAP baseline |
| `codeql.yml` | Análisis estático de seguridad |
| `sonar.yml` | Sonar condicional por secretos |
| `cd-template.yml` | Plantilla de despliegue (manual) |

---

## Mejoras implementadas

- ✅ Seguridad backend reforzada (helmet, rate-limit, límites de payload).
- ✅ Frontend estabilizado para CI (lint sin errores bloqueantes).
- ✅ Suite de pruebas consolidada (208 tests) y cobertura documentada.
- ✅ Automatización de accesibilidad con Cypress+axe por pantallas clave.
- ✅ Instrumentación SUS completa (cuestionario + script + protocolo).
- ✅ Informe técnico 7.2 ampliado con explicación conceptual por apartado.
- ✅ Ejecución real de SonarQube local con métricas y Quality Gate.

---

## Comparación antes y después

| Indicador | Antes | Después |
| --------- | ----- | ------- |
| Pruebas automatizadas | Parcial | 208 tests ejecutables |
| Lint en CI | Informativo | Bloqueante en errores |
| Seguridad HTTP | Básica | Endurecida y documentada |
| Sonar | Solo intención | Configuración + ejecución real + métricas API |
| Accesibilidad | Casos aislados | Suite a11y estructurada + checklist |
| SUS | Sin instrumentación | Flujo completo implementado |
| Informe 7.2 | Estructura inicial | Expediente integral detallado |

---

## Conclusiones

El punto 7.2 queda técnicamente consolidado y explicativo: cada apartado (a–e) tiene concepto, aplicación al sistema, evidencias y estado.

SonarQube ya quedó ejecutado localmente con resultado real (`projectKey: sgoha`, Quality Gate `OK`), por lo que el expediente supera la fase de preparación y entra en fase de mejora continua basada en métricas.

El cierre técnico final se centra en tres frentes: elevar cobertura efectiva, reducir issues de seguridad/calidad reportados por Sonar y completar validaciones humanas (WCAG manual y SUS real).
