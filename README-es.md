## 🛠️ MellonKey◉fs◉ — v2.1.0 — Portfolio Project
> MellonKey◉fs◉, inspirado en la puerta de Moria.

Documento redactado originalmente en español por https://github.com/fabiola-serralunga. Traducido al inglés mediante un modelo de lenguaje grande (LLM) y alojado en README.md

### v2.1.0 *MellonKeyDecrypt* — *The Narya Keepers* - Documentación

>Sucesor de *Mellon-Key-Simple* v1.0.0: misma lógica de mint/verificación, pero el archivo a entregar ya no vive en R2 — está **encriptado con AES-256-GCM en Arweave**, y un **Cloudflare Worker portable** lo busca (con redundancia de gateways), lo desencripta en vuelo y se lo entrega únicamente al holder verificado del SBT.
<br>

🌐 https://mellonkeyfs.pages.dev -- sitio de *MellonKeyDecrypt*

🔗 https://mellonkeyfs.pages.dev/deliver -- sitio de desencriptado y download. El archivo que contiene el digital asset se encuentra encriptado y subido a Arweave. En deliver.html, Cloudflare Worker verifica la posesión del Soulbound Token, desencripta el archivo y lo entrega. Asimismo, visibiliza el Certificado de Adquisición y muestra estado del contrato.

🔗 https://base-sepolia.blockscout.com/address/0x9DF53bB4F3A56b7927560C2638074f55F5DDD20E?tab=tokens_nfts -- para visualizar el SBTs acuñado en el proyecto en su espacio de testnet. En *Cirdan Narya Keeper*, el contrato fue desplegado de tal manera que se pueda observar la imagen vectorizada construida durante el deployment del contrato: cuya base es código vectorizado más impresión de datos del digital asset. 

📜⛓️🔷 Contratos en Solidity siguen el patrón *abstract contract* `SBTDescriptor...` + el contrato que hereda. Por ejemplo, `SBTDescriptorCirdan.sol` + `TestSBTCirdan.sol`. 
Fueron codificados manteniendo en mente el objetivo de generar un token que mientras exista la red donde fue acuñado (en este caso Base Sepolia) siempre conserve y visualice sus metadatos de nombre, descripción, imagen y atributos. 

