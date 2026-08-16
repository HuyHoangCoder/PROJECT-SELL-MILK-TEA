# Bài 06 — Vòng đời một request: Router → Controller → Model

> **Phần 1 · Backend** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [05 — Mongoose Model](05-mongoose-model.md) · Bài sau: [07 — CRUD trọn vẹn với Category](07-crud-category.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Kể lại được **7 chặng** một request phải đi qua, từ lúc bấm Send tới lúc JSON hiện ra.
- Hiểu vì sao dự án tách ba thư mục `routes/` — `controllers/` — `models/`.
- Đọc hiểu từng dòng `yotea-be/src/routes/category.js` và `yotea-be/src/controllers/category.js`.
- Phân biệt dứt khoát `req.params`, `req.query`, `req.body`.
- Giải thích được vì sao route khai báo `"/category"` mà URL thật lại là `/api/category`.
- Tránh được cái bẫy **thứ tự khai báo route** khiến một API "biến mất" không báo lỗi.
- **Tự tay** viết controller + route đầu tiên cho Topping và gọi được bằng Postman.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 05](05-mongoose-model.md) — trong máy đã có `yotea-be/src/models/topping.js` do chính bạn viết.
- Đã hiểu `app.js` ở [Bài 04](04-express-va-appjs.md): `app.use()`, middleware, prefix `/api`.
- MongoDB đang chạy, backend `npm start` lên được cổng 8080. Có Postman và MongoDB Compass.

> 💡 **Nối mạch:** ở bài trước bạn đã **khai báo hình dạng dữ liệu** cho Topping bằng một
> Mongoose Schema — nhưng model đó đang nằm im, chưa ai gọi tới. Bài này ta làm tiếp phần
> **đường dây điện**: dựng route và controller để thế giới bên ngoài chạm được vào nó qua HTTP.

---

## 1. Vòng đời một request — bức tranh toàn cảnh

Bạn vào quán Yotea gọi ly trà sữa: **thu ngân** ghi phiếu (Router), **bảo vệ** kiểm tra thẻ
nếu là đơn nội bộ (Middleware), **pha chế** đọc phiếu và làm (Controller), đi lấy nguyên liệu
trong **kho** (Model → MongoDB), rồi ly trà được đưa ra quầy (`res.json`). Điểm mấu chốt:
**thu ngân không pha chế, pha chế không giữ kho.** Express cũng vậy.

```
 CLIENT ── GET http://localhost:8080/api/category?_sort=createdAt ──┐
 (React / Postman)                                                  ▼
 ┌── 1. app.listen(8080) ─────────────────── yotea-be/src/app.js:52 ──┐
 │      Express nhận gói tin, dựng ra 2 object: req và res            │
 ├── 2. MIDDLEWARE TOÀN CỤC ──────────────── yotea-be/src/app.js:25-27┤
 │      express.json() → đổ body JSON vào req.body                    │
 │      cors()         → gắn header cho FE cổng 3000 gọi sang         │
 │      morgan("tiny") → in 1 dòng log ra terminal                    │
 ├── 3. MOUNT PREFIX ─────────────────────── yotea-be/src/app.js:29-42┤
 │      URL bắt đầu bằng "/api"? → CẮT BỎ "/api", đưa phần còn lại    │
 │      ("/category") lần lượt cho 14 router thử khớp                 │
 ├── 4. ROUTER ─────────────────── yotea-be/src/routes/category.js:10 ┤
 │      router.get("/category", list) → KHỚP! (đúng method + path)    │
 ├── 5. MIDDLEWARE CỦA ROUTE (route đọc thì không có, route ghi thì có)┤
 │      requireSignin → isAuth → isAdmin ... mỗi chốt gọi next()      │
 ├── 6. CONTROLLER ────── yotea-be/src/controllers/category.js:34-118 ┤
 │      Đọc req.query, dựng filter/sortOpt, rồi gọi xuống model       │
 ├── 7. MODEL ──────────────────── yotea-be/src/models/category.js:22 ┤
 │      CategoryModel.find(filter)...exec()  ──►  MongoDB :27017      │
 └────────────────────────────────────────────────────────────────────┘
                                    │ mảng document
                                    ▼
                    res.json(newCates) ──► HTTP 200 + JSON ──► CLIENT
```

| # | Chặng | File thật trong dự án | Hỏng thì thấy gì |
|---|---|---|---|
| 1 | Server lắng nghe | `yotea-be/src/app.js:52` | Postman báo `ECONNREFUSED` |
| 2 | Middleware toàn cục | `yotea-be/src/app.js:25-27` | `req.body` là `undefined` |
| 3 | Mount prefix `/api` | `yotea-be/src/app.js:29-42` | `Cannot GET /category` |
| 4 | Router khớp path | `yotea-be/src/routes/category.js` | `Cannot GET /api/category` |
| 5 | Middleware của route | `yotea-be/src/middlewares/checkAuth.js` | `401 Bạn không phải là Admin` |
| 6 | Controller | `yotea-be/src/controllers/category.js` | JSON lỗi 400 kèm `message` |
| 7 | Model → MongoDB | `yotea-be/src/models/category.js` | Request treo, hoặc lỗi validate |

> 💡 **Mẹo debug vàng:** khi API "không chạy", hãy hỏi *"nó chết ở chặng thứ mấy?"* Nhìn
> terminal backend: **morgan không in dòng nào** → chưa tới chặng 3-4 (sai URL, sai cổng).
> In `404` → chết ở chặng 4. In `400` → đã vào tới controller rồi.

