# Bài 12 — Phân quyền: `requireSignin`, `isAuth`, `isAdmin`, `router.param`

> **Phần 1 · Backend với Express + MongoDB** — Thời lượng ước tính: **~70 phút**
> ⬅️ Bài trước: [11 — Mã hoá mật khẩu và xác thực bằng JWT](11-mat-khau-va-jwt.md) · Bài sau: [13 — Viết tài liệu API tự động với Swagger](13-swagger-tai-lieu-api.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Phân biệt được **Authentication** (bạn là ai) và **Authorization** (bạn được làm gì).
- Đọc hiểu từng dòng của `checkAuth.js` — ba middleware `requireSignin`, `isAuth`, `isAdmin`.
- Hiểu cơ chế `router.param("userId", userById)` và vì sao nó chạy **trước** mọi middleware khác.
- Đọc được bảng quyền của **toàn bộ 70 endpoint** dự án và chỉ ra endpoint nào đang hở.
- Tự tay **khoá các route ghi của Topping** và kiểm chứng 4 tình huống bằng Postman.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 11](11-mat-khau-va-jwt.md): biết token được ký ở đâu, chứa gì, gọi `POST /api/signin` thế nào.
- Chức năng **Topping** đã đủ 5 thao tác CRUD (Bài 07), có `slug` (Bài 08), có bộ lọc query (Bài 09), đã nối quan hệ với `OrderDetail` (Bài 10).
- Postman đang chạy, MongoDB đang bật, backend `npm start` chạy được.
- Trong database có **ít nhất 2 tài khoản**: một cái `role = 1` (admin), một cái `role = 0` (user thường).

> Ở [bài trước](11-mat-khau-va-jwt.md) bạn đã lấy được **tấm vé** (token JWT). Bài này ta học cách
> **kiểm vé**: ai được vào cửa nào — và vì sao Yotea kiểm vé theo một cách rất lạ là nhét id người
> dùng lên URL.

---

## 1. Authentication ≠ Authorization

Hai từ nhìn na ná nhau nhưng là hai việc khác hẳn. Tưởng tượng bạn đi làm ở toà nhà văn phòng:

| Ví dụ đời thường | Thuật ngữ | Trả lời câu hỏi |
|---|---|---|
| Bảo vệ soi thẻ, đúng mặt đúng tên → cho vào toà nhà | **Authentication** (xác thực) | *"Bạn **là ai**?"* |
| Thẻ quẹt được tầng 3, quẹt tầng 10 (phòng giám đốc) thì đèn đỏ | **Authorization** (phân quyền) | *"Bạn **được làm gì**?"* |

Mấu chốt: **xác thực xong chưa có nghĩa là được phép**. Bạn chứng minh mình là "Nguyễn Văn A" —
đúng, nhưng Nguyễn Văn A vẫn không được xoá sản phẩm của cửa hàng. Trong Yotea, ba middleware chia
nhau đúng ba việc:

| Middleware | Loại | Nhiệm vụ |
|---|---|---|
| `requireSignin` | **Authentication** | Có token không? Hợp lệ không? Còn hạn không? |
| `isAuth` | Nửa nọ nửa kia | Người trong token có đúng là người ghi trên URL không? |
| `isAdmin` | **Authorization** | Người đó có `role = 1` không? |

> 📖 **Thuật ngữ:** *middleware* — hàm nhận `(req, res, next)`, chắn giữa request và controller.
> Nó có đúng 2 lựa chọn: gọi `next()` để **cho đi tiếp**, hoặc `res.status(...).json(...)` để
> **chặn và trả lời luôn**. Không làm cả hai → request **treo vĩnh viễn**.

---

## 2. Soi code thật: `middlewares/checkAuth.js`

Cả bộ máy phân quyền gói gọn trong **29 dòng**. Đây là nguyên văn toàn bộ file.

`yotea-be/src/middlewares/checkAuth.js:1-29`

```js
import expressJWT from "express-jwt";

export const requireSignin = expressJWT({
  algorithms: ["HS256"],
  secret: "TuongVy",
  requestProperty: "auth",
});

export const isAuth = (req, res, next) => {
  const status = req.profile._id == req.auth._id;

  if (!status) {
    res.status(400).json({
      message: "Bạn không có quyền truy cập",
    });
  } else {
    next();
  }
};

export const isAdmin = (req, res, next) => {
  if (!req.profile.role) {
    res.status(401).json({
      message: "Bạn không phải là Admin",
    });
  } else {
    next();
  }
};
```

### 2.1. `requireSignin` — người soát vé

Khác hai cái dưới, `requireSignin` **không phải hàm bạn tự viết**. Bạn gọi `expressJWT({...})` một
lần, thư viện **sản xuất ra** một middleware, rồi bạn đem gắn vào route.

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import expressJWT from "express-jwt"` | Import mặc định (không ngoặc nhọn) — **cú pháp cũ của express-jwt v6** |
| 4 | `algorithms: ["HS256"]` | Chỉ chấp nhận token ký bằng HMAC-SHA256. Bắt buộc khai, thiếu là thư viện báo lỗi |
| 5 | `secret: "TuongVy"` | Khoá bí mật để **verify** chữ ký, phải trùng khoá lúc ký ở `controllers/auth.js:52` |
| 6 | `requestProperty: "auth"` | Giải mã xong thì nhét payload vào **`req.auth`** |

Khi có request tới, nó đọc header `Authorization` (đúng dạng `Bearer <token>` — chữ `Bearer`, **một
dấu cách**, rồi token), kiểm tra **chữ ký** và **hạn dùng**. Sai/thiếu/hết hạn → ném
`UnauthorizedError` (401), dừng ngay. Đúng → gán payload vào `req.auth` rồi `next()`.

Payload đó chính là thứ đã ký lúc đăng nhập — `yotea-be/src/controllers/auth.js:50-54`:

```js
      const token = jwt.sign(
        { _id: user._id, email: user.email },
        "TuongVy",
        { expiresIn: "3h" }
      );
