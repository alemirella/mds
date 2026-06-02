# Guía de Implementación — Green Software SGOHA
## Código exacto para copiar y pegar, archivo por archivo

---

## PASO 0 — Instalar las dependencias necesarias

Abrí una terminal, entrá a la carpeta `backend` y ejecutá:

```bash
cd backend
npm install compression @tgwf/co2
```

Eso instala dos cosas:
- `compression` → middleware que activa Gzip en todas las respuestas de Express
- `@tgwf/co2` → librería que calcula el CO2 estimado por bytes transferidos

---

## PASO 1 — Crear el middleware de medición CO2

**Archivo nuevo a crear:**
`backend/src/middlewares/co2.middleware.js`

Creá ese archivo (no existe todavía) y pegá esto adentro:

```js
import { co2 } from "@tgwf/co2";

const swd = new co2({ model: "swd" });

// Acumuladores globales para el dashboard final
let totalRequests = 0;
let totalBytes = 0;
let totalCO2 = 0;
const endpointStats = new Map();

export function co2Middleware(req, res, next) {
  let bytesSent = 0;

  // Interceptamos res.write para contar bytes en streaming
  const originalWrite = res.write.bind(res);
  res.write = function (chunk, ...args) {
    if (chunk) {
      bytesSent += Buffer.byteLength(
        typeof chunk === "string" ? chunk : chunk
      );
    }
    return originalWrite(chunk, ...args);
  };

  // Interceptamos res.end para contar el último chunk y registrar métricas
  const originalEnd = res.end.bind(res);
  res.end = function (chunk, ...args) {
    if (chunk) {
      bytesSent += Buffer.byteLength(
        typeof chunk === "string" ? chunk : chunk
      );
    }

    const co2Grams = swd.perByte(bytesSent);
    const route = `${req.method} ${req.path}`;

    totalRequests++;
    totalBytes += bytesSent;
    totalCO2 += co2Grams;

    // Acumular por endpoint
    if (!endpointStats.has(route)) {
      endpointStats.set(route, { count: 0, bytes: 0, co2: 0 });
    }
    const ep = endpointStats.get(route);
    ep.count++;
    ep.bytes += bytesSent;
    ep.co2 += co2Grams;

    const fecha = new Date().toLocaleString("es-PE");
    console.log(
      `[CO2] ${fecha} | ${req.method.padEnd(7)} ${req.path.padEnd(40)} | ` +
        `HTTP ${res.statusCode} | ${bytesSent} B | ${co2Grams.toFixed(9)} g CO2`
    );

    return originalEnd(chunk, ...args);
  };

  next();
}

// Imprime el dashboard de métricas al detener el servidor (Ctrl+C)
export function printCO2Dashboard() {
  console.log("\n╔══════════════════════════════════════════════════════╗");
  console.log("║         DASHBOARD GREEN SOFTWARE — SGOHA             ║");
  console.log("╠══════════════════════════════════════════════════════╣");
  console.log(`║  Total solicitudes  : ${String(totalRequests).padEnd(30)}║`);
  console.log(`║  Total bytes        : ${String(totalBytes + " B").padEnd(30)}║`);
  console.log(
    `║  CO2 Total          : ${String(totalCO2.toFixed(9) + " g").padEnd(30)}║`
  );
  console.log(
    `║  CO2 Promedio/req   : ${String(
      totalRequests > 0 ? (totalCO2 / totalRequests).toFixed(9) + " g" : "0"
    ).padEnd(30)}║`
  );
  console.log("╠══════════════════════════════════════════════════════╣");
  console.log("║  DETALLE POR ENDPOINT                                ║");
  console.log("╠══════════════════════════════════════════════════════╣");

  const sorted = [...endpointStats.entries()].sort(
    (a, b) => b[1].co2 - a[1].co2
  );
  for (const [route, stats] of sorted) {
    console.log(`║  ${route.substring(0, 35).padEnd(35)}          ║`);
    console.log(
      `║    Reqs: ${String(stats.count).padEnd(5)} Bytes: ${String(
        stats.bytes + " B"
      ).padEnd(10)} CO2: ${stats.co2.toFixed(9)} g  ║`
    );
  }
  console.log("╚══════════════════════════════════════════════════════╝\n");
}
```

