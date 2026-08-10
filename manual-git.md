# Manual de git para trabajo en equipo

> Manual práctico y genérico. Vale para cualquier repo del equipo.
> **El porqué de cada regla —qué incidente la obligó, qué se descartó y con qué
> evidencia— está en [`decisiones-git.md`](./decisiones-git.md)**, que es
> append-only. Este documento dice *qué hacer*; ese otro dice *por qué*.
>
> Si una regla de acá te parece burocracia, no la saltees: buscá su entrada en el
> registro. Todas nacieron de algo que ya pasó y costó horas.

---

## 0. Antes de planear: el estado real está en el remoto

**Tu working tree no es el estado del proyecto.** La troncal se mueve mientras
trabajás, y planear contra lo que ves en disco es cómo se escribe dos veces la
misma función.

```bash
git fetch --prune
gh pr list
```

Esto va **antes de planear**, no antes de pushear. Si tus notas y el remoto se
contradicen, **manda el remoto**.

**¿Cuál es la troncal de este repo?** No adivines — no todos usan el mismo
nombre:

```bash
git branch -r          # el remoto lo dice
```

De acá en adelante `<troncal>` = `main` o `master`, según el repo.

---

## 1. Mapa de ramas

```
<troncal> ──●───────●───────●──────────▶   única rama permanente
             \       \       \             siempre desplegable
              feat/a  feat/b  fix/c        cortas · un tema · un dueño
                │       │       │
                └───┬───┴───────┘
                    ▼
       integration/<aaaa-mm-dd>            se crea, se prueba, SE TIRA
                                           nunca se mergea a nada

       release/<aaaammdd>  ◀── de la troncal, antes de un corte
                └── tag v<N>                sólo cherry-pick de fixes
```

| Rama | Sale de | Va a | Muere |
|---|---|---|---|
| `feat\|fix\|chore\|docs/<tema>` | `<troncal>` | PR a `<troncal>` | al mergear |
| `integration/<fecha>` | `<troncal>` + merge de N ramas | **a ningún lado** | el mismo día |
| `release/<fecha>` | `<troncal>` | tag | pasado el corte |

**No hay `develop`.** Una colectora permanente es un segundo lugar donde estar
desactualizado, y salvo que alguien la verifique sola cada día, nadie sabe si
está sana. La colectora de integración de acá **es efímera a propósito**: se
muere sola y por eso nunca miente.

### Empezar una rama

```bash
git fetch --prune
git switch -c feat/tema-corto origin/<troncal>
```

Prefijos: `feat/` `fix/` `chore/` `docs/`. Minúsculas, guiones, sin espacios.
**Un PR = un tema.** Si aparece algo no relacionado, otra rama.

---

## 2. Un worktree por rama

Un worktree es un directorio de trabajo más, del mismo repo, en otra rama. Cambiás
de tarea cambiando de carpeta: no perdés el build, ni el estado del editor, ni
tenés que hacer stash de nada.

### Layout

Los worktrees van **al lado** del clon, nunca adentro:

```
dev/
├── proyecto/                    ← clon principal
├── proyecto-wt-feat-login/
└── proyecto-wt-fix-sync/
```

```bash
git worktree list                                        # SIEMPRE antes de crear
git worktree add ../proyecto-wt-feat-login -b feat/login origin/<troncal>
```

🔴 **Nunca dentro de un directorio temporal del sistema.** Un worktree en `TEMP`
se borra solo, y con él el trabajo sin pushear. Si el repo tiene un script de
siembra (`scripts/seed-worktree.*`), usalo: se niega a crear ahí.

### Cómo se destruye

```bash
git worktree remove ../proyecto-wt-feat-login     # NUNCA rm -rf
git worktree move  <viejo> <nuevo>                # NUNCA mv
git worktree prune --dry-run                      # si borraste uno a mano
```

`rm -rf` deja una referencia fantasma que después confunde a `git worktree list`.

### El costo real: lo que git NO copia

Un worktree nuevo **no trae nada de lo gitignoreado**. Y eso suele ser
justamente lo que hace que el proyecto funcione: `.env`, credenciales locales,
cachés caras de reconstruir, artefactos compilados.