```

Nên sau `requireSignin`, `req.auth` là `{ "_id": "6249a1...", "email": "admin@gmail.com", "iat": ..., "exp": ... }`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn (1):** payload **không chứa `role`**. Quyết định này kéo theo
> *toàn bộ* sự rườm rà của bài hôm nay: token không nói được bạn có phải admin không, nên backend
> buộc phải tra lại database — và để tra được, nó bắt client gửi `:userId` lên URL. Nếu payload có
> thêm `role`, dự án đã không cần `:userId`, không cần `isAuth`, cũng chẳng cần `router.param`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn (2):** `secret: "TuongVy"` viết cứng trong code và đẩy lên Git.
> Ai đọc được repo là **tự ký được token của bất kỳ ai**, kể cả admin. Cách đúng: để trong `.env`
> — sẽ sửa ở [Bài 34](34-refactor-du-an.md).

> 💡 **Mẹo phiên bản:** `yotea-be/package.json:19` khai `"express-jwt": "^6.1.1"`. Từ **v7** thư
> viện đổi sang export có tên: `import { expressjwt } from "express-jwt"` rồi `expressjwt({...})`
> (chữ `j` thường). Gặp `expressJWT is not a function` ở dự án khác thì đó là lý do.

### 2.2. `isAuth` — "id trên URL có đúng là bạn không?"

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 10 | `req.profile._id == req.auth._id` | So id **tra từ DB** (theo `:userId` trên URL) với id **trong token** |
| 12 | `if (!status)` | Không khớp → có người đang mượn token để thao tác lên tài khoản khác |
| 13-15 | `res.status(400).json({...})` | Chặn, trả 400 |
| 16-18 | `else { next(); }` | Khớp → đi tiếp |

Hai biến quan trọng nhất bài này, nhớ kỹ kẻo lẫn:

| Biến | Ai gán | Chứa gì | Kiểu của `_id` |
|---|---|---|---|
| `req.auth` | `requireSignin` (express-jwt) | Payload token → người **thật sự** đang đăng nhập | **String** |
| `req.profile` | `userById` (qua `router.param`) | User tra từ DB theo `:userId` **trên URL** | **ObjectId** |

**Vì sao `==` chứ không phải `===`?** Ở đây là **cố ý**:

```js
req.profile._id === req.auth._id   // false — ObjectId vs String, LUÔN LUÔN false
req.profile._id == req.auth._id    // true  — == tự gọi toString() rồi so sánh
```

Nếu ai đó "dọn code" đổi `==` thành `===`, **mọi route cần đăng nhập sẽ hỏng ngay**.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** nên viết tường minh — `req.profile._id.equals(req.auth._id)`
> hoặc `String(req.profile._id) === req.auth._id`. Và mã lỗi đúng là **401**, không phải 400.

### 2.3. `isAdmin` — "bạn có phải quản trị viên không?"

Trường `role` khai ở `yotea-be/src/models/user.js:54-57`:

```js
    role: {
      type: Number,
      default: 0,
    },
```

Dòng `checkAuth.js:22` — `if (!req.profile.role)`: `role = 0` → `!0` là `true` → **chặn 401**;
`role = 1` → `!1` là `false` → `next()` cho qua controller.

Chú ý `isAdmin` đọc `req.profile.role` — **dữ liệu tươi vừa lấy từ DB**, không phải từ token. Đây
là **ưu điểm duy nhất** của thiết kế `:userId`: hạ quyền một admin có hiệu lực ngay, không phải chờ
token hết hạn.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `if (!req.profile.role)` chỉ kiểm tra *truthy* → `role = 2`,
> `role = 99`, `role = -1` **đều được coi là admin**. Chuẩn hơn: `if (req.profile.role !== 1)`. Và
> mã đúng cho "đã đăng nhập nhưng không đủ quyền" là **403 Forbidden**.

---

## 3. `router.param` — mảnh ghép còn thiếu

Đọc tới đây bạn phải thấy một lỗ hổng logic: `isAuth` và `isAdmin` đều đọc `req.profile`, nhưng
**không middleware nào gán nó cả**. Vậy `req.profile` từ đâu ra?

`yotea-be/src/controllers/user.js:288-306`

```js
export const userById = async (req, res, next, id) => {
    try {
        const user = await User.findById(id).exec();

        if (!user) {
            res.status(400).json({
                message: "Không tìm thấy User"
            });
        } else {
            req.profile = user;
            next();
        }
    } catch (error) {
        res.status(400).json({
            message: "Không tìm thấy User",
            error
        });
    }
}
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 288 | `async (req, res, next, id)` | **BỐN** tham số — chữ ký đặc biệt của *param callback*. `id` là giá trị `:userId` đọc từ URL |
| 290 | `User.findById(id).exec()` | Một truy vấn DB, chạy trên **mọi** request có `:userId` |
| 292-296 | `if (!user)` | Id trên URL không tồn tại → 400 và **dừng** (không gọi `next()`) |
| 297 | `req.profile = user` | **Đây chính là chỗ `req.profile` ra đời** |
| 298 | `next()` | Cho đi tiếp — `requireSignin` mới bắt đầu chạy |
| 300-305 | `catch` | Id sai định dạng ObjectId (ví dụ `abc`) → Mongoose ném `CastError` → 400 |

Cuối mỗi file route có đúng một dòng đăng ký — `yotea-be/src/routes/product.js:15`:

```js
router.param("userId", userById);
```