### Tabla de Contenidos
- [MellonKey◉fs◉](#mellonkeyfs)
- [MellonKeyDecrypt v2.1.0 — *The Narya Keepers*](#mellonkeydecrypt-v210--the-narya-keepers)
- [1. Flujo completo](#1-flujo-completo)
- [2. Redundancia de gateways Arweave (dentro del worker)](#2-redundancia-de-gateways-arweave-dentro-del-worker)
- [3. Portabilidad del worker](#3-portabilidad-del-worker)
- [4. Encriptar y subir el archivo (por cada edición)](#4-encriptar-y-subir-el-archivo-por-cada-edición)
- [5. Deploy del worker y el monorepo](#5-deploy-del-worker-y-el-monorepo)
  - [5.1 Cómo convive `worker/` con los archivos de Pages en `public/`](#51-cómo-convive-worker-con-los-archivos-de-pages-en-public)
  - [5.2 Comandos](#52-comandos)
  - [5.3 Probar en local (VSCode Live Server + wrangler dev) — y el error 405](#53-probar-en-local-vscode-live-server--wrangler-dev--y-el-error-405)
- [6. Contratos en Sepolia: metadata 100 % on-chain](#6-contratos-en-sepolia-metadata-100--on-chain)
  - [6.1 Dentro de ese JSON, los datos críticos están garantizados por construcción:](#61-dentro-de-ese-json-los-datos-críticos-están-garantizados-por-construcción)
  - [6.2 El arte: vectorización controlada](#62-el-arte-vectorización-controlada)
- [7. Un `descriptor.sol` por edición](#7-un-descriptorsol-por-edición)
  - [Dentro de cada descriptor hay dos clases de datos, con distinta naturaleza:](#dentro-de-cada-descriptor-hay-dos-clases-de-datos-con-distinta-naturaleza)
  - [El deploy: contrato abstracto + contrato final](#el-deploy-contrato-abstracto--contrato-final)
- [8. Dar de alta una edición (resumen)](#8-dar-de-alta-una-edición-resumen)
- [9. Seguridad — notas](#9-seguridad--notas)


## MellonKey◉fs◉

MellonKey◉fs◉ es un ecosistema de desarrollo creado como portfolio técnico. Reúne las Pruebas de Concepto (PoC) y los resultados de un proceso de diseño iterativo originado en el Proyecto ·sarocha·. Durante la fase de investigación, este último evolucionó hacia una arquitectura que se ajustaba de forma más eficiente a sus objetivos y alcances finales, permitiendo que MellonKey◉fs◉ permanezca como un valioso repositorio de soluciones, dilemas resueltos y casos de uso transferibles a otros proyectos.

## MellonKeyDecrypt v2.1.0 — *The Narya Keepers*

Sucesor de *Mellon-Key-Simple* v1.0.0: misma lógica de mint/verificación, pero el
archivo a entregar ya no vive en R2 — está **encriptado con AES-256-GCM en Arweave**,
y un **Cloudflare Worker portable** lo busca (con redundancia de gateways), lo
desencripta en vuelo y se lo entrega únicamente al holder verificado del SBT.

```
mellonkey-decrypt/
├── public/                        # Sitio estático — hosteable en CUALQUIER lado
│   ├── index.html                 #   Mint de los 3 Narya Keepers (Base Sepolia)
│   ├── deliver.html               #   Certificado + descarga desencriptada
│   ├── editions.json              #   ÚNICA fuente de verdad (worker la importa al deployar)
│   └── css/                       #   Hojas propias del proyecto
├── worker/                        # Worker portable (reemplaza a functions/api/download.js)
│   ├── src/index.js               #   Toda la lógica: firma, ownerOf, gateways, AES-GCM
│   ├── wrangler.toml              #   Config de deploy
│   ├── package-lock.json          #   
│   └── package.json               #   ethers v6 (transporte fetch, nativo de Workers)
└── README.md
```

---

## 1. Flujo completo

1. El usuario mintea el SBT (`claim()`) en `index.html` — Base Sepolia, 0.02 ETH, 1 por wallet.
2. En `deliver.html` conecta su wallet: la página busca su tokenId on-chain
   (evento `Transfer 0x0 -> wallet` + doble check con `ownerOf()`).
3. Firma un mensaje que liga **edición + tokenId + timestamp** (anti-replay, TTL 15 min).
4. La página envía `{wallet, signature, message, tokenId, edition}` al worker
   (`delivery.apiBase` + `/api/download`).
5. El worker:
   - valida formato y TTL de la firma,
   - recupera la dirección con `verifyMessage` y la compara con `wallet`,
   - consulta `ownerOf(tokenId)` en el contrato de la edición vía RPC,
   - baja el archivo **encriptado** de Arweave siguiendo la cadena de gateways,
   - lo **desencripta** (AES-256-GCM) con una key en secret,
   - responde el binario plano con `Content-Disposition: attachment`.
6. El navegador guarda el archivo con el `downloadName` de la edición.

La clave NUNCA viaja al cliente: el desencriptado ocurre íntegramente en el worker.

## 2. Redundancia de gateways Arweave (dentro del worker)

`editions.json` ya no hardcodea ningún gateway: ni el archivo entregable
(campo `arweaveTx`) ni la imagen de preview (campo `image`) referencian un
gateway — ambos usan el esquema `ar://TXID`. El worker resuelve el gateway
del archivo así (e `index.html` usa la MISMA cadena, en el navegador, para
resolver las imágenes de las cards):

| Orden | Origen | Configuración |
|-------|--------|---------------|
| 1° | Gateway canónico `https://arweave.net` | constante de última instancia (el valor manda en `delivery.canonicalGateway`) |
| 2° | Hasta **3 gateways de fallback probados** | `delivery.fallbackGateways` (los vacíos se saltean) |
| 3° | **Última instancia**: API en Vercel (AR.IO SDK, lista ya filtrada por performance) | `delivery.resolverApi` → se toma el **primero** |

**Imágenes en el navegador (`image: "ar://TXID"`)**: ningún navegador entiende
`ar://`, así que `index.html` resuelve cada `<img>` en cliente con el mismo orden: gateway canónico → fallbacks → `resolverApi`. El mecanismo es
intercambio de `src` ante error (`onerror`), o sea carga `<img>` pura, sin CORS.
Una URL `https` legada en `image` sigue funcionando tal cual (compatibilidad
hacia atrás). Funciona desde cualquier hosting, incluido el clon en Arweave.

Detalles de robustez ya implementados:

- El resolver de Vercel se consulta **solo** si canónico y fallbacks fallan (lazy).
- Cada intento tiene timeout propio (`delivery.gatewayTimeoutMs`, default 10 s).
- Un body sospechosamente chico (páginas de error con HTTP 200) se descarta.
- Si una copia se obtiene pero **no desencripta**, la cadena sigue con el próximo
  gateway; solo al agotarse todo se devuelve el error (502 si no se obtuvo nada,
  500 si se obtuvo pero no desencripta).

## 3. Portabilidad del worker

El worker **no depende del hosting de la página**:

- `editions.json` → `delivery.apiBase`:
  - `""` → misma origen (el worker está en una ruta del mismo dominio, ej. Pages).
  - `"https://mellonkey-decrypt.00k.workers.dev"` → la página puede
    estar hosteada donde sea (Pages, localhost, **un clon servido desde Arweave**).
- CORS abierto por defecto (la autenticación es por firma de wallet, no hay
  cookies ni sesiones). Para restringirlo: secret `ALLOWED_ORIGINS` con orígenes
  separados por coma.
- `GET /` del worker responde un health check JSON (ediciones activas, si hay clave).

> Sobre un clon en Arweave las consideraciones resultan obvias pero cabe insistir en ellas: en web2 `editions.json` se actualiza con cada nueva edición ("Gil-Galad Narya Keeper", "Cirdan Narya Keeper", etc), se buildea y se hace redeploy automático del worker, mientras en Arweave quedará congelado en un TXID. La configuración **autoritativa para la entrega** quedará bundlada en el worker al deployar y el worker buscará un TXID, no un `editions.json`. Cada nueva edición implica actualizar el TXID y redeployar el worker para que el clon siga sirviendo esa entrega. 

## 4. Encriptar y subir el archivo (por cada edición)

```bash
# Encripta y genera clave nueva 
node tools/encrypt-for-arweave.mjs mi-archivo.png

# O usa una clave propia (openssl rand -hex 32)
node tools/encrypt-for-arweave.mjs mi-archivo.png -k 3f2a...64-hex-chars...9b1c

# Verifica integridad antes de subir 
node tools/encrypt-for-arweave.mjs --verify mi-archivo.png.enc -k <clave> mi-archivo.png
```

Formato del blob (idéntico al que el worker desencripta con WebCrypto):

```
[ IV: 12 bytes ][ ciphertext: N bytes ][ GCM tag: 16 bytes ]
```

Luego, por ahora de forma manual (se está trabajando en automatización):

1. Aloja el `.enc` en Arweave **sin modificarlo** (arweave.app, Irys, etc.).
2. En `editions.json`, carga la edición:
   - `"contract"`: address del SBT deployado,
   - `"arweaveTx": "ar://<TXID>"`,
   - `"active": true`.
3. Configurá el secret del worker (una vez por clave):
   `wrangler secret put ***_KEY` con 64 hex chars.
4. `wrangler deploy` (desde `worker/`) — el worker **importa editions.json al
   buildear**, así que después de editarlo siempre se redeploya. 

## 5. Deploy del worker y el monorepo

### 5.1 Cómo convive `worker/` con los archivos de Pages en `public/` 

Son dos piezas separadas y cada una se publica por su lado:

- **`public/` es el SITIO** (`index.html`, `deliver.html`, `editions.json`,
  `css/`). Eso es lo que se publica en el hosting — por ejemplo, en Cloudflare
  Pages, el output directory del proyecto apunta a `public`.
- **`worker/` es el BACKEND** (reemplaza a `functions/api/download.js`). NO se
  sube al hosting de la página: se deploya aparte con wrangler y queda vivo en
  `https://mellonkey-decrypt.00k.workers.dev`. La página lo encuentra vía `delivery.apiBase` de `editions.json`:
  `""` = mismo origen, o la URL completa del worker = página hosteada donde se prefiera: Cloudflare, Vercel, Arweave, etc.

**Node: se instala UNA vez en el sistema** (de nodejs.org, versión LTS). Dentro
de `worker/` no se "instala node": ahí solo corre `npm install` la primera vez,
que descarga las dependencias del proyecto (wrangler + ethers) a `node_modules/`
dentro de esa carpeta.

**La estructura relativa importa**: el worker IMPORTA `../public/editions.json`
al buildear, así que `worker/` debe quedar en la misma jerarquía de
`public/`. Dado que la actualización se realiza vía un repositorio en Github lo más limpio es conservar en la raíz ambas carpetas, en la misma jerarquía:


### 5.2 Comandos

```bash
cd worker
npm install
wrangler login                        # primera vez
wrangler secret put ***_KEY           # requerido: 64 hex chars (32 bytes)
wrangler secret put ALCHEMY_API_KEY   # opcional: RPC premium; sin él se usa sepolia.base.org
wrangler deploy
```

Pruebas locales → ver la sección 5.3.

### 5.3 Probar en local (VSCode Live Server + wrangler dev) — y el error 405

Dos servidores corren en paralelo, y `editions.json` tiene que apuntar al worker:

```bash
cd worker
npx wrangler dev          # terminal 1: worker en http://localhost:8787
# terminal 2: servir el sitio (por ejemplo, Live Server de VSCode, o:)
npx serve public          # o cualquier server estático
```

En `editions.json` (el que sirve la página):

```json
"delivery": { "apiBase": "http://localhost:8787" }
```

- Con `""` NO funciona en local dado que el html/css/js iría al server estático
  (Live Server :5500), que a un POST responde **405 Method Not Allowed** con
  body vacío — y el navegador muestra `Unexpected end of JSON input`.
  Si el POST no aparece en la consola como `:8787/api/download`,
  `apiBase` es incorrercto.
- Verificación rápida del worker: `http://localhost:8787/health` → JSON
  con `"keyConfigured": true` confirma que `.dev.vars` cargó la clave.
- `.dev.vars` vive en `worker/`, junto a `wrangler.toml`, formato `CLAVE=valor`
  sin comillas ni espacios. 
- Si  `ALLOWED_ORIGINS` en local, se suman los orígenes de la página:
  `http://127.0.0.1:5500,http://localhost:5500`.
- La MISMA `editions.json` alimenta ambos lados (la página la lee por HTTP y el
  worker la importa al buildear): tras editarla, `wrangler dev` recarga solo.
- Al pasar a producción: `apiBase` se configura con la URL del worker
  (`https://mellonkey-decrypt.00k.workers.dev`); `""` solo si el
  worker corre en el MISMO dominio que la página.

**Si el POST ya llega al worker pero falla la descarga (502 "Could not fetch
the encrypted file from any Arweave gateway")**, la terminal `wrangler dev` (`[gateway-chain] ...`) explicita las causas: 

- **timeout en algún fallback** — ese gateway está lento o caído desde la red.
  La cadena sigue con el próximo candidato.
- **HTTP 404 en un fallback** — ese gateway no tiene indexado ese TXID (los
  AR.IO no replican todo al instante). Sigue la cadena.
- **timeout del `[ar-io-resolver]`** — la API en Vercel tardó más que
  `gatewayTimeoutMs` (default 10 s). Es el último recurso: si el canónico
  funcionó, nunca se la consulta. El timeout puede setearse con
  `"gatewayTimeoutMs": 20000` en `delivery`.

**Si los gateways descargan el archivo pero todos dicen `undecryptable blob`**,
el sobre llegó bien y lo que falla es la apertura. Dos causas posibles:

- **La clave no es la misma** con la que se encriptó el archivo (o tiene algún
  carácter de más/menos al pegarla). La clave es una K de 64 hex chars — con o sin `0x`, el código resuelve ambas posibilidades, a prueba de un desarrollador distraído.
- **El formato del contenedor**: desde v2.0.3 el worker auto-detecta los dos
  formatos soportados y elige el correcto solo:
  - `SRCHENC1` — toolchain (`gen-master-key.mjs` + `encrypt-artwork.mjs`):
    `["SRCHENC1" 8B][versión 1B][reservados 3B][IV 12B][ciphertext+tag]`
  - `RAW` — `encrypt-for-arweave.mjs` del proyecto: `[IV 12B][ciphertext+tag]`
  En ambos el interior es el mismo AES-256-GCM (IV de 12 bytes, tag GCM al
  final). La terminal del worker loguea qué contenedor detectó
  (`[decrypt] container: ...`), así que si algún día apareciera un tercero,
  se ve al instante.

El error final que llega a `deliver.html` lista los intentos con su
diagnóstico (ej. `Attempts: https://arweave.net -> HTTP 403; ...`).

## 6. Contratos en Sepolia: metadata 100 % on-chain
Cada SBT se describe a sí mismo sin depender de nada externo. El tokenURI no apunta a IPFS, a Arweave, ni a ningún servidor: es un data:application/json;base64,... que el contrato construye por sí solo, cada vez que un explorador lo pide. El SVG vive como constante interna del contrato: el arte es bytecode. Si el sitio cae, si el hosting desaparece, el token sigue mostrando lo suyo.

Los contratos fueron codificados manteniendo en mente el objetivo de generar un token que mientras exista la red donde fue acuñado (en este caso Base Sepolia) siempre conserve y visualice sus metadatos de nombre, descripción, imagen y atributos. 

Eso implica evitar la dependencia de un metadata.json externo que adolece de al menos estos potenciales fallos: 
- si es incorporado hardcodeando gateways para que sea leído por cualquier plataforma estos pueden desaparecer, y si no hay quien actualice los setters del contrato se perderá su contenido 
- si son incoporados con esquemaUri (especialmente el ar:// que por caso no es leído por Metamask, aunque ipfs:// es más aceptado), pueden no ser legibles por ciertas plataformas
- si se elige IPFS:// -que es ampliamente aceptado por diferentes plataformas- obliga al creador a sostener cuentas onerosas que mantengan sus archivos vivos en los nodos. Esta desarrolladora entiende que nada garantiza que servicios gratuitos sigan existiendo ni siquiera mañana mismo: situación reñida con la permanencia de los datos.
- si se elige ar:// no es legible por diversas plataformas. 

### 6.1 Dentro de ese JSON, los datos críticos están garantizados por construcción:

- name — "<nombre del activo> #<tokenId>", generado con el ID del token.
- de scription — armada por el contrato con el nombre del activo, el creador y la colección, más la URL de reveal.
- image / image_data — el arte se expone en dos formatos a la vez: como data:image/svg+xml;base64 (estándar ERC-721, lo que leen wallets, marketplaces y exploradores) y como SVG crudo en image_data (convención de arte on-chain). Dos puertas para que cualquier visor encuentre siempre la imagen. Además, el propio SVG lleva impreso nombre, colección y creador.
- No existe un estado en el que el token se muestre sin nombre, sin descripción o sin imagen: no hay ningún archivo externo que se pueda perder.

### 6.2 El arte: vectorización controlada
La imagen del token es la vectorización de una imagen estrictamente relacionada con el activo digital que se entrega: la cara del token anticipa aquello que se recibirá al reclamarlo.

Para generarla se programó un script en Python (`generador_svg_sol.py`) que hace el camino completo: reduce y cuantiza la imagen a 4 colores (K-Means), la vectoriza con vtracer y entrega un SVG de entre 4 y 6 KB, un `file.svg` para que el creador observe cómo se visualizará. Asimismo, entrega un `file.txt` con reglas: todos los atributos van con comillas simples, porque el SVG se incrusta crudo dentro del JSON del contrato; antes de escribir el archivo final, simula el JSON exacto que producirá el contrato y lo valida — si algo no cierra, el archivo .txt no se genera.

Ese tamaño es la clave operativa: el contrato compilado ocupa ~8 KB de bytecode frente al límite de 24,5 KB de EIP-170 (unos 16 KB de margen), así que el deploy sale sin dificultades.

## 7. Un `descriptor.sol` por edición
Cada edición tiene su propio descriptor: un contrato Solidity dedicado que porta exclusivamente su identidad y su memoria visual — el nombre del activo, el creador, la colección, el SVG vectorizado como constante, y los TXIDs de Arweave de esa edición. El descriptor de Gil-Galad no comparte una sola línea de arte con el de Cirdan ni con el de Mithrandir: tres descriptores, tres identidades, tres SVG horneados en tres bytecodes distintos. Las ediciones son impermeables entre sí: regenerar o corregir el arte de una no toca a las demás.

### Dentro de cada descriptor hay dos clases de datos, con distinta naturaleza:

- Constantes de bytecode (el arte y la identidad): digitalAssetName, creatorName, collectionName y el SVG base. Inmutables, sin costo de storage, leídas directo del código.
- Datos de despliegue en storage: los TXIDs de Arweave (manifiesto de animación, archivo cifrado), que se pasan en el constructor y cuentan con setters.
- El descriptor arma además el JSON completo del token: name, description, image, image_data, animation_url, external_url, attributes y encrypted_media — todo el certificado, no solo la imagen.

### El deploy: contrato abstracto + contrato final

El descriptor se escribe como abstract contract: no se puede deployar por sí solo — el compilador lo rechaza. Es la mitad de una pieza que solo existe unida a la otra. Encima se escribe el contrato que sí se deploya, que hereda del descriptor y aporta la maquinaria del token:

solidity

contract SBTCirdan is SBTDescriptorCirdan, ERC721, Ownable, Pausable, ReentrancyGuard {
    // claim() con pago directo al owner, soulbound enforcement, pausa,
    // control de reentrancia, registro de fechas de mint...
}

Al compilar, Solidity funde ambos en un único bytecode: descriptor (arte + identidad + metadata) y lógica (mint, soulbound, seguridad) quedan fusionados en el mismo contrato desplegado. No hay llamadas externas para mostrar el metadata, no hay proxies, no hay piezas que se puedan desincronizar.

La consecuencia práctica es elegante: la maquinaria del token es idéntica entre ediciones; lo que cambia es el alma — y el alma viaja en el descriptor. Una misma plantilla de contrato, tres identidades irrepetibles. 


## 8. Dar de alta una edición (resumen)

1. Deploy del contrato SBT por cada edición abierta. Se importa el address en editions.json.
2. `editions.json`: `contract`, `arweaveTx: "ar://<TXID del archivo encriptado>"`, `image: "ar://<TXID de la imagen>"`, `active: true`, `downloadName`.
3. `wrangler deploy`.
4. Si el sitio vive en otro hosting (ej. clon Arweave): se actualiza también el
   `editions.json` del clon (o se resube) para que la card deje de decir *Coming soon*.

## 9. Seguridad — notas

- La `***_KEY` vive solo en los secrets del worker; el cliente jamás la recibe.
- La firma del mensaje vale 15 minutos y liga edición + tokenId: sirve contra replay.
- La entrega exige `ownerOf(tokenId) == wallet` on-chain en el contrato de ESA edición.
- El token es soulbound (`_update` bloquea transfers): la llave se puede mintear y quemar.
- Roadmap anotado: fallback de workers (múltiples workers espejo) y lógica de
  fallback adicional para la entrega del archivo.

*Stay Human... Be Kind* 🌿

---------------------------------------------------------------------------------

Así se visualiza cada token dentro de una billetera MetaMask gracias a la arquitectura NFT ó SBT 100% en cadena (Fully On-Chain).

<img width="591" height="828" alt="2026-10-12 10_35_37-gilgalad-metamask" src="https://github.com/user-attachments/assets/2b7a3db2-a62e-4bb1-b8c7-5c0c317c320a" />

<img width="586" height="814" alt="2026-10-12 10_36_54-cirdan-metamask" src="https://github.com/user-attachments/assets/ad4e1648-dd4f-4c3a-b07b-7847742a92a0" />

<img width="589" height="815" alt="2026-10-12 10_37_29-mithrandir-metamask" src="https://github.com/user-attachments/assets/695628ef-a11c-4b27-8bf4-2a61929aca5e" />
