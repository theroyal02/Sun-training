# RESTful API Design, Protocol Standards & Security Architecture

Tài liệu chuẩn hóa kiến thức thiết kế, triển khai và bảo mật Web API cho hệ thống phân tán.

---

## 1. Kiến trúc RESTful & Nguyên tắc Giao thức HTTP

REST (Representational State Transfer) là kiến trúc hướng tài nguyên (Resource-Oriented Architecture), hoạt động trên nền giao thức HTTP không trạng thái (Stateless).

### 1.1. Các ràng buộc cốt lõi (Core Constraints)

```
+---------------+     HTTP Stateless Request      +---------------+
|               | ----------------------------->  |               |
|    Client     |                                 | Server / API  |
| (SPA, Mobile) | <-----------------------------  |   Gateway     |
+---------------+       JSON State Response       +---------------+
```

- **Statelessness**: Server không lưu context của client giữa các request. Mọi thông tin xác thực, phân trang, lọc phải nằm trọn vẹn trong từng request.
- **Client-Server Decoupling**: Giao diện người dùng và logic lưu trữ dữ liệu độc lập hoàn toàn, giao tiếp qua contract dữ liệu cố định (thường là JSON).
- **Uniform Interface**: Định danh tài nguyên qua URI chuẩn, thao tác qua HTTP verbs tiêu chuẩn và chuyển giao trạng thái (representations).
- **Cacheability**: Mọi response phải chỉ rõ khả năng cache qua header (`Cache-Control`, `ETag`) nhằm giảm tải network và server.

### 1.2. Phân loại HTTP Methods: Safe & Idempotent

| Method | Ý nghĩa nghiệp vụ | Safe? | Idempotent? | Request Body | Response Success |
| :--- | :--- | :---: | :---: | :---: | :--- |
| `GET` | Đọc tài nguyên | **Có** | **Có** | Không khuyến nghị | `200 OK` |
| `POST` | Tạo mới tài nguyên / Trigger action | Không | Không | Bắt buộc (Payload) | `201 Created` |
| `PUT` | Thay thế toàn bộ tài nguyên (Full replace) | Không | **Có** | Toàn bộ schema | `200 OK` / `204 No Content` |
| `PATCH` | Cập nhật một phần (Delta update) | Không | Không* | Các trường thay đổi | `200 OK` |
| `DELETE` | Xóa tài nguyên | Không | **Có** | Tùy chọn | `200 OK` / `204 No Content` |
| `HEAD` | Giống GET nhưng chỉ lấy Headers | **Có** | **Có** | Không | `200 OK` (No Body) |
| `OPTIONS` | CORS Preflight / Khám phá method hỗ trợ | **Có** | **Có** | Không | `204 No Content` |

> *\*Lưu ý:* `PATCH` về bản chất toán học không bắt buộc phải idempotent (ví dụ: thao tác `{"op": "increment", "value": 5}` sẽ làm thay đổi giá trị sau mỗi lần gọi lặp lại).

### 1.3. Chuẩn thiết kế URI hướng tài nguyên (Resource-Oriented URI)

- Dùng **danh từ số nhiều** cho collections, tuyệt đối không dùng động từ trên URI:
  - Chuẩn: `POST /api/v1/orders` (Tạo order)
  - Sai: `POST /api/v1/createOrder` (RPC style)
- Phân cấp quan hệ thực thể (Sub-resources) tối đa 2 cấp:
  - `GET /api/v1/merchants/102/products`
  - Nếu sâu hơn, tách thành endpoint độc lập: `GET /api/v1/order-items/998` thay vì `/merchants/102/orders/55/items/998`.
- Thao tác phi CRUD (Actions) biểu diễn qua sub-resource mang nghĩa trạng thái:
  - `POST /api/v1/orders/123/cancellation` hoặc `POST /api/v1/users/45/password-reset`

---

## 2. Giải phẫu HTTP Request & Response (Protocol Deep-Dive)

### 2.1. Cấu trúc HTTP Request

```http
POST /api/v1/checkout/sessions?source=campaign HTTP/1.1
Host: api.platform.internal
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json
Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d

{
  "orderId": "ord_88910",
  "currency": "USD",
  "amount": 25000
}
```

