# Bài 13 — Viết tài liệu API tự động với Swagger

> **Phần 1 · Backend** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [12 — Phân quyền: `requireSignin`, `isAuth`, `isAdmin`](12-phan-quyen-middleware.md) · Bài sau: [14 — Cấu trúc dự án React & luồng khởi động](14-cau-truc-react-app.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu vì sao dự án có backend và frontend **bắt buộc** phải có tài liệu API, và vì sao tài liệu viết tay luôn thất bại.
- Phân biệt được **OpenAPI** (chuẩn mô tả), **swagger-jsdoc** (sinh đặc tả từ comment) và **swagger-ui-express** (dựng trang web đọc đặc tả).
- Đọc hiểu từng dòng YAML trong khối `@swagger`: `paths`, `parameters`, `requestBody`, `content`, `schema`, `$ref`, `responses`.
- Phát hiện một sự thật bất ngờ: dự án đã viết **15 khối** comment Swagger nhưng **chưa hề bật** trang tài liệu.
- Tự tay tạo `swagger.js`, mount `/api-docs`, bật nút **Authorize** và bấm **Try it out** ngay trên trình duyệt.
- Tự viết tài liệu Swagger cho **toàn bộ API Topping** bạn đã xây từ Bài 06 đến Bài 12.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 12](12-phan-quyen-middleware.md) — API Topping của bạn đã đủ 5 thao tác CRUD, có slug, có bộ lọc query, có quan hệ với OrderDetail, và các route ghi đã khoá bằng `requireSignin`, `isAuth`, `isAdmin`.
- MongoDB đang chạy; `npm start` tại `yotea-be` lên được cổng 8080.
- Một tài khoản **admin** để lát nữa test các API cần token.

> Ở bài trước bạn đã khoá các route ghi của Topping để chỉ admin gọi được. Bài này ta làm
> tiếp bước cuối của phần backend: **viết tài liệu** cho đống API đó, để người làm frontend
> không phải mở file controller ra đoán nữa.

---

## 1. Vì sao cần tài liệu API?

Một dự án web thường chia hai người: bạn A viết backend (biết rõ API trả về gì), bạn B viết
frontend (**không** biết API trả về gì). Kết quả là một chuỗi tin nhắn Zalo bất tận:
*"API topping gọi sao thế?" — "POST `/api/toppings/:userId`" — "userId nào?" — "Sao tao gọi
lại 401?"*. Mỗi câu hỏi tốn 5–30 phút của **cả hai**. Yotea có **70 endpoint**.

Cách chữa thường thấy là mở Google Docs gõ bảng "Danh sách API". Nhưng:

| Vấn đề | Hậu quả |
|---|---|
| Tài liệu nằm **tách rời** code | Sửa code xong quên sửa tài liệu — chắc chắn xảy ra |
| Không ai kiểm tra tài liệu đúng hay sai | Frontend làm theo tài liệu sai, debug 2 tiếng mới biết |
| Không bấm thử được | Vẫn phải mở Postman gõ tay lại từ đầu |

Nguyên tắc vàng: **tài liệu phải nằm ngay cạnh code nó mô tả.** Sửa hàm `create` thì phần
mô tả `create` nằm ngay trên đầu hàm — khó mà quên. Đó chính là ý tưởng của Swagger.

---

## 2. OpenAPI, Swagger, và hai thư viện dễ lẫn

> 📖 **Thuật ngữ:**
> - **OpenAPI** — một **chuẩn mô tả API** dạng YAML/JSON, quy định muốn mô tả một endpoint
>   thì phải ghi những khoá nào. Bản hiện hành là **3.0**.
> - **Swagger** — tên bộ công cụ xoay quanh chuẩn đó. Ngày xưa chuẩn này *tên là* Swagger,
>   sau đổi thành OpenAPI, nên hai chữ hay dùng lẫn.

Hai thư viện dự án đã cài (`yotea-be/package.json:24-25`):

```json
    "swagger-jsdoc": "^6.2.0",
    "swagger-ui-express": "^4.3.0",
```

Vai trò **hoàn toàn khác nhau**:

| Thư viện | Nhận vào | Trả ra | Ví von |
|---|---|---|---|
| `swagger-jsdoc` | Các file `.js` có comment `@swagger` | Một **object JavaScript** đúng chuẩn OpenAPI | Người **biên soạn** sách từ các mẩu ghi chú |
| `swagger-ui-express` | Object OpenAPI ở trên | Một **trang web** đẹp, bấm thử được | Người **in và đóng bìa** cuốn sách |

```
src/models/product.js  ──┐  swagger-jsdoc          swagger-ui-express
src/controllers/*.js   ──┼──► đọc + gộp ──► specs ──► dựng trang ──► /api-docs
(comment @swagger)     ──┘
```

Thiếu một trong hai là hỏng: có comment mà không chạy `swagger-jsdoc` thì comment chỉ là
comment; có `specs` mà không mount `swagger-ui-express` thì không có trang nào để mở.
Yotea đang **thiếu cả hai bước sau** — ta chứng minh ở mục 4.

---

## 3. Soi code thật trong dự án

### 3.1. Khối `components/schemas` — khai "hình dạng" của một Product

`yotea-be/src/models/product.js:47-92`