Đọc là: *"Trong router này, hễ URL nào có tham số tên `userId`, hãy chạy `userById` trước đã."*
Bốn điều cần nhớ:

1. **Chạy trước tất cả** — trước `requireSignin`, `isAuth`, `isAdmin`, và tất nhiên trước controller.
2. **Chỉ chạy khi URL thật sự có param đó.** `GET /api/products` không có `:userId` → `req.profile` là `undefined`.
3. **Chỉ chạy một lần** cho mỗi request.
4. **Phạm vi là router, không phải toàn app** → mọi file route phải lặp lại dòng này:
   `routes/category.js:14`, `routes/product.js:15`, `routes/users.js:15`, `routes/comment.js:14`,
   `routes/rating.js:14`, `routes/order.js:14`, `routes/contact.js:14`, `routes/favoritesProduct.js:14`…

> 💡 **Câu hỏi 9/10 người học đều hỏi:** dòng `router.param(...)` nằm **sau** tất cả
> `router.get/post/...` — sao vẫn chạy trước? Vì lúc nạp file, Express không thực thi request nào,
> nó chỉ **ghi vào hai cuốn sổ riêng**: sổ *route* và sổ *param*. Khi request tới, Express tra sổ
> param **trước**, rồi mới tới sổ route. Vị trí dòng code hoàn toàn không quan trọng.

**Sơ đồ thứ tự chạy** với `PUT /api/products/650abc.../6249a1f2...` (sửa sản phẩm, cần admin):

```
Client ─ PUT /api/products/:id/:userId + header "Authorization: Bearer ..." ─►
  ▼ express.json() → cors() → morgan()                          (app.js:25-27)
  ▼ ① router.param("userId", userById)   ◄── CHẠY TRƯỚC TIÊN
  │    findById(:userId) → req.profile = <User>
  │    không thấy → 400 "Không tìm thấy User"              ✖ DỪNG
  ▼ ② requireSignin  (express-jwt)  verify header → req.auth = payload
  │    thiếu/sai/hết hạn → 401 UnauthorizedError           ✖ DỪNG
  ▼ ③ isAuth      req.profile._id == req.auth._id ?
  │    lệch → 400 "Bạn không có quyền truy cập"            ✖ DỪNG
  ▼ ④ isAdmin     req.profile.role truthy ?
  │    role = 0 → 401 "Bạn không phải là Admin"            ✖ DỪNG
  ▼ ⑤ update (controller) → findOneAndUpdate → res.json(...)   ✔ 200
```

Mỗi chốt chỉ có 2 lối ra: **`next()` đi tiếp**, hoặc **`res.json()` trả lời và dừng hẳn**.

---

## 4. Soi code thật: `routes/product.js`

File route đầy đủ nhất dự án — có cả route công khai, route admin, và một route "lạ".

`yotea-be/src/routes/product.js:1-17`

```js
import { Router } from "express";
import { clientUpdate, create, list, read, remove, update } from "../controllers/product";
import { userById } from "../controllers/user";
import { isAdmin, isAuth, requireSignin } from "../middlewares/checkAuth";

const router = Router();

router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/products/:slug", read);
router.get("/products", list);
router.put("/products/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.patch("/products/userUpdate/:id", clientUpdate);
router.delete("/products/:id/:userId", requireSignin, isAuth, isAdmin, remove);

router.param("userId", userById);

export default router;
```

**Đọc từng dòng:**

| Dòng | Route | Ai gọi được? |
|---|---|---|
| 8 | `POST /products/:userId` | **Chỉ admin.** `:userId` là id của **admin đang đăng nhập**, không phải id sản phẩm sắp tạo |
| 9 | `GET /products/:slug` | **Công khai** — khách xem chi tiết sản phẩm |
| 10 | `GET /products` | **Công khai** — trang thực đơn |
| 11 | `PUT /products/:id/:userId` | **Chỉ admin.** `:id` = sản phẩm bị sửa, `:userId` = admin đang đăng nhập |
| 12 | `PATCH /products/userUpdate/:id` | **Công khai!** Không một middleware nào |
| 13 | `DELETE /products/:id/:userId` | **Chỉ admin** |
| 15 | `router.param("userId", userById)` | Đăng ký param callback cho cả router |

Quy luật chung của gần như mọi file route: **GET → không middleware**; **POST/PUT/DELETE →
`requireSignin, isAuth, isAdmin` và URL kết thúc bằng `/:userId`**.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dòng 12 không có middleware nào. Nó tồn tại để frontend tăng
> `view` / `favorites` mà không cần đăng nhập. Hệ quả: ai cũng gõ được một lệnh `curl` đặt
> `favorites` của một sản phẩm thành `999999`, chiếm luôn ô "Sản phẩm được yêu thích nhất" ở trang
> chủ. Cách đúng: backend tự `$inc` thêm 1, không nhận số từ client.

---

## 5. Bảng quyền của TOÀN BỘ endpoint

Bảng dưới dựng từ bản đồ source `be-03` / `be-04`, rồi **kiểm chứng lại bằng grep toàn bộ 14 file
trong `yotea-be/src/routes/`**. Ký hiệu: 🌐 công khai · 🔑 cần đăng nhập (`requireSignin, isAuth`)
· 👑 cần admin (thêm `isAdmin`). Dấu ⚠️ = chỗ đáng lo.

