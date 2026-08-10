# Registro de decisiones — flujo de git

> El **por qué** de cada regla de [`manual-git.md`](./manual-git.md). El manual
> dice qué hacer; acá está qué pasó que lo obligó.
>
> **Cómo se usa este documento.** Cuando una regla del manual parezca burocracia,
> buscá su entrada acá antes de saltearla. Ninguna se escribió por gusto: cada
> una costó horas de trabajo tirado o de diagnóstico apuntando al lugar
> equivocado.
>
> **Regla del registro: es append-only.** Una decisión nueva **no reemplaza** a la
> anterior — la supersede citándola por número, y la vieja queda con su fecha y su
> evidencia. Un registro que se edita para quedar coherente deja de ser un
> registro: no se puede distinguir *cambiamos de opinión con motivo* de *siempre
> pensamos esto*.
>
> **Formato de cada entrada**: qué se decidió · qué lo obligó (medido, no
> recordado) · qué se descartó y por qué · a qué sección del manual corresponde.
>
> Los incidentes se describen **sin atribuirlos a personas**. En todos los casos
> la causa fue estructural: la herramienta permitía el error, o el flujo no daba
> forma de detectarlo. Un flujo que depende de que nadie se equivoque no es un
> flujo.

---

## 1 · Colectora de integración efímera, no una rama `develop` permanente

**Decidido** el 10-ago-2026. → manual §1, §8

**Qué se decidió.** La troncal es la única rama permanente. Cuando hace falta
probar varios PRs juntos se crea `integration/<fecha>`, que **no se mergea a
ningún lado** y se borra el mismo día. Para un corte con fecha,
`release/<aaaammdd>` con cherry-pick.

**Qué lo obligó.** Llegó a haber **cinco PRs abiertos simultáneos** de tres
personas, que iban a convivir en una demo y que nadie había probado juntos. No
existía ningún lugar donde hacerlo sin mergear a la troncal primero.

**Qué se descartó.** Una rama `develop` permanente al estilo GitFlow — que es lo
que se pidió originalmente. Se descartó por una razón medible: **el repo no tenía
CI**. Sin CI, nadie verifica que `develop` esté sana, así que sería un segundo
lugar donde estar desactualizado y una segunda fuente de "yo lo probé y andaba".
La colectora efímera no tiene ese problema **porque se muere sola**: no puede
quedar vieja.

También se descartó trunk-based estricto sin colectoras: es lo que la industria
recomienda para equipos chicos, pero no cubre el caso real de arriba.

---

## 2 · Merge commit por defecto; squash sólo si nada apila encima

**Decidido** el 10-ago-2026. → manual §6, §7

**Qué se decidió.** El squash queda **apagado en el remoto**
(`allow_squash_merge=false`). Los PRs apilados declaran su base en el cuerpo.

**Qué lo obligó.** Un PR apilado se mergeó con squash. El squash reescribe los
SHAs, así que la rama hija quedó **muerta**: su contenido estaba en la troncal
pero sus commits no existían. El síntoma es traicionero — `git log` la muestra
"por delante" de la troncal, así que parece que tiene trabajo pendiente cuando en
realidad no tiene a dónde volver. Se siguió commiteando en ella un rato antes de
que alguien se diera cuenta.

**Qué se descartó.** Dejar el squash habilitado y confiar en la disciplina de
elegir bien en cada merge. Se descartó porque el botón correcto y el botón que
rompe están **uno al lado del otro**, la consecuencia aparece días después, y el
que mergea no siempre es el que sabe si hay algo apilado.

**Efecto lateral buscado.** El detalle de los commits se conserva. En este
proyecto la traza técnica —qué se midió, qué mutante murió— vive en los mensajes
de commit; el squash la borraba.

### 2.b · Borrar la rama del padre es parte de mergear la pila

**Agregado el 10-ago-2026, después de tropezarlo mergeando estos mismos
documentos.** No supersede a la entrada 2: es un segundo modo de falla, distinto.

**Qué pasó.** Se mergeó el padre con `gh pr merge <n> --merge`, sin
`--delete-branch`, siguiendo una instrucción escrita en este mismo trabajo. La
rama del padre quedó viva. GitHub **re-apunta un PR hijo a la base del padre sólo
cuando esa rama se borra al mergear**, así que el hijo siguió apuntándole — y al
mergearlo, su contenido entró **en la rama del padre, que ya estaba mergeada**.

