# Identidad con privacidad: volver CIEGO al operador (cripto off-chain)

Cuando el sistema debe **custodiar identidades sin poder ver quién es quién** —ni
siquiera con compromiso total de su backend— la privacidad NO la da la cadena: la da
una capa de **criptografía off-chain**. Esta guía es la experiencia de construir esa
capa para HoFi (PRD "Identidad Custodial Ciega"); aplica a cualquier sistema que
quiera seudonimato + búsqueda + recuperación de emergencia sin un punto que lo sepa todo.

## 0. El reparto: ceguera (off-chain) ≠ verificabilidad (on-chain)

Dos problemas distintos, dos capas distintas. **No los confundas:**

- **Ceguera (~70%, off-chain, este archivo):** que el operador (backend, DB, logs,
  colas) no pueda mapear un seudónimo a una persona. Es cripto simétrica/KDF/umbral
  corriendo del lado del cliente o en dominios de confianza separados.
- **Verificabilidad (~30%, on-chain, ver `midnight.md`):** probar atributos
  ("es miembro", "reputación ≥ N") sin revelar identidad. Es ZK (Midnight/Compact).

Una blockchain pública (Cardano L1, eUTXO) es **lo contrario de privada**: todo
datum/dirección/valor es visible y permanente. **Nunca pongas PII —ni hashes de PII—
en un datum o en metadata CIP-20/68.** Lo on-chain lleva, a lo sumo, un *commitment*
opaco. La PII vive cifrada off-chain.

## 1. Seudónimo opaco, NO derivado de la PII

El identificador que ven el backend, los logs y la derivación de wallets debe ser
**aleatorio**, no `hash(email)` ni `hash(phone)`:

```python
def new_subject_id() -> str:
    return "subj_" + secrets.token_hex(16)   # 128 bits, opaco
```

Si lo derivás de la PII, cualquiera que conozca el algoritmo y *adivine* el email
confirma la identidad (ataque de diccionario offline). Aleatorio = no hay nada que
adivinar. Este `subject_id` reemplaza al `person_id`/email en TODO lo que toca el
operador.

## 2. Blind index: buscar sin revelar el valor

Para hacer login por email/wallet sin guardar el email en claro, indexá con un HMAC
con clave de servidor:

```python
def blind_index(type_, value, *, key):       # HMAC-SHA256_k(type ‖ 0x1f ‖ value)
    msg = type_.encode() + b"\x1f" + value.encode()
    return hmac.new(key, msg, sha256).hexdigest()
```

`(type, value) → persona` en claro se vuelve `(type, blind_index) → subject_id`. Con
la misma `key`, el mismo valor da el mismo índice (permite el lookup); sin la `key`,
el índice no revela el valor.

- La `key` del blind index es un **secreto de servidor SEPARADO** de las KEK del
  vault (otro dominio): un breach de la `key` permite *confirmar* un valor adivinado
  (HMAC no es reversible) pero NO descifra PII.
- **⚠️ Minimizá los índices por sujeto (hallazgo del estado del arte).** Cada blind
  index es una superficie de correlación: es **exact-match**, delata **plaintexts
  duplicados** entre sujetos, y la fuga **crece con la cantidad de índices**
  (CipherSweet; IACR ePrint 2019/806). Indexá **solo los factores de login**
  (los que necesitás para resolver el seudónimo); la PII que no se busca (teléfono,
  nombre) va cifrada en el vault SIN índice. Dedupeá y poné un **cap explícito** que
  falle ruidoso ante el sprawl.

## 3. Envelope encryption: el operador no tiene ninguna llave

El registro de PII se cifra con un **DEK por registro**; el DEK se *envuelve* con una
o más **KEK**. El operador **nunca** tiene una KEK → no puede desenvolver ningún DEK.

```
PII --AES-256-GCM(DEK, AAD=subject_id)--> ciphertext   (en la DB)
DEK --AES-256-GCM(KEK_user, AAD="user")--> wrapped_dek_user
DEK --AES-256-GCM(KEK_break, AAD="breakglass")--> wrapped_dek_break  (opcional)
```