| Router | 🌐 Công khai | 🔑 Cần đăng nhập | 👑 Cần admin |
|---|---|---|---|
| `auth.js` | `POST /signin` · `POST /signup` · `POST /checkPassword` ⚠️ | — | — |
| `category.js` | `GET /category/:slug` · `GET /category` | — | `POST /category/:userId` · `PUT` · `DELETE /category/:id/:userId` |
| `product.js` | `GET /products/:slug` · `GET /products` · `PATCH /products/userUpdate/:id` ⚠️ | — | `POST /products/:userId` · `PUT` · `DELETE /products/:id/:userId` |
| `slider.js` | `GET /slider/:id` · `GET /slider` | — | `POST /slider/:userId` · `PUT` · `DELETE /slider/:id/:userId` |
| `store.js` | `GET /store/:id` · `GET /store` | — | `POST /store/:userId` · `PUT` · `DELETE /store/:id/:userId` |
| `cateNews.js` | `GET /cateNews/:slug` · `GET /cateNews` | — | `POST /cateNews/:userId` · `PUT` · `DELETE /cateNews/:id/:userId` |
| `news.js` | `GET /news/:slug` · `GET /news` | — | `POST /news/:userId` · `PUT` · `DELETE /news/:id/:userId` |
| `users.js` | `GET /users/:id` ⚠️ · `GET /users` ⚠️ | `PUT /users/updateInfo/:myId/:userId` ⚠️ | `POST /users/:userId` · `PUT` · `DELETE /users/:id/:userId` |
| `order.js` | `POST /orders` · `GET /orders/:id` ⚠️ · `GET /orders` ⚠️ · `DELETE /orders/:id` ⚠️ | `PUT /orders/:id/:userId` ⚠️ | — |
| `orderDetail.js` | Cả **5** endpoint ⚠️ (`POST` · `GET /:id` · `GET` · `PUT /:id` · `DELETE /:id`) | — | — |
| `comment.js` | `GET /comments/:id` · `GET /comments` | `POST /comments/:userId` · `PUT` · `DELETE /comments/:id/:userId` ⚠️ | — |
| `rating.js` | `GET /ratings/:id` · `GET /ratings` | `POST /ratings/:userId` · `PUT` · `DELETE /ratings/:id/:userId` ⚠️ | — |
| `favoritesProduct.js` | `GET /favoritesProduct/:id` · `GET /favoritesProduct` | `POST /favoritesProduct/:userId` · `PUT` · `DELETE /favoritesProduct/:id/:userId` ⚠️ | — |
| `contact.js` | `POST /contact` · `GET /contact/:id` ⚠️ · `GET /contact` ⚠️ | — | `PUT` · `DELETE /contact/:id/:userId` |

**Thống kê (đếm lại từ grep):** tổng **70 endpoint** — 🌐 **36 công khai (51%)**, 🔑 **11 cần đăng
nhập**, 👑 **23 cần admin**. Hơn một nửa không kiểm tra gì cả, và trong 36 cái công khai đó có
**9 thao tác GHI**.

---

## 6. ⚠️ Phân tích bảo mật: thiết kế `:userId` trên URL là sai lầm

Đây là phần quan trọng nhất bài. Đọc chậm.

### 6.1. (a) Rườm rà và (b) thừa dữ liệu

Để gọi `DELETE /api/products/:id/:userId`, frontend phải tự **móc id của mình từ localStorage rồi
ghép vào URL**. Mỗi hàm API phải nhớ làm việc đó; quên một chỗ là lỗi. Server thì phải chạy thêm
một `findById` chỉ để lấy đúng trường `role`. Id người dùng nằm chình ình trên URL nên bị ghi vào
**access log** (`morgan`, `app.js:27`), vào header `Referer`, vào lịch sử trình duyệt.

Mà tất cả đều **thừa**: token đã chứa `_id` rồi. Server chỉ cần đọc `req.auth._id` là biết chính
xác ai đang gọi — token có chữ ký, không giả được. Bắt client gửi lại `_id` lần nữa qua URL tạo ra
ảo giác rằng "id trên URL" là nguồn tin đáng tin. Nó **không** đáng tin, và `isAuth` sinh ra chỉ để
dọn dẹp cái ảo giác đó — một middleware tồn tại chỉ vì thiết kế thừa.

### 6.2. (c) Những route đang THIẾU bảo vệ (đã grep kiểm chứng)

**① Toàn bộ `/api/orderDetail` — không một middleware nào.** `yotea-be/src/routes/orderDetail.js:6-10`:

```js
router.post("/orderDetail", create);
router.get("/orderDetail/:id", read);
router.get("/orderDetail", list);
router.put("/orderDetail/:id", update);
router.delete("/orderDetail/:id", remove);
```

Đây là **router duy nhất** trong dự án thậm chí không `import` file `checkAuth.js`.

**② `order` — hở ở 2 chỗ.** `yotea-be/src/routes/order.js:11-12`:

```js
router.put("/orders/:id/:userId", requireSignin, isAuth, update);
router.delete("/orders/:id", remove);
```

Dòng 11 có `requireSignin, isAuth` nhưng **thiếu `isAdmin`** → user thường tự đổi trạng thái đơn
hàng thành "đã giao". Dòng 12 **trống hoàn toàn** → ai cũng xoá được đơn hàng bất kỳ.

**③ `comment` / `rating` / `favoritesProduct` — chốt sai chỗ.** `yotea-be/src/routes/comment.js:11-12`:

```js
router.put("/comments/:id/:userId", requireSignin, isAuth, update);
router.delete("/comments/:id/:userId", requireSignin, isAuth, remove);
```

`isAuth` chỉ chứng minh *"tôi là chính tôi"*. Nó **không** kiểm tra bình luận `:id` kia có phải của
tôi không, controller cũng chỉ lọc theo `_id` của comment → user A xoá được comment của user B chỉ
bằng cách ghép id của mình vào cuối URL. `rating.js:11-12` và `favoritesProduct.js:11-12` y hệt.

**④ `users` — leo thang đặc quyền.** `yotea-be/src/routes/users.js:12`:

```js
router.put("/users/updateInfo/:myId/:userId", requireSignin, isAuth, update);
```

`isAuth` kiểm tra `:userId` (id của tôi) — đúng. Nhưng bản ghi **bị sửa** là `:myId`, và `:myId`
**không được kiểm tra gì cả**.

### 6.3. Kẻ xấu làm được gì — ba kịch bản có thật

```bash
# Kịch bản 1 — xoá sạch đơn hàng của cửa hàng (không cần token)
curl http://localhost:8080/api/orders          # công khai, lộ luôn tên/sđt/email/địa chỉ khách
curl -X DELETE http://localhost:8080/api/orders/650abc1234567890abcdef01

# Kịch bản 2 — sửa giá trong chi tiết đơn hàng của người khác (không cần token)
curl -X PUT http://localhost:8080/api/orderDetail/650abc1234567890abcdef02 \
  -H "Content-Type: application/json" -d "{\"productPrice\": 0, \"quantity\": 100}"

# Kịch bản 3 — tự phong mình làm admin (chỉ cần một tài khoản thường)
curl http://localhost:8080/api/users           # GET /api/users công khai, trả cả hash mật khẩu
curl -X PUT http://localhost:8080/api/users/updateInfo/<id-cua-toi>/<id-cua-toi> \
  -H "Authorization: Bearer <token-user-thuong>" \
  -H "Content-Type: application/json" -d "{\"role\": 1}"
```

Kịch bản 3 lọt vì `:userId` đúng là tôi (`isAuth` gật đầu), route này **không có `isAdmin`**, và
controller `update` (`controllers/user.js:225-240`) nhận nguyên `req.body` không lọc trường nào.
Sau lệnh đó, tài khoản thường của bạn trở thành admin toàn quyền.

> 🔒 **Ghi chú bảo mật:** đây không phải lý thuyết — chạy thử trên máy mình là thấy nó hoạt động.
> Ta tổng rà soát và vá ở [Bài 33](33-ra-soat-bao-mat.md) và [Bài 34](34-refactor-du-an.md).

### 6.4. Hệ quả âm thầm: dữ liệu topping bị nuốt mất

Chuyện này dính trực tiếp tới Topping bạn đang xây. Frontend gửi lên `toppingId` và `toppingPrice`
khi đặt hàng (`yotea-fe/src/pages/user/cart/CheckoutPage.js:73` và `:85`), nhưng schema `OrderDetail`
(`yotea-be/src/models/orderDetail.js:3-31`) chỉ khai đúng 6 trường — `orderId`, `productId`,
`productPrice`, `quantity`, `ice`, `sugar` — **không có `toppingId`, không có `sizeId`**.

Mongoose ở chế độ `strict` mặc định **âm thầm vứt bỏ** những trường lạ đó: không lỗi, không cảnh
báo. Đơn hàng lưu xuống mất sạch thông tin topping, trong khi `totalPrice` lại đã cộng tiền topping
vào rồi. Và vì `POST /api/orderDetail` là public (mục 6.2 ①), không ai kiểm tra để phát hiện sai lệch.

Bạn đã vá phần schema ở [Bài 10](10-quan-he-va-populate.md). Hôm nay ta vá phần **khoá cửa**.

---

## 7. Cách làm đúng: đọc id từ token, bỏ hẳn `:userId` trên URL

Ý tưởng rất đơn giản: **thay `router.param` bằng middleware tự tra user từ `req.auth._id`.**
Đoạn dưới **là code bạn tự viết — dự án Yotea KHÔNG có file này:**

```js
// yotea-be/src/middlewares/checkAuthV2.js  ← file MỚI, bạn tự tạo
import User from "../models/user";

// Nạp user hiện tại TỪ TOKEN. Bắt buộc chạy SAU requireSignin.
export const loadCurrentUser = async (req, res, next) => {
  try {
    const user = await User.findById(req.auth._id).select("-password -__v").exec();
    if (!user) return res.status(401).json({ message: "Tài khoản không tồn tại" });
    if (!user.active) return res.status(403).json({ message: "Tài khoản đã bị khoá" });

    req.currentUser = user;
    next();
  } catch (error) {
    return res.status(401).json({ message: "Không xác thực được người dùng" });
  }
};

// Chỉ cho admin đi tiếp. Bắt buộc chạy SAU loadCurrentUser.
export const isAdminByToken = (req, res, next) => {
  if (req.currentUser.role !== 1) {
    return res.status(403).json({ message: "Bạn không phải là Admin" });
  }
  next();
};
```

Route trở nên gọn và đúng REST hơn hẳn:

```js
router.delete("/products/:id/:userId", requireSignin, isAuth, isAdmin, remove);              // cách cũ
router.delete("/products/:id", requireSignin, loadCurrentUser, isAdminByToken, remove);      // cách đúng
```

| | Cách của dự án | Cách đúng |
|---|---|---|
| Nguồn danh tính | URL (client tự khai) **+** token | **Chỉ token** |
| Client phải làm gì | Móc `_id` từ localStorage, ghép vào URL | Chỉ gắn header `Authorization` |
| Cần `isAuth` không | **Có** — để đối chiếu URL với token | **Không** — chẳng còn gì để đối chiếu |
| Rò rỉ id ra log / Referer | Có | Không |
| Mã lỗi khi thiếu quyền | 400 / 401 | 403 (đúng chuẩn) |

> 💡 Còn một cách gọn hơn nữa: **nhét `role` vào payload JWT** lúc ký token, khi đó chỉ cần đọc
> `req.auth.role`, khỏi tra DB. Đánh đổi: hạ quyền admin chỉ có hiệu lực sau khi token cũ hết hạn
> (tối đa 3 tiếng) — với hệ thống nhỏ như Yotea thì hoàn toàn chấp nhận được.

