# ADR-002: Trazabilidad Blockchain — PostgreSQL + Merkle + Polygon L2

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-002 |
| **Título** | Trazabilidad blockchain: PostgreSQL operativo + hash por boleta + Merkle root por lote anclado en Polygon L2 |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass tiene como diferenciador de producto la **trazabilidad criptográfica pública de cada boleta anclada en blockchain**. Requisitos funcionales:

- F-TB-01: Cada boleta emitida debe tener un hash criptográfico único y verificable.
- F-TB-02: Periódicamente se ancla en Polygon (L2) la raíz Merkle del lote de boletas emitidas.
- F-TB-03: Verificación pública (sin login) de que una boleta pertenece a un lote anclado.
- F-TB-04: Diseño no bloqueante para futura tokenización NFT/reventa.
- R-05: Balance costo/frecuencia de anclaje — si muy frecuente, costo on-chain crece; si muy bajo, la trazabilidad pierde valor inmediato.
- R-09: El modelo de boleta debe ser compatible con tokenización NFT sin migración destructiva.

## Decisión

**Estrategia de trazabilidad en tres capas:**

### Capa 1: PostgreSQL operativo (fuente de verdad)
- `ms-tickets` almacena cada boleta con su `ticket_hash` (SHA-256 del código único + salt por boleta) como columna en la tabla `tickets`.
- El hash se calcula en el momento de emisión y es inmutable.
- PostgreSQL es la fuente de verdad autoritativa del estado de cada boleta y su hash.

### Capa 2: Árbol Merkle por lote (off-chain)
- `fn-blockchain-anchor` (Lambda Go) agrupa hashes de boletas emitidas en el período (5 min o 500 boletas, supuesto S-13).
- Construye un árbol Merkle binario con los hashes como hojas. Calcula la raíz Merkle.
- Almacena en DynamoDB `traceability`: `batch_id`, `merkle_root`, `ticket_hashes[]`, `merkle_tree` (serializado), `polygon_tx_hash`, `status`, `created_at`.
- Para cada boleta del lote, almacena la **prueba de inclusión** (Merkle proof): camino de hashes desde la hoja hasta la raíz.

### Capa 3: Anclaje en Polygon L2 (on-chain)
- Solo la **raíz Merkle** se ancla on-chain (no los hashes individuales). Esto minimiza costo (~$0.001-0.01 por lote en MATIC).
- Smart contract mínimo en Polygon: `function anchorMerkleRoot(bytes32 merkleRoot, uint256 batchTimestamp) external`.
- La transacción exitosa devuelve un `tx_hash` que se almacena en DynamoDB.
- La raíz anclada es **inmutable y públicamente verificable** vía PolygonScan o llamada RPC.

### Verificación pública
Un verificador público (sin login) puede:
1. Ingresar un código de boleta o ticket_hash.
2. El sistema (API pública en `ms-tickets` o lambda dedicada) recupera del DynamoDB la prueba de inclusión para ese hash.
3. Verifica off-chain que el hash está en el árbol (usando la prueba de inclusión) y que la raíz del árbol coincide con la raíz anclada en Polygon.
4. Muestra: "Esta boleta fue emitida legítimamente en el lote #N, anclado en Polygon tx 0x..."

### Compatibilidad con futura tokenización NFT (V2+)
- El `ticket_hash` de cada boleta puede usarse como **token ID** en un ERC-721 o ERC-1155 futuro.
- El diseño del smart contract de anclaje actual no interfiere con un futuro contrato NFT; ambos pueden coexistir.
- La prueba de inclusión Merkle ya está almacenada; para tokenizar, solo se necesitaría un mapping adicional `ticket_hash → owner_address` on-chain.
- **Diseño no bloqueante confirmado:** V1 no impide ni complica la evolución a NFT.

### Frecuencia de anclaje
- Supuesto S-13: cada **5 minutos o 500 boletas**, lo que ocurra primero.
- La Lambda se ejecuta vía EventBridge Scheduler (`rate(5 minutes)`).
- También consume eventos `TicketIssued` vía SQS para acumular. Si en 5 minutos no se alcanzan 500 boletas, se ancla el lote parcial.
- Si se alcanzan 500 boletas antes de 5 minutos, el próximo ciclo programado ancla el lote acumulado.

## Alternativas Consideradas

| Alternativa | Evaluación |
|---|---|
| **Anclar cada boleta individualmente en Polygon** | Costo ~$0.001 por boleta → $1 por cada 1000 boletas, $1000 por millón. No escala. Sin beneficio de agrupación. |
| **Ethereum L1 en lugar de Polygon L2** | Costo $1-10 por transacción. Inviable para anclajes frecuentes. Polygon L2 cuesta ~$0.001-0.01. |
| **Solo Merkle off-chain, sin anclaje on-chain** | No cumple F-TB-03 (verificación pública). La raíz off-chain no es inmutable ni confiable para terceros. |
| **Hyperledger o blockchain privada** | Complejidad operativa alta. No es "públicamente verificable" sin dar acceso a la red privada. Polygon es público y verificable por cualquiera. |
| **IPFS para almacenar el árbol completo** | Complejidad adicional. Polygon solo para la raíz es suficiente; los datos completos están en DynamoDB (off-chain). |

## Consecuencias

**Positivas:**
- Costo mínimo por anclaje (fracción de centavo). 1000 lotes/día = ~$10/día.
- Verificación pública vía PolygonScan sin necesidad de confiar en CriptoPass.
- El hash de boleta es inmutable desde su emisión; no se puede falsificar una boleta sin conocer el salt.
- Diseño preparado para tokenización NFT futura sin migración destructiva.
- DynamoDB como almacenamiento serverless para metadatos Merkle (sin RDS para la lambda).

**Negativas:**
- Latencia de ~5 min entre emisión y trazabilidad pública (aceptable para V1; la verificación en puerta usa la BD operativa, no Polygon).
- Si Polygon L2 tiene congestión, el anclaje puede demorar > 1 minuto (mitigado con reintentos y backoff).
- Dependencia de un proveedor RPC de Polygon (Infura/Alchemy) o nodo propio.

**Riesgos mitigados:**
- **R-05 (Costo de anclaje):** Merkle tree agrupa N boletas en 1 transacción → costo O(1) por lote, no O(N).
- **R-09 (Evolución a NFT):** `ticket_hash` como token ID semilla. Contrato de anclaje actual no interfiere con futuro contrato NFT.
- **EC-14 (Anclaje falla):** Reintentos con backoff. Lote en estado "pendiente". Las boletas son válidas internamente aunque el anclaje esté pendiente.

---

## Notas

- **Polygon testnet (Amoy)** para dev/staging. **Polygon mainnet** para prod.
- La wallet de CriptoPass en Polygon debe tener fondos de MATIC para gas. V1: fondeo manual. V2: automatizado.
- El smart contract de anclaje se auditará antes de deploy a mainnet (es código mínimo pero inmutable).
- La verificación pública se puede ofrecer como una página estática en el portal (`criptopass.com/verify`) que consulta la API de trazabilidad.