```js
/**
 * @swagger
 * components:
 *  schemas:
 *   Products:
 *    type: object
 *    properties:
 *      _id:
 *        type: string
 *      name:
 *        type: string
 *      image:
 *        type: string
 *      price:
 *        type: number
 *      description:
 *        type: string
 *      status:
 *        type: number
 *        default: 0
 *      view:
 *        type: number
 *        default: 0
 *      favorites:
 *        type: number
 *        default: 0
 *      categoryId:
 *        type: string
 *      slug:
 *        type: string
 *    required:
 *      - name
 *      - image
 *      - price
 *      - categoryId
 *    example:
 *      name: Trà sữa ô long bạch kim
 *      image: https://res.cloudinary.com/levantuan/image/upload/v1645172924/assignment-js/ntnmsjdifbepbbbelzvq.png
 *      price: 20000
 *      description: Mô tả sản phẩm
 *      status: 0
 *      view: 10
 *      favorites: 20
 *      categoryId: _fdakfakhxss
 *      slug: tra-sua-o-long-bach-kim
 */
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 47 | `/**` | Phải mở bằng `/**` (hai dấu sao). Viết `/*` là swagger-jsdoc bỏ qua |
| 48 | `* @swagger` | Thẻ đánh dấu: "phần dưới là YAML, hãy đọc tôi" |
| 49-50 | `components:` → `schemas:` | Khoá gốc chứa các thành phần **tái sử dụng**; `schemas` là ngăn chứa **kiểu dữ liệu** |
| 51 | `Products:` | Tên schema — cái tên này sẽ được `$ref` gọi lại ở nơi khác |
| 52-53 | `type: object` / `properties:` | Product là object; bên dưới liệt kê các trường |
| 54-76 | `_id`, `name`, `price`… | Mỗi trường khai `type`. OpenAPI chỉ có `string`, `number`, `integer`, `boolean`, `array`, `object` |
| 77-81 | `required:` | Các trường **bắt buộc**, viết dạng gạch đầu dòng YAML |
| 82-91 | `example:` | Dữ liệu mẫu — Swagger UI **điền sẵn** vào ô nhập khi bạn bấm "Try it out" |

> 💡 **Chú ý thụt lề.** YAML dựa hoàn toàn vào thụt lề. swagger-jsdoc cắt bỏ phần `` * ``
> đầu dòng rồi mới đưa cho bộ đọc YAML — nghĩa là **thụt lề tính từ sau dấu `*`**. Lệch một
> dấu cách là cả khối vô nghĩa. Đây là lỗi số 1 khi viết Swagger.

Để ý khối này nằm ở **cuối file model**, sau cả `export default` (dòng 45). Đặt đâu cũng
được — swagger-jsdoc đọc file như một file **văn bản**, không quan tâm comment gắn với hàm nào.

### 3.2. Khối `tags` — gom nhóm endpoint

`yotea-be/src/models/product.js:94-99`

```js
/**
 * @swagger
 * tags:
 *  name: Products
 *  description: API dành cho Product
 */
```

`tags` tạo ra các **nhóm gấp/mở** trên giao diện. Endpoint nào ghi `tags: [Products]` sẽ
chui vào nhóm này.

### 3.3. Khối `paths` — mô tả endpoint có path param và body

`yotea-be/src/controllers/product.js:4-34`

```js
/**
 * @swagger
 * /api/products/{userId}:
 *  post:
 *   tags: [Products]
 *   summary: Tạo sản phẩm mới
 *   description: Bắt buộc đăng nhập
 *   parameters:
 *     - in: path
 *       name: userId
 *       description: Id user đã đăng nhập
 *       required: true
 *       schema:
 *         type: string
 *         example: 623fec6776be914e8a89297d
 *   requestBody:
 *    required: true
 *    content:
 *     application/json:
 *      schema:
 *       $ref: '#/components/schemas/Products'
 *   responses:
 *    200:
 *     description: Tạo sản phẩm thành công
 *     content:
 *       application/json:
 *        schema:
 *          $ref: '#/components/schemas/Products'
 *    400:
 *     description: Tạo sản phẩm không thành công
 */
```

Khối này mô tả đúng route `yotea-be/src/routes/product.js:8`:

```js
router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 6 | `/api/products/{userId}:` | Đường dẫn. Express viết `:userId`, OpenAPI viết `{userId}` — **hai cú pháp khác nhau**, phải đổi tay |
| 7 | `post:` | Method HTTP. Một path có thể liệt kê nhiều method |
| 8 | `tags: [Products]` | Xếp endpoint vào nhóm `Products` |
| 9-10 | `summary:` / `description:` | Một dòng ngắn hiện cạnh URL / mô tả dài hiện khi mở ra |
| 11 | `parameters:` | Các tham số **không nằm trong body** |
| 12 | `- in: path` | `in` cho biết tham số nằm đâu: `path`, `query`, `header`, `cookie` |
| 13 | `name: userId` | Phải **trùng khít** với `{userId}` ở dòng 6 |
| 15 | `required: true` | Với `in: path` thì **luôn** phải `true` |
| 16-18 | `schema:` | Kiểu dữ liệu của tham số + giá trị mẫu |
| 19 | `requestBody:` | Dữ liệu gửi trong **body**, chỉ dùng cho POST/PUT/PATCH |
| 21-22 | `content:` → `application/json:` | Body có thể nhiều định dạng; ở đây là JSON (khớp `express.json()` tại `app.js:25`) |
| 23-24 | `schema: $ref: ...` | Hình dạng body chính là schema `Products` khai ở model |
| 25 | `responses:` | Các khả năng trả về, **khoá là mã trạng thái HTTP** |
| 26-31 | `200:` | Thành công, kèm mô tả và hình dạng dữ liệu trả về |
| 32-33 | `400:` | Thất bại — khớp `res.status(400)` trong `catch` của controller |

