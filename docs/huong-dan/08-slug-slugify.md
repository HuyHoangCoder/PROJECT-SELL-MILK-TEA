# Bài 08 — Slug thân thiện SEO với slugify

> **Phần 1 · Backend cơ bản** — Thời lượng ước tính: **~50 phút**
> ⬅️ Bài trước: [07 — CRUD trọn vẹn với Category](07-crud-category.md) · Bài sau: [09 — Bộ lọc kiểu json-server](09-bo-loc-query.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Giải thích được **slug** là gì và vì sao `/san-pham/tra-sua-tran-chau-duong-den` tốt hơn `/san-pham/6650a1f2c4e8b91234abcd01`.
- Dùng được thư viện **slugify** với các option `lower`, `locale`, `strict`, `remove` — và biết **chính xác** dự án Yotea đang dùng thiếu option nào.
- Đọc hiểu được đoạn sinh slug trong `create` và `update` của `controllers/product.js`.
- Hiểu vì sao trường `slug` khai báo `unique: true`, và **tự tay tái hiện được lỗi `E11000 duplicate key`** — một bug thật của dự án.
- Tự viết được hàm sinh slug **không bao giờ trùng** cho chức năng Topping bạn đang xây.
- Biết resource nào trong Yotea có slug, resource nào không, và vì sao.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 07 — CRUD trọn vẹn với Category](07-crud-category.md). Ở bài trước bạn đã có đủ **5 thao tác CRUD cho Topping** và test được bằng Postman.
- Backend chạy được bằng `npm start` (cổng 8080), MongoDB đang bật.
- Postman đã có sẵn token của tài khoản admin.

> 💡 Ở bài 07 bạn đã làm Topping "chạy được". Bài này ta làm cho nó **chạy đẹp**: mỗi
> topping sẽ có một đường dẫn thân thiện, và hai topping trùng tên cũng không làm sập API.

---

## 1. Slug là gì?

### 1.1. Hai cái URL, cùng trỏ tới một sản phẩm

Hãy nhìn hai đường dẫn sau. Cả hai đều mở đúng một ly trà sữa:

```
❌  http://localhost:3000/san-pham/6650a1f2c4e8b91234abcd01
✅  http://localhost:3000/san-pham/tra-sua-tran-chau-duong-den
```

Chuỗi `6650a1f2c4e8b91234abcd01` là `_id` — ObjectId 24 ký tự hex do MongoDB tự sinh
(đã nói ở [Bài 03](03-kien-thuc-nen.md) mục 4.2). Chuỗi `tra-sua-tran-chau-duong-den`
là **slug**: phiên bản "đã dọn dẹp" của tên sản phẩm, chỉ còn chữ thường không dấu, số
và dấu gạch ngang.

> 📖 **Thuật ngữ:** *slug* — trong ngành báo chí, "slug" là cái tên rút gọn dán lên bản
> thảo để nhận diện nhanh. Web mượn lại nghĩa đó: một mẩu chuỗi ngắn, dễ đọc, đại diện
> cho một bản ghi trên URL.

### 1.2. Slug được lợi gì?

| Khía cạnh | Dùng `_id` | Dùng `slug` |
|---|---|---|
| **Người dùng đọc URL** | Không hiểu gì | Đọc là biết đang xem món nào |
| **Chia sẻ qua Zalo/Messenger** | Trông như link rác, dễ bị nghi ngờ | Nhìn là muốn bấm |
| **SEO** | Google không lấy được từ khoá nào từ chuỗi hex | Google thấy `tra-sua-tran-chau-duong-den` → hiểu trang này nói về trà sữa trân châu đường đen |
| **Gõ tay / đọc qua điện thoại** | Bất khả thi | Được |
| **Lộ thông tin hệ thống** | Lộ `_id` thật của record | Che được `_id` |
| **Đổi tên sản phẩm** | URL không đổi (ổn định) | URL đổi theo → link cũ chết |

Hai dòng cuối là **hai mặt của cùng một đồng xu**: slug đẹp hơn nhưng kém ổn định hơn.
Nhiều website lớn chọn giải pháp lai — `/san-pham/tra-sua-tran-chau-duong-den-6650a1f2`
— vừa đẹp vừa không bao giờ trùng. Yotea chọn phương án **slug thuần**, nên sẽ dính đúng
hai vấn đề mà ta mổ xẻ ở mục 4.

### 1.3. Quy tắc của một slug tốt

```
Trà sữa trân châu đường đen
        ↓ bỏ dấu tiếng Việt
Tra sua tran chau duong den
        ↓ hạ chữ thường
tra sua tran chau duong den
        ↓ thay khoảng trắng bằng dấu -
tra-sua-tran-chau-duong-den
        ↓ bỏ ký tự đặc biệt, gộp dấu - liên tiếp
tra-sua-tran-chau-duong-den   ✅
```

Bốn bước đó chính là việc mà thư viện `slugify` làm hộ bạn.

---

## 2. Thư viện `slugify`

Dự án khai báo nó ở `yotea-be/package.json:23`:

```json
    "slugify": "^1.6.5",
```

Cách dùng cơ bản:

```js
import slugify from "slugify";

slugify("Trà sữa trân châu đường đen");
```

### 2.1. Bốn option cần biết

| Option | Kiểu | Tác dụng |
|---|---|---|
| `lower` | boolean | Hạ toàn bộ về chữ thường |
| `locale` | string | Chọn bảng chuyển ký tự theo ngôn ngữ. **`"vi"` là cái ta cần** |
| `strict` | boolean | Vứt bỏ mọi ký tự không phải chữ/số/dấu gạch ngang |
| `remove` | RegExp | Tự chỉ định các ký tự cần xoá |
| `replacement` | string | Ký tự nối, mặc định là `"-"` |
| `trim` | boolean | Cắt dấu nối thừa ở hai đầu, mặc định `true` |