---

## 8. 🛠️ Tự tay làm — khoá các route ghi của Topping

> Mục tiêu: sau phần này, `GET /api/toppings` vẫn ai cũng gọi được, nhưng `POST` / `PUT` / `DELETE`
> thì **chỉ admin** mới gọi nổi.

Ở [Bài 10](10-quan-he-va-populate.md) bạn đã nối Topping vào `OrderDetail` bằng `ref` + `populate`.
Bài này ta làm tiếp việc còn thiếu: **đặt khoá cho cánh cửa**.

### Bước 1 — Nhìn lại file route hiện tại

Mở `yotea-be/src/routes/topping.js` (file **bạn tự viết** từ Bài 06, hoàn thiện ở Bài 07). Năm dòng
route của bạn đang **mở toang**, đại khái thế này:

```js
// yotea-be/src/routes/topping.js  ← file của BẠN, TRƯỚC khi sửa
router.post("/toppings", create);
router.get("/toppings/:slug", read);
router.get("/toppings", list);
router.put("/toppings/:id", update);
router.delete("/toppings/:id", remove);
```

### Bước 2 — Sửa lại toàn bộ file

Thêm 2 dòng import, gắn middleware cho 3 route ghi, thêm `router.param`. Hai dòng `get` **giữ
nguyên** — thực đơn phải công khai thì khách mới xem được topping.

```js
// yotea-be/src/routes/topping.js  ← file của BẠN, SAU khi sửa (toàn bộ file)
import { Router } from "express";
import { create, list, read, remove, update } from "../controllers/topping";
import { userById } from "../controllers/user";
import { isAdmin, isAuth, requireSignin } from "../middlewares/checkAuth";

const router = Router();

router.post("/toppings/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/toppings/:slug", read);
router.get("/toppings", list);
router.put("/toppings/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.delete("/toppings/:id/:userId", requireSignin, isAuth, isAdmin, remove);

router.param("userId", userById);

export default router;
```

**Đọc từng dòng bạn vừa sửa:**

| Dòng | Thay đổi | Vì sao |
|---|---|---|
| 3-4 | Import `userById` + 3 middleware | Cả hai đều là export **có tên** → phải dùng ngoặc nhọn |
| 8 | Thêm `/:userId` + 3 middleware | Chỉ admin được thêm topping |
| 9-10 | Không đổi | Route đọc phải công khai |
| 11 | Thêm `/:userId` + 3 middleware | `:id` = topping bị sửa, `:userId` = admin đang đăng nhập |
| 12 | Thêm `/:userId` + 3 middleware | Chỉ admin được xoá |
| 14 | `router.param("userId", userById)` | **Bắt buộc** — thiếu là `req.profile` undefined, `isAuth` nổ |

> ⚠️ Đừng quên dòng 14. Đây là lỗi số một của người mới: gắn `isAuth` mà quên `router.param` →
> server trả **500** kèm `TypeError: Cannot read properties of undefined (reading '_id')`.

### Bước 3 — Soát lại controller rồi khởi động lại

Mở `yotea-be/src/controllers/topping.js` (file của bạn), xác nhận `update` và `remove` lọc theo
**`req.params.id`** chứ không phải `req.params.userId`:

```js
// yotea-be/src/controllers/topping.js  ← đoạn bạn cần soát lại
const filter = { _id: req.params.id };        // ✅ đúng: id của TOPPING
// const filter = { _id: req.params.userId }; // ❌ sai: đây là id của admin
```

Nhầm hai cái này là bug rất khó tìm, vì hệ thống vẫn trả 200 và... không xoá gì cả. Xong thì:

```bash
# đứng tại thư mục yotea-be
npm start
```

Terminal phải in `App is running on port: 8080` và `Connected to MongoDB`.

---

## 9. ✅ Kiểm chứng kết quả

### 9.1. Lấy hai token

Gọi `POST http://localhost:8080/api/signin` hai lần (một admin, một user thường). Tab **Body** →
**raw** → kiểu **JSON**:

```json
{ "email": "admin@gmail.com", "password": "123456" }
```

Response trả về `{ "token": "eyJhbGci...", "user": { "_id": "6249a1...", "role": 1 } }`. Chép lại
**cả `token` lẫn `user._id`** của hai tài khoản.

### 9.2. Cách gắn token vào Postman

Thao tác này bạn sẽ dùng suốt phần còn lại của giáo trình:

1. Mở request cần gọi.
2. Chọn tab **Authorization** (cạnh tab **Params**, **Headers**, **Body**).
3. Ô **Auth Type**: chọn **Bearer Token**.
4. Ô **Token** bên phải: dán token vào — **chỉ dán chuỗi token**, KHÔNG gõ thêm chữ `Bearer`.
5. Muốn kiểm tra, sang tab **Headers** tick ô *hidden* — sẽ thấy `Authorization: Bearer eyJhbGci...`.

> 💡 Cách thủ công tương đương: tab **Headers**, thêm key `Authorization`, value
> `Bearer eyJhbGci...` (đúng **một** dấu cách sau chữ `Bearer`).

### 9.3. Bốn tình huống bắt buộc phải kiểm

Dùng request `POST http://localhost:8080/api/toppings/<id-người-gọi>` với Body JSON
`{ "name": "Trân châu đường đen", "price": 10000 }`:

| # | Tình huống | Kết quả bắt buộc | Ai chặn |
|---|---|---|---|
| 1 | **Không gắn token** (Auth Type = No Auth) | **401** — body là trang HTML kèm `UnauthorizedError: No authorization token was found` | `requireSignin` |
| 2 | **Token user thường**, `:userId` = id user đó | **401** `{ "message": "Bạn không phải là Admin" }` | `isAdmin` |
| 3 | **Token admin**, `:userId` = id admin | **200** kèm topping vừa tạo (có `_id`, `slug`, `createdAt`) | không ai — qua hết |
| 4 | **Token admin**, `:userId` = id người khác | **400** `{ "message": "Bạn không có quyền truy cập" }` | `isAuth` |

Đọc kỹ tình huống 2: `userById` chạy được (tìm thấy user) → `requireSignin` gật (token hợp lệ) →
`isAuth` gật (id URL khớp id token) → **`isAdmin` chặn** vì `role = 0`. Đúng thứ tự sơ đồ mục 3.

Cuối cùng đừng quên: `GET http://localhost:8080/api/toppings` **không cần token** vẫn phải trả về
mảng topping. Nếu nó cũng đòi token, bạn đã lỡ tay gắn middleware vào dòng `get`.

---

## 10. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `TypeError: Cannot read properties of undefined (reading '_id')` | Route có `isAuth` nhưng thiếu `router.param("userId", userById)` hoặc URL không có `:userId` | Thêm dòng `router.param(...)`, chắc chắn path kết thúc bằng `/:userId` |
| `UnauthorizedError: No authorization token was found` (401, trả HTML) | Quên gắn token, hoặc thiếu chữ `Bearer` | Tab **Authorization** → **Bearer Token** → dán token (không kèm chữ `Bearer`) |
| `UnauthorizedError: jwt expired` | Token quá 3 tiếng (`expiresIn: "3h"`) | Gọi lại `POST /api/signin` lấy token mới |
| `UnauthorizedError: invalid signature` | Token ký bằng secret khác | Secret ở `checkAuth.js:5` phải trùng secret ở `controllers/auth.js:52` |
| `400 { "message": "Bạn không có quyền truy cập" }` | `:userId` trên URL khác `_id` trong token | Dán đúng `user._id` mà `/api/signin` vừa trả về |
| `401 { "message": "Bạn không phải là Admin" }` | Tài khoản có `role = 0` | Sửa `role` thành `1` trong MongoDB Compass rồi **đăng nhập lại** |
| `400 { "message": "Không tìm thấy User" }` | `:userId` sai hoặc không đúng định dạng ObjectId 24 ký tự | Kiểm tra lại chuỗi id đã dán |
| `expressJWT is not a function` | Cài nhầm express-jwt v7+ | Dùng `import { expressjwt } from "express-jwt"`, hoặc cài lại `express-jwt@6.1.1` |
| Request **treo**, Postman quay mãi | Middleware tự viết không gọi `next()` cũng không `res` | Mỗi nhánh `if/else` phải kết thúc bằng `next()` hoặc `res...json()` |

> ⚠️ Vì sao lỗi token trả về **HTML** chứ không phải JSON? Vì `app.js` **không có** error-handling
> middleware `(err, req, res, next)` ở cuối file, lỗi do `express-jwt` ném ra rơi vào trình xử lý
> mặc định của Express. Hệ quả thật: frontend đọc `error.response.data.message` nhận `undefined`
> → toast báo lỗi hiện ra trống trơn. Ta vá ở [Bài 34](34-refactor-du-an.md).

---

## 11. 📝 Bài tập

**Bài 1.** Trong `routes/topping.js` của bạn, dòng `router.param("userId", userById)` nằm ở **cuối
file**. Giải thích vì sao nó vẫn chạy **trước** `requireSignin`. Và nếu xoá dòng đó đi thì điều gì
xảy ra khi gọi `DELETE /api/toppings/<id>/<userId>` với token admin hợp lệ?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Vì sao vẫn chạy trước:** khi Node nạp file route, nó không thực thi request nào — chỉ **đăng ký**
vào hai cuốn sổ riêng biệt của router: sổ *route* (do `router.get/post/...` ghi) và sổ *param
callback* (do `router.param` ghi). Lúc có request thật, Express khớp route trước, chạy **toàn bộ
param callback liên quan**, xong mới tới chuỗi middleware của route. Hai cuốn sổ tách rời nên thứ
tự dòng code trong file **không ảnh hưởng gì**.

**Nếu xoá dòng đó:** `requireSignin` vẫn gán được `req.auth`; nhưng `isAuth` chạy
`req.profile._id == req.auth._id` với `req.profile` là `undefined` → `TypeError: Cannot read
properties of undefined (reading '_id')`. Vì `isAuth` **không phải** hàm `async` nên Express bắt
được lỗi → trả **500 Internal Server Error** kèm stack trace HTML.

Bài học: `isAuth` và `isAdmin` **phụ thuộc hoàn toàn** vào `req.profile`. Ba thứ luôn phải đi cùng
nhau: `:userId` trên URL + `router.param` + hai middleware.

</details>

**Bài 2.** Áp dụng cách làm đúng ở mục 7 cho Topping: viết lại `routes/topping.js` dùng
`loadCurrentUser` + `isAdminByToken` (file `checkAuthV2.js` bạn tự tạo), **bỏ hẳn `:userId`** khỏi
URL. Gợi ý: có thể gom nhiều middleware vào một mảng.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```js
// yotea-be/src/routes/topping.js  ← phiên bản V2 của BẠN, KHÔNG có :userId
import { Router } from "express";
import { create, list, read, remove, update } from "../controllers/topping";
import { requireSignin } from "../middlewares/checkAuth";
import { isAdminByToken, loadCurrentUser } from "../middlewares/checkAuthV2";

const router = Router();

const onlyAdmin = [requireSignin, loadCurrentUser, isAdminByToken];

router.post("/toppings", onlyAdmin, create);
router.get("/toppings/:slug", read);
router.get("/toppings", list);
router.put("/toppings/:id", onlyAdmin, update);
router.delete("/toppings/:id", onlyAdmin, remove);

export default router;
```