---

## 2. Vì sao phải tách ba lớp?

Bạn hoàn toàn **có thể** nhét cả dự án vào một `app.js` dài 3000 dòng — nó chạy được. Nhưng
rồi muốn tìm *"API lấy danh mục nằm đâu"* bạn phải `Ctrl+F` giữa 3000 dòng; muốn đổi cách
truy vấn thì phải sửa **14 chỗ** vì logic bị chép đi chép lại.

**Separation of concerns** (tách bạch mối quan tâm): *mỗi file chỉ lo đúng một loại việc.*

| Lớp | Thư mục | Trả lời câu hỏi | Được biết | KHÔNG được biết |
|---|---|---|---|---|
| **Routes** | `yotea-be/src/routes/` | *"URL nào, method nào, qua chốt nào?"* | Tên hàm controller | Cách truy vấn database |
| **Controllers** | `yotea-be/src/controllers/` | *"Nhận dữ liệu rồi làm gì, trả gì?"* | `req`, `res`, các model | URL của chính nó |
| **Models** | `yotea-be/src/models/` | *"Dữ liệu có hình dạng thế nào?"* | Schema, kiểu dữ liệu | `req`, `res`, HTTP |

Bốn lợi ích rất cụ thể khi bảo trì:

1. **Tìm code nhanh.** "API xoá danh mục lỗi" → mở `routes/category.js` thấy `remove` → nhảy
   sang `controllers/category.js` tìm `export const remove`. Mười giây.
2. **Đổi URL không đụng logic.** Đổi `/api/category` thành `/api/categories`? Sửa đúng **một
   dòng** trong file route; controller không hề hay biết.
3. **Dùng lại được.** Model `Category` được cả `controllers/category.js` lẫn
   `controllers/product.js` cùng dùng.
4. **Test được từng mảnh.** Gọi thẳng hàm controller với một `req` giả, không cần dựng server.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** ba lớp đã tách, nhưng lớp thứ tư — **service** (logic
> nghiệp vụ thuần) — thì chưa có. Hệ quả: đoạn code dựng bộ lọc query bị **chép nguyên xi vào
> 13 controller**, khoảng 533 dòng trùng lặp ([Bài 09](09-bo-loc-query.md) sẽ đếm cho bạn
> xem). Ta thực sự refactor ở [Bài 34](34-refactor-du-an.md).

---

## 3. Soi code thật — lớp Router

`yotea-be/src/routes/category.js:1-16`

```js
import { Router } from "express";
import { create, list, read, remove, update, getProductByCate } from "../controllers/category";
import { userById } from "../controllers/user";
import { isAdmin, isAuth, requireSignin } from "../middlewares/checkAuth";

const router = Router();

router.post("/category/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/category/:slug", read);
router.get("/category", list);
router.put("/category/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.delete("/category/:id/:userId", requireSignin, isAuth, isAdmin, remove);

router.param("userId", userById)

export default router;
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import { Router } from "express"` | Lấy class `Router` khỏi Express (có ngoặc nhọn → named export) |
| 2 | `import { create, list, read, ... }` | Lấy 6 hàm xử lý. Router chỉ biết **tên hàm**, không biết bên trong làm gì |
| 3 | `import { userById }` | Hàm tra user theo id, dùng cho `router.param` ở dòng 14 |
| 4 | `import { isAdmin, isAuth, requireSignin }` | Ba chốt kiểm tra quyền — [Bài 12](12-phan-quyen-middleware.md) |
| 6 | `const router = Router()` | Tạo một **mini-app** riêng, có đủ `.get/.post/.put/.delete` như `app` |
| 8 | `router.post("/category/:userId", ..., create)` | POST → **tạo mới**. Các hàm sau đường dẫn chạy **lần lượt trái sang phải**; `create` là chốt cuối |
| 9 | `router.get("/category/:slug", read)` | GET có thêm đoạn đường dẫn → **đọc một** danh mục theo slug |
| 10 | `router.get("/category", list)` | GET trơn → **đọc danh sách** |
| 11 | `router.put("/category/:id/:userId", ...)` | PUT → **cập nhật**. Đường dẫn có **hai** tham số |
| 12 | `router.delete("/category/:id/:userId", ...)` | DELETE → **xoá** |
| 14 | `router.param("userId", userById)` | *"Hễ đường dẫn nào có `:userId` thì chạy `userById` trước"* — trước cả `requireSignin` |
| 16 | `export default router` | Xuất mini-app này ra để `app.js` gắn vào |

**`Router()` là cái gì?** Hình dung nó như **một cuốn sổ đăng ký**: bạn ghi *"ai gọi GET
`/category` thì đưa cho hàm `list`"*. Cuốn sổ tự nó chưa làm gì — nó chỉ có tác dụng khi được
giao cho server bằng `app.use("/api", categoryRouter)` (`yotea-be/src/app.js:29`). Đây là lý
do dự án có **14 file route riêng** thay vì viết `app.get(...)` 60 lần trong `app.js`.

Chú ý: **cùng đường dẫn nhưng khác method là hai route hoàn toàn khác nhau.**
`POST /api/category/abc` và `PUT /api/category/abc` không hề đụng nhau.