### 3.4. `$ref` trỏ đi đâu, và vì sao nên dùng?

`$ref: '#/components/schemas/Products'` đọc theo từng mảnh: `#` là "trong **chính** tài
liệu này" (không phải file ngoài), rồi đi vào `components` → `schemas` → lấy schema tên
`Products`. Tức nó trỏ thẳng về khối bạn vừa đọc ở `yotea-be/src/models/product.js:51`.

**Vì sao khai một lần rồi tham chiếu lại?** Sản phẩm xuất hiện ở **6 chỗ** trong tài liệu
(body của `create`, body của `update`, kết quả của `read`/`update`/`remove`, phần tử mảng
của `list`). Chép tay 40 dòng `properties` sáu lần thì thêm một trường mới là phải sửa
**6 chỗ**, chắc chắn sót. Khai một lần → sửa một chỗ → cả tài liệu tự đúng. Đây là nguyên
tắc **DRY** áp dụng cho tài liệu.

Khi API trả về **mảng**, dùng `type: array` + `items` — như `yotea-be/src/controllers/product.js:101-105`:

```yaml
 *      application/json:
 *       schema:
 *        type: array
 *        items:
 *         $ref: '#/components/schemas/Products'
```

Đọc là *"trả về một mảng, mỗi phần tử có hình dạng `Products`"*.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** hàm `list` (`yotea-be/src/controllers/product.js:107-180`)
> nhận rất nhiều query string (`_sort`, `_order`, `_start`, `_limit`, `_expand`, `q`,
> `_like`, `_gte`, `_lte`…) mà bạn đã học ở [Bài 09](09-bo-loc-query.md), nhưng khối comment
> mô tả nó (`:91-106`) **không khai một tham số nào**. Người đọc sẽ tưởng API này không lọc
> được gì. Ta sẽ khai đầy đủ cho API Topping ở mục 5 để thấy cách làm đúng.

### 3.5. Body **không** dùng `$ref` — khai thẳng tại chỗ

`yotea-be/src/controllers/auth.js:4-30`

```js
/**
 * @swagger
 * paths:
 *   /api/signin:
 *    post:
 *     tags: [Auth]
 *     summary: Đăng nhập tài khoản
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *              email:
 *               type: string
 *              password:
 *               type: string
 *             example:
 *              email: admin@gmail.com
 *              password: admin
 *     responses:
 *      200:
 *        description: Trả về thông tin tài khoản đăng nhập
 *      400:
 *        description: Đăng nhập không thành công
 */
```

Hai điểm đáng chú ý:

1. Khối này bắt đầu bằng `paths:` rồi mới tới `/api/signin:`, trong khi khối ở mục 3.3
   nhảy thẳng vào `/api/products/{userId}:`. **Cả hai đều chạy** — swagger-jsdoc thấy khoá
   gốc bắt đầu bằng dấu `/` thì tự hiểu đó là path và nhét vào `paths` giúp. Dự án viết lẫn
   lộn hai kiểu; bạn nên chọn **một** kiểu và dùng nhất quán.
2. Body chỉ có 2 trường và dùng đúng một lần, nên khai thẳng tại chỗ là hợp lý. Quy tắc:
   **dùng lại từ 2 lần trở lên mới tách ra `components/schemas`.**

### 3.6. Một `$ref` gãy — bug thật trong dự án

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `$ref: "#/components/schemas/Users"` xuất hiện
> **8 lần** — tại `controllers/auth.js:80` và `controllers/user.js:23, 31, 75, 108, 214,
> 221, 270` — nhưng schema `Users` **chưa bao giờ được khai báo**. Bạn tự kiểm chứng được:
> mở `yotea-be/src/models/user.js` (98 dòng), không có một chữ `@swagger` nào. Toàn dự án
> chỉ có đúng **một** khối `components/schemas`, nằm ở `models/product.js:49`.
>
> Hậu quả: khi bật trang tài liệu, những chỗ đó hiện lỗi đỏ kiểu
> `Could not resolve reference: #/components/schemas/Users`. Cách sửa nằm ở **Bài tập 2**.

---

## 4. ⚠️ Điểm mấu chốt: toàn bộ công sức đó đang nằm chết

Hãy đọc file khởi động server. Nó chỉ có **52 dòng** — đọc hết trong 30 giây.

`yotea-be/src/app.js:22-30`

```js
const app = express();

// middleware
app.use(express.json());
app.use(cors());
app.use(morgan("tiny"));

app.use("/api", categoryRouter);
app.use("/api", productRouter);
```

Phần còn lại: dòng 1-4 import `express`, `cors`, `morgan`, `mongoose`; dòng 6-20 là 14 lệnh
`import ...Router from "./routes/..."`; dòng 31-42 là 12 lệnh `app.use("/api", ...)` nữa;
dòng 44-48 `mongoose.connect(...)`; dòng 51-52 `app.listen(PORT, ...)`. Hết file.

Bạn thấy chữ `swagger` ở đâu không? **Không có, một chữ cũng không.**

- Không có `import swaggerJsdoc from "swagger-jsdoc"`.
- Không có `import swaggerUi from "swagger-ui-express"`.
- Không có `app.use("/api-docs", ...)`.
- Toàn repo cũng **không có** file cấu hình Swagger nào (`swagger.js`, `swagger.json`, `swagger.yaml`).

**Kết luận chắc chắn:** hai gói đã được cài, 15 khối comment `@swagger` đã được gõ tỉ mỉ
hàng trăm dòng, nhưng **chưa ai bật trang tài liệu lên**. Mở `http://localhost:8080/api-docs`
lúc này chỉ nhận được `Cannot GET /api-docs`.