Ba điểm hay: URL ngắn và đúng REST (client chỉ cần gắn header); gom 3 middleware vào **một mảng
`onlyAdmin`** — Express nhận mảng middleware bình thường, đổi chính sách chỉ sửa một chỗ; và không
cần `isAuth` nữa vì chẳng còn "id trên URL" nào để đối chiếu — một middleware **biến mất** chỉ nhờ
sửa thiết kế cho đúng.

Kiểm chứng: gọi `POST http://localhost:8080/api/toppings` (không có id trên URL) với token admin →
200; với token user thường → **403** `{ "message": "Bạn không phải là Admin" }`; không token → 401.

</details>

**Bài 3.** Dùng `Ctrl + Shift + F` trong VS Code, tìm `router.delete` trong `yotea-be/src/routes/`.
Liệt kê tất cả endpoint `DELETE` **không** được `isAdmin` bảo vệ, và nói kẻ xấu làm được gì.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Có **13 dòng `router.delete`** trong 14 file route. Năm cái không có `isAdmin`:

| Endpoint | Middleware | Kẻ xấu làm được gì |
|---|---|---|
| `DELETE /api/orders/:id` (`routes/order.js:12`) | **KHÔNG CÓ GÌ** | Xoá sạch đơn hàng, không cần tài khoản |
| `DELETE /api/orderDetail/:id` (`routes/orderDetail.js:10`) | **KHÔNG CÓ GÌ** | Xoá từng món trong đơn của khách |
| `DELETE /api/comments/:id/:userId` (`routes/comment.js:12`) | `requireSignin, isAuth` | User đăng nhập bất kỳ xoá được bình luận **của người khác** |
| `DELETE /api/ratings/:id/:userId` (`routes/rating.js:12`) | `requireSignin, isAuth` | Xoá đánh giá người khác → thao túng điểm sao sản phẩm |
| `DELETE /api/favoritesProduct/:id/:userId` (`routes/favoritesProduct.js:12`) | `requireSignin, isAuth` | Xoá mục yêu thích của người khác |

8 endpoint `DELETE` còn lại (`category`, `product`, `slider`, `store`, `cateNews`, `news`, `users`,
`contact`) đều đủ `requireSignin, isAuth, isAdmin` — nhóm này an toàn.

Hai dòng đầu chỉ cần gắn thêm `requireSignin, isAuth, isAdmin` (đề xuất thôi — **đừng sửa file dự
án** ở bài này, để dành [Bài 34](34-refactor-du-an.md)). Riêng comment/rating/favorites thì gắn
`isAdmin` là **sai nghiệp vụ** (user phải tự xoá được bình luận của mình); cách đúng là kiểm tra
**quyền sở hữu** ngay trong controller:

```js
// đoạn bạn tự viết thêm, dự án chưa có
const comment = await Comment.findById(req.params.id).exec();
if (!comment) return res.status(404).json({ message: "Không tìm thấy bình luận" });
if (String(comment.userId) !== req.auth._id && req.profile.role !== 1) {
  return res.status(403).json({ message: "Bạn không được xoá bình luận của người khác" });
}
```

</details>

---

## 📌 Tóm tắt

- **Authentication** = "bạn là ai" (token hợp lệ không). **Authorization** = "bạn được làm gì" (`role`).
  Xác thực xong **chưa chắc** được phép.
- `requireSignin` do `express-jwt` v6 sinh ra (cú pháp cũ `expressJWT({...})`): đọc header
  `Authorization: Bearer <token>`, verify bằng `secret`, gắn payload vào **`req.auth`** nhờ
  `requestProperty: "auth"`.
- `req.profile` **không** do middleware auth tạo ra — nó do `userById` gán qua
  `router.param("userId", userById)`, và param callback **luôn chạy trước** mọi middleware của route.
- Thứ tự đầy đủ: **`router.param` → `requireSignin` → `isAuth` → `isAdmin` → controller**. Mỗi chốt
  chỉ có 2 lối ra: `next()` hoặc `res...json()`.
- `isAuth` dùng `==` (không phải `===`) là **cố ý**: `req.profile._id` là ObjectId, `req.auth._id`
  là String. Đổi sang `===` sẽ làm hỏng mọi route cần đăng nhập.
- Toàn dự án có **70 endpoint**: 36 công khai (51%), 11 cần đăng nhập, 23 cần admin — và **9 thao
  tác ghi** không được bảo vệ (cả router `orderDetail`, `DELETE /api/orders/:id`,
  `PATCH /api/products/userUpdate/:id`).
- Thiết kế nhét `:userId` lên URL là **sai lầm**: rườm rà, thừa (token đã có `_id`), rò rỉ id ra
  log, và đẻ ra lỗ hổng `PUT /api/users/updateInfo/:myId/:userId` cho phép user thường **tự phong admin**.
- Cách đúng: middleware đọc `req.auth._id` từ token, tự truy vấn user, trả **403** khi thiếu quyền.

**Từ khoá tra cứu thêm:** `express-jwt`, `JWT Bearer token`, `authentication vs authorization`,
`express router.param`, `role based access control`, `broken access control OWASP`,
`mass assignment vulnerability`, `HTTP 401 vs 403`

➡️ **Bài tiếp theo:** [13 — Viết tài liệu API tự động với Swagger](13-swagger-tai-lieu-api.md) —
dự án đã cài sẵn `swagger-jsdoc` và viết 15 khối chú thích `@swagger`, nhưng chưa từng bật lên lần
nào. Ta sẽ bật nó, rồi viết tài liệu cho API Topping của bạn.
