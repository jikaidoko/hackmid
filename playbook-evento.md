# Playbook del evento — Amparo · Midnight Hackathon Buenos Aires

> **Qué es esto:** el documento único de trabajo para las 24hs, con horarios y quién
> hace qué. Reúne y **supersede** a `hofi-protocol-cardano/docs/20-plan-mvp-amparo-
> hackathon-midnight.md` (que queda como registro histórico de cómo llegamos hasta
> acá) y a `amparo-dry-run-playbook.md` (cuyos gotchas técnicos están consolidados
> en §8 de este doc). Complementa, sin duplicar, a `runbook-despliegue.md` (el
> detalle operativo de wallet/redes vive ahí — acá solo se referencia).
> **Equipo:** 4 personas. **Track:** Open. **Evento:** 7-8 ago 2026, Palermo.
> **Caso de uso:** #5 — Compliance programable e identidad reutilizable.

---

## 1. Objetivo

Construir en 24hs netas de hacking un MVP autocontenido en Midnight que muestra el
flujo completo: una organización **admite un caso** en su padrón (la primitiva de
identidad — inspirada en `hofi-passport`), la denunciante **registra denuncias**
contra ese caso admitido, y prueba criptográficamente **"tengo ≥N denuncias
previas"** ante un verificador — sin revelar identidad, fechas ni contenido. El
ledger público muestra solo agregados sin personas.

**Tagline:** *Amparo — protección que no expone.*

## 2. Destinatarios

- **Inmediato:** el jurado (rúbrica §5) — premio + Midnight Build Club.
- **Real:** la vertical de continuidad verificable para canales de denuncia con
  cumplimiento auto-aseverado — judicial, laboral, público o privado (§10.6).
- **Usuarios en la ficción de la demo:** denunciante (Marta), verificador
  (juzgado/RRHH/compliance), sociedad (ledger público).

## 3. La escena de demo (guion) — PIVOT A CASO AMBIENTAL (decisión en vivo, día del evento)

> **Sofía**, vecina de un barrio junto al **Arroyo Los Sauces** (ficticio),
> reporta el vertido de residuos industriales de **"Textil Norte S.A."**
> (ficticia) — un sitio ya **admitido** como caso por el organismo de control
> ambiental. Con su identidad custodial —enrolada con su voz, simulada en
> UI— su historial de denuncias crece en privado: es su **tercer reporte**
> (de distintos casos a lo largo del tiempo), y puede probar **"≥3 reportes
> previos: VERDADERO"** sin revelar cuáles, cuándo, ni quién es —una
> credencial portátil de denunciante confiable. Al mismo tiempo, su reporte
> es el que hace que el **contador público de ESTE caso** cruce el umbral:
> el ledger marca el sitio como **"EN REVISIÓN"** — no por lo que ella
> reveló de sí misma, sino porque suficientes reportes independientes
> coincidieron en el mismo caso.

**⚠️ Encuadre elegido en vivo, el día del evento:** se abandona el ángulo de
violencia de género/policial y se pivotea a delito ambiental. Motivos que se
sostienen incluso mejor que el encuadre anterior: (a) un ilícito ambiental es
más neutro políticamente que violencia policial — baja el riesgo de la regla
de "agenda política" (doc 20 §12.2), (b) encaja sin fricción en la slide de
posicionamiento ampliado que ya estaba escrita (§10.6) — cumplimiento
ambiental es una categoría más de organización que hoy se auto-certifica.

