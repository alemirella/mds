# Informe Técnico Integral — SGOHA
## Revisión de calidad mediante SonarQube, OWASP Top 10 2025, WCAG y SUS

---

## 1. Datos de identificación

| Campo | Valor |
| --- | --- |
| Proyecto | SGOHA — Sistema de Generación Óptima de Horarios Académicos |
| Versión analizada | 2.0.0 |
| Repositorio | `talitakeren/Repositorio_proyecto` |
| Rama | `qa-calidad` |
| Commit de referencia | `fb539a2` |
| Fecha de corte | `[COMPLETAR: fecha de entrega final]` |
| Stack | React 19 + Vite · Node/Express · MongoDB/Mongoose · JWT · Jest · Cypress |
| Entorno de evaluación | Local (Docker) + GitHub Actions |
| Integrantes | Contreras Infanzón Alexandra Mirella, Espinoza Zarate Juan Carlos, Huaman Raymundo Yenifer Nicole, Olivera Paredes Talita Keren, Vega Carhuallanqui Tatiana |

---

## 2. Resumen ejecutivo

- Cobertura global de pruebas: **30,3%** de líneas (208 tests Jest, todos en verde).
- Backend sin vulnerabilidades auditables tras actualización de `qs` a `6.15.2`.
- Frontend con ESLint sin errores bloqueantes (3 warnings documentados).
- SonarQube ejecutado localmente con análisis real (`projectKey: sgoha`), Quality Gate `OK`.
- `[COMPLETAR: resumen de hallazgos WCAG manual una vez finalizado]`
- `[COMPLETAR: resultado SUS con muestra real una vez aplicado el cuestionario]`

---

## 3. SonarQube

### 3.1 Configuración

| Parámetro | Valor |
| --- | --- |
| Herramienta | SonarQube Community (Docker) + sonar-scanner-cli |
| Archivo de configuración | `sonar-project.properties` |
| Fuentes analizadas | `frontend/src`, `backend/src` |
| Tests incluidos | `tests/`, `cypress/` |
| Cobertura | LCOV (`tests/reports/coverage/*/lcov.info`) |
| Exclusiones | `node_modules`, `dist`, `build`, `coverage` |

### 3.2 Resultados

| Métrica | Valor | Rating |
| --- | ---: | :--: |
| Quality Gate | `OK` | 🟢 |
| Bugs | 4 | — |
| Vulnerabilities | 3 | — |
| Security Hotspots | 7 | — |
| Code Smells | 777 | — |
| Duplicated Lines Density | 1,4% | — |
| Coverage (Sonar) | 15,5% | — |
| Reliability Rating | 4.0 | D |
| Security Rating | 5.0 | E |
| Maintainability Rating | 1.0 | A |
| Technical Debt (`sqale_index`) | 3794 min | — |
| Cognitive Complexity | 1555 | — |
| Issues totales | 784 | — |
| LOC (`ncloc`) | 19750 | — |

> Fuente: `docs/reportes/sonar/SONARQUBE_LOCAL_EXECUTION.md` — ejecución real vía Docker, métricas consultadas por API de SonarQube (`/api/measures`, `/api/qualitygates/project_status`).

**Capturas de evidencia:** `[PENDIENTE: capturas de dashboard]`
- `[ ]` Captura del Quality Gate en el panel principal → guardar en `docs/evidencias/sonarqube/01-quality-gate.png`
- `[ ]` Captura de la pestaña Issues (bugs/smells) → `docs/evidencias/sonarqube/02-issues.png`
- `[ ]` Captura de la cobertura importada (LCOV) → `docs/evidencias/sonarqube/03-coverage.png`

### 3.3 Interpretación

- El Quality Gate en `OK` indica que el proyecto cumple el criterio activo configurado en la instancia local.
- La diferencia entre cobertura Sonar (15,5%) y cobertura Jest consolidada (30,3%) se debe a diferencias en el alcance de fuentes y el método de cómputo entre ambas herramientas.
- Maintainability Rating en A es positivo pese al alto número de Code Smells, porque el ratio deuda/código se mantiene bajo.
- Security Rating en E (5.0) es el punto más crítico a remediar: requiere revisar las 3 vulnerabilities y 7 security hotspots reportados.
- `csp.service.js` y los módulos de matrícula concentran la mayor complejidad cognitiva (1555 total).

### 3.4 Mejoras aplicadas / pendientes

