# Hướng dẫn tích hợp LampNet

Tài liệu này dành cho đội app muốn lưu dữ liệu lên LampNet hoặc cho thiết bị của người dùng tham gia góp tài nguyên. Mọi tham chiếu trỏ bằng **tên route** hoặc **tên hàm SDK**, không trỏ số dòng.

Những phần ghi **"đang làm"** chưa có trong mã đang phát hành; đừng dựa vào chúng khi lập kế hoạch tích hợp.

---

## 1. Tổng quan — ba lối vào

| Lối vào | Dùng khi | Chạy ở đâu |
|---|---|---|
| **HTTP API của node** | Tải lên, tải xuống, xoá, tra cứu tài liệu; gửi yêu cầu kết nạp thiết bị; các vòng góp tài nguyên | Server của app hoặc trực tiếp từ client |
| **SDK di động `lampnet-mobile-sdk`** (UniFFI, Kotlin/Swift) | Sinh khoá thiết bị, đo cấu hình máy, dựng yêu cầu kết nạp đã ký, ký các yêu cầu góp tài nguyên, niêm và mở dữ liệu phía client | Trong app Android/iOS |
| **Thư viện Rust phía client** (cấp theo yêu cầu) | App hoặc gateway viết bằng Rust cần tự niêm dữ liệu; liên hệ đội LampNet để được cấp quyền | Tiến trình của app |

Nguyên tắc chung: **SDK không gọi mạng**. Các hàm SDK chỉ tính toán và ký; app tự gửi kết quả lên HTTP API. Nhờ vậy app giữ toàn quyền về lớp mạng (proxy, thử lại, timeout).

---

## 2. HTTP API của node

### 2.1 Quy ước chung

- Thân request tối đa mặc định 100 MB. File lớn hơn dùng **phiên tải lên** (mục 2.3). Phiên tải lên có trần riêng, mặc định 256 MiB mỗi phiên, đổi theo cấu hình nút; vượt trần thì lệnh mở phiên trả `413`.
- Mỗi request có timeout 300 giây. Hết giờ thì node trả `408`.
- Một số route công khai có giới hạn tần suất theo IP. Vượt giới hạn thì node trả `429` kèm header `Retry-After`; một số route tra cứu khi quá tải trả `503` kèm `Retry-After`. App vẫn phải xử lý các mã này ở mọi route.
- Định danh tài liệu (CID) có dạng `ln1q_…` hoặc `ln1…` (bech32). `GET /<cid>` nhận cả tiền tố `lamp://`; các route khác dùng CID trần.

### 2.2 Tải lên một lần — `POST /mirage/put`

Xác thực: **Bearer** (mục 3.1). Thân `multipart/form-data`:

| Trường | Bắt buộc | Ý nghĩa |
|---|---|---|
| `file` | có | Nội dung. File rỗng → `400`. |
| `doc_type` | không | Nhãn loại tài liệu do app đặt. |
| `redundancy` | không | Mức dư thừa mong muốn; node kẹp vào khoảng cho phép. |
| `privacy` (hoặc `data_class`) | không | Cấp riêng tư, xem mục 4. Giá trị lạ → `400`. |
| `password` | tuỳ cấp | Bắt buộc với cấp niêm nội dung (`private`, `shielded`); thiếu → `400`. |

Tham số `?mode=` được bỏ qua: mọi object mới ghi theo chế độ đối xứng.

Mặc định node trả CID dạng văn bản thuần. `?format=json` trả JSON; `cid`, `size_bytes`, `k`, `n` luôn có. Các trường `shards_placed_local`, `shards_scheduled_remote`, `shards_confirmed_remote`, `distribution_status`, `distribution_pending_peers`, `distribution_mode`, `distribution_note` có thể vắng: tải lại object đã có thì không có `shards_confirmed_remote`, và không còn peer chờ thì không có `distribution_pending_peers`. `shards_scheduled_remote` là kế hoạch, không phải xác nhận.

Mã lỗi chính: `400` (thiếu file, file rỗng, cấp riêng tư sai, thiếu mật khẩu), `401`/`403` (xác thực), `413` (lỗi đọc thân tải lên, gồm cả vượt trần), `503` (node tạm không kiểm được vé PhoenixKey).

### 2.3 Phiên tải lên (file lớn, tải tiếp được)

Có hai cách mở phiên, sau đó dùng chung các bước gửi mảnh và chốt:

| Route | Phương thức | Xác thực | Việc làm |
|---|---|---|---|
| `/mirage/session/open` | POST | Bearer | Mở phiên |
| `/v1/session/open` | POST | Chữ ký DID (mục 3.2) | Mở phiên |
| `/mirage/session/:sid/chunk` · `/v1/session/:sid/chunk` | POST | Chủ phiên | Gửi một mảnh tại `?offset=` |
| `/mirage/session/:sid/finalize` · `/v1/session/:sid/finalize` | POST | Chủ phiên | Chốt phiên, gửi kèm `blake3` (hex 64 ký tự) của toàn bộ nội dung |
| `/mirage/session/:sid` | GET | Chủ phiên | Xem trạng thái phiên |
| `/mirage/session/:sid` | DELETE | Chủ phiên | Huỷ phiên |

`/mirage/session/open` nhận JSON có `total_size` bắt buộc; các trường tuỳ chọn gồm `doc_type`, `content_type`, `device_class`, `storage_class`, `bandwidth_bps`, `latency_ms`, `retention_days`, `redundancy`, `magic_balance`. `total_size = 0` → `400`.

`/v1/session/open` nhận JSON `{total_len, bucket, n_chunks, content_type, doc_type}` và trả `{session_id, session_cap, expires_at, cap_exp, shard_size, thread_count, per_shard_ceiling}`. Các request tiếp theo của phiên gắn `session_cap` vào header `x-lampnet-session-cap`. Cap sống tối đa 10 phút, trong khi phiên sống 2 giờ (trần 12 giờ), và không có đường gia hạn cap: quá 10 phút thì mảnh và lệnh chốt trả `401`, phải mở phiên mới. Mỗi mảnh không vượt `per_shard_ceiling` (trần cứng 16 MB).

Nếu node chưa cấu hình bộ phân giải DID, `/v1/session/open` trả `503`.

Phiên tải lên không có trường `privacy` hay `password`, và node không niêm nội dung phiên. Muốn riêng tư thì niêm phía máy trước khi tải. Object qua phiên DID mà thiếu cấp được cổng đọc coi là chặt nhất: chỉ chủ đọc được.

Mã lỗi của phiên: `401` thiếu xác thực · `404` phiên không tồn tại · `403` không phải chủ phiên · `410` phiên quá hạn · `400` offset ngoài phạm vi hoặc băm không khớp · `413` mảnh hoặc tổng dung lượng quá lớn · `409` chốt khi chưa nhận đủ · `402` không đủ số dư dịch vụ · `429` quá nhiều phiên đồng thời · `507` kho tạm đầy.

### 2.4 Tải xuống — `GET /<cid>`

Node phục vụ tệp tĩnh trước; đường dẫn không khớp tệp tĩnh nào mới được coi là CID.

- Object có chủ cấp `public`: ai có CID đều tải được.
- Object có chủ cấp khác, hoặc bản ghi thiếu cấp: phải chứng minh DID của chủ (mục 3.2). Không gửi chứng minh → `404`, cùng mã với CID không tồn tại. Chữ ký DID sai → `401`. Node không đọc được sổ chủ → `503`.
- Object không có chủ (`legacy:unowned`): ai có CID đều tải được.
- CID sai định dạng → `404`. Vượt giới hạn tần suất → `429` kèm `Retry-After`.
- Object do node niêm (cấp `private` hoặc `shielded`, tải lên kèm `password`): node trả bản đã niêm. Cấp `public` kèm `password` thì node bỏ qua mật khẩu và lưu bản rõ, không báo lỗi. SDK di động không có hàm mở bản này, định dạng phong bì không được đặc tả, và `moc_open` không đọc được. App cần đọc lại trên điện thoại thì tự niêm bằng `moc_*` (mục 4.3) trước khi tải lên.

Quyền đọc theo cấp riêng tư đã chạy ở node. Scope `lampnet.read` chưa được kiểm ở đường đọc.

### 2.5 Xoá — `DELETE /v1/delete/:cid`

Chỉ chủ sở hữu (chứng minh bằng chữ ký DID) được xoá. Object không có bản ghi chủ, hoặc là `legacy:unowned`, thì **bị từ chối** (fail-closed), trả `404`. Chữ ký DID hợp lệ nhưng không phải chủ cũng nhận `404`, cùng mã với CID không tồn tại. Mã lỗi: `400` CID sai định dạng · `401` thiếu hoặc sai xác thực · `404` · `429` quá giới hạn tần suất · `503` object đang bận hoặc node tạm không kiểm được DID, thử lại · `500`.

### 2.6 Tra cứu công khai

| Route | Phương thức | Việc làm |
|---|---|---|
| `/v1/inspect/:cid` | GET | Thông tin một object |
| `/v1/documents` | GET | Danh sách tài liệu |
| `/v1/state` | GET | Trạng thái node |
| `/v1/node/stats` | GET | Số liệu node |
| `/v1/peer_id` · `/v1/signaling_config` · `/v1/portals` | GET | Thông tin kết nối |
| `/health` | GET | Kiểm sống |

