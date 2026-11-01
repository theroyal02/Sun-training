# TypeScript Engineering Handbook

## 1. Bản chất & Cơ chế hoạt động

TypeScript là một static type checker cho JavaScript. Toàn bộ thông tin type chỉ tồn tại ở bước biên dịch (Compile-time) và bị triệt tiêu hoàn toàn khi dịch sang JavaScript (Type Erasure).

- **Structural Typing (Duck Typing)**: Hai types có cùng hình thái cấu trúc (shape) được coi là tương thích, không phụ thuộc vào tên gọi khai báo.
- **Fail-fast**: Phát hiện type mismatch, undefined access và typos trước khi code được nạp vào runtime.

---

## 2. Thiết lập `tsconfig.json` chuẩn Production

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
```

---

## 3. Hệ thống Kiểu cốt lõi (Type Fundamentals)

### Suy luận kiểu (Type Inference) vs Khai báo tường minh (Annotation)
Tận dụng inference cho các biến nguyên thủy; chỉ chú thích tường minh khi xử lý logic phức tạp, contract hàm hoặc biến chưa khởi tạo.

```typescript
let active = true;           // Inferred boolean
const port = 3000;           // Literal type: 3000
let dynamicId: string | null = null; // Cần explicit annotation
```

### So sánh: `unknown` vs `any` vs `never`

| Kiểu | Đặc điểm | Trường hợp sử dụng |
| :--- | :--- | :--- |
| `any` | Tắt hoàn toàn type checker (Bypass safety) | **Hạn chế dùng tối đa**. |
| `unknown` | Type-safe counterpart của `any`. Phải narrow type trước khi thao tác | Dữ liệu từ API bên ngoài, raw JSON payload. |
| `never` | Đại diện cho giá trị không bao giờ xuất hiện | Hàm throw error vô tận, kiểm tra Exhaustive check. |
| `void` | Hàm không trả về giá trị có nghĩa | Function callbacks không return. |

```typescript
function parsePayload(input: unknown): string {
  if (typeof input === "string") return input.toUpperCase();
  if (input instanceof Error) return input.message;
  throw new Error("Payload không hợp lệ");
}
```

---

## 4. Object Types: `interface` vs `type`

| Tiêu chí | `interface` | `type` (Type Alias) |
| :--- | :--- | :--- |
| **Mở rộng (Extensibility)** | Kế thừa bằng `extends`, hỗ trợ Declaration Merging | Hợp nhất bằng Intersection (`&`) |
| **Tính linh hoạt** | Chỉ mô tả Object / Class shape | Mô tả được cả Union, Tuple, Primitive types |
| **Khuyến nghị** | Dùng cho Public API contracts, DTO, OOP | Dùng cho Union types, Utility types, Transformers |

```typescript
// Interface cho OOP contracts
interface BaseEntity {
  readonly id: string;
  createdAt: Date;
}

interface UserEntity extends BaseEntity {
  name: string;
  email?: string; // Optional property
}

// Type alias cho union / state machine
type NetworkStatus = "idle" | "loading" | "success" | "failed";
```

---

## 5. Discriminated Unions & Exhaustive Check

Mô hình hóa State Machine bằng một thuộc tính định danh chung (`kind` hoặc `status`):

```typescript
type ApiResponse =
  | { status: "success"; data: { id: number; name: string } }
  | { status: "error"; message: string }
  | { status: "loading" };

function handleResponse(res: ApiResponse) {
  switch (res.status) {
    case "success":
      return console.log(res.data.name);
    case "error":
      return console.error(res.message);
    case "loading":
      return console.info("Fetching...");
    default: {
      // Exhaustive Check: đảm bảo không bao giờ bỏ sót case mới thêm vào
      const _exhaustiveCheck: never = res;
      throw new Error(`Unhandled case: ${_exhaustiveCheck}`);
    }
  }
}
```

---

## 6. Type Guards & Narrowing

### User-defined Type Guard (`param is Type`)
Hữu ích khi cần tái sử dụng logic kiểm tra kiểu ở runtime:

```typescript
interface AdminUser {
  role: "ADMIN";
  permissions: string[];
}

function isAdmin(user: unknown): user is AdminUser {
  return (
    typeof user === "object" &&
    user !== null &&
    "role" in user &&
    (user as { role: unknown }).role === "ADMIN"
  );
}
```

---

## 7. Generics & Constraints

Tham số hóa kiểu dữ liệu giúp code linh hoạt nhưng vẫn an toàn tuyệt đối:

```typescript
interface Identifiable {
  id: string | number;
}

// Giới hạn generic T bắt buộc phải có thuộc tính của Identifiable
function findRecordById<T extends Identifiable>(records: T[], id: T["id"]): T | undefined {
  return records.find((item) => item.id === id);
}

// keyof & Index Access Type
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

---

## 8. Essential Utility Types

```typescript
interface Task {
  id: string;
  title: string;
  completed: boolean;
  assignedTo?: string;
}

// 1. Partial: Mọi thuộc tính thành optional (Update DTO)
type UpdateTaskPayload = Partial<Task>;

// 2. Pick & Omit: Trích xuất hoặc lược bỏ fields
type TaskSummary = Pick<Task, "id" | "title">;
type CreateTaskPayload = Omit<Task, "id">;

// 3. Readonly: Đóng băng mutating ở compile-time
type ImmutableTask = Readonly<Task>;

// 4. Record: Định nghĩa Key-Value pairs an toàn
type TaskRegistry = Record<string, Task>;

// 5. Awaited & ReturnType: Trích xuất kiểu trả về của async function
async function getTaskFromDb() {
  return { id: "1", title: "Review PR" };
}
type TaskFromDb = Awaited<ReturnType<typeof getTaskFromDb>>; // { id: string, title: string }
```

---

## 9. Null Safety Operators

```typescript
type Profile = {
  settings?: {
    theme?: string;
  };
};

const user: Profile = {};

// Optional chaining (?.): Chặn crash nếu chuỗi thuộc tính là null/undefined
const activeTheme = user.settings?.theme;

// Nullish Coalescing (??): Chỉ fallback khi giá trị là null hoặc undefined
const selectedPort = process.env.PORT ?? 3000; // Khác biệt với `||` (vẫn nhận 0 hoặc "")
```

---

## 10. Core Best Practices

1. **Strict Null Checks**: Luôn bật trong tsconfig để tiêu diệt lỗi `Cannot read properties of undefined`.
2. **Hạn chế Type Assertion (`as`)**: `as` chỉ che mắt compiler. Dùng Type Guard hoặc schema validation thay thế.
3. **Ưu tiên `as const`**: Thay thế `enum` bằng Object hằng số `as const` để tiết kiệm bundle size và kiểm soát type chặt hơn:
   ```typescript
   const Role = {
     ADMIN: "ADMIN",
     USER: "USER",
   } as const;

   type Role = (typeof Role)[keyof typeof Role]; // "ADMIN" | "USER"
   ```
4. **Validate I/O Boundaries**: Sử dụng các thư viện schema (Zod, Valibot) khi parse dữ liệu từ HTTP requests hoặc external APIs trước khi chuyển sang TypeScript types.

