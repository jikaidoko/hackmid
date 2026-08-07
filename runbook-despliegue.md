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
2. **Reconexión de una wallet ya sincronizada: minutos**, gracias a una
   mitigación real que encontramos en `hofi-passport` (patrón portado de
   `hofi-consensus F1`): cada sub-wallet serializa su estado a
   `.wallet-state/` cada ~30s, y al reiniciar hace `restore(blob)` — sync
   **incremental** desde el último save, no desde génesis. Esto es lo que
   confirma, con código y no solo con anécdota, que el costo grande es
   **único**. Sigue aplicando el resto de la lección: la red pública a veces
   tambalea y el margen se puede estirar — arrancarlo temprano igual.

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
- [ ] Esperar los tres healthchecks del compose (no asumir que "arrancó" =
  "está sano"): `node` expone `/health` cada 2s (hasta 20 reintentos),
  `indexer` depende de `node` sano y chequea su propio archivo `running`
  cada 10s. Verificar con `docker compose -f docker-compose.midnight.yml ps`
  que los tres servicios digan `healthy`, no solo `Up`.
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
- [ ] **Pinnear `@midnight-ntwrk/wallet-sdk` en `>=1.2.0` en el `package.json`
  del repo del evento** — la 1.2 resolvió el bug de OOM que doc 21 documentó.
  El repo de referencia (`hofi-protocol-cardano`, monorepo HoFi) todavía tiene
  fijado `^1.1.0`: **no copiar ese pin** al armar el proyecto del evento el
  7-ago — sería reintroducir un bug ya resuelto.
- [ ] Proof server respondiendo: primera prueba de la sesión descarga
  ~33 MB de params públicos desde S3 (lento la primera vez). El volumen
  `.zk-params/` ya persiste esto entre corridas — si alguien borró esa
  carpeta o es una máquina nueva, dar tiempo a esa descarga ANTES de la demo,
  no durante.
- [ ] `check-wallet` (`npm run check-wallet` en `contracts-midnight`, adaptado
  al proyecto del evento) contra la red local, temprano, para confirmar que
  la wallet de génesis sincroniza y expone sus claves — no esperar al primer
  deploy real para descubrir un problema de conectividad.

---

## 2. Camino garantizado: red LOCAL standalone

```bash
# 1) Levantar la red (una vez por sesión de trabajo; queda corriendo)
cd contracts-midnight/   # o el nombre del paquete Midnight del repo del evento
npm run mn:up

# 2) Confirmar salud (no asumir — verificar)
docker compose -f docker-compose.midnight.yml ps
# node, indexer y proof-server deben decir "healthy"

# 3) .env del proyecto apuntando a LOCAL (undeployed) — valores de referencia:
#    MN_NETWORK=undeployed
#    MN_PROOF_SERVER_URL=http://127.0.0.1:6300
#    MN_INDEXER_URL=http://127.0.0.1:8088/api/v4/graphql
#    MN_INDEXER_WS_URL=ws://127.0.0.1:8088/api/v4/graphql/ws
#    MN_NODE_URL=http://127.0.0.1:9944
#    MN_WALLET_SEED=   (vacío = usa la wallet de génesis pre-fondeada, 0x00…01)

# 4) Compilar el contrato del evento (WSL)
wsl -e bash -lc 'export PATH="$HOME/.local/bin:$PATH"; cd /mnt/c/.../<repo-evento>/; \
  compact compile src/<circuito>.compact src/managed/<circuito>'

# 5) Tests de simulador (rápido, sin proof server — el gate de QA, 15% rúbrica)
npm test

# 6) check-wallet, luego deploy real (usa proof server + node locales)
npm run check-wallet
npm run deploy        # o el script que el proyecto del evento defina

# Al cerrar la sesión (NO en medio de la demo):
npm run mn:down        # -v borra los volúmenes — perder el estado es reiniciar de cero
```

**Health-probe reusable:** `preprod-health.mjs` (en
`hofi-protocol-cardano/packages/contracts-midnight/scripts/`) también sirve
para LOCAL apuntándolo con env vars:

```bash
MN_PREPROD_RPC_URL=http://127.0.0.1:9944 \
MN_PREPROD_INDEXER_URL=http://127.0.0.1:8088/api/v4/graphql \
node scripts/preprod-health.mjs --ws-seconds=30
```

Útil como chequeo de 30 segundos antes de un ensayo de demo — no reemplaza
mirar `docker compose ps`, lo complementa.

---

## 3. Preprod — viable, con margen de resync

Usar la wallet ya sincronizada (seed verificado — §1 último ítem). Si no se
usó en varios días, arrancar el resync temprano y darle su margen (minutos,
puede estirarse). No usarla recién sincronizada para la demo en vivo frente
al jurado sin haberla probado antes ese mismo día — para lo que no admite
sorpresas, red LOCAL (§2).

```bash
# .env apuntando a preprod (valores de referencia — confirmar contra el
# .env real del equipo, que ya tiene el seed correcto):
#   MN_NETWORK=preprod
#   MN_PROOF_SERVER_URL=http://127.0.0.1:6300   (local, aunque la red sea preprod)
#   MN_INDEXER_URL=https://indexer.preprod.midnight.network/api/v4/graphql
#   MN_INDEXER_WS_URL=wss://indexer.preprod.midnight.network/api/v4/graphql/ws
#   MN_NODE_URL=https://rpc.preprod.midnight.network
#   MN_WALLET_SEED=<el seed corregido — nunca en el repo ni en el chat>

npm run proof-server &      # el proof server SIEMPRE corre local, aunque la red sea preprod
npm run check-wallet        # confirma sync antes de cualquier deploy — no asumir
npm run deploy               # o el script del proyecto del evento
```

Chequeo rápido de salud antes de arrancar (no reemplaza `check-wallet`):

```bash
node scripts/preprod-health.mjs --ws-seconds=30
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
| Wallet "syncing" que no avanza (LOCAL o preprod) | Resync normal en curso — Docker recién reiniciado, host con hipo de red, o preprod tras días de inactividad | Darle el margen de §1 (minutos, puede estirarse). No cancelar a los pocos segundos |
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
- `hofi-protocol-cardano/packages/contracts-midnight/scripts/preprod-health.mjs`
  — probe HTTP+WS, reusable contra local con env vars (§2).
- `hofi-protocol-cardano/packages/contracts-midnight/docker-compose.midnight.yml`
  + `standalone.env` — la definición exacta de la red local.
- `hofi-passport/contracts/midnight/scripts/check-wallet.ts` (agregado 7-ago,
  `npm run check-wallet`) — construye la wallet y espera sync, sin firmar ni
  enviar nada. Espejo del script homónimo del monorepo.
- `hofi-passport/contracts/midnight/src/midnight/providers.ts` (líneas
  ~59-65, ~206-217) — el mecanismo real de `restoreOrStart`/`.wallet-state/`
  que hace que el segundo sync sea de minutos. Patrón "ODATANO/NIGHTGATE",
  portado de `hofi-consensus F1`. Vale la pena releerlo si el contrato del
  evento necesita su propia interacción de wallet — el patrón es el mismo,
  se reescribe de cero (regla net-new), no se copia.
- `amparo-prep/amparo-dry-run-playbook.md` — el patrón del contrato en sí
  (conocimiento, no código a copiar — regla net-new).
- `hofi-protocol-cardano/docs/20-plan-mvp-amparo-hackathon-midnight.md` —
  plan, roles, cronograma, rúbrica.
