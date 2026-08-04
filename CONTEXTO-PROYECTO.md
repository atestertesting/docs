# Contexto del proyecto — SafePoint

> Documento de contexto autocontenido para entregar a otra IA / desarrollador.
> Resume arquitectura, decisiones, funcionalidades y estado del repositorio.
> Última actualización: agosto 2026. **Todo lo descrito está en `master`** (el equipo también
> aporta en paralelo). Lo más reciente **mergeado a `master`**: **escalas BAI/BDI-II** +
> interpretación/matriz/export del DASS-21, **datos demográficos** del paciente, **Informe
> Psicológico de Seguimiento** (PDF), **ficha/expediente del paciente**, **permisos PACIENTES**,
> **gestor programa citas**, imagen de referencia en charlas, y el **Plan de trabajo tipo Gantt
> COMPLETO**: cronograma, asistencia, **seguimiento preventivo (3c)**, **participación integral**,
> **alcance general (todos los proyectos)** + filtro por proyecto, y dashboards con Chart.js.
> Ver secciones 11, 15 y 16. *(Aportes del equipo en master: comentarios/bitácora del cronograma,
> avance de talleres por semana, modalidad presencial/virtual, ranking trimestral, calificaciones.)*

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
| Videollamada | Enlaces **Jitsi** autogenerados (`https://meet.jit.si/SafePoint-Cita-…`) |
| Correo | `Mail::to()->send()` (confirmación de cita, best-effort en try/catch) |
| Frontend | **Bootstrap 5.3 + Bootstrap Icons por CDN** (sin build) |
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
  imagen, reunion, evaluacion, audio`), `publicado`, `publicar_en` (programada).
  `hasMany` → Video/Imagen/Documento/Audio; `hasOne` → Reunion/EvaluacionPsicologica.
- **Like** y **Comentario** → interacción del feed. **Comentario** ahora soporta
  **adjunto** (`adjunto_url`, `adjunto_nombre`; `cuerpo` nullable → permite adjunto solo).

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
(solo citas `completada`).

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

> Los ítems **1–16** están en `master`. Los **17–24** van en el **PR consolidado
> `release/consolidado-2026-07`** (pendiente de merge; ver secciones 11, 15 y 16).

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

### Seeders (`DatabaseSeeder`, en orden)
`RolSeeder`, `PermissionSeeder`, `UserSeeder`, `ModeradorSeeder`,
`PsicologoDemoSeeder`, `HorarioDefectoSeeder`, `ProyectoSeeder`, `ContenidoDemoSeeder`,
`CuestionarioDass21Seeder`, `CuestionarioBaiSeeder`, `CuestionarioBdiSeeder`,
`CuestionarioNosacq50Seeder`, `Nosacq50RespuestasDemoSeeder`,
`FeriadoSeeder`. (No hay `CharlaSeeder`: las charlas se crean desde la UI.)

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

---

## 11. Estado del repositorio y flujo de trabajo

### Regla de git (importante)
- **NO se trabaja sobre `master` directamente.** Cada funcionalidad va en su rama
  `feat/...` y se **consulta al dueño antes de hacer `git push`** y antes de mergear.
- **Integración continua local:** tras cada cambio, la rama se **fusiona en
  `integracion/local`** (creada desde `master`) para ver todo junto y resolver conflictos
  poco a poco. `integracion/local` **NO se sube** al remoto; es solo de revisión.
- `gh` CLI **no** está instalado; los PRs se crean con el enlace de "compare".
- Tras sincronizar: `composer install`, `php artisan migrate`,
  `php artisan storage:link`, `php artisan optimize:clear`.

### Estado actual
- **`master` (`origin`) está al día con TODO lo de este documento** (nada pendiente de PR
  por nuestra parte). Se mergearon, en orden:
  1. **PR consolidado `release/consolidado-2026-07`** (#21): escalas BAI/BDI-II, interpretación/
     matriz/export DASS-21, datos demográficos, Informe de Seguimiento (PDF), ficha/expediente,
     permisos PACIENTES, gestor programa citas, imagen de charla. *(Antes, PRs #16/#18/#20:
     pacientes, dashboard progreso, asignación masiva, alcance de citas, tema oscuro.)*
  2. **PR `release/plan-gantt-2026-08`**: el **Plan/Gantt completo** — seguimiento 3c,
     participación integral, alcance general + filtro, dashboards Chart.js (9 commits,
     fast-forward; ver sección 16).
- **`integracion/local`** (solo local, **NO se sube**): rama de integración continua. Tras el
  merge quedó **igual a `origin/master`**. Los PRs se generan ramificando de `master` y
  fusionando `integracion/local` (fast-forward cuando master no divergió).
- **El equipo aporta en paralelo en `master`** (comentarios/bitácora del cronograma, avance de
  talleres por semana, modalidad, ranking trimestral, calificaciones). Conviene **sincronizar
  `integracion/local` con `origin/master`** antes de empezar cambios nuevos.
- Al desplegar: `composer install` (dompdf), `php artisan migrate` (aditivas), `optimize:clear`.

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
**Sigue pendiente:** contraseña temporal fija + throttle de login, `whereRaw CONCAT`,
login en claro, tests, `ext-zip` en `composer.json`, seeders demo en producción, y el
*alcance clínico del Moderador* (aún ve la ficha de **todos** los pacientes).

**Prioridad alta**
- **Seguridad:** contraseña temporal **hardcodeada** `'SafePoint123'` (igual para todos
  los pacientes importados, se muestra en pantalla, sin forzar cambio) — `ImportadorPacientes`.
  Y **login sin `throttle`** (fuerza bruta) — `routes/web.php`. Es la cadena más explotable.
- **Rendimiento:** `whereRaw("CONCAT(fecha,' ',hora) >= ?")` (5 sitios) **anula los
  índices** de `citas` → comparar por columnas. `FeedController` carga **todo** el feed
  en memoria y pagina en PHP. `DashboardProgresoController` materializa todo en PHP.
- **Frontend:** el **login queda en claro** (layout guest sin `data-bs-theme` + `bg-white`);
  `dashboard/progreso.blade.php` usa hex pastel fijos que rompen el modo oscuro.
- **Calidad:** **sin tests reales** (solo `ExampleTest`); README genérico de Laravel;
  `ext-zip` no declarada en `composer.json`; **seeders demo se ejecutan con los de producción**.

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