| Mejora | Estado |
| --- | :--: |
| Ejecución real de SonarQube local con métricas verificables | ✅ |
| Integración condicional en CI (`sonar.yml`) | ⚙️ Requiere `SONAR_TOKEN` |
| Remediar Security Rating (E → objetivo C o mejor) | `[PENDIENTE]` |
| Reducir Code Smells en módulos críticos | `[PENDIENTE]` |

---

## 4. OWASP Top 10 2025

### 4.1 Alcance de la auditoría

Revisión de código + `npm audit` + verificación de controles implementados sobre `backend/src`, `frontend/src` y pipelines CI/CD.

### 4.2 Matriz de hallazgos por categoría

| Categoría | Componente | Escenario | Riesgo | Control actual | Estado |
| --- | --- | --- | :--: | --- | :--: |
| Broken Access Control | API REST | Alumno accede a `PUT /api/users` | 🟠 | `protect` + `authorizeRoles` por ruta | 🟢 |
| Security Misconfiguration | Express | CORS abierto en desarrollo | 🟡 | CORS restrictivo en producción (`CLIENT_URL`) | 🟢 |
| Supply Chain | npm | Dependencia vulnerable | 🟡 | `npm audit`, Dependabot, CI | 🟡 |
| Cryptographic Failures | Auth | JWT débil o sin expiración | 🟡 | `jsonwebtoken` + `JWT_SECRET` en env | 🟢 |
| Injection | Mongoose | NoSQL injection en filtros | 🟢 | Esquemas tipados, sin `$where` dinámico | 🟢 |
| Insecure Design | Matrícula | Bypass de prerrequisitos | 🟠 | Validación servidor en `enrollment.service` | 🟡 |
| Authentication Failures | Login | Fuerza bruta | 🟠 | `loginRateLimiter` (20/15 min) | 🟢 |
| Integrity Failures | CI | Workflow malicioso | 🟡 | Permisos mínimos en Actions | 🟡 |
| Logging Failures | Producción | Sin alertas de abuso | 🟡 | `morgan` combined en prod | 🟡 |
| Exception Handling | API | Stack trace al cliente | 🟢 | `errorHandler` oculta stack en prod | 🟢 |

### 4.3 Mitigaciones implementadas

| ID | Hallazgo | Corrección | Evidencia en código |
| --- | --- | --- | --- |
| OW-001 | Sin helmet (histórico) | `securityHeaders` activado | `backend/.../security.middleware.js` |
| OW-002 | Sin rate limit en login | `loginRateLimiter` implementado | `backend/.../security.middleware.js` |
| OW-003 | `qs` con CVE (GHSA-q8mj-m7cp-5q26) | Actualizado a `6.15.2` | `backend/package.json` |
| OW-004 | JWT en `localStorage` (riesgo XSS) | Documentado, CSP como mitigación | `OWASP_ANALYSIS.md` |

### 4.4 Auditoría de dependencias (npm audit)

| Resultado | Backend | Frontend |
| --- | :--: | :--: |
| Vulnerabilidades | 0 (tras fix `qs`) | 5 (2 alta, en dependencias dev) |

**Evidencia:** `[PENDIENTE: generar y adjuntar]`
- `[ ]` Ejecutar `npm run audit:security`
- `[ ]` Verificar que genere `docs/reportes/security/backend-npm-audit.json`
- `[ ]` Verificar que genere `docs/reportes/security/frontend-npm-audit.json`

### 4.5 CodeQL (análisis estático complementario)

| Parámetro | Valor |
| --- | --- |
| Lenguaje | `javascript-typescript` |
| Consultas | `security-extended` |
| Disparadores | push/PR en `main`, cron semanal, manual |

**Evidencia:** `[PENDIENTE]`
- `[ ]` Captura de alertas en GitHub → Security → Code scanning → `docs/evidencias/owasp/03-codeql.png`

### 4.6 Riesgo residual

- Security Hotspots de SonarQube (7) requieren revisión manual antes de cerrarse como falsos positivos o corregirse.
- Dependencias frontend con hallazgos de severidad alta (dev) en seguimiento, sin impacto directo en producción.

---

## 5. WCAG (Accesibilidad)

### 5.1 Alcance

Pantallas evaluadas: login, dashboard ADMIN, usuarios, cursos, docentes, disponibilidad, aulas, estudiantes, franjas, matrícula (admin y alumno), horarios, restricciones, configuración, portales docente/alumno.

