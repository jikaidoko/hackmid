# Amparo — flujo de git del equipo

> Repo: `origin` → `jikaidoko/hackmid` · rama troncal: **`master`** (no `main`).
> Válido para este repo Y para cualquier otro del equipo (mismo patrón que ya
> usa `hofi-protocol-cardano`/`hofi-passport`).

## Regla de oro

**Nunca `git push` directo a `master`.** Todo cambio entra por rama dedicada +
Pull Request. Sin excepciones, ni siquiera "es un solo archivo".

## 1. Empezar una feature nueva

```bash
git checkout master
git pull origin master
git checkout -b feat/nombre-corto
```

Prefijo según el tipo de cambio: `feat/` `fix/` `chore/` `docs/`.

## 2. Traer trabajo que quedó sin commitear (stash)

Si tenías cambios sueltos en otra rama (o en `master`) y querés moverlos a la
rama nueva:

```bash
git stash push -u -m "feat/nombre-corto"   # -u incluye archivos nuevos (untracked)
git checkout master
git pull origin master
git checkout -b feat/nombre-corto
git stash pop
```

`git stash list` para ver qué quedó guardado; `git stash drop` si algo sobra.

## 3. Commitear

```bash
git add <archivos puntuales>     # evitar "git add -A" a ciegas: revisar qué entra
git status                        # confirmar qué se va a commitear
git commit -m "feat: descripción corta en imperativo"
```

Prefijo de commit igual al de la rama (`feat:` `fix:` `chore:` `docs:`).

## 4. Subir y abrir PR

```bash
git push -u origin feat/nombre-corto
```

Después: abrir el PR en GitHub contra `master`, no mergear sin al menos una
revisión (aunque sea del compañero de al lado). Mergear desde la web (Squash
and merge) — no `git push` directo a `master` ni localmente ni por acá.

## 5. Cerrar el loop

```bash
git checkout master
git pull origin master
git branch -d feat/nombre-corto              # borra la rama local ya mergeada
git push origin --delete feat/nombre-corto   # borra la rama remota (opcional, GitHub lo ofrece solo)
```

## Convenciones

- **Nombre de rama**: `tipo/tema-en-guiones`, sin mayúsculas, sin espacios.
  Ej: `feat/registerFiling-circuit`, `fix/proof-server-port`.
- **Mensaje de commit**: imperativo, corto, prefijo de tipo. Si hace falta más
  contexto, cuerpo del commit separado por línea en blanco — no todo en el
  título.
- **Un PR = un tema.** Si mientras trabajás aparece algo no relacionado, otra
  rama.

## Comandos de emergencia

```bash
git status                        # primero esto, siempre, antes de cualquier otro comando
git diff                          # ver qué cambió sin commitear
git log --oneline -10             # últimos commits
git reset --soft HEAD~1           # deshacer el último commit, dejando los cambios en staging
git checkout -- <archivo>         # descartar cambios sin commitear de un archivo puntual
git stash                         # guardar todo lo sucio rápido si hay que cambiar de rama YA
```

Si `git status`/`git commit` tiran un error de lock (`index.lock` o similar)
y no hay ningún otro `git` corriendo de verdad (revisar con el administrador
de tareas / `ps aux | grep git`), borrar el archivo de lock a mano:

```bash
rm -f .git/index.lock .git/HEAD.lock
```

y reintentar el comando.