- **AAD = separación de dominio.** El `subject_id` como AAD del registro ata el blob
  a su seudónimo (no se puede mover un ciphertext de un sujeto a otro). El dominio
  (`"user"`/`"breakglass"`) como AAD de la envoltura impide confundir una envoltura
  con la del otro custodio. Domain-separá **todo** (cada AAD/etiqueta HMAC distinta).
- **Modelo de amenaza:** asumí compromiso TOTAL del store del operador (DB + logs +
  colas). Si las KEK no viven ahí, el dump no revela a nadie. Por eso las KEK NO
  están en la DB ni en el backend.

## 4. Derivar la KEK del usuario: HKDF vs Scrypt según entropía

La KEK del usuario se deriva **client-side** de un secreto que solo el usuario tiene
(firma de wallet CIP-30, llave `R` del extractor de voz, PIN). El backend nunca ve el
factor ni la KEK.

```python
def derive_user_kek(factor_secret, *, salt, low_entropy=False):
    if low_entropy:                          # PIN / password
        return Scrypt(salt=salt, length=32, n=2**15, r=8, p=1).derive(factor_secret)
    return HKDF(SHA256(), 32, salt, info=b"...").derive(factor_secret)   # alta entropía
```

- **Alta entropía** (firma de wallet, `R` del fuzzy extractor) → **HKDF** (rápido,
  suficiente).
- **Baja entropía** (PIN/password) → **Scrypt** (memory-hard) para resistir fuerza
  bruta offline si se filtra el `salt`. Nunca HKDF para un PIN.
- El `salt` es público (por-sujeto), puede vivir en el vault; no revela nada.

## 5. Break-glass de umbral: Shamir sobre GF(256) — y su trampa

La de-anonimización de emergencia se hace con una **KEK de break-glass** partida en
`n` shares; hacen falta `k` para reconstruirla (Shamir Secret Sharing). Los custodios
viven en un **dominio distinto** al KMS del seed de wallets (separación de poderes);
ninguno solo desanonimiza.

**⚠️ Trampa de GF(256): el generador es 0x03, NO 0x02.** Shamir byte-a-byte opera en
GF(2^8) con el polinomio reductor de AES (`0x11b`). Bajo `0x11b`, **2 tiene orden 51**
— NO genera el grupo multiplicativo de 255 elementos, así que las tablas exp/log
quedan incompletas y la interpolación falla en silencio para ciertos bytes. **3 sí
genera el grupo.** Multiplicar por 3 = `x ^ (x<<1)` con la reducción:

```python
x = 1
for i in range(255):
    exp[i] = x; log[x] = i
    x ^= (x << 1)                 # x * 3 en GF(2^8)
    if x & 0x100: x ^= 0x11B
```

Verificá la tabla con un producto conocido (p. ej. `0x53 · 0xCA == 1`) antes de
confiar en el split/combine. Es ~40 líneas puras, sin dependencia nueva.

## 6. Umbral para FIRMAR ≠ umbral para RECUPERAR (hallazgo clave)

No uses el mismo primitivo para los dos casos:

- **Shamir reconstruye la llave en un punto** → solo sirve para el **break-glass**
  (un evento de recuperación puntual, controlado, auditado). Reensamblar la llave en
  cada operación anula la ventaja del umbral.
- **MPC/TSS (FROST-Ed25519) nunca reensambla la llave** → es lo correcto para la
  **firma viva** (cada tx). Fireblocks/ZenGo. Más fuerte, más complejo (rondas,
  nonces, coordinación entre custodios).
- **Native multisig de Cardano** (`atLeast k-of-n`, scripts nativos) → alternativa
  **Cardano-nativa**: umbral **visible on-chain**, sin cripto exótica (el ledger
  valida el `atLeast`); cambia la dirección a script y la forma de la tx. La más
  simple si querés umbral auditable on-chain.