Nivel objetivo: **WCAG 2.2 AA**

### 5.2 Resultados automáticos (axe-core + Cypress)

| Pantalla | Spec Cypress | Violaciones críticas (umbral) |
| --- | --- | :--: |
| Login | `login.cy.js`, `a11y-login.cy.js` | 0 |
| Dashboard ADMIN | `admin-dashboard.cy.js` | 0 |
| Cursos | `courses.cy.js` | 0 |
| Docentes | `teachers.cy.js` | 0 |
| Disponibilidad docente | `teacher-availability.cy.js` | 0 |
| Matrícula alumno | `student-enrollment.cy.js` | 0 |
| Horarios | `schedules.cy.js` | 0 |

> Comando de ejecución: `npm run test:a11y`

**Resultado Lighthouse:** `[PENDIENTE: ejecutar y completar]`
- `[ ]` Ejecutar `npx @lhci/cli autorun`
- `[ ]` Registrar puntaje de accesibilidad por pantalla
- `[ ]` Captura → `docs/evidencias/wcag/01-lighthouse-login.png`

### 5.3 Resultados manuales (checklist)

| Criterio | Nivel | Pantalla | Resultado | Hallazgo |
| --- | :--: | --- | :--: | --- |
| 1.1.1 Contenido no textual | A | Login | `[PENDIENTE]` | — |
| 1.3.1 Info y relaciones | A | Dashboard | `[PENDIENTE]` | — |
| 1.4.3 Contraste mínimo | AA | Formularios | 🟡 | Algunos textos `slate-400`, revisar ratio 4.5:1 |
| 2.1.1 Teclado | A | Matrícula alumno | `[PENDIENTE]` | — |
| 2.4.3 Orden de foco | A | Modales | 🟢 | Modal con trap básico |
| 2.4.7 Foco visible | AA | Global | 🟢 | Tailwind `focus-visible` en botones |
| 2.5.3 Etiqueta en nombre | A | Botones icono | 🟡 | Algunos `title` sin `aria-label` |
| 3.3.1 Identificación de errores | A | Login | 🟢 | Mensajes bajo campos |
| 3.3.2 Etiquetas | A | Usuarios | 🟢 | `Input` con label |
| 4.1.2 Nombre, rol, valor | A | Disponibilidad | `[PENDIENTE]` | — |
| Zoom 200% | AA | Horarios | `[PENDIENTE]` | — |
| Escape cierra modal | A | Cursos | 🟢 | `Modal.jsx` |

**Para completar los `[PENDIENTE]`:**
- `[ ]` Navegar la app solo con teclado (Tab, Shift+Tab, Enter, Escape) en login → dashboard → matrícula
- `[ ]` Probar con lector de pantalla (NVDA o VoiceOver) en login y formulario de curso
- `[ ]` Probar zoom 200% en Chrome sobre la tabla de horarios
- `[ ]` Capturas → `docs/evidencias/wcag/03-keyboard.png`

### 5.4 Correcciones implementadas

| ID | Corrección | Archivo |
| --- | --- | --- |
| A11Y-01 | `lang="es"` en documento | `frontend/index.html` |
| A11Y-02 | `aria-invalid`, `aria-describedby` en inputs | `components/ui/Input.jsx` |
| A11Y-03 | Foco y `role="dialog"` en modales | `components/ui/Modal.jsx` |
| A11Y-04 | Alertas de formulario | `pages/ClassroomList.jsx` |

---

## 6. SUS (Usabilidad)

### 6.1 Metodología

| Parámetro | Valor |
| --- | --- |
| Instrumento | System Usability Scale (10 preguntas, escala 1–5) |
| Tamaño de muestra recomendado | Mínimo 5 participantes |
| Perfiles requeridos | ≥ 1 ADMIN, ≥ 2 TEACHER, ≥ 2 STUDENT |
| Reclutamiento | Usuarios externos al equipo de desarrollo |
| Anonimización | Código `P001`...`P00N`, sin nombres |

**Tareas previas al cuestionario (por rol):**
- **ADMIN:** iniciar sesión → registrar curso → registrar docente/aula → revisar matrícula → generar horario
- **TEACHER:** iniciar sesión → registrar disponibilidad → consultar cursos/horario
- **STUDENT:** iniciar sesión → seleccionar cursos → validar matrícula → consultar horario

### 6.2 Resultados

`[PENDIENTE: completar con datos reales]`