Bốn phương thức truyền dữ liệu từ Client:
1. **Path Parameters**: Định danh tài nguyên cụ thể (`/api/v1/tenants/{tenantId}/invoices/{invoiceId}`).
2. **Query Parameters**: Bộ lọc (`?status=active`), sắp xếp (`?sort=-createdAt`), phân trang (`?page=2&limit=20`), hoặc projection (`?fields=id,title`).
3. **Request Headers**: Metadata hạ tầng, chứng thực (`Authorization`), định dạng dữ liệu (`Content-Type`, `Accept`), khóa chống trùng lặp (`Idempotency-Key`).
4. **Request Body**: Payload dữ liệu phức tạp (`application/json`, `multipart/form-data`).

### 2.2. Phân loại HTTP Status Codes theo tiêu chuẩn sản phẩm

```
1xx (Informational) -> Protocol switches (101 Switching Protocols)
2xx (Success)       -> Request xử lý thành công
3xx (Redirection)   -> Tài nguyên chuyển vị trí (301 Permanent, 304 Not Modified)
4xx (Client Error)  -> Request sai cú pháp, thiếu quyền hoặc vi phạm nghiệp vụ
5xx (Server Error)  -> Server sập, timeout, hạ tầng downstream gặp sự cố
```

| Mã Code | Tên chuẩn | Khi nào sử dụng? |
| :--- | :--- | :--- |
| `200` | `OK` | Đọc thành công (`GET`), hoặc cập nhật thành công trả về data. |
| `201` | `Created` | Tạo tài nguyên mới (`POST`), kèm header `Location: /api/v1/...`. |
| `204` | `No Content` | Thao tác thành công nhưng không cần trả body (`DELETE`, `PUT`). |
| `304` | `Not Modified` | Client cache còn hiệu lực qua `If-None-Match` (ETag). |
| `400` | `Bad Request` | Payload sai cú pháp, JSON invalid. |
| `401` | `Unauthorized` | Chưa xác thực hoặc Token không hợp lệ / hết hạn (Thiếu AuthN). |
| `403` | `Forbidden` | Đã xác thực nhưng không có quyền truy cập tài nguyên (Thiếu AuthZ). |
| `404` | `Not Found` | Không tồn tại URI hoặc che giấu tài nguyên không có quyền đọc. |
| `409` | `Conflict` | Trùng lặp unique constraint (Email đã tồn tại, race-condition). |
| `422` | `Unprocessable Entity`| JSON hợp lệ nhưng vi phạm validation rules (Trường bắt buộc, sai định dạng). |
| `429` | `Too Many Requests`| Bị chặn bởi Rate Limiter (Kèm header `Retry-After`). |
| `500` | `Internal Server Error`| Bug unhandled exception phía backend code. |
| `502` | `Bad Gateway` | Reverse proxy (Nginx, Kong) không kết nối được upstream service. |
| `503` | `Service Unavailable` | Hệ thống bảo trì, quá tải, circuit breaker đang open. |
| `504` | `Gateway Timeout` | Upstream service xử lý quá lâu vượt quá timeout của reverse proxy. |

---

## 3. Tiêu chuẩn Thiết kế Tài liệu & Schema API (Production Spec)

### 3.1. Tiếp cận Spec-First với OpenAPI Specification (OAS 3.1)

Không viết document chay bằng text thủ công (dễ bị drift lệch pha so với code). Hệ thống chuẩn hóa dùng **OpenAPI Specification (OAS)**:
- Single Source of Truth cho cả Frontend, Backend, QA và SDK Generator.
- Sinh tự động Mock Server (Prism), Client SDK (openapi-generator), và Interactive Portal (Scalar, Swagger UI).

### 3.2. Chuẩn hóa Định dạng Lỗi: RFC 7807 (Problem Details)

Tránh trả về chuỗi text đơn thuần (`{"error": "Lỗi rồi"}`). Sản phẩm chuyên nghiệp sử dụng chuẩn **RFC 7807**:

```json
{
  "type": "https://api.platform.internal/errors/insufficient-funds",
  "title": "Insufficient Account Balance",
  "status": 422,
  "detail": "Your balance is $45.00, but the transaction requires $120.00.",
  "instance": "/api/v1/payments/tx_99812",
  "invalidParams": [
    {
      "field": "amount",
      "reason": "Exceeds available balance"
    }
  ],
  "timestamp": "2026-11-09T09:27:36Z",
  "traceId": "c4b3a887-fae1-4509"
}
```

