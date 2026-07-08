# Contexto del proyecto — SafePoint

> Documento de contexto autocontenido para entregar a otra IA / desarrollador.
> Resume arquitectura, decisiones, funcionalidades y estado del repositorio.
> Última actualización: julio 2026 (tras integrar en `master`: cuestionarios/escalas,
> adjuntos en comentarios, citas psicológicas + horarios, y recordatorios por rol).

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
- Paleta corporativa (solo estos 3 colores):
  - Blanco `#ffffff`
  - Indigo `#232762` (primario: sidebar, botones, encabezados)
  - Naranja `#f99e16` (acento: estados activos, iconos, CTAs)
- Tokens y componentes en `public/css/corporate.css`
  (`--corp-indigo`, `--corp-orange`, `.btn-corp`, `.card-corp`, `.sidebar-*`, `.feed-*`).
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
  `usuarios, roles, proyectos, modulos, contenidos, cuestionarios, reuniones,
  documentos, videos, imagenes, evaluaciones, citas, charlas, feriados, psicologos`.
- **Administrador es super-admin** vía `Gate::before()` en `AppServiceProvider`.
- **Moderador (psicólogo):** gestiona contenido/módulos/cuestionarios/**charlas** y
  `citas.ver/editar` (atiende citas, no agenda).
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

**Cuestionarios / escalas** — ver sección 7.

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
  (`pendiente|finalizada`), `foto` (evidencia), `creado_por`. Pivote `charla_user`
  con `asistio`. `estaFinalizada()`.

---

## 6. Funcionalidades (todas en `master`)

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

### Seeders (`DatabaseSeeder`, en orden)
`RolSeeder`, `PermissionSeeder`, `UserSeeder`, `ModeradorSeeder`,
`PsicologoDemoSeeder`, `HorarioDefectoSeeder`, `ProyectoSeeder`, `ContenidoDemoSeeder`,
`CuestionarioDass21Seeder`, `CuestionarioNosacq50Seeder`, `Nosacq50RespuestasDemoSeeder`,
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
- **DASS-21** — escala 0-3, suma (×2), 3 dimensiones (depresión/ansiedad/estrés).
- **NOSACQ-50** — escala 1-4, promedio, **7 dimensiones**, **21 ítems inversos**;
  `requiere_puesto` separa Trabajadores/Directivos.

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

**Paneles**
- Usuario: **Mis citas** (ver/cancelar, respeta política de cancelación).
- Psicólogo: **Mi horario** (`App\Livewire\GestionHorario`, define su disponibilidad;
  se precargan 12 bloques por defecto editables).
- Admin/Moderador: **Gestión de citas** (estado), CRUD de **Psicólogos** (con botón
  Horarios) y **Feriados** (tabla).

---

## 9. Módulo de Charlas

- CRUD de charlas + **asistencia** (`charla_user.asistio`) vía Livewire
  `App\Livewire\GestionCharla` en `/charlas/{id}` (`charlas.show`).
- Flujo: agregar asistentes → **finalizar** → subir **foto de evidencia**.
- **En el feed**: charlas **finalizadas con foto** (registro con asistencia) y
  charlas **programadas a futuro** (anuncio). Cada tarjeta tiene ancla `#charla-ID`
  (los recordatorios enlazan a ella con realce `:target`). Gestores ven **todas** las
  charlas con pie de gestión (N inscritos + Gestionar).

> ⚠️ **Convención de foto** (charla) y adjuntos: se guardan como **ruta relativa**
> `/storage/...` (con `parse_url(..., PHP_URL_PATH)`), no absoluta, para que la imagen
> resuelva sin importar el host/puerto. El **borrado** (`GestionCharla` y
> `CharlaController::eliminarFoto`) usa el mismo prefijo `/storage/`. Los contenidos
> (videos/imágenes/documentos) **sí** siguen usando URL absoluta (coherente aparte).

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
  `feat/...` y se **consulta al dueño antes de hacer `git push`**.
- `gh` CLI **no** está instalado; los PRs se crean con el enlace de "compare".
- Tras sincronizar: `composer install`, `php artisan migrate`,
  `php artisan storage:link`, `php artisan optimize:clear`.

### Estado actual
- `master` (`origin`) al día, contiene **todo** lo de las secciones 6–10:
  cuestionarios/escalas, adjuntos en comentarios, citas + horarios, charlas en el
  feed, recordatorios por rol, y los fixes de foto de charla (guardado y borrado
  con ruta relativa).
- Ramas de esas features **mergeadas y borradas** (repo limpio).

### Repo de documentación
- Este archivo `docs/CONTEXTO-PROYECTO.md` **no** se versiona en el repo del proyecto;
  vive en el git personal **`github.com/atestertesting/docs.git`**.

---

## 12. Cómo levantar el proyecto

```bash
composer install                    # incluye Livewire y getid3
cp .env.example .env                # DB_DATABASE=psicologia, MySQL 127.0.0.1:3306, root
php artisan key:generate
php artisan migrate:fresh --seed
php artisan storage:link            # multimedia/fotos/adjuntos al storage
php artisan serve                   # http://127.0.0.1:8000
```

- No requiere `npm` para la UI (Bootstrap, Chart.js, flatpickr por CDN).
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