**Ése es el motivo real por el que la gente termina compartiendo un solo checkout
y pisándose.** El worktree limpio es la práctica correcta; sembrarlo a mano es lo
caro. Por eso el repo debería tener un script que lo siembre, y ese script tiene
una regla no negociable:

> 🔴 **Si falta un item requerido, el script falla RUIDOSO.** No lo saltea.
> Un `.env` ausente en silencio no da error: da una app corriendo contra la
> configuración equivocada, con datos que se ven válidos. El silencio de un guard
> es indistinguible de su aprobación.

### 🔴 `node_modules` se instala por worktree. Nunca se comparte

Vas a encontrar mucho consejo en internet para compartir `node_modules` entre
worktrees —symlinks, hard links, un store común— y ahorrar disco.

**En un proyecto con dependencias nativas o wasm, eso es un bug esperando.**
Compartir el árbol reintroduce copias duplicadas de un mismo paquete, cada copia
trae sus propias clases, y el resultado **compila y testea en verde** y revienta
en runtime. `npm ci` en cada worktree. El disco es barato; ese bug no.

---

## 3. Checkout compartido: tres reglas

Si más de una persona (o más de una sesión) toca el mismo directorio, el trabajo
sin commitear de otro es invisible y frágil.

1. **`git diff -- <archivo>` antes de toda escritura completa.** Reescribir un
   archivo que tenía un cambio ajeno sin commitear lo borra sin recuperación:
   nunca estuvo en el índice, así que no hay `reflog` que lo traiga.
2. **Nada de `git add -A` a ciegas.** `git status` primero, archivos puntuales
   después.
3. **Trabajo ajeno sin commitear no se pisa: se avisa.** Si lo necesitás en tu
   rama, que la otra persona lo commitee primero.

La forma de no tener este problema es la sección 2: **un worktree por rama.**

---

## 4. Verificá la rama SOLA

Esto es lo que más caro sale y menos se hace.

**Tus tests no corrieron sobre tu rama.** Corrieron sobre tu *working tree*, que
es tu rama **más** todo lo que hay sin commitear —tuyo y ajeno. Un verde así no
prueba que tu rama funcione: prueba que funciona *tu máquina en este momento*.

El modo de falla es silencioso y llega tarde: el PR se mergea, y la troncal
rompe con un error que en tu máquina nunca existió.

```bash
git worktree add ../proyecto-wt-verify --detach origin/<troncal>
cd ../proyecto-wt-verify
git merge --no-commit --no-ff feat/tu-rama    # así se va a ver mergeada
# sembrar y correr la suite completa acá
git worktree remove ../proyecto-wt-verify
```

**Si no podés nombrar dónde corriste los tests, no los corriste.**

---

## 5. Repartir el trabajo antes de empezar

Dos personas escribiendo la misma función en paralelo es la falla más cara del
trabajo concurrente: nadie se entera hasta el merge, y ahí hay que tirar trabajo
terminado y funcionando.

Antes de abrir el editor:

```bash
git fetch --prune && gh pr list && git branch -r
```

Y decilo: **qué módulo vas a tocar**. El archivo `.github/CODEOWNERS` es el mapa
de dueños del repo — mirá quién figura en el área que vas a tocar antes de
empezar. **Un módulo, un dueño por vez.**

---

## 6. Mergear

### Si el repo tiene CI

El check verde es la garantía. Miralo **antes** de mergear, y confirmá que corrió
sobre el merge, no sólo sobre tu rama.

### Si el repo NO tiene CI

🔴 **`MERGEABLE` / `CLEAN` de GitHub significa solamente "sin conflicto de
texto".** Nadie verificó que el resultado compile ni que los tests pasen. Probalo
a mano, en este orden:

```bash
git fetch --prune
git merge --no-commit --no-ff origin/<troncal>   # el merge, sin cerrarlo
npm test && npm run typecheck                    # los tests del resultado
git merge --abort                                # deshacer
```