---

## PASO 2 — Modificar `app.js` (compresión + favicon + CO2 middleware)

**Archivo a editar:** `backend/src/app.js`

Reemplazá **todo** el contenido del archivo con esto:

```js
import express from "express";
import cors from "cors";
import morgan from "morgan";
import compression from "compression";  // ← NUEVO

import { co2Middleware } from "./middlewares/co2.middleware.js";  // ← NUEVO

import authRoutes from "./routes/auth.routes.js";
import userRoutes from "./routes/user.routes.js";
import courseRoutes from "./routes/course.routes.js";
import teacherRoutes from "./routes/teacher.routes.js";
import classroomRoutes from "./routes/classroom.routes.js";
import studentRoutes from "./routes/student.routes.js";
import timeslotRoutes from "./routes/timeslot.routes.js";
import enrollmentRoutes from "./routes/enrollment.routes.js";
import scheduleRoutes from "./routes/schedule.routes.js";
import dashboardRoutes from "./routes/dashboard.routes.js";
import restrictionRoutes from "./routes/restriction.routes.js";
import settingsRoutes from "./routes/settings.routes.js";

import { notFound, errorHandler } from "./middlewares/error.middleware.js";

const app = express();
const isProduction = process.env.NODE_ENV === "production";
const clientUrl = process.env.CLIENT_URL || "http://localhost:5173";

app.use(
  cors(
    isProduction
      ? { origin: clientUrl.split(",").map((o) => o.trim()) }
      : { origin: true }
  )
);
app.use(compression());          // ← NUEVO: Gzip en todas las respuestas
app.use(express.json());
app.use(morgan("dev"));
app.use(co2Middleware);           // ← NUEVO: medición CO2 global

// NUEVO: Silenciar favicon — evita que el navegador mande peticiones huérfanas
app.get("/favicon.ico", (_req, res) => res.status(204).end());

app.get("/api/health", (_req, res) => {
  res.json({ success: true, message: "SGOHA API operativa" });
});

app.use("/api/auth", authRoutes);
app.use("/api/users", userRoutes);
app.use("/api/courses", courseRoutes);
app.use("/api/teachers", teacherRoutes);
app.use("/api/classrooms", classroomRoutes);
app.use("/api/students", studentRoutes);
app.use("/api/timeslots", timeslotRoutes);
app.use("/api/enrollments", enrollmentRoutes);
app.use("/api/schedules", scheduleRoutes);
app.use("/api/dashboard", dashboardRoutes);
app.use("/api/restrictions", restrictionRoutes);
app.use("/api/settings", settingsRoutes);

app.use(notFound);
app.use(errorHandler);

export default app;
```

---

## PASO 3 — Modificar `server.js` para imprimir el dashboard al cerrar

**Archivo a editar:** `backend/src/server.js`

Reemplazá **todo** el contenido con esto:

```js
import "./config/env.js";
import connectDB from "./config/db.js";
import app from "./app.js";
import { timeslotService } from "./services/timeslot.service.js";
import { printCO2Dashboard } from "./middlewares/co2.middleware.js";  // ← NUEVO

const PORT = process.env.PORT || 5001;

await connectDB();

try {
  const sync = await timeslotService.syncOfficial();
  console.log(
    `Franjas HORALV sincronizadas: ${sync.total} total (${sync.created} nuevas, ${sync.removed} obsoletas eliminadas)`
  );
} catch (err) {
  console.warn(
    "Advertencia: no se pudo sincronizar franjas HORALV:",
    err.message
  );
}

app.listen(PORT, () => {
  console.log(`Servidor corriendo en puerto ${PORT}`);
});

// NUEVO: Imprimir dashboard de CO2 al detener el servidor con Ctrl+C
process.on("SIGINT", () => {
  printCO2Dashboard();
  process.exit(0);
});
```

---

## PASO 4 — Modificar `course.controller.js` (caché HTTP)

**Archivo a editar:** `backend/src/controllers/course.controller.js`

Solo cambiá la función `listCourses` (las demás quedan igual):