### 2.2. ⚠️ slugify xử lý tiếng Việt ra sao — kiểm chứng thực tế

Đây là phần quan trọng nhất của bài. Mình đã chạy thật với đúng phiên bản `slugify@1.6.5`
đang cài trong `yotea-be/node_modules`. Bạn cũng nên tự chạy lại:

```bash
# đứng tại thư mục yotea-be
node -e "const s=require('slugify'); console.log(s('Trà sữa trân châu đường đen'))"
```

Kết quả **không truyền option nào** — đúng như dự án đang làm:

| Đầu vào | `slugify(name)` — dự án đang dùng |
|---|---|
| `Trà sữa trân châu đường đen` | `Tra-sua-tran-chau-djuong-djen` |
| `Trà Sữa Ô Long Bạch Kim` | `Tra-Sua-O-Long-Bach-Kim` |
| `Trà Sữa / Đá Xay` | `Tra-Sua-DJa-Xay` |
| `Combo 2 ly 50%` | `Combo-2-ly-50percent` |
| `Trà sữa & Cà phê` | `Tra-sua-and-Ca-phe` |
| `Thạch Phô Mai` | `Thach-Pho-Mai` |

Ba điều rút ra:

**(1) Dấu tiếng Việt được bỏ đúng — trừ chữ `đ`/`Đ`.**
`à â ê ô ơ ư ạ ệ ộ...` đều được chuyển thành chữ cái Latin trần. Nhưng `đ` bị biến thành
**`dj`** và `Đ` thành **`DJ`**. Lý do: bảng ký tự mặc định của slugify lấy theo tiếng
Croatia, nơi `đ` phát âm gần với `dj`. Với tiếng Việt thì đây là **lỗi chính tả**:
"đường đen" → `djuong-djen` thay vì `duong-den`.

**(2) Chữ hoa KHÔNG được hạ xuống.** Vì code không truyền `{ lower: true }`, slug trả về
vẫn giữ nguyên `Tra-Sua-O-Long-Bach-Kim`.

**(3) Ký tự đặc biệt bị "dịch" chứ không bị xoá.** `%` → `percent`, `&` → `and`.
Còn `/` thì bị xoá hẳn.

### 2.3. Sửa lại thì ra gì?

Chỉ cần thêm ba option, mọi thứ đúng ngay:

```js
slugify("Trà sữa trân châu đường đen", { lower: true, locale: "vi", strict: true });
// → "tra-sua-tran-chau-duong-den"

slugify("Trà Sữa / Đá Xay", { lower: true, locale: "vi", strict: true });
// → "tra-sua-da-xay"
```

- `locale: "vi"` nạp bảng ký tự tiếng Việt → `đ` thành `d`, `Đ` thành `D`. ✅
- `lower: true` hạ chữ thường ngay từ tầng thư viện.
- `strict: true` dọn nốt các ký tự lạ.

> ⚠️ **Lưu ý về `remove`:** `remove` chạy **sau** bảng ký tự, nên
> `slugify("Combo 2 ly 50%", { remove: /[%]/g })` vẫn ra `Combo-2-ly-50percent` —
> vì `%` đã bị đổi thành chữ `percent` trước khi regex kịp xoá. Muốn bỏ hẳn `%`
> thì phải xử lý chuỗi **trước** khi gọi slugify: `name.replace(/%/g, "")`.

---

## 3. Soi code thật trong dự án

### 3.1. Import và hàm `create`

`yotea-be/src/controllers/product.js:1-2`

```js
import slugify from "slugify";
import Product from "../models/product";
```

`yotea-be/src/controllers/product.js:35-47`

