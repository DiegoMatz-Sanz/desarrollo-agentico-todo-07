# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es

Aplicación TODO (una sola lista de tareas, sin usuarios) en Go: Buffalo + Pop (ORM) + plantillas Plush sobre PostgreSQL. Además contiene `adw`, un workflow agéntico que trabaja issues de GitHub de extremo a extremo (planifica, implementa, verifica con evidencias, documenta y abre el PR). La app es el "sujeto" sobre el que trabaja `adw`; `adws/`, `config/adw.yml` y `.claude/` no son parte de la app.

La documentación vigente está en `docs/` (índice en `docs/README.md`). Antes de tocar comportamiento, léela: `docs/knowledge/overview.md` (qué hace, rutas, datos) y `docs/procedures/conventions.md` (capas, patrones y rúbricas con las que se juzga un cambio).

## Comandos

```bash
bin/setup                                   # idempotente: buffalo/soda, npm ci, Chromium, PostgreSQL y base todo_test migrada
bin/dev                                     # app en primer plano (host: PostgreSQL en Docker + `buffalo dev`); `bin/dev --docker` todo en contenedores
GO_ENV=test go test -p 1 ./...              # tests de Go; -p 1 obligatorio: todos los paquetes comparten todo_test
GO_ENV=test go test -p 1 ./actions -run Test_TasksCreate   # un test (comando estándar de go test, no documentado en el repo)
npx playwright test                         # tests de navegador (arranca la app con GO_ENV=test en :3001)
npx playwright test e2e/tareas/anadir-una-tarea.spec.ts    # un spec
bash .claude/skills/check/scripts/check.sh [gate...]       # lo mismo que la CI: gofmt, go vet, migrate, test, build, e2e, docker (p. ej. `check.sh migrate test`)
go test -p 1 ./adws                         # tests del lanzador de adw (gh/git/claude simulados)
bin/adw [--once]                            # lanzador de issues; --once ejecuta Claude de verdad
```

- Los tests de Go necesitan `todo_test` migrada (`bin/setup` o `soda migrate -e test`). La app aplica migraciones pendientes al arrancar.
- App en http://localhost:3000, `/healthz` indica salud (503 si falla la base). Puertos 3000/3001/5432 en el checkout principal.
- `.claude/commands`: `/env:start` y `/env:stop` levantan/paran el entorno para el agente; `/git:commit`; `/prime:*` cargan contexto.

## Worktrees por caso

Cada caso de `adw` trabaja en `trees/issue-N` (`bin/tree create N` / `bin/tree remove N [--force]`) con su propio PostgreSQL, contenedores y puertos (N mod 1000 = n: `APP_PORT` 13000+n, `E2E_PORT` 14000+n, `DB_PORT` 15000+n). `go test` y Playwright **no** leen el `.env` solos: antes de trabajar en un worktree (y antes de abrir `claude`) ejecuta `set -a; . ./.env; set +a`. `check.sh`, `bin/setup` y `bin/dev` lo cargan solos. No pongas `PORT` en el `.env` (pisaría el puerto que Playwright da a la app de test).

## Arquitectura de la app

Dependencias: `actions` → `models`. `models` nunca importa `actions` ni `templates`.

- `actions/app.go`: solo rutas y middleware (HTTPS en producción, CSRF —desactivado con `GO_ENV=test`—, transacción por petición).
- `actions/tasks.go`: acciones `<Recurso><Acción>` (`TasksCreate`, …). Patrón: `tx := c.Value("tx").(*pop.Connection)`; `TrimSpace` a la entrada; `ValidateAndCreate/Update`; error de validación → re-render con `422`; éxito → flash + redirect `303` a `/`; no encontrado → `c.Error(404, err)`. `PUT`/`DELETE` llegan como `POST` + `_method`.
- `models/task.go`: tipo `Task`, `Validate` y reglas del dominio (`TitleMaxLength` = 200). Mensajes de validación en español, frase completa con punto. Los cálculos del dominio no van en plantillas ni JS.
- `templates/` (Plush, salida siempre con `<%= %>`), `public/assets/app.{css,js}` (mejora progresiva, sin dependencias), `migrations/` (Fizz para esquema, SQL para datos, siempre con `down`; embebidas en el binario), `fixtures/` (TOML para tests).
- Interfaz y comentarios en español; los identificadores exportados llevan comentario que empieza por su nombre.
- Tests: `ActionSuite`/`ModelSuite` (gobuffalo/suite), nombres `Test_<Recurso><Acción>_<Caso>`; comprueban código HTTP **y** efecto en base. Lo que depende del navegador va en `e2e/<grupo>/<caso>.spec.ts` (un test por fichero, fixtures de `e2e/fixtures.ts`, localizadores `getByRole`).
- Un cambio que altere un comportamiento/regla/texto/comando descrito en `docs/knowledge/` o `docs/procedures/` actualiza esos documentos en el mismo cambio, escribiendo lo vigente (sin «antes/ahora» ni número de caso).

### Zonas que un caso no toca salvo que la issue lo pida

`.github/`, `.claude/`, `.devcontainer/`, `adws/`, `config/`, `go.mod`/`go.sum`, `package*.json`, `playwright.config.ts`, migraciones ya existentes, `docs/procedures/conventions.md` y `docs/history/` de otros casos.

## Arquitectura de `adw` (requiere leer varios ficheros)

- **Trigger** (`adws/`, Go): sondea issues con la etiqueta `adw` cuyo turno es del agente (sin `adw:waiting-for-human`, sin `stage:*`, sin asignar a otro), crea el worktree y lanza `claude -p "/adw <payload>"` en él. Config en `config/adw.yml` (`label`, `interval_seconds`, `max_rejections`, `max_parallel`, `model`, `prompt`). Detalle en `adws/README.md`.
- **Skill `/adw`** (`.claude/skills/adw/SKILL.md` + `references/`): no tiene flujo escrito; ante cada evento reconstruye el estado de los artefactos (Issue, Environment, Plan, Change, Verification, Documentation) según su ciclo de vida GSM (`references/gsm.md`, `artifacts/`, `services/`, `roles.md`, `control.md`, `adapters/github.md`) y compone el workflow. El estado vive en etiquetas y comentarios de la issue de GitHub (panel del caso editado en su sitio), no en el repo.
- **Roles → subagentes** (`.claude/agents/`): `planner`, `verification-planner`, `implementer`, `documenter`, `checker`, `evidence-collector`, `judge`. `adws/agents_test.go` exige que el catálogo de roles, el ejecutor de Claude Code, `.claude/agents/` y `config/adw.yml` coincidan: si cambias uno, cambia los demás.
- **Verificación**: el plan de verificación se escribe sin ver el plan de implementación; la validación lanza tres verificadores independientes (estructural, funcional, fidelidad) con rúbricas fijas (`conventions.md`). Un rechazo vuelve al plan hasta `max_rejections`.
- Logs por caso en `.adw/logs/<issue>/` y evidencias en `.adw/evidence/` (ignorados por git; los escriben hooks de `.claude/settings.json`).
- Mejoras pendientes del sistema: `ROADMAP.md`.

## CI

`.github/workflows/ci.yaml` (PR y push a `main`): `test` (fmt, vet, migraciones, Go, build, con PostgreSQL), `e2e` (Playwright) y `docker` (build de la imagen; en local es solo informativo).
