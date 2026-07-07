# Contexto del proyecto — SafePoint

> Documento de contexto autocontenido para entregar a otra IA / desarrollador.
> Resume arquitectura, decisiones, funcionalidades y estado del repositorio.
> Última actualización: julio 2026.

---

## 1. Resumen ejecutivo

**SafePoint** es una plataforma web de **acompañamiento psicológico y clima
laboral**. Administradores y moderadores gestionan proyectos, módulos de
entrenamiento y contenidos (videos, imágenes, documentos, audios, reuniones,
evaluaciones); los usuarios consumen el contenido publicado en un **feed estilo
red social** (con me gusta y comentarios) y responden **cuestionarios/escalas
psicométricas** que se les asignan.

El control de acceso es por **roles y permisos (RBAC)** con tres roles:
Administrador, Moderador y Usuario.

---

## 2. Stack técnico

| Componente | Detalle |
|---|---|
| Framework | **Laravel 13** (`laravel/framework ^13.8`) |
| Lenguaje | **PHP 8.3** |
| Base de datos | **MySQL**, BD `psicologia` |
| Entorno | **Laragon** en Windows (shell: PowerShell / Git Bash) |
| Auth/RBAC | **spatie/laravel-permission 8** |
| UI reactiva | **Livewire 4** (tablas de usuarios/roles, confirmaciones) |
| Multimedia | **spatie/laravel-package-tools** + `james-heinrich/getid3` (metadatos audio); storage local (`storage:link`) |
| Gráficos | **Chart.js 4** por CDN (diagramas radar de cuestionarios) |
| Frontend | **Bootstrap 5.3 + Bootstrap Icons por CDN** (sin build) |
| Estilos propios | `public/css/corporate.css` |
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
- El pivot `role_user` se reemplazó por las tablas de Spatie.
- `App\Models\User` usa el trait `HasRoles`; helpers `esGestor()` (Admin/Moderador),
  `proyectoIds()`, `puedeVerContenido()`.

### Roles y permisos

- **48 permisos** `{recurso}.{accion}` sobre 12 recursos (`usuarios, roles,
  proyectos, modulos, contenidos, cuestionarios, reuniones, documentos, videos,
  imagenes, evaluaciones, citas`) × (`ver, crear, editar, eliminar`).
- **Administrador es super-admin** vía `Gate::before()` en `AppServiceProvider`.
- Middleware en `bootstrap/app.php`: `role`, `permission`, `role_or_permission`.
  Uso: `->middleware('permission:contenidos.crear')`.

---

## 5. Modelo de datos (entidades relevantes)

**Jerarquía de contenido**
- **Proyecto** → muchos **Módulos**; pivote `proyecto_user` (miembros del proyecto).
- **ModuloEntrenamiento** (`modulos_entrenamiento`) → pertenece a Proyecto; muchos
  **Contenidos**; campo `publicado`.
- **Contenido** (`contenidos`) → pertenece a un Módulo. `tipo` (enum: `video,
  documento, imagen, reunion, evaluacion, audio`), `publicado`, `publicar_en`
  (publicación programada). Relaciones `hasMany` → **Video/Imagen/Documento/Audio**;
  `hasOne` → **Reunion/EvaluacionPsicologica**.
- **Like** y **Comentario** → interacción del feed (pertenecen a Contenido + User).

**Alcance por proyecto:** el feed y el dashboard del Usuario se filtran por los
proyectos a los que pertenece (`proyecto_user`). Admin/Moderador (`esGestor`) ven todo.

**Cuestionarios / escalas psicométricas**
- **Cuestionario** (`cuestionarios`): `titulo`, `descripcion`, `tipo` (nullable,
  clave en `config/cuestionarios.php`), `modulo_id` (nullable), `activo`.
- **PreguntaCuestionario** (`preguntas_cuestionario`): `enunciado`, `orden`,
  `dimension`, `invertida` (ítem inverso).
- **CuestionarioAsignacion** (`cuestionario_asignaciones`): un gestor habilita un
  cuestionario a un `user`; `estado` (pendiente/completado), `es_directivo`
  (trabajador/directivo, para NOSACQ), `completado_at`.
- **CuestionarioRespuesta** (`cuestionario_respuestas`): `valor` Likert por pregunta.

**Otros:** Moderador (perfil de un User), Cita, Cuestionario de quiz heredado, etc.

---

## 6. Funcionalidades (todas en `master`)

1. **Login** Bootstrap split-screen + credenciales demo. Redirección post-login:
   Usuario → **feed**; Moderador/Admin → **dashboard**.
2. **Navigation drawer** (sidebar) filtrado por permiso; el filtro acepta permiso
   `null`, string (`can`) o **Closure** (condición dinámica).
3. **Dashboard** role-aware: gestores ven permisos; usuarios ven su **progreso**
   de contenido (por proyecto).
4. **Proyectos** — CRUD + gestión de sus **módulos** + **miembros** (`proyecto_user`).
5. **Módulos** — CRUD, publicar/despublicar, gestión de sus **contenidos**.
6. **Contenidos** — creación **por tipo** (crea su recurso relacionado en
   transacción; soporta **subida de multimedia al storage** o URL), editar,
   eliminar, publicar; **publicación programada** (`publicar_en`), buscador.
7. **Feed** estilo red social — contenido **vigente** (publicado o programado ya
   cumplido) y del alcance del usuario; **me gusta** y **comentarios** reales.
8. **Usuarios / Roles y permisos** — gestión con **tablas Livewire** reactivas.
9. **i18n ES/EN** — selector de idioma; `SetLocale` (usuario > sesión > default);
   `POST /idioma/{locale}`; textos con `__()` y `lang/en.json`.