```js
// Reemplazá solo esta función, dejá el resto del archivo igual
export const listCourses = asyncHandler(async (req, res) => {
  const { search, classroomType, active } = req.query;
  const items = await courseService.list({ search, classroomType, active });
  // NUEVO: caché de 60 segundos — el navegador no vuelve a pedir si no cambiaron
  res.set("Cache-Control", "public, max-age=60");
  ok(res, items);
});
```

---

## PASO 5 — Modificar `course.service.js` (proyección MongoDB)

**Archivo a editar:** `backend/src/services/course.service.js`

Solo cambiá la función `list` dentro del objeto `courseService` (las demás quedan igual):

```js
// Reemplazá solo la propiedad "list:", dejá el resto del archivo igual
list: (params = {}) =>
  Course.find(buildListQuery(params))
    .select("code name credits classroomTypeRequired active prerequisites")  // ← NUEVO
    .populate("prerequisites", "code name")
    .sort({ code: 1 }),
```

---

## PASO 6 — Modificar `schedule.service.js` (proyección en consultas del motor CSP)

**Archivo a editar:** `backend/src/services/schedule.service.js`

Dentro de la función `generate`, reemplazá estas cuatro líneas:

**ANTES (lo que hay ahora):**
```js
const enrollments = await Enrollment.find(scheduleEligibleEnrollmentQuery())
  .populate("student")
  .populate("courses");
const teachers = await Teacher.find({ active: true }).populate("availableCourses");
const classrooms = await Classroom.find({ active: true, status: "AVAILABLE" });
const timeSlots = await TimeSlot.find({ active: true });
```

**DESPUÉS (reemplazalo por esto):**
```js
const enrollments = await Enrollment.find(scheduleEligibleEnrollmentQuery())
  .populate("student", "name code")           // ← NUEVO: solo campos necesarios
  .populate("courses", "_id code name classroomTypeRequired");  // ← NUEVO
const teachers = await Teacher.find({ active: true })
  .select("name availability availableCourses")  // ← NUEVO
  .populate("availableCourses", "_id");           // ← NUEVO
const classrooms = await Classroom.find({ active: true, status: "AVAILABLE" })
  .select("name capacity classroomType status");  // ← NUEVO
const timeSlots = await TimeSlot.find({ active: true })
  .select("day startTime endTime active");        // ← NUEVO
```

---

## PASO 7 — Modificar `AppRouter.jsx` (Lazy Loading en el frontend)

**Archivo a editar:** `frontend/src/routes/AppRouter.jsx`

Reemplazá **todo** el contenido con esto:

```jsx
import { lazy, Suspense } from "react";  // ← NUEVO: importar lazy y Suspense
import { BrowserRouter, Navigate, Route, Routes } from "react-router-dom";
import { AuthProvider } from "../context/AuthContext.jsx";
import RoleRoute from "./RoleRoute.jsx";
import GuestRoute from "./GuestRoute.jsx";

import AdminLayout from "../layouts/AdminLayout.jsx";
import TeacherLayout from "../layouts/TeacherLayout.jsx";
import StudentLayout from "../layouts/StudentLayout.jsx";

import LoginPage from "../pages/auth/LoginPage.jsx";

// Admin — páginas cargadas al inicio (pequeñas, siempre necesarias)
import DashboardPage from "../pages/dashboard/DashboardPage.jsx";
import AdminPlaceholderPage from "../pages/admin/AdminPlaceholderPage.jsx";
import UsersPage from "../pages/admin/UsersPage.jsx";
import CoursesPage from "../pages/courses/CoursesPage.jsx";
import TeachersPage from "../pages/teachers/TeachersPage.jsx";
import TeacherAvailabilityAdminPage from "../pages/teachers/TeacherAvailabilityPage.jsx";
import ClassroomsPage from "../pages/classrooms/ClassroomsPage.jsx";
import StudentsPage from "../pages/students/StudentsPage.jsx";
import TimeSlotsPage from "../pages/timeslots/TimeSlotsPage.jsx";
import EnrollmentsPage from "../pages/enrollments/EnrollmentsPage.jsx";
import RestrictionsPage from "../pages/restrictions/RestrictionsPage.jsx";
import SettingsPage from "../pages/settings/SettingsPage.jsx";

// NUEVO: páginas pesadas con Lazy Loading — solo se descargan cuando el usuario las visita
const SchedulesPage = lazy(() =>
  import("../pages/schedules/SchedulesPage.jsx")
);
const ScheduleGenerationPage = lazy(() =>
  import("../pages/schedules/ScheduleGenerationPage.jsx")
);
const ScheduleResultsPage = lazy(() =>
  import("../pages/schedules/ScheduleResultsPage.jsx")
);
const ClassroomSchedulePage = lazy(() =>
  import("../pages/schedules/ClassroomSchedulePage.jsx")
);

// Cuenta compartida
import AccountPage from "../pages/account/AccountPage.jsx";

// Docente (portal)
import TeacherHomePage from "../pages/teacher/TeacherHomePage.jsx";
import TeacherAvailabilityPage from "../pages/teacher/TeacherAvailabilityPage.jsx";
import TeacherCoursesPage from "../pages/teacher/TeacherCoursesPage.jsx";
import TeacherSchedulePage from "../pages/teacher/TeacherSchedulePage.jsx";
import TeacherProfilePage from "../pages/teacher/TeacherProfilePage.jsx";

// Alumno (portal)
import StudentHomePage from "../pages/student/StudentHomePage.jsx";
import StudentEnrollmentPage from "../pages/student/StudentEnrollmentPage.jsx";
import StudentEnrollmentValidationPage from "../pages/student/StudentEnrollmentValidationPage.jsx";
import StudentSchedulePage from "../pages/student/StudentSchedulePage.jsx";
import StudentProfilePage from "../pages/student/StudentProfilePage.jsx";

// NUEVO: fallback visual mientras carga el chunk lazy
function LoadingFallback() {
  return (
    <div className="flex h-40 items-center justify-center text-sm text-slate-400">
      Cargando...
    </div>
  );
}

export default function AppRouter() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route
            path="/login"
            element={
              <GuestRoute>
                <LoginPage />
              </GuestRoute>
            }
          />

          {/* ============ Administrador ============ */}
          <Route
            element={
              <RoleRoute roles={["ADMIN"]}>
                <AdminLayout />
              </RoleRoute>
            }
          >
            <Route path="/dashboard" element={<DashboardPage />} />
            <Route path="/users" element={<UsersPage />} />
            <Route path="/account" element={<AccountPage />} />
            <Route path="/courses" element={<CoursesPage />} />
            <Route path="/teachers" element={<TeachersPage />} />
            <Route
              path="/teachers/:id/availability"
              element={<TeacherAvailabilityAdminPage />}
            />
            <Route path="/classrooms" element={<ClassroomsPage />} />
            <Route path="/students" element={<StudentsPage />} />
            <Route path="/timeslots" element={<TimeSlotsPage />} />
            <Route path="/enrollments" element={<EnrollmentsPage />} />
            {/* NUEVO: rutas lazy envueltas en Suspense */}
            <Route
              path="/schedules"
              element={
                <Suspense fallback={<LoadingFallback />}>
                  <SchedulesPage />
                </Suspense>
              }
            />
            <Route
              path="/schedules/generate"
              element={
                <Suspense fallback={<LoadingFallback />}>
                  <ScheduleGenerationPage />
                </Suspense>
              }
            />
            <Route
              path="/schedules/results"
              element={
                <Suspense fallback={<LoadingFallback />}>
                  <ScheduleResultsPage />
                </Suspense>
              }
            />
            <Route
              path="/schedules/classroom"
              element={
                <Suspense fallback={<LoadingFallback />}>
                  <ClassroomSchedulePage />
                </Suspense>
              }
            />
            <Route path="/restrictions" element={<RestrictionsPage />} />
            <Route path="/settings" element={<SettingsPage />} />
          </Route>

          {/* ============ Docente ============ */}
          <Route
            element={
              <RoleRoute roles={["TEACHER"]}>
                <TeacherLayout />
              </RoleRoute>
            }
          >
            <Route path="/teacher" element={<Navigate to="/teacher/home" replace />} />
            <Route path="/teacher/home" element={<TeacherHomePage />} />
            <Route
              path="/teacher/availability"
              element={<TeacherAvailabilityPage />}
            />
            <Route path="/teacher/courses" element={<TeacherCoursesPage />} />
            <Route path="/teacher/schedule" element={<TeacherSchedulePage />} />
            <Route path="/teacher/profile" element={<TeacherProfilePage />} />
            <Route path="/teacher/account" element={<AccountPage />} />
          </Route>

          {/* ============ Alumno ============ */}
          <Route
            element={
              <RoleRoute roles={["STUDENT"]}>
                <StudentLayout />
              </RoleRoute>
            }
          >
            <Route path="/student" element={<Navigate to="/student/home" replace />} />
            <Route path="/student/home" element={<StudentHomePage />} />
            <Route path="/student/enrollment" element={<StudentEnrollmentPage />} />
            <Route
              path="/student/courses"
              element={<Navigate to="/student/enrollment-validation" replace />}
            />
            <Route
              path="/student/enrollment-validation"
              element={<StudentEnrollmentValidationPage />}
            />
            <Route path="/student/schedule" element={<StudentSchedulePage />} />
            <Route path="/student/profile" element={<StudentProfilePage />} />
            <Route path="/student/account" element={<AccountPage />} />
          </Route>

          <Route path="/" element={<Navigate to="/login" replace />} />
          <Route path="*" element={<Navigate to="/login" replace />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

---

## PASO 8 — Modificar `index.html` del frontend (suprimir petición favicon)

**Archivo a editar:** `frontend/index.html`

Reemplazá **todo** el contenido con esto:

```html
<!doctype html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" href="data:," />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>SGOHA</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

