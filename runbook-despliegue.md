# Runbook de despliegue — Hack Buenos Aires (Amparo)

> **Para quién:** quien esté al teclado desplegando/probando durante las 24hs.
> Léelo ANTES de tocar una wallet, no mientras algo ya está fallando.
> **Qué NO es:** no reemplaza doc 20 (plan/roles/cronograma) ni doc 21 (el
> reporte técnico completo de preprod, en `hofi-protocol-cardano/docs/21-…`).
> Esto es el checklist de manos-en-el-teclado.

---

## 0. El veredicto, primero — para no discutirlo a las 3am

**⚠️ CORREGIDO (7-ago, en vivo): el diagnóstico de doc 21 tenía la causa raíz
mal.** Lo que parecía "preprod no completa el sync jamás" era un **seed mal
derivado** de nuestro lado, no una limitación de la infraestructura de
Midnight. Con el seed corregido, la wallet **sincronizó y viene siendo usada
en varias pruebas exitosas contra preprod**. El reporte del foro (doc 21) le
atribuye el problema al SDK/infra de Midnight — **no publicarlo tal cual**;
está pendiente de revisión o retracción (ver §7).

**Preprod es viable para el evento — con DOS costos operativos distintos, no
confundirlos:**

1. **Primer sync de una wallet (nunca antes sincronizada): 2-3 horas.**
   Confirmado con medición real (nuestra corrida tardó 3h) y por el propio
   código del SDK: escanea desde génesis, sin `birthday`. Esto NO se hace el
   día del evento — tiene que estar hecho de antes, en la máquina que
   efectivamente se va a usar (ver el punto crítico abajo).
2. **Reconexión de una wallet ya sincronizada: ~11 minutos** (medido el 7-ago:
   **643s**, restaurando estado del 26-jul — 11 días de gap). El orden de
   magnitud correcto frente a las 2-3hs del arranque en frío, pero **no son
   "un par de minutos"**: arrancarlo 5 minutos antes de mostrar algo llega
   tarde. Esto se logra gracias a una
   mitigación real que encontramos en `hofi-passport` (patrón portado de
   `hofi-consensus F1`): cada sub-wallet serializa su estado a
   `.wallet-state/` cada ~30s, y al reiniciar hace `restore(blob)` — sync
   **incremental** desde el último save, no desde génesis. Esto es lo que
   confirma, con código y no solo con anécdota, que el costo grande es
   **único**. Sigue aplicando el resto de la lección: la red pública a veces
   tambalea y el margen se puede estirar — arrancarlo temprano igual.

   ⚠️ **El save tarda 30s en dispararse.** Un script que sincroniza y sale antes
   de eso **no persiste nada** y el próximo arranque vuelve a ser desde génesis,
   sin que nada avise. Medido en el repo del evento: las corridas locales
   terminan en ~2s y `.wallet-state/` no se creaba nunca. En preprod el sync
   dura mucho más que un tick, así que no muerde — pero si alguien "prueba que
   la persistencia anda" contra local y sale en verde, **no probó nada**. Para
   ejercitarla de verdad: `MN_WALLET_PERSIST_MS=200`, y confirmar que el
   siguiente arranque diga que está restaurando.

**✅ Punto de logística CONFIRMADO (7-ago): el despliegue del evento corre en
la misma máquina que ya tiene `.wallet-state/`.** Esto no es un detalle menor
— `.wallet-state/` vive en el filesystem, no en git (gitignored a propósito,
material sensible). Si el evento se hubiera corrido desde una máquina
distinta, no había atajo: se volvía a pagar el costo completo de 2-3 horas
sin aviso, probablemente en medio del cronograma. Quedó resuelto ANTES de
necesitarlo — exactamente el tipo de sorpresa que este runbook existe para
evitar.

*Si en algún momento el equipo migra a otra máquina (laptop rota, etc.):
copiar `.wallet-state/` entero a la nueva máquina ANTES de intentar
desplegar ahí (protegido como secreto, nunca por canales inseguros) — o
asumir 2-3h de resync desde cero.*

