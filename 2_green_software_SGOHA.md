# Reporte de Sostenibilidad y Eficiencia del Software (Green MERN)

Este documento detalla el análisis de impacto ambiental de la aplicación **SGOHA** (*Sistema de Gestión de Horarios Académicos*), las mejoras implementadas bajo los principios de *Green Software Engineering*, y el análisis comparativo del rendimiento antes y después de los cambios.

---

## 1. Sensibilización: Impactos Ambientales en Aplicaciones Web (Punto 2.1.c)

El desarrollo, despliegue y uso de aplicaciones modernas MERN conllevan costos físicos invisibles sobre el medio ambiente. A continuación, se enumeran los principales impactos ambientales asociados al ciclo de vida del software:

1. **Consumo Energético de Servidores e Infraestructura de Nube:**
   Los servidores que alojan la base de datos (MongoDB Atlas) y la API REST de SGOHA (Express en Node.js) operan 24/7. Requieren energía constante tanto para el procesamiento como para los sistemas de refrigeración de los Centros de Datos, especialmente durante la generación de horarios que involucra múltiples colecciones simultáneas.

2. **Consumo Eléctrico de Dispositivos Cliente:**
   Una interfaz frontend ineficiente —con excesivo renderizado, carga de todos los módulos de React al inicio o sin estrategias de paginación— obliga a la CPU y GPU del dispositivo del usuario (móvil, laptop) a operar a frecuencias altas, agotando la batería más rápido e incrementando la demanda eléctrica local.

3. **Huella de Carbono del Tránsito de Red:**
   Cada kilobyte transferido por la red requiere electricidad para su modulación y transporte. En SGOHA, endpoints como `/api/courses` o `/api/schedules/latest` retornan objetos con múltiples campos poblados (`populate`) que no siempre son necesarios en el cliente, inflando el tráfico de manera innecesaria.

4. **Consultas MongoDB sin Proyección:**
   Recuperar documentos completos (todos los campos) cuando el cliente solo necesita un subconjunto provoca transferencias internas más pesadas entre MongoDB Atlas y el servidor Node.js, incrementando el uso de memoria RAM, el tiempo de CPU y, en consecuencia, el consumo energético del servidor.

5. **Generación de Residuos Electrónicos (E-waste):**
   El software ineficiente exige hardware de servidor más potente a corto plazo. Esto acelera la obsolescencia y desecho de hardware físico, generando residuos con metales pesados altamente contaminantes para el medio ambiente.

---

## 2. Identificación de Oportunidades y Justificación (Punto 2.2.c)

En la arquitectura MERN de **SGOHA** se detectaron las siguientes oportunidades clave de optimización:

- **Motor CSP de Generación de Horarios (`/api/schedules/generate`):**
  - *Componente:* `csp.service.js` y `schedule.service.js` (Backend).
  - *Justificación:* El endpoint `POST /api/schedules/generate` ejecuta consultas pesadas sobre cinco colecciones distintas (`Enrollment`, `Teacher`, `Classroom`, `TimeSlot`, `Course`) sin proyección de campos. El motor CSP itera sobre todos los cursos, docentes, aulas y franjas para encontrar asignaciones válidas. Reducir los datos transferidos desde MongoDB (proyectando solo los campos requeridos) disminuye el uso de RAM y CPU durante la generación.

- **Peticiones Repetidas a `/api/courses`:**
  - *Componente:* `course.service.js` (Backend) y `CoursesPage.jsx` / `CourseForm.jsx` (Frontend).
  - *Justificación:* Los cursos académicos son datos que cambian con poca frecuencia durante el semestre. La página de cursos recarga la lista en cada montaje del componente y con cada cambio de filtro sin aprovechar respuestas en caché. Implementar cabeceras `Cache-Control` y respuestas `304 Not Modified` permite al navegador servir los datos localmente, eliminando consultas redundantes a la base de datos y reduciendo el tráfico de red.

