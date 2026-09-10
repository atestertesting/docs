# Contexto del proyecto — SafePoint

> Documento de contexto autocontenido para entregar a otra IA / desarrollador.
> Resume arquitectura, decisiones, funcionalidades y estado del repositorio.
> Última actualización: **13 de agosto de 2026**.
>
> **Novedades (13-ago-2026):**
> - **En 3 PRs abiertos** — mergear en orden `#36 → #37 → #38`, sin conflictos (probado):
>   - **#36 `release/apartado-2026-08`** — nuevo tipo de contenido **"Apartado"**: consigna de
>     texto libre + adjuntos mixtos (word/excel/pdf/imagen/video); es **interactivo**: cada
>     paciente envía **su** respuesta (texto + adjuntos) y el gestor la revisa. En apartados se
>     oculta la barra de comentarios del feed.
>   - **#37 `release/experiencia-paciente-2026-08`** — **experiencia por rol** (el paciente
>     *entra* al proyecto, no ve "Usuarios del proyecto", solo módulos publicados/contenidos
>     vigentes; **menú por rol** vía `config/menu.php`, oculta Módulos/Contenidos del menú) +
>     **rediseño "taller" en tarjetas** de Proyectos/Módulos/Contenidos (role-aware, responsive) +
>     **reordenar drag-drop** integrado **sobre las tarjetas** (arrastrar el icono de la tarjeta).
>   - **#38 `release/cuestionarios-2026-08`** — **cuestionarios pendientes en notificaciones** y
>     el botón **"Volver"** de la revisión regresa a la **ficha del paciente**.
> - **Aportes del equipo ya en `master`** (PRs #31–#35): **visor de imagen y de PDF tipo Facebook**
>   en el feed (lightbox con comentarios al costado), **reordenar** módulos/contenidos con
>   **arrastrar y soltar**, **Informe Psicológico con campos editables** (plantilla configurable),
>   y el relabel del feed **"Mi respuesta / Responder"**.
> - **Herramientas:** `gh` CLI **ya instalado** y autenticado (ver sección 11).
>
> ---
>
> Lo **mergeado a `master`** (PRs previos): **escalas
> BAI/BDI-II** + interpretación/matriz/export del DASS-21, **datos demográficos** del paciente,
> **Informe Psicológico de Seguimiento** (PDF), **ficha/expediente del paciente**, **permisos
> PACIENTES**, **gestor programa citas**, imagen de referencia en charlas, y el **Plan de trabajo
> tipo Gantt COMPLETO**: cronograma, asistencia, **seguimiento preventivo (3c)**, **participación
> integral**, **alcance general** + filtro por proyecto, y dashboards con Chart.js.
> **Mergeado a `master` (PR #29 `release/mejoras-y-deuda-2026-08`, 10 commits; ver sección 11):**
> **Jitsi propio embebido** (iframe) con botón *Unirse*, **cronograma reactivo** (enfoque a la
> semana actual, scroll preservado, marcar días por AJAX), **notificaciones por polling**, y la
> **deuda técnica de más valor resuelta**: throttle de login + clave temporal configurable con
> cambio obligatorio, `Cita::scopeDesde()` con índice, `ext-zip` declarada, seeders demo y panel
> de cuentas demo **solo fuera de producción**, y una **suite de tests** (login/throttle, RBAC,
> scope de citas) con **aislamiento de BD blindado**.
> *(Aportes del equipo en master: comentarios/bitácora del cronograma, avance de talleres por
> semana, modalidad presencial/virtual, ranking trimestral, calificaciones.)*

---

## 1. Resumen ejecutivo

**SafePoint** es una plataforma web de **acompañamiento psicológico y clima
laboral**. Administradores y moderadores (psicólogos) gestionan proyectos, módulos
de entrenamiento y contenidos (videos, imágenes, documentos, audios, reuniones);
los usuarios consumen el contenido publicado en un **feed estilo red social** (con
me gusta, comentarios y **adjuntos**), responden **cuestionarios/escalas
psicométricas** que se les asignan, **agendan citas psicológicas** y participan en
**charlas**. Cada rol ve **recordatorios/notificaciones** adaptados a su función.

El control de acceso es por **roles y permisos (RBAC)** con tres roles:
Administrador, Moderador (psicólogo) y Usuario.

---

## 2. Stack técnico

| Componente | Detalle |
|---|---|
| Framework | **Laravel 13** (`laravel/framework ^13.8`) |
| Lenguaje | **PHP 8.3** |
| Base de datos | **MySQL**, BD `psicologia` |
| Entorno | **Laragon** en Windows (shell: PowerShell / Git Bash) |
| Auth/RBAC | **spatie/laravel-permission 8** |
| UI reactiva | **Livewire 4** (usuarios/roles, gestión de charla, horarios, agendar cita) |
| Multimedia | `james-heinrich/getid3` (metadatos audio); storage local (`storage:link`) |
| Importación | **phpoffice/phpspreadsheet 5.8** (carga masiva de pacientes `.xlsx`/CSV, export de matriz de resultados) — **requiere `ext-zip`** en PHP |
| PDF | **barryvdh/laravel-dompdf** (Informe Psicológico de Seguimiento en PDF) |
| Gráficos | **Chart.js 4** por CDN (diagramas radar de cuestionarios) |
| Calendario | **flatpickr 4.6** por CDN (agendar cita: bloquea domingos y feriados) |
| Videollamada | **Jitsi self-hosted** del cliente (`jitsi-meet.internationalsos-peru.com`, sin auth), sala autogenerada `SafePoint-…`. Config en `config/citas.php`. **Embebido** vía External API (`external_api.js` + `JitsiMeetExternalAPI`) en un modal, con *fallback* a nueva pestaña si el servidor bloquea el iframe |
| Notificaciones | **Polling** (sin WebSockets/Reverb): `BROADCAST_CONNECTION=log`; la campanita consulta un endpoint cada N segundos (`RecordatoriosController`) |
| Correo | `Mail::to()->send()` (confirmación de cita, best-effort en try/catch) |
| Tests | **PHPUnit** contra **BD MySQL dedicada `psicologia_test`** (nunca la real); ver secciones 11 y 14 |
| Frontend | **Bootstrap 5.3 + Bootstrap Icons por CDN** (sin build); **Alpine.js** (vía Livewire) para el visor tipo Facebook del feed; **SortableJS 1.15** por CDN para reordenar módulos/contenidos arrastrando |
| Estilos propios | `public/css/corporate.css` (con cache-busting `?v=filemtime`) |
| i18n | ES/EN con `__()` + `lang/en.json` |
| Tooling | Pint (formato). Vite+Tailwind presentes pero **no usados** en las vistas |

> ⚠️ **Importante:** las vistas Blade cargan **Bootstrap por CDN** y un CSS
> corporativo en `public/css/`. **No hace falta `npm run build`** para la UI.
> Livewire sí requiere `composer install` (su JS se sirve en una ruta con prefijo
> hasheado, p. ej. `/livewire-xxxx/livewire.js`).

---

## 3. Identidad visual (obligatoria)

- Nombre de la app en toda la UI: **SafePoint**.
- Paleta corporativa **actualizada — paleta "Sereno" (azul + celeste)**:
  - Blanco `#ffffff`
  - **Azul marino `#233a72`** (marca: sidebar, botones, títulos). Se distingue el
    **relleno** de marca `--corp-brand` (sólido, fijo en ambos temas) del **acento de
    texto** `--corp-indigo` (se aclara en modo oscuro para legibilidad de títulos).
  - **Celeste `#2f9bd8`** (acento: iconos, badges, hover de enlaces, `.btn-corp-accent`);
    tokens `--corp-accent` / `--corp-accent-600`. Los alias **`--corp-orange*` siguen
    existiendo** apuntando al celeste (compatibilidad con vistas previas).
  - *Nota histórica:* la paleta original era Indigo `#232762` + Naranja `#f99e16`; se
    cambió el naranja (recordaba a la marca de un banco) → teal → **celeste**, y el
    índigo se **suavizó** a azul marino.
- **Tema claro/oscuro** con el modo nativo de Bootstrap 5.3 (`data-bs-theme` en `<html>`):
  conmutador en la topbar, preferencia en `localStorage`, **sin parpadeo** (script en
  `<head>`), respeta `prefers-color-scheme`. El bloque `[data-bs-theme="dark"]` de
  `corporate.css` redefine los tokens `--corp-*`. *(Pendiente: el login sigue en claro.)*
- Tokens y componentes en `public/css/corporate.css`
  (`--corp-brand`, `--corp-indigo`, `--corp-accent`, `.btn-corp`, `.card-corp`, `.sidebar-*`, `.feed-*`).
- Dominio y textos **en español** (con `__()` para permitir EN).

---

## 4. RBAC — control de acceso (decisión clave)

- `App\Models\Rol` **extiende** `Spatie\Permission\Models\Role` y usa la tabla `rols`.
- ⚠️ **Decisión no obvia:** Spatie tiene **hardcodeada** la columna `name`, por lo
  que la columna original `nombre` se **renombró a `name`** (valores en español:
  *Administrador / Moderador / Usuario*). Se conservó la columna extra `activo`.
- `config/permission.php`: `models.role = Rol::class`, `table_names.roles = 'rols'`,
  `column_names.role_pivot_key = 'rol_id'`.
- `App\Models\User` usa el trait `HasRoles`; helpers `esGestor()` (Admin/Moderador),
  `proyectoIds()`, `puedeVerContenido()`, `moderador()`, `citas()`, `charlas()`,
  `citasProximas()`, `proximaCharla()`, `recordatorios()`.

### Roles y permisos
- Permisos `{recurso}.{accion}` × (`ver, crear, editar, eliminar`). Recursos:
  `usuarios, pacientes, roles, proyectos, modulos, contenidos, cuestionarios, reuniones,
  documentos, videos, imagenes, evaluaciones, citas, charlas, feriados, psicologos, planes`.
- **`pacientes` separado de `usuarios`** (decisión): las rutas/vistas de pacientes usan
  `pacientes.*` (antes reusaban `usuarios.*`), para que el **Moderador vea la ficha del
  paciente sin acceder a la gestión del staff**. Migración **aditiva** (`givePermissionTo`,
  no `syncPermissions`) para no resetear permisos existentes.
- **Administrador es super-admin** vía `Gate::before()` en `AppServiceProvider`.
- **Moderador (psicólogo):** gestiona contenido/módulos/cuestionarios/**charlas**,
  **planes** (`planes.*`), **ve pacientes** (`pacientes.ver`) y `citas.ver/crear/editar`
  (ahora **también agenda** citas para sus pacientes; ver sección 8).
- **Usuario (paciente):** `.ver` de consulta + `citas.crear` (agenda sus citas).
- Middleware en `bootstrap/app.php`: `role`, `permission`, `role_or_permission`.
- **Nota:** el subsistema *evaluaciones* existe en BD/permisos pero se **quitó de la
  UI** por ser redundante con *cuestionarios* (decisión de producto).

---

## 5. Modelo de datos (entidades relevantes)

**Jerarquía de contenido**
- **Proyecto** → muchos **Módulos**; pivote `proyecto_user` (miembros del proyecto).
- **ModuloEntrenamiento** (`modulos_entrenamiento`) → pertenece a Proyecto; `publicado`.
- **Contenido** (`contenidos`) → pertenece a un Módulo. `tipo` (`video, documento,
  imagen, reunion, evaluacion, audio, apartado`), `publicado`, `publicar_en` (programada),
  `orden` (reordenable por drag-drop), `cuerpo` (consigna del apartado, nullable).
  `hasMany` → Video/Imagen/Documento/Audio; `hasOne` → Reunion/EvaluacionPsicologica.
- **Contenido tipo `apartado`** (PR #36) — contenido **interactivo**: el gestor define una
  **consigna** (`cuerpo`, texto libre) y adjuntos de referencia; cada paciente envía **su propia
  respuesta**. Modelos: **`Adjunto`** (polimórfico-ligero: `contenido_id` + `respuesta_id`
  nullable; `esImagen()`/`esVideo()`) y **`ApartadoRespuesta`** (`contenido_id`, `user_id`,
  `texto`, con adjuntos). `Contenido->cuerpo/adjuntos()/respuestas()/respuestaDe($user)`.
  `App\Services\GuardadorAdjuntos` guarda URLs **relativas** `/storage/...` (word/excel/pdf/
  imagen/video, máx. 20 MB c/u). Rutas `apartado.responder` (POST) / `apartado.respuestas` (GET).
- **Reordenar (drag-drop, aporte del equipo, PR #33)** — `Contenido::reordenar()` y
  `ModuloEntrenamiento::reordenar()` reasignan `orden` de forma consecutiva desde la menor
  posición del conjunto (respeta paginación); un no-gestor solo reordena lo de sus proyectos.
- **Like** y **Comentario** → interacción del feed. **Comentario** ahora soporta
  **adjunto** (`adjunto_url`, `adjunto_nombre`; `cuerpo` nullable → permite adjunto solo).
  En el feed la acción de comentar se muestra como **"Mi respuesta / Responder"** (relabel del
  equipo); en contenidos tipo **apartado** esa barra se **oculta** (la interacción es la respuesta).

**Alcance por proyecto:** feed y dashboard del Usuario se filtran por sus proyectos
(`proyecto_user`). Admin/Moderador (`esGestor`) ven todo.

**Cuestionarios / escalas** — ver sección 7. La **asignación** es un modelo propio
`CuestionarioAsignacion` (`cuestionario_asignaciones`: `cuestionario_id`, `user_id`,
`asignado_por`, `estado` `pendiente|completado`, `completado_at`, `es_directivo`).
Se asigna **individual** o **masivo por proyecto/todos** (`CuestionarioController::asignarMasivo`,
idempotente). Las respuestas van en `CuestionarioRespuesta` (`asignacion_id`, `pregunta_id`, `valor`).

**Pacientes** (rol `Usuario`) — pantalla dedicada `PacienteController` (separada de
*Usuarios*), rol **fijo** *Usuario* sin selector, con **carga masiva** desde Excel
`.xlsx` o CSV con **previsualización** (`App\Livewire\CargaMasivaPacientes` +
`App\Services\ImportadorPacientes`, detección de `.xlsx` por firma `PK` y lectura con
PhpSpreadsheet). Contraseña temporal por defecto (ver deuda técnica, sección 14).
- **Datos demográficos/laborales** en `users` (nullable): `sexo` (Masculino/Femenino),
  `fecha_nacimiento` (date), `puesto`, `area`. Helper `User::edad()` (derivada de la fecha).
  Se capturan en el formulario y en la carga masiva (el importador normaliza sexo M/F y
  fecha). Alimentan la columna "Puesto" de la matriz de resultados y preparan baremos (MCMI-II).
- **Ficha/expediente del paciente** (`PacienteController::show`, `pacientes.show`,
  `pacientes.ver`): página con pestañas **Datos · Informes · Cuestionarios · Citas**.
  Los informes se consultan por `user_id` (a través de sus citas); la pestaña
  Cuestionarios reusa `CalificadorCuestionario` para mostrar puntaje + nivel.

**Informe Psicológico de Seguimiento** (`informes_seguimiento`, 1:1 con una **cita
completada**) — ver sección 15. Snapshot de identificación/firma + 8 secciones (I–VIII),
`estado_informe` (`borrador|finalizado`). `Cita::informe()` y `Cita::admiteInforme()`
(solo citas `completada`). **Campos editables (aporte del equipo, PR #34):** una **plantilla
configurable** define los campos por defecto de cada informe nuevo — modelos **`CampoInforme`**
(plantilla global: título, tipo `texto libre|opción múltiple|escala`, opciones, min/max) y
**`SeccionInforme`** (campos concretos por informe); Livewire `GestionCamposInforme`
(`informes.plantilla`) y `RedactarInforme`.

**Plan de trabajo / Cronograma tipo Gantt** (`planes` + `plan_actividades`) — ver
sección 16. Plan por **proyecto** con actividades por **bloque**; `plan_actividades.charla_id`
(opcional) vincula una actividad-taller con una **Charla** para consolidar asistencia.

**Citas psicológicas** (agendamiento)
- **Cita** (`citas`): `user_id`, `moderador_id`, `modalidad` (`presencial|virtual`),
  `fecha`, `hora`, `hora_fin`, datos del paciente (`paciente_nombre/contacto/email`),
  `tipo` (`primera_vez|seguimiento`), `motivo`, `enlace_virtual`, `estado`
  (`pendiente|confirmada|cancelada|completada`). Métodos: `esVirtual()`, `inicio()`,
  `puedeCancelar()` (respeta anticipación mínima configurable).
- **Feriado** (`feriados`): `fecha` (unique), `nombre`. **CRUD, no hardcodeado**
  (seeder con feriados Perú 2026). Bloquea el agendamiento en esas fechas.
- **Horario** (`horarios`): `moderador_id`, `dia` (1-6 ISO), `modalidad`,
  `hora_inicio`, `hora_fin`. Disponibilidad por psicólogo.
- **Moderador** = perfil de un User (psicólogo). Añadidos: `foto`, relación
  `horarios()`, `crearHorarioPorDefecto()` (siembra 6 días × 2 modalidades desde config).

**Charlas**
- **Charla** (`charlas`): `titulo`, `descripcion`, `fecha` (datetime), `estado`
  (`pendiente|finalizada`), `foto` (**evidencia**, tras finalizar), `imagen` (**referencia**,
  la sube quien crea la charla), `creado_por`. Pivote `charla_user` con `asistio`.
  `estaFinalizada()`. En el feed se muestra la evidencia si existe, si no la de referencia.

---

## 6. Funcionalidades

> Los ítems **1–30** están en `master`. Los **25–30** llegaron en el PR #29
> **`release/mejoras-y-deuda-2026-08`**. Los **31–33** son aportes del **equipo** ya en `master`
> (PRs #31–#35); los **34–36** están en **PRs abiertos nuestros** (#36/#37/#38). Ver sección 11.

1. **Login** Bootstrap split-screen + credenciales demo. Post-login: Usuario → feed;
   Moderador/Admin → dashboard.
2. **Navigation drawer** (sidebar) filtrado por permiso; el filtro acepta permiso
   `null`, string (`can`) o **Closure** (condición dinámica). Grupos: Principal,
   Gestión, Contenido y seguimiento, Agendamiento.
3. **Dashboard** role-aware: gestores ven permisos; usuarios ven su **progreso** de
   contenido. **Franja de recordatorios** arriba (ver sección 10). "Agendar cita"
   solo para pacientes.
4. **Proyectos / Módulos / Contenidos** — CRUD, miembros, publicar, multimedia al
   storage, publicación programada (`publicar_en`), buscador.
5. **Feed** estilo red social — contenido vigente + del alcance del usuario; **me
   gusta**, **comentarios con adjuntos** (con **foto de perfil** vía `<x-avatar>`), y
   **charlas** (finalizadas con foto = registro; programadas = anuncio con ancla
   `#charla-ID`). **Orden: por lo más reciente CREADO** (charlas y contenidos
   entremezclados; ya no se fijan las charlas al inicio). Gestores ven todas las
   charlas con pie de gestión.
6. **Usuarios / Roles y permisos** — tablas Livewire reactivas.
7. **i18n ES/EN** — `SetLocale` (usuario > sesión > default); textos con `__()`.
   ⚠️ Evitar `trans_choice` (aplica fallback a EN); usar `__()` con ternario para plurales.
8. **Cuestionarios / escalas** — sección 7.
9. **Citas psicológicas + horarios** — sección 8.
10. **Charlas** — sección 9.
11. **Recordatorios / notificaciones por rol** — sección 10.
12. **Pacientes + carga masiva Excel/CSV** — sección 5. Rol fijo
    *Usuario*, previsualización antes de importar, plantilla `.xlsx` descargable.
13. **Dashboard ejecutivo de progreso** — `/dashboard/progreso` (ruta
    `progreso`, solo gestores). KPIs globales, **avance general** con dona y
    **distribución por nivel**, **progreso por proyecto**, lista **"Requieren atención"**
    (pacientes sin actividad reciente), **cronograma** de actividades y **detalle por
    paciente**. "Contenido realizado" se infiere del **comentario** (mismo criterio que
    el panel del paciente); cuestionarios y charlas usan datos reales. Enlace en el
    sidebar dentro del grupo **Principal**.
14. **Asignación masiva de cuestionarios** por proyecto o a todos.
15. **Alcance de citas por psicólogo**: el **Moderador** solo ve/gestiona
    **sus** citas (`moderador_id`); el **Administrador** ve todas (`CitaAdminController`).
16. **Tema claro/oscuro + paleta "Sereno"** — sección 3.
17. **Escalas BAI y BDI-II** *(PR consolidado)* — sección 7. Nuevos `tipo` en
    `config/cuestionarios.php` + seeders (`CuestionarioBaiSeeder`, `CuestionarioBdiSeeder`).
    BDI-II usa **opciones por ítem** (`preguntas_cuestionario.opciones`).
18. **DASS-21: interpretación + matriz por persona + Excel** *(PR consolidado)* — texto de
    interpretación por nivel, tabla "por persona" (`App\Services\MatrizResultados`) y
    **exportación a `.xlsx`** con el formato del documento.
19. **Datos demográficos del paciente** *(PR consolidado)* — sexo, fecha nac., puesto, área
    (form + carga masiva); ver sección 5.
20. **Informe Psicológico de Seguimiento** *(PR consolidado)* — sección 15.
21. **Ficha/expediente del paciente** *(PR consolidado)* — pestañas Datos · Informes ·
    Cuestionarios · Citas; ver sección 5.
22. **Recurso de permisos PACIENTES** *(PR consolidado)* — separado del staff; ver sección 4.
23. **Gestor programa citas + enganche a informe** *(PR consolidado)* — sección 8.
24. **Plan de trabajo (Gantt) + Asistencia** *(PR consolidado)* — sección 16.
25. **Jitsi self-hosted embebido** *(PR 2026-08)* — sala en modal con External API +
    botón **Unirse** en citas virtuales, **charlas** y **notificaciones**; fallback a
    nueva pestaña. Config en `config/citas.php`. Ver sección 8/10.
26. **Cronograma reactivo** *(PR 2026-08)* — al abrir **enfoca la semana actual**;
    marcar días de avance por **AJAX** sin recargar (scroll preservado).
27. **Notificaciones por polling** *(PR 2026-08)* — la campanita se refresca sola sin
    recargar la página (sin WebSockets). Ver sección 10.
28. **Seguridad de acceso** *(PR 2026-08)* — **throttle** de login (bloqueo tras 5
    intentos) + **clave temporal configurable** (`.env`) con **cambio obligatorio** al
    primer acceso (middleware `ForzarCambioClave`). **Cuentas demo ocultas en producción**.
    Ver sección 14.
29. **Rendimiento de citas** *(PR 2026-08)* — `Cita::scopeDesde()` compara por columnas
    (usa índice), reemplaza `whereRaw CONCAT`. Ver sección 14.
30. **Suite de tests + aislamiento de BD** *(PR 2026-08)* — PHPUnit sobre `psicologia_test`,
    con `force` en `phpunit.xml` y una **red de seguridad** en `TestCase` que aborta si la
    conexión no es de test. `ext-zip` declarada; seeders demo solo fuera de producción.
    Ver secciones 11 y 14.
31. **Visores tipo Facebook en el feed** *(equipo, PRs #31/#32 — en master)* — al hacer clic en
    una **imagen** o un **PDF** del feed se abre un **lightbox** (imagen/iframe a la izquierda,
    comentarios a la derecha) teletransportado al `<body>` con Alpine; relabel de comentar a
    **"Mi respuesta / Responder"**.
32. **Reordenar módulos y contenidos (drag-drop)** *(equipo, PR #33 — en master)* — arrastrar y
    soltar con **SortableJS**, persiste por AJAX (`modulos.reordenar` / `contenidos.reordenar`).
    En nuestro PR #37 se **integró sobre las tarjetas** del rediseño (se arrastra el icono).
33. **Informe con campos editables** *(equipo, PR #34 — en master)* — plantilla configurable de
    campos por defecto + edición por informe (ver secciones 5 y 15).
34. **Contenido "Apartado" interactivo** *(nuestro, PR #36 abierto)* — consigna + adjuntos
    mixtos; el paciente responde (texto + adjuntos), el gestor revisa; barra de comentarios
    oculta en apartados. Ver sección 5.
35. **Experiencia por rol + rediseño en tarjetas + reorder** *(nuestro, PR #37 abierto)* —
    el paciente *entra* al proyecto (no gestiona, no ve "Usuarios del proyecto", solo lo
    publicado/vigente, sin Estado); **menú por rol** desacoplado del acceso (`config/menu.php`,
    oculta Módulos/Contenidos del menú, gestión desde Proyectos); **rediseño "taller"** en
    tarjetas de Proyectos/Módulos/Contenidos (con barra de avance y chip "Hecho" para el
    paciente); reordenar drag-drop **sobre las tarjetas** del gestor.
36. **Cuestionarios: pendientes en notificaciones + "Volver" a la ficha** *(nuestro, PR #38
    abierto)* — los cuestionarios pendientes aparecen en las notificaciones (acción Responder);
    el botón "Volver" de la revisión regresa a la **ficha del paciente**, no al listado.

### Seeders (`DatabaseSeeder`, en orden)
`RolSeeder`, `PermissionSeeder`, `UserSeeder`, `ModeradorSeeder`,
`PsicologoDemoSeeder`, `HorarioDefectoSeeder`, `ProyectoSeeder`, `ContenidoDemoSeeder`,
`CuestionarioDass21Seeder`, `CuestionarioBaiSeeder`, `CuestionarioBdiSeeder`,
`CuestionarioNosacq50Seeder`, `FeriadoSeeder`. (No hay `CharlaSeeder`: las charlas se crean
desde la UI.) **Separación producción/demo:** `DatabaseSeeder` corre siempre los seeders base
(roles, permisos, admin, escalas, feriados) y **solo fuera de producción** los de **ejemplo**
(`PsicologoDemoSeeder`, `ContenidoDemoSeeder`, `Nosacq50RespuestasDemoSeeder`).

### Usuarios demo (contraseña: `password`)
| Email | Rol |
|---|---|
| `admin@psicologia.test` | Administrador |
| `moderador@psicologia.test` | Moderador (psicólogo) |
| `usuario@psicologia.test` | Usuario |

---

## 7. Módulo de Cuestionarios / escalas psicométricas

Escalas de puntaje continuo, **config-driven** (`config/cuestionarios.php` por `tipo`)
+ `App\Services\CalificadorCuestionario`:
- `metodo`: `suma` (subpuntaje = suma × `factor`) o `promedio` (media de ítems).
- Ítems `invertida` se puntúan al revés (`min+max - valor`).
- Nivel por dimensión con `bandas`.

**Escalas incluidas**
- **DASS-21** — escala 0-3, suma (×2), 3 dimensiones (depresión/ansiedad/estrés). Con
  **texto de interpretación por nivel** en resultados.
- **BAI** (ansiedad de Beck) — escala 0-3, suma, 1 dimensión; niveles muy baja/moderada/severa.
- **BDI-II** (depresión de Beck) — escala 0-3, suma, 1 dimensión; niveles mínima/leve/
  moderada/grave. Usa **opciones por ítem** (`preguntas_cuestionario.opciones`, JSON) —
  ítems 16/18 con variantes a/b **aplanadas a 0-3**.
- **NOSACQ-50** — escala 1-4, promedio, **7 dimensiones**, **21 ítems inversos**;
  `requiere_puesto` separa Trabajadores/Directivos.

**Resultados por persona + Excel** (`App\Services\MatrizResultados`): matriz genérica
(una fila por asignación, con respuestas Q1..Qn + puntaje/nivel por dimensión) que sirve
para DASS-21/BAI/BDI. `CuestionarioController::exportarResultados` genera un `.xlsx` con el
formato del documento (cabecera con fill corporativo, `freezePane('C2')`). El nivel de cada
resultado incluye su **interpretación** (definida en `config/cuestionarios.php`, banda `bandas`).

**Flujo:** gestor asigna en `/cuestionarios/{id}` → Usuario responde en
**"Mis cuestionarios"** (solo no-gestores) → Admin revisa respuestas y ve
**resultados por dimensión** en `/cuestionarios/{id}/resultados` con **diagramas
radar (Chart.js)** (Diagrama 1 Todos; Diagrama 2 Trabajadores vs Directivos) +
tabla de niveles. `App\Services\ResultadosCuestionario` agrega la media por dimensión.

**Añadir escala:** `Cuestionario` con ese `tipo` + entrada en
`config/cuestionarios.php` + seeder de preguntas etiquetadas por `dimension`/`invertida`.

**Campañas anónimas de cuestionario (link/QR) — NUEVO 2026-08-24:** un cuestionario clínico puede
responderse de forma **anónima por campaña** (enlace/QR, sin login), **reutilizando su motor de
calificación**. Tablas dedicadas `campanas_cuestionario` / `respuestas_campana` /
`respuesta_campana_items`; público `RespuestaCampanaController` (`/rc/{token}`), gestor
`CampanaCuestionarioController` (`campanas.*`) con QR (bacon), **% de avance por zona** y
**resultados** (radar + split Trabajadores/Directivos + matriz + Excel, **filtrados por campaña**).
**Resultados con filtros y descargas (2026-08-27):** los resultados agregados (CEAL y NOSACQ) se
pueden **filtrar por zona y por demográficos** (re-agregación **en servidor**; en NOSACQ vía scope
`RespuestaCampana::filtrado()` aplicado a `ResultadosCampana`/`MatrizCampana`) y **por dimensión**
(multi-select client-side que reconstruye el/los gráfico(s) y oculta filas de la tabla). **Guardia de
anonimato:** subgrupos con menos de `config('cuestionarios.min_subgrupo')` (=5) respuestas **no se
muestran** (radar/tabla/matriz) y se fuerza n=0 para que ni un número del subgrupo llegue al HTML/JS.
**Descargas PNG:** gráficos (canvas→PNG) y tablas (`html2canvas` con `onclone` para texto oscuro
sobre fondo blanco). En CEAL, además, tabla a color (heatmap por nivel) y MFRPS filtrable por zona.
El cálculo reutiliza `CalificadorCuestionario` vía una **asignación transitoria** (no persistida:
`setRelation('respuestas', …)`) — servicios `ResultadosCampana` / `MatrizCampana`. **NOSACQ-50 es
"solo anónimo"** (flag `solo_anonimo` en `config/cuestionarios.php`): fuera de la tabla clínica y de
"Mis cuestionarios"; se lista en la sección **"Cuestionarios anónimos"** junto al CEAL-SM. **Deploy:**
`php artisan migrate` (3 tablas nuevas).

---

## 8. Módulo de Citas psicológicas + horarios

**Agendar** (`App\Livewire\AgendarCita`, wizard): modalidad (presencial/virtual) →
psicólogo → **fecha** (flatpickr, bloquea domingos y feriados) → **cupo de 30 min** →
datos del paciente → **confirmación**. Genera **enlace Jitsi** si es virtual y envía
**correo de confirmación**. `DB::transaction` con `lockForUpdate` → **sin doble reserva**.

**Disponibilidad** (`App\Services\AgendaCitas`): calcula bloques según el **Horario**
del psicólogo por día/modalidad (fallback a `config/citas.php` si no definió horario).
`config/citas.php`: `sesion_minutos=30`, `cancelacion_horas_min=24`, modalidades con
horario por defecto (presencial 12:00-15:00, virtual 13:00-15:00).

**Modo gestor (nuevo):** el mismo `AgendarCita` admite `mount(gestor: true)` para que un
gestor **programe una cita a un paciente existente** (botón "Nueva cita" en Gestión de citas,
permiso `citas.crear`): paso 0 elige paciente; el **Moderador se auto-asigna** como psicólogo
(salta ese paso), el **Admin elige** cualquiera; guarda `user_id` = paciente. Candado: el
Moderador solo puede agendarse a sí mismo. Admite `?paciente=ID` para precargar (lo usa el
botón **"Programar próxima sesión"** del informe; ver sección 15).

**Paneles**
- Usuario: **Mis citas** (ver/cancelar, respeta política de cancelación).
- Psicólogo: **Mi horario** (`App\Livewire\GestionHorario`, define su disponibilidad;
  se precargan 12 bloques por defecto editables).
- Admin/Moderador: **Gestión de citas** (estado + **columna Informe** + **Nueva cita**),
  CRUD de **Psicólogos** (con botón Horarios) y **Feriados** (tabla).

---

## 9. Módulo de Charlas

> **NUEVO (2026-08-24): las Charlas viven DENTRO de Contenidos** (son un tipo de contenido).
> Se crean desde el formulario de contenido eligiendo tipo **"Charla"** (no hay creación suelta; se
> retiraron `charlas.create/store`, su vista y el ítem "Charlas" del menú lateral). Al crear quedan
> **ligadas al proyecto y módulo** del módulo elegido (`charlas.proyecto_id` / `charlas.modulo_id`) y
> se **auto-inscriben los pacientes** (rol Usuario) de ese proyecto como asistentes; botón
> **"Sincronizar con el proyecto"** (en `GestionCharla`) para sumar a quien se agregue después.
> Aparecen en la **página del módulo** (junto a los contenidos, sin entrar al reordenado) y en la
> sección **"Charlas"** del índice de Contenidos. `App\Services\CharlaService` centraliza
> creación/imagen/Jitsi/inscripción (lo usan `ContenidoController@store` y `CharlaController@update`).
> No hubo fusión de datos: el motor de charlas (asistencia/feed/recordatorios/Jitsi/Plan) queda intacto.

- CRUD de charlas + **asistencia** (`charla_user.asistio`) vía Livewire
  `App\Livewire\GestionCharla` en `/charlas/{id}` (`charlas.show`).
- Flujo: agregar asistentes → **finalizar** → subir **foto de evidencia**. Al **crear/editar**
  la charla se puede adjuntar una **imagen de referencia** (`imagen`, distinta de `foto`;
  `CharlaController` con `enctype=multipart`, guardada como `/storage/...`).
- **En el feed**: charlas **finalizadas con foto** (registro con asistencia) y
  charlas **programadas a futuro** (anuncio). Cada tarjeta tiene ancla `#charla-ID`
  (los recordatorios enlazan a ella con realce `:target`). Gestores ven **todas** las
  charlas con pie de gestión (N inscritos + Gestionar).

> ⚠️ **Convención de archivos subidos**: se guardan como **ruta relativa** `/storage/...`
> (no absoluta con host), para que resuelvan sin importar el dominio/puerto. Aplica a la
> **foto/imagen de charla**, **adjuntos de comentario** y — desde el PR consolidado —
> **también a los contenidos** (video/imagen/documento/audio en `ContenidoController`, antes
> guardaban la URL absoluta con `APP_URL` → se rompían al navegar en otro dominio). El
> **borrado** usa el mismo prefijo `/storage/` (reconoce también la forma absoluta antigua).

---

## 10. Recordatorios / notificaciones por rol

`App\Services\Recordatorios::para(User)` construye una lista homogénea de
recordatorios según el rol, renderizada por `resources/views/partials/recordatorio.blade.php`
en **dos lugares**: la **franja del panel** (dashboard) y la **campanita** de la
topbar (`<x-recordatorios-bell>`, visible en Feed / Contenidos / Mis cuestionarios / Mis citas).

- **Usuario (paciente):** **todas** sus citas próximas + su próxima charla.
- **Psicólogo (Moderador):** próxima cita que atenderá (con paciente) + agenda de hoy.
- **Administrador:** citas por confirmar + próxima charla del sistema.

Cada ítem: icono, color (naranja=cita, índigo=charla), título, líneas y acciones
(p. ej. *Unirse* si la cita virtual está por comenzar, *Ver mis citas*, *Ver en el feed*).

> **Unirse embebido (Jitsi):** el botón *Unirse* aparece para **citas, charlas y reuniones**
> virtuales cercanas a su hora, tanto en la campanita como en la franja del dashboard. Abre la
> sala **dentro de un modal** (iframe con `JitsiMeetExternalAPI`), con *fallback* a nueva pestaña
> si el servidor rechaza el iframe. Las charlas virtuales guardan su `enlace_reunion` (sala
> `SafePoint-Charla-…`). Ver stack (sección 2) y `config/citas.php`.

---

## 11. Estado del repositorio y flujo de trabajo

### Regla de git (importante)
- **NO se trabaja sobre `master` directamente.** Cada funcionalidad va en su rama
  `feat/...` y se **consulta al dueño antes de hacer `git push`** y antes de mergear.
- **Integración continua local:** tras cada cambio, la rama se **fusiona en
  `integracion/local`** (creada desde `master`) para ver todo junto y resolver conflictos
  poco a poco. `integracion/local` **NO se sube** al remoto; es solo de revisión.
- `gh` CLI **ya está instalado** (`C:\Program Files\GitHub CLI\gh.exe`, autenticado como
  `atestertesting`, scopes `repo`/`workflow`; resuelve a `cristhian199228/proyecto_psicologia`).
  Usar `gh pr create/list/view/merge`. *(Ojo: una terminal abierta antes de instalarlo tiene el
  PATH viejo; invocar por ruta completa o abrir una nueva.)*
- ⚠️ **La rama `master` local suele quedar desactualizada**: el master real es `origin/master`
  y el **equipo mergea PRs a master en paralelo**. Basar ramas/PRs y calcular diffs **siempre
  contra `origin/master`** (`git fetch origin master` primero). Traer periódicamente el trabajo
  del equipo con `git merge origin/master` en `integracion/local`.
- Tras sincronizar / un merge de master: `php artisan migrate`, `php artisan route:cache`
  (el proyecto **cachea rutas** — sin esto, rutas nuevas dan "Route not defined"),
  `php artisan storage:link`, `php artisan optimize:clear`. (`composer install`/`npm` solo si
  cambiaron dependencias.)

### Estado actual
- **Mergeado a `master`** (en orden): PRs #16/#18/#20 (pacientes, dashboard progreso, asignación
  masiva, alcance de citas, tema oscuro) → **PR consolidado `release/consolidado-2026-07`** (#21:
  escalas BAI/BDI-II, interpretación/matriz/export DASS-21, datos demográficos, Informe de
  Seguimiento (PDF), ficha/expediente, permisos PACIENTES, gestor programa citas, imagen de
  charla) → **PR `release/plan-gantt-2026-08`** (Plan/Gantt completo — seguimiento 3c,
  participación, alcance general + filtro, dashboards) → **PR #29 `release/mejoras-y-deuda-2026-08`**
  (merge `80f7ce0`): Jitsi propio embebido + *Unirse*, cronograma reactivo (enfoque hoy, scroll,
  AJAX), notificaciones por polling, throttle+clave temporal con cambio obligatorio, `scopeDesde()`
  con índice + `APP_URL`, `ext-zip` + seeders demo separados, panel demo solo fuera de producción,
  y **suite de tests con aislamiento de BD**.
- **Aportes del equipo ya en `master`** (tras el PR #29): PRs **#31–#35** — visor de imagen y de
  **PDF** tipo Facebook en el feed, **reordenar** módulos/contenidos (drag-drop), **informe con
  campos editables** (plantilla), y el relabel **"Mi respuesta / Responder"**. `origin/master`
  quedó en `90eb7ad`.
- **PRs abiertos nuestros (13-ago-2026), mergear en orden `#36 → #37 → #38`** — reconstruidos
  sobre el `origin/master` actual, los tres `mergeable = clean` y **sin conflictos** (probado con
  merge secuencial `--no-ff`). Es una **pila lineal**: `master → #36 (apartado) → #37
  (experiencia+reorder) → #38 (cuestionarios)`; cada PR muestra solo su diff y al mergear #36
  GitHub reapunta #37 a master (y luego #38). Cadena de ramas: `release/apartado-2026-08` →
  `release/experiencia-paciente-2026-08` → `release/cuestionarios-2026-08`.
- **`integracion/local`** (solo local, **NO se sube**): ya tiene `git merge origin/master` con los
  aportes del equipo **+** nuestro trabajo, todo integrado y probado (**45 tests en verde**). El
  **conflicto de fondo** (nuestro rediseño en tarjetas **vs** el drag-drop del equipo en la
  **tabla**) se resolvió **conservando ambos**: se mantienen las tarjetas y el reordenar se
  **recableó sobre ellas** (se arrastra el icono). El tope de la pila de PRs es **byte-idéntico**
  a `integracion/local`. Los PRs se generan ramificando de `integracion/local` a `release/*`.
  Conviene **sincronizar `integracion/local` con `origin/master`** antes de empezar cambios nuevos.
- **Tandas recientes en `master` (ago-2026):** **#40** (CEAL-SM + NOSACQ-50 anónimos + Charlas en
  Contenidos) → **#41** (contenidos con período en el Plan/Gantt) → **#42** (mejoras UX: perfil con
  foto, feed por recencia, wizard anónimo, navegación) → **#43** (perfil tipo Facebook, ranking con
  foto, dark difuminado, login dark, flash auto-descartar, CEAL descargas/filtros/tabla color/MFRPS)
  → **#44** (Livewire "por página" en módulos) → **#45** (cronograma PNG) → **#46** (filtro por
  dimensión en CEAL + filtros/descargas PNG en NOSACQ) → **#47** (2 fixes de seguridad + Feed
  paginado en BD + eliminar proyecto/módulo/cuestionario + citas cancelar/reprogramar; ver §17)
  → **#48** (fixes: descripción de proyecto OPCIONAL —columna `proyectos.descripcion` a nullable,
  reventaba al crear sin descripción—; y el "Volver" del informe respeta el origen: desde la ficha
  del paciente regresa a la ficha, no a Gestión de citas, con botón "Volver" contextual)
  → **#49/#50** (comentarios de contenido en el Plan, en el modal) → **#51** (imagen del cronograma
  con "Actividad" a la izquierda) → **#52** (UX móvil: navegación inferior + PWA + Panel rediseñado
  + inicio por rol; ver §17) → **#53** (forzar HTTPS en URLs cuando `APP_URL` es https; ver §17)
  → **#54** (editar nombre/descripción de módulos, botón ✏️ en listado, ficha de proyecto y módulo)
  → **#55** (fix: paciente sin proyecto veía 404 en Ranking → página amable; bottom-nav coherente)
  → **#56–#59** (aportes del equipo: texto justificado en tarjetas, abrir imagen del apartado en el
  visor, "Volver" del apartado a la página anterior, y **modo libre por contenido** —columna
  `contenidos.es_libre`: comentarios públicos sin calificación ni ranking) → **#60** (filtros por
  demográficos NUMÉRICOS —edad, N.º de hijos— por rango en Resultados de CEAL y NOSACQ; ver §17)
  → **#61** (aporte del equipo: selector de tipo de gráfico en Resultados de CEAL, 7 vistas sin
  recargar). `origin/master` en `5b7e7db`; `integracion/local` **sincronizado 0/0**.
  Suite **181 tests Feature en verde**. *Pendiente en cola: fase 2 del móvil (tablas anchas → tarjetas,
  Gantt/gráficos); correos de citas síncronos (encolar cuando el servidor tenga worker);
  `DashboardProgresoController` aún materializa en PHP. **HTTPS ya en producción** (`safepoint.internationalsos-peru.com`).*
- Al desplegar: `composer install` (**nuevo `ext-zip`**), `php artisan migrate` (aditivas),
  `php artisan optimize` (NO `optimize:clear` en prod: deja la app sin cachés → lenta).

### Tests y seguridad de la BD (importante)
- **`php artisan test`** corre PHPUnit contra **`psicologia_test`** (MySQL, no la real). Requiere
  crearla una vez: `CREATE DATABASE psicologia_test`. Se usa MySQL y no SQLite porque el proyecto
  tiene migraciones con SQL específico de MySQL (`ALTER … MODIFY … ENUM`).
- **Doble candado anti-desastre** (tras un incidente en que un `migrate:fresh` dejó la BD real
  vacía): `phpunit.xml` fija `DB_DATABASE=psicologia_test` con `force="true"` (sin `force`, como
  `php artisan test` ya cargó `.env`, el override se ignoraba y `RefreshDatabase` podía tocar
  `psicologia`); y `Tests\TestCase::refreshApplication()` **aborta la suite** si la conexión no
  apunta a una BD de test, **antes** de cualquier `migrate:fresh`.
- **Recomendación operativa:** respaldar antes de trabajos pesados —
  `mysqldump -uroot psicologia > backup.sql`. No hay backups automáticos.

### Repo de documentación
- Este archivo `docs/CONTEXTO-PROYECTO.md` **no** se versiona en el repo del proyecto;
  vive en el git personal **`github.com/atestertesting/docs.git`**.
- 🔒 **REGLA (obligatoria): `CONTEXTO-PROYECTO.md` NUNCA se elimina ni se oculta.** Por más que
  se actualice/fusione con `master`, este archivo debe permanecer. Como `/docs` está en
  `.gitignore`, ningún `git merge origin/master`, cambio de rama ni `checkout` del **repo
  principal** lo toca (es un repo anidado, aparte). Si la carpeta local `docs/` llegara a faltar,
  **re-clonarla** (`gh repo clone atestertesting/docs docs`) — no recrear el archivo desde cero,
  para no perder el historial. Tras cualquier mejora/análisis, **actualizar este documento y
  hacer `git push`** al repo de docs.
- 🧭 **REGLA: las ideas, análisis y pendientes se registran en la sección 17
  "Pendiente / futuro (backlog)".** Es el lugar único para lo que queda por hacer o proponer.

---

## 12. Cómo levantar el proyecto

```bash
# Requisito PHP: habilitar ext-zip (para PhpSpreadsheet / carga masiva Excel).
#   En Laragon: descomentar `extension=zip` en php.ini y REINICIAR Laragon.
composer install                    # incluye Livewire, getid3, phpspreadsheet y dompdf
cp .env.example .env                # DB_DATABASE=psicologia, MySQL 127.0.0.1:3306, root
php artisan key:generate
php artisan migrate:fresh --seed
php artisan storage:link            # multimedia/fotos/adjuntos al storage
php artisan serve                   # http://127.0.0.1:8000
```

- No requiere `npm` para la UI (Bootstrap, Chart.js, flatpickr por CDN).
- **`ext-zip` obligatoria** para la carga masiva de pacientes en `.xlsx` (un `.xlsx`
  es un ZIP). Sin ella, la subida/descarga de plantilla falla en runtime. En Laragon,
  reiniciar el servicio tras habilitarla (el proceso web no relee php.ini en caliente).
- Tras cambiar de rama o mergear: `php artisan optimize:clear && php artisan migrate`.

---

## 13. Convenciones para seguir desarrollando

- Respetar la **paleta corporativa** y reusar las clases de `corporate.css`.
- Vistas Blade con Bootstrap (CDN), iconos `bi-*`, textos en español en `__()`
  (añadir la clave a `lang/en.json`). **No usar `trans_choice`** (fallback a EN):
  usar `__()` con ternario singular/plural.
- Proteger rutas con `permission:{recurso}.{accion}`; Administrador omite checks
  (`Gate::before`). Para "gestión" de cuestionarios se usa `cuestionarios.crear`.
- Archivos subidos por usuarios (foto de charla, adjuntos de comentario): guardar
  **URL relativa** `/storage/...` y borrar con el mismo prefijo.
- Validación en controladores; operaciones multi-tabla y agendamiento en **transacción**
  (`lockForUpdate` para evitar doble reserva).
- Formatear con **Pint** antes de cerrar (`vendor/bin/pint <archivos>`).
- Verificar en servidor real (login + flujo por rol) antes de dar por terminado.
- Posibles siguientes pasos: Cronbach's Alpha y filtros por sector del NOSACQ,
  editar/eliminar cuestionarios desde la UI, exportar resultados (PDF/CSV),
  notificaciones por correo/push más completas, recordatorios reactivos (Livewire).

---

## 14. Deuda técnica conocida (de la auditoría, julio 2026)

Análisis multi-área sobre `integracion/local`. **Fortalezas confirmadas:** autorización
a nivel de objeto sólida, transacciones + `lockForUpdate`, sin mass assignment, sin
inyección SQL ni XSS, seeders idempotentes, login sin enumeración de usuarios.

**Ya resuelto en el PR consolidado:** URLs de archivos de **contenido** ahora relativas
`/storage/...` (antes absolutas con `APP_URL` → imágenes rotas por dominio); `confirm()`
nativo reemplazado por el **diálogo estilizado** (`data-confirm`) en el informe;
**autoría del informe** restringida al **psicólogo tratante** (el Admin no redacta informes
ajenos, solo lee/PDF) — mitiga parcialmente el punto de *alcance del Moderador*.

**Ya resuelto en el PR `release/mejoras-y-deuda-2026-08`** (ver secciones 6 y 11):
- **Seguridad:** login con **throttle** (5 intentos → bloqueo, `RateLimiter`); **clave temporal
  configurable** (`config/auth.php` ← `PASSWORD_TEMPORAL_PACIENTE`) con **cambio obligatorio** al
  primer acceso (`debe_cambiar_password` + middleware `ForzarCambioClave` + `ClaveController`);
  **cuentas demo del login ocultas en producción** (`@unless production`).
- **Rendimiento:** `whereRaw CONCAT(fecha,' ',hora)` reemplazado por `Cita::scopeDesde()`
  (comparación por columnas, usa índice) en los ~5 sitios; `APP_URL` corregido a la URL real.
- **Calidad/infra:** **suite de tests** (login/throttle, RBAC, scope de citas, panel demo) con
  **aislamiento de BD blindado**; `ext-zip` declarada en `composer.json`; **seeders demo
  separados** de los de producción.

**Sigue pendiente:** el **login sigue en claro** (sin tema oscuro); *alcance clínico del
Moderador* (aún ve la ficha de **todos** los pacientes — decisión de producto); refactor
`PacienteController`≈`UserController`; claves i18n huérfanas/duplicadas; token de sala Jitsi
8→16 chars; `FeedController`/`DashboardProgresoController` materializan en PHP.

**Prioridad alta**
- ✅ **Seguridad (RESUELTO, PR 2026-08):** clave temporal ahora **configurable** con cambio
  obligatorio; login con **throttle**. *(Antes: `'SafePoint123'` hardcodeada + login sin throttle,
  la cadena más explotable.)*
- **Rendimiento:** ✅ el `whereRaw CONCAT` (5 sitios) se reemplazó por `Cita::scopeDesde()`
  (usa índice). **Sigue pendiente:** `FeedController` carga **todo** el feed en memoria y pagina
  en PHP; `DashboardProgresoController` materializa todo en PHP.
- **Frontend:** el **login queda en claro** (layout guest sin `data-bs-theme` + `bg-white`);
  `dashboard/progreso.blade.php` usa hex pastel fijos que rompen el modo oscuro.
- **Calidad:** ✅ **RESUELTO (PR 2026-08):** hay **suite de tests** (antes solo `ExampleTest`),
  `ext-zip` declarada y **seeders demo separados** de producción. **Sigue pendiente:** README
  genérico de Laravel; sin CI/PHPStan.

**Prioridad media**
- **Redundancia:** `PacienteController` ≈ `UserController` (CRUD casi idéntico);
  claves i18n **huérfanas** (flujo CSV viejo) y **duplicadas** en `lang/en.json`; regla
  "contenido realizado = comentado" duplicada en los dos dashboards; literal `'Usuario'`
  repetido en 3 archivos; borrado de foto de charla duplicado (mover al modelo).
- **Alcance del Moderador:** hoy un psicólogo ve datos clínicos (respuestas, progreso) de
  **todos** los pacientes, sin importar proyecto/asignación — revisar need-to-know.
- **Frontend:** cabecera de página duplicada en ~40 vistas (→ `<x-page-header>`); avatar
  y tarjeta KPI como componentes; `confirm()` nativo vs. diálogo estilizado; email de
  cita sin `__()`.
- **Tooling:** sin FormRequests (validación inline), sin CI, sin PHPStan/Larastan.

**Prioridad baja:** `welcome.blade.php` muerto; token `--corp-orange-50` inexistente;
overlay blanco en `contenidos-table`; botones de borrar sin `aria-label`; docstring
desactualizado en `ImportadorPacientes`.

---

## 15. Informe Psicológico de Seguimiento

Documento clínico individual **por cita completada** (`informes_seguimiento`, 1:1 con la
cita, `cita_id` unique). Nace del ciclo: *indicador detectado en un test → cita → informe →
recomendación → próxima cita*.

- **Estructura**: I. Datos de identificación + secciones **II–VIII** (Motivo, Antecedentes,
  Problema actual, Observaciones clínicas, Impresión psicológica, Conclusiones,
  Recomendaciones). Se guarda **snapshot** de identificación y firma (nombre/colegiatura del
  psicólogo) para que un informe finalizado no cambie si luego se edita el perfil.
- **Ciclo**: `estado_informe` `borrador → finalizado` (finalizado = solo lectura + PDF;
  "Reabrir" vuelve a borrador). PDF con **dompdf** (`informes.pdf`, formato del documento).
- **Autoría vs lectura** (`InformeSeguimientoController`): **crear/editar/finalizar/reabrir =
  solo el moderador dueño de la cita**; **ver/PDF = dueño o Administrador**. El Admin no
  redacta informes ajenos.
- **Motivo precargado**: al crear, si no hay motivo, se toma el **último indicador elevado**
  del paciente (`App\Services\IndicadorTest`, recorre DASS-21/BAI/BDI y devuelve p. ej.
  "BDI-II: Depresión leve") — editable.
- **Enganche a cita**: botón **"Programar próxima sesión"** abre el asistente en modo gestor
  con el paciente precargado (`?paciente=ID`, tipo seguimiento).
- **Accesos**: columna **Informe** en Gestión de citas (crear/continuar/ver según estado) y
  en la pestaña Informes de la ficha del paciente. `ocupacion` se precarga del `puesto`.
- **Campos editables (aporte del equipo, PR #34 — en master)**: una **plantilla global**
  (`informes.plantilla`, Livewire `GestionCamposInforme`) define los campos por defecto de cada
  informe nuevo (tipo *texto libre / opción múltiple / escala*, con opciones o min/max); al
  redactar (`RedactarInforme`) se pueden ajustar por informe. Modelos `CampoInforme` (plantilla)
  y `SeccionInforme` (campos concretos del informe). Enlace en el menú **Agendamiento**.

---

## 16. Plan de trabajo — Cronograma tipo Gantt (COMPLETO)

Fase 3 del "Plan de Seguridad Integral" (del documento/Excel del cliente). Menú **"Plan de
trabajo"**, permiso `planes.*` (Admin y Moderador). **Todo el Excel está clonado y en `master`.**

### Alcance del plan (general vs proyecto)
- Un plan puede ser **General (todos los proyectos)** o **de un proyecto**: `planes.proyecto_id`
  es **nullable** (null = general). Selector **"Alcance"** en el form (General por defecto).
  `Plan::esGeneral()`. La idea del cliente: un **único plan general vigente por año/ciclo**;
  los anteriores quedan como **histórico**. La lista infiere **Vigente/Histórico** por fechas
  (`Plan::estaVigente()`), sin columnas extra.
- **Filtro por proyecto** (`?proyecto=ID`) en cronograma/asistencia/seguimiento/participación
  (como el dashboard de progreso): acota las **personas** al proyecto elegido; las actividades
  del cronograma no cambian. Partial `planes/_filtro-proyecto`.

### 3a — Cronograma/Gantt (`planes` + `plan_actividades`)
- Actividades por **bloque** (Propuesta · Data y análisis · Programa · Gestión y cierre), con
  `responsable`, `plazo_texto`, `fecha_inicio/fin`, `estado`, `avance %`, `modalidad`.
- **Grilla semanal** (lun–sáb, labels tipo `2.7`): color **derivado** de fechas + estado.
  Columna sticky + scroll. `Plan::semanas()`, `PlanActividad::estadoEnSemana()`.
- Herramientas del cronograma: **Avance de talleres** (gráfica), **Comentarios/bitácora** por
  franja (aporte del equipo), semanas en **rojo** + nota, **avance de talleres por semana**
  (días 0–5), **modalidad** presencial/virtual. El menú separa **herramientas** de las **vistas**
  (Asistencia · Participación · Seguimiento en un grupo).

### 3b — Asistencia (`asistencia`, `planes.asistencia`)
- Matriz **pacientes × talleres** (actividades con `charla_id`), asistencia leída de `charla_user`,
  con % por persona. Se registra en cada Charla; el plan **consolida**.

### 3c — Seguimiento preventivo (`seguimiento`, `plan_seguimientos`)
- Matriz **pacientes × semanas**: nº de **sesiones** (citas completadas) por semana. Se muestra
  **dentro del cronograma** (bloque más) y en su vista propia. Nombre completo.
- **Padrón confirmado** (tabla `plan_seguimientos`) con **sugerencias automáticas** según el
  último indicador de test elevado (`IndicadorTest`) + alta/baja manual.
- **% con sentido** (reemplaza el % por semanas del Excel, que era engañoso): con **meta** →
  progreso hacia el alta (realizadas/meta) + estado **"Alta"**; sin meta → **adherencia**
  (asistidas/agendadas); sin citas → "—". `plan_seguimientos.meta_sesiones` editable por paciente.
- Gráfico **"% de seguimiento por persona"** (Chart.js, dentro de la vista).

### Participación integral (`participacion`, `planes.participacion`)
- Matriz **paciente × cada ítem** del programa (contenidos por tipo Video/Audio/Documento/Imagen/
  Reunión + Charlas). **Participó = comentó el contenido o asistió a la charla** (misma regla
  "contenido realizado = comentado" del dashboard). **% por grupo** + **% Promedio**.
- **Dashboard** con 2 gráficos (Chart.js): **"% por ítem"** y **"% por semana"**.

### 3d — Bitácora/Comentarios y Reconocimiento
- **Aporte del equipo (ya en master):** comentarios/bitácora por franja del cronograma
  (`plan_actividad_comentarios`, `plan_semanas`), y **ranking trimestral** de participación.

> Nota técnica: los **gráficos Chart.js** van en un contenedor de **altura fija** (con
> `maintainAspectRatio:false`, si el canvas no tiene contenedor con altura, crece sin parar).
> Chart.js se carga por **CDN** (necesita internet para verse).

**Pendiente / futuro (de este módulo):** **Hub de Reportes** (lista global de informes) como cierre
de la fase de reportes; estados de asistencia más ricos (regular/reforzamiento/motivo de ausencia)
si el cliente los pide.

---

## 17. Pendiente / futuro (backlog de ideas y análisis)

> **Este es el lugar único para registrar lo que queda por hacer, ideas y análisis.** Cada vez que
> se proponga una mejora o se haga un análisis, **anotarlo aquí** (y actualizar/subir este doc).
> No borrar los ítems: al implementarse, marcarlos ✅ y mover un resumen a la sección que
> corresponda (5/6/…).

### Contenidos reutilizables + edición sin actualización "en vivo" ⭐ (pedido del cliente)
- **Reutilizables:** un mismo **Contenido** debe poder **asignarse a varios Módulos** (hoy un
  contenido pertenece a **un** módulo: `contenidos.modulo_id`). La idea es una relación
  **muchos-a-muchos** (p. ej. pivote `contenido_modulo`) o un mecanismo de "copiar/enlazar"
  contenido existente a otro módulo, para no recrearlo cada vez.
- **Sin actualización en vivo al editar:** cuando un gestor **edita** un Contenido, el cambio
  **NO** debe propagarse **en vivo** a las asignaciones ya hechas (para no alterar lo que un
  paciente ya vio/respondió, ni el contenido ya asignado a un Módulo). Es decir, la edición debe
  comportarse como una **versión/instantánea** por asignación (o pedir confirmación explícita de
  "actualizar en todos los módulos"), en lugar de mutar el registro compartido en caliente.
- **Notas de implementación:** cuidado con el tipo **Apartado** (las `ApartadoRespuesta` cuelgan
  de `contenido_id`) y con "contenido realizado = comentado" (dashboards/participación); si un
  contenido se comparte entre módulos hay que decidir si el avance/respuestas son por
  (contenido) o por (contenido × módulo). *(Antecedente: hubo un intento de "reutilizar
  contenido" que se **revirtió**; retomar con este diseño de no-live-update.)*

### CEAL-SM anónimo con link/QR (IMPLEMENTADO — rama `feat/ceal-anonimo`) ⭐
> **Estado (2026-08-24): las 5 fases están hechas y mergeadas a `integracion/local`.** Entra en el
> **PR consolidado** de esta tanda (CEAL anónimo + NOSACQ anónimo + Charlas en Contenidos).
> **UI:** CEAL-SM vive **dentro de "Cuestionarios"** (sección "Cuestionarios anónimos", no en el
> menú); breadcrumb `Panel / Cuestionarios / CEAL-SM (anónimo)`. **Deploy:** requiere
> `composer install` (nueva dependencia `bacon/bacon-qr-code`) y `php artisan migrate` (3 tablas
> nuevas). Formulario público sin login en `/r/{token}`.
> Pendiente/futuro del propio CEAL: afinar etiqueta GHQ "Mejor vs Más" por ítem; el ítem "AL"
> (definición de bullying) entró como pregunta y debería ser solo texto; desglose de resultados
> por zona/demográfico; export del monitoreo a Excel.

Pedido del cliente (TGP): aplicar el **CEAL-SM / SUSESO** (riesgo psicosocial laboral) de forma
**anónima**, generando **link + QR** (estilo Microsoft Forms) para que cualquiera responda sin login.
Documentos fuente en `storage/docs/`: `CUESTIONARIO.pdf`, `CEAL-SM SUSESO Autocalificacion (2).xlsx`
(hojas Cuestionario/Puntajes/Cortes = ítems, tipos de puntaje y baremos), `BAREMOS…pdf`, y
`SEGUIMIENTO TGP MONITOREO 2025.xlsx` (monitoreo de avance por zona).

**Decisiones tomadas:** (1) por ahora **solo CEAL-SM** (no cuestionarios genéricos); (2) anti-duplicado
por **cookie** (un envío por navegador, sin identificar) + rate-limit; (3) **QR server-side en SVG**
(`bacon/bacon-qr-code`, sin `ext-gd`, imprimible/offline); (4) el "tamaño" es la **población objetivo
por zona** (Costa 19, Sierra 38, Selva 40, San Isidro 179 → 276) para calcular **% de avance**;
(5) campañas con **apertura y cierre** (+ cierre manual).

**Instrumento:** 12 dimensiones psicosociales (CT, EM, DP, RC, CR, QL, CM, IT, TV, CJ, VU, VA) +
**Salud mental GHQ-12**; ~76 ítems; **6 tipos de puntaje** (A directo 4→0, B protector 0→4, C
vulnerabilidad 1→4, D violencia 0→4, P/N para GHQ). Cortes por dimensión (Bajo/Medio/Alto) en la
hoja "Cortes". *(**OTP = San Isidro** — confirmado por el cliente: es una sola zona; las zonas son
Costa/Sierra/Selva/San Isidro con 19/38/40/179 = 276. La satisfacción **TEA14** del baremo PDF no
está en la hoja de autocalificación — se dejó fuera por decisión del cliente.)*

**Plan por fases:**
1. **Datos:** `config/ceal.php` (dimensiones+cortes, tipos de puntaje, ítems) generado desde el xlsx.
2. **Modelos/migraciones (tablas dedicadas, sin `user_id`):** `campanas_ceal` (token, activo,
   abre_en/cierra_en, poblaciones por zona, creado_por), `respuestas_ceal` (zona + demográficos,
   sin identidad), `respuesta_ceal_items` (código, valor, puntos).
3. **Formulario público** sin auth (`/r/{token}` o `/r/{token}/{zona}`), layout limpio mobile-first,
   cookie anti-duplicado + rate-limit + CSRF + honeypot.
4. **QR + compartir** (SVG por zona, imprimible).
5. **Dashboards del gestor:** (a) **monitoreo** realizadas/total/% por zona (= el Excel); (b)
   **resultados** por dimensión con baremos, **solo agregados** (nunca individual).

Rama: `feat/ceal-anonimo`.

### NOSACQ-50 anónimo por campaña (IMPLEMENTADO — 2026-08-24) ⭐
Cliente TGP: NOSACQ-50 (clima de seguridad) también anónimo por link/QR. Se **reutilizó el motor
clínico** (config `nosacq50`, ítems en BD, `CalificadorCuestionario` con promedio + ítems inversos,
radar + split Trabajadores/Directivos) en vez de portarlo al CEAL. Se añadió la **capa anónima**
(campañas + responder público `/rc/{token}` + resultados por campaña) en **tablas separadas**
(`campanas_cuestionario`/`respuestas_campana`/`respuesta_campana_items`) — ver sección 7. NOSACQ-50
quedó **"solo anónimo"** (flag `solo_anonimo`, fuera del flujo por login), sin tocar BAI/BDI/DASS.
*Pendiente/futuro:* re-sincronizar demográficos por campaña; desglose de resultados por zona.

### Charlas dentro de Contenidos (IMPLEMENTADO — 2026-08-24) ⭐
Las charlas pasan a ser un **tipo de contenido** (ver sección 9): se crean desde el formulario de
contenido, quedan ligadas a proyecto+módulo, **auto-inscriben pacientes** y se **sincronizan**. Se
mantuvo el **motor de charlas** intacto (asistencia/feed/recordatorios/Jitsi/Plan) — **no hubo
fusión de datos**. *Pendiente/futuro:* al **eliminar** una charla desde el módulo hoy redirige al
hub de Contenidos (no al módulo); las charlas antiguas sin proyecto quedan solo en la sección
Charlas (no en un módulo); opción de **mezclar** charlas en la misma tabla de contenidos (hoy van en
bloque aparte, fuera del reordenado).

### Mi perfil + avatares (IMPLEMENTADO — 2026-08-25) ⭐
Página **"Mi perfil"** (autoservicio, cualquier usuario): **foto de perfil** (`users.avatar`,
`/storage/…`, borra la anterior al reemplazar; fallback a la inicial), **nombre** e **idioma**
editables, y **cambiar contraseña** integrado (verifica la actual). Correo/rol/puesto/área de **solo
lectura** (los gestiona el gestor; alimentan reportes). Componente reutilizable **`<x-avatar>`** (foto
o inicial) usado en topbar, comentarios (feed y plan), asistentes de charla, tablas de
usuarios/pacientes, progreso y saludo del dashboard. `PerfilController` + rutas `perfil.*` bajo auth.
**Deploy:** `php artisan migrate` (users.avatar) + `storage:link`. *(Los psicólogos usan su propio
campo `Moderador.foto`, aún sin unificar con `users.avatar`.)*

### Cuestionarios anónimos: asistente por pasos + autosave (IMPLEMENTADO — 2026-08-25) ⭐
Los formularios públicos largos (CEAL `/r/{token}` y NOSACQ `/rc/{token}`) pasan a un **asistente por
pasos** (uno por dimensión, barra de progreso, validación por paso; sigue siendo **un solo `<form>`**)
y **autosave en `localStorage`** por token: si se cae el internet / cierra / refresca, **conserva
respuestas y el paso** y ofrece "Empezar de nuevo"; el borrador se limpia en la página de gracias
(registro confirmado). Todo front (partial `partials/wizard-cuestionario.blade.php`), sin backend.

### Feed por recencia (IMPLEMENTADO — 2026-08-25)
El feed pasó de "charlas fijas al inicio + contenidos en orden curricular" a un **hilo único ordenado
por fecha de CREACIÓN** (lo último creado arriba, charlas y contenidos entremezclados).

### Gestión de contenidos desde el módulo + navegación (IMPLEMENTADO — 2026-08-25)
- Desde la página del **módulo**, el gestor **edita y publica/despublica** un contenido (incluidos
  borradores) sin salir; el **título de un borrador** abre su edición.
- **"Volver"** de un contenido (revisar) ya no queda en bucle tras guardar la respuesta del apartado
  (va al módulo / Feed en vez de a sí mismo).
- **Crear contenido**: "Volver"/"Cancelar" regresan al **módulo de origen** si vienes de él.

### Resultados anónimos: filtros + descargas PNG (IMPLEMENTADO — 2026-08-27) ⭐
Sobre los resultados agregados de **CEAL** y **NOSACQ-50** (ver detalle técnico en §7):
- **Filtros por zona y por demográficos** (re-agregan en servidor) con **guardia de anonimato**
  (`cuestionarios.min_subgrupo`=5 / `ceal.min_subgrupo`=5): subgrupos chicos no muestran nada.
- **Filtro por dimensión** (single y múltiple, client-side) que reconstruye gráficos y oculta filas.
- **Descargas PNG** de gráficos (radar/barras vía canvas) y tablas (`html2canvas`). CEAL: tabla a
  color + MFRPS filtrable por zona.
- PRs: **#43** (CEAL descargas/filtros/tabla color/MFRPS + lote UX: perfil tipo Facebook, ranking con
  foto, dark difuminado, login dark, flash auto-descartar), **#46** (filtro por dimensión en CEAL +
  réplica completa en NOSACQ). Todo en `master`. Suite **128 tests en verde**.
- *Nota UX pendiente:* al filtrar el radar de NOSACQ a **<3 dimensiones** degenera (un radar necesita
  ≥3 ejes); se dejó habilitado, pero se podría cambiar a barras como en CEAL si se pide.

### Aportes del equipo integrados (2026-08-27)
Fusionados a `master` por el equipo y ya en `integracion/local`: **PR #44** ("por página" en vivo con
Livewire en la lista de contenidos del módulo — `App\Livewire\ContenidosDelModulo`) y **PR #45**
(descargar el cronograma/Gantt como **PNG**). También el trabajo previo (Jitsi con usuario del
sistema, contenidos con período en el Plan/Gantt).

### Deploy: cache-bust del CSS + método GitHub→servidor
- **Cache-bust por hash (PR #47):** los layouts versionan `corporate.css` por **`md5_file`** (hash de
  contenido) en vez de `filemtime`, porque el FTP suele conservar el mtime y el navegador seguía
  sirviendo el CSS viejo tras un deploy.
- **`.gitignore`:** se añadieron `/storage/docs` y `/public/.user.ini` (evita commitearlos por error).
- ⚠️ **El método de despliegue GitHub→servidor NO está documentado** (sin CI/CD, sin `.cpanel.yml`,
  sin script ni credenciales persistentes). Lo único verificable es el `git push` a GitHub. El
  hosting es tipo **cPanel/FPM** (`public/.user.ini`). `public/` **sí** se versiona (solo se ignoran
  `build/hot/storage`); `corporate.css` está en `master` (883 líneas). Un incidente típico: blade
  nuevo + `corporate.css` viejo en el servidor porque la subida no refrescó `public/css/`. **Recomendado:**
  apuntar el document root del dominio a la carpeta `public/` del proyecto y desplegar por `git pull`.

### Seguridad: hallazgos del pentest y fixes (PR #47 — 2026-08-28) ⭐
Se analizó un **reporte de pentest (VAPT) de Astra** para *International SOS – Pasaporte Médico* (app
DISTINTA de SafePoint, mismo cliente). Hallazgo #1: acceso indebido a PII de pacientes. Inspiró una
**auditoría IDOR/broken-access-control en SafePoint** (la mayoría del sistema valida propiedad bien).
Se corrigieron 2 cosas y quedó todo en `master`:
- **Fuga del padrón de pacientes (citas, modo gestor):** `citas.admin.crear` y `AgendarCita` en modo
  gestor se protegían con `citas.crear` — permiso que TAMBIÉN tiene el rol Usuario (autoservicio). Un
  paciente podía abrir el modo gestor y enumerar nombre/email de **todos** los pacientes. Fix en dos
  capas: la ruta pasa a `citas.editar` y el `mount` exige `esGestor()`.
- **Adjuntos de paciente en disco privado:** las subidas de paciente (adjuntos de apartado/respuesta y
  de comentarios) se servían por `/storage/...` sin auth (nombre aleatorio, pero accesible por URL).
  Ahora viven en disco **privado** (`storage/app/private`) y se descargan por rutas autenticadas
  (`AdjuntoController@descargar`, `ComentarioController@adjunto`) que validan propiedad (respuesta) o
  visibilidad (contenido/comentario). Se sirven con `response()->file` (BinaryFileResponse → soporta
  HTTP Range: imágenes/PDF embebidos y seek de video). Migración movió los archivos existentes y
  reescribió URLs. Recursos educativos (video/doc) y avatares/charlas siguen en público.
- **RBAC — dato clave:** el rol **Administrador es super-admin** (`Gate::before` en `AppServiceProvider`):
  todo `@can`/`permission:` es `true` para él, así que sus toggles de permiso son **decorativos**. Para
  restringir una acción a no-administradores se usa OTRO rol (p. ej. `cuestionarios.eliminar` se dejó
  fuera del Moderador → solo el Admin borra cuestionarios).

### Rendimiento: Feed paginado en la BD (PR #47 — 2026-08-28) ⭐
`FeedController` materializaba TODOS los contenidos + charlas en PHP y paginaba con `forPage` (cargaba
miles para mostrar 8). Ahora hace un **UNION** de `(id, created_at, tipo)` de ambas tablas, pagina con
`LIMIT/OFFSET` **en la BD** e **hidrata solo los ~8 items** de la página. Escala O(perPage) en filas.
*Pendiente:* `DashboardProgresoController` sigue materializando en PHP (paginar/precalcular cuando el
nº de pacientes crezca).

### Eliminar entidades con confirmación (PR #47 — 2026-08-28)
Botones **Eliminar** con confirmación (`data-confirm` + `<dialog>`) y borrado en cascada (FKs
`cascadeOnDelete`): **proyecto** (solo Admin), **módulo** (desde su propia página; ya existía desde el
proyecto), **cuestionario/plantilla** (solo Admin; destructivo: arrastra preguntas/asignaciones/
campañas). Cascada borra filas, no los archivos físicos de recursos (huérfanos, limpieza futura).

### Citas: cancelar (con motivo) y reprogramar por el paciente (PR #47 — 2026-08-28)
El paciente YA podía cancelar tras confirmar (regla de 24h, `citas.cancelacion_horas_min`); se mejoró:
- **Cancelar** ahora pide **motivo** (col. `citas.motivo_cancelacion`) y **avisa por correo** al
  psicólogo (`CitaCancelada`, best-effort). Cuando falta la ventana de 24h, se muestra un aviso
  explicativo en vez de ocultar el botón sin más.
- **Reprogramar** reusa el flujo de agendar (respeta disponibilidad); es **no destructivo**: la cita
  original se cancela solo tras crear la nueva (`AgendarCita` con `reprogramarDe`).
- *Nota deploy:* los correos de citas se envían **síncronos** (`Mail::send`); encolarlos (`->queue()` +
  worker) está **parqueado** hasta confirmar que el servidor puede mantener un `queue:work`.

### UX móvil + PWA (IMPLEMENTADO — 2026-09-03, PR #52) ⭐
Pedido del cliente: que en móvil se vea como app (menú tipo FB/IG abajo) e instalable.
- **Navegación móvil** (<992px, desktop intacto): **barra inferior fija** por rol (4 accesos + ☰ Más,
  con badge de cuestionarios pendientes en el paciente) y un **bottom-sheet** que sube (overlay oscuro
  + arrastrar hacia abajo para cerrar) con el **menú completo por rol**. Se quitó la hamburguesa del
  topbar en móvil. El menú se extrajo a `resources/views/partials/nav-menu.blade.php` y lo reusan el
  sidebar (desktop) y el sheet (móvil) — misma lógica de permisos/roles. Barra del paciente:
  Feed·Citas·Cuestionarios·Ranking; del gestor: Feed·Panel·Pacientes·Gestión de citas.
- **PWA** (`public/manifest.webmanifest`, `public/sw.js`, `public/offline.html`, `public/icons/*`):
  instalable, `display: standalone`, íconos corazón-pulso (192/512 + maskable), service worker
  **conservador** (red primero; solo página offline en navegaciones sin conexión; NO cachea HTML
  dinámico). Cabecera PWA (`partials/pwa-head`) en los layouts app y guest. **Requiere HTTPS en
  producción** (en localhost sirve para probar; en `http://*.test` el SW no registra).
- **Panel del paciente** rediseñado: tarjetas accionables (próxima cita / cuestionarios pendientes /
  progreso) + acciones rápidas contextuales; hero estilo Facebook con la **foto de portada** del
  usuario (misma `users.portada` de Mi perfil, SOLO LECTURA en el Panel; respaldo al índigo si no hay).
- **Inicio por rol** (`User::rutaInicio()`): el paciente SIEMPRE arranca en el **Feed** (también desde
  la URL raíz `/`); el gestor en el Panel.
- *Pendiente (fase 2):* adaptar a móvil las ~35 vistas con tablas anchas (tarjetas apiladas), el Gantt
  y los gráficos.

### Producción HTTPS + fixes de módulos/ranking (IMPLEMENTADO — 2026-09-09) ⭐
La app pasó del servidor de pruebas HTTP (`10.10.10.29`) a **producción con HTTPS**
(`safepoint.internationalsos-peru.com`). Tanda de correcciones a partir de pruebas del cliente:
- **#53 — Forzar HTTPS** (`AppServiceProvider`): `URL::forceScheme('https')` **condicionado al esquema
  de `APP_URL`** (no a `APP_ENV`), para no romper HTTP/local; se activa con `APP_URL=https://…`.
  Evita contenido mixto y habilita instalar la PWA (requiere contexto seguro).
- **#54 — Editar módulos:** antes solo se podía crear/publicar/eliminar; ahora `edit()`+`update()`
  (permiso `modulos.editar`) editan **nombre y descripción**, con botón ✏️ en los 3 lugares (listado
  global, ficha del proyecto, página del módulo). No se toca proyecto/orden/publicación.
- **#55 — Ranking sin proyecto:** el paciente sin proyecto recibía `abort(404)`; ahora ve una **página
  amable** ("Aún no perteneces a ningún proyecto") y el **bottom-nav** solo muestra "Ranking" si tiene
  proyecto (si no, "Proyectos"), coherente con el sidebar.
- *Nota de deploy (móvil):* como el `sw.js` cambió, en el dispositivo conviene **Unregister** del
  service worker / "Clear site data" para no servir respuestas viejas desde caché.

### Filtros por demográficos numéricos en Resultados (IMPLEMENTADO — 2026-09-09, PR #60) ⭐
Los Resultados grupales anónimos solo filtraban por demográficos **categóricos** (género, estado civil,
tiempo en la empresa, zona). Ahora también por los **numéricos** (edad, N.º de hijos) mediante **rangos**
(dropdown), no valor exacto —los rangos también favorecen el anonimato.
- **Rangos:** Edad → Hasta 25 · 26–35 · 36–45 · 46–55 · 56 o más; N.º de hijos → 0 · 1 · 2 · 3 o más;
  otro numérico → tramos de 10.
- **Paridad CEAL ↔ NOSACQ**: lógica de buckets en el **trait** `App\Support\RangosDemograficos`
  (sin duplicar). En NOSACQ el rango se propaga por `RespuestaCampana::scopeFiltrado()` y los servicios
  `ResultadosCampana`/`MatrizCampana`; en CEAL se aplica inline en la consulta.
- Se conserva la **guardia de anonimato** (subgrupo < 5 → resultados ocultos) y el export (PNG/Excel).
- El equipo sumó (#61) un **selector de tipo de gráfico** (7 vistas) en Resultados de CEAL, sin recargar.

### Otras ideas / pendientes en cola
- ✅ **Acceso a la gestión GLOBAL de Contenidos/Módulos — RESUELTO (2026-08-25):** se agregaron
  botones **"Todos los contenidos"** y **"Todos los módulos"** en la página de **Proyectos** (con
  **Volver** a Proyectos), manteniendo la regla del cliente (siguen ocultos del sidebar; no se tocó
  `config/menu.php`).
- **Búsqueda global de contenidos** — se retiró la búsqueda al consolidar módulos/contenidos
  dentro de Proyectos; el cliente podría volver a pedirla más adelante.
- **Integración Jitsi con JWT** — a la espera de que el cliente habilite JWT y entregue
  `app_id`/`app_secret`; hoy la sala es sin auth.
- **Alcance clínico del Moderador** — hoy un psicólogo ve datos de **todos** los pacientes;
  revisar need-to-know por proyecto/asignación (decisión de producto).
- ✅ **Login en modo oscuro — RESUELTO (PR #43):** el layout guest respeta el tema guardado
  (`localStorage['safepoint-theme']` sin parpadeo) y el login tiene su propio toggle.
- **Rendimiento** — ✅ Feed paginado en BD (PR #47); falta `DashboardProgresoController` (aún
  materializa en PHP) y encolar los correos de citas. Overhead en LOCAL (Laragon): opcache apagado +
  sesión/caché en BD; en prod usar opcache + `php artisan optimize`. **Tooling** — sin CI/PHPStan;
  FormRequests en vez de validación inline.