**Lo caro es el diagnóstico, no el error.** El PR hijo quedó en `MERGED`, con su
merge commit y su tilde. Todo indicaba éxito. La troncal simplemente no tenía los
archivos. Medido: `git log --oneline origin/master..origin/<rama-del-padre>`
devolvió **6 commits** que nadie sabía que estaban afuera.

**Qué se descartó.** Arreglarlo con un push directo a la troncal, que era un solo
comando. Se descartó porque la regla no tiene excepción "es para arreglar algo":
un bypass justificado una vez es el que después se cita. Se arregló con otro PR.

**Familia.** Es la misma de siempre en este registro: **un estado de éxito que no
mide lo que uno cree que mide.** `MERGED` significa "este PR se cerró mergeando",
no "esto llegó a la troncal" — igual que `SUCCESS` con `continue-on-error` no
significa que los pasos pasaron, y que `MERGEABLE` no significa que compile.

**🔴 Cola de esta entrada: la primera verificación que se escribió no corría.**
Decía `git log origin/<troncal>..origin/<rama-del-padre>` — y la regla que la
acompaña te manda a **borrar esa rama** al mergear. Al ejecutarla en serio devolvió
`fatal: ambiguous argument ... unknown revision`.

La corrección es anotar el **SHA** del hijo antes de mergear y preguntar después
`git merge-base --is-ancestor "$SHA" origin/<troncal>`: un SHA sigue siendo
alcanzable cuando la rama que lo apuntaba ya no existe. Y la lección, que es más
grande que el comando:

> **Una verificación no puede depender de un estado que el paso anterior
> destruyó.** Al escribir un chequeo, preguntate contra qué lo vas a medir
> **cuando el trabajo ya esté hecho**, no cuando todavía está a medias.

Con una segunda, del mismo día y del mismo tipo: el primer intento de probar la
receta nueva usó como caso negativo un `HEAD` que estaba parado justo en la
troncal, así que **dio verde por construcción**. Un test que no puede dar rojo no
prueba nada — es el mismo agujero que este registro le reprocha a los mocks y a
los guards, cometido sobre sí mismo.

---

## 3 · Un worktree por rama, y un script que lo siembre

**Decidido** el 10-ago-2026. → manual §2

**Qué se decidió.** Cada rama en su propio worktree, hermano del clon, con
nombre `<proyecto>-wt-<slug>`. Nunca dentro de un directorio temporal del
sistema. El repo trae `scripts/seed-worktree.ps1` para poblarlo.

**Qué lo obligó.** Dos cosas a la vez.

Primero: **tres worktrees vivían en `AppData\Local\Temp\...\scratchpad`**, con
ramas reales y commits sin pushear. Ese directorio se limpia solo.