Một số route tra cứu có thể trả `429` hoặc `503` kèm `Retry-After`; gặp thì app phải tôn trọng header này.

Lưu ý: `/v1/state` báo số peer mà **một** node thấy, không phải cỡ toàn mạng.

### 2.7 Kết nạp thiết bị (join)

| Route | Phương thức | Xác thực | Việc làm |
|---|---|---|---|
| `/v1/join/challenge` | GET | không | Lấy `{nonce_hex, ttl_slot, issued_at_slot, verifier_did}`. `503` = node tạm không nhận yêu cầu kết nạp; thử lại sau (phản hồi này không kèm `Retry-After`). Thân có thể có thêm `attest_ext`; node chưa trả trường này thì coi như mảng rỗng. |
| `/v1/join/v2/request` | POST | Chữ ký khoá thiết bị nằm trong thân | Gửi nguyên văn `request_json` do SDK dựng. Lần đầu trả `{decision, score, initial_rep, tier, phase}`. Thiết bị đã là thành viên trả `{registered, tier, phase, decision}`, không có `score` và `initial_rep`. |
| `/v1/join/status/:device_pubkey_hex` | GET | Header ký (mục 3.3) | Trả `{registered, tier, phase}`. |
| `/v1/device/register` | POST | Chữ ký trong thân **và** 4 header DID (mục 3.2) | Gắn khoá thiết bị vào DID. |

Lỗi của `/v1/join/v2/request` là JSON `{"error": <mã>, "detail": …, "retry_after_secs"?: …}`:

| HTTP | Mã |
|---|---|
| 400 | `malformed_json`, `invalid_subject_did`, `invalid_cardano_address`, `did_mismatch` |
| 401 | `bad_challenge`, `did_proof_rejected` |
| 409 | `duplicate_fingerprint`, `did_bound_to_other_key`, `key_bound_to_other_did`, `did_key_cap_exceeded` |
| 422 | `field_too_long`, `bad_attestation`, `invalid_hw_gpus` (node cũ chưa trả mã này) |
| 500 | `member_state_missing` |
| 429, 503 | Node tạm không nhận; thử lại sau, theo `retry_after_secs` hoặc `Retry-After` nếu có. `429` từ giới hạn theo IP là văn bản thuần, kèm `Retry-After`, không có thân JSON |

`/v1/device/register` nhận `{did, device_pubkey, timestamp, signature}` (cách ký ở mục 3.3), trả `200` với `status` là `registered` hoặc `already_bound`, kèm `device_count` và `note`. Lỗi: `400` thân sai hoặc `did_mismatch` · `401` chữ ký DID sai · `409 device_bound_to_different_did` · `422 invalid_signature` · `429 per_did_cap_exceeded` và `429 total_cap_exceeded` không nên thử lại; `429` khác thì thử lại sau · `503` node chưa cấu hình phân giải DID (`resolver_not_did_specific` là lỗi cấu hình, không tự hết khi thử lại).

Tích hợp kết nạp dùng `/v1/join/v2/request`.

### 2.8 Góp tài nguyên từ điện thoại

Các route `POST` dưới đây đòi chữ ký khoá thiết bị trong thân; `GET /v1/mobile/rewards/:device_pubkey` thì không. App dựng thân bằng các hàm của SDK ở cột cuối.

| Route | Phương thức | Việc làm | Hàm SDK dựng thân |
|---|---|---|---|
| `/v1/mobile/lease` | POST | Xin một việc tính toán | `build_lease_request` |
| `/v1/mobile/payload/:lease_id` | POST | Lấy dữ liệu của việc | `build_payload_fetch_request` |
| `/v1/mobile/report` | POST | Nộp kết quả | `report_receipt` |
| `/v1/mobile/storage/offer` | POST | Đề nghị giữ dữ liệu | `build_storage_offer` |
| `/v1/mobile/storage/shard/:lease_id` | POST | Tải mảnh được giao | `build_shard_fetch_request` |
| `/v1/mobile/storage/challenge/:lease_id` | POST | Xin câu hỏi kiểm chứng | `build_audit_challenge_request` |
| `/v1/mobile/storage/proof` | POST | Nộp lời đáp kiểm chứng | `answer_storage_audit` |
| `/v1/mobile/rewards/:device_pubkey` | GET | Xem công tích luỹ: `{device_pubkey_hex, units, storage_units, nanothread}`. Khoá chưa có công, kể cả khoá lạ, trả các số `0` | — |

