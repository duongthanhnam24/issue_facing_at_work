# Webhook & Cơ chế ký HMAC

**Ngày:** 2026-07-16
**Category:** security
**Tags:** webhook, hmac, authentication, replay-attack
**Stack:** Python (FastAPI), HMAC-SHA256

---

## Dự án
TTS ADMIN server

## Context

Đang thiết kế hệ thống TTS có tính năng webhook để push event cho khách hàng.
Có 3 loại event:

- `quota.threshold` — khách dùng đạt ngưỡng cảnh báo (mặc định 80%)
- `quota.exceeded` — khách dùng vượt hạn mức
- `key.expiring` — API key sắp hết hạn (7 ngày trước)

Webhook = server-to-server push: khách đăng ký 1 URL public, khi có event bên mình POST tới đó kèm payload. Khác với polling (khách phải liên tục hỏi "có gì mới không?"), webhook là push realtime.

## Vấn đề bảo mật

URL webhook do khách cung cấp là **public**: ai biết URL cũng có thể POST tới đó. Phát sinh 2 rủi ro:

**Rủi ro 1 — Giả mạo (spoofing):**
Attacker biết URL → tự POST payload giả `{"event":"quota.exceeded", "usage": 999999}` → server khách tưởng thật, xử lý sai (block user, gửi email hoảng loạn...).

**Rủi ro 2 — Sửa trên đường truyền (tampering):**
Request thật bị MITM proxy sửa 1 số trong body giữa đường.

## Giải pháp: HMAC-SHA256

HMAC = hàm hash một chiều **có key**. Cho vào `(secret_key, message)` → ra chuỗi hash cố định 64 ký tự hex.

**3 tính chất quan trọng:**

- **Một chiều (one-way):** biết hash không suy ngược ra được message hay secret
- **Nhạy với thay đổi:** message đổi 1 byte → hash khác hoàn toàn
- **Cần đúng key:** không biết secret thì không tính ra được đúng hash

**Ẩn dụ:** HMAC như vân tay của `(body + secret)`. Ai có công thức lấy vân tay + có 2 nguyên liệu đó thì tự lấy được. Vân tay không "giải mã" ra thứ gì — nó chỉ dùng để so khớp.

## Cách vận hành

### Setup — Sinh & trao secret (1 lần)

Khi admin tạo webhook cho khách:

1. Server sinh secret bằng `secrets.token_urlsafe(32)` → ~43 ký tự random cryptographic-strong (VD: `ALvCPQ2rBJnq-GId7kj9xAJImf4hv_Rac4sGg3RkCBw`)
2. Lưu vào DB — cột `webhooks.secret`
3. Trả về response API **1 lần duy nhất** — admin copy đưa cho khách
4. Khách config vào server họ (biến môi trường / config file)

Từ đây cả 2 bên đều có cùng 1 secret. **Secret không bao giờ được truyền lại trên mạng nữa.**

### Runtime — Bên gửi (server mình)

Mỗi lần fire event:

```python
# Bước 1: serialize payload thành bytes cố định
body = json.dumps(payload, separators=(",", ":")).encode("utf-8")

# Bước 2: tính HMAC-SHA256(secret, body)
signature = "sha256=" + hmac.new(
    secret.encode(), body, hashlib.sha256
).hexdigest()

# Bước 3: POST đi kèm header
```

```
POST https://khach.com/tts-webhook
  Content-Type: application/json
  X-TTS-Signature: sha256=70c12508899f0f04df83184823259293c81b6fef669740f822f7f2bfc40a8690
  X-TTS-Timestamp: 1784112000
  X-TTS-Delivery: 3
  X-TTS-Event: quota.exceeded

  {"event":"quota.exceeded","data":{...}}
```

### Runtime — Bên nhận (server khách)

```python
@app.post("/tts-webhook")
async def receive(request: Request):
    body = await request.body()  # raw bytes — KHÔNG parse JSON trước khi verify
    sig_header = request.headers.get("X-TTS-Signature", "")

    # Tính lại HMAC bằng secret đã lưu sẵn
    expected = "sha256=" + hmac.new(
        WEBHOOK_SECRET.encode(), body, hashlib.sha256
    ).hexdigest()

    # So sánh bằng compare_digest (chống timing attack)
    if not hmac.compare_digest(sig_header, expected):
        raise HTTPException(401, "invalid signature")

    # Verify xong mới an toàn parse & xử lý
    payload = json.loads(body)
    handle_event(payload)
```

### Sơ đồ tổng

```
Server mình                              Server khách
─────────────                           ──────────────
secret (DB)                             secret (env)   ← 2 bên có sẵn từ trước
    │                                       │
    ▼                                       ▼
body ─── HMAC ─→ sig                    body arrive ─── HMAC ─→ sig_expected
    │             │                                              │
    └─────POST body + sig──────────────────────────────────→ so khớp?
                                                                  │
                                                          match  → xử lý
                                                          khác   → reject 401
```

## Q&A

### Q: HMAC có mã hoá payload không?

**Không.** Body vẫn là plaintext, ai sniff được request cũng đọc được nội dung. HMAC chỉ chống **giả mạo** và **sửa đổi**, không chống **lộ thông tin**. Muốn chống lộ → dùng HTTPS (mã hoá tầng transport).

### Q: Vì sao có X-TTS-Timestamp?

**Chống replay attack.** Nếu attacker sniff được 1 request thật (đã ký hợp lệ) rồi gửi lại nhiều lần → chữ ký vẫn khớp (vì body không đổi). Timestamp cho phép bên nhận reject nếu request quá cũ (VD > 5 phút) — trong khoảng đó thì attacker khó khai thác kịp.

## Lesson

- **HMAC chỉ giải quyết integrity + authenticity, không giải quyết confidentiality.** Vẫn cần HTTPS.
- **Verify signature trước khi parse JSON.** Nếu parse trước rồi mới verify, attacker có thể exploit JSON parser trước khi bị chặn.
- **Dùng `hmac.compare_digest`**, không dùng `==` — chống timing attack.
- **Tính HMAC trên raw bytes**, không tính trên object đã parse — vì serialize lại có thể ra body khác (thứ tự key, whitespace).
- **Secret chỉ hiển thị 1 lần khi tạo.** Lỡ mất → phải rotate, không thể "xem lại".
- **Timestamp bắt buộc** để chống replay, dù đã có signature.

## Reference

- [RFC 2104 — HMAC](https://datatracker.ietf.org/doc/html/rfc2104)
- [Stripe Webhook signing](https://stripe.com/docs/webhooks/signatures) — tham khảo cách industry standard
- [GitHub Webhook security](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries)