### 3.3. Chiến lược Phân trang (Pagination Patterns)

1. **Offset-based Pagination**:
   - Query: `?page=3&limit=20` (hoặc `?offset=40&limit=20`).
   - *Ưu điểm*: Dễ code, nhảy trang tùy ý.
   - *Nhược điểm*: Khi bảng hàng triệu record, `OFFSET 1000000` quét toàn bộ index gây chậm (`O(N)`). Bị duplicate/skip dữ liệu khi có record mới chèn vào giữa lúc người dùng xem.
2. **Cursor-based / Keyset Pagination**:
   - Query: `?cursor=eyJpZCI6MTA0OTQsImNyZWF0ZWRBdCI6MTcwMDAwMH0&limit=20`.
   - *Ưu điểm*: Tối ưu tuyệt đối (`O(1)` qua index seek `WHERE id > last_seen_id`), dữ liệu nhất quán không bị trùng lặp.
   - *Nhược điểm*: Không hỗ trợ nhảy cóc trang ngẫu nhiên (chỉ đi tới `next` / lùi `prev`).

---

## 4. Kiến trúc Xác thực & Phân quyền (Authentication & Authorization)

```
+-------------------------------------------------------------------------------+
|                             Security Separation                               |
+-------------------------------------------------------------------------------+
|  Authentication (AuthN) : "Bạn là ai?"     -> Verify Credentials / Tokens     |
|  Authorization  (AuthZ) : "Bạn được làm gì?" -> RBAC, ABAC, Permission Matrix |
+-------------------------------------------------------------------------------+
```

### 4.1. Stateful Auth: Session & Cookie

```
Client                      Server                      Redis Session Store
  |                            |                                |
  |-- 1. POST /login --------->|                                |
  |      (user, pass)          |-- 2. Verify DB & Gen SID ----->|
  |                            |<-- 3. Store Session Record ----|
  |<-- 4. Set-Cookie: sid=xyz -|                                |
  |                            |                                |
  |-- 5. GET /orders --------->|                                |
  |      Cookie: sid=xyz       |-- 6. Check SID Valid? -------->|
  |                            |<-- 7. Return User Payload -----|
  |<-- 8. 200 OK (Data) -------|                                |
```

- **Đặc trưng**: Trạng thái phiên lưu tại server (Redis/DB).
- **Cấu hình Cookie an toàn (Zero-Trust Browser)**:
  - `HttpOnly`: JavaScript không thể đọc được cookie (`document.cookie`), triệt tiêu rủi ro đánh cắp token qua XSS.
  - `Secure`: Cookie chỉ được truyền qua kết nối mã hóa TLS/HTTPS.
  - `SameSite=Lax` hoặc `SameSite=Strict`: Ngăn browser gửi cookie trong cross-site request, phòng chống triệt để tấn công CSRF.
- **Đánh giá**:
  - *Mạnh*: Revoke tức thì (chỉ cần xóa key trong Redis là logout lập tức).
  - *Yếu*: Overhead I/O tra cứu session store ở mỗi request khi hệ thống mở rộng nhiều cụm.

---

### 4.2. Stateless Auth: JSON Web Token (JWT)

JWT là định dạng token khép kín (self-contained), truyền tải thông tin đã được ký điện tử giữa các bên.

#### Cấu trúc chuỗi JWT: `Header.Payload.Signature`

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c3JfMTA5Iiwicm9sZSI6ImFkbWluIiwiZXhwIjoxNzk0MjE2MDAwfQ.K3gK-B6f01s9JmYqT...
```

1. **Header (JOSE Header)**: Thuật toán mã hóa (`alg: "HS256" | "RS256"`) và loại token (`typ: "JWT"`).
2. **Payload (Claims)**:
   - *Registered Claims*: `sub` (Subject ID), `iss` (Issuer), `exp` (Expiry Time), `iat` (Issued At).
   - *Custom Claims*: `role`, `tenantId`, `permissions`.
   - **CẢNH BÁO**: Payload chỉ mã hóa **Base64Url**, ai cũng giải mã ra đọc được. Tuyệt đối không để password, secret key hoặc dữ liệu nhạy cảm vào payload.
3. **Signature (Chữ ký điện tử)**:
   - Ký bằng thuật toán đối xứng (HMAC-SHA256 với Secret chung) hoặc bất đối xứng (RSA/ECDSA với Private Key ký, Public Key xác thực).
   - Đảm bảo tính toàn vẹn (Integrity): Nếu client sửa 1 ký tự trong payload, chữ ký không còn khớp -> Request bị từ chối.

#### Mô hình Dual-Token: Access Token & Refresh Token Rotation

```
Client                         API Gateway / Auth Server
  |                                        |
  |-- 1. POST /auth/login ---------------->|
  |<-- 2. AccessToken (15m) + RefreshToken |
  |                                        |
  |-- 3. Request API (Header: Bearer AT) ->| (Xác thực stateless trong RAM)
  |<-- 4. Response Data -------------------|
  |                                        |
  | [AccessToken Hết Hạn]                  |
  |-- 5. POST /auth/refresh (Gửi RT) ----->|
  |                                        |-- Verify RT hợp lệ?
  |                                        |-- Hủy RT cũ, phát hành cặp mới (Rotation)
  |<-- 6. Cặp AT mới + RT mới -------------|