```js
export const create = async (req, res) => {
  req.body.slug = slugify(req.body.name);

  try {
    const product = await new Product(req.body).save();
    res.json(product);
  } catch (error) {
    res.status(400).json({
      message: "Thêm sản phẩm thất bại",
      error,
    });
  }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 35 | `export const create = async (req, res) => {` | Controller tạo sản phẩm mới |
| 36 | `req.body.slug = slugify(req.body.name);` | **Ghi thẳng vào `req.body`** một trường `slug` mới, sinh từ `name`. Không truyền option nào |
| 38 | `try {` | Từ đây mới có lưới an toàn |
| 39 | `await new Product(req.body).save();` | Dựng document từ **toàn bộ** `req.body` (kể cả `slug` vừa nhét vào) rồi lưu |
| 40 | `res.json(product);` | Trả document vừa lưu — đã có `_id`, `slug`, `createdAt` |
| 41–46 | `catch` | Mọi lỗi lưu (thiếu field, trùng slug…) → 400 kèm `error` |

> ⚠️ **Chỗ này dự án làm chưa chuẩn (bug thật, gây SẬP SERVER):** dòng 36 nằm **ngoài**
> `try`. Nếu admin gửi body thiếu `name`, `slugify(undefined)` ném
> `Error: slugify: string argument expected`. Vì hàm là `async`, lỗi này biến thành
> **rejected promise** mà Express 4 không bắt được → Node giết tiến trình, server chết,
> `nodemon` phải restart. Cách sửa: kiểm tra `req.body.name` trước, hoặc đưa dòng 36 vào
> trong `try`. Ta sẽ áp dụng cách đúng ngay trong phần 🛠️ bên dưới.

### 3.2. Hàm `update` — sinh lại slug mỗi lần sửa

`yotea-be/src/controllers/product.js:220-241`

```js
export const update = async (req, res) => {
  const filter = { _id: req.params.id };
  const update = {
    ...req.body,
    slug: slugify(req.body.name),
  };
  const options = { new: true };

  try {
    const product = await Product.findOneAndUpdate(
      filter,
      update,
      options
    ).exec();
    res.json(product);
  } catch (error) {
    res.status(400).json({
      message: "Cập nhật sản phẩm thất bại",
      error,
    });
  }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 221 | `const filter = { _id: req.params.id };` | Tìm sản phẩm theo `_id` lấy từ URL |
| 222–225 | `{ ...req.body, slug: slugify(req.body.name) }` | **Spread** toàn bộ dữ liệu client gửi lên, rồi **ghi đè** trường `slug` bằng slug mới |
| 226 | `const options = { new: true };` | Bảo Mongoose trả về document **sau** khi sửa (mặc định là bản cũ) |
| 229–233 | `findOneAndUpdate(filter, update, options)` | Một lệnh: tìm + sửa + trả về |
| 235–240 | `catch` | Lỗi → 400 |

Thứ tự trong object ở dòng 222–225 rất quan trọng: `...req.body` đứng **trước**, `slug`
đứng **sau**. Nhờ vậy dù client có cố gửi kèm `slug: "hack-me"` thì nó cũng bị đè.

Hai controller khác làm y hệt:

`yotea-be/src/controllers/category.js:120-126`

```js
export const update = async (req, res) => {
    const filter = { _id: req.params.id };
    const update = {
        ...req.body,
        slug: slugify(req.body.name)
    };
    const options = { new: true };
```

Còn `news` thì sinh slug từ `title` chứ không phải `name` — `yotea-be/src/controllers/news.js:5`:

```js
    req.body.slug = slugify(req.body.title);
```

### 3.3. Khai báo `slug` trong Model

`yotea-be/src/models/product.js:35-40`

```js
    slug: {
        type: String,
        required: true,
        unique: true,
        lowercase: true,
    }
```

`yotea-be/src/models/category.js:9-14`

```js
    slug: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
    },
```

**Đọc từng dòng:**

| Thuộc tính | Ý nghĩa |
|---|---|
| `type: String` | Slug là chuỗi |
| `required: true` | Không có slug thì không cho lưu |
| `unique: true` | Tạo **unique index** trên MongoDB — hai document không được cùng slug |
| `lowercase: true` | **Setter**: Mongoose tự hạ chữ thường trước khi lưu |

### 3.4. `lowercase: true` chính là thứ cứu vãn cho việc thiếu `{ lower: true }`

Nhớ mục 2.2 chứ? `slugify("Trà Sữa Ô Long Bạch Kim")` trả về `Tra-Sua-O-Long-Bach-Kim`
— **còn chữ hoa**. Vậy vì sao slug trong database lại là chữ thường?

Vì `lowercase: true` ở model là một **setter của Mongoose**: nó chạy trong lúc Mongoose
"cast" (ép kiểu) giá trị, **trước khi** ghi xuống MongoDB. Mình đã kiểm chứng cả hai
đường ghi:

```js
// đường 1 — qua .save()   (dùng trong create)
new Product({ name: "X", slug: "Tra-Sua-DJa-Xay" }).slug
// → "tra-sua-dja-xay"

// đường 2 — qua findOneAndUpdate()   (dùng trong update)
// Mongoose cast update thành: { $set: { slug: "tra-sua-dja-xay", name: "X" } }
```

Cả hai đều được hạ chữ thường. Đây là lý do dự án **chạy đúng phần chữ hoa** dù controller
quên `{ lower: true }`.

> ⚠️ **Nhưng đây là một sự phụ thuộc ngầm rất dễ vỡ.** Nếu mai này ai đó bỏ
> `lowercase: true` khỏi model (tưởng là thừa), slug sẽ lập tức có chữ hoa và mọi link
> `/san-pham/tra-sua-...` đang tồn tại sẽ **404 hết** — mà không ai hiểu tại sao. Nguyên
> tắc: **đừng để một tầng phải cứu lỗi của tầng khác**. Sinh slug đúng ngay tại controller
> mới là cách làm sạch sẽ.
>
> Và lưu ý: `lowercase` **không** cứu được chữ `đ`. `Đá Xay` vẫn ra `dja-xay`, sai chính tả.

---

## 4. 🐞 `unique: true` và lỗi E11000 — bug thật của Yotea

### 4.1. Vì sao slug phải `unique`?

Vì backend tra sản phẩm **bằng slug** — `yotea-be/src/routes/product.js:8-13`:

```js
router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/products/:slug", read);
router.get("/products", list);
router.put("/products/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.patch("/products/userUpdate/:id", clientUpdate);
router.delete("/products/:id/:userId", requireSignin, isAuth, isAdmin, remove);
```

Dòng 9: `GET /api/products/:slug` → controller `read` chạy `Product.findOne({ slug: ... })`.
Nếu hai sản phẩm cùng slug, `findOne` chỉ trả về **một** cái — sản phẩm còn lại vĩnh viễn
không ai truy cập được. `unique: true` chính là hàng rào chặn tình huống đó.

### 4.2. Tái hiện lỗi trong 30 giây

Vào trang admin (hoặc Postman), thêm **hai sản phẩm trùng tên**:

```
POST http://localhost:8080/api/products/<userId>
Authorization: Bearer <token admin>

{ "name": "Trà sữa matcha", "image": "https://...", "price": 40000, "categoryId": "..." }
```

Lần 1: `200 OK`, slug = `tra-sua-matcha`.
Lần 2 (y hệt): **400 Bad Request**.

Trong **terminal backend** bạn thấy dòng:

```
E11000 duplicate key error collection: yotea.products index: slug_1 dup key: { slug: "tra-sua-matcha" }
```

Còn JSON trả về Postman trông như thế này:

```json
{
  "message": "Thêm sản phẩm thất bại",
  "error": {
    "index": 0,
    "code": 11000,
    "keyPattern": { "slug": 1 },
    "keyValue": { "slug": "tra-sua-matcha" }
  }
}
```

> 💡 **Vì sao JSON không có dòng chữ `E11000 ...`?** Vì nó nằm ở `error.message`, mà
> `message` của mọi đối tượng `Error` trong JavaScript là thuộc tính **không enumerable**
> — `res.json()` bỏ qua. Muốn đọc nó thì phải `console.log(error)` ở backend, hoặc bắt
> theo mã: `if (error.code === 11000) { ... }`.

### 4.3. Vì sao đây là bug chứ không phải "tính năng"?

Một quán trà sữa **hoàn toàn có thể** có hai sản phẩm cùng tên (hai size, hai chi nhánh,
hoặc đơn giản là admin gõ trùng). Model `Product` cũng **không** bắt `name` phải unique
— chỉ `slug` mới unique. Nghĩa là dự án cho phép trùng tên nhưng lại chặn ở tầng slug,
và thông báo cho admin chỉ vỏn vẹn *"Thêm sản phẩm thất bại"* — không nói trùng cái gì.

### 4.4. Ba cách khắc phục

| Cách | Slug sinh ra | Ưu | Nhược |
|---|---|---|---|
| **A. Hậu tố đếm** — dò database, gặp trùng thì thêm `-1`, `-2`… | `tra-sua-matcha`, `tra-sua-matcha-1` | Đẹp, dễ đọc | Tốn 1+ truy vấn; hai request cùng lúc vẫn có thể đụng nhau |
| **B. Hậu tố timestamp** — nối `Date.now()` dạng base36 | `tra-sua-matcha-mb3k7x1p` | Không cần truy vấn, gần như không bao giờ trùng | Slug xấu hơn |
| **C. Hậu tố ngẫu nhiên ngắn** — 4–6 ký tự random | `tra-sua-matcha-a7f2` | Ngắn gọn | Vẫn có xác suất trùng (rất nhỏ) |

Cách A đẹp nhất và là cách các CMS lớn (WordPress, Ghost) dùng. Ta sẽ làm cách A, có
**lưới an toàn bằng cách B** — chính là nội dung phần tiếp theo.

---

## 5. 🛠️ Tự tay làm — Topping có slug và không bao giờ trùng

> Mục tiêu phần này: cuối phần, bạn tạo được **hai topping cùng tên "Trân châu đường đen"**
> mà API vẫn trả `200 OK`, với slug lần lượt là `tran-chau-duong-den` và
> `tran-chau-duong-den-1`. Đồng thời `GET /api/toppings/tran-chau-duong-den` lấy được
> đúng topping.

Ở [Bài 07](07-crud-category.md) bạn đã có `models/topping.js`, `controllers/topping.js`,
`routes/topping.js` với đủ 5 thao tác CRUD. Bài này ta **thêm trường `slug`** vào bộ đó.

### Bước 1 — Thêm trường `slug` vào model Topping

Mở file bạn đã tạo ở Bài 05 và thêm khối `slug`:

```js
// yotea-be/src/models/topping.js  ← file BẠN tự tạo từ Bài 05, giờ sửa lại
// (đoạn dưới là code người học tự viết, dự án Yotea KHÔNG có file này)
import { Schema, model } from "mongoose";

const toppingSchema = new Schema(
  {
    name: { type: String, required: true },
    price: { type: Number, required: true },
    image: String,
    status: { type: Number, default: 0 },
    slug: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
    },
  },
  { timestamps: true }
);

toppingSchema.index({ "$**": "text" });

export default model("Topping", toppingSchema);
```

Bốn thuộc tính của `slug` khai báo **giống hệt** `yotea-be/src/models/product.js:35-40` —
ta đang bắt chước đúng cách dự án làm, chỉ khác là phần sinh slug sẽ làm cẩn thận hơn.

> ⚠️ Nếu collection `toppings` trong MongoDB **đã có sẵn dữ liệu từ bài 07** (chưa có
> `slug`), unique index sẽ build lỗi vì nhiều document cùng `slug = null`. Cách xử lý
> nhanh nhất khi đang học: mở **MongoDB Compass** → database `yotea` → xoá collection
> `toppings` → khởi động lại backend.

### Bước 2 — Viết hàm sinh slug dùng chung

Mở `yotea-be/src/controllers/topping.js` (file bạn tạo ở Bài 06). Thêm import và **hai
hàm helper** ngay dưới phần import:

```js
// yotea-be/src/controllers/topping.js  ← code BẠN tự viết thêm
import slugify from "slugify";
import Topping from "../models/topping";

// Bước 2a: sinh slug thô — có đủ 3 option mà dự án Yotea còn thiếu
const buildSlug = (name) =>
  slugify(name, { lower: true, locale: "vi", strict: true });

// Bước 2b: dò database, nếu trùng thì nối thêm hậu tố -1, -2, -3...
//   ignoreId: khi UPDATE, bỏ qua chính bản ghi đang sửa
const createUniqueSlug = async (name, ignoreId = null) => {
  const base = buildSlug(name);
  let slug = base;

  for (let i = 1; i <= 20; i++) {
    const filter = ignoreId ? { slug, _id: { $ne: ignoreId } } : { slug };
    const existed = await Topping.findOne(filter).exec();

    if (!existed) return slug;

    slug = `${base}-${i}`;
  }

  // lưới an toàn: quá 20 lần trùng thì dùng hậu tố thời gian
  return `${base}-${Date.now().toString(36)}`;
};
```

**Đọc từng phần:**

| Phần | Ý nghĩa |
|---|---|
| `{ lower: true, locale: "vi", strict: true }` | Ba option ở mục 2.3 — `đường đen` ra `duong-den` chứ không phải `djuong-djen` |
| `ignoreId = null` | Tham số mặc định ([Bài 03](03-kien-thuc-nen.md) mục 1.6). `create` không truyền, `update` truyền `_id` của chính nó |
| `{ slug, _id: { $ne: ignoreId } }` | *"có bản ghi nào slug này mà KHÔNG phải là tôi không?"* — nếu không loại trừ, sửa topping mà giữ nguyên tên sẽ tự đụng chính mình |
| `for (let i = 1; i <= 20; i++)` | Vòng lặp **có giới hạn**. Dùng `while (true)` sẽ treo server nếu logic sai |
| `Date.now().toString(36)` | Đổi mốc thời gian mili-giây sang hệ 36 → chuỗi ngắn như `mb3k7x1p` |

### Bước 3 — Dùng nó trong `create`

Sửa hàm `create` bạn viết ở Bài 07:

```js
// yotea-be/src/controllers/topping.js  ← code BẠN tự viết thêm
export const create = async (req, res) => {
  try {
    if (!req.body.name) {
      return res.status(400).json({ message: "Thiếu tên topping" });
    }

    req.body.slug = await createUniqueSlug(req.body.name);

    const topping = await new Topping(req.body).save();
    res.json(topping);
  } catch (error) {
    res.status(400).json({
      message: "Thêm topping thất bại",
      error,
    });
  }
};
```

Ba điểm khác biệt so với `controllers/product.js:35-47` — và cả ba đều là **cố ý**:

1. Việc sinh slug nằm **bên trong** `try` → không còn nguy cơ sập server (bug ở mục 3.1).
2. Có kiểm tra `req.body.name` và `return` ngay → thông báo lỗi rõ ràng cho admin.
3. `await createUniqueSlug(...)` thay vì `slugify(...)` → không còn E11000.

### Bước 4 — Dùng nó trong `update`

```js
// yotea-be/src/controllers/topping.js  ← code BẠN tự viết thêm
export const update = async (req, res) => {
  try {
    if (!req.body.name) {
      return res.status(400).json({ message: "Thiếu tên topping" });
    }

    const filter = { _id: req.params.id };
    const update = {
      ...req.body,
      slug: await createUniqueSlug(req.body.name, req.params.id),
    };
    const options = { new: true };

    const topping = await Topping.findOneAndUpdate(
      filter,
      update,
      options
    ).exec();
    res.json(topping);
  } catch (error) {
    res.status(400).json({
      message: "Cập nhật topping thất bại",
      error,
    });
  }
};
```

Chú ý tham số thứ hai `req.params.id` truyền vào `createUniqueSlug` — đây chính là
`ignoreId`. Không có nó, sửa giá của topping mà không đổi tên sẽ khiến slug bị đội lên
`-1`, `-2`, `-3`… sau mỗi lần bấm Lưu.

### Bước 5 — Cho phép đọc topping theo slug

Ở Bài 07 route `read` của bạn đang tra theo `:id`. Giờ Topping đã có slug, hãy đổi sang
`:slug` cho khớp cách dự án làm với product — sửa `yotea-be/src/routes/topping.js`:

```js
// yotea-be/src/routes/topping.js  ← code BẠN tự viết thêm
router.post("/toppings", create);
router.get("/toppings/:slug", read);     // ← đổi từ /toppings/:id
router.get("/toppings", list);
router.put("/toppings/:id", update);
router.delete("/toppings/:id", remove);
```

Chỉ `read` đổi sang slug thôi, còn `update`/`remove` vẫn dùng `_id` — **giống hệt** cách
`yotea-be/src/routes/product.js:9,11,13` làm. Lý do: slug có thể đổi khi admin sửa tên,
còn `_id` thì vĩnh viễn không đổi, nên dùng `_id` cho thao tác ghi mới an toàn.

Và trong controller, sửa lại filter (bài 07 bạn đang dùng `{ _id: req.params.id }`):

```js
// yotea-be/src/controllers/topping.js  ← code BẠN tự viết thêm
export const read = async (req, res) => {
  try {
    const topping = await Topping.findOne({ slug: req.params.slug })
      .select("-__v")
      .exec();

    if (!topping) {
      return res.status(404).json({ message: "Không tìm thấy topping" });
    }

    res.json(topping);
  } catch (error) {
    res.status(400).json({
      message: "Không tìm thấy topping",
      error,
    });
  }
};
```

> 💡 Phần `if (!topping) return res.status(404)` bạn đã viết từ Bài 07 — giữ nguyên nhé.
> Toàn bộ 6 hàm `read` của dự án Yotea **không có** bước này: không tìm thấy thì trả
> `200` kèm `null`, khiến frontend tưởng thành công rồi mới nổ ở dòng sau.

---

## 6. ✅ Kiểm chứng kết quả

```bash
# đứng tại thư mục yotea-be
npm start
```

> ⚠️ Route Topping của bạn hiện **chưa có middleware bảo vệ** — ai gọi cũng được. Việc
> khoá các route ghi bằng `requireSignin` / `isAuth` / `isAdmin` là nội dung
> [Bài 12](12-phan-quyen-middleware.md). Nên các request dưới đây **không cần** header
> `Authorization`.

### Test 1 — tạo topping đầu tiên

```
POST http://localhost:8080/api/toppings
Content-Type: application/json

{ "name": "Trân châu đường đen", "price": 10000, "status": 1 }
```

Phải nhận được `200 OK`:

```json
{
  "_id": "6650a1f2c4e8b91234abcd01",
  "name": "Trân châu đường đen",
  "price": 10000,
  "status": 1,
  "slug": "tran-chau-duong-den",
  "createdAt": "2026-08-15T09:12:00.000Z",
  "updatedAt": "2026-08-15T09:12:00.000Z",
  "__v": 0
}
```

Nhìn kỹ: `tran-chau-duong-den` — **`d` chứ không phải `dj`**. Đó là công của `locale: "vi"`.

### Test 2 — tạo topping TRÙNG TÊN (bài kiểm tra chính)

Gửi lại **y hệt** request trên. Trước khi có bài này, bạn sẽ nhận `400` + E11000.
Bây giờ phải là `200 OK`:

```json
{
  "_id": "6650a1f2c4e8b91234abcd02",
  "name": "Trân châu đường đen",
  "price": 10000,
  "status": 1,
  "slug": "tran-chau-duong-den-1"
}
```

Gửi lần thứ ba → `tran-chau-duong-den-2`. 🎉

### Test 3 — đọc theo slug

```
GET http://localhost:8080/api/toppings/tran-chau-duong-den
```

→ trả về topping thứ nhất. Đổi thành `.../tran-chau-duong-den-1` → trả về topping thứ hai.
Gõ một slug không tồn tại → `404` kèm `{ "message": "Không tìm thấy topping" }`.

### Test 4 — sửa mà không đổi tên

```
PUT http://localhost:8080/api/toppings/6650a1f2c4e8b91234abcd01
{ "name": "Trân châu đường đen", "price": 12000, "status": 1 }
```

Slug trả về phải **vẫn là** `tran-chau-duong-den`, không bị đội thành `-3`. Nếu bị đội,
bạn quên truyền `req.params.id` vào `createUniqueSlug` ở Bước 4.

---

## 7. Slug được dùng ở đâu bên Frontend

### 7.1. Route nhận slug

`yotea-fe/src/App.js:96-103`

```jsx
        {
          path: "san-pham/:slug",
          element: <ProductDetailPage />,
        },
        {
          path: "san-pham/:slug/page/:page",
          element: <ProductDetailPage />,
        },
```

### 7.2. Link trỏ tới slug

`yotea-fe/src/components/user/home/HomeProducts.js:104-108`

```jsx
              <Link
                to={`/san-pham/${item.slug}`}
                style={{ backgroundImage: `url(${item.image})` }}
                className="bg-cover pt-[100%] bg-center block"
              />
```

### 7.3. Trang chi tiết bóc slug ra rồi gọi API

`yotea-fe/src/pages/user/ProductDetailPage.js:64-70`

```jsx
  const { slug, page } = useParams();

  useEffect(() => {
    window.scrollTo(0, 0);

    const getProduct = async () => {
      const { data } = await get(slug);
```

### 7.4. Tầng API

`yotea-fe/src/api/product.js:49-52`

```js
export const get = (slug) => {
  const url = `/${DB_NAME}/${slug}/?_expand=categoryId`;
  return instance.get(url);
};
```

### 7.5. Chuỗi mắt xích đầy đủ

```
Người dùng bấm thẻ sản phẩm ở trang chủ
        │  HomeProducts.js:105 → <Link to="/san-pham/tra-sua-matcha">
        ▼
React Router khớp App.js:97  "san-pham/:slug"
        │
        ▼
ProductDetailPage.js:64  const { slug } = useParams()   → slug = "tra-sua-matcha"
        │
        ▼
api/product.js:50  GET http://localhost:8080/api/products/tra-sua-matcha/?_expand=categoryId
        │
        ▼
routes/product.js:9  router.get("/products/:slug", read)
        │
        ▼
controllers/product.js:74  const filter = { slug: req.params.slug }
        │
        ▼
MongoDB: db.products.findOne({ slug: "tra-sua-matcha" })
```

---

## 8. Resource nào có slug, resource nào không?

| Resource | Có `slug`? | Vị trí khai báo | Sinh từ trường | `read` tra theo |
|---|---|---|---|---|
| **Product** | ✅ | `yotea-be/src/models/product.js:35-40` | `name` | `slug` |
| **Category** | ✅ | `yotea-be/src/models/category.js:9-14` | `name` | `slug` |
| **News** | ✅ | `yotea-be/src/models/news.js:9-14` | **`title`** | `slug` |
| **CateNews** | ✅ | `yotea-be/src/models/cateNews.js:9-14` | `name` | `slug` |
| Slider (`Slide`) | ❌ | — | — | `_id` |
| Store | ❌ | — | — | `_id` |
| Order | ❌ | — | — | `_id` |
| OrderDetail | ❌ | — | — | — |
| Contact | ❌ | — | — | `_id` |
| User | ❌ | — | — | `_id` |
| Comment / Rating / FavoritesProduct | ❌ | — | — | — |

**Quy luật rất rõ:** cái gì **có trang riêng cho người dùng xem và cần Google index**
thì có slug. Cái gì **chỉ nội bộ / riêng tư / không ai search** thì không.

- `Product`, `Category`, `News`, `CateNews` → đều có route công khai trong `App.js`
  (`san-pham/:slug`, `danh-muc/:slug`, `bai-viet/:slug`, `tin-tuc/:slug`).
- `Order` → **đơn hàng của riêng một người**. Đưa `/don-hang/don-cua-nguyen-van-a` lên
  Google là thảm hoạ quyền riêng tư. Hơn nữa đơn hàng chẳng có "tên" nào để làm slug.
- `Contact` → phản hồi của khách, chỉ admin đọc.
- `User` → tuyệt đối không.
- `Slider`, `Store` → có tên nhưng **không có trang chi tiết riêng**; slider hiện trên
  trang chủ, cửa hàng hiện trong danh sách ở `StorePage`. Không cần URL riêng nên
  `yotea-be/src/routes/slider.js` và `routes/store.js` đều dùng `router.get(".../:id", read)`.

> ⚠️ **Chỗ này dự án làm chưa nhất quán:** 4 resource tra theo `slug`, 2 resource tra theo
> `_id`, và tên endpoint thì lúc số nhiều (`/api/products`) lúc số ít (`/api/category`,
> `/api/news`). Khi tự làm dự án, hãy chọn **một** quy ước và giữ đến cùng.

---

## 9. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `slugify: string argument expected` (server sập) | Gọi `slugify(undefined)` vì body thiếu `name` | Kiểm tra `if (!req.body.name) return res.status(400)...` **trước**, và đặt lời gọi trong `try` |
| `E11000 duplicate key error ... index: slug_1` | Hai bản ghi cùng slug | Dùng `createUniqueSlug` ở Bước 2 |
| `E11000 ... dup key: { slug: null }` | Vừa thêm `slug` unique vào collection **đã có dữ liệu cũ** | Xoá collection trong Compass, hoặc backfill slug cho từng document rồi khởi động lại |
| Slug ra `djuong-djen` thay vì `duong-den` | Thiếu `locale: "vi"` | `slugify(name, { lower: true, locale: "vi", strict: true })` |
| Slug ra `Tra-Sua-Matcha` (còn chữ hoa) | Thiếu `{ lower: true }` **và** model thiếu `lowercase: true` | Thêm `lower: true` ở controller |
| Sửa topping xong slug tự thành `-1`, `-2`, `-3`… | Quên truyền `ignoreId` khi update | `createUniqueSlug(req.body.name, req.params.id)` |
| `GET /api/toppings/xyz` trả `200` kèm `null` | `findOne` không thấy nhưng vẫn `res.json(null)` | Thêm `if (!topping) return res.status(404)...` |
| Đổi tên sản phẩm xong link cũ chết | Slug đổi theo tên, không có redirect | Lưu mảng `oldSlugs` rồi tra thêm ở `read` (xem Bài tập 3) |

---

## 10. 📝 Bài tập

**Bài 1.** Không chạy code, hãy dự đoán kết quả của các lời gọi sau với `slugify@1.6.5`,
rồi mở terminal ở `yotea-be` chạy `node -e "..."` để đối chiếu:

```js
slugify("Trà Sữa Đường Đen 50% Đá");
slugify("Trà Sữa Đường Đen 50% Đá", { lower: true });
slugify("Trà Sữa Đường Đen 50% Đá", { lower: true, locale: "vi" });
slugify("Trà Sữa Đường Đen 50% Đá", { lower: true, locale: "vi", strict: true });
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```
1. "Tra-Sua-DJuong-DJen-50percent-DJa"
2. "tra-sua-djuong-djen-50percent-dja"
3. "tra-sua-duong-den-50percent-da"
4. "tra-sua-duong-den-50percent-da"
```

Ba điều đáng chú ý:

- Dòng 1 là **kết quả thật** mà `controllers/product.js:36` đang tạo ra (trước khi
  `lowercase: true` của model hạ nó thành dòng 2).
- `locale: "vi"` là thứ **duy nhất** sửa được `DJ` → `d`. `lower` và `strict` đều không cứu được.
- `50percent` vẫn còn ở cả 4 dòng — vì `%` bị bảng ký tự đổi thành chữ `percent` **trước**
  khi `strict`/`remove` kịp xử lý. Muốn bỏ hẳn, phải làm sạch chuỗi trước:

```js
slugify("Trà Sữa Đường Đen 50% Đá".replace(/%/g, ""), {
  lower: true,
  locale: "vi",
  strict: true,
});
// → "tra-sua-duong-den-50-da"
```

</details>

**Bài 2.** Hàm `getById` ở `yotea-fe/src/api/product.js:54-57` **luôn** trả về `200` kèm
`null`, dù `id` truyền vào hoàn toàn đúng. Hãy giải thích tại sao, dựa vào những gì bạn
đã học trong bài này.

```js
export const getById = (id) => {
  const url = `/${DB_NAME}/${id}/?_expand=categoryId`;
  return instance.get(url);
};
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Lần theo đúng chuỗi mắt xích ở mục 7.5:

1. `getById("6650a1f2c4e8b91234abcd01")` bắn ra
   `GET /api/products/6650a1f2c4e8b91234abcd01/?_expand=categoryId`.
2. Backend chỉ có **một** route khớp — `yotea-be/src/routes/product.js:9`:
   `router.get("/products/:slug", read)`. Express không phân biệt được đâu là id đâu là
   slug, nó chỉ thấy "một đoạn chuỗi sau `/products/`" và gán vào `req.params.slug`.
3. `controllers/product.js:74` dựng filter:
   `const filter = { slug: "6650a1f2c4e8b91234abcd01" };`
4. MongoDB đi tìm document có **trường `slug`** bằng chuỗi đó. Không bao giờ có, vì slug
   luôn là chữ-thường-có-gạch-ngang, còn đây là ObjectId hex.
5. `findOne` trả `null`, và vì `read` không kiểm tra `null` (mục 5 Bước 5), nó chạy thẳng
   `res.json(null)` với status **200**.

→ **`getById` là code chết.** Cách sửa gọn nhất là ở backend, cho `read` nhận cả hai:

```js
// code người học tự viết thêm — dự án chưa có
import { Types } from "mongoose";

const key = Types.ObjectId.isValid(req.params.slug)
  ? { _id: req.params.slug }
  : { slug: req.params.slug };

const product = await Product.findOne(key).select("-__v").populate(populate).exec();
```

Đây cũng là một mẹo hay cho dự án riêng của bạn: cho phép tra **cả slug lẫn id** trên
cùng một endpoint.

</details>

**Bài 3.** *(khó)* Admin sửa tên sản phẩm từ "Trà sữa matcha" thành "Trà sữa matcha đá xay".
Slug đổi từ `tra-sua-matcha` thành `tra-sua-matcha-da-xay`. Mọi link cũ mà khách đã lưu,
đã share lên Facebook, hoặc Google đã index đều **404**. Hãy thiết kế giải pháp giữ link
cũ sống, và viết code cho model + hàm `read` của **Topping** (không sửa file dự án).

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Ý tưởng: mỗi lần slug đổi, **cất slug cũ vào một mảng** rồi cho `read` tra cả mảng đó.

**Bước 1 — thêm trường vào model** (`yotea-be/src/models/topping.js`, code bạn tự viết):

```js
    slug: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
    },
    oldSlugs: {
      type: [String],
      default: [],
      index: true,
    },
```

**Bước 2 — trong `update`, cất slug cũ lại:**

```js
// yotea-be/src/controllers/topping.js — code bạn tự viết thêm
export const update = async (req, res) => {
  try {
    if (!req.body.name) {
      return res.status(400).json({ message: "Thiếu tên topping" });
    }

    const current = await Topping.findById(req.params.id).exec();
    if (!current) {
      return res.status(404).json({ message: "Không tìm thấy topping" });
    }

    const newSlug = await createUniqueSlug(req.body.name, req.params.id);

    const update = {
      ...req.body,
      slug: newSlug,
    };

    // slug thật sự thay đổi → nhớ lại slug cũ
    if (newSlug !== current.slug) {
      update.$addToSet = { oldSlugs: current.slug };
    }

    const topping = await Topping.findOneAndUpdate(
      { _id: req.params.id },
      update,
      { new: true }
    ).exec();

    res.json(topping);
  } catch (error) {
    res.status(400).json({ message: "Cập nhật topping thất bại", error });
  }
};
```

**Bước 3 — `read` tra cả slug mới lẫn slug cũ:**

```js
// yotea-be/src/controllers/topping.js — code bạn tự viết thêm
export const read = async (req, res) => {
  try {
    const { slug } = req.params;

    const topping = await Topping.findOne({
      $or: [{ slug }, { oldSlugs: slug }],
    })
      .select("-__v")
      .exec();

    if (!topping) {
      return res.status(404).json({ message: "Không tìm thấy topping" });
    }

    // slug cũ → báo cho frontend biết địa chỉ mới
    if (topping.slug !== slug) {
      return res.status(301).set("Location", `/api/toppings/${topping.slug}`).json(topping);
    }

    res.json(topping);
  } catch (error) {
    res.status(400).json({ message: "Không tìm thấy topping", error });
  }
};
```

Vài điểm đáng học:

- `$or` cho phép match **một trong hai** điều kiện.
- `oldSlugs: slug` — với trường kiểu mảng, MongoDB tự hiểu là *"mảng này có chứa phần
  tử bằng giá trị đó không"*. Không cần viết `$in`.
- `$addToSet` chỉ thêm khi chưa có → không sinh trùng lặp trong mảng.
- Mã **301 Moved Permanently** là tín hiệu chuẩn để Google chuyển toàn bộ điểm SEO của
  URL cũ sang URL mới. Đây chính là cách WordPress, Shopify… xử lý.

Cách này chính xác là thứ Yotea còn thiếu — bạn vừa làm tốt hơn dự án gốc.

</details>

---

## 📌 Tóm tắt

- **Slug** là chuỗi thân thiện thay cho `_id` trên URL: dễ đọc, dễ share, và Google
  hiểu được nội dung trang.
- `slugify(name)` **không truyền option** — như dự án đang làm ở
  `controllers/product.js:36` và `:224` — cho ra slug **còn chữ hoa** và biến `đ` thành
  `dj` (`Trà sữa đường đen` → `Tra-sua-djuong-djen`).
- Phần chữ hoa được **cứu bởi `lowercase: true`** khai báo ở model (`models/product.js:35-40`),
  một setter chạy cả khi `.save()` lẫn khi `findOneAndUpdate`. Nhưng đây là phụ thuộc
  ngầm dễ vỡ — nên sửa đúng ngay tại controller bằng
  `{ lower: true, locale: "vi", strict: true }`.
- `slug` khai báo `unique: true` để `findOne({ slug })` luôn cho kết quả duy nhất. Cái giá
  phải trả: **hai sản phẩm trùng tên → lỗi `E11000 duplicate key`**, và admin chỉ nhận
  được thông báo mơ hồ *"Thêm sản phẩm thất bại"*.
- Cách khắc phục: dò database rồi nối hậu tố `-1`, `-2`… (đẹp), hoặc nối
  `Date.now().toString(36)` (chắc ăn). Nhớ **loại trừ chính bản ghi đang sửa** khi update.
- Slug chỉ dành cho resource **công khai, cần SEO**: `Product`, `Category`, `News`,
  `CateNews`. `Order`, `Contact`, `User`, `Slider`, `Store` không có — vì riêng tư hoặc
  không có trang chi tiết riêng.
- Bạn đã thêm slug tự-sinh-không-trùng cho Topping, và tạo được hai topping cùng tên mà
  API vẫn `200 OK`.

**Từ khoá tra cứu thêm:** `slugify npm options`, `SEO friendly URL`, `mongoose unique index`,
`E11000 duplicate key error`, `mongoose lowercase setter`, `301 redirect old slug`

➡️ **Bài tiếp theo:** [09 — Bộ lọc kiểu json-server: `_sort`, `_limit`, `_expand`, `_like`, `q`…](09-bo-loc-query.md)
— ta sẽ mổ xẻ hàm `list()` dài 40 dòng được copy-paste **13 lần** trong dự án, rồi thêm bộ
lọc query cho chính API Topping của bạn.
