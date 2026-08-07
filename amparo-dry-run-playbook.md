# Amparo — Playbook del dry-run (R1 / contrato Compact)

> ⚠️ **SUPERSEDED (7-ago-2026)** por `amparo-prep/playbook-evento.md` §8 —
> los gotchas de este doc están consolidados ahí, junto con la solución que
> le dieron a §4 (el agujero de spam): el contrato se partió en `admitCase` +
> `registerFiling`/`proveRepeatFilings`. Queda acá por fidelidad histórica.
>
> **Qué es esto:** el mapa para ensayar y cronometrar el camino del contrato antes del
> Hack Buenos Aires (7-8 ago 2026). Sale del desarrollo de **passport F3**
> (`hofi-passport/contracts/midnight/identity_disclosure.compact`), que implementó y
> probó el núcleo de Amparo (contador privado + umbral) en el dominio identidad.
>
> 🚨 **REGLA NET-NEW (doc 20 §5, §11):** en el evento se escribe TODO de cero desde el
> 7-ago 10:00. Ni una línea de passport/reputation/castVote entra al repo. **Este
> playbook es CONOCIMIENTO que viaja, no código para pegar.** El skeleton de abajo es
> para *entender el patrón*; en el evento se reescribe de cero. Los archivos del
> monorepo/passport se leen como referencia, jamás se copian al repo del evento.

---

## 0. Qué de esto ya está de-riskeado (y qué NO)

**De-riskeado (passport F3 lo probó):**
- El **gate duro** (contrato que no compila = descalificación): sabemos que un
  contador+umbral compila con compactc 0.31.0 y cómo se testea en el simulador.
- La **soundness del umbral**: el patrón "commiteá el valor en la hoja → atalo a la
  root → probá `≥N` sin revelarlo" funciona y es sound (probado con tests adversariales).
- Los **gotchas** del lenguaje (abajo).
- El **toolchain** end-to-end en 1 de 3 máquinas.

**NO de-riskeado (el dry-run SIGUE siendo necesario):**
- **Las horas HUMANAS reales.** passport F3 lo hizo una IA con el monorepo cargado; R1
  necesita cronometrar SU velocidad escribiendo de cero. Ése es el punto del dry-run.
- **R2 (3 vistas V0 + proof-server) y R3 (pitch).** Esto no tocó nada de eso.
- **El contador SELF-SERVICE puro** (ver §4): passport usó el modelo atestado; Amparo
  probablemente quiera que el contador crezca solo con `registerFiling`. Ése es el
  desafío abierto a ensayar.

---

## 1. Toolchain (validado en la máquina de referencia — replicar en las 3)

- **Compilador `compact`**: vive en WSL (`~/.local/bin/compact`). Versión del *tool*
  0.5.1 → compilador por defecto **0.31.0** (= el verificado del monorepo; language 0.23,
  runtime `@midnight-ntwrk/compact-runtime` **0.16.0**). `compact list` para ver versiones.
- **node/npm/tsx**: del lado **Windows** (node v22). docker en WSL (para `mn:up`/proof-server).
- **Flujo cross-boundary** (archivos en `C:`, accesibles desde WSL por `/mnt/c`):

```bash
# 1) compilar el circuito (WSL) → genera src/managed/<nombre>/
wsl -e bash -lc 'export PATH="$HOME/.local/bin:$PATH"; cd /mnt/c/.../<repo>/; \
  compact compile src/<circuito>.compact src/managed/<circuito>'

# 2) deps + tests de simulador (Windows)
npm install          # solo @midnight-ntwrk/compact-runtime@0.16.0 + tsx + typescript
npm test             # tsx --test src/**/*.test.ts
```

- `managed/` va **gitignored** (regenerable). `npm run compile` asume `compact` en PATH
  (no está en Windows) → el paso 1 se corre por WSL a mano.
- **Prep pendiente**: hacer esto en las **3 máquinas** + wallet/tDUST testnet para el
  stretch `mn:up` (nodo+indexer+proof-server por docker; loop local sin faucet).

---

## 2. Gotchas medidos (los que cuestan minutos si no se saben)

- **`sealed` es keyword reservado** en Compact → nombrar el param de otra forma (`seal`).
- **Comparación sin resta para evitar underflow de Uint**: `birthYear + 18 <= currentYear`,
  NO `currentYear - birthYear >= 18`. Vale para cualquier umbral sobre `Uint`.
- **`Uint` se ensancha al sumar** (`epoch + 1` → Uint más ancho) → estrechar explícito
  con `as Uint<32>` si se reasigna a un ledger `Uint<32>`.
- **Todo param es potencialmente privado**: usar un valor como dato PÚBLICO (escribirlo al
  ledger) exige `disclose(...)`. Un valor usado solo en `assert` o dentro de un hash que
  luego se `disclose` NO necesita `disclose` propio.
- **`root()` del ADT MerkleTree es runtime-only**: no se puede llamar en circuito. Si el
  circuito necesita la root nueva, entra como parámetro (y se gatea la confianza aparte).
- **Simulador = deploy-free y prover-free**: `createConstructorContext` + `createCircuitContext`
  del `@midnight-ntwrk/compact-runtime` ejecutan la lógica (asserts/estado) SIN generar
  prueba ZK ni tocar testnet. Es el loop de test rápido — el gate de QA (15% rúbrica).