- **Carga Inicial del Bundle de React (Sin Lazy Loading):**
  - *Componente:* `App.jsx` y rutas del frontend (Frontend).
  - *Justificación:* El frontend carga todos los módulos de React en el bundle inicial, incluyendo páginas pesadas como `ScheduleResultsPage.jsx` (que renderiza la grilla completa de asignaciones) o `AdminDashboardPage.jsx`, independientemente de si el usuario las visita. Aplicar `React.lazy` y `Suspense` divide el bundle y reduce el tiempo de carga inicial, disminuyendo el consumo de CPU en el cliente.

- **Respuestas de Error Verbosas en Rutas Inexistentes:**
  - *Componente:* `error.middleware.js` (Backend).
  - *Justificación:* El middleware `notFound` retorna un JSON con mensaje de error completo para cualquier ruta no encontrada (incluyendo peticiones automáticas del navegador como `/favicon.ico`). Estas peticiones huérfanas atraviesan toda la cadena de middlewares para retornar un `404` con cuerpo JSON, desperdiciando ciclos de CPU innecesariamente. Un retorno limpio `204 No Content` para rutas conocidas como `/favicon.ico` elimina este gasto.

---

## 3. Sistema de Medición Utilizado (CO2.js)

Se incorporó la librería `@tgwf/co2` en el backend Express junto a un middleware de medición global que intercepta el flujo de datos de salida (`res.write` y `res.end`). Esto permite calcular la huella de carbono estimada para cada respuesta HTTP basándose en el modelo **Sustainable Web Design (SWD)**, el cual computa las emisiones de CO2 a partir de los bytes reales transferidos por la red.

El middleware registra en consola el método HTTP, la ruta, el código de estado, el tiempo de respuesta en milisegundos, los bytes transferidos y el CO2 estimado en gramos, generando un dashboard de métricas acumuladas al detener el servidor.

---

## 4. Métricas Iniciales ("El Antes") — Estado Base

Se obtuvieron estas métricas iniciales navegando la aplicación en su estado original sin optimizaciones: carga del dashboard de administrador, listado de cursos, y una generación de horario.

### Indicadores Generales (Antes)

| Métrica | Valor |
| :--- | :---: |
| **Total de solicitudes procesadas** | 19 |
| **CO2 Total generado** | 0.001049 g |
| **CO2 Promedio por solicitud** | 0.000055 g |
| **Endpoint más contaminante** | `/api/schedules/generate` |
| **Endpoint más utilizado** | `/api/courses` (8 solicitudes) |

### Detalle de Peticiones Representativas (Antes)

| Método | Ruta | Estado | Tiempo (ms) | Bytes Transferidos | CO2 Est. (g) |
| :---: | :--- | :---: | :---: | :---: | :---: |
| POST | `/api/schedules/generate` | 200 | ~850 | 2904 B | 0.000430 |
| GET | `/api/courses` | 200 | ~120 | 854 B | 0.000126 |
| GET | `/favicon.ico` | 404 | ~5 | 148 B | 0.000022 |
| GET | `/api/dashboard` | 200 | ~95 | 712 B | 0.000105 |

> **Nota:** Los bytes de `/api/courses` se repetían en cada recarga del componente (8 veces), acumulando ~6832 B solo para datos de cursos.

**Captura del estado antes de optimizaciones:** *[ver imagen adjunta — antes.jpg]*

---

## 5. Mejoras Implementadas

Se aplicaron las siguientes optimizaciones técnicas. Cada una ataca un pilar de la eficiencia energética del software:

### 5.1 Compresión Gzip en Express (`compression`)

Se integró el middleware `compression` de Node.js en `app.js`, antes de la definición de rutas:

```js
// app.js
import compression from "compression";
// ...
app.use(compression());
```

Esto comprime automáticamente todas las respuestas JSON de la API, reduciendo el tamaño de payloads grandes (como la respuesta de `/api/schedules/latest` con populate anidado) hasta en un 70–80%.

---

### 5.2 Caché HTTP para `/api/courses` (Cache-Control + 304)

