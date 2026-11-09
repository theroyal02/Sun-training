# API Development Handbook

Tài liệu tổng hợp và quy chuẩn hóa kiến thức phát triển Web API.

---

## 1. Tổng quan về API & Chuẩn RESTful

### 1.1. Khái niệm API
API (Application Programming Interface) là giao thức cầu nối cho phép hai ứng dụng độc lập trao đổi dữ liệu qua mô hình Request - Response trên nền HTTP.

```
+------------+       HTTP Request (Method + URI + Payload)       +------------+
|            | ------------------------------------------------> |            |
|   Client   |                                                   |   Server   |
|            | <------------------------------------------------ |            |
+------------+          HTTP Response (Status + JSON Data)       +------------+
```

### 1.2. Nguyên tắc thiết kế RESTful API
- **Phân tách Client - Server**: Giao diện người dùng và hạ tầng xử lý dữ liệu hoạt động độc lập.
- **Không lưu trạng thái (Stateless)**: Server không duy trì ngữ cảnh client giữa các lần gọi; mỗi request phải chứa đầy đủ dữ liệu cần thiết.
- **Định dạng dữ liệu**: Ưu tiên sử dụng định dạng JSON cho việc serialize payload.
- **Định danh bằng danh từ**: URI luôn dùng danh từ (thường là số nhiều) để đại diện cho tài nguyên (`/products`, `/users`), không dùng động từ.

### 1.3. Các HTTP Methods tiêu chuẩn

| Method | Mục đích | Tác động dữ liệu | Ví dụ Endpoint |
| :--- | :--- | :--- | :--- |
| `GET` | Đọc dữ liệu | Không thay đổi dữ liệu | `GET /api/v1/users` |
| `POST` | Khởi tạo tài nguyên mới | Thêm mới bản ghi | `POST /api/v1/users` |
| `PUT` | Ghi đè toàn bộ tài nguyên | Thay thế bản ghi cũ | `PUT /api/v1/users/12` |
| `PATCH` | Cập nhật một phần tài nguyên | Sửa đổi trường cụ thể | `PATCH /api/v1/users/12` |
| `DELETE` | Xóa tài nguyên | Xóa bản ghi khỏi hệ thống | `DELETE /api/v1/users/12` |

---

## 2. Cấu trúc HTTP Request & Response

### 2.1. Thành phần của HTTP Request

```http
POST /api/v1/books HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "title": "Clean Code",
  "author": "Robert C. Martin"
}
```

1. **Start Line**: Gồm HTTP Method, đường dẫn URI và phiên bản HTTP.
2. **Request Headers**: Chứa metadata của request (`Content-Type`, `User-Agent`, `Authorization`).
3. **Request Body**: Dữ liệu gửi lên server (dùng cho `POST`, `PUT`, `PATCH`).

**4 hình thức truyền dữ liệu từ Client:**
- **Path Parameter**: Định vị một tài nguyên cụ thể (`/books/42`).
- **Query Parameter**: Lọc, sắp xếp hoặc phân trang (`/books?genre=tech&sort=asc&page=1`).
- **Request Header**: Truyền metadata cấu hình hoặc token bảo mật.
- **Request Body**: Dữ liệu tải trọng dạng JSON hoặc form upload.

### 2.2. Thành phần của HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "success": true,
  "data": {
    "id": 42,
    "title": "Clean Code"
  }
}
```

1. **Status Line**: Phiên bản HTTP, mã trạng thái số và thông điệp trạng thái đi kèm.
2. **Response Headers**: Metadata cấu hình phản hồi từ server (`Content-Type`, `Set-Cookie`).
3. **Response Body**: Dữ liệu trả về cho client.

### 2.3. Các nhóm mã trạng thái HTTP (Status Codes)

- `1xx (Informational)`: Tiếp nhận yêu cầu, đang tiếp tục xử lý giao thức.
- `2xx (Success)`: Yêu cầu đã được tiếp nhận và xử lý thành công (`200 OK`, `201 Created`, `204 No Content`).
- `3xx (Redirection)`: Yêu cầu chuyển hướng tài nguyên sang vị trí khác.
- `4xx (Client Error)`: Lỗi do phía client gửi yêu cầu không hợp lệ (`400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`).
- `5xx (Server Error)`: Lỗi phát sinh từ phía server khi xử lý (`500 Internal Server Error`, `502 Bad Gateway`).

---

## 3. Tiêu chuẩn Xây dựng API Document

### 3.1. Vai trò của tài liệu API
- Làm rõ hợp đồng giao tiếp giữa đội ngũ Frontend và Backend.
- Rút ngắn thời gian làm quen hệ thống (onboarding) cho thành viên mới.
- Hỗ trợ công tác kiểm thử (testing), tích hợp và bảo trì hệ thống lâu dài.

### 3.2. Cấu trúc tài liệu chuẩn cho từng Endpoint

1. **Thông tin Endpoint**:
   - Tên mục đích, HTTP Method, đường dẫn đầy đủ (`POST /api/v1/articles`).
   - Yêu cầu quyền truy cập hoặc loại xác thực cần thiết.
2. **Quy định Request**:
   - Chi tiết danh sách Path params, Query params, Headers.
   - Schema của Request Body: kiểu dữ liệu, các trường bắt buộc (required) hay tùy chọn.
3. **Quy định Response**:
   - Mã HTTP status trả về khi thành công.
   - Cấu trúc JSON mẫu và định dạng từng trường dữ liệu.
4. **Quy định Lỗi (Error Handling)**:
   - Liệt kê các mã lỗi có thể gặp, nguyên nhân cụ thể và cấu trúc thông báo lỗi trả về.
5. **Quy ước chung**:
   - Nhất quán tiền tố phiên bản API (`/api/v1`).
   - Sử dụng định dạng JSON làm chuẩn truyền thông nhất.
   - Cập nhật tài liệu song song với các thay đổi mã nguồn.

---

## 4. Cơ chế Xác thực & Phân quyền (API Authentication)

- **Authentication (AuthN)**: Xác định người dùng là ai.
- **Authorization (AuthZ)**: Xác định người dùng có quyền thực hiện hành động gì.

```
                      +-----------------------------+
                      |     Client Request          |
                      +-----------------------------+
                                     |
                                     v
                      +-----------------------------+
                      | AuthN: Người dùng hợp lệ?   | -- [Không] --> 401 Unauthorized
                      +-----------------------------+
                                     | [Có]
                                     v
                      +-----------------------------+
                      | AuthZ: Có quyền truy cập?   | -- [Không] --> 403 Forbidden
                      +-----------------------------+
                                     | [Có]
                                     v
                      +-----------------------------+
                      | Xử lý tài nguyên / 200 OK   |
                      +-----------------------------+