El cambio clave es `<link rel="icon" href="data:," />` — eso le dice al navegador "el favicon es vacío", así nunca hace la petición a `/favicon.ico`.

---

## CÓMO TOMAR LAS MÉTRICAS (capturas del antes y después)

### Captura del ANTES
1. **Revertí temporalmente** los cambios (o abrí el repo en el estado original de la rama sin estas modificaciones)
2. Corré el backend: `cd backend && npm run dev`
3. Abrí el frontend: `cd frontend && npm run dev`
4. Navegá la app: entrá al dashboard, andá a Cursos, generá un horario
5. Volvé a la terminal del backend y **tomá captura de pantalla** de los logs `[CO2]`
6. Presioná `Ctrl+C` en la terminal del backend → se imprime el **dashboard final**
7. **Tomá captura** del dashboard completo → esa es tu imagen "antes"

### Captura del DESPUÉS
1. Aplicá todos los cambios de los pasos 1 al 8 de esta guía
2. **Reiniciá** el backend: `Ctrl+C` y volvé a `npm run dev`
3. **Limpiá la caché del navegador** (F12 → Application → Clear storage → Clear site data)
4. Repetí exactamente las mismas acciones: dashboard → Cursos → generar horario
5. Volvé a la terminal y **tomá captura** de los logs `[CO2]`
6. Presioná `Ctrl+C` → se imprime el dashboard final
7. **Tomá captura** → esa es tu imagen "después"

### Subir las imágenes
1. Entrá a https://ibb.co
2. Subí las dos capturas
3. Copiá los links directos (terminan en `.jpg` o `.png`)
4. Pegá esos links en el MD reemplazando los textos `[ver imagen adjunta — antes.jpg]`

---

## RESUMEN: qué archivo cambia y qué hace cada cambio

| Archivo | Tipo de cambio | Técnica Green |
| :--- | :--- | :--- |
| `backend/src/middlewares/co2.middleware.js` | Archivo **nuevo** | Medición CO2 con CO2.js |
| `backend/src/app.js` | Modificación | Compresión Gzip + favicon 204 + CO2 middleware |
| `backend/src/server.js` | Modificación | Dashboard CO2 al cerrar |
| `backend/src/controllers/course.controller.js` | Modificación | Caché HTTP (Cache-Control) |
| `backend/src/services/course.service.js` | Modificación | Proyección MongoDB `.select()` |
| `backend/src/services/schedule.service.js` | Modificación | Proyección MongoDB en consultas CSP |
| `frontend/src/routes/AppRouter.jsx` | Modificación | Lazy Loading con React.lazy + Suspense |
| `frontend/index.html` | Modificación | Suprimir petición favicon |