`/v1/mobile/storage/offer` đòi khoá thiết bị đã gắn DID, không thì `403 device_not_bound`. `/v1/mobile/lease` cũng trả `403 device_not_bound` khi node có bộ phân giải theo từng DID và khoá chưa gắn. SDK không gửi bằng chứng DID khi kết nạp, nên phải gọi `/v1/device/register` trước. Trên production, khi nút chưa bật tính năng góp ổ, cả bốn route `/v1/mobile/storage/offer`, `/shard`, `/challenge`, `/proof` đều trả `503`. Ở mọi môi trường, `/v1/mobile/storage/offer` trả `503 no_shard_available` khi nút chưa có mảnh vừa với `requested_bytes`; thử lại sau hoặc đổi nút.

Mã lỗi chung của nhóm lưu trữ: `400` thân hoặc chữ ký sai định dạng, hoặc dung lượng khai không hợp lệ · `403` chữ ký không khớp, thiết bị không khớp, hoặc `device_not_bound` · `404` lease không tồn tại · `409` trạng thái không hợp lệ (hết hạn, câu hỏi không khớp, hết lượt) hoặc bằng chứng không khớp · `429` kiểm chứng quá sớm, hoặc khoá đã giữ đủ số lease đang mở · `500` lỗi phía node · `503` xem đoạn trên, hoặc node hết chỗ cấp lease.

### 2.9 Route KHÔNG dành cho app

Node còn các route phục vụ vận hành và liên lạc giữa các node. Chúng không thuộc API cho app và có thể đổi không báo trước; app chỉ dùng các route có trong tài liệu này.

---

## 3. Xác thực

### 3.1 Bearer — vé PhoenixKey

Header: `Authorization: Bearer <token>`.

Khi node bật cổng PhoenixKey, token là **`app_token` PhoenixKey** dạng JWS. Cổng tắt mặc định; khi tắt, `app_token` bị từ chối với `403 Invalid token`. Khi cổng tắt, đường Bearer chỉ nhận thẻ dùng chung của node. Dưới đây là các điều kiện kiểm khi cổng bật:

- Thuật toán chỉ `EdDSA`; `HS*`, `RS*`, `none` bị từ chối. Header phải có `kid`; `typ` nếu có phải là `JWT` hoặc `at+jwt`.
- `exp` bắt buộc; `iat` không được ở tương lai quá độ lệch giờ cho phép (mặc định 60 giây).
- `iss` phải khớp issuer đã cấu hình; `aud` phải chứa ServiceDID của LampNet.
- Ghi cần scope `lampnet.write`; token không có scope bị từ chối khi ghi. Scope `lampnet.read` chưa được kiểm.
- Ghi bắt buộc có `jti`.
- Xác thực Bearer **không** tạo chủ sở hữu. Object chỉ có chủ khi yêu cầu mang chữ ký DID hợp lệ (mục 3.2): `/mirage/put` kèm 4 header DID, ký trên BLAKE3 của byte trường `file`; hoặc tải qua `/v1/session/open`. Thiếu DID thì object là `legacy:unowned`: ai có CID đều đọc được và không xoá được.
- Ở `/mirage/put`, node chưa cấu hình bộ phân giải DID thì không kiểm 4 header DID, kể cả khi chữ ký sai. Nút không bật chế độ chặt vẫn nhận tải lên và object thành `legacy:unowned`, không báo lỗi; nút bật chế độ chặt trả `400`, kể cả khi có đủ header. Node đã cấu hình bộ phân giải: thiếu `x-lampnet-did` thì nút chặt trả `400`, nút không chặt ghi `legacy:unowned`; có `x-lampnet-did` mà header còn lại thiếu hoặc sai thì bị từ chối, thường là `401`. Sau khi tải object không công khai, gọi `GET /<cid>` không kèm chứng minh để kiểm: object có chủ phải trả `404`.

Node hiện không kiểm `X-LN-Body-Hash`, `X-LN-Nonce`, `X-LN-Ts`, `X-LN-PoP` ở đường này: chính sách triển khai hiện tắt phép kiểm này. Vé bị lộ có thể dùng lại cho thân khác cho tới khi hết hạn, nên giữ thời hạn vé ngắn.

Lỗi là văn bản thuần, không có mã chữ: `401 Missing Authorization header` · `401 Expected `Bearer <token>` scheme` · `403 Invalid token`. Khi cổng bật, vé kiểm hỏng trả thân `"<mã>: <mô tả>"` với status theo lỗi; `503` khi cổng cấu hình sai, không lấy được bộ khoá công khai, hoặc không tra được trạng thái vé ở dịch vụ định danh (nếu node bật kiểm thu hồi): đây là lỗi tạm phía node, thử lại sau.