### 4.1. Thống kê: ai đã có tài liệu, ai chưa

Đứng ở thư mục `yotea-be` và đếm bằng `grep -rn "@swagger" src/controllers src/models`
(Windows PowerShell: `Select-String -Path .\src\controllers\*.js,.\src\models\*.js -Pattern "@swagger"`):

| File | Số khối `@swagger` | Dòng chứa `* @swagger` |
|---|---|---|
| `src/controllers/product.js` | 6 | 5, 50, 92, 183, 244, 299 |
| `src/controllers/user.js` | 5 | 4, 56, 95, 189, 244 |
| `src/controllers/auth.js` | 2 | 5, 69 |
| `src/models/product.js` | 2 | 48, 95 |
| **Tổng** | **15** | |

Còn lại **hoàn toàn trắng**:

| Nhóm | Danh sách |
|---|---|
| Controller chưa có khối nào (**11/14**) | `category`, `cateNews`, `news`, `slider`, `store`, `contact`, `comment`, `rating`, `favoritesProduct`, `order`, `orderDetail` |
| Model chưa có khối nào (**13/14**) | tất cả trừ `product.js` |

Đếm theo endpoint: 14 file route khai tổng cộng **70 endpoint**, chỉ **13** có comment mô tả
(6 product + 5 user + 2 auth) — khoảng **19%**. Ngay cả `checkPassword`
(`yotea-be/src/routes/auth.js:8`) cũng bị bỏ quên dù nằm chung file với hai API đã có tài liệu.

> 💡 **Bài học nghề nghiệp:** viết tài liệu mà không bật lên xem thì cũng như viết test mà
> không chạy. Bước "kiểm chứng" quan trọng ngang bước "làm".

---

## 5. 🛠️ Tự tay làm

> Mục tiêu: cuối phần này bạn mở `http://localhost:8080/api-docs` sẽ thấy trang tài liệu đầy
> đủ, bấm **Authorize** dán token vào, bấm **Try it out** gọi được API Topping thật — ngay
> trên trình duyệt, không cần Postman.
>
> Toàn bộ code trong mục 5 là **code bạn tự viết thêm**, dự án gốc chưa có.

### Bước 1 — Tạo file cấu hình `swagger.js`

```js
// yotea-be/src/swagger.js  ← file MỚI, bạn tự tạo
import swaggerJsdoc from "swagger-jsdoc";

const options = {
  definition: {
    openapi: "3.0.0",
    info: {
      title: "Yotea API",
      version: "1.0.0",
      description: "Tài liệu API cho website bán trà sữa Yotea",
    },
    servers: [
      { url: "http://localhost:8080", description: "Máy của bạn (development)" },
    ],
  },
  apis: ["./src/controllers/*.js", "./src/models/*.js"],
};

const specs = swaggerJsdoc(options);

export default specs;
```

| Khoá | Ý nghĩa |
|---|---|
| `definition` | Phần "khung" của tài liệu — thứ **không** lấy được từ comment |
| `openapi: "3.0.0"` | Khai phiên bản chuẩn. Thiếu dòng này swagger-jsdoc coi là Swagger 2.0 và mọi khối `components` bị hiểu sai |
| `info.title` / `info.version` | Tên và phiên bản hiện ở đầu trang. **Bắt buộc** có |
| `servers[0].url` | Địa chỉ gốc mà nút "Try it out" sẽ bắn request tới |
| `apis` | Danh sách **đường dẫn file** để swagger-jsdoc quét comment |

> ⚠️ **Vì sao `servers` là `http://localhost:8080` chứ không phải `.../api`?** Vì các comment
> sẵn có đã viết path **đầy đủ kèm tiền tố**, ví dụ `/api/products/{userId}`
> (`controllers/product.js:6`). Swagger UI nối `servers.url` + path. Nếu để `servers` là
> `http://localhost:8080/api` thì URL gọi ra thành `http://localhost:8080/api/api/products/...`
> → **404**.

> ⚠️ **Đường dẫn trong `apis` tính từ đâu?** Từ **thư mục bạn gõ `npm start`**, tức `yotea-be`,
> chứ **không** phải từ vị trí file `swagger.js`. Đó là lý do phải viết `./src/controllers/*.js`
> (đúng) chứ không phải `./controllers/*.js` (sai — quét được 0 file, trang tài liệu trống
> trơn mà **không báo lỗi gì cả**). Muốn quét cả file route thì thêm `"./src/routes/*.js"`.

### Bước 2 — Mount trang tài liệu vào `app.js`

Mở `yotea-be/src/app.js`. Thêm **2 dòng import** ngay sau dòng 4 (`import mongoose...`):

```js
// yotea-be/src/app.js — 2 dòng bạn tự thêm
import swaggerUi from "swagger-ui-express";
import specs from "./swagger";
```

Rồi thêm **1 dòng mount** ngay sau `app.use(morgan("tiny"));` (dòng 27), tức **trước** khối
14 dòng `app.use("/api", ...)`:

```js
// yotea-be/src/app.js — dòng bạn tự thêm
app.use("/api-docs", swaggerUi.serve, swaggerUi.setup(specs));
```

**Vì sao đặt đúng chỗ đó?**

