# Node.js Architecture & Core Internals

## 1. Kiến trúc nền tảng (Runtime Architecture)

Node.js là runtime JavaScript chạy phía server xây dựng trên V8 Engine (Google) và thư viện Libuv (C/C++).

```
+-------------------------------------------------------+
|                   V8 Engine (JS Code)                 |
+-------------------------------------------------------+
|                    Node.js Core API                   |
+-------------------------------------------------------+
|           Libuv (Event Loop + Thread Pool)            |
+-------------------------------------------------------+
|               OS Kernel (Epoll / Kqueue / IOCP)       |
+-------------------------------------------------------+
```

- **Single Threaded**: Main thread chỉ chạy code JavaScript người dùng viết.
- **Non-blocking I/O**: Giao toàn bộ thao tác File/Network cho OS kernel hoặc Thread Pool (Libuv mặc định 4 threads).
- **Phù hợp**: Web API, Microservices, Realtime I/O (WebSocket, SSE).
- **Không phù hợp**: CPU-bound tasks (Machine Learning, Video Encoding, xử lý ma trận lớn) - sẽ làm treo main thread.

---

## 2. Package Management & Semantic Versioning

- `package.json`: Khai báo manifest dự án, scripts và ranges phiên bản dependencies.
- `package-lock.json`: Lưu snapshot cây phụ thuộc và hash toàn vẹn (integrity hash). Bắt buộc commit để tránh "works on my machine".

```bash
# Cài đặt production dependency
npm i express dotenv

# Cài đặt dev dependency
npm i -D @types/node tsx

# Cài đặt chuẩn từ package-lock (khuyên dùng trong CI/CD)
npm ci
```

Semantic Versioning (`^` vs `~`):
- `^1.2.3`: Nhận updates tương thích ngược: `>= 1.2.3 < 2.0.0` (Minor + Patch).
- `~1.2.3`: Chỉ nhận bản vá lỗi: `>= 1.2.3 < 1.3.0` (Chỉ Patch).

---

## 3. Module System: CommonJS vs ESM

Node.js hỗ trợ cả hai hệ thống module. Không trộn lẫn tuỳ tiện.

### CommonJS (CJS)
- Cơ chế nạp: Đồng bộ (Synchronous), resolve lúc runtime.
- Cú pháp: `require()` và `module.exports`.

```javascript
// math.cjs
const sum = (a, b) => a + b;
module.exports = { sum };

// main.cjs
const { sum } = require("./math.cjs");
```

### ECMAScript Modules (ESM)
- Cơ chế nạp: Bất đồng bộ (Asynchronous), phân tích tĩnh (Static analysis) lúc compile-phase.
- Cú pháp: `import` và `export`. Kích hoạt bằng `"type": "module"` trong `package.json` hoặc đuôi `.mjs`.

```javascript
// math.mjs
export const sum = (a, b) => a + b;

// main.mjs
import { sum } from "./math.mjs";
```

---

## 4. Xử lý Bất đồng bộ (Asynchronous Pattern)

### Promises & Async/Await
Luôn xử lý reject tường minh, không để lọt Unhandled Rejection.

```javascript
import fs from "fs/promises";

async function readProjectConfig(path) {
  try {
    const raw = await fs.readFile(path, "utf-8");
    return JSON.parse(raw);
  } catch (err) {
    if (err.code === "ENOENT") throw new Error(`File not found: ${path}`);
    throw err;
  }
}
```

### Concurrent Execution
Dùng `Promise.all` khi các tác vụ độc lập cần chạy song song, hoặc `Promise.allSettled` nếu cần kiểm tra kết quả từng tác vụ độc lập với rủi ro lỗi.

```javascript
const [users, orders] = await Promise.all([fetchUsers(), fetchOrders()]);
```

---

## 5. Event Loop & Execution Phases

Event loop điều phối các callbacks theo các pha tuần tự:

1. **Timers**: Thực thi callbacks từ `setTimeout()` và `setInterval()`.
2. **Pending Callbacks**: Xử lý I/O callbacks bị hoãn lại từ vòng lặp trước.
3. **Idle, Prepare**: Tác vụ nội bộ của Libuv.
4. **Poll**: Đợi I/O mới; chạy các callback liên quan đến I/O.
5. **Check**: Thực thi callbacks của `setImmediate()`.
6. **Close Callbacks**: Dọn dẹp tài nguyên (vd: `socket.on('close', ...)`).

> **Microtask Queue**: `process.nextTick()` và `Promise.then()` được ưu tiên chạy ngay lập tức giữa các pha trước khi Event Loop chuyển sang pha tiếp theo. `process.nextTick()` có độ ưu tiên cao hơn Promise.

---

## 6. Native File System & Path Handling

Tuyệt đối không dùng toán tử ghép chuỗi `+` hoặc template literals để nối đường dẫn file. Luôn dùng module `path`.

```javascript
import path from "node:path";
import fs from "node:fs/promises";

// Tự động tương thích dấu phân cách '\' (Windows) và '/' (POSIX)
const targetPath = path.resolve(process.cwd(), "data", "records.json");

await fs.writeFile(targetPath, JSON.stringify({ ok: true }), "utf-8");
```

---

## 7. Buffer & Stream Architecture

Tránh đọc toàn bộ file lớn vào bộ nhớ (RAM exhaustion). Dùng Stream để xử lý theo chunk.

```
+------------+  Chunk 64KB  +---------+  Chunk 64KB  +------------+
| ReadStream | -----------> | Transform| -----------> | WriteStream|
+------------+              +---------+              +------------+
```

```javascript
import fs from "node:fs";
import { pipeline } from "node:stream/promises";
import zlib from "node:zlib";

async function compressFile(source, target) {
  // pipeline tự động quản lý đóng Stream và giải phóng Backpressure
  await pipeline(
    fs.createReadStream(source),
    zlib.createGzip(),
    fs.createWriteStream(target)
  );
}
```

---

## 8. Event-Driven Architecture (EventEmitter)

Mô hình Publisher-Subscriber nội bộ của Node.js:

```javascript
import { EventEmitter } from "node:events";

class OrderQueue extends EventEmitter {
  process(orderId) {
    this.emit("order:received", { orderId, timestamp: Date.now() });
  }
}

const queue = new OrderQueue();

// Đăng ký listener
queue.on("order:received", (data) => {
  console.log("Processing order:", data.orderId);
});

// BẮT BUỘC: Lắng nghe sự kiện 'error' để tiến trình không bị crash
queue.on("error", (err) => {
  console.error("Critical stream error:", err.message);
});
```

---

## 9. Native HTTP Web Server

Tạo server thuần chuẩn hiệu năng không phụ thuộc framework:

```javascript
import http from "node:http";

const server = http.createServer(async (req, res) => {
  const { method, url } = req;

  if (method === "GET" && url === "/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    return res.end(JSON.stringify({ status: "healthy" }));
  }

  res.writeHead(404, { "Content-Type": "text/plain" });
  res.end("Not Found");
});

server.listen(8080, () => console.log("Server listening on :8080"));
```

---

## 10. Production Checklist

1. **Không chặn Event Loop**: Tránh các hàm synchronous (`fs.readFileSync`, `crypto.pbkdf2Sync`) trong request pipeline.
2. **Cấu hình Graceful Shutdown**: Lắng nghe `SIGTERM` / `SIGINT` để đóng database pool và HTTP connections trước khi kill process.
3. **Quản lý biến môi trường**: Validate config khi bootstrap app bằng dotenv hoặc thư viện schema validation (Zod, Joi).
4. **Giám sát uncaught exceptions**:
   ```javascript
   process.on("unhandledRejection", (reason) => {
     console.error("Unhandled Rejection:", reason);
     process.exit(1);
   });
   ```