**La síntesis A+B (decisión #14, §6):** el caso ambiental permitió construir
LAS DOS variantes del mecanismo en el mismo flujo, sin duplicar circuitos —
ver §4 para el diseño técnico completo. Esto es posible porque, a diferencia
de un presunto agresor humano (doc 19 §5b, donde el conteo debe quedar
oculto por presunción de inocencia), acá el "acusado" es un **sitio/incidente
admitido**, no una persona — y doc 19 mismo autoriza transparencia radical
sobre instituciones ("ranking hacia arriba, privacidad hacia abajo").

**Nombres ficticios — NO cambiar por nombres reales.** Ningún caso, río o
empresa real (el Riachuelo, por ejemplo, tiene causa judicial real vigente —
evitar cualquier parecido).

**Tres vistas de frontend (Google Stitch, no V0 — ver §9 cambio de herramienta):**
1. **Denunciante** (Sofía) — enrolamiento por voz (simulado) → identidad
   custodial; registra reportes contra casos admitidos; ve su contador
   privado propio crecer (alimenta A).
2. **Verificador** (organismo ambiental / auditor de compliance — lenguaje
   deliberadamente genérico, ver §10.6) — solicita/recibe la prueba de
   umbral de Sofía; ve SOLO el booleano.
3. **Ledger público en vivo** — muestra el contador POR CASO (a diferencia
   del diseño anterior, acá es intencionalmente público — es institucional,
   no personal) y el momento en que el caso pasa a "EN REVISIÓN" al cruzar
   el umbral. El golpe visual del dual-ledger, ahora con un cambio de estado
   visible en vivo.

Bocetos de referencia (paleta/layout, no código): `amparo-prep/mockups/tres-vistas-boceto.html`
— la paleta/estructura sigue sirviendo, solo cambia el copy (vestigio del
encuadre anterior, actualizar textos al integrar).

## 4. Alcance del contrato Compact — DOS CIRCUITOS, `registerFiling` con doble responsabilidad

**Cambio central de esta semana, ampliado en vivo (decisión #14, §6):** el
contrato se divide en dos piezas afines, escribibles en paralelo, y
`registerFiling` ahora alimenta LOS DOS mecanismos (A: credencial personal
de reincidencia; B: alarma pública por caso) en la misma llamada — sin
circuitos nuevos, más responsabilidad en uno ya planeado.

### 4a. `admitCase` — la primitiva de identidad (inspirada en `hofi-passport`)

- Ledger: `admittedCases: MerkleTree<20, Bytes<32>>` + `admittedRoot:
  MerkleTreeDigest` (root congelada, actualizada solo por este circuito).
- Circuito `admitCase(caseCommitment: Bytes<32>)`, gateado por la autoridad
  del organismo de control (un witness de firma/commitment de esa
  autoridad — el mismo patrón de membership proof que `identity_disclosure.compact`
  ya probó sound).
- **Esto cierra el agujero del dry-run playbook §8.4**: el agregado público ya
  no es spameable, porque `registerFiling` (4b) exige que el caso esté admitido.

### 4b. `registerFiling` — el núcleo, con doble salida (A + B)

- **Entrada**: `caseCommitment`, `path: MerkleTreePath<20, Bytes<32>>`.
- Prueba `path.leaf == caseCommitment` y `merkleTreePathRoot(path) ==
  admittedRoot` (comparación PURA — nunca `.checkRoot()`, que solo acepta la
  root *actual* y serializaría a todo el que registre — gotcha §8.3).
- **Nullifier atado al PAR (identidad, caso)**: `H(dominio, subjectSecret,
  caseCommitment)`, contra un `Set<Bytes<32>>` de nullifiers gastados. Evita
  que la misma persona infle el contador de un caso reportándolo mil veces —
  versión más ajustada que el diseño original (que solo evitaba reusar el
  mismo *sello*, no ataba identidad+caso).
- **Salida A (privada)**: incrementa el contador privado de la identidad
  (witness `subjectSecret`) — igual que el diseño original, alimenta
  `proveRepeatFilings`.
- **Salida B (pública, NUEVA)**: incrementa `Map<Bytes<32>, Counter>` keyed
  por `caseCommitment` — gotcha ya conocido, no auto-inicializa
  (`if (!m.member(k)) { m.insert(k, default<Counter>); }`). Si el contador
  del caso cruza un umbral (mismo N que abajo, o uno propio), inserta
  `caseCommitment` en un `Set<Bytes<32>>` de "casos en revisión" — el flag
  que la vista 3 muestra cambiar en vivo.
- **Por qué B puede ser pública acá y no en el diseño de doc 19 §5b**: ese
  caso trataba de un presunto agresor humano (presunción de inocencia); acá
  el sujeto del conteo es un sitio/incidente admitido por una autoridad, no
  una persona — doc 19 mismo autoriza transparencia radical sobre
  instituciones. Ver razonamiento completo en §3.

### 4c. `proveRepeatFilings(N)` — disclosure selectivo (Salida A)

- Sin cambios respecto al diseño original: revela SOLO `contador ≥ N`, gasta
  un nullifier de un solo uso por contexto de presentación.

### 4d. Escalonado de alcance — TRES niveles, no dos

1. **Núcleo (obligatorio, antes de hora 16)**: `admitCase` + `registerFiling`
   con SOLO la salida A (contador privado) + `proveRepeatFilings`. Esto solo
   ya demuestra el mecanismo completo de extremo a extremo.
2. **Extensión tier 1 (si el núcleo compila+testea+demuestra en hora 16)**:
   sumar la salida B plana — el `Map<caseCommitment, Counter>`, sin el flag.
   Costo bajo, ya es casi el mismo código que incrementar un contador más.
3. **Stretch tier 2 (solo si sobra tiempo después de tier 1, ideal para el
   momento fuerte de la demo pero prescindible)**: el `Set` de "casos en
   revisión" y la lógica de umbral que lo dispara. Es lo más vistoso — el
   ledger cambiando de estado en vivo — pero es lo primero que se cae si el
   reloj aprieta.

**Regla de corte sin cambios de espíritu**: cada nivel es opcional respecto
al anterior, nunca al revés. Nadie empieza tier 2 sin tier 1 sólido, nadie
toca tier 1 sin el núcleo compilando y testeado.

## 5. Reglas del evento que gobiernan el plan

*(sin cambios respecto a doc 20 §5 — repetido acá para no tener que saltar de
documento)*

- **Código net-new**: todo se escribe durante el evento desde 7-ago 10:00. Ni
  `identity_disclosure.compact`, ni `castVote.compact`, ni nada del monorepo
  entra al repo. El conocimiento viaja (§8); el código no.
- **Technical gate**: contrato que no compila = descalificación automática.
- **Entrega** (8-ago 13:00, tratar como 12:30 por la ambigüedad de huso — doc
  20 §12.3): repo Git público, label `midnightntwrk`, Apache 2.0, README claro,
  pitch deck, video demo.
- **Rúbrica**: Engineering 40% · QA 15% · Product & Vision 15% · UX 15% ·
  Communication 10% · BizDev 5%. **55% es contrato+tests** — con dos circuitos
  ahora, el doble de superficie para ese 55%, de ahí el valor de partir R1.
- **CELS fuera de todo documento y del pitch** (doc 20 §12.1) — sigue vigente,
  reforzado por el cambio de encuadre de §3.
- **WhatsApp/Kapso fuera del código**, mencionado solo en el deck como canal
  de bajo riesgo (doc 20 §5).

## 6. Todas las decisiones tomadas — cronológico

*(1-7 son de la entrevista original, doc 20 §6; 8+ son de esta semana)*

1. Escena ganadora: la denunciante — `proveRepeatFilings`.
2. Voz simulada en UI (opción 1) — sigue vigente, ver §8.5 para qué hay
   REALMENTE detrás ahora (`admitCase`, no un placeholder vacío).
3. Tres vistas (denunciante + verificador + ledger).
4. Track Open (comprometido tras consulta al organizador).
5. Nombre: **Amparo**.
6. Honestidad estructural: repo fresco, docs como antecedente conceptual.
7. CELS fuera de todo documento y del pitch (reglas del certamen, doc 20 §12.1).
8. **Circuito partido en `admitCase` + `registerFiling`/`proveRepeatFilings`**
   (§4) — cierra el agujero de spam Y trae la primitiva de identidad al flujo,
   sin costo de scope adicional (aprovecha al 4º integrante para paralelizar,
   no para ampliar superficie).
9. **Marta pierde el ángulo policial/estatal** — violencia de género genérica
   (§3).
10. **Posicionamiento ampliado**: Amparo como capa de continuidad verificable
    para CUALQUIER organización con cumplimiento auto-aseverado — no solo el
    caso judicial. Entra como UNA slide de roadmap/mercado (§10.6), la demo
    en sí no cambia.
11. **V0 → Google Stitch** para frontend (exporta HTML/CSS/Tailwind real, no
    solo mockup — §9).
12. **Preprod pasa de "descartado" a "viable"**: el diagnóstico original
    (doc 21) tenía la causa raíz mal — un seed mal derivado, no una
    limitación de Midnight. Con seed correcto + SDK `wallet-sdk >=1.2.0` +
    la wallet ya sincronizada en la máquina del evento, preprod es un
    despliegue real posible, no solo la red local. Detalle operativo
    completo en `runbook-despliegue.md`.
13. **doc 21 (reporte al foro) sigue en borrador** — mezcla causa propia (seed)
    con causa real de Midnight (bug de OOM, resuelto en SDK 1.2). No publicar
    sin separar ambas cosas.
14. **Pivot en vivo, día del evento: caso ambiental + síntesis A+B.** Se
    abandona violencia de género/policial por un delito ambiental (empresa
    ficticia vertiendo residuos industriales a un curso de agua ficticio).
    `registerFiling` gana una salida pública por caso (Map+flag) además de
    la privada por identidad — mismo circuito, más responsabilidad, sin
    incógnitas técnicas nuevas (los gotchas ya estaban catalogados en §8).
    Justificación completa y diseño técnico en §3 y §4.

## 7. Equipo, roles y recursos — 4 PERSONAS

| Rol | Responsabilidad | Depende de |
|---|---|---|
| **R1a — Circuito `admitCase`** | Merkle tree de casos admitidos, gate de autoridad de la org, tests de simulador | Nadie — arranca primero |
| **R1b — Circuito `registerFiling` + `proveRepeatFilings`** | Contador privado, verificación de membership contra `admittedRoot`, disclosure selectivo, tests | La **interfaz** de R1a (acordada a la hora 0 — ver abajo), no el circuito terminado |
| **R2 — Frontend/Integración** | 3 vistas en Stitch, export a Tailwind, conexión wallet/proof-server/contrato, e2e | R1a+R1b para la integración real; puede avanzar UI en paralelo |
| **R3 — Producto/Pitch** | Deck, video, guion de Marta (revisado §3), README, label, entrega 13:00, slide de posicionamiento ampliado (§10.6) | Nadie — arranca primero |

**Interfaz acordada a la hora 0 (para que R1a y R1b no se bloqueen mutuamente):**
`admitCase` publica `admittedRoot: MerkleTreeDigest` en el ledger. `registerFiling`
toma esa root como dato público a leer (no como parámetro que R1a le pase a mano) y
un `MerkleTreePath<20, Bytes<32>>` como witness. R1b puede mockear un root de
prueba y arrancar en paralelo sin esperar a que R1a termine — se integran contra
el root REAL recién en el bloque de 14:00-16:00 (§9).

**Administración de recursos (4 Claude Pro, o el mix que uses):**
- Sesiones separadas por dominio (R1a / R1b / R2 / R3) — nunca mezclar
  contextos.
- Prompts de arranque ensayados de antemano — ver §11.
- Reservar margen de uso para las horas críticas (integración 14:00-16:00,
  noche del viernes, mañana del sábado).

## 8. Gotchas técnicos consolidados (conocimiento — NO copiar código, regla net-new)

*Todo esto viene de trabajo ya hecho (passport F3, `identity_disclosure.compact`,
compilado y testeado) y de `hofi-verified-in-practice.md`. Es lo que el dry-run
iba a medir/confirmar; se lee antes del evento, se reescribe de cero durante.*

### 8.1 De-riskeado (ya se sabe que funciona)

- El gate duro: un contador+umbral compila con compactc 0.31.0 y se testea en
  el simulador.
- La soundness del patrón de umbral: "commiteá el valor en la hoja → atalo a
  la root → probá `≥N` sin revelarlo" — probado con tests adversariales.
- El toolchain end-to-end (compilador en WSL, simulador en Windows/Node nativo).

### 8.2 Gotchas del lenguaje (cuestan minutos si no se saben)

- `sealed` es keyword reservado en Compact → nombrar el parámetro distinto
  (ej. `seal`).
- Comparación de umbral SIN resta, para evitar underflow de `Uint`:
  `birthYear + 18 <= currentYear`, NO `currentYear - birthYear >= 18`. Aplica
  a cualquier comparación de umbral sobre `Uint`.
- `Uint` se ensancha al sumar (`epoch + 1` → tipo más ancho) → estrechar
  explícito con `as Uint<32>` si se reasigna a un ledger `Uint<32>`.
- Todo parámetro es potencialmente privado: escribirlo al ledger exige
  `disclose(...)`. Un valor usado solo en `assert`, o dentro de un hash que
  luego se `disclose`, no necesita `disclose` propio.
- `root()` del ADT `MerkleTree` es runtime-only — no se puede llamar
  in-circuit. Si el circuito necesita la root nueva, entra como parámetro
  (y se gatea la confianza aparte — por eso `admitCase` está gateado por
  autoridad de la org, §4a).
- `Uint<0..N>` — el límite superior es EXCLUSIVO en runtime: `Uint<0..7>`
  admite 0..6, no 7. El `.d.ts` generado no lo marca (erasure a `bigint`) —
  solo el simulador lo atrapa.

### 8.3 El hallazgo que definió el diseño de §4

`MerkleTree.checkRoot()` acepta SOLO la root actual, nunca una histórica —
cualquier inserción invalida el path de todos los demás miembros. Esto
destruye cualquier diseño donde un cohorte prueba contra una foto tomada al
abrir una ventana. **La solución**: congelar la root en un campo de ledger
aparte (`admittedRoot`, en nuestro caso) y verificar por **comparación pura**
(`merkleTreePathRoot(path) == admittedRoot`), no vía `.checkRoot()` — eso es
lo que permite que `registerFiling` (4b) verifique contra el padrón de
`admitCase` (4a) sin serializar a todos los que registran una denuncia el
mismo período.

### 8.4 El agujero de seguridad que motivó el circuito partido (§4)

Sin una restricción externa, `registerFiling` tomando el sello del evento
como witness SIN restricción permite que una identidad lo llame ilimitadas
veces con sellos frescos e infle el agregado público — el nullifier solo
frena el mismo sello dos veces, no una inundación de sellos distintos. La
solución (§4a): el sello debe anclar a evidencia externa única — acá, membership
en el padrón de `admitCase`, admitido solo por la organización.

### 8.5 Qué hay REALMENTE detrás de "voz simulada en UI"

La decisión de UI (§6.2) sigue vigente: la demo muestra "grabá tu voz" pero es
un placeholder de UI. Lo que cambió es qué pasa DESPUÉS de ese placeholder:
ya no es un generador de identidad genérico — es el circuito `admitCase` real
(§4a). El pitch lo puede decir con más precisión ahora: la biometría de voz es
infraestructura futura fuera del repo; la ADMISIÓN del caso es real y
compilada durante el evento.

### 8.6 Midir el costo real de un circuito (si el tiempo alcanza para optimizar)

- `k` (no la cantidad de asserts) determina el tiempo de prueba — se lee con
  `Zkir.deserialize(ir).getK()`. k=15 es el techo local (params en disco).
- El costo real está en los bloques de `persistentHash` (SHA-256, ~2048 filas
  por bloque de 64 bytes), no en los asserts ni en la profundidad de Merkle
  (`transientHash`/Poseidon es casi gratis). Un solo hash puede llevar un
  circuito de k=7 a k=13. **No es prioridad para el MVP** — anotado por si
  hay margen el sábado a la mañana.

### 8.7 Simulador — el loop rápido, sin testnet ni proof server

`@midnight-ntwrk/compact-runtime` (`createConstructorContext` +
`createCircuitContext`) ejecuta la lógica del circuito (asserts, transiciones
de estado) sin generar una prueba ZK real. Correrlo con `tsx --test` (Node 22;
`node --test` no resuelve los imports `.js`→`.ts` del código generado). Es el
gate de QA (15% de la rúbrica) — antes de tocar testnet o el proof server.

## 9. Cronograma del evento (24hs netas) — actualizado para 4 personas y 2 circuitos

**Viernes 7-ago**

| Horario | R1a (admitCase) | R1b (registerFiling+prove) | R2 (frontend) | R3 (pitch) |
|---|---|---|---|---|
| 10:00-11:00 | Kickoff: repo (label, Apache 2.0, README). Arranca ledger+constructor de `admitCase` | Arranca estructura del contador privado (sin depender de R1a aún) | Scaffolding dApp: wallet SDK, cliente proof server. Primer prompt Stitch (vista 1) | Deck base + guion de Marta (revisado §3) |
| 11:00-14:00 | `admitCase` compilando + primer test de simulador | `registerFiling` **paso 1 (núcleo, §4d/§11)** con root MOCKEADA + esqueleto de `proveRepeatFilings` | Vista denunciante (Sofía) en Stitch → export Tailwind, integrando wallet | Guion completo (caso ambiental, §3) + estructura del video + borrador de la slide de posicionamiento ampliado (§10.6) |
| 14:00-16:00 | **Integración real**: `registerFiling` deja de usar el mock, verifica contra el `admittedRoot` real de `admitCase` | ídem (trabajo conjunto R1a+R1b) — núcleo (paso 1) debe quedar sólido acá | Vista verificador. Proof server local levantado | Revisión de guion contra las reglas (nada político, sin nombres/logos/casos reales — doc 20 §12.2) |
| **16:00** | **Checkpoint: ¿compilan los dos circuitos integrados (núcleo, paso 1) y pasa un test e2e (admitCase→registerFiling→proveRepeatFilings)?** Si no ⇒ R2/R3 refuerzan a R1a/R1b, tiers 1 y 2 (§4d) NO se tocan | | | |
| 16:00-18:00 | Si el checkpoint pasó: `registerFiling` **paso 2 — tier 1**, `Map<caseCommitment, Counter>` público | ídem | Vista ledger público, mostrando el contador por caso | Deck: slides 4-6 (demo, doble inversión, roadmap) |
| 18:00-20:00 | Si tier 1 quedó sólido: **paso 3 — tier 2 (stretch)**, el `Set` de "casos en revisión" + lógica de umbral | ídem | Vista ledger: el estado "EN REVISIÓN" cambiando en vivo | Deck casi final |
| 20:00-22:00 | Integración e2e completa: Sofía → `admitCase` → `registerFiling` (con los tiers que hayan quedado) → `proveRepeatFilings` → verificador ve booleano → ledger ve el caso (y su flag, si tier 2 entró). Deploy a LOCAL (garantizado, `runbook-despliegue.md` §2) | | Conexión de las 3 vistas al flujo real | Ensayo del pitch |
| 22:00-00:00 | Suite de tests completa. Intento de deploy a PREPROD (ahora viable — `runbook-despliegue.md` §3, wallet ya sincronizada en esta máquina). Commit estable etiquetado (fallback) | | | |

**Sábado 8-ago**

| Horario | Actividad |
|---|---|
| 09:00-11:00 | Pulido: README completo, edge cases de UI, ensayo de demo en vivo. **Grabar el video ANTES de seguir tocando código** |
| 11:00-12:30 | Freeze de código 12:00. Deck final, video subido, links públicos verificados desde una máquina sin permisos |
| 13:00 | Entrega. 14:00-16:00 pitches privados; 16:30 públicos |

## 10. Estructura del pitch (10% + 15% + 5% de la rúbrica)

1. El problema: la anonimización/el silencio institucional como ceguera de no
   retorno + el cumplimiento auto-aseverado de las herramientas de reporte
   existentes.
2. **Sofía** — la historia del caso ambiental (§3): un vertido industrial,
   sin empresas ni organismos reales, sin ángulo político-partidario.
3. **La doble inversión** (el corazón del pitch ahora): la MISMA prueba
   criptográfica sirve dos signos a la vez — una credencial DE ella
   (reincidencia como denunciante confiable, privada) y una alarma SOBRE el
   caso (el sitio pasa a "en revisión", pública). Es la tesis del doc 16
   literal: la primitiva es real porque sirve a quien acumula lo contrario,
   mostrada en un solo flujo.
4. Demo en vivo (video como plan B): las tres vistas — el momento fuerte es
   la vista 3 cambiando de estado en vivo ("EN REVISIÓN") mientras entran
   reportes.
5. Dual-ledger: qué quedó privado (identidad, historial de Sofía) y qué
   público a propósito (el conteo por caso — porque es institucional, no
   personal; explicar por qué esa distinción es la línea de diseño, no un
   descuido).
6. Una sola slide de roadmap/mercado: el mismo mecanismo sirve a cualquier
   organización, pública o privada, que hoy certifica su propio cumplimiento
   sin poder probarlo — ambiental, laboral, protocolos contra violencia de
   género/acoso/discriminación. **No** reescribir el guion alrededor de esto
   — es una slide, no el eje.
7. Honestidad estructural: diseño previo documentado, código 100% del evento.

## 11. Prompts iniciales por rol

*Punto de partida para la hora 0 — ajustar en el momento, no copiar a ciegas.*

**⚠️ Regla para las cuatro sesiones: referencia de solo lectura, nunca working
directory.** El repo del evento se crea en una carpeta nueva y separada de
`amparo-prep`. Cada prompt de abajo indica archivos puntuales para leer como
conocimiento — no "leé toda la carpeta". Cerrar cada prompt con esta línea
(ya incluida abajo): *"Esto es referencia para entender el patrón — regla
net-new del evento: nada de acá se copia literal, todo se reescribe de cero
en el repo nuevo."*

**R1a — `admitCase`:**
> Vamos a escribir un circuito Compact desde cero (compactc 0.31.0, language
> 0.23.0, runtime @midnight-ntwrk/compact-runtime 0.16.0). Es la primitiva de
> admisión de casos de un canal de denuncias con privacidad. Necesito: un
> ledger `admittedCases: MerkleTree<20, Bytes<32>>` + `admittedRoot:
> MerkleTreeDigest` (root congelada, actualizada solo acá); un circuito
> `admitCase(caseCommitment: Bytes<32>)` gateado por firma/commitment de
> autoridad de la organización, que inserta la hoja y actualiza la root.
> Patrón conceptual de referencia (NO copiar código de ningún repo): membership
> proof tipo Semaphore — hoja opaca, inclusión probada sin revelar cuál.
> Podés leer como referencia (no copiar): `amparo-prep/playbook-evento.md`
> §4a y §8 (gotchas — el de `checkRoot()`/root congelada es crítico acá);
> `amparo-prep/skills/midnight-compact/references/zk-patterns.md` y
> `ledger-operations.md`; y el circuito ya compilado
> `hofi-passport/contracts/midnight/identity_disclosure.compact` (el
> membership proof más cercano a esto). Esto es referencia para entender el
> patrón — regla net-new del evento: nada de acá se copia literal, todo se
> reescribe de cero en el repo nuevo. Empecemos por el ledger y el
> constructor, después el circuito de admisión, con tests de simulador
> (`createConstructorContext`/`createCircuitContext`) antes de tocar el
> proof server.

**R1b — `registerFiling` + `proveRepeatFilings` (con doble salida A+B, §4):**
> Circuito Compact desde cero. Depende de un `admittedRoot: MerkleTreeDigest`
> que publica otro circuito en desarrollo en paralelo (`admitCase`) — por
> ahora mockealo como parámetro de test. Construyamos en TRES pasos, cada uno
> compilando y testeando antes de seguir (no todo junto):
> **Paso 1 (núcleo):** estado privado (contador de denuncias por identidad,
> witness `subjectSecret`); `registerFiling(caseCommitment, path:
> MerkleTreePath<20, Bytes<32>>)` que prueba `path.leaf == caseCommitment` y
> `merkleTreePathRoot(path) == admittedRoot` (comparación PURA, nunca
> `.checkRoot()`), gasta un nullifier `H(dominio, subjectSecret,
> caseCommitment)` — atado al PAR identidad+caso, no solo al caso — y si todo
> pasa incrementa el contador PRIVADO; `proveRepeatFilings(N: Uint<32>)` que
> prueba `contador >= N` sin revelarlo, con su propio nullifier de un solo
> uso por contexto.
> **Paso 2 (extensión tier 1):** sumar a `registerFiling` un
> `Map<Bytes<32>, Counter>` keyed por `caseCommitment` que también incrementa
> (recordá: `Map<K,Counter>` no auto-inicializa, `if (!m.member(k)) {
> m.insert(k, default<Counter>); }` antes de `.increment(1)`).
> **Paso 3 (stretch tier 2, solo si sobra tiempo):** si ese contador cruza un
> umbral, insertar `caseCommitment` en un `Set<Bytes<32>>` de "casos en
> revisión".
> Cuidado: comparación de umbral sin resta, `Uint` se ensancha al sumar
> (`as Uint<32>` explícito), `sealed` es keyword reservado, todo param que se
> escribe a ledger público necesita `disclose(...)`. Podés leer como
> referencia (no copiar): `amparo-prep/playbook-evento.md` §4b/§4d y §8
> completo (los gotchas de `Map<K,Counter>` sin auto-init y `Uint<0..N>`
> exclusivo aplican directo acá);
> `amparo-prep/skills/midnight-expert/references/hofi-verified-in-practice.md`.
> Esto es referencia para entender el patrón — regla net-new del evento: nada
> de acá se copia literal, todo se reescribe de cero en el repo nuevo. Tests
> de simulador en cada paso antes de avanzar al siguiente.

**R2 — Frontend (Google Stitch):**
> Necesito 3 pantallas de una dApp Midnight sobre denuncias ambientales, cada
> una como prompt separado a Stitch, pidiendo export a HTML/CSS/Tailwind (no
> solo Figma). Vista 1 (Sofía, denunciante): fondo violeta noche #26215C,
> acentos ámbar #FAEEDA/#EF9F27, mobile-first, botón de micrófono central
> para enrolamiento (simulado), tarjetas "Reporte sellado" con candado,
> contador personal visible ("Tus reportes: N sellados"). Vista 2
> (Verificador — organismo ambiental/auditor de compliance, lenguaje
> genérico): superficie clara casi vacía, una tarjeta central de resultado
> verde (#EAF3DE/#3B6D11) con "≥N reportes previos: VERDADERO", debajo la
> lista de lo que NO se revela (identidad, fechas, contenido). Vista 3
> (Ledger público): estética terminal, fondo #2C2C2A, texto monoespaciado
> #B4B2A9, lista de casos admitidos con SU PROPIO contador público de
> reportes, y un estado que cambia a "EN REVISIÓN" (con acento visual
> distinto, ej. ámbar/rojo) cuando cruza el umbral — este cambio de estado en
> vivo es el momento fuerte de la demo, dale protagonismo visual. Arrancá
> integrando la vista 1 al wallet SDK desde el principio. Referencia de
> paleta/layout (no código a copiar): `amparo-prep/mockups/tres-vistas-boceto.html`
> — el copy ahí sigue con el encuadre anterior, actualizalo al caso ambiental.

**R3 — Producto/Pitch:**
> Guion de pitch de 3 y 5 minutos para un jurado técnico (Midnight
> Foundation) sobre Amparo: capa de continuidad verificable para canales de
> denuncia con cumplimiento auto-aseverado, con un caso ambiental (empresa
> ficticia vertiendo residuos industriales a un curso de agua ficticio).
> Estructura en §10 de este playbook — el eje narrativo es la DOBLE
> inversión: la misma prueba es credencial privada de la denunciante Y alarma
> pública sobre el caso, a la vez, con la explicación de por qué eso es
> correcto (una institución/sitio puede ser transparente; una persona no).
> Restricciones duras: nada de nombres/logos de terceros ni empresas/ríos
> reales, nada que se lea como mensaje político-partidario, Sofía y el caso
> explícitamente ficticios. Leé como referencia (no copiar):
> `amparo-prep/playbook-evento.md` §3 (guion), §10 (estructura completa) y
> `hofi-protocol-cardano/docs/20-plan-mvp-amparo-hackathon-midnight.md` §12
> (las restricciones del reglamento, con el detalle de por qué).

## 12. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Contrato no compila (gate) | `admitCase` y núcleo primero; checkpoint 16:00 explícito (§9); commit estable etiquetado |
| R1a bloquea a R1b (dependencia entre circuitos) | Interfaz acordada a la hora 0 (§7): R1b mockea la root y arranca en paralelo |
| Preprod con fricción en vivo | `runbook-despliegue.md` — LOCAL es el camino garantizado para lo crítico; preprod es upside, no dependencia |
| Infra local inestable en vivo | Video grabado sábado 09:00 ANTES de tocar más código; commit estable etiquetado como fallback |
| Tokens/uso agotado en hora crítica | Presupuesto por rol (§7); exploración temprana, ejecución tardía |
| Scope creep con agregados | Regla de corte hora 16 — acordada de antemano, no se discute en el evento |
| Stitch genera UI desconectada del contrato | R2 integra desde la vista 1, nunca "UI primero, conexión después" |
| Zona gris net-new | Repo fresco, cero imports propios, docs citados como antecedente conceptual únicamente |
| Guion del pitch se lee como mensaje político | Revisión explícita contra doc 20 §12.2 en el bloque de 14:00-16:00 del viernes (§9) |

## 13. Pendientes / estado

1. ~~Aplicación al formulario~~ y ~~Discord~~ — dar por hechos si no se avisó
   lo contrario; verificar si queda tiempo.
2. Acuerdo interno de distribución del premio, por escrito, entre los 4
   (doc 20 §12.4) — con el 4º integrante sumado, revisar que siga acordado.
3. doc 21 (reporte al foro) — revisar antes de publicar (§6.13). No bloquea
   el evento.
4. Este playbook y `runbook-despliegue.md` son los dos documentos vivos de
   las 24hs — el resto (`amparo-dry-run-playbook.md`, doc 20) queda como
   referencia histórica.

## 14. Referencias

- `amparo-prep/runbook-despliegue.md` — operación de wallet/redes, minuto a
  minuto (no duplicado acá).
- `amparo-prep/skills/midnight-*` — pack de skills canónico con los gotchas
  verificados (`midnight-expert/references/hofi-verified-in-practice.md`).
- `amparo-prep/mockups/tres-vistas-boceto.html` — boceto de paleta/layout
  para los prompts de Stitch.
- `amparo-prep/amparo-dry-run-playbook.md` — versión original, superseded
  por §8 de este doc (queda por fidelidad histórica).
- `hofi-protocol-cardano/docs/20-plan-mvp-amparo-hackathon-midnight.md` —
  plan original, superseded por este doc.
- `hofi-protocol-cardano/docs/21-foro-preprod-ws-report.md` — diagnóstico
  técnico de preprod, pendiente de revisión antes de publicar.
- `hofi-passport/contracts/midnight/` — `identity_disclosure.compact` +
  `providers.ts` (el patrón de persistencia de wallet) — referencia de
  lectura, nunca de copia (regla net-new).