### 3.2 Chữ ký DID (khoá P-256)

Dùng cho `/v1/session/open`, `/v1/device/register`, `/mirage/put` khi cần có chủ, tải xuống object có chủ không công khai, và xoá. Bốn header:

| Header | Nội dung |
|---|---|
| `x-lampnet-did` | DID người gọi |
| `x-lampnet-did-ts` | Thời điểm ký (giây Unix) |
| `x-lampnet-did-nonce` | Nonce một lần |
| `x-lampnet-did-sig` | Chữ ký P-256 dạng DER, hex |

Cách dựng chữ ký:

1. **Query chuẩn hoá**: tách theo `&`, mỗi cặp tách ở dấu `=` đầu tiên, giải mã percent cả khoá lẫn giá trị. Đoạn rỗng (ví dụ `a=1&&b=2`) bị bỏ qua; đoạn không có `=` có giá trị rỗng; khoá trùng hoặc byte không phải UTF-8 sau khi giải mã → bị từ chối. Sắp các cặp theo `(khoá, giá trị)`, mã hoá lại `&`, `=`, `%` và byte điều khiển thành `%XX` (chữ hoa), rồi nối `khoá=giá_trị` bằng `&`. Không có query thì là chuỗi rỗng. Giữ khoá và giá trị ở ASCII: phép mã hoá lại xử lý từng byte, nên byte ngoài ASCII có thể làm lệch chữ ký.
2. **Challenge** (32 byte) = BLAKE3 của chuỗi byte: với lần lượt `LAMPNET_MIRAGE_CHALLENGE_v1`, phương thức HTTP viết hoa, đường dẫn, query chuẩn hoá, BLAKE3 của thân (32 byte; thân rỗng thì là BLAKE3 của chuỗi rỗng), nonce — mỗi phần đứng sau độ dài của nó dạng u32 little-endian.
3. **Thông điệp ký** = `<challenge dạng hex>:LAMPNET_MIRAGE_CLIENT_AUTH_v1:<ts>`, với `ts` trùng giá trị header `x-lampnet-did-ts`. Độ lệch giờ tối đa 60 giây.
4. Ký thông điệp bằng ECDSA P-256 với SHA-256, xuất chữ ký dạng DER rồi hex.

Mỗi nonce chỉ dùng một lần. Nếu node chưa cấu hình bộ phân giải DID, `/v1/session/open` trả `503`; riêng `/mirage/put` thì bỏ qua header DID (mục 3.1).

### 3.3 Chữ ký khoá thiết bị (Ed25519) — join và góp tài nguyên

Khoá thiết bị là khoá Ed25519 dẫn từ seed thiết bị (mục 5.2).

- **`/v1/join/v2/request`**: chữ ký nằm trong `request_json` do `join_and_contribute` dựng; app gửi nguyên văn, không sửa.
- **`/v1/join/status/:device_pubkey_hex`**: ba header `x-lampnet-join-status-pubkey`, `x-lampnet-join-status-ts`, `x-lampnet-join-status-sig`, dựng bằng `build_join_status_request`. Độ lệch giờ tối đa 300 giây. Hàm trả JSON `{device_pubkey_hex, ts, sig_hex}`, app tự chuyển sang header; `verifier_did` lấy từ `/v1/join/challenge` của đúng node đó. Thiếu hoặc sai → `401`; khoá ký khác khoá trên đường dẫn → `403`; đường dẫn sai định dạng → `400`.
- **`/v1/mobile/*`**: chữ ký nằm trong thân do các hàm `build_*` dựng.
- **`/v1/device/register`**: SDK không có hàm dựng; app tự ký bằng thư viện Ed25519. Giải hex chuỗi từ `generate_device_seed()` thành 32 byte rồi dùng thẳng làm seed RFC 8032.
  - `domain` = `lampnet/device/register/v1`.
  - `hash` = BLAKE3 của `len(domain) ‖ domain ‖ len(did) ‖ did ‖ len(device_pubkey) ‖ device_pubkey ‖ len(ts) ‖ ts`, với `len` là độ dài u32 little-endian.
  - `did` là byte UTF-8 của DID đã chứng minh qua header. `device_pubkey` là 32 byte thô. `ts` là `timestamp` dạng i64 little-endian, 8 byte.
  - Ký Ed25519 thẳng lên 32 byte `hash`, không băm lại.
  - Thân: `device_pubkey` hex 32 byte, `signature` hex 64 byte, `timestamp` số nguyên, `did` tuỳ chọn; có giá trị thì phải khớp DID ở header, không thì `400 did_mismatch`.

  Vector kiểm, seed chỉ dùng thử: seed = 32 byte `0x07`, `did` = `did:phoenix:example`, `timestamp` = `1700000000`. Vector chỉ để kiểm phép băm và phép ký; DID này không hợp lệ khi gửi lên node.

  ```
  device_pubkey = ea4a6c63e29c520abef5507b132ec5f9954776aebebe7b92421eea691446d22c
  hash          = c53f9ac45f7595a3ef085e8ae208d2c7c4ad08aace19671839f60e5137e63c12
  signature     = c452bdfa15e209cc9f796227a40850dc475f0d73ded577b56b7ba5cda941359cedb468b0e0f80984973ce1b44cd5fe906421e35d76d3dbc887ae0c5da0a1ed07
  ```