| Lý do | Giải thích |
|---|---|
| Sau `express.json()`, `cors()`, `morgan()` | Trang tài liệu cũng cần được log và cần header CORS như mọi request khác |
| Trước các router `/api` | Cho rõ ràng dễ đọc. (Express thực ra **không** nhầm `/api-docs` với `/api`: `app.use("/api", ...)` chỉ khớp khi ký tự kế tiếp là `/` hoặc hết chuỗi, mà đây là dấu `-`.) |
| Không đặt sau `app.listen()` | Mọi `app.use` phải chạy **trước** khi server bắt đầu nghe |

Vì sao có tới ba tham số? `swaggerUi.serve` phục vụ đống file tĩnh (CSS, JS) của giao diện;
`swaggerUi.setup(specs)` trả về trang HTML đã nhồi sẵn `specs` của bạn.

### Bước 3 — Chạy và bấm thử "Try it out"

```bash
# đứng tại thư mục yotea-be
npm start
```

Mở `http://localhost:8080/api-docs` → thấy tiêu đề **Yotea API 1.0.0** và ba nhóm
`Products`, `Users`, `Auth`. Thử endpoint dễ nhất: mở nhóm **Products** → chọn
`GET /api/products` → bấm **Try it out** (góc phải) → bấm **Execute** màu xanh → kéo xuống
**Server response** phải thấy `Code 200` và mảng JSON sản phẩm thật lấy từ MongoDB của bạn.
Giao diện còn hiện sẵn dòng **Curl** tương ứng, copy dán vào terminal là chạy được.

### Bước 4 — Khai `securitySchemes` để test được API cần token

Bấm `POST /api/products/{userId}` lúc này sẽ nhận **401**, vì route đó có `requireSignin`
(`yotea-be/src/routes/product.js:8`) mà Swagger UI chưa biết gửi header `Authorization`.

Nhắc lại từ [Bài 12](12-phan-quyen-middleware.md) — `yotea-be/src/middlewares/checkAuth.js:3-7`:

```js
export const requireSignin = expressJWT({
  algorithms: ["HS256"],
  secret: "TuongVy",
  requestProperty: "auth",
});
```

Nó đọc header `Authorization: Bearer <token>`. Vậy hãy khai đúng kiểu đó: bổ sung khoá
`components` vào trong `definition` của `yotea-be/src/swagger.js` (code bạn tự viết thêm):

```js
    components: {
      securitySchemes: {
        bearerAuth: {
          type: "http",
          scheme: "bearer",
          bearerFormat: "JWT",
          description: "Dán token từ POST /api/signin vào đây (KHÔNG gõ chữ Bearer)",
        },
      },
    },
```

| Khoá | Ý nghĩa |
|---|---|
| `securitySchemes` | Ngăn khai "các cách xác thực mà API này chấp nhận" |
| `bearerAuth` | **Tên tự đặt** — lát nữa endpoint sẽ gọi lại đúng tên này |
| `type: "http"` + `scheme: "bearer"` | Xác thực qua header `Authorization: Bearer <token>` |
| `bearerFormat: "JWT"` | Chỉ để hiển thị cho người đọc biết đây là JWT |

Khai xong mới là "hệ thống có hỗ trợ". Muốn endpoint nào **yêu cầu** token thì thêm hai dòng
`security:` / `- bearerAuth: []` vào chính khối comment của endpoint đó (xem Bước 5).

Cách dùng: gọi `POST /api/signin` bằng Try it out → copy giá trị `token` (chuỗi dài bắt đầu
bằng `eyJ...`) → bấm nút **Authorize** 🔓 góc trên bên phải → dán token → **Authorize** →
**Close**. Ổ khoá chuyển sang đóng 🔒, từ giờ mọi request bấm từ Swagger UI đều tự kèm header
`Authorization: Bearer eyJ...`.

> 🔒 **Ghi chú bảo mật:** token của dự án sống **3 giờ** (`yotea-be/src/controllers/auth.js:53`).
> Hết hạn thì signin lại rồi Authorize lại.

### Bước 5 — Viết tài liệu cho toàn bộ API Topping

Mở lại `yotea-be/src/models/topping.js` và `yotea-be/src/controllers/topping.js` bạn đã viết
từ Bài 05 đến Bài 12.

**5a. Schema + tag.** Thêm vào **cuối** `yotea-be/src/models/topping.js`, sau `export default`:

```js
// comment bạn tự viết thêm — dự án gốc chưa có file topping.js
/**
 * @swagger
 * components:
 *  schemas:
 *   Toppings:
 *    type: object
 *    properties:
 *      _id:    { type: string }
 *      name:   { type: string }
 *      price:  { type: number }
 *      status: { type: number, default: 0 }
 *      slug:   { type: string }
 *    required:
 *      - name
 *      - price
 *    example:
 *      name: Trân châu đường đen
 *      price: 10000
 *      slug: tran-chau-duong-den
 */

/**
 * @swagger
 * tags:
 *  name: Toppings
 *  description: API quản lý topping (trân châu, thạch, pudding...)
 */
```

> 💡 `{ type: string }` là **YAML flow style** — viết object gọn trên một dòng, tương đương
> xuống dòng thụt lề. Nếu model Topping của bạn có thêm/bớt trường (ví dụ có `image`), sửa
> `properties` cho **khớp đúng** schema Mongoose. Tài liệu sai còn tệ hơn không có tài liệu.

**5b. Năm endpoint trong controller.** Mỗi khối đặt **ngay trên** hàm nó mô tả, trong
`yotea-be/src/controllers/topping.js` (toàn bộ là comment bạn tự viết thêm):