Recién ahí mergeás desde la web. Y **después del merge, re-verificá la troncal**:
lo que probaste fue una simulación local, no lo que quedó publicado.

### Método de merge

| Situación | Método |
|---|---|
| Nada apila encima y el historial de la rama es ruido | squash |
| **Cualquier rama apila encima** | **merge commit** |
| El detalle de los commits es la traza del proyecto | **merge commit** |

🔴 **El squash reescribe los SHAs.** Si otra rama apilaba sobre los tuyos, esa
rama queda **muerta**: su contenido está en la troncal pero sus commits no
existen, y `git log` la muestra "por delante" mientras en realidad no tiene a
dónde volver. Ante la duda, **merge commit**.

---

## 7. PRs apilados

Una rama que sale de otra rama en vez de la troncal. Sirve para partir un cambio
grande en revisiones digeribles, y tiene tres reglas.

**1. Declaralo en el cuerpo del PR.** Qué rama es la base y con qué método se
mergea. Quien mergea no puede adivinarlo.

**2. Mergeá de abajo hacia arriba, con merge commit.** Primero el padre; después
actualizás el hijo y lo mergeás.

**3. Para propagar un rebase por toda la pila, `--update-refs`:**

```bash
git rebase --update-refs origin/<troncal>
```

Sin ese flag tenés que rebasar cada rama de la pila a mano contra la anterior, y
un error en el medio deja SHAs huérfanos. Con el flag, git reapunta todas las
ramas de la pila en una sola pasada.

### Detectar una rama que el squash dejó muerta

Síntoma: tu rama figura "adelante" de la troncal, pero su contenido **ya está**
en la troncal.

```bash
git log --oneline origin/<troncal>..tu-rama    # commits que "faltan"
git diff origin/<troncal>..tu-rama             # ¿y el contenido? vacío o ajeno
```

Contenido vacío + commits por delante = **rama muerta**. No sigas commiteando
ahí: sacá la troncal fresca y volvé a ramificar.

---

## 8. Colectora de integración

**Cuándo**: hay N PRs abiertos que se van a mergear juntos y necesitás probarlos
*en conjunto* antes de mergear ninguno — típicamente antes de una demo o un corte.

```bash
git fetch --prune
git switch -c integration/$(date +%F) origin/<troncal>
git merge --no-ff origin/feat/a origin/feat/b origin/fix/c
# sembrar, correr todo, anotar qué rompió y contra quién
```

**La regla que la hace segura: no se mergea a ninguna parte.** Es un banco de
pruebas, no un paso del camino. Los arreglos que aparezcan van **a la rama que
los causó**, no a la colectora. Cuando terminaste:

```bash
git switch <troncal> && git branch -D integration/<fecha>
```

Si te dan ganas de mergear la colectora a la troncal, lo que querías era mergear
los PRs. Mergealos.

---

## 9. Release y tag

Para un corte con fecha (demo, entrega, versión):

```bash
git switch -c release/$(date +%Y%m%d) origin/<troncal>
git push -u origin release/$(date +%Y%m%d)
```

A partir de ahí, la troncal sigue moviéndose y **la release no**. Sólo entran
fixes críticos, por cherry-pick, uno por uno:

```bash
git cherry-pick <sha>       # el fix ya mergeado en la troncal
```

El fix se arregla **primero en la troncal** y se trae; nunca al revés, o queda
sólo en la release y vuelve a aparecer en el próximo corte.

```bash
git tag -a v1 -m "corte del <fecha>" && git push origin v1
```

---

## 10. Cerrar el loop

```bash
git switch <troncal> && git pull
git worktree remove ../proyecto-wt-feat-login
git branch -d feat/login
git push origin --delete feat/login          # si el repo no lo borra solo
```

Y una que se olvida siempre: **no dejes un checkout parado en una rama mergeada
o muerta.** La próxima sesión arranca ahí, cree que está en la troncal, y trabaja
sobre una base que ya no existe.

---

## 11. Automatización: qué hace cada pieza