---

## 4. Cấp riêng tư

### 4.1 Ba cấp

| Cấp | Giá trị chấp nhận | Nội dung | Tên tài liệu |
|---|---|---|---|
| `Public` | `public`, `bulk`, `2`, `0x02` | Không mã hoá | Công khai |
| `Private` | `private`, `vault`, `1`, `0x01` | Niêm bằng khoá dẫn từ mật khẩu | Vẫn công khai |
| `Shielded` | `shielded`, `4`, `0x04` | Niêm | Giấu khỏi CID và sổ đăng ký |

Hai cấp niêm nội dung (`Private`, `Shielded`) bắt buộc có `password` khi tải lên. Độ an toàn của bản niêm phụ thuộc vào mật khẩu: dùng mật khẩu mạnh, sinh ngẫu nhiên, hoặc tự niêm phía client (mục 4.3).

`vault` và `bulk` là tên cũ, vẫn được nhận để tương thích; tích hợp mới dùng `public` / `private` / `shielded`. Mọi dữ liệu đi **cùng một đường**, chỉ khác cấp riêng tư — không có kho riêng cho từng loại.

### 4.2 Khi client không nói gì

Không có trường `privacy`: có gửi `password` thì object là `Private`, không gửi thì là `Public`. Nên luôn ghi rõ `privacy` để không phụ thuộc mặc định.

### 4.3 Niêm phía client

Hướng tích hợp là **client tự niêm** dữ liệu trước khi gửi, bằng các hàm `moc_*` của SDK di động (mục 5.3), hoặc thư viện Rust tương đương do đội LampNet cấp. Khoá nội dung sinh và giữ ở client; muốn chia sẻ thì bọc khoá cho khoá công khai của người nhận (`moc_wrap_key`). Gateway không giữ khoá chủ để dẫn khoá cho người dùng.

**Đang làm:**
- Node tiếp nhận object đã niêm theo định dạng MOC. Hiện node chưa tính CID theo định dạng MOC, nên CID do `moc_cid` tính **không trùng** CID node trả về. App phải dùng CID node trả để tải xuống.

---

## 5. SDK di động `lampnet-mobile-sdk` 1.0.0

### 5.1 Lấy bản dựng

- Binding sinh bằng UniFFI 0.28 (proc-macro). Kotlin: package `uniffi.lampnet_mobile_sdk`, tệp `lampnet_mobile_sdk.kt`. Swift: `lampnet_mobile_sdk.swift`.
- Bản dựng Android đóng gói sẵn được phát hành trong kho mã nguồn riêng; **liên hệ đội LampNet để được cấp quyền tải**.
- Mọi lỗi đi qua cầu FFI là `SdkError::Failed(String)`; phía Kotlin/Swift nhận `SdkException`.

### 5.2 Seed thiết bị — đọc trước khi dùng

`generate_device_seed()` sinh seed 32 byte (hex) cho khoá thiết bị.

- Gọi **một lần** khi cài app, cất ngay vào **Android Keystore / iOS Keychain**.
- Mỗi lần cần ký thì đọc seed từ kho khoá của hệ điều hành rồi truyền thẳng vào hàm SDK ở tầng native.
- **Không đưa seed qua cầu JavaScript** (React Native, WebView…), không ghi log, không gửi lên server.
- Mất seed = mất danh tính thiết bị và công đã tích luỹ gắn với nó.

### 5.3 Danh sách hàm export, theo việc

**Kết nạp thiết bị**

| Hàm | Việc làm |
|---|---|
| `generate_device_seed()` | Sinh seed thiết bị (mục 5.2) |
| `benchmark_device()` | Đo cấu hình máy, điền các trường đo đạc của `JoinConfig` |
| `join_and_contribute(config: JoinConfig)` → `JoinOutcome` | Dựng và ký yêu cầu kết nạp, tính kế hoạch góp tài nguyên. Chạy hoàn toàn cục bộ |
| `build_join_status_request(verifier_did, ts, seed_hex)` | Dựng JSON để app chuyển thành ba header cho `/v1/join/status/:device_pubkey_hex` (mục 3.3) |