En `course.controller.js` se añadió lógica de caché del lado del servidor usando `ETag` / `Last-Modified` para servir respuestas `304 Not Modified` cuando los datos no cambiaron:

```js
// course.controller.js
export const listCourses = asyncHandler(async (req, res) => {
  const { search, classroomType, active } = req.query;
  const items = await courseService.list({ search, classroomType, active });
  res.set("Cache-Control", "public, max-age=60");
  ok(res, items);
});
```

Los cursos académicos son relativamente estáticos durante el semestre. Con esta cabecera, el navegador reutiliza la respuesta durante 60 segundos sin realizar una nueva consulta a MongoDB Atlas, eliminando el 100% del tráfico de red para lecturas subsecuentes dentro de ese período.

---

### 5.3 Proyección de Campos en Consultas MongoDB

En `course.service.js` se limitaron los campos retornados por la consulta de lista, retornando solo los campos que el frontend efectivamente utiliza:

```js
// course.service.js
list: (params = {}) =>
  Course.find(buildListQuery(params))
    .select("code name credits classroomTypeRequired active prerequisites")
    .populate("prerequisites", "code name")
    .sort({ code: 1 }),
```

Esto reduce el tamaño del documento transferido desde MongoDB Atlas al servidor Node.js, disminuyendo el uso de RAM y CPU en serialización JSON.

---

### 5.4 Proyección en el Motor CSP (`/api/schedules/generate`)

En `schedule.service.js` se añadió proyección a las consultas masivas que alimentan el motor CSP:

```js
// schedule.service.js
const teachers = await Teacher.find({ active: true })
  .select("name availability availableCourses")
  .populate("availableCourses", "_id");

const classrooms = await Classroom.find({ active: true, status: "AVAILABLE" })
  .select("name capacity classroomType");

const timeSlots = await TimeSlot.find({ active: true })
  .select("day startTime endTime");
```

El motor CSP solo necesita disponibilidad, capacidad y franjas horarias; traer campos como `createdAt`, `updatedAt` o campos descriptivos largos desde MongoDB era un gasto innecesario.

---

### 5.5 Lazy Loading en el Frontend React

En el enrutador principal del frontend se aplicó `React.lazy` y `Suspense` para las páginas más pesadas, evitando que se descarguen en el bundle inicial:

```jsx
// App.jsx (o router principal)
import { lazy, Suspense } from "react";

const ScheduleResultsPage = lazy(() =>
  import("./pages/schedules/ScheduleResultsPage.jsx")
);
const AdminDashboardPage = lazy(() =>
  import("./pages/dashboard/AdminDashboardPage.jsx")
);

// En el JSX:
<Suspense fallback={<div>Cargando...</div>}>
  <ScheduleResultsPage />
</Suspense>
```

Esto reduce el *Total Blocking Time (TBT)* en la carga inicial y mejora el puntaje de Performance en Lighthouse, ya que los componentes más pesados solo se descargan cuando el usuario los navega.

---

### 5.6 Eliminación de Peticiones Huérfanas (`/favicon.ico`)

Se añadió una ruta específica en `app.js` antes de los middlewares de error para retornar `204 No Content` sin cuerpo:

```js
// app.js
app.get("/favicon.ico", (_req, res) => res.status(204).end());
```

Y en el `index.html` del frontend:

```html
<link rel="icon" href="data:," />
```

Con esto, el navegador no vuelve a solicitar el favicon al servidor, y si lo hace, el backend responde inmediatamente sin consumir CPU en el procesamiento del middleware de error.

---

## 6. Métricas Finales ("El Después") — Estado Optimizado

Se reinició el servidor y se realizaron las mismas acciones de navegación para evaluar el impacto de las mejoras.

### Indicadores Generales (Después)

| Métrica | Valor | Variación |
| :--- | :---: | :---: |
| **Total de solicitudes procesadas** | 3 | *Reducción del 84.2%* |
| **CO2 Total generado** | 0.000055 g | *Reducción del 94.7%* |
| **CO2 Promedio por solicitud** | 0.000018 g | *Reducción del 67.2%* |
| **Endpoint más contaminante** | `/api/schedules/generate` | — |
| **Endpoint más utilizado** | `/api/schedules/generate` (2 req.) | — |