```js
/**
 * @swagger
 * /api/toppings:
 *  get:
 *   tags: [Toppings]
 *   summary: Lấy danh sách topping
 *   description: Hỗ trợ lọc, sắp xếp, phân trang và tìm kiếm toàn văn
 *   parameters:
 *     - { in: query, name: _sort,  description: Trường sắp xếp, schema: { type: string, example: price } }
 *     - { in: query, name: _order, description: Chiều sắp xếp,  schema: { type: string, enum: [asc, desc] } }
 *     - { in: query, name: _start, description: Bỏ qua bao nhiêu bản ghi, schema: { type: number, example: 0 } }
 *     - { in: query, name: _limit, description: Lấy tối đa bao nhiêu bản ghi, schema: { type: number, example: 8 } }
 *     - { in: query, name: q,      description: Từ khoá tìm kiếm, schema: { type: string } }
 *   responses:
 *    200:
 *     description: Mảng topping
 *     content:
 *      application/json:
 *       schema:
 *        type: array
 *        items:
 *         $ref: '#/components/schemas/Toppings'
 *    400: { description: Không lấy được danh sách topping }
 */
export const list = async (req, res) => { /* code Bài 09 của bạn */ };

/**
 * @swagger
 * /api/toppings/{slug}:
 *  get:
 *   tags: [Toppings]
 *   summary: Lấy chi tiết một topping theo slug
 *   parameters:
 *     - { in: path, name: slug, required: true, schema: { type: string, example: tran-chau-duong-den } }
 *   responses:
 *    200:
 *     description: Thông tin topping
 *     content:
 *      application/json:
 *       schema:
 *        $ref: '#/components/schemas/Toppings'
 *    400: { description: Không tìm thấy topping }
 */
export const read = async (req, res) => { /* code Bài 08 của bạn */ };

/**
 * @swagger
 * /api/toppings/{userId}:
 *  post:
 *   tags: [Toppings]
 *   summary: Tạo topping mới
 *   description: Chỉ Admin. Bắt buộc gửi token.
 *   security:
 *     - bearerAuth: []
 *   parameters:
 *     - { in: path, name: userId, description: Id admin đang đăng nhập, required: true, schema: { type: string } }
 *   requestBody:
 *    required: true
 *    content:
 *     application/json:
 *      schema:
 *       $ref: '#/components/schemas/Toppings'
 *   responses:
 *    200: { description: Topping vừa tạo }
 *    400: { description: Tạo topping thất bại }
 *    401: { description: Chưa đăng nhập hoặc không phải Admin }
 */
export const create = async (req, res) => { /* code Bài 07 của bạn */ };

/**
 * @swagger
 * /api/toppings/{id}/{userId}:
 *  put:
 *   tags: [Toppings]
 *   summary: Cập nhật topping
 *   security:
 *     - bearerAuth: []
 *   parameters:
 *     - { in: path, name: id,     required: true, schema: { type: string } }
 *     - { in: path, name: userId, required: true, schema: { type: string } }
 *   requestBody:
 *    required: true
 *    content:
 *     application/json:
 *      schema:
 *       $ref: '#/components/schemas/Toppings'
 *   responses:
 *    200: { description: Topping sau khi cập nhật }
 *    400: { description: Cập nhật topping thất bại }
 *    401: { description: Chưa đăng nhập hoặc không phải Admin }
 *  delete:
 *   tags: [Toppings]
 *   summary: Xoá topping
 *   security:
 *     - bearerAuth: []
 *   parameters:
 *     - { in: path, name: id,     required: true, schema: { type: string } }
 *     - { in: path, name: userId, required: true, schema: { type: string } }
 *   responses:
 *    200: { description: Topping vừa bị xoá }
 *    400: { description: Xoá topping thất bại }
 *    401: { description: Chưa đăng nhập hoặc không phải Admin }
 */
export const update = async (req, res) => { /* code Bài 07 của bạn */ };
```

Hai mẹo trong đoạn trên: (1) `put` và `delete` dùng chung path `/api/toppings/{id}/{userId}`
nên gộp vào **một** khối, hai method là hai khoá con — dự án gốc tách làm hai khối riêng
(`controllers/product.js:182-219` và `:298-329`), vẫn chạy vì swagger-jsdoc gộp lại giúp,
nhưng viết chung dễ đọc hơn; (2) endpoint nào ghi `security: - bearerAuth: []` sẽ hiện biểu
tượng ổ khoá trên giao diện.

---

## 6. ✅ Kiểm chứng kết quả

```bash
# đứng tại thư mục yotea-be
npm start
```

Terminal in ra `App is running on port: 8080` và `Connected to MongoDB`. Rồi làm lần lượt
6 bước, sai chỗ nào dừng lại sửa chỗ đó:

| # | Việc cần làm | Kết quả phải thấy |
|---|---|---|
| 1 | Mở `http://localhost:8080/api-docs` | Trang Swagger, tiêu đề **Yotea API 1.0.0** |
| 2 | Đếm số nhóm | 4 nhóm: `Products`, `Users`, `Auth`, **`Toppings`** |
| 3 | `GET /api/toppings` → Try it out → Execute | `Code 200` + mảng topping thật |
| 4 | `POST /api/toppings/{userId}` khi **chưa** Authorize | `Code 401` |
| 5 | Signin → Authorize (dán token) → thử lại bước 4 | `Code 200` + topping vừa tạo |
| 6 | Mở lại `GET /api/toppings` | Topping vừa tạo đã có trong danh sách |

Ở bước 3, phần **Server response** phải giống thế này:

```json
[
  {
    "_id": "6650a1f2c4e8b91234abcd01",
    "name": "Trân châu đường đen",
    "price": 10000,
    "status": 0,
    "slug": "tran-chau-duong-den"
  }
]
```

Muốn soi **đặc tả thô** mà swagger-jsdoc sinh ra (rất hữu ích khi debug), thêm tạm route sau
vào `app.js` rồi mở `http://localhost:8080/api-docs.json`. Nếu `paths` là `{}` rỗng → chắc
chắn đường dẫn trong `apis` sai.

```js
// yotea-be/src/app.js — route tạm để debug, bạn tự thêm rồi xoá đi sau
app.get("/api-docs.json", (req, res) => res.json(specs));
```

---

## 7. 🐞 Lỗi thường gặp

| Thông báo lỗi / hiện tượng | Nguyên nhân | Cách sửa |
|---|---|---|
| `Cannot GET /api-docs` | Chưa thêm `app.use("/api-docs", ...)`, hoặc đặt sau `app.listen` | Xem lại Bước 2 |
| Trang mở được nhưng **trống trơn** | `apis` trỏ sai đường dẫn (tính từ nơi gõ `npm start`) | Sửa thành `./src/controllers/*.js` |
| `Error in ./src/controllers/topping.js : YAMLSemanticError...` | Thụt lề YAML sai, hoặc dùng **Tab** thay dấu cách | YAML **cấm Tab**. Chỉnh các khoá cùng cấp thẳng hàng nhau |
| `Could not resolve reference: #/components/schemas/Toppings` | Tên schema gõ sai, hoặc file model chưa nằm trong `apis` | Kiểm tra chính tả (`Toppings` ≠ `Topping`), thêm `./src/models/*.js` |
| Execute ra URL `.../api/api/toppings` → 404 | `servers.url` đã có `/api` mà path cũng có `/api` | Để `servers.url` là `http://localhost:8080` |
| Luôn `401` dù đã Authorize | Token hết hạn (3 giờ), hoặc dán kèm chữ `Bearer` vào ô | Signin lại; ô Authorize chỉ dán **phần token** |
| Endpoint không hiện ổ khoá | Comment thiếu `security: - bearerAuth: []` | Thêm vào, xem Bước 4 |
| Endpoint không hiện lên dù đã viết comment | Mở comment bằng `/*` thay vì `/**`, hoặc thiếu `* @swagger` | Sửa lại đúng `/**` + `* @swagger` |
| Sửa comment mà trang không đổi | swagger-jsdoc chỉ đọc file **một lần lúc khởi động** | `Ctrl + C` rồi `npm start` lại |
| `Cannot find module 'swagger-jsdoc'` | Chưa cài | `npm install` tại `yotea-be` |

---

## 8. 📝 Bài tập

**Bài 1.** Endpoint `PATCH /api/products/userUpdate/:id` (`yotea-be/src/routes/product.js:12`)
đã có tài liệu tại `yotea-be/src/controllers/product.js:243-277`. Đọc khối đó và trả lời: vì
sao `requestBody` **không** dùng `$ref: '#/components/schemas/Products'` mà lại khai
`type: object` với đúng hai trường `view` và `favorites`?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Vì controller `clientUpdate` (`yotea-be/src/controllers/product.js:278-296`) chỉ đọc đúng
hai trường đó ra khỏi body:

```js
const { view, favorites } = req.body;
```

Mọi trường khác client gửi lên đều bị **bỏ qua**. Nếu tài liệu ghi `$ref: Products`, người
đọc sẽ tưởng có thể gửi `name`, `price`, `image`… và cập nhật được — sai hoàn toàn.

**Nguyên tắc rút ra:** `$ref` chỉ dùng khi API thật sự nhận **cả** schema đó. Khi API chỉ
nhận một phần, hãy khai thẳng đúng phần đó. Tài liệu mô tả *hành vi thật của code*, không
phải *hình dạng của bảng dữ liệu*.

</details>

**Bài 2.** Sửa `$ref` gãy đã nêu ở mục 3.6: viết khối `components/schemas/Users` cho
`yotea-be/src/models/user.js`, đối chiếu với schema Mongoose ở dòng 4-64 của file đó.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Thêm vào cuối `yotea-be/src/models/user.js` (code bạn tự viết thêm):

```js
/**
 * @swagger
 * components:
 *  schemas:
 *   Users:
 *    type: object
 *    properties:
 *      _id:      { type: string }
 *      email:    { type: string }
 *      username: { type: string }
 *      fullName: { type: string }
 *      phone:    { type: string }
 *      address:  { type: string }
 *      avatar:   { type: string }
 *      role:     { type: number, default: 0 }
 *      active:   { type: number, default: 1 }
 *    required: [email, password, username, fullName, phone]
 *    example:
 *      email: admin@gmail.com
 *      username: admin
 *      fullName: Nguyễn Văn A
 *      phone: "0912345678"
 *      role: 1
 */
```

Hai chỗ tinh tế:

1. `password` **có** trong `required` (đăng ký bắt buộc gửi) nhưng **không** có trong
   `properties`, để tránh gợi ý rằng API trả mật khẩu về. Chuẩn hơn nữa là tách hai schema:
   `UserInput` (có `password`) và `UserResponse` (không có).
2. `phone: "0912345678"` phải để trong **nháy kép** — không có nháy, YAML đọc thành số và
   nuốt mất số 0 ở đầu.

</details>