**Niêm và mở dữ liệu (MOC)**

| Hàm | Việc làm |
|---|---|
| `moc_seal(k_obj, security_level, content_type?, doc_type?, payload)` | Niêm dữ liệu bằng khoá nội dung 32 byte. `security_level`: `0` = L0, `1` = L1, `2` = LMax |
| `moc_open(...)` → `MocOpened{payload, security_level, content_type, doc_type}` | Mở dữ liệu đã niêm |
| `moc_cid(...)` | Tính CID của bản niêm (xem lưu ý mục 4.3) |
| `moc_seal_with_aad(key, aad, plaintext)` | Niêm XChaCha20-Poly1305 với `aad` của bên gọi, trả `nonce(24) ‖ ct ‖ tag(16)`. `aad` không nằm trên dây |
| `moc_open_with_aad(key, aad, sealed)` | Mở bản niêm với đúng `aad` lúc niêm |
| `moc_generate_keypair()` → `KeypairRec{secret, public}` | Sinh cặp khoá X25519 |
| `moc_wrap_key(k_obj, recipient_pub)` → `WrappedKeyRec{ephemeral_pub, nonce, ct}` | Bọc khoá nội dung cho người nhận |
| `moc_unwrap_key(...)` | Mở khoá nội dung đã bọc |

**Góp tài nguyên**

| Hàm | Việc làm |
|---|---|
| `gate_should_run(charging, wifi, battery_pct, user_active)` | Chỉ cho chạy khi đang sạc, có Wi-Fi, đủ pin và người dùng không dùng máy |
| `gate_should_answer_audit(wifi, battery_pct)` | Điều kiện để trả lời câu hỏi kiểm chứng lưu trữ |
| `build_lease_request(...)` | Thân `/v1/mobile/lease` |
| `build_payload_fetch_request(...)` | Thân `/v1/mobile/payload/:lease_id` |
| `execute_compute_lease(...)` | Chạy việc tính toán đã nhận |
| `report_receipt(...)` | Thân `/v1/mobile/report` |
| `build_storage_offer(offer: StorageOffer{device_pubkey, free_mb, requested_bytes, ts}, seed_hex)` | Thân `/v1/mobile/storage/offer` |
| `build_shard_fetch_request(...)` | Thân `/v1/mobile/storage/shard/:lease_id` |
| `build_audit_challenge_request(...)` | Thân `/v1/mobile/storage/challenge/:lease_id` |
| `answer_storage_audit(...)` | Thân `/v1/mobile/storage/proof` |
| `now_unix_secs()` | Đồng hồ dùng chung cho các trường `ts` |

**Tiện ích**

| Hàm | Việc làm |
|---|---|
| `resolve_uri(uri)` → `UriResolution{data_class, hash}` | Tách một URI LampNet thành cấp dữ liệu (`vault` / `bulk` / `shielded`) và băm 32 byte |

### 5.4 Luồng kết nạp

1. `GET /v1/join/challenge` → lấy `nonce_hex` và `verifier_did`.
2. `benchmark_device()` → điền số đo vào `JoinConfig`.
3. Dựng `JoinConfig`:
   - `subject_did`: dạng `did:phoenix:…`. Trước khi có định danh sinh trắc, dùng `did:phoenix:tmp:<device_id>`. Dạng tạm chỉ dùng được khi node không bật kiểm DID chặt (nếu bật, trả `400 invalid_subject_did`).
   - `cardano_address`: địa chỉ thanh toán Preview `addr_test1…` (base hoặc enterprise); địa chỉ khác bị từ chối với `400 invalid_cardano_address`. Để rỗng nếu chưa có.
   - `bootstrap_did` = `verifier_did`; `challenge_nonce_hex` = `nonce_hex`; `seed_hex` = seed từ kho khoá.
   - `attestation_mode`: `"Hardware"` hoặc `"Software"`.
   - Các trường phần cứng và đo đạc: `hw_model`, `chip_id`, `cpu_cores`, `cpu_score`, `ram_mb`, `disk_gb`, `bandwidth_mbps`, `measured_flops`, `measured_bandwidth_mbps`, `measured_disk_iops`, `s_net`, `s_avail`, `s_energy`, `network_load`.
   - Bản dựng mới: `gpus` (`List<GpuSpec>`) và `server_attest_ext` (lấy từ `attest_ext` của challenge). Hai trường không có giá trị mặc định nên phải truyền, kể cả khi rỗng. Bản dựng cũ cùng version 1.0.0 không có hai trường này, nên đối chiếu với binding sinh ra từ đúng bản dựng đang dùng.