```

### 4.1. Basic Authentication & Session - Cookie

#### HTTP Basic Authentication
- Client gửi chuỗi `username:password` mã hóa Base64 trong header:
  ```text
  Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
  ```
- *Lưu ý*: Base64 không phải là mã hóa bảo mật, bắt buộc phải truyền qua giao thức HTTPS.

#### Session & Cookie
- **Quy trình hoạt động**:
  1. Client gửi thông tin đăng nhập đến Server.
  2. Server kiểm tra và tạo một bản ghi session lưu ở phía server, đồng thời gửi session ID về client qua header `Set-Cookie`.
  3. Trình duyệt tự động đính kèm cookie chứa session ID trong các request kế tiếp.
  4. Server dùng session ID đối chiếu để định danh người dùng.
  5. Khi đăng xuất, server hủy session và client dọn dẹp cookie.
- **Ưu điểm**: Thu hồi phiên (revoke) ngay lập tức tại server; cookie hỗ trợ các cờ bảo vệ `HttpOnly`, `Secure`, `SameSite`.
- **Nhược điểm**: Tốn bộ nhớ server để lưu session; khó mở rộng quy mô khi chạy nhiều server độc lập; cần phòng ngừa tấn công CSRF.

### 4.2. JSON Web Token (JWT)

JWT là cơ chế token khép kín được ký số, cho phép server xác thực không trạng thái (Stateless).

#### Quy trình hoạt động:
1. Client gửi thông tin đăng nhập.
2. Server xác thực và tạo chuỗi JWT trả về cho client.
3. Client lưu token và gửi lên ở header các request tiếp theo:
   ```text
   Authorization: Bearer <token>
   ```
4. Server xác minh chữ ký và thời hạn của token để cấp quyền truy cập.

#### Cấu trúc chuỗi JWT (`Header.Payload.Signature`):

```text
+-----------------------+ . +-----------------------+ . +-----------------------+
|        Header         |   |        Payload        |   |       Signature       |
| (Thuật toán & Loại)   |   |   (Thông tin claims)  |   |    (Chữ ký số HMAC)   |
+-----------------------+   +-----------------------+   +-----------------------+
```

- **Header**: Khai báo thuật toán mã hóa (ví dụ `HS256`) và loại token (`JWT`).
- **Payload**: Chứa các claims (thông tin định danh như `sub`, `iss`, `exp`). Payload chỉ encode Base64Url, không được lưu mật khẩu hay dữ liệu nhạy cảm.
- **Signature**: Tạo từ `HMACSHA256(base64Url(Header) + "." + base64Url(Payload), secret_key)`. Dùng để kiểm tra token có bị can thiệp thay đổi hay không.

### 4.3. Giao thức OAuth

OAuth là giao thức ủy quyền (Authorization) cho phép ứng dụng bên thứ ba truy cập tài nguyên thay mặt người dùng mà không cần sở hữu mật khẩu của người dùng.

- **OAuth 1.0**: Dùng chữ ký số phức tạp cho từng request, độ an toàn cao nhưng khó triển khai.
- **OAuth 2.0**: Sử dụng cơ chế Access Token, hỗ trợ nhiều luồng xác thực linh hoạt cho web và mobile, dễ triển khai hơn.

#### Cơ chế Access Token và Refresh Token:
1. Người dùng cấp quyền, Client nhận được **Access Token** (thời hạn ngắn) và **Refresh Token** (thời hạn dài).
2. Client đính kèm Access Token khi gọi API đến Resource Server.
3. Khi Access Token hết hạn, Client gửi Refresh Token đến Authorization Server để nhận Access Token mới mà không bắt người dùng nhập lại mật khẩu.
4. Nếu Refresh Token hết hạn hoặc bị thu hồi, người dùng phải tiến hành đăng nhập lại từ đầu.