**Bài 3.** Ở [Bài 10](10-quan-he-va-populate.md) bạn đã nối Topping với OrderDetail. Mở
`yotea-be/src/models/orderDetail.js` (35 dòng) và trả lời: nếu viết tài liệu Swagger cho
`POST /api/orderDetail` dựa trên **những gì frontend thật sự gửi lên** trong
`yotea-fe/src/pages/user/cart/CheckoutPage.js`, bạn sẽ phát hiện điều gì bất thường?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Schema `orderDetail` chỉ khai **6 trường**: `orderId`, `productId`, `productPrice`,
`quantity`, `ice`, `sugar` (`yotea-be/src/models/orderDetail.js:3-31`). **Không có**
`toppingId`, cũng **không có** `sizeId`.

Nhưng `CheckoutPage.js` lại gửi lên các trường đó. Mongoose mặc định chạy ở chế độ
`strict: true` — mọi trường **không có trong schema** bị **âm thầm loại bỏ**: không báo lỗi,
không cảnh báo, API vẫn trả `200`. Dữ liệu topping của khách **bốc hơi** giữa đường mà không
ai biết.

Đây đúng là loại bug mà viết tài liệu giúp lộ ra: khi ngồi liệt kê từng trường trong
`requestBody`, bạn buộc phải đối chiếu **cái frontend gửi** với **cái model nhận** — và thấy
ngay hai bên lệch nhau.

Cách sửa: bổ sung `toppingId` (dạng `ObjectId, ref: "Topping"`) vào schema `orderDetail`, rồi
khai nó trong `components/schemas/OrderDetails`. Mổ xẻ kỹ ở [Bài 28](28-thanh-toan.md) và
[Bài 34](34-refactor-du-an.md).

**Mẹo phòng bug loại này:** khai schema với `{ strict: "throw" }` — gửi thừa trường sẽ nhận
`StrictModeError` ngay lập tức thay vì im lặng nuốt mất.

</details>

**Bài 4.** Còn **11 controller** chưa có một dòng tài liệu nào. Chọn `category` và viết trọn
bộ: `components/schemas/Categories` trong `models/category.js`, khối `tags`, và các khối
`paths` cho 5 route ở `yotea-be/src/routes/category.js:8-12`.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Bảng đối chiếu 5 route để không nhầm khi đổi cú pháp:

| Route Express | Path OpenAPI | Method | Cần token? |
|---|---|---|---|
| `/category/:userId` | `/api/category/{userId}` | `post` | Có (admin) |
| `/category/:slug` | `/api/category/{slug}` | `get` | Không |
| `/category` | `/api/category` | `get` | Không |
| `/category/:id/:userId` | `/api/category/{id}/{userId}` | `put` | Có (admin) |
| `/category/:id/:userId` | `/api/category/{id}/{userId}` | `delete` | Có (admin) |

Quy trình: (1) chép danh sách trường từ `models/category.js` thành schema `Categories`;
(2) thêm khối `tags` với `name: Categories`; (3) đổi `:param` của Express thành `{param}` của
OpenAPI; (4) route nào có `requireSignin` thì thêm `security: - bearerAuth: []` và ghi thêm
phản hồi `401`. Hai dòng cuối dùng chung path → gộp `put` và `delete` vào một khối.

> ⚠️ Bẫy có thật: `GET /api/category/{slug}` và `GET /api/category` là **hai path khác nhau**
> với OpenAPI, phải viết thành hai khoá riêng. Đừng gộp.

</details>

---

## 📌 Tóm tắt

- Tài liệu API **phải nằm cạnh code**, nếu không nó lạc hậu ngay tuần đầu tiên.
- **OpenAPI 3.0** là chuẩn; **swagger-jsdoc** đọc comment `@swagger` sinh ra đặc tả;
  **swagger-ui-express** biến đặc tả thành trang web bấm thử được. Thiếu một là hỏng.
- Cấu trúc một khối: `paths` → path → method → `tags`/`summary`/`description` → `parameters`
  (`in: path` / `in: query`) → `requestBody` → `content` → `schema` → `responses` (khoá là mã HTTP).
- `$ref: '#/components/schemas/X'` trỏ về schema khai một lần trong `components` — khai một
  chỗ, dùng nhiều nơi, sửa một lần là đúng cả tài liệu.
- **Dự án Yotea đã viết 15 khối `@swagger` nhưng `app.js` chưa hề mount swagger-ui**, nên toàn
  bộ công sức đó chưa dùng được. Chỉ 13/70 endpoint có tài liệu; 11/14 controller và 13/14
  model hoàn toàn trắng; `$ref` tới `Users` còn bị gãy vì schema đó chưa từng được khai.
- Ba việc nhỏ (tạo `swagger.js`, 2 dòng import, 1 dòng `app.use`) là đủ để hồi sinh đống
  comment đó thành một trang tài liệu sống.
- Khai `securitySchemes.bearerAuth` + `security: - bearerAuth: []` để bấm **Authorize** và
  test được cả API cần token ngay trên trình duyệt.
- YAML **cấm Tab**, **thụt lề tính từ sau dấu `*`**, comment phải mở bằng `/**`.

**Từ khoá tra cứu thêm:** `OpenAPI 3.0 specification`, `swagger-jsdoc options apis`,
`swagger-ui-express setup`, `openapi securitySchemes bearerAuth`, `openapi $ref components schemas`

➡️ **Bài tiếp theo:** [14 — Cấu trúc dự án React & luồng khởi động](14-cau-truc-react-app.md) —
backend đã xong và đã có tài liệu; giờ ta bước sang phía trình duyệt, mổ xẻ xem React khởi
động từ dòng nào và dữ liệu chảy vào giao diện ra sao.
