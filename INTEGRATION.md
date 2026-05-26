# LampNet — Integration Guide cho App Team

> File duy nhất cho team OriLife + AladinWork + ProofChat (và app tương tự) tích hợp LampNet.
> Mọi link tới code đều click được dẫn thẳng GitHub.
> Đối tượng: AI Agent + dev của app team.

---

## Mục lục

1. [Bootstrap nhanh](#1-bootstrap-nhanh)
2. [Trạng thái LampNet hiện tại](#2-trạng-thái-lampnet-hiện-tại)
3. [Mô hình tích hợp đề xuất](#3-mô-hình-tích-hợp-đề-xuất)
4. [Phân loại dữ liệu — 3 lớp](#4-phân-loại-dữ-liệu--3-lớp)
5. [API LampNet sẵn dùng](#5-api-lampnet-sẵn-dùng)
6. [Atomic task backlog cho OriLife](#6-atomic-task-backlog-cho-orilife)
7. [Atomic task backlog cho AladinWork](#7-atomic-task-backlog-cho-aladinwork)
8. [Atomic task backlog cho ProofChat](#8-atomic-task-backlog-cho-proofchat)
9. [Encryption client-side](#9-encryption-client-side)
10. [Cardano anchor pattern](#10-cardano-anchor-pattern)
11. [Limitations + workaround](#11-limitations--workaround)
12. [Liên hệ](#12-liên-hệ)

---

## 1. Bootstrap nhanh

### Đối tượng
- AI agent thực hiện task tích hợp song song.
- App developer tự code hoặc review agent output.

### Đọc theo thứ tự (đọc trong 5 phút)
1. §2 Trạng thái LampNet để biết hệ thống làm được gì.
2. §3 Mô hình tích hợp để biết app build cái gì.
3. §6/§7/§8 Backlog cho app của bạn.

### Bootstrap script cho agent

```
Agent: pick 1 task không có dependency open từ §6/§7/§8.
Agent: đọc spec reference (link GitHub trong task).
Agent: implement + test.
Agent: tạo PR theo convention §11.
```

---

## 2. Trạng thái LampNet hiện tại

Repo canonical: [`LampNetCloud/lampnet-hivemind`](https://github.com/LampNetCloud/lampnet-hivemind) — Rust workspace mono-repo.

### 2.1 Đang chạy production

| Capability | Endpoint / Function | Code link |
|---|---|---|
| Upload blob → CID | `POST /mirage/put` | [upload_handler() line 1014](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L1014) |
| Download blob | `GET /:cid` | [lampnet-node.rs](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs) |
| Inspect meta | `GET /v1/inspect/:cid` | [route line 943](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L943) |
| List portals | `GET /v1/portals` | [route line 951](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L951) |
| Telemetry map | `GET /v1/telemetry/map/:cid` | [route line 944](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L944) |
| LT erasure code | encoder/decoder | [lampnet-codec/src/lt_code.rs](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-codec/src/lt_code.rs) |
| BLAKE3 + Bao Merkle | hash + verify | [lampnet-codec/src/hash.rs](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-codec/src/hash.rs) |
| CID v0 (legacy) | `ln1q_<16hex>_<safe_name>` | [gen_cid() line 66](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-codec/src/cid.rs#L66) |
| **CID v2 (Bech32m)** | `lamp://ln1<bech32m>` | [gen_cid_v2() line 78](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-codec/src/cid.rs#L78) |
| Carpet QUIC + ICE | transport | [lampnet-carpet](https://github.com/LampNetCloud/lampnet-hivemind/tree/main/lampnet-carpet) |
| Compass libp2p DHT | discovery | [lampnet-compass](https://github.com/LampNetCloud/lampnet-hivemind/tree/main/lampnet-compass) |
| Rust SDK (low-level) | `LampNetSDK::vaporize` | [lampnet-mirage/src/sdk.rs](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/sdk.rs) |

Public endpoint: `https://lampnet.cloud` (Vietnix primary) + `https://sg.lampnet.cloud` (Contabo).

### 2.2 Có code nhưng chưa wire / kích hoạt

| Capability | Tình trạng | Code link |
|---|---|---|
| `gen_cid_v2` (Bech32m 256-bit) | **CODE CÓ SẴN** nhưng [upload_handler line 1098](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L1098) vẫn dùng `gen_cid` v0 | Wire flag (xem task LampNet team) |
| `verify_chunk` Merkle proof | Có function ở [hash.rs line 86](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-codec/src/hash.rs#L86) nhưng [reconstruct_from_vault](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L2758) không gọi | Wire flag |
| Per-DID namespace `./vault/{did_short}/...` | Spec có, code chưa | Đợi LampNet team |
| Server-side encryption (AES-256-GCM) | Spec có, code chưa | App tự encrypt client-side |
| PhoenixKey JWT auth | Spec có, code chưa | App tự verify DID Ed25519 trong gateway riêng |

### 2.3 KHÔNG có (app tự host trên VPS)

- Mutable database (Postgres / MongoDB / vector DB) — app dùng riêng.
- Real-time pub/sub (Redis / Kafka) — app dùng riêng.
- ML inference + GPU pool — app dùng riêng.
- Long-running daemon hosting — app deploy VPS riêng.
- WebRTC SFU cho stream media — ProofChat dùng Mediasoup riêng.

---

## 3. Mô hình tích hợp đề xuất

App **PHẢI** build gateway service riêng giữa app backend và LampNet:

```
App Mobile/Web
     │
     │ HTTPS
     ▼
App Backend (VPS — Tiger/Contabo/AWS)
     │  - DB, ML, real-time, business logic
     │
     │ HTTP localhost
     ▼
<app>-gateway (Rust)
     │  - Encrypt client-side (AES-256-GCM)
     │  - Verify caller DID (Ed25519)
     │  - Chunk file > 80MB
     │  - Retry + LRU cache
     │
     │ HTTPS
     ▼
LampNet (Vietnix + Contabo portal)
     │
     │ BLAKE3 Merkle root anchor
     ▼
Cardano L1 (NFT CIP-68 datum)
```

### Tại sao app build gateway riêng

| Lý do | Giải thích |
|---|---|
| **Decoupling** | LampNet upgrade API → chỉ gateway sửa, app backend không động |
| **Privacy** | App encrypt client-side TRƯỚC khi gửi — LampNet server không thấy plaintext |
| **Multi-tenant** | App tự handle quota/auth per user trong gateway |
| **Caching** | Gateway giữ LRU local — giảm phụ thuộc LampNet uptime |
| **Auth flexibility** | App verify Ed25519 trực tiếp — không bị ép theo PhoenixKey JWT khi chưa sẵn |

---

## 4. Phân loại dữ liệu — 3 lớp

App PHẢI phân loại data trước khi quyết định lưu ở đâu:

### Lớp A — Lưu LampNet được NGAY (không cần đợi gì)

| Tiêu chí | Ví dụ OriLife | Ví dụ AladinWork | Ví dụ ProofChat |
|---|---|---|---|
| Immutable | 3D mesh, evidence | Job attachment, profile photo | Message archive backup |
| Verify được bằng hash | Cryptographic checksum | File hash | Message thread hash |
| Latency 200-800ms chấp nhận | Archive | Download attachment | Restore history |
| Không nhạy cảm cực đoan | Mesh public, metadata cần encrypt | Profile photo public, doc encrypt | Encrypted ciphertext OK |

→ **Upload NGAY** qua `POST /mirage/put`.

### Lớp B — Lưu LampNet nhưng cần encrypt client-side

| Tiêu chí | Ví dụ |
|---|---|
| Cần encryption per-user (key derive từ DID) | OriLife customer purchase data |
| Cần access control | AladinWork job document share |
| Cần share/revoke linh hoạt | ProofChat group attachment |

→ App encrypt trong gateway TRƯỚC khi upload (xem §9).

### Lớp C — KHÔNG dùng LampNet (giữ trên VPS)

| Tiêu chí | Ví dụ |
|---|---|
| Mutable + query thường xuyên | User profile, job state, tree metadata |
| Real-time push | Chat message, notification |
| Latency < 10ms | Cache, session token |
| Vector similarity | DINO embedding, semantic search |
| Geo query | PostGIS for farm location |

→ Self-host Postgres / Redis / pgvector / PostGIS.

---

## 5. API LampNet sẵn dùng

### 5.1 Upload file

```bash
curl -X POST https://lampnet.cloud/mirage/put \
  -F "file=@evidence.tar" \
  -F "doc_type=orilife.tree_evidence" \
  -F "redundancy=2.0"
```

Response:
```json
{ "cid": "ln1q_a3f4...", "size_bytes": 87654321, "k": 134, "n": 268 }
```

Code: [upload_handler() line 1014](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L1014).

**Khuyến nghị**:
- `redundancy=2.0` (chịu được 50% node fail; default 1.5 cho dev).
- File ≤ 80MB (chunk thủ công nếu lớn hơn — xem §11.1).

### 5.2 Download file

```bash
curl -o evidence.tar "https://lampnet.cloud/ln1q_a3f4..."
```

**Khuyến nghị**: timeout 30s cho file < 10MB, 120s cho file < 100MB. Retry 3 lần exponential backoff. Verify checksum sau khi tải.

### 5.3 Inspect metadata

```bash
curl "https://lampnet.cloud/v1/inspect/ln1q_a3f4..."
```

Response:
```json
{
  "cid": "ln1q_a3f4...",
  "k": 134, "n": 268, "size": 87654321,
  "symbol_size": 65536,
  "doc_type": "orilife.tree_evidence",
  "shard_locations": ["12D3KooW...vietnix", "12D3KooW...contabo"]
}
```

Use case: verify shard có ở ≥ 2 portal trước khi anchor Cardano.

### 5.4 List portal health

```bash
curl "https://lampnet.cloud/v1/portals"
```

Use case: fallback logic, monitoring.

### 5.5 Endpoint CHƯA nên dùng

| Endpoint | Lý do skip |
|---|---|
| `POST /v1/auth/verify` | PhoenixKey JWT chưa wire — gateway tự verify Ed25519 |
| `POST /v1/objects/:cid/share` | AccessControl spec draft, chưa code |
| `DELETE /v1/delete/:cid` | Owner check chưa có ở [line 3281](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L3281) — đừng dùng, treat data immutable |

---

## 6. Atomic task backlog cho OriLife

> Mỗi task = 1 PR. Agent pick task không có dependency open, implement, test, PR.

### Complexity scale

| Mức | Định nghĩa |
|---|---|
| **S** | < 100 dòng code, 1 file, < 5 test |
| **M** | 100-300 dòng, 2-3 file, integration test |
| **L** | 300-800 dòng, nhiều file, E2E test |
| **XL** | > 800 dòng → split |

### T-OR-001 — Scaffold `orilife-gateway` Rust crate

**Complexity**: M | **Dependencies**: none

**Scope**: tạo crate Rust trong [`OriLifeTrace/orilife-infra/`](https://github.com/OriLifeTrace/orilife-infra) (repo này hiện rỗng).

**Files cần tạo**:
```
orilife-infra/lampnet-gateway/
├── Cargo.toml
├── src/main.rs (axum hello world port 8090)
├── Dockerfile
├── docker-compose.dev.yml
└── README.md
```

**Cargo.toml minimal**:
```toml
[package]
name = "orilife-gateway"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.5", features = ["trace", "cors"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
bytes = "1.5"
reqwest = { version = "0.12", features = ["multipart", "json"] }
ed25519-dalek = "2"
blake3 = "1.5"
aes-gcm = "0.10"
rand = "0.8"
hex = "0.4"
base64 = "0.22"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

**Acceptance**:
- [ ] `cargo build` pass không warning.
- [ ] `docker-compose up`, `curl localhost:8090/health` → 200.
- [ ] README có usage section.

---

### T-OR-002 — Implement `LampNetClient` HTTP wrapper

**Complexity**: S | **Dependencies**: T-OR-001

**Scope**: wrap LampNet HTTP API thành Rust struct.

**File**: `orilife-infra/lampnet-gateway/src/lampnet_client.rs` (CREATE).

**Skeleton**:
```rust
use anyhow::Result;
use bytes::Bytes;
use reqwest::multipart;

pub struct LampNetClient {
    base_url: String,
    http: reqwest::Client,
}

impl LampNetClient {
    pub fn from_env() -> Self {
        Self {
            base_url: std::env::var("LAMPNET_BASE_URL")
                .unwrap_or_else(|_| "https://lampnet.cloud".to_string()),
            http: reqwest::Client::builder()
                .timeout(std::time::Duration::from_secs(120))
                .build().unwrap(),
        }
    }

    pub async fn put(&self, data: Vec<u8>, doc_type: &str) -> Result<String> {
        let form = multipart::Form::new()
            .part("file", multipart::Part::bytes(data).file_name("blob.bin"))
            .text("doc_type", doc_type.to_string())
            .text("redundancy", "2.0");
        let resp: serde_json::Value = self.http
            .post(format!("{}/mirage/put", self.base_url))
            .multipart(form).send().await?
            .error_for_status()?.json().await?;
        Ok(resp["cid"].as_str().ok_or_else(|| anyhow::anyhow!("missing cid"))?.to_string())
    }

    pub async fn get(&self, cid: &str) -> Result<Bytes> {
        Ok(self.http.get(format!("{}/{}", self.base_url, cid))
            .send().await?.error_for_status()?.bytes().await?)
    }

    pub async fn inspect(&self, cid: &str) -> Result<serde_json::Value> {
        Ok(self.http.get(format!("{}/v1/inspect/{}", self.base_url, cid))
            .send().await?.error_for_status()?.json().await?)
    }
}
```

**Acceptance**:
- [ ] Integration test gọi `lampnet.cloud` thật, upload 1MB → CID nhận về.
- [ ] Download CID → bytes khớp original.
- [ ] Retry 3 lần khi network fail.

---

### T-OR-003 — Implement client-side encryption AES-256-GCM

**Complexity**: M | **Dependencies**: T-OR-001

**Scope**: encrypt payload với key derive từ farmer DID + master_secret.

**File**: `orilife-infra/lampnet-gateway/src/crypto.rs` (CREATE).

**Skeleton**: xem §9.2.

**Acceptance**:
- [ ] Round trip encrypt(x, did) → decrypt(.., did) → x (1000 random tests).
- [ ] Wrong DID → decrypt fails.
- [ ] Tamper 1 bit → decrypt fails.

---

### T-OR-004 — Implement Ed25519 DID auth middleware

**Complexity**: M | **Dependencies**: T-OR-001

**Scope**: axum middleware verify header `X-OriLife-Signature: <did_b64>.<ts_b64>.<sig_b64>`.

**File**: `orilife-infra/lampnet-gateway/src/auth.rs` (CREATE).

**Acceptance**:
- [ ] Signature hợp lệ → claims extracted.
- [ ] Timestamp > 5 phút → 401.
- [ ] Signature sai → 401.
- [ ] DID không có trong registry → 403.

---

### T-OR-005 — Implement `POST /v1/ingest` endpoint

**Complexity**: L | **Dependencies**: T-OR-002, T-OR-003, T-OR-004

**Scope**: handler nhận multipart, encrypt từng evidence, upload qua LampNetClient, compute Merkle root, trả anchor payload.

**File**: `orilife-infra/lampnet-gateway/src/handlers/ingest.rs` (CREATE).

**Response 200**:
```json
{
  "tree_id": "tree-abc-123",
  "evidence_cids": ["ln1q_a3f4...", "ln1q_b5e6..."],
  "merkle_root": "<hex 64>",
  "anchor_payload": {
    "lampnet_root": "<hex 64>",
    "tree_id": "tree-abc-123",
    "farmer_did": "did:phoenix:..."
  }
}
```

**Acceptance**:
- [ ] Upload 3 file → 3 CID returned, merkle deterministic.
- [ ] File > 100MB → 413 (defer chunking sang T-OR-007).
- [ ] DID không auth → 401.

---

### T-OR-006 — Implement `GET /v1/retrieve/:cid` với LRU cache

**Complexity**: M | **Dependencies**: T-OR-002

**Scope**: download từ LampNet, cache LRU local (capacity tùy chỉnh, default 10GB / TTL 7 ngày), stream response.

**File**: `orilife-infra/lampnet-gateway/src/handlers/retrieve.rs` (CREATE) + `cache.rs` dùng [`moka`](https://crates.io/crates/moka) crate.

**Acceptance**:
- [ ] Download lần 2 → cache hit (metric).
- [ ] Cache full → LRU evict.
- [ ] LampNet down + cache hit → vẫn serve.

---

### T-OR-007 — Chunking file > 80MB transparent

**Complexity**: M | **Dependencies**: T-OR-002

**Scope**: chia file > 80MB thành chunks, upload song song, lưu manifest CIDs list.

**File**: `orilife-infra/lampnet-gateway/src/chunking.rs` (CREATE).

**Acceptance**:
- [ ] File 250MB → 4 chunks → manifest CID.
- [ ] Re-assemble: download chunks → concat → bytes khớp.

---

### T-OR-008 — Implement `POST /v1/anchor` (Cardano tx CBOR)

**Complexity**: L | **Dependencies**: T-OR-005

**Scope**: build unsigned Cardano tx với datum chứa `lampnet_root + tree_id` theo CIP-68.

**File**: `orilife-infra/lampnet-gateway/src/handlers/anchor.rs` + `cardano.rs`.

**Decision cần OriLife team chốt** (option A hoặc B):
- **A**: spawn `cardano-cli` subprocess (cần Cardano node local).
- **B**: dùng [cardano-serialization-lib](https://github.com/Emurgo/cardano-serialization-lib) Rust binding.

Em đề xuất B vì không cần node local.

**Acceptance**:
- [ ] Build tx CBOR với datum đúng schema CIP-68.
- [ ] Mobile (Mesh.js / Lucid) ký được tx này.
- [ ] Submit qua Blockfrost preprod → confirmed.

---

### T-OR-009 — Python client trong `orilife-core`

**Complexity**: S | **Dependencies**: T-OR-005, T-OR-006

**Scope**: thêm `orilife-core/src/integrations/lampnet.py` httpx client gọi gateway.

**File**: [`orilife-core/src/integrations/lampnet.py`](https://github.com/OriLifeTrace/orilife-core) (CREATE).

**Acceptance**:
- [ ] Mock gateway, unit test pass.
- [ ] Integration test real gateway local pass.
- [ ] pytest coverage ≥ 80%.

---

### T-OR-010 — Postgres schema migration

**Complexity**: S | **Dependencies**: T-OR-005

**Scope**: thêm cột vào `trees` table cho LampNet anchor.

**File**: [`orilife-database/migrations/`](https://github.com/OriLifeTrace/orilife-database) (repo rỗng, tạo `002_add_lampnet_fields.sql`).

**SQL**:
```sql
ALTER TABLE trees ADD COLUMN lampnet_root_cid VARCHAR(64);
ALTER TABLE trees ADD COLUMN lampnet_evidence_cids JSONB;
ALTER TABLE trees ADD COLUMN anchor_status VARCHAR(20) DEFAULT 'unanchored'
  CHECK (anchor_status IN ('unanchored', 'pending', 'confirmed', 'failed'));
ALTER TABLE trees ADD COLUMN cardano_tx_id VARCHAR(64);
ALTER TABLE trees ADD COLUMN anchored_at TIMESTAMPTZ;

CREATE INDEX idx_trees_anchor_status ON trees(anchor_status)
  WHERE anchor_status != 'unanchored';
```

---

### T-OR-011 — Update `treeService.py` flow

**Complexity**: M | **Dependencies**: T-OR-009, T-OR-010

**Scope**: mở rộng `create_tree` để upload evidence + build anchor payload.

**File**: [`orilife-core/src/service/treeService.py`](https://github.com/OriLifeTrace/orilife-core) (MODIFY).

---

### T-OR-012 — E2E integration test

**Complexity**: M | **Dependencies**: T-OR-011

**Scope**: pytest scenario full flow create → sign → submit → confirm.

**Scenarios**:
1. Happy path: create tree với 3 evidence → confirm anchor.
2. LampNet portal down → graceful degradation (status `unanchored`).
3. Cardano tx fail → status `failed` + alert.
4. Re-upload same evidence (idempotent) → same CIDs.
5. Verify integrity third-party: pull NFT, compute local root, match.

---

### T-OR-013 — Mobile Cardano sign integration

**Complexity**: M | **Dependencies**: T-OR-008

**Scope**: React Native screen ký Cardano tx từ anchor payload qua [Mesh.js](https://meshjs.dev/) hoặc Lucid.

**File**: [`orilife-mobile-app/src/screens/ConfirmAnchorScreen.tsx`](https://github.com/OriLifeTrace/orilife-mobile-app) (CREATE).

---

## 7. Atomic task backlog cho AladinWork

### T-AW-001 — Scaffold `aladinwork-gateway`

**Complexity**: M | **Dependencies**: none

Tương tự T-OR-001. Path đề xuất: `AladinWork/SmartContract/lampnet-gateway/` (hoặc location AladinWork team chọn).

Customize:
- doc_type prefix: `aladinwork.*`
- DID auth derive từ user account (Aladin DID hoặc App-internal).

---

### T-AW-002 — Implement `POST /v1/upload-attachment`

**Complexity**: M | **Dependencies**: T-AW-001

**Scope**: upload job attachment + profile photo. Optional: thumbnail generation server-side.

---

### T-AW-003 — Implement `POST /v1/anchor-job-receipt`

**Complexity**: L | **Dependencies**: T-AW-002

**Scope**: khi job complete, anchor `(job_id, completion_proof_cids)` lên Cardano escrow datum.

---

### T-AW-004 — Laravel client trong `Aladin-Admin`

**Complexity**: S | **Dependencies**: T-AW-002

**Scope**: PHP Guzzle client gọi gateway. Wrap upload trong admin panel.

**File**: [`AladinWork/Aladin-Admin/app/Services/LampNetService.php`](https://github.com/AladinWork) (CREATE).

---

## 8. Atomic task backlog cho ProofChat

### T-PC-001 — Scaffold `proofchat-gateway`

**Complexity**: M | **Dependencies**: none

Tương tự T-OR-001. Path: `ProofChat/BE/lampnet-gateway/`.

---

### T-PC-002 — Implement `POST /v1/archive-thread`

**Complexity**: M | **Dependencies**: T-PC-001

**Scope**: archive 1 thread chat. Thread đã encrypt qua Signal Protocol → wrap thêm 1 lớp group key (OpenMLS hoặc symmetric per-thread) trước khi upload LampNet để third-party portal không suy luận được pattern.

**Note**: lý do double encrypt vì Signal ciphertexts có metadata structure (header X3DH/Double Ratchet). Wrap lớp ngoài để hoàn toàn opaque với LampNet.

---

### T-PC-003 — Implement `POST /v1/upload-avatar`

**Complexity**: S | **Dependencies**: T-PC-001

**Scope**: upload group/user avatar (binary file).

---

### T-PC-004 — NestJS module trong `BE`

**Complexity**: S | **Dependencies**: T-PC-002

**Scope**: NestJS module wrap gateway HTTP client.

**File**: [`ProofChat/BE/src/lampnet/`](https://github.com/ProofChat) (CREATE).

---

## 9. Encryption client-side

LampNet hiện CHƯA mã hóa data tại rest. App muốn lưu data semi-private PHẢI tự encrypt trước khi upload.

### 9.1 Pattern khuyến nghị (AES-256-GCM)

Trong `crypto.rs` của gateway:

```rust
use aes_gcm::{Aes256Gcm, KeyInit, aead::Aead, Nonce};
use rand::RngCore;
use anyhow::Result;

pub fn encrypt_payload(plaintext: &[u8], owner_did: &str) -> Result<Vec<u8>> {
    let key = derive_owner_key(owner_did)?;
    let cipher = Aes256Gcm::new(&key.into());
    let mut nonce_bytes = [0u8; 12];
    rand::thread_rng().fill_bytes(&mut nonce_bytes);
    let nonce = Nonce::from_slice(&nonce_bytes);
    let aad = format!("app.v1.{}", owner_did);
    let ct = cipher.encrypt(nonce, aes_gcm::aead::Payload {
        msg: plaintext, aad: aad.as_bytes(),
    }).map_err(|e| anyhow::anyhow!("encrypt: {}", e))?;
    let mut out = Vec::with_capacity(12 + ct.len());
    out.extend_from_slice(&nonce_bytes);
    out.extend_from_slice(&ct);
    Ok(out)
}

pub fn decrypt_payload(ciphertext: &[u8], owner_did: &str) -> Result<Vec<u8>> {
    if ciphertext.len() < 12 { return Err(anyhow::anyhow!("too short")); }
    let key = derive_owner_key(owner_did)?;
    let cipher = Aes256Gcm::new(&key.into());
    let (nonce_bytes, ct) = ciphertext.split_at(12);
    let nonce = Nonce::from_slice(nonce_bytes);
    let aad = format!("app.v1.{}", owner_did);
    cipher.decrypt(nonce, aes_gcm::aead::Payload {
        msg: ct, aad: aad.as_bytes(),
    }).map_err(|e| anyhow::anyhow!("decrypt: {}", e))
}

fn derive_owner_key(owner_did: &str) -> Result<[u8; 32]> {
    let master = std::env::var("APP_MASTER_SECRET")
        .map_err(|_| anyhow::anyhow!("APP_MASTER_SECRET not set"))?;
    let context = blake3::derive_key("app.v1.owner.key", master.as_bytes());
    let mut hasher = blake3::Hasher::new_keyed(&context);
    hasher.update(owner_did.as_bytes());
    Ok(*hasher.finalize().as_bytes())
}
```

### 9.2 Key management

| Mức | Key derivation | Backup |
|---|---|---|
| **Phase đơn giản** | `key = BLAKE3.derive(master, did)` | `/etc/<app>/master.key` chmod 600 + Vault |
| **Phase production** | X25519 ECIES với user pubkey từ PhoenixKey | User device keystore + Guardian recovery |
| **Phase tương lai** | Server-side ECIES theo [Encryption.md spec](#) | Bypass khi LampNet ship |

**KHÔNG BAO GIỜ**: commit `.key` vào git, log key ra console, gửi qua kênh không TLS.

---

## 10. Cardano anchor pattern

### 10.1 Flow

```
1. App backend → gateway POST /v1/anchor
   body: { owner_did, content_blob_list, app_meta }

2. Gateway:
   2a. Encrypt mỗi blob → upload LampNet → list CIDs
   2b. Sort CIDs lexically
   2c. Compute lampnet_root = binary_merkle(BLAKE3, sorted_cids)
   2d. Build unsigned Cardano tx CBOR (datum chứa lampnet_root + app_meta)

3. App backend → mobile: trả tx CBOR + anchor_payload

4. User ký tx trên mobile (PhoenixKey hardware key hoặc wallet)

5. App backend → submit tx → Blockfrost preprod

6. Confirmed (~20 phút):
   anchor_status = 'confirmed'
   cardano_tx_id = <hash>
```

### 10.2 Compute Merkle root (Rust, in gateway)

```rust
use blake3;

pub fn compute_merkle_root(cids: &[String]) -> [u8; 32] {
    let mut sorted: Vec<&String> = cids.iter().collect();
    sorted.sort();   // Deterministic
    let leaves: Vec<[u8; 32]> = sorted.iter()
        .map(|c| *blake3::hash(c.as_bytes()).as_bytes())
        .collect();
    binary_merkle_root(&leaves)
}

fn binary_merkle_root(leaves: &[[u8; 32]]) -> [u8; 32] {
    if leaves.is_empty() { return [0u8; 32]; }
    if leaves.len() == 1 { return leaves[0]; }
    let mut next = Vec::with_capacity((leaves.len() + 1) / 2);
    for chunk in leaves.chunks(2) {
        let left = chunk[0];
        let right = chunk.get(1).copied().unwrap_or(left);
        let mut hasher = blake3::Hasher::new();
        hasher.update(&left);
        hasher.update(&right);
        next.push(*hasher.finalize().as_bytes());
    }
    binary_merkle_root(&next)
}
```

### 10.3 Verify integrity (third party)

```
1. Pull NFT từ Cardano → datum.lampnet_root
2. Pull list CIDs từ app backend
3. Local: root_local = compute_merkle_root(CIDs)
4. Assert root_local == datum.lampnet_root → integrity OK
5. Optional: download mỗi CID, decrypt với key (nếu có), verify content
```

---

## 11. Limitations + workaround

### 11.1 File > 100MB
LampNet limit body 100MB. Workaround: chunk + concat. App tự lưu manifest `{chunks: [...], chunk_size}` trong DB.

### 11.2 KHÔNG có DELETE reliable
[delete_handler line 3281](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-mirage/src/bin/lampnet-node.rs#L3281) chưa có owner check. Treat data immutable. KHÔNG upload PII có TTL.

### 11.3 Multi-portal sync chậm (1-5s)
Đợi 5s sau upload TRƯỚC khi anchor Cardano. Hoặc check `/v1/inspect/<cid>` thấy shard trên ≥ 2 portal.

### 11.4 CID v0 sẽ bị thay v2
Hiện upload trả format `ln1q_<16hex>_<safe_name>` ([gen_cid line 66](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-codec/src/cid.rs#L66)). Format v2 stable `lamp://ln1<bech32m>` đã có ở [gen_cid_v2 line 78](https://github.com/LampNetCloud/lampnet-hivemind/blob/main/lampnet-codec/src/cid.rs#L78) nhưng chưa wire vào upload_handler.

**Workaround**: app lưu BOTH `cid_v0` và `cid_v2` (NULL ban đầu) trong DB. Khi LampNet team wire xong → populate v2 + re-anchor Cardano (nếu cần) bằng v2.

### 11.5 KHÔNG có cancel upload
Mobile app handle timeout client-side.

### 11.6 KHÔNG có search/filter
App build index riêng trong DB: `(file_name, cid, uploaded_at, owner_did, doc_type, ...)`.

### 11.7 KHÔNG mã hóa server-side
Gateway lưu plaintext nếu upload không encrypt. **App PHẢI encrypt client-side trước upload cho data semi-private** (xem §9).

---

## 12. Liên hệ

| Vấn đề | Liên hệ |
|---|---|
| Bug LampNet core | Issue [`LampNetCloud/lampnet-hivemind`](https://github.com/LampNetCloud/lampnet-hivemind/issues) |
| Spec / decision | Issue [`LampNetCloud/lampnet-docs`](https://github.com/LampNetCloud/lampnet-docs/issues) |
| Integration help | Issue repo này [`LampNetCloud/lampnet-sdk`](https://github.com/LampNetCloud/lampnet-sdk/issues) |
| Khẩn cấp production | Telegram (channel TBD) |

---

## Branch + PR convention

### Branch naming
```
agent/<task-id>-<short-desc>
ví dụ: agent/T-OR-005-ingest-handler
```

### Commit message
```
feat(<scope>): <description> [T-OR-XXX]

Refs: T-OR-XXX
Co-Authored-By: <agent-name> <noreply@anthropic.com>
```

### PR template
```markdown
## Task: T-OR-XXX
## Spec reference
- lampnet-sdk/INTEGRATION.md §6 T-OR-XXX
## Acceptance checklist
- [ ] (copy từ task)
## Test
- `cargo test --package orilife-gateway`
- Manual curl ...
## Reviewer
@<app-team-lead>
```

---

## Status board

| Task ID | Branch | PR | Status | Reviewer |
|---|---|---|---|---|
| T-OR-001 | — | — | open | — |
| ... | | | | |

(Có thể migrate sang GitHub Project board nếu team dùng.)

---

## Changelog
- v0.1 (2026-05-26): Initial consolidated guide. 13 task OR + 4 AW + 4 PC = 21 atomic tasks.