**`export default router` — vì sao không ngoặc nhọn?** Vì mỗi file route chỉ xuất ra **đúng
một thứ**, nên bên `app.js` được đặt tên tuỳ ý — `yotea-be/src/app.js:7`:

```js
import categoryRouter from "./routes/category";
```

Tên `categoryRouter` do người viết `app.js` tự đặt; trong `routes/category.js` biến đó tên là
`router`. Cả hai đều đúng, vì `export default` không ràng buộc tên.

### 3.1. Khai báo `"/category"` mà URL thật là `/api/category`?

Bí mật nằm ở tham số thứ nhất của `app.use`. `yotea-be/src/app.js:29-31`

```js
app.use("/api", categoryRouter);
app.use("/api", productRouter);
app.use("/api", sliderRouter);
```

(và 11 dòng tương tự nữa, tới `yotea-be/src/app.js:42`)

`app.use("/api", categoryRouter)` đọc là: *"mọi URL bắt đầu bằng `/api`, hãy **cắt bỏ** phần
`/api` rồi đưa phần còn lại cho `categoryRouter` xử lý."*

```
URL client gọi :  /api/category/tra-sua
                  └┬─┘└──────┬────────┘
                   │         └── phần router NHÌN THẤY
                   └── app.use nuốt mất, router KHÔNG nhìn thấy
```

Công thức phải thuộc: **URL thật = prefix ở `app.use` + path khai báo trong router.**

Vì cả 14 router dùng chung prefix `/api`, **tên resource bắt buộc phải nằm trong file route**.
Nếu hai router cùng khai báo `/category`, router nào được `app.use` **trước** (số dòng nhỏ
hơn) sẽ thắng; router sau bị che vĩnh viễn.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** tên resource **lẫn lộn số ít / số nhiều**:
> `/api/products` (số nhiều) nhưng `/api/category`, `/api/slider`, `/api/orderDetail` (số ít,
> cái cuối còn camelCase). REST chuẩn khuyên **danh từ số nhiều, chữ thường, gạch nối**:
> `/api/categories`, `/api/order-details`. Đổi bây giờ sẽ vỡ toàn bộ frontend nên ta giữ
> nguyên — nhưng khi tự viết Topping ở mục 6, hãy đặt `/toppings` cho đúng chuẩn.

---

## 4. Soi code thật — lớp Controller

Mọi hàm controller trong dự án đều cùng hình dạng: `export const tenHam = async (req, res) => {...}`

- `req` (**request**) — Express dựng sẵn, chứa **mọi thứ client gửi lên**.
- `res` (**response**) — công cụ để **trả lời**. Mỗi request chỉ được trả lời **một lần**.

### 4.1. `req.params` — hàm `read`

`yotea-be/src/controllers/category.js:19-32`

