# Bitácora del proyecto — SafePoint

Registro cronológico de todo lo desarrollado y **fusionado a `master`**, desde el inicio hasta hoy.

- **Rango:** 24-jun-2026 → 11-sep-2026 (~2 meses y medio).
- **Volumen:** **63 Pull Requests** fusionados (#1–#66; los nº faltantes se cerraron o consolidaron) en `master`.
- **Fuentes:** historial de git + PRs de GitHub + `CONTEXTO-PROYECTO.md`.
- **Flujo de trabajo:** ramas `feat/*` → se fusionan en `integracion/local` (rama local, no se sube) → se abren PRs desde ramas `release/*` sobre `master`. Regla: **NUNCA push directo a `master`**.
- **Stack:** Laravel 13 · Livewire 4 · Blade · Bootstrap 5.3 (CDN) · Chart.js · MySQL. Sin React/Vue ni build de Vite para la UI.

> **Cómo leerla:** la sección 1 agrupa por módulo (para entender el *qué*); la sección 2 es la cronología PR por PR (para el *cuándo*); la sección 3 resume las notas de despliegue acumuladas.

---

## 1. Resumen por módulo

### Fundamentos e infraestructura
- **#1** Esquema ER: modelos, migraciones y seeders base.
- **#2** Internacionalización ESP/ENG.
- **#7** Usuarios, roles y permisos (RBAC con spatie/laravel-permission; el rol `Rol` extiende el de Spatie).
- **#20** Tema oscuro (`data-bs-theme`, persistido en localStorage).
- **#21** Release consolidado 2026-07 (BAI/BDI-II, interpretación/matriz/export DASS-21, demográficos, Informe de Seguimiento PDF, ficha/expediente, permisos de pacientes, imagen de charla).

### Proyectos · Módulos · Contenidos
- **#4** Gestión jerárquica de proyectos → módulos → contenidos.
- **#5** Usuarios por proyecto + alcance (un paciente solo ve el contenido de sus proyectos).
- **#8** Buscador de contenidos; **#9** fix: los selectores permiten escribir de inmediato.
- **#33** Reordenar módulos y contenidos con arrastrar y soltar.
- **#44** "Por página" en vivo (Livewire) en la lista de contenidos del módulo.
- **#41** Contenidos con período (fecha inicio/fin) asociables al Plan/Gantt.

### Feed e interacción (estilo red social)
- **#3** Publicación programada de contenidos.
- **#6** Feed con likes y comentarios.
- **#12** Adjuntos en los comentarios.
- **#31** Visor de imagen tipo Facebook; **#32** visor de PDF tipo Facebook.
- **#35** "Mi respuesta" y botón "Responder" en el feed.
- **#36** Contenido "Apartado" interactivo (el paciente responde con texto/adjuntos).

### Charlas
- **#10** Charlas con registro de asistencia y foto de evidencia.
- **#14** fix: la foto se guarda como ruta relativa `/storage/...`.
- **#22** Charlas presenciales o virtuales (modalidad).
- **#30** Charlas en el feed.
- **#40** Charlas pasan a ser un **tipo de contenido** dentro de Contenidos.

### Cuestionarios y escalas psicométricas
- **#11** DASS-21 (config-driven + calificador).
- **#17** Asignación masiva (por proyecto o a todos).
- **#40** **Cuestionarios anónimos** por campaña (enlace/QR): CEAL-SM (motor propio) y NOSACQ-50 (reusa el motor clínico); solo-anónimo.
- **#42** Asistente por pasos + autosave (localStorage) en los formularios anónimos largos.
- **#43** CEAL Resultados: descargas PNG, tabla a color, filtros por zona/demográficos, MFRPS.
- **#46** Filtro por **dimensión** (CEAL) + filtros y descargas PNG en NOSACQ-50 + guardia de anonimato.

### Citas psicológicas
- **#13** Agendamiento de citas + horarios.
- **#16** fix: el psicólogo solo ve/gestiona sus citas; el admin ve todas.
- **#39** fix(Jitsi): entrar a la sala con el usuario del sistema, sin prejoin.
- **#47** El paciente puede **cancelar** (con motivo + aviso al psicólogo) y **reprogramar** (no destructivo); aviso de la ventana de 24h.

### Plan de trabajo (cronograma tipo Gantt)
- **#23 / #49 / #50** Comentarios por semana en el cronograma y por contenido (también en el modal).
- **#26** Tipo de comentario (comentario / reporte).
- **#27** Calificación por persona/contenido + ranking con empates.
- **#28** Release Plan/Gantt (seguimiento, participación, alcance general + filtro, dashboards).
- **#45 / #51** Descargar el cronograma como imagen PNG (con la columna "Actividad" a la izquierda).
- **#41** Contenidos con período pintados como barras en el Gantt.

### Pacientes · Dashboard · Informe de seguimiento
- **#18** Gestión de pacientes (carga masiva Excel, ficha).
- **#19** Dashboard de progreso de pacientes.
- **#34** Informe psicológico con campos editables (plantilla + por informe).
- **#48** fix: el "Volver" del informe respeta el origen (regresa a la ficha del paciente).

### Ranking · Calificación · Recordatorios
- **#25** Ranking trimestral de participación por proyecto.
- **#27** Calificación por persona/contenido.
- **#15** Panel de recordatorios por rol (campanita de notificaciones).

### UX y diseño
- **#42** Perfil con foto, feed por recencia, navegación mejorada.
- **#43** Perfil tipo Facebook (portada), ranking con foto, fondo difuminado en modo oscuro, login en modo oscuro.

### Experiencia móvil y PWA (#52)
- **Navegación móvil** (<992px): barra inferior fija por rol (4 accesos + ☰ Más, con badges) + **bottom-sheet** deslizable (overlay + arrastrar para cerrar) con el menú completo. Desktop intacto. Menú extraído a `partials/nav-menu` (reusado por sidebar y sheet).
- **PWA instalable**: `manifest.webmanifest` (standalone, íconos 192/512 + maskable), service worker conservador (red primero + página offline), `apple-touch-icon`. *Requiere HTTPS en producción.*
- **Panel del paciente** rediseñado (tarjetas accionables: próxima cita / cuestionarios / progreso) + hero con la **foto de portada** del usuario.
- **Inicio por rol**: el paciente siempre arranca en el Feed (helper `User::rutaInicio()`), también desde `/`.

### Seguridad (auditoría inspirada en el pentest de Astra)
- **#47** Fix de **fuga del padrón de pacientes** (citas modo gestor gate demasiado amplio) + **adjuntos de paciente en disco privado** con descarga autenticada.

### Rendimiento
- **#47** **Feed paginado en la base de datos** (UNION) en vez de materializar todo en PHP.

### Fixes varios
- **#9** buscador, **#14** ruta de foto de charla, **#48** crear proyecto sin descripción (columna nullable) + navegación del informe.

### Releases consolidados (agrupan varias features)
- **#21** consolidado 2026-07 · **#28** Plan/Gantt · **#29** mejoras y deuda · **#30** charlas/feed.

---

## 2. Cronología (PR por PR)

| PR | Fecha | Título |
|----|-------|--------|
| #1 | 2026-06-25 | Modelos, migraciones y seeders del esquema ER |
| #2 | 2026-06-30 | Internacionalización ESP/ENG |
| #3 | 2026-06-30 | Publicación programada |
| #4 | 2026-07-02 | Gestión jerárquica de proyectos, módulos y contenidos |
| #5 | 2026-07-02 | Usuarios por proyecto + alcance |
| #6 | 2026-07-02 | Feed: like y comentarios |
| #7 | 2026-07-02 | Usuarios, roles y permisos |
| #8 | 2026-07-02 | Buscador de contenidos |
| #9 | 2026-07-02 | fix: el buscador permite escribir de inmediato |
| #10 | 2026-07-02 | Charlas con asistencia y foto de evidencia |
| #11 | 2026-07-08 | Cuestionarios DASS-21 |
| #12 | 2026-07-08 | Adjuntos en comentarios |
| #13 | 2026-07-08 | Citas psicológicas |
| #14 | 2026-07-08 | fix: foto de charla como ruta relativa |
| #15 | 2026-07-08 | Panel de recordatorios |
| #16 | 2026-07-30 | fix: el psicólogo solo ve sus citas; el admin todas |
| #17 | 2026-07-30 | Asignación masiva de cuestionarios |
| #18 | 2026-07-30 | Gestión de pacientes |
| #19 | 2026-07-30 | Dashboard de progreso |
| #20 | 2026-07-30 | Tema oscuro |
| #21 | 2026-07-31 | Release consolidado 2026-07 |
| #22 | 2026-08-03 | Charlas presenciales o virtuales (modalidad) |
| #23 | 2026-08-03 | Comentarios por semana en el cronograma |
| #25 | 2026-08-03 | Ranking trimestral de participación |
| #26 | 2026-08-04 | Tipo de comentario (comentario/reporte) en el cronograma |
| #27 | 2026-08-04 | Calificación por persona/contenido + ranking por empates |
| #28 | 2026-08-04 | Release Plan/Gantt |
| #29 | 2026-08-05 | Release mejoras y deuda |
| #30 | 2026-08-07 | Release charlas/feed |
| #31 | 2026-08-12 | Visor de imagen tipo Facebook |
| #32 | 2026-08-12 | Visor de PDF tipo Facebook |
| #33 | 2026-08-12 | Reordenar módulos y contenidos (drag & drop) |
| #34 | 2026-08-12 | Informe psicológico con campos editables |
| #35 | 2026-08-12 | Feed: "Mi respuesta" y botón "Responder" |
| #36 | 2026-08-14 | Contenido "Apartado" interactivo |
| #39 | 2026-08-24 | fix(Jitsi): entrar con el usuario del sistema |
| #40 | 2026-08-24 | Cuestionarios anónimos (CEAL-SM + NOSACQ-50) y Charlas en Contenidos |
| #41 | 2026-08-25 | Contenidos con período en el Plan/Gantt |
| #42 | 2026-08-25 | UX: perfil con foto, feed por recencia, wizard anónimo, navegación |
| #43 | 2026-08-26 | UX: perfil tipo Facebook, ranking con foto, dark difuminado, login dark, CEAL filtros/PNG |
| #44 | 2026-08-26 | "Por página" en vivo (Livewire) en módulos |
| #45 | 2026-08-26 | Descargar el cronograma como PNG |
| #46 | 2026-08-27 | Resultados anónimos: filtro por dimensión (CEAL) + filtros/PNG en NOSACQ |
| #47 | 2026-08-28 | Seguridad (padrón + adjuntos privados), Feed paginado en BD, gestión (eliminar + citas cancelar/reprogramar) |
| #48 | 2026-08-31 | Fixes: crear proyecto sin descripción + navegación "Volver" del informe |
| #49 | 2026-08-31 | Comentarios por semana en contenidos + volver al plan al editar |
| #50 | 2026-09-02 | Plan: comentarios de contenido en el modal + fix filtro y orden |
| #51 | 2026-09-02 | Plan: imagen del cronograma con la columna "Actividad" a la izquierda |
| #52 | 2026-09-03 | UX móvil: navegación inferior + PWA instalable + Panel rediseñado + inicio por rol |
| #53 | 2026-09-09 | Forzar HTTPS en las URLs cuando `APP_URL` es https (producción con SSL) |
| #54 | 2026-09-09 | Editar módulos (nombre + descripción) con botón ✏️ en listado, proyecto y módulo |
| #55 | 2026-09-09 | Fix: paciente sin proyecto veía 404 en Ranking → página amable + bottom-nav coherente |
| #56 | 2026-09-08 | Feed: texto del cuerpo justificado en las tarjetas de contenido (equipo) |
| #57 | 2026-09-08 | Feed: abrir la imagen adjunta del apartado en el visor ampliado (equipo) |
| #58 | 2026-09-08 | Apartado: "Volver" regresa a la página anterior (feed/módulo) (equipo) |
| #59 | 2026-09-08 | Modo libre por contenido (`es_libre`): comentarios públicos sin calificación ni ranking (equipo) |
| #60 | 2026-09-09 | Resultados: filtros por demográficos numéricos (edad, N.º de hijos) por rango, CEAL + NOSACQ |
| #61 | 2026-09-10 | Resultados CEAL: selector de tipo de gráfico (7 vistas, sin recargar) (equipo) |
| #62 | 2026-09-10 | CEAL: 4 gráficos de participación/demografía (Participantes, género, sector, riesgo por zona) |
| #63 | 2026-09-10 | Tabla de detalle de resultados adaptada al tema (claro/oscuro) |
| #64 | 2026-09-10 | "Riesgo por zona" se muestra sin mínimo por zona |
| #65 | 2026-09-10 | Descarga en cuadro + texto por tema + iOS; réplica de demografía a NOSACQ (selector único) |
| #66 | 2026-09-11 | CEAL: "Detalle por dimensión" como opción del selector de gráfico |

---

## 3. Notas de despliegue acumuladas

Al desplegar a producción (o al poner al día un entorno), lo aditivo/obligatorio:

- **`composer install`** — nueva dependencia `ext-zip` (PhpSpreadsheet / carga masiva Excel) desde #21; `bacon/bacon-qr-code` desde #40 (QR de cuestionarios anónimos).
- **`php artisan migrate`** — muchas migraciones aditivas a lo largo del proyecto. Las más sensibles recientes:
  - #47: mueve **adjuntos a disco privado** (idempotente) + `citas.motivo_cancelacion`.
  - #48: `proyectos.descripcion` → **nullable** (sin esto revienta crear proyecto sin descripción).
- **`php artisan db:seed --class=PermissionSeeder`** — tras #47, el permiso de **eliminar cuestionarios** queda solo para Administrador.
- **`php artisan storage:link`** — para servir `/storage` (avatares, portadas, recursos educativos).
- **`php artisan optimize`** (config/route/view cache) — con **opcache** activo en el servidor, mejora notable de tiempos. (En prod, NO `optimize:clear`.)

## 4. Pendientes en cola (no fusionados)

- ✅ **Navegación móvil + PWA — HECHO (#52).** ✅ **HTTPS en producción — HECHO (#53):** la app está
  publicada en `safepoint.internationalsos-peru.com` con SSL, la PWA ya es instalable. Pendiente la
  **fase 2** de móvil: adaptar las ~35 vistas con **tablas anchas** (tarjetas apiladas), el **Gantt** y los **gráficos**.
- **Encolar los correos de citas** (`Mail::queue()` + worker `queue:work`), hoy son síncronos. Parqueado hasta confirmar que el servidor mantiene un worker.
- **Paginar en BD** `DashboardProgresoController` (aún materializa en PHP) cuando crezca el nº de pacientes.
- **Método de despliegue GitHub→servidor**: se despliega por **`git pull` en el servidor** (confirmado 2026-09); tras el pull, `php artisan migrate --force` (si hay migración) + `php artisan optimize`; en móvil, *Unregister* del service worker si cambió `sw.js`.

---

*Detalle técnico ampliado por módulo, decisiones y deuda: ver `CONTEXTO-PROYECTO.md` (§5–§17).*