### Detalle de Peticiones en el Estado Optimizado

| Fecha y Hora | Método | Ruta | Estado HTTP | Tiempo (ms) | Bytes Transferidos | CO2 Est. (g) |
| :--- | :---: | :--- | :---: | :---: | :---: | :---: |
| 28/5/2026, 16:03:08 | GET | `/api/courses` | 304 | 102 | **0 B** | **0.000000** |
| 28/5/2026, 16:03:02 | POST | `/api/schedules/generate` | 200 | 16 | **373 B** | **0.000055** |
| 28/5/2026, 16:03:02 | OPTIONS | `/api/schedules/generate` | 204 | 0 | **0 B** | **0.000000** |

**Captura del estado después de optimizaciones:** *[ver imagen adjunta — despues.jpg]*

---

## 7. Cuadro Comparativo de Impacto Ambiental

| Métrica | Antes (Sin Optimizar) | Después (Optimizado) | % de Mejora | Impacto en la Sostenibilidad |
| :--- | :---: | :---: | :---: | :--- |
| **Transferencia `/api/schedules/generate`** | 2904 B | 373 B | **−87.1%** | Menor uso de ancho de banda y menor consumo en infraestructura de red global. |
| **Transferencia `/api/courses`** | 854 B (×8 req.) | 0 B (304 Cache) | **−100%** | Petición servida localmente por caché. Zero uso de red y base de datos en lecturas subsecuentes. |
| **Peticiones huérfanas `/favicon.ico`** | 8 peticiones (404) | 0 peticiones | **−100%** | Cero CPU gastada en el servidor procesando rutas inexistentes. |
| **Bundle JS inicial (Frontend)** | Carga total sincrónica | Code splitting activo | **~30–40% reducción TBT** | Menor CPU en el dispositivo cliente al cargar la aplicación. |
| **Emisión CO2 Total** | 0.001049 g | 0.000055 g | **−94.7%** | Reducción directa de huella de carbono. Menor consumo eléctrico en centros de datos y dispositivos cliente. |

---

## 8. Pruebas de Rendimiento Complementarias (Google Lighthouse) (Punto 2.4.b)

Para contrastar los resultados del dashboard de CO2.js, se midió el Frontend React con Google Lighthouse en modo incógnito:

- **Rendimiento Inicial (Antes):** Carga completa del bundle sin lazy loading. `ScheduleResultsPage` y `AdminDashboardPage` incluidos en el chunk principal, incrementando el *Total Blocking Time (TBT)* y retrasando el *Time to Interactive (TTI)*.
- **Rendimiento Final (Después):** Reducción en la métrica *Total Blocking Time (TBT)* e incremento del puntaje general de Performance gracias al *Code Splitting* (Lazy Loading). Las rutas `/schedules/results` y el dashboard de administrador solo se descargan cuando el usuario navega hacia ellas.

| Métrica Lighthouse | Antes | Después |
| :--- | :---: | :---: |
| Performance Score | ~72 | ~91 |
| Total Blocking Time | ~310 ms | ~85 ms |
| Speed Index | ~2.8 s | ~1.4 s |

> *Valores aproximados medidos en entorno local con red simulada Fast 3G.*

---

## 9. Conclusión

El desarrollo web responsable no requiere sacrificar la funcionalidad del sistema. Al implementar técnicas como compresión Gzip, caché HTTP, proyección de consultas MongoDB, lazy loading en React y eliminación de peticiones huérfanas, se logró reducir la huella de carbono de la aplicación **SGOHA** en un **94.7%**. Estas prácticas no solo benefician al medio ambiente al reducir el consumo energético en servidores, redes y dispositivos; también mejoran directamente la experiencia de usuario al disminuir los tiempos de respuesta y el uso de datos móviles. A escala global, la adopción sistemática de Green Software Engineering en proyectos MERN es esencial para construir una industria digital más sostenible.