```js
export const read = async (req, res) => {
    const filter = { slug: req.params.slug };
    const populate = req.query["_expand"];

    try {
        const category = await CategoryModel.findOne(filter).select("-__v").populate(populate).exec();
        res.json(category);
    } catch (error) {
        res.status(400).json({
            message: "Không tìm thấy danh mục",
            error
        });
    }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 19 | `async (req, res) =>` | Hàm bất đồng bộ, vì tí nữa phải `await` database |
| 20 | `req.params.slug` | Lấy giá trị ở vị trí `:slug` của route dòng 9 file `routes/category.js` |
| 21 | `req.query["_expand"]` | Lấy tuỳ chọn sau dấu `?` trên URL |
| 23 | `try {` | Lưới an toàn — không có nó, một lỗi truy vấn làm **sập cả server** |
| 24 | `await CategoryModel.findOne(...)` | Gọi xuống **lớp Model**; `.exec()` để Mongoose trả Promise thật |
| 25 | `res.json(category)` | Trả JSON, status mặc định **200** |
| 27-30 | `res.status(400).json({...})` | Lỗi → mã 400 kèm `message` tiếng Việt cho FE hiển thị |

Gọi `GET http://localhost:8080/api/category/tra-sua` thì `req.params` = `{ slug: "tra-sua" }`.
Dấu hai chấm nghĩa là *"đoạn này là biến, nhét giá trị thật vào `req.params` với tên đứng sau dấu `:`"*.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** slug không tồn tại → `findOne` trả `null` → dòng 25
> `res.json(null)` với status **200 OK**. Frontend tưởng thành công rồi nổ khi đọc
> `category.name`. Đúng ra phải trả **404**:
> `if (!category) return res.status(404).json({ message: "Không tìm thấy danh mục" });`

### 4.2. `req.query` — đầu hàm `list`

`yotea-be/src/controllers/category.js:34-40`

```js
export const list = async (req, res) => {
    const populate = req.query["_expand"];

    let sortOpt = {};
    if (req.query["_sort"]) {
        const sortArr = req.query["_sort"].split(",");
        const orderArr = (req.query["_order"] || "").split(",");
```

Với `GET /api/category?_sort=createdAt&_order=desc&_limit=5`, Express tự bóc chuỗi sau dấu `?`
thành `req.query` = `{ _sort: "createdAt", _order: "desc", _limit: "5" }`.

Ba điều phải nhớ:

1. **Giá trị luôn là chuỗi**, kể cả `_limit=5` → `"5"`. Muốn tính toán phải `Number(...)`.
2. **Luôn là tuỳ chọn** — không truyền thì `undefined`, nên code phải phòng thủ, như dòng 40
   dùng `(req.query["_order"] || "")` để tránh gọi `.split()` trên `undefined`.
3. Toàn bộ cỗ máy lọc/sắp xếp/phân trang của dự án chạy trên `req.query` — mổ xẻ trọn vẹn ở
   [Bài 09](09-bo-loc-query.md).

### 4.3. `req.body` — hàm `create`

`yotea-be/src/controllers/category.js:5-17`

```js
export const create = async (req, res) => {
    req.body.slug = slugify(req.body.name);

    try {
        const category = await new CategoryModel(req.body).save();
        res.json(category);
    } catch (error) {
        res.status(400).json({
            message: "Thêm danh mục thất bại",
            error
        });
    }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 6 | `req.body.slug = slugify(req.body.name)` | Đọc `name` client gửi lên, sinh slug, **ghi ngược** vào `req.body` |
| 9 | `new CategoryModel(req.body).save()` | Đưa nguyên `req.body` cho Model; Mongoose lọc theo Schema rồi lưu |
| 10 | `res.json(category)` | Trả document vừa tạo, đã có `_id`, `createdAt` |
| 12-15 | `res.status(400).json(...)` | Thất bại → 400 + `message` + object `error` gốc |

`req.body` chỉ có dữ liệu **nhờ** middleware `express.json()` ở `yotea-be/src/app.js:25`. Bỏ
dòng đó đi, `req.body` là `undefined` và dòng 6 nổ ngay.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dòng 6 nằm **ngoài** `try`. Client quên gửi `name` →
> `slugify(undefined)` ném lỗi khi `try` chưa kịp bắt → Express 4 không bắt được lỗi async →
> **Node giết cả tiến trình server**. Chỉ cần đẩy dòng 6 vào trong `try`. Chi tiết ở
> [Bài 08](08-slug-slugify.md).

### 4.4. Các phương thức của `res`

| Cách viết | Status | Dùng khi |
|---|---|---|
| `res.json(data)` | **200** (mặc định) | Thành công. Tự set `Content-Type: application/json` |
| `res.status(400).json({...})` | 400 | Client gửi sai — thấy ở **mọi** `catch` của dự án |
| `res.status(401).json({...})` | 401 | Không đủ quyền — `isAdmin` trong `checkAuth.js` |
| `res.status(404).json({...})` | 404 | Không tìm thấy — **dự án chưa dùng**, nhưng bạn nên dùng |

`res.status(...)` trả về chính `res` nên nối chuỗi được.

> ⚠️ **Chỉ được trả lời MỘT lần.** Gọi `res.json()` hai lần trong cùng request sẽ ra lỗi
> `Cannot set headers after they are sent to the client`. Vì thế nên viết
> `return res.status(400).json(...)` — chữ `return` chặn code chạy tiếp xuống dưới.

### 4.5. Bảng vàng: `req.params` vs `req.query` vs `req.body`

| | `req.params` | `req.query` | `req.body` |
|---|---|---|---|
| **Nằm đâu** | Trong đường dẫn, chỗ có `:` | Sau dấu `?` | Trong thân request, không trên URL |
| **Ai định nghĩa** | Bạn, khi khai báo route | Client tuỳ ý gắn thêm | Client gửi lên |
| **Bắt buộc?** | Có — thiếu là không khớp route | Không | Không |
| **Kiểu dữ liệu** | Luôn **string** | Luôn **string** (hoặc mảng string) | Đúng kiểu JSON đã gửi |
| **Cần middleware?** | Không | Không | **Có** — `express.json()` (`app.js:25`) |
| **Dùng cho** | Định danh tài nguyên | Lọc / sắp xếp / phân trang | Dữ liệu tạo mới, cập nhật |
| **Method hay đi kèm** | mọi method | GET | POST, PUT, PATCH |

Ba ví dụ bằng **URL thật của Yotea**:

```
① GET http://localhost:8080/api/category/tra-sua-truyen-thong      (khớp routes/category.js:9)
  → req.params = { slug: "tra-sua-truyen-thong" }   req.query = {}   req.body = {}

② GET http://localhost:8080/api/products?categoryId=6249a1&_sort=createdAt&_order=desc&_limit=8
                                                                    (khớp routes/product.js:10)
  → req.params = {}
  → req.query  = { categoryId: "6249a1", _sort: "createdAt", _order: "desc", _limit: "8" }

③ POST http://localhost:8080/api/category/6250f1a2b3c4d5e6f7890123 (khớp routes/category.js:8)
  Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
  Body  : { "name": "Trà trái cây", "image": "https://res.cloudinary.com/..." }
  → req.params = { userId: "6250f1a2b3c4d5e6f7890123" }
  → req.body   = { name: "Trà trái cây", image: "https://res.cloudinary.com/..." }
```

> 🔒 **Ghi chú bảo mật:** để ý `:userId` — id người đang đăng nhập bị **nhét vào URL**. Nó
> đúng ra phải lấy từ token trong header (`req.auth`), vì URL bị ghi vào log server, lịch sử
> trình duyệt, và ai cũng sửa được. Xem [Bài 12](12-phan-quyen-middleware.md) và
> [Bài 33](33-ra-soat-bao-mat.md).

---

## 5. Cái bẫy chết người: Express khớp route theo **THỨ TỰ KHAI BÁO**

Express duyệt danh sách route **từ trên xuống** và dừng ở **cái đầu tiên khớp**. Không có
chuyện "chọn cái khớp sát nhất" như một số framework khác.

`yotea-be/src/routes/product.js:8-13`

```js
router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/products/:slug", read);
router.get("/products", list);
router.put("/products/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.patch("/products/userUpdate/:id", clientUpdate);
router.delete("/products/:id/:userId", requireSignin, isAuth, isAdmin, remove);
```

Dòng 9 là `"/products/:slug"` — một **route động**. `:slug` nuốt **bất kỳ** chuỗi nào ở vị trí
đó: `tra-sua`, `123`, `abc`, và cả `moi-nhat`.

**Tình huống hỏng.** Sếp bảo: *"làm API `GET /api/products/moi-nhat` trả 10 sản phẩm mới
nhất."* Bạn thêm vào **sau** dòng 9 (code **bạn tự thêm**, dự án không có):

```js
// ❌ SAI
router.get("/products/:slug", read);        // ← dòng 9 sẵn có
router.get("/products/moi-nhat", newest);   // ← dòng bạn thêm, KHÔNG BAO GIỜ CHẠY
```

Gọi thử, bạn nhận về `null`. Không lỗi, không 404, terminal chỉ in `200`. Vì Express dừng ở
dòng 9: `:slug` = `"moi-nhat"` → `read()` tìm sản phẩm có slug `"moi-nhat"` → không có →
`res.json(null)`. Route bạn vừa thêm không bao giờ tới lượt.

**Cách sửa: route tĩnh phải đứng TRƯỚC route động.**

```js
// ✅ ĐÚNG — code bạn tự sửa
router.get("/products/moi-nhat", newest);   // tĩnh — lên trước
router.get("/products/:slug", read);        // động — hứng phần còn lại
```

| Ưu tiên | Kiểu route | Ví dụ |
|---|---|---|
| 1 (trên cùng) | Tĩnh, nhiều đoạn nhất | `/products/userUpdate/:id` |
| 2 | Tĩnh, ít đoạn | `/products/moi-nhat` |
| 3 | Động một tham số | `/products/:slug` |
| 4 (dưới cùng) | Trơn | `/products` |

Riêng `/products` (dòng 10) đứng **sau** `/products/:slug` (dòng 9) **không sao**, vì chúng
khác **số đoạn đường dẫn** — chỉ route **cùng số đoạn** mới tranh nhau.

> ⚠️ **Dự án đang "may mắn thoát nạn":** dòng 12 khai báo
> `router.patch("/products/userUpdate/:id", clientUpdate)` — `userUpdate` là đoạn **tĩnh** mà
> lại đứng **sau** `"/products/:slug"` ở dòng 9. Nó chạy được **chỉ vì** method là PATCH còn
> dòng 9 là GET → hai route không giẫm chân nhau. Ngày nào ai đó đổi nó thành
> `router.get(...)`, API sẽ chết lặng ngay lập tức.

### Muốn thêm một API mới thì đụng vào những file nào?

| # | File | Việc phải làm | Bắt buộc? |
|---|---|---|---|
| 1 | `yotea-be/src/models/<ten>.js` | Khai báo Schema + `export default model(...)` | ✅ (nếu là resource mới) |
| 2 | `yotea-be/src/controllers/<ten>.js` | Viết `create`/`read`/`list`/`update`/`remove`, mỗi hàm `export const` | ✅ |
| 3 | `yotea-be/src/routes/<ten>.js` | `const router = Router()`, khai báo path, `export default router` | ✅ |
| 4 | `yotea-be/src/app.js` — vùng import (dòng 7-20) | Thêm `import <ten>Router from "./routes/<ten>";` | ✅ |
| 5 | `yotea-be/src/app.js` — vùng mount (dòng 29-42) | Thêm `app.use("/api", <ten>Router);` | ✅ |
| 6 | `yotea-be/src/middlewares/checkAuth.js` | Không sửa — chỉ **import** vào route nếu cần khoá quyền | ⬜ Tuỳ |
| 7 | `yotea-fe/src/api/<ten>.js` | Hàm axios gọi API mới — [Bài 18](18-tang-api-axios.md) | ⬜ Khi làm frontend |

Nếu chỉ **thêm một endpoint** vào resource đã có, bạn chỉ đụng file **2** và **3**.

---

## 6. 🛠️ Tự tay làm — dựng đường dây cho Topping

> **Mục tiêu:** cuối phần này, gõ `GET http://localhost:8080/api/toppings` trong Postman sẽ
> nhận về `[]`. Đó là API **đầu tiên trong đời** do chính bạn dựng từ đầu tới cuối.

Nhắc lại mạch: **ở bài 05 bạn đã tạo `yotea-be/src/models/topping.js`**. Bài này ta làm tiếp
hai lớp còn thiếu — **controller** và **route** — rồi cắm chúng vào `app.js`.

> ⚠️ **Nguyên tắc bất di bất dịch:** toàn bộ code phần này là code **bạn tự viết thêm** — dự
> án Yotea gốc **không có** chức năng Topping. Tuyệt đối **không sửa** file có sẵn nào, ngoại
> trừ **đúng hai dòng** thêm vào `app.js` ở Bước 3.

### Bước 0 — Kiểm tra thành quả bài 05

Mở `yotea-be/src/models/topping.js`. Nếu làm đúng bài trước, file **bạn tự viết ở bài 05** đó
phải khai báo một Schema có `name` (String, `required`), `price` (Number, `required`), `image`
(String), kèm `{ timestamps: true }`, và kết thúc bằng `export default model("Topping", toppingSchema);`.
Chưa có file này? Quay lại [Bài 05](05-mongoose-model.md) làm cho xong đã.

### Bước 1 — Viết controller

Tạo **file mới** `yotea-be/src/controllers/topping.js` (code **bạn tự viết**):

```js
// yotea-be/src/controllers/topping.js  ← file MỚI, bạn tự tạo
import Topping from "../models/topping";

export const list = async (req, res) => {
  try {
    const toppings = await Topping.find().exec();
    res.json(toppings);
  } catch (error) {
    res.status(400).json({
      message: "Không tìm thấy topping",
      error,
    });
  }
};
```

| Phần | Vì sao viết vậy |
|---|---|
| `import Topping from "../models/topping"` | Không ngoặc nhọn vì model dùng `export default`. `../` = lùi từ `controllers/` ra `src/` rồi vào `models/` |
| `export const list` | Named export → bên route phải import **đúng tên** `{ list }` |
| `async (req, res)` | Đúng chữ ký controller, giống hệt `controllers/category.js:34` |
| `Topping.find()` | Không truyền filter = **lấy tất cả**. Bản rút gọn của `CategoryModel.find(filter)` (`controllers/category.js:93`) |
| `.exec()` | Bảo Mongoose trả Promise thật để `await` cho chuẩn |
| `try/catch` | Bắt chước mọi controller trong dự án — không có nó, lỗi database làm sập server |

> 💡 Ở bài này ta **cố tình** chưa làm bộ lọc `_sort`/`_limit`/`q`. Đơn giản trước, phức tạp
> sau — [Bài 09](09-bo-loc-query.md) sẽ nâng cấp đúng hàm này.

### Bước 2 — Viết route

Tạo **file mới** `yotea-be/src/routes/topping.js` (code **bạn tự viết**):

```js
// yotea-be/src/routes/topping.js  ← file MỚI, bạn tự tạo
import { Router } from "express";
import { list } from "../controllers/topping";

const router = Router();

router.get("/toppings", list);

export default router;
```

Đối chiếu với `yotea-be/src/routes/category.js` ở mục 3: y hệt bộ khung, chỉ bớt middleware và
4 route còn lại. Ba điểm cần nhớ: khai báo `"/toppings"` (**không** có `/api`, vì `app.js` sẽ
gắn prefix đó); dùng **số nhiều** cho đúng chuẩn REST; và `export default router` để `app.js`
import được.

### Bước 3 — Cắm vào `app.js` (đúng **hai dòng**)

Mở `yotea-be/src/app.js`. Thêm dòng import **ngay sau dòng 20**
(`import favoritesRouter from "./routes/favoritesProduct";`):

```js
import toppingRouter from "./routes/topping";   // ← dòng bạn thêm
```

Rồi thêm dòng mount **ngay sau dòng 42** (`app.use("/api", favoritesRouter);`):

```js
app.use("/api", toppingRouter);                 // ← dòng bạn thêm
```

> 💡 Từ giờ số dòng `app.js` của bạn sẽ **lệch** so với số dòng giáo trình trích dẫn (lệch 1
> từ dòng 21, lệch 2 từ dòng 43). Chuyện bình thường khi tự thêm code.

### Bước 4 — Khởi động lại server

```bash
# đứng tại thư mục yotea-be
npm start
```

`nodemon` thấy file thay đổi sẽ tự restart. Terminal phải in `App is running on port: 8080`
và `Connected to MongoDB`.

---

## 7. ✅ Kiểm chứng kết quả

### 7.1. Gọi API lần đầu

Postman → request mới → Method **GET** → URL `http://localhost:8080/api/toppings` → không cần
Header, không cần Body → **Send**. Kết quả phải là `[]` với status **200 OK**.

> 🎉 **Mảng rỗng `[]` là ĐÚNG, không phải lỗi!** Đây là chỗ 100% người mới hoảng hốt. `[]`
> nghĩa là request đã đi trọn cả 7 chặng — router khớp, controller chạy, model truy vấn
> MongoDB thành công — và MongoDB **thành thật trả lời rằng collection `toppings` hiện chưa có
> bản ghi nào**. Đường dây đã thông. Nếu hỏng ở chặng nào đó bạn sẽ nhận
> `Cannot GET /api/toppings` (chặng 3-4) hoặc một object `{ "message": ... }` (chặng 6-7),
> chứ không phải `[]`.

Đồng thời terminal backend (morgan) in một dòng:

```
GET /api/toppings 200 2 - 12.345 ms
```

Đọc là: method `GET`, path `/api/toppings`, status `200`, body dài `2` byte — đúng bằng hai ký
tự `[` và `]`.

### 7.2. Thêm dữ liệu tạm bằng MongoDB Compass

Ta chưa viết API `create` (để dành [Bài 07](07-crud-category.md)), nên tạm nhét dữ liệu thẳng
vào database:

1. Mở **MongoDB Compass**, kết nối `mongodb://localhost:27017`, chọn database **`yotea`**.
2. Tìm collection **`toppings`**. **Chưa thấy?** Bình thường — MongoDB chỉ tạo collection khi
   có bản ghi đầu tiên; bấm **Create Collection**, đặt tên chính xác `toppings`.
   **Vì sao là `toppings`?** Vì model viết `model("Topping", toppingSchema)`; Mongoose tự
   chuyển `"Topping"` → chữ thường → số nhiều → `toppings`.
3. **ADD DATA → Insert Document**, chuyển sang chế độ `{ }` (JSON) và dán, rồi bấm **Insert**:

```json
{
  "name": "Trân châu đường đen",
  "price": 10000,
  "image": "https://res.cloudinary.com/demo/image/upload/tran-chau.jpg",
  "createdAt": { "$date": "2026-08-15T03:00:00.000Z" },
  "updatedAt": { "$date": "2026-08-15T03:00:00.000Z" }
}
```

4. Quay lại Postman bấm **Send** lần nữa — bây giờ phải nhận được:

```json
[
  {
    "_id": "66be1f0a9c1d4e0012a3b4c5",
    "name": "Trân châu đường đen",
    "price": 10000,
    "image": "https://res.cloudinary.com/demo/image/upload/tran-chau.jpg",
    "createdAt": "2026-08-15T03:00:00.000Z",
    "updatedAt": "2026-08-15T03:00:00.000Z"
  }
]
```

(`_id` của bạn sẽ khác — MongoDB tự sinh.)

Chúc mừng — bạn vừa hoàn thành **trọn vẹn một vòng đời request** do chính mình dựng: Postman →
`app.js` → `routes/topping.js` → `controllers/topping.js` → `models/topping.js` → MongoDB → và
ngược lại.

> 💡 Thêm 2-3 topping nữa (thạch dừa, pudding trứng, kem cheese) để mấy bài sau có dữ liệu mà
> lọc, mà sắp xếp cho vui mắt.

---

## 8. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Cannot GET /api/toppings` (HTML, 404) | Quên `app.use("/api", toppingRouter)`, hoặc gõ `/topping` thiếu `s` | Kiểm tra Bước 3 và path trong `routes/topping.js` |
| `Cannot GET /toppings` | Gọi thiếu prefix `/api` | URL đúng: `http://localhost:8080/api/toppings` |
| `list is not a function` | Viết `import list from ...` thay vì `import { list } from ...` | Controller dùng `export const` → phải có **ngoặc nhọn** |
| `Cannot find module '../models/topping'` | Sai đường dẫn tương đối hoặc tên file hoa/thường lệch | Từ `controllers/` phải là `../models/topping` |
| `Topping.find is not a function` | Model quên `export default model(...)`, hoặc import có ngoặc nhọn | Model dùng `export default` → import **không** ngoặc nhọn |
| Trả `[]` mãi dù đã thêm dữ liệu | Chèn nhầm database (`test`) hoặc nhầm tên collection (`topping`) | Đối chiếu database `yotea` và collection `toppings` |
| `MongooseServerSelectionError: ECONNREFUSED ::1:27017` | MongoDB chưa chạy | PowerShell quyền admin: `net start MongoDB` |
| `Error: listen EADDRINUSE :::8080` | Tiến trình cũ còn giữ cổng 8080 | `npx kill-port 8080` rồi `npm start` lại |
| `Unexpected token 'export'` | Chạy `node src/app.js` thay vì `npm start` | Backend bắt buộc chạy qua `babel-node` |
| Sửa file mà API không đổi | `nodemon` chết lặng từ lần crash trước | `Ctrl+C` rồi `npm start` lại |

---

## 9. 📝 Bài tập

**Bài 1.** Cho URL sau, viết ra chính xác `req.params`, `req.query`, `req.body`, và cho biết
nó khớp dòng nào trong file route nào:

```
PUT http://localhost:8080/api/category/6250aaa11122233344455566/6250bbb11122233344455566?_expand=products
Body: { "name": "Trà sữa đặc biệt", "image": "https://.../tra-sua.jpg" }
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Khớp `yotea-be/src/routes/category.js:11` — `router.put("/category/:id/:userId", requireSignin, isAuth, isAdmin, update);`

```js
req.params  // { id: "6250aaa11122233344455566", userId: "6250bbb11122233344455566" }
req.query   // { _expand: "products" }
req.body    // { name: "Trà sữa đặc biệt", image: "https://.../tra-sua.jpg" }
```

Hai điểm phụ: thứ tự tên trong `req.params` do **route** quyết định chứ không do URL — đảo hai
id là bạn sửa nhầm bản ghi khác. Và `_expand=products` ở đây **vô tác dụng**, vì hàm `update`
(`yotea-be/src/controllers/category.js:120-137`) không hề đọc `req.query`; chỉ `read` và `list`
mới dùng `_expand`.

</details>

**Bài 2.** (Khó) Bạn cần thêm cho Topping endpoint `GET /api/toppings/pho-bien` (topping bán
chạy) **và** `GET /api/toppings/:slug` (một topping theo slug). Viết `routes/topping.js` sao
cho **cả hai** đều chạy, và giải thích vì sao thứ tự đó là bắt buộc.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Code **bạn tự viết** (dự án không có):

```js
// yotea-be/src/routes/topping.js
import { Router } from "express";
import { list, popular, read } from "../controllers/topping";

const router = Router();

router.get("/toppings/pho-bien", popular);  // TĨNH — phải đứng TRƯỚC
router.get("/toppings/:slug", read);        // ĐỘNG — hứng phần còn lại
router.get("/toppings", list);

export default router;
```

**Vì sao bắt buộc:** Express dừng ở route **đầu tiên khớp**. Đặt `"/toppings/:slug"` lên trước
thì khi gọi `/api/toppings/pho-bien`, `:slug` nhận giá trị `"pho-bien"` và chạy `read` → tìm
topping có `slug = "pho-bien"` → không có → trả `200 null`. Route `popular` **không bao giờ**
được gọi, và tệ nhất là **không có thông báo lỗi nào** để lần ra.

Đây chính xác là cái bẫy đang rình `yotea-be/src/routes/product.js:9` và `:12` — chúng chỉ
thoát nạn nhờ khác method (GET vs PATCH).

**Câu hỏi thưởng:** đặt `router.get("/toppings", list)` ở cuối có sao không? Không sao —
`/toppings` có **một** đoạn, `/toppings/:slug` cần **hai**, chúng không thể khớp lẫn nhau.

</details>

**Bài 3.** (Thám tử) Frontend gửi `POST /api/orderDetail` với body chứa `sizeId`, `sizePrice`,
`toppingId`, `toppingPrice` — xem `yotea-fe/src/pages/user/cart/CheckoutPage.js:76-87`:

```jsx
        await addOrderDetail({
          orderId,
          productId,
          productPrice,
          sizeId,
          sizePrice,
          quantity,
          ice,
          sugar,
          toppingId,
          toppingPrice,
        });
```

Còn Schema phía backend (`yotea-be/src/models/orderDetail.js:3-31`) chỉ khai báo 6 trường:
`orderId`, `productId`, `productPrice`, `quantity`, `ice`, `sugar`. Hỏi: sau khi lưu xong,
trong MongoDB có mấy trong bốn trường kia?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Đáp án: KHÔNG có trường nào — cả bốn đều bị vứt bỏ.**

Mongoose mặc định chạy ở chế độ **`strict: true`**: mọi trường **không** có trong Schema sẽ bị
**âm thầm loại bỏ** khi lưu. Không cảnh báo, không lỗi, không log. API vẫn trả `200` cùng một
document trông rất "thành công".

**Hệ quả nghiệp vụ:** khách chọn size L và thêm trân châu, trả thêm tiền — nhưng đơn hàng lưu
xuống database **mất sạch** thông tin đó. Nhân viên pha chế không biết làm size nào, thêm
topping gì. Đây là **bug thật, đang sống** trong dự án.

**Vì sao bài này nhắc:** đó chính là lý do ta chọn xây chức năng **Topping** làm mạch thực hành
xuyên suốt. Đến [Bài 10 — Quan hệ dữ liệu & populate](10-quan-he-va-populate.md), bạn sẽ tự tay
nối `Topping` vào `OrderDetail` bằng một trường `toppingId` có `ref: "Topping"` — vá đúng lỗ
hổng này.

**Bẫy phụ trong cùng đoạn code:** vòng lặp bao ngoài là `cart.forEach(async ...)`
(`CheckoutPage.js:64-89`) — `forEach` **không biết chờ** `await`, nên `dispatch(finishOrder())`
ngay dưới chạy trước khi các `addOrderDetail` kịp xong. Mổ xẻ ở [Bài 28](28-thanh-toan.md).

</details>

---

## 📌 Tóm tắt

- Một request đi qua **7 chặng**: `app.listen` → middleware toàn cục → mount `/api` → router →
  middleware của route → controller → model → MongoDB → `res.json` → client.
- **Ba lớp, ba trách nhiệm:** `routes/` biết URL, `controllers/` biết nghiệp vụ, `models/` biết
  hình dạng dữ liệu.
- `Router()` là một **mini-app** rời; nó chỉ sống khi được `app.use("/api", router)` cắm vào.
- **URL thật = prefix ở `app.use` + path khai báo trong router** — đó là lý do `"/category"` ở
  `routes/category.js:10` thành `GET /api/category`.
- `req.params` (đoạn `:` trên đường dẫn) · `req.query` (sau dấu `?`) · `req.body` (thân request,
  cần `express.json()`). Hai cái đầu **luôn là chuỗi**.
- `res.json(x)` = 200; `res.status(400).json({...})` = lỗi. Mỗi request chỉ trả lời **một lần**.
- Express khớp route theo **thứ tự khai báo, dừng ở cái đầu tiên khớp** → **route tĩnh phải
  đứng trước route động**, nếu không API của bạn "biến mất" mà không báo lỗi.
- Thêm một resource mới = đụng **5 chỗ**: model, controller, route, import trong `app.js`, mount
  trong `app.js`.
- Mongoose `strict: true` **âm thầm vứt bỏ** mọi trường không có trong Schema — đúng thứ đang
  xảy ra với `toppingId` / `sizeId` của dự án.

**Từ khoá tra cứu thêm:** `express router`, `express middleware order`, `req.params vs req.query vs req.body`, `express route matching order`, `separation of concerns MVC`, `mongoose strict mode`

➡️ **Bài tiếp theo:** [07 — CRUD trọn vẹn với Category + test bằng Postman](07-crud-category.md) — bạn mới có mỗi thao tác "đọc danh sách"; bài sau ta lắp nốt 4 thao tác còn lại cho Topping và test cả 5 bằng Postman.