Regla: **Shamir para recuperar, MPC/native-multisig para firmar.** Mantené el KMS del
seed y el custodio de la KEK-break-glass en dominios distintos en cualquier ruta.

## 7. Biometría (voz) → llave: los fuzzy extractors NO son no-vinculables por defecto

Un *fuzzy extractor* convierte una lectura biométrica ruidosa (embedding de voz) en
una llave estable `R` (que alimenta `derive_user_kek`) + un *helper data* público `P`
que corrige el ruido al reproducir.

**⚠️ El hallazgo de mayor impacto:** la construcción **clásica (Dodis-Reyzin-Smith
2004) NO es no-vinculable.** Múltiples enrolamientos de la misma biometría producen
varios `P` que, **combinados, filtran el secreto y son linkables** entre sí (Blanton
et al. SECRYPT'11; Computers & Security 2025). Un `commit(R)` on-chain **no** lo
arregla: la fuga está en `P`, no en el commitment. El SoTA migró a extractors
**reusable / oblivious** (2024).

La no-vinculabilidad se cierra con **al menos una** de:
1. una construcción **reusable/oblivious** para el extractor, y/o
2. apoyarse en la capa **ZK/nullifier de Midnight** (el `commit(R)` entra como hoja
   del árbol y las credenciales se prueban con nullifier por contexto; aunque `P`
   filtre algo, el uso no se correlaciona on-chain — ver `midnight.md`).

Siempre: extracción **client-side** (el backend nunca ve la lectura cruda ni `R`) y
**liveness/anti-replay** (challenge-response fresco, para que un audio grabado o un
`P` filtrado no se reusen).

## 8. Principio: NO inventamos criptografía — separá el *seam* del primitivo

El error más caro es escribir tu propio threshold signing o tu propio fuzzy extractor.
En cambio:

- **Implementá el *seam* (la interfaz) y el default seguro ahora;** dejá el primitivo
  pesado a una **librería auditada** detrás de la misma interfaz.
- Patrón concreto (Python `Protocol`): definí `Signer` (`address` + `sign_tx`) con un
  `SingleKeySigner` real por default y `NativeMultisigSigner`/`MpcSigner` como stubs
  que lanzan `NotImplementedError` con la guía del spike. Igual con `FuzzyExtractor`
  (`enroll`/`reproduce`) + el **contrato de seguridad escrito en el docstring** como
  spec ejecutable, y un test que lo fija con un doble **explícitamente marcado
  inseguro** (nunca un primitivo de juguete que se confunda con producción).
- Migrar a umbral mañana = cambiar la implementación de la interfaz, no reescribir los
  llamadores. Eso es lo que compra el seam.

## 9. Qué reusar (no rodar a mano)

- Simétrico/KDF: `cryptography` (AES-256-GCM, HKDF, Scrypt) — auditada, suele venir
  ya como dep transitiva de pycardano.
- Shamir GF(256): ~40 líneas puras (con la trampa del §5 resuelta) — aceptable sin dep.
- Threshold signing (FROST), fuzzy extractor reusable: **librería vetada**, jamás propio.

## 10. Tests sin red ni secretos (lo que SÍ verificás)

Como en el resto del proyecto (`*_test.py` con runner en `__main__`, sin pytest
obligatorio), ejercitá las **propiedades de ceguera**, no solo el happy path:

- un dump del store NO contiene PII (`b"nombre" not in ciphertext`, ni en
  `wrapped_dek`, ni en las claves del índice — solo HMAC hex de 64 chars).
- la UX resuelve el nombre **solo** con la capability del usuario; sin ella, seudónimo.
- AAD ata cada ciphertext a su `subject_id` y cada envoltura a su dominio (probá que
  una KEK ajena NO abre el registro).
- break-glass `k`-de-`n` desanonimiza; `k-1` no.
- minimización: un sujeto no excede el cap de índices; la PII no-buscable no genera índice.
