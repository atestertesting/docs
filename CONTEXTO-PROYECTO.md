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
   gusta**, **comentarios con adjuntos**, y **charlas** (finalizadas con foto =
   registro; programadas = anuncio con ancla `#charla-ID`). Gestores ven todas las
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

**Pendiente / futuro:** **Hub de Reportes** (lista global de informes) como cierre de la fase de
reportes; estados de asistencia más ricos (regular/reforzamiento/motivo de ausencia) si el cliente
los pide.