Segundo, y es la causa de fondo: **el `.gitignore` del repo define nueve clases
de estado por-checkout** —configuración, credenciales, cachés y artefactos
compilados— y un worktree nuevo no trae **nada** de eso. Una de esas piezas
cuesta **2 a 3 horas** de reconstruir. O sea: el worktree limpio era la práctica
correcta *y* la cara, así que todos terminaban trabajando en el mismo checkout y
pisándose (ver #5).

**Qué se descartó.** Documentar la siembra como una lista de pasos manuales. Se
descartó porque el ítem más importante de la lista —el archivo de configuración—
**falla mudo si falta**: la aplicación no da error, arranca contra la
configuración equivocada y devuelve datos que se ven válidos. Una lista de pasos
que se puede saltear a medias, en un ítem cuyo olvido no produce error, no es un
control.

**De ahí sale la regla dura del script**: si falta un item requerido, **falla
ruidoso**. El silencio de un guard es indistinguible de su aprobación.

### 3.b · El eje no es "hace falta", es "cuánto cuesta recuperarlo"

**Corregido el 10-ago-2026, después de una revisión externa. Supersede el
criterio de la entrada 3.**

La primera versión del script trataba los nueve items como una lista plana con
un flag binario `Required` sí/no. **Ése era el eje equivocado**: casi todo hace
falta, así que la pregunta no discrimina. Dos consecuencias, las dos medidas:

1. **Copiaba 36 MB al pedo** — `src/managed` (18 MB) y `public/zk` (18 MB), que
   un build reproduce idénticos en segundos.
2. **Peor: tres items irrecuperables estaban marcados como opcionales** —
   `subjects.*.json`, `deployment.*.json` y `midnight-level-db/`. Si faltaban, el
   script seguía y los mandaba a un renglón de *skips*. Y `subjects.*.json` es lo
   único que puede reconstruir los nullifiers de una credencial: la cadena no
   liga una denuncia a nadie, a propósito.

El eje correcto es **el costo de recuperación**, y da tres grupos con tres
comportamientos:

| | Qué | Comportamiento |
|---|---|---|
| 🔴 Irrecuperable | `subjects.*.json`, `deployment.*.json`, `.env*`, `midnight-level-db/` | copiar; copia fallida = fatal; ausencia = **aviso fuerte** |
| 🟠 Caro pero automático | `.wallet-state/` (2-3 h), `.zk-params/` (33 MB), `node_modules/` | copiar si está; si no, nombrar comando **y precio** |
| 🟢 Regenerable | `src/managed/`, `public/zk/`, `dist/` | **no copiar**, sólo decir el comando |

**El grupo rojo es el único que justifica el script.** Con eso, el worktree
limpio deja de ser la opción cara — que era el problema entero de la entrada 3.

Un matiz que la clasificación deja abierto y conviene decidir explícitamente:
dentro del rojo, **copiar y linkear no son equivalentes**. Copiar
`subjects.*.json` a N worktrees permite que dos diverjan y que la copia que
pierde desaparezca; linkear mantiene una sola. Pero un LevelDB linkeado no
soporta dos procesos a la vez. Hoy el script **copia**; la alternativa está
registrada acá para que no se redescubra.

### 3.c · El `.env` fantasma: dos fallbacks silenciosos encadenados

**Medido el 10-ago-2026**, y el mecanismo es peor de lo que decía la entrada 3.

Había **dos `.env` byte-idénticos**, 404 bytes cada uno, con la misma fecha al
segundo: uno en la raíz del repo y otro en `contracts/`. El de la raíz **no lo
lee nadie**. Pero la falla no es muda una vez, es muda **dos**:

```ts
try { process.loadEnvFile(resolve(PKG_ROOT, '.env')); }
catch { /* sólo un comentario */ }
...
const networkId = env.MN_NETWORK ?? 'undeployed';
```

El `catch` vacío se traga que el archivo no exista; el `??` elige la red local.
Cada uno por separado es defendible. **Encadenados convierten "falta un archivo"
en "estás corriendo contra otra red" sin una sola línea de aviso** — y como las
dos redes comparten formato de dirección, la equivocada responde con datos que
se ven válidos.

**Patrón para buscar esta familia en cualquier código**: un `catch` vacío y,
aguas abajo, un `?? default`. No busques uno solo; buscá el par.

---

## 4 · `node_modules` nunca se comparte entre worktrees

**Decidido** el 10-ago-2026. → manual §2

**Qué se decidió.** `npm ci` en cada worktree. Nada de symlinks, hard links ni
stores compartidos.

**Qué lo obligó.** El proyecto tiene dependencias wasm, y **dos copias del mismo
paquete wasm rompen la aplicación con todo en verde**: cada copia trae sus
propias clases, y la falla aparece en runtime, no en el typecheck ni en los
tests. El equipo ya había pagado ese bug una vez y lo cerró con `overrides`; el
hoisting de un árbol compartido es exactamente cómo vuelve.

**Qué se descartó.** El consejo estándar de las guías de worktrees, que
recomienda unánimemente compartir el árbol de dependencias para ahorrar disco
(pnpm con store común, hard links, symlinks). **Es buen consejo en general y es
malo acá.** Vale la pena registrarlo justamente por eso: la próxima persona que
lea una guía de worktrees va a encontrar la recomendación contraria, bien
argumentada, y necesita saber por qué acá no aplica.

---

## 5 · Verificar la rama sola, en un worktree limpio

**Decidido** el 10-ago-2026. → manual §4

**Qué se decidió.** Antes de declarar una rama lista: worktree limpio desde la
troncal, merge de la rama, y ahí correr la suite.

**Qué lo obligó.** Una rama midió **verde apoyándose en un archivo sin commitear
de otra persona**. Todo lo que se reportó —módulos que transforman, build,
página cargando— era cierto *en ese working tree*, que era la rama **más** lo
ajeno. La rama sola no cargaba. Se detectó por casualidad al revisar qué había
entrado realmente en el commit.

**El costo real** no es el bug: es que el reporte de verificación era detallado,
específico y falso, así que consumió la confianza que hace que un reporte sirva.

**Qué se descartó.** Confiar en `git status` antes de medir. Se descartó porque
en un checkout compartido `git status` muestra decenas de archivos ajenos
modificados todo el tiempo: la señal está ahí y no se ve. La única verificación
que distingue es **correr en un árbol que sólo tiene tu rama**.

---

## 6 · `CODEOWNERS` repartido entre varias personas

**Decidido** el 10-ago-2026. → manual §5, §11

**Qué se decidió.** `.github/CODEOWNERS` con la titularidad **repartida por
área** entre los colaboradores. Cumple dos funciones: pedir review automático, y
ser el mapa de quién es dueño de qué.

**Qué lo obligó.** Una misma función de derivación de estado se escribió **dos
veces en paralelo**, por dos personas, el mismo día, con tres nombres idénticos
que colisionaban. Ninguna de las dos era mala; había que tirar una. Semanas
después volvió a pasar con cuatro archivos de configuración de build.

En los dos casos el trabajo duplicado era **descubrible antes de empezar** —
estaba en el remoto o anunciado en un PR abierto— y nadie miró.

**Qué se descartó.** Un acuerdo verbal de avisar qué se toca. Ya existía y no
alcanzó: depende de acordarse en el momento exacto en que uno se sienta a
trabajar, que es cuando menos ganas hay de coordinar. `CODEOWNERS` es el mismo
acuerdo, escrito, versionado y con efecto automático.

🔴 **Detalle que lo vuelve decorativo si se ignora**: a un code owner **no se le
pide review de su propio PR**. Si una sola persona es dueña de todo, el archivo
no dispara nunca. Por eso la titularidad va repartida, no concentrada.

---

## 7 · Automatización nativa de GitHub, sin integrador externo

**Decidido** el 10-ago-2026. → manual §11

**Qué se decidió.** GitHub Projects (workflow *Auto-add*), CODEOWNERS y la app
oficial de GitHub para Slack. Nada de Zapier ni equivalentes.

**Qué lo obligó.** Se pidió automatizar: "que un PR listo dispare el review de
otro colaborador y avise por Slack". Al medir el camino nativo, resultó que
**GitHub ya hace exactamente eso**:

| Lo pedido | Pieza nativa |
|---|---|
| un PR listo dispara el review | CODEOWNERS pide review al marcar *ready for review*, y **no** en draft |
| avisar a la persona | la app oficial de GitHub manda **DM directo** al reviewer |
| cada PR inscripto en el tablero | workflow built-in *Auto-add to project* |

**Qué se descartó.** Zapier. Habría agregado un tercer sistema, con **un token
del repositorio guardado fuera del repositorio**, para duplicar lo que las tres
piezas de arriba ya hacen. Un punto de falla más y un secreto más que administrar
— y este equipo ya se quemó con dos secretos de la misma forma conviviendo en un
mismo archivo.

**Cuándo habría que revisar esta decisión**: si el disparador tuviera que salir
de GitHub hacia un sistema que no es GitHub. Ahí un integrador se justifica.

**🔴 Una afirmación que hubo que degradar.** En la primera versión de esta
entrada decía, como hecho, que en cuenta gratuita el auto-add de Projects admite
"un workflow y un repositorio". Al ir a la fuente, **eso sale de un reporte de
usuario en el foro de la comunidad, cerrado sin respuesta de GitHub**. La
documentación no lo dice, y lo que sí documenta son límites de *items por
proyecto*, no de cantidad de proyectos.

Queda como **incierto declarado**, no como número: en el manual va con la
instrucción de medirlo en la cuenta propia. Registrar la degradación importa
tanto como el dato — un número sin fuente, escrito con confianza, después se cita
como si estuviera verificado.

**Familia de esto**: es el mismo error que este registro le reprocha al resto del
flujo — afirmar sin medir. Vale igual cuando lo comete el documento.

---

## 8 · CI con GitHub Actions

**Decidido** el 10-ago-2026. → manual §6, §12

**Qué se decidió.** Un workflow de CI que corre en cada PR.

**Qué lo obligó.** El repositorio **no tenía ningún check**
(`statusCheckRollup: 0`). O sea que el `MERGEABLE / CLEAN` que muestra GitHub
significaba **solamente "sin conflicto de texto"**: nadie verificaba que el
resultado compilara ni que los tests pasaran. Cada merge dependía de que alguien
hiciera a mano un `merge --no-commit` + tests + `--abort`, y de que dijera la
verdad sobre el resultado.

Es la pieza que sostiene a todas las demás: sin ella, "verde" es la palabra de
una persona sobre un árbol que nadie más vio (ver #5).

**Qué se descartó.** Nada, y ése es el punto: **Actions es gratis e ilimitado en
repositorios públicos** y ya estaba habilitado. El costo de no tenerlo era
íntegramente de oportunidad.

**🔴 Un supuesto que se cayó al medirlo.** El plan daba por hecho que había una
"etapa barata" —typecheck y build— que corría sin el compilador de contratos.
**Es falso.** Medido escondiendo el artefacto compilado y corriendo las dos
cosas: el build del frontend y el typecheck de contratos fallan los dos con
`TS2307: Cannot find module './managed/...'`. La nota previa de que "el frontend
typechequea sin el contrato compilado" vale para dos módulos aislados, no para
el build completo.

Lo que **sí** corre sin compilador es la instalación de los dos árboles de
dependencias y el guard de wasm — que no es poco: es el control de la clase de
bug más cara del proyecto, y hasta ahora sólo corría cuando alguien se acordaba.

**Cómo quedó, y por qué en dos jobs.** El job `ci` instala los dos árboles y
corre el guard de wasm. El job `contracts` instala el compilador con el
instalador oficial, pinea la versión, compila con `--skip-zk` y corre typecheck y
suite completa.

`contracts` **entró sin gatear** (`continue-on-error: true`), porque su paso de
toolchain nunca había corrido en un runner y **un gate que nadie vio pasar no es
un gate**. Pasó los nueve pasos en su primera corrida —**64 tests, 64 pass, 0
fail**, en 36 segundos— así que se le sacó el `continue-on-error` y ahora
bloquea. La secuencia importa más que el resultado: se prometió lo medido, se
midió, y recién ahí se prometió más.

🔴 **Y hay una trampa en el medio que casi la deja pasar.** Con
`continue-on-error: true` el job **reporta `SUCCESS` aunque sus pasos fallen**.
Mirar el estado del job no probaba nada; hubo que pedir las conclusiones **paso
por paso** y el conteo de tests del log. Es la misma familia que el `EXIT=0` de
un pipe que termina en `tail`: **el exit que ves no siempre es el del comando que
te importa.**

---

## 9 · `enforce_admins: false` en la protección de la troncal

**Decidido** el 10-ago-2026. → manual §12

**Qué se decidió.** La troncal queda protegida (sin push directo, sin
force-push, PR obligatorio), **pero el admin puede destrabar**.

**Qué lo obligó.** El repositorio no tenía protección alguna: la API contestaba
`404 Branch not protected`. "Nunca pushees a la troncal" era un acuerdo verbal.
Al mismo tiempo, buena parte del trabajo ocurre en franjas donde no hay un
segundo par de ojos despierto.

**Qué se descartó.** `enforce_admins: true`. Un guard que bloquea el flujo
legítimo más frecuente **entrena el bypass**: la primera vez que la regla impide
un arreglo urgente, se desactiva "por hoy" y no se vuelve a activar. Un guard
apagado protege menos que uno con una excepción declarada.

**Por qué está escrito acá y no escondido en la configuración.** Una excepción
que nadie documentó se lee, meses después, como un descuido — y el siguiente que
la encuentre va a "arreglarla" sin saber qué compraba.

---

## Pendientes registrados (no decididos todavía)

- **Reescribir el historial de la troncal** para sacar commits con atribución
  automática que contradicen la convención del proyecto. Es reescritura de
  historia en un repositorio público con varios colaboradores: necesita una
  ventana coordinada, no una decisión individual.
- **Tests del contrato en CI** — depende de si el compilador se puede instalar en
  el runner (ver #8).
- **Escalar el auto-add de Projects a más de un repositorio** — bloqueado por el
  límite de la cuenta gratuita (ver #7).