```

- **Access Token**: Sống cực ngắn (5 - 15 phút), lưu trong memory (biến runtime trong app/client state) để tránh tấn công XSS/lưu persistent.
- **Refresh Token**: Sống dài hơn (7 - 30 ngày), lưu trong `HttpOnly, Secure Cookie`.
- **Refresh Token Rotation (RTR)**: Mỗi khi dùng Refresh Token để lấy Access Token mới, hệ thống phải cấp một Refresh Token hoàn toàn mới và vô hiệu hóa token cũ. Nếu phát hiện Refresh Token cũ bị gửi lại lần 2, hệ thống kích hoạt **Reuse Detection**: lập tức thu hồi toàn bộ phiên của user đó vì dấu hiệu token đã bị lộ.

---

### 4.3. Delegated Authorization: OAuth 2.0 & OpenID Connect (OIDC)

OAuth 2.0 là **khung ủy quyền (Authorization Framework)** cho phép ứng dụng bên thứ ba (Third-party App) truy cập tài nguyên của người dùng mà không cần biết mật khẩu của họ. OpenID Connect (OIDC) là tầng **Authentication** xây trên nền OAuth 2.0 (bổ sung `id_token`).

#### 4 Roles trong OAuth 2.0
1. **Resource Owner**: Người dùng sở hữu tài khoản dữ liệu.
2. **Client**: Ứng dụng muốn truy cập dữ liệu (Web SPA, Mobile App, Backend Service).
3. **Authorization Server**: Máy chủ xác thực (Google Identity, Auth0, Keycloak).
4. **Resource Server**: Máy chủ lưu trữ API và dữ liệu người dùng.

#### Luồng chuẩn hiện đại: Authorization Code Flow với PKCE (RFC 7636)

Đây là chuẩn bảo mật bắt buộc hiện nay cho cả Single Page Apps (React/Vue), Mobile Apps và Web Apps truyền thống:

```
User Browser               Client App              Auth Server           Resource Server
     |                         |                        |                       |
     |-- 1. Bấm Login -------->|                        |                       |
     |                         |-- 2. Tạo code_verifier |                       |
     |                         |      & code_challenge  |                       |
     |<-- 3. Redirect to Auth -|                        |                       |
     |       (?challenge=xyz)  |                        |                       |
     |                                                  |                       |
     |-- 4. User đăng nhập & Approve Consent ---------->|                       |
     |<-- 5. Redirect về Client với Authorization Code -|                       |
     |                                                  |                       |
     |-- 6. Gửi Auth Code + code_verifier ban đầu ----->|                       |
     |                         |                        |-- Verify:             |
     |                         |                        |   SHA256(verifier)    |
     |                         |                        |   == challenge?       |
     |                         |<-- 7. Trả Access Token |                       |
     |                         |      (& id_token/RT)---|                       |
     |                         |                                                |
     |                         |-- 8. Gọi API với Bearer Token ---------------->|
     |                         |<-- 9. Trả dữ liệu được ủy quyền ---------------|
```

- **PKCE (Proof Key for Code Exchange)** giải quyết triệt để nguy cơ Authorization Code bị mã độc can thiệp đánh chặn trên trình duyệt/thiết bị di động: Kẻ tấn công nếu có đánh cắp được Authorization Code ở bước 5 cũng không thể đổi thành token vì không có `code_verifier` bí mật sinh ra trong bộ nhớ client.