4. `join_and_contribute(config)` → `JoinOutcome`.
5. `POST /v1/join/v2/request` với thân là nguyên văn `JoinOutcome.request_json`.
6. Kiểm lại trạng thái bằng `GET /v1/join/status/:device_pubkey_hex` (header từ `build_join_status_request`).

### 5.5 Kiểu `JoinOutcome`

| Trường | Ý nghĩa |
|---|---|
| `decision` | `JoinDecision`: `Full` · `Conditional` · `Deferred` · `Rejected` — đánh giá phía máy |
| `score`, `initial_rep`, `tier`, `s_hw` | Điểm và hạng ước tính phía máy |
| `well_formed`, `verify_msg` | Yêu cầu có đúng khuôn không. `well_formed = true` **không** có nghĩa server sẽ chấp nhận |
| `device_pubkey_hex`, `fingerprint_hex` | Khoá công khai và vân tay thiết bị |
| `plan` | `ContributionPlan` (dưới đây) |
| `request_json` | Thân gửi lên `/v1/join/v2/request` |
| `sent_gpus` | Yêu cầu có gửi danh sách GPU hay không. `true` mà gặp `422 bad_attestation` thì dựng lại không GPU với nonce mới và thử đúng một lần; SDK không tự thử lại. `gpus` khác rỗng mà `sent_gpus = false` thì app ghi cảnh báo |

Quyết định cuối cùng là phản hồi của `/v1/join/v2/request`, không phải `decision` phía máy.

`ContributionPlan` mô tả thiết bị sẽ góp gì, để app hiển thị cho người dùng trước khi đồng ý:

| Trường | Ý nghĩa |
|---|---|
| `storage_gb` | Dung lượng lưu trữ góp |
| `compute_gflops` | Năng lực tính toán góp |
| `bandwidth_mbps` | Băng thông góp |
| `power_added_watts`, `power_total_watts` | Công suất điện thêm vào / tổng |
| `energy_added_kwh_day`, `energy_total_kwh_day` | Điện năng mỗi ngày thêm vào / tổng |

---

## 6. Thưởng

Thưởng cho thiết bị góp tài nguyên tính bằng token **CARP**.

App xem công đã tích luỹ qua `GET /v1/mobile/rewards/:device_pubkey` (các trường `units`, `storage_units`, `nanothread`). Đây là đơn vị **công**, chưa phải số CARP; việc quy đổi và chi trả do hệ thống thưởng thực hiện theo kỳ, không qua API của app.

---

## 7. Câu hỏi thường gặp và giới hạn

**SDK có tự gửi yêu cầu lên mạng không?**
Không. Mọi hàm SDK chạy cục bộ; app tự gửi `request_json` và các thân `build_*`.

**Tôi dùng CID nào để tải xuống?**
CID do node trả về khi tải lên. CID từ `moc_cid` chưa trùng với CID của node (mục 4.3).

**Tải xuống object có chủ mà nhận `404`?**
Object không tồn tại hoặc yêu cầu không có chứng minh DID của chủ đều trả `404`. Chữ ký DID sai trả `401`. Kiểm lại header (mục 3.2).

**File lớn hơn 100 MB?**
Dùng phiên tải lên (mục 2.3), trong trần của phiên. Mỗi mảnh không vượt `per_shard_ceiling`. Ở `/v1/session/*`, mảnh và lệnh chốt phải xong trong 10 phút sống của cap.

**`well_formed = true` nhưng server từ chối?**
`well_formed` chỉ kiểm khuôn phía máy. Đọc mã `error` trong phản hồi (bảng mục 2.7). Thường gặp: `bad_challenge` (nonce hết hạn, lấy lại challenge) và các mã `409` (khoá hoặc vân tay đã gắn với DID khác).

**Có cần gọi `/v1/device/register` không?**
Bắt buộc nếu muốn góp ổ, và nếu muốn nhận việc tính toán trên node có bộ phân giải theo từng DID: `/v1/mobile/storage/offer` và `/v1/mobile/lease` trả `403 device_not_bound` với khoá chưa gắn. Chỉ kết nạp thì không cần. Route đòi cả chữ ký trong thân (mục 3.3) và 4 header DID.

**Giới hạn đã biết**
- Scope `lampnet.read` và tiếp nhận object MOC phía node: đang làm.
- Kết nạp dùng `/v1/join/v2/request`; gắn khoá vào DID để góp ổ hoặc nhận việc tính toán dùng `/v1/device/register`.