**La red LOCAL standalone sigue siendo el respaldo garantizado para todo lo
que sea en vivo/crítico** (la demo frente al jurado, un ensayo bajo presión de
tiempo) — no depende del humor de una red pública ni de ningún resync. Node +
indexer + proof server en Docker, wallet de génesis pre-fondeada, sin faucet.
Úsenla para lo que no admite sorpresas; preprod para lo que quieran mostrar
como testnet real, sincronizado con anticipación.

*(doc 20 §11 corregido en consecuencia: ninguna de las dos redes es "el
plan C" — son dos caminos válidos con perfiles de riesgo distintos.)*

---

## 1. Pre-vuelo — hacer esto TEMPRANO, nunca a último momento

Esta es la lección más cara de las últimas semanas: **arrancar el sync
apenas se llega, no cinco minutos antes de mostrarle algo a alguien.**

- [ ] `npm run mn:up` apenas alguien se sienta a trabajar en la integración
  (R2, o quien esté libre) — no esperar a necesitarlo.
- [ ] Esperar los healthchecks del compose (no asumir que "arrancó" = "está
  sano"): `node` expone `/health` cada 2s (hasta 20 reintentos), `indexer`
  depende de `node` sano y chequea su propio archivo `running` cada 10s.
  ⚠️ **CORREGIDO (7-ago, medido): `docker compose ps` marca DOS `healthy`, nunca
  tres.** El `proof-server` no tiene —ni puede tener— healthcheck: la imagen es
  **distroless**, no trae shell ni cliente HTTP, así que ninguna forma de
  `healthcheck` de compose puede ejecutarse adentro (verificado: `docker exec`
  falla con `"sh": executable file not found`). Se queda en `Up` para siempre,
  ande bien o mal. **Esperar tres `healthy` no termina nunca** — y a las 3am eso
  se lee como "la red no levanta" y termina en un `mn:down` sobre un stack sano.
  El chequeo que sí responde por los tres corre desde el host:

  ```bash
  npm run mn:health                     # los 3 por HTTP
  npm run mn:health -- --ws-seconds=30  # + estabilidad de subscriptions
  ```
- [x] ~~Confirmar que `.wallet-state/` está presente en la máquina que se va
  a usar el 7-8 de agosto~~ **CONFIRMADO — es la misma máquina (§0).** Si en
  algún momento cambia la máquina del despliegue, este ítem vuelve a abrirse.
- [ ] Con `.wallet-state/` presente: arrancar el resync apenas alguien se
  sienta, no cinco minutos antes de mostrarla. Son minutos — pero la red
  pública de preprod a veces tambalea y ese margen se puede estirar sin
  aviso.
- [ ] Lo mismo aplica, más previsible, a la red LOCAL si Docker viene de un
  reinicio de la máquina: también son minutos, con menos sorpresas porque no
  depende de infraestructura pública.
- [ ] **Verificar el `.env` antes de asumir cuál red se está usando** — un
  seed apuntado a la red equivocada, o mal derivado, es indistinguible de un
  problema de red hasta que se revisa (fue la causa raíz real detrás de doc 21).
- [x] ~~**Pinnear `@midnight-ntwrk/wallet-sdk` en `>=1.2.0`** en el repo del
  evento~~ **HECHO (7-ago)**: quedó `">=1.2.0 <2"`, resuelve 1.2.0. El aviso era
  correcto y está verificado: el lock del monorepo resuelve **1.1.0 exacto**
  (el `^1.1.0` no alcanza para sacarlo de ahí), así que copiarlo habría
  reintroducido el OOM.
- [ ] 🆕 **Contar las copias de los dos paquetes wasm** — mordió el 7-ago y es de
  los caros porque **typecheck y tests quedan en verde**: ninguno arma una tx.

  ```bash
  npm ls @midnight-ntwrk/ledger-v8 @midnight-ntwrk/onchain-runtime-v3
  ```

  Cada uno tiene que reportar **una sola** versión resuelta. Dos copias = dos
  instancias wasm, cada una con sus propias clases, y el objeto que arma una
  falla el chequeo de tipo de la otra: `expected instance of LedgerParameters`
  y `expected instance of StateValue`, **desde adentro de una dependencia, en
  medio del deploy**, sin nada que apunte a la duplicación. Pasa porque
  `midnight-js-protocol` los pinea exactos mientras el resto pide `^`. En el
  repo del evento ya está cerrado con `overrides`; si alguien toca dependencias,
  **el chequeo es este comando, no que compile**.
- [ ] Proof server respondiendo: primera prueba de la sesión descarga
  ~33 MB de params públicos desde S3 (lento la primera vez). El volumen
  `.zk-params/` ya persiste esto entre corridas — si alguien borró esa
  carpeta o es una máquina nueva, dar tiempo a esa descarga ANTES de la demo,
  no durante.
- [ ] `npm run check-wallet` (en `contracts/` del repo del evento) contra la red
  local, temprano, para confirmar que la wallet de génesis sincroniza y expone
  sus claves — no esperar al primer deploy real para descubrir un problema de
  conectividad.
- [ ] **Correr los tres en orden, no saltar al deploy.** `mn:health` dice que los
  servicios contestan · `check-wallet` dice que ESE seed arma una wallet que
  sincroniza · `deploy` es el primer paso que genera una prueba. En ese orden,
  cada falla nombra su causa; salteando, un indexer que todavía arrancaba se lee
  como un error de balanceo.

---

## 2. Camino garantizado: red LOCAL standalone

✅ **Camino verificado de punta a punta el 7-ago** en `midnight-hackathon-ba`
(rama `feat/midnight-network-harness`): red local sana → wallet sincronizada →
deploy con prueba ZK del constructor → `admitCase` con gate de autoridad →
espejo alineado contra el árbol real. **No hace falta `.env` para local**: los
defaults del código ya apuntan ahí.

```bash
# 1) Levantar la red (una vez por sesión de trabajo; queda corriendo)
cd contracts/
npm run mn:up

# 2) Confirmar salud — los TRES, desde el host (ver §1: ps solo marca dos)
npm run mn:health

# 3) Compilar el contrato (WSL: el compilador no está en el PATH de Windows)
wsl -e bash -lc 'export PATH="$HOME/.local/bin:$PATH"; \
  cd /mnt/c/Users/<usuario>/dev/midnight-hackathon-ba/contracts; npm run compile'

# 4) Tests de simulador (rápido, sin proof server — el gate de QA, 15% rúbrica)
npm test

# 5) Wallet, luego deploy real (usa proof server + node locales)
npm run check-wallet
npm run deploy

# 6) Ejercitar el circuito gateado de punta a punta
npm run admit-case

# Al cerrar la sesión (NO en medio de la demo):
npm run mn:down        # -v borra los volúmenes — perder el estado es reiniciar de cero
```

**Apuntar a otra red por UNA corrida, sin tocar el `.env`:** las variables del
shell **ganan** sobre el `.env` (verificado). Sirve para no editar un archivo
que tiene el seed de otra red:

```bash
MN_NETWORK=undeployed MN_NODE_URL=http://127.0.0.1:9944 npm run check-wallet
```

Y el `.wallet-state/` no se pisa entre redes: el directorio se llama
`<red>-<hash del seed>`, así que una corrida local convive con la de preprod.

**Probe de 30 segundos antes de un ensayo:** `npm run mn:health -- --ws-seconds=30`
sostiene las subscriptions abiertas, que es de lo que depende el sync de una
wallet y ningún chequeo HTTP observa.

---

## 3. Preprod — viable, con margen de resync

Usar la wallet ya sincronizada (seed verificado — §1). Si no se usó en varios
días, arrancar el resync temprano y darle su margen: **~11 min medidos el
7-ago**, y puede estirarse. No usarla recién sincronizada para la demo en vivo
frente al jurado sin haberla probado antes ese mismo día — para lo que no
admite sorpresas, red LOCAL (§2).

```bash
# .env apuntando a preprod (valores de referencia — confirmar contra el
# .env real del equipo, que ya tiene el seed correcto):
#   MN_NETWORK=preprod
#   MN_PROOF_SERVER_URL=http://127.0.0.1:6300   (local, aunque la red sea preprod)
#   MN_INDEXER_URL=https://indexer.preprod.midnight.network/api/v4/graphql
#   MN_INDEXER_WS_URL=wss://indexer.preprod.midnight.network/api/v4/graphql/ws
#   MN_NODE_URL=https://rpc.preprod.midnight.network
#   MN_WALLET_SEED=<el seed corregido — nunca en el repo ni en el chat>

npm run proof-server        # SOLO ese servicio: el proof server corre local aunque
                            # la red sea preprod. NO usar mn:up, que además levanta
                            # un nodo local y te deja dos cadenas en los mismos puertos.
npm run check-wallet        # confirma sync antes de cualquier deploy — no asumir
npm run deploy
```

Chequeo rápido de salud antes de arrancar (no reemplaza `check-wallet`):

```bash
npm run mn:health -- --ws-seconds=30 \
  --node=https://rpc.preprod.midnight.network \
  --indexer=https://indexer.preprod.midnight.network/api/v4/graphql
```

⚠️ **Este probe mide subscriptions livianas, no el sync pesado de la
wallet.** Verde acá es una señal de que la red está arriba — no una garantía
de que el resync de la wallet vaya a andar sin fricción. Es información
parcial, no un semáforo definitivo (la distinción que costó entender en
doc 21).

---

## 4. Triage de fallas — síntoma → causa probable → qué hacer

| Síntoma | Causa probable | Acción |
|---|---|---|
| `docker compose ps` no marca `healthy` a los ~30s | Primera vez, descargando imágenes o params ZK | Esperar; no cancelar. Si pasan >3 min sin cambio, `mn:down` y `mn:up` de nuevo |
| **El `proof-server` nunca pasa de `Up` a `healthy`** | ✅ **Es lo esperado, no una falla.** La imagen es distroless: ningún healthcheck puede correr adentro | `npm run mn:health` — chequea los tres desde el host. **No reiniciar el stack por esto** |
| `expected instance of LedgerParameters` / `expected instance of StateValue`, en medio del deploy | Dos copias de un paquete wasm ⇒ dos instancias, cada una con sus clases | `npm ls @midnight-ntwrk/ledger-v8 @midnight-ntwrk/onchain-runtime-v3` — cada uno tiene que dar UNA versión. Se cierra con `overrides` (§1) |
| `Cannot read properties of undefined (reading 'ctor')` al desplegar | Se pasó una instancia del contrato donde va el **descriptor** (`CompiledContract.make(...)` + witnesses + assets) | Revisar el nombre de la opción: es `compiledContract`, y un cast `as never` sobre el objeto de opciones **tapa el error de tipos** |
| `Password must contain at least 3 of...` en medio del deploy | La contraseña del estado privado no cumple la política (≥16 chars, 3 de 4 clases) | Ajustar `MN_PRIVATE_STATE_PASSWORD`. Falla en la primera escritura, o sea ya arrancado el deploy |
| `Contract address not set. Call setContractAddress()` | Se escribió el estado privado a mano antes de encontrar el contrato | No escribirlo aparte: va como `initialPrivateState` de `findDeployedContract`, que es quien lo guarda |
| Wallet "syncing" que no avanza (LOCAL o preprod) | Resync normal en curso — Docker recién reiniciado, host con hipo de red, o preprod tras días de inactividad | Darle el margen de §1 (~11 min medidos, puede estirarse). No cancelar a los pocos segundos |
| El script dice `syncing from genesis. This is the slow path` cuando esperabas minutos | No hay estado en `.wallet-state/` para esa combinación red+seed | Confirmar que el directorio `<red>-<hash>` existe. Si cambió el seed, es otra wallet: el resync es completo y no hay atajo |
| Wallet "syncing" que **nunca** avanza, `CloseEvent`/`ErrorEvent` repetidos, sin mejorar con tiempo | Seed mal derivado o mal pegado en `MN_WALLET_SEED` (causa raíz real de doc 21) | Verificar el seed contra la fuente correcta — no asumir problema de red antes de revisar esto |
| `FATAL ERROR: JavaScript heap out of memory` | `wallet-sdk` < 1.2 (bug conocido, parcheado) | Confirmar el pin en `package.json` (§1) — no debería aparecer con `>=1.2.0` |
| Archivos en 0 bytes / el compilador no encuentra el `.compact` | Sandbox Windows↔mount desincronizado (trampa conocida del proyecto) | `wc -c` al archivo antes de confiar en el build; reintentar el compile |
| `compact: command not found` en Windows | El compilador vive en WSL, no en PATH nativo de Windows | Usar el comando WSL de §2 paso 4, no `compact` directo en PowerShell |
| Proof server tarda mucho en la primera prueba | Descarga de ~33 MB de params ZK desde S3 | Normal la primera vez por máquina; el volumen `.zk-params/` lo cachea para las siguientes |
| `esbuild`/`tsx` tira `Error: You installed esbuild for another platform…` | `node_modules` instalado en Windows y corrido desde Linux (WSL con node_modules copiado, CI, un sandbox) — binario nativo no coincide | Correr `npm install` (o al menos reinstalar `esbuild`) DESDE el entorno que efectivamente ejecuta. No copiar `node_modules` entre Windows y WSL/Linux |
| Deploy exitoso pero la demo en vivo falla justo antes del pitch | Cualquiera de los anteriores, bajo presión de tiempo | Usar el **video grabado el sábado 09:00** (doc 20 §9) — no intentar arreglar en vivo frente al jurado |

---

## 5. Quién corre esto

Por defecto R2 (frontend/integración, dueño de wallet + proof server en doc
20 §7) — pero con 4 personas, cualquiera libre puede levantar `mn:up` y
dejarlo sano mientras R2 sigue con las vistas. Es infraestructura, no lógica
de negocio: no hace falta ser quien la usa para ponerla en marcha.

## 6. Abort criteria

Si a la hora del **checkpoint de doc 20 §9** (16:00 del viernes) la red local
no está sana, es un problema de infraestructura del equipo, no de Midnight —
se resuelve reinstalando Docker Desktop o migrando a la laptop de otro
integrante, no debuggeando el stack en el momento. El commit estable
etiquetado (doc 20 §9, 22:00-00:00) es el fallback; el video del sábado 09:00
es el fallback del fallback. Nunca debuggear infraestructura frente al jurado.

## 7. Referencias

- `hofi-protocol-cardano/docs/21-foro-preprod-ws-report.md` — el diagnóstico
  técnico del incidente original (logs y números). **Sigue en borrador, sin
  publicar, y necesita revisión**: mezcla una causa nuestra (seed mal
  derivado) con una causa real de Midnight (bug de OOM, ya resuelto en SDK
  1.2). No publicar sin separar ambas cosas.
- 🆕 `midnight-hackathon-ba/contracts/scripts/mn-health.mjs` — **el probe que
  usa el evento** (`npm run mn:health`). Cubre los tres servicios desde el host,
  incluido el proof server que ningún healthcheck de compose puede mirar. Acepta
  `--node=`, `--indexer=`, `--proof=` y `--ws-seconds=N`, así que sirve igual
  contra local y contra preprod.
- 🆕 `midnight-hackathon-ba/contracts/.env.example` — la referencia de
  configuración al día, con los defaults locales y el bloque de preprod.
- 🆕 `midnight-hackathon-ba/contracts/docker-compose.midnight.yml` +
  `standalone.env` — **la red local del evento**, escrita de cero. Documenta en
  el propio archivo por qué el proof server no lleva healthcheck.
- 🆕 `midnight-hackathon-ba/contracts/src/midnight/providers.ts` — el mecanismo
  de `restoreOrStart` / `.wallet-state/` que hace que el segundo sync sea de
  minutos, reescrito para este repo (regla net-new). Ahí vive también el
  backpressure del sync y el cap que evita el OOM.
- Referencias de patrón (conocimiento, **no** código a copiar — regla net-new):
  `hofi-protocol-cardano/packages/contracts-midnight/` y
  `hofi-passport/contracts/midnight/`.
- `amparo-prep/amparo-dry-run-playbook.md` — el patrón del contrato en sí
  (conocimiento, no código a copiar — regla net-new).
- `hofi-protocol-cardano/docs/20-plan-mvp-amparo-hackathon-midnight.md` —
  plan, roles, cronograma, rúbrica.