10. **Cuestionarios / escalas** — ver sección 7.

### Seeders (`DatabaseSeeder`, en orden)
`RolSeeder`, `PermissionSeeder`, `UserSeeder`, `ModeradorSeeder`, `ProyectoSeeder`,
`ContenidoDemoSeeder`, `CuestionarioDass21Seeder`, `CuestionarioNosacq50Seeder`,
`Nosacq50RespuestasDemoSeeder`.

### Usuarios demo (contraseña: `password`)
| Email | Rol |
|---|---|
| `admin@psicologia.test` | Administrador |
| `moderador@psicologia.test` | Moderador |
| `usuario@psicologia.test` | Usuario |

---

## 7. Módulo de Cuestionarios / escalas psicométricas

Se **repurposó** la sección "Cuestionarios" (tablas heredadas de quiz) para
**escalas psicométricas** de puntaje continuo.

**Calificación config-driven** (`config/cuestionarios.php` por `tipo`) +
`App\Services\CalificadorCuestionario`:
- `metodo`: `suma` (subpuntaje = suma × `factor`) o `promedio` (media de ítems).
- Ítems `invertida` se puntúan al revés dentro de la escala (`min+max - valor`).
- Nivel por dimensión con `bandas` (primer `hasta` no superado; `null` = tope).

**Escalas incluidas**
- **DASS-21** — escala 0-3, `metodo` suma (×2), 3 dimensiones
  (depresión/ansiedad/estrés), niveles Normal → Extremadamente severo.
- **NOSACQ-50** — escala 1-4, `metodo` promedio, **7 dimensiones** de clima de
  seguridad, **21 ítems inversos**, niveles Bajo/Med. bajo/Med. bueno/Bueno.
  `requiere_puesto` → pide "¿tiene puesto directivo?" para separar Trabajadores/Directivos.

**Flujo**
1. Admin/Moderador (`cuestionarios.crear`): en `/cuestionarios/{id}` **asigna** el
   cuestionario a un usuario y ve el estado de cada asignación.
2. Usuario: **"Mis cuestionarios"** (solo visible para no-gestores) → responde el
   formulario Likert; las respuestas se guardan y la asignación pasa a completada.
3. Admin: **revisa respuestas** (`/cuestionarios/asignaciones/{id}`) y ve los
   **resultados calculados** por dimensión.
4. **Resultados y diagramas** (`/cuestionarios/{id}/resultados`):
   `App\Services\ResultadosCuestionario` agrega la **media por dimensión** de todas
   las respuestas completadas (Todos / Trabajadores / Directivos). La vista muestra
   **diagramas radar (Chart.js)** — Diagrama 1 (todos), Diagrama 2 (trabajadores vs
   directivos) — y la **tabla de resultados por dimensión** con su nivel.

**Para agregar más escalas del mismo tipo:** crear el `Cuestionario` con ese `tipo`
+ un `tipo` en `config/cuestionarios.php` + un seeder con las preguntas etiquetadas
por `dimension` (e `invertida` si aplica).

---

## 8. Estado del repositorio y flujo de trabajo

### Regla de git (importante)
- **NO se trabaja sobre `master` directamente.** Cada funcionalidad va en su rama
  `feat/...` y se **consulta al dueño antes de hacer `git push`**.
- Los PRs se mergean en GitHub; luego se sincroniza con
  `git pull --ff-only origin master` (+ `composer install`, `php artisan migrate`,
  `php artisan storage:link` cuando aplica).
- `gh` CLI **no** está instalado; tras el push se usa el enlace
  `.../pull/new/<rama>`.

### Estado actual
- `master`: al día con `origin`, contiene todo lo de la sección 6.
- Rama abierta: **`feat/cuestionarios-dass21`** (DASS-21 + NOSACQ-50 + diagramas),
  ya subida a `origin` (PR pendiente de crear/revisar).

---

## 9. Cómo levantar el proyecto

```bash
composer install                    # incluye Livewire y getid3
cp .env.example .env                # DB_DATABASE=psicologia, MySQL 127.0.0.1:3306, root
php artisan key:generate
php artisan migrate:fresh --seed
php artisan storage:link            # multimedia subida al storage
php artisan serve                   # http://127.0.0.1:8000
```

- No requiere `npm` para la UI (Bootstrap y Chart.js por CDN).
- Tras cambiar de rama o mergear: `php artisan config:clear && php artisan migrate`.

---

## 10. Convenciones para seguir desarrollando

- Respetar la **paleta corporativa** y reusar las clases de `corporate.css`.
- Vistas Blade con Bootstrap (CDN), iconos `bi-*`, textos en español envueltos en
  `__()` (y añadir la clave a `lang/en.json`).
- Proteger rutas con `permission:{recurso}.{accion}`; Administrador omite los checks
  (`Gate::before`). Para "gestión" de cuestionarios se usa `cuestionarios.crear`
  (excluye al rol Usuario, que sí tiene `.ver`).
- Validación en controladores; operaciones multi-tabla en **transacción**.
- Formatear con **Pint** antes de cerrar (`vendor/bin/pint <archivos>`).
- Verificar en servidor real (login + flujo) antes de dar por terminado.
- Posibles siguientes pasos: Cronbach's Alpha y filtros por sector del NOSACQ,
  editar/eliminar cuestionarios y preguntas desde la UI, exportar resultados (PDF/CSV),
  agenda de citas, evaluaciones psicológicas interactivas.