| Pieza | Qué resuelve |
|---|---|
| **`.github/CODEOWNERS`** | Pide review **automáticamente** al marcar el PR *ready for review*. También es el mapa de dueños de la sección 5. |
| **Template de PR** | Obliga a declarar la base apilada, el método de merge y **cómo se verificó**. |
| **CI (GitHub Actions)** | Convierte "verde" en algo que no depende de la palabra de una persona. |
| **GitHub Projects** | Un tablero con el estado de cada PR. Se puebla solo con el workflow *Auto-add to project*. |
| **App oficial de GitHub para Slack** | DM directo al reviewer cuando le piden review: `/github subscribe <owner>/<repo> pulls reviews` |

🔴 **Dos detalles que hacen la diferencia entre que funcione y que sea
decorativo:**

- **A un code owner no se le pide review de su propio PR.** Si una sola persona
  es dueña de todo el repo, CODEOWNERS **no dispara nunca**. La titularidad tiene
  que estar repartida.
- **CODEOWNERS no se dispara en draft.** Se dispara al marcar *ready for review*.
  Eso es una feature: abrí en draft mientras trabajás, marcá ready cuando querés
  ojos encima.

**Por qué no hay un integrador externo (Zapier y parientes).** Todo lo de arriba
es nativo y gratis. Un integrador agregaría un tercer sistema con un token del
repo para duplicar lo que ya funciona: un punto de falla más y un secreto más
para manejar mal. Tendría sentido sólo para disparar algo que *no* es de GitHub.

---

## 12. Guardrails del remoto

Un documento no impide nada. Estas tres opciones sí, y se aplican una vez:

```bash
# el squash queda apagado (mata las ramas apiladas) y las ramas se borran solas
gh api -X PATCH repos/<owner>/<repo> \
  -F allow_squash_merge=false -F delete_branch_on_merge=true

# la troncal deja de aceptar push directo
gh api -X PUT repos/<owner>/<repo>/branches/<troncal>/protection --input - <<'JSON'
{ "required_status_checks": { "strict": true, "contexts": ["ci"] },
  "enforce_admins": false,
  "required_pull_request_reviews": { "required_approving_review_count": 1 },
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false }
JSON
```

Verificalo, no lo asumas — sin protección la API contesta `404`:

```bash
gh api repos/<owner>/<repo>/branches/<troncal>/protection
```

⚠️ `required_status_checks` sólo se puede fijar **después** de que el CI corrió
al menos una vez y GitHub conoce el nombre del check. Son dos pasos.

⚠️ `enforce_admins: false` es deliberado: deja al admin destrabar una emergencia.
Un guard que bloquea el flujo legítimo más frecuente **enseña a saltearlo**, y un
bypass entrenado es peor que no tener guard.

**Lo que ningún guard puede impedir**, y por eso vive en este documento: pisar
trabajo ajeno sin commitear (§3), medir verde sobre el working tree (§4), y
escribir dos veces la misma función (§5).

---

## 13. Emergencias

```bash
git status                       # primero esto, siempre
git diff                         # qué cambió sin commitear
git log --oneline -10
git reset --soft HEAD~1          # deshacer el último commit, cambios en staging
git restore <archivo>            # descartar cambios de un archivo puntual
git stash push -u -m "motivo"    # guardar todo rápido (-u incluye untracked)
git reflog                       # dónde estuvo HEAD: casi todo se recupera acá
```

**La troncal local quedó atrás.** No la mergees, reapuntala:

```bash
git switch <troncal> && git fetch --prune && git reset --hard origin/<troncal>
```

(Sólo si no tenés commits propios ahí — que no deberías: nunca se commitea en la
troncal.)

**Worktree fantasma** (`git worktree list` muestra uno que ya no existe):

```bash
git worktree prune --dry-run && git worktree prune
```

**Error de `index.lock`** sin ningún git corriendo de verdad (verificalo en el
administrador de tareas antes):

```bash
rm -f .git/index.lock .git/HEAD.lock
```

**Lo que casi nunca se recupera**: un archivo sin commitear que fue sobrescrito.
Nunca estuvo en el índice ⇒ no está en `reflog` ni en `fsck`. Por eso §3 es la
regla y no una sugerencia.
