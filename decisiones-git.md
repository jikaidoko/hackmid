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

**Límite conocido**: en cuenta gratuita, el auto-add de Projects admite **un
workflow y un repositorio**. Alcanza para un repo; no escala a varios.

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