- **Root off-chain == in-circuit**: construir el árbol off-chain con `StateBoundedMerkleTree`
  (la MISMA impl que respalda el ADT de la stdlib), NO hashear a mano. El round-trip
  (root off-chain == la que el circuito reconstruye con `merkleTreePathRoot`) es un test.

---

## 3. El patrón del núcleo (CONOCIMIENTO — reescribir de cero en el evento)

El núcleo de Amparo (doc 20 §4): `registerFiling(commitment)` + `proveRepeatFilings(N)`
que revela SOLO `contador ≥ N`. El patrón que probó ser sound y compilar:

**Idea (NO copiar; entender y reescribir):**
1. **Hoja atestada** = commitment que ata la identidad (un secreto `sk` por `witness`) a
   sus datos, incluido el **contador** de denuncias. `leaf = H(tag, sk, H(datos, contador))`.
2. **`boundIdentity`**: recomponé la hoja desde los `witness` y exigí su inclusión en la
   `root` pública (`assert(path.leaf == leaf)` + `assert(merkleTreePathRoot(path) == root)`).
   **Esto es la soundness**: el prover NO puede mentir el contador — reclamar `contador=5`
   estando atestado con `3` recompone OTRA hoja que no está en el árbol → rechazo.
3. **`proveRepeatFilings(N)`**: `boundIdentity()` + `assert(contador >= N)` + gastar un
   nullifier de credencial. Se `disclose` SOLO el nullifier opaco; **el contador nunca se
   revela**, solo el booleano del umbral.
4. **Dual-ledger** (el golpe visual + 40% engineering): un `Counter` público
   (`eventTotal`) + un `Set` de nullifiers de evento, alimentados por `registerFiling`. El
   ledger público muestra "N denuncias este período", sin personas.

**Nullifier**: `H(dominio, sk, context)`. Ata `(sk, context)`, NO el hecho. `context` DEBE
ser un **nonce de un solo uso** por presentación (expediente/desafío del juzgado).

---

## 4. ⚠️ El agujero que Amparo TIENE que decidir (descubierto en F3)

En passport F3, `registerFiling` toma el sello del evento como un `witness` **sin
restringir** → una identidad puede llamarlo ilimitadas veces con sellos frescos e **inflar
el agregado público**. El nullifier solo frena el mismo sello dos veces.

**Implicancia para Amparo:** el número público "14 denuncias este período" es **spameable**
salvo que el sello (`H(case_id ‖ evidencia ‖ salt)`) se **ancle a evidencia externa única**
— un `case_id` emitido por un registro autoritativo, o inclusión en un padrón de casos. El
dual-ledger vale 40% de rúbrica; decidir esto ANTES del evento (aunque sea "en el MVP el
case_id lo firma un juzgado mock") evita descubrirlo a las 3am.

**Dos modelos del contador** (elegir en el dry-run):
- **Atestado** (lo que hizo F3): un atestador re-commitea el contador → nueva root. Sound y
  simple, pero necesita un atestador. `proveRepeatFilings` prueba el conteo atestado.
- **Self-service puro** (más fiel a doc 20 §4): el contador crece solo con `registerFiling`.
  Más difícil (requiere encadenar el contador en commitments o una root congelada por
  período, como el `epochRoot` de `rotateReputation`) — ES el desafío a cronometrar.

---

## 5. Cómo usar esto en el dry-run (21-27 jul y 1-2 ago)

**Dry-run #1 (R1):** escribir de cero un contador+umbral de OTRO dominio (se descarta).
- Cronometrar: setup de proyecto → `registerFiling` compilando → `proveRepeatFilings`
  compilando → primer test de simulador verde. **Medir horas reales de CADA hito.**
- Ensayar la disciplina: commit estable etiquetado por hito (fallback de demo).
- Meta de checkpoint del evento (§9 doc 20): a la **hora 4-6**, núcleo compila + un test pasa.
  Si el dry-run mide más que eso, ajustar el cronograma o el alcance ANTES del evento.

**Dry-run #2 (completo, 1-2 ago):** simulacro de las 24hs — de cero a demo de un caso
análogo descartable, con R2 (3 vistas + proof-server) y R3 (pitch) en paralelo. Medir
cuellos de botella y consumo de tokens; recalibrar el cronograma real con esos números.

**Regla de oro:** todo lo escrito se borra. Lo que queda es el conocimiento, los tiempos
medidos y los prompts ensayados.

---

## 6. Referencias (LEER como mapa, NO copiar)

- `hofi-passport/contracts/midnight/` — el desarrollo de referencia (circuito + witnesses +
  attested-set + tests de simulador + `README.md` + `docs/adr/0004-*`). El más cercano a Amparo.
- Monorepo `packages/contracts-midnight/src/`: `certify.compact` (pertenencia+nullifier),
  `rotateReputation.compact` (umbral/atributo + el patrón de root congelada por época),
  `certify.test.ts` (harness de simulador), `cert/certified-set.ts` (root off-chain).
- Skills `midnight-*` (concepts/compact/api/dapp-dev/wallet/network/expert). El conocimiento
  verified-in-practice (incl. los gotchas de §2) vive en
  `midnight-expert/references/hofi-verified-in-practice.md`.
- doc 19 (vertical CELS completa) + doc 20 (plan del evento) — antecedente conceptual público.