| Indicador | Valor |
| --- | ---: |
| Participantes | `[PENDIENTE]` |
| Promedio SUS | `[PENDIENTE]` |
| Mediana | `[PENDIENTE]` |
| Interpretación | `[PENDIENTE]` |

**Pasos para completar:**
- `[ ]` Aplicar cuestionario (`docs/plantillas/CUESTIONARIO_SUS.md`) a mínimo 5 personas reales
- `[ ]` Cargar respuestas en `docs/reportes/usability/sus-responses.csv` (formato igual a `sus-responses-template.csv`)
- `[ ]` Ejecutar: `npm run sus:calculate` (genera `sus-results.json` y `.md` reales)
- `[ ]` Reemplazar el resultado demo (`sus-demo.csv`, 1 participante) por el resultado real

### 6.3 Oportunidades de mejora

`[PENDIENTE: completar tras analizar comentarios abiertos del cuestionario]`

---

## 7. Testing automatizado

### 7.1 Resumen

| Métrica | Valor |
| --- | ---: |
| Pruebas totales (Jest) | 208 |
| Pruebas aprobadas | 208 |
| Pruebas fallidas | 0 |
| Specs E2E (Cypress) | 18 |
| Cobertura de líneas | 30,3% |
| Cobertura de sentencias | 29,0% |
| Cobertura de funciones | 22,3% |
| Cobertura de ramas | 14,1% |

### 7.2 Resultados por suite

| Suite | Pruebas | Estado |
| --- | ---: | :--: |
| Backend unitarias | 61 | ✅ |
| Backend integración API | 38 | ✅ |
| Frontend unitarias | 82 | ✅ (verificado en ejecución directa) |
| Frontend integración MSW | 27 | ✅ |
| E2E Cypress | 18 specs | `[PENDIENTE: ejecutar y registrar resultado]` |

**Comando de reproducción:** `npm run test:coverage`

**Áreas de riesgo (baja cobertura):**
- `csp.service.js` (motor de generación de horarios)
- `scheduleService`, `restrictionService`

---

## 8. Conclusiones

- El sistema cuenta con una base técnica sólida: testing real y reproducible, controles de seguridad OWASP implementados en código, y un análisis SonarQube ejecutado con métricas verificables.
- Los puntos de cierre pendientes son de **evidencia y aplicación real**, no de diseño: capturas visuales, checklist WCAG manual completo, y aplicación de SUS con participantes reales.
- El Security Rating (E) y la baja cobertura en el motor CSP son los riesgos técnicos más relevantes a comunicar en la sustentación.

---

## 9. Checklist final de pendientes (antes de entregar)

| # | Pendiente | Sección |
| :--: | --- | --- |
| 1 | Capturas de dashboard SonarQube (Quality Gate, Issues, Coverage) | 3.2 |
| 2 | Generar `backend-npm-audit.json` y `frontend-npm-audit.json` | 4.4 |
| 3 | Captura de alertas CodeQL en GitHub Security | 4.5 |
| 4 | Ejecutar Lighthouse y registrar puntajes | 5.2 |
| 5 | Completar 5 criterios pendientes del checklist WCAG manual | 5.3 |
| 6 | Capturas de navegación por teclado / lector de pantalla | 5.3 |
| 7 | Aplicar SUS con mínimo 5 participantes reales | 6.2 |
| 8 | Ejecutar specs E2E de Cypress y registrar resultado | 7.2 |
| 9 | Captura de workflows de CI/CD en verde (GitHub Actions) | — |

---

## Anexos — referencias en el repositorio

| Documento | Ruta |
| --- | --- |
| Informe técnico original | `docs/INFORME_TECNICO_INTEGRAL_7_2.md` |
| Análisis de cobertura | `docs/COVERAGE_ANALYSIS.md` |
| Ejecución SonarQube | `docs/reportes/sonar/SONARQUBE_LOCAL_EXECUTION.md` |
| Análisis OWASP | `docs/reportes/security/OWASP_ANALYSIS.md` |
| Checklist WCAG manual | `docs/reportes/accessibility/WCAG_MANUAL_CHECKLIST.md` |
| Protocolo SUS | `docs/reportes/usability/SUS_EVALUATION_PROTOCOL.md` |
| Matriz de hallazgos | `docs/plantillas/MATRIZ_HALLAZGOS.md` |
| Cuestionario SUS | `docs/plantillas/CUESTIONARIO_SUS.md` |
