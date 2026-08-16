# Bài 05 — Mongoose Model: Schema, kiểu dữ liệu, timestamps, index

> **Phần 1 · Backend với Express & Mongoose** — Thời lượng ước tính: **~70 phút**
> ⬅️ Bài trước: [04 — Mổ xẻ `app.js`](04-express-va-appjs.md) · Bài sau: [06 — Vòng đời một request](06-vong-doi-mot-request.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Phân biệt rành mạch ba khái niệm **Schema — Model — Document**.
- Đọc hiểu **từng dòng** của `models/category.js` và `models/product.js`.
- Thuộc bảng option của một field: `type`, `required`, `default`, `unique`, `trim`, `lowercase`, `min`, `ref`.
- Biết `{ timestamps: true }` sinh ra cái gì và vì sao gần như mọi màn hình của Yotea sống nhờ nó.
- Hiểu `schema.index({ "$**": "text" })` phục vụ tìm kiếm `?q=...` ra sao, và **yếu ở đâu với tiếng Việt**.
- **Tự tay viết file model đầu tiên của mình**: `yotea-be/src/models/topping.js`.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 04](04-express-va-appjs.md): bạn đã biết `app.js` gọi `mongoose.connect(...)` nối tới `mongodb://localhost:27017/yotea`.
- MongoDB đang chạy, và đã cài **MongoDB Compass** để nhìn tận mắt dữ liệu.
- Nhớ lại mục 4 của [Bài 03](03-kien-thuc-nen.md): collection, document, `_id` kiểu ObjectId.

> 💡 **Mạch thực hành xuyên suốt phần backend:** ở bài trước bạn mới chỉ *đọc* `app.js`. Từ bài này,
> song song với việc soi code có sẵn, bạn sẽ **tự xây một chức năng hoàn toàn mới cho Yotea: Topping**
> (trân châu, thạch, pudding…). Bài 05 làm phần móng — file model. Bài 06 dựng route + controller.
> Bài 07 làm đủ CRUD. Cứ mỗi bài thêm một tầng, tới cuối phần backend bạn sẽ có một chức năng do
> **chính bạn viết từ số 0**, đứng ngang hàng với Category hay Product của tác giả gốc.

---

## 1. Từ MongoDB "tự do" tới Mongoose "có kỷ luật"

### 1.1. Vấn đề: MongoDB không ép bạn phải đúng

MongoDB thuần **không có cấu trúc bắt buộc**. Ba document lệch nhau hoàn toàn vẫn nằm chung một collection:

```js
{ name: "Trà sữa trân châu", price: 35000 }
{ name: "Trà đào cam sả",    price: "bốn mươi nghìn" }   // giá là CHUỖI
{ ten:  "Matcha latte" }                                  // sai tên trường, thiếu giá
```

Hôm sau bạn viết `product.price * quantity` để tính tiền: dòng hai ra `NaN`, dòng ba ra `undefined`,
trang giỏ hàng vỡ trận. Tệ nhất là lỗi chỉ nổ **khi khách đã bấm thanh toán**, chứ không nổ lúc lưu.

**Mongoose** chen vào giữa Node.js và MongoDB. Trước khi cho ghi bất cứ thứ gì, nó soi lại bản khai
báo cấu trúc bạn viết sẵn — gọi là **Schema**. Sai kiểu? Chặn. Thiếu trường bắt buộc? Chặn. Thừa
trường lạ? Âm thầm cắt bỏ.

### 1.2. Ba khái niệm phải phân biệt được

Hình dung một xưởng làm bánh:

| Khái niệm | Trong xưởng bánh | Trong Mongoose | Trong Yotea |
|---|---|---|---|
| **Schema** | Bản vẽ khuôn | Bản khai báo trường + ràng buộc | `const categorySchema = new Schema({...})` |
| **Model** | **Cái khuôn** cầm lên đúc được | Lớp sinh từ Schema, có sẵn `.find()`, `.save()` | `model("Category", categorySchema)` |
| **Document** | **Cái bánh** đúc ra | Một bản ghi cụ thể | `{ _id: "665...", name: "Trà sữa" }` |
| **Collection** | Khay đựng bánh | Nơi gom document cùng loại | collection `categories` |

Ba câu chốt: **Schema là bản vẽ** (chỉ mô tả) · **Model là khuôn đúc** (từ nó mới gọi được
`Category.find()`) · **Document là sản phẩm đúc ra** (thứ thật sự nằm trong MongoDB).

> 📖 **Thuật ngữ:** *collection* — tương đương "bảng" bên SQL. Mongoose **tự đặt tên collection** bằng
> cách lấy tên model, hạ chữ thường và thêm "s": `model("Category", ...)` → `categories`. Nhớ quy tắc
> này, lát nữa bạn sẽ cần.

### 1.3. Bộ khung chung của MỌI file model trong dự án

Cả 14 file trong `yotea-be/src/models/` đều theo đúng bộ xương 4 phần:

```
1) import { Schema, model } from "mongoose";                              ← lấy đồ nghề
2) const xSchema = new Schema({ ...trường... }, { timestamps: true });    ← vẽ bản vẽ
3) xSchema.index({ "$**": "text" });                                      ← chỉ mục tìm kiếm
4) export default model("X", xSchema);                                    ← đúc khuôn, xuất ra
```

Nắm bộ xương này thì mở file model nào bạn cũng đọc được trong 30 giây.

---

## 2. Soi code thật — `category.js`, file model đơn giản nhất

`yotea-be/src/models/category.js:1-22`

```js
import { Schema, model } from "mongoose";

const categorySchema = new Schema(
  {
    name: {
      type: String,
      required: true,
    },
    slug: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
    },
    image: String,
  },
  { timestamps: true }
);

categorySchema.index({ "$**": "text" });

export default model("Category", categorySchema);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import { Schema, model } from "mongoose"` | Lấy hai công cụ: `Schema` để vẽ bản vẽ, `model` để đúc khuôn |
| 3 | `new Schema(` | Hàm này nhận **2 tham số**: object mô tả các trường, và object tuỳ chọn |
| 5-8 | `name: { type: String, required: true }` | Tên danh mục, kiểu chuỗi, **không được để trống** |
| 9-14 | `slug: { ... }` | Chuỗi, bắt buộc, **không được trùng** (`unique`), luôn bị **hạ về chữ thường** (`lowercase`) |
| 15 | `image: String` | **Khai báo rút gọn** — chỉ nêu kiểu, không ràng buộc. Tương đương `image: { type: String }` |
| 17 | `{ timestamps: true }` | Tham số **thứ hai** — bật hai trường thời gian tự động (mục 5) |
| 20 | `categorySchema.index({ "$**": "text" })` | Tạo **text index** phục vụ tìm kiếm (mục 6) |
| 22 | `export default model("Category", categorySchema)` | Đúc khuôn tên `Category` để controller `import` được |

Ba điều đáng để ý: `name` **không** `unique` (tạo hai danh mục trùng tên vẫn được, miễn slug khác);
dòng 15 dùng cú pháp rút gọn trong khi mọi trường khác dùng object đầy đủ — hai cách **hoàn toàn tương
đương**; `image` không `required` nên frontend phải tự phòng trường hợp `item.image` là `undefined`.

---

## 3. Soi code thật — `product.js`, file đầy đủ hơn hẳn

Model `Product` là trái tim của Yotea, gom gần đủ mọi kỹ thuật bài này cần dạy.

`yotea-be/src/models/product.js:1-45`

```js
import { Schema, model, ObjectId } from "mongoose";

const productSchema = new Schema({
    name: {
        type: String,
        trim: true,
        required: true,
    },
    image: {
        type: String,
        required: true,
    },
    price: {
        type: Number,
        required: true,
    },
    description: String,
    status: {
        type: Number,
        default: 0,
    },
    view: {
        type: Number,
        default: 0,
    },
    favorites: {
        type: Number,
        default: 0,
    },
    categoryId: {
        type: ObjectId,
        ref: "Category",
        required: true,
    },
    slug: {
        type: String,
        required: true,
        unique: true,
        lowercase: true,
    }
}, { timestamps: true });

productSchema.index({'$**': 'text'});

export default model("Product", productSchema);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 1 | `import { ..., ObjectId }` | Lấy thêm kiểu `ObjectId` để khai báo quan hệ |
| 4-8 | `name: { type: String, trim: true, required: true }` | `trim` **tự cắt khoảng trắng thừa** hai đầu trước khi lưu |
| 9-12 | `image` | URL ảnh (upload lên Cloudinary). Bắt buộc |
| 13-16 | `price: { type: Number, required: true }` | Giá bán kiểu **số**. Gửi chuỗi `"35000"` Mongoose tự đổi sang `35000` |
| 17 | `description: String` | Khai báo rút gọn, **không bắt buộc** |
| 18-21 | `status: { type: Number, default: 0 }` | Trạng thái hiển thị; không gửi thì mặc định `0` |
| 22-25 | `view` | Bộ đếm lượt xem, dùng cho khối "sản phẩm xem nhiều" |
| 26-29 | `favorites` | Bộ đếm lượt yêu thích |
| 30-34 | `categoryId: { type: ObjectId, ref: "Category", required: true }` | **Quan hệ**: chứa `_id` của một document bên `categories` |
| 35-40 | `slug` | Chuỗi, bắt buộc, duy nhất, luôn chữ thường |
| 41 | `}, { timestamps: true });` | Đóng danh sách trường + bật `createdAt`/`updatedAt` |
| 43 | `productSchema.index({'$**': 'text'})` | Text index để tìm kiếm sản phẩm |
| 45 | `export default model("Product", productSchema)` | Collection thật sẽ tên là `products` |

Ý nghĩa `status` lấy từ chính giao diện admin (`yotea-fe/src/pages/admin/product/AddProductPage.js:196-198`):

```jsx
<option value="">-- Chọn trạng thái sản phẩm --</option>
<option value={0}>Ẩn</option>
<option value={1}>Hiển thị</option>
```

→ **`0` = Ẩn, `1` = Hiển thị.**

> 💡 **Mẹo đọc code người khác:** gặp một trường `Number` không hiểu giá trị nghĩa là gì, hãy đi tìm
> cái `<select>` tương ứng bên frontend. Nhãn hiển thị cho người dùng chính là lời giải thích đầy đủ
> nhất mà tác giả để lại.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `product.js:13-16` khai báo `price` là `Number, required` nhưng
> **không có `min: 0`**. Admin gõ nhầm `-50000` thì Mongoose vẫn lưu ngon lành, và giỏ hàng cộng ra
> tổng tiền **âm**. Đúng chuẩn chỉ tốn một chữ:
> ```js
> price: { type: Number, required: true, min: 0 },   // ← nên có
> ```
> `view` và `favorites` là bộ đếm, cũng không bao giờ được âm, cũng thiếu `min: 0`. Đây là điển hình
> của "validate ở frontend thì có, ở backend thì không" — mà kẻ tấn công thì gọi thẳng API, chẳng thèm
> mở giao diện. Bàn kỹ ở [Bài 33](33-ra-soat-bao-mat.md).

---

## 4. Bảng tra cứu: các option của một field

Bảng này bạn sẽ mở lại nhiều lần. Cột cuối chỉ đúng chỗ trong dự án đang dùng option đó.

| Option | Giá trị | Tác dụng | Ví dụ có thật trong Yotea |
|---|---|---|---|
| `type` | Kiểu dữ liệu | Khai báo kiểu. **Bắt buộc** nếu dùng cú pháp object | `type: String` — `category.js:6` |
| `required` | `true` | Không có giá trị thì **từ chối lưu**, ném `ValidationError` | `required: true` — `product.js:15` |
| `default` | Giá trị bất kỳ | Client không gửi thì Mongoose tự điền | `default: 0` — `product.js:20` |
| `unique` | `true` | Tạo **unique index** trong MongoDB → trùng là lỗi `E11000` | `unique: true` — `category.js:12` |
| `trim` | `true` | Tự cắt khoảng trắng đầu/cuối chuỗi | `trim: true` — `product.js:6` |
| `lowercase` | `true` | Tự hạ chuỗi về chữ thường | `lowercase: true` — `product.js:39` |
| `uppercase` | `true` | Ngược lại — tự nâng lên chữ hoa | `uppercase: true` — `voucher.js:8` |
| `min` / `max` | Số hoặc Date | Giới hạn giá trị — **chỉ áp dụng cho `Number` và `Date`** | `min: 4` — `user.js:14` (⚠️ dùng sai) |
| `minlength` / `maxlength` | Số | Giới hạn **độ dài chuỗi** — đây mới là thứ dùng cho `String` | Dự án **không dùng chỗ nào** |
| `ref` | Tên model | Trường này trỏ tới model nào → mở khoá `.populate()` | `ref: "Category"` — `product.js:32` |

### 4.1. Cạm bẫy `min` với chuỗi

`yotea-be/src/models/user.js:11-15`

```js
    password: {
      type: String,
      required: true,
      min: 4,
    },
```

Tác giả **có ý** bắt mật khẩu dài tối thiểu 4 ký tự. Nhưng `min` chỉ có nghĩa với `Number` và `Date`;
với `String` nó **bị bỏ qua hoàn toàn, không báo lỗi gì**.

> ⚠️ **Kết quả: backend Yotea không hề kiểm tra độ dài mật khẩu.** Đăng ký với mật khẩu `"a"` vẫn lọt.
> Muốn đúng ý phải viết `minlength: 4`. Bài học: **option sai tên hoặc sai kiểu thì Mongoose im lặng
> bỏ qua.** Đây là loại lỗi nguy hiểm nhất — code trông như đang bảo vệ bạn, mà thật ra không.

Về cú pháp, nhớ hai cách khai báo tương đương nhau — hễ cần thêm bất kỳ option nào, bạn **buộc phải**
chuyển sang dạng object:

```js
image: String                              // rút gọn
image: { type: String }                    // đầy đủ — y hệt dòng trên
image: { type: String, required: true }    // đầy đủ + có ràng buộc
```

---

## 5. Kiểu dữ liệu và `{ timestamps: true }`

### 5.1. Năm kiểu mà Yotea dùng

| Kiểu | Lưu cái gì | Nơi dùng trong dự án |
|---|---|---|
| `String` | Chuỗi văn bản | `name`, `slug`, `image`, `description`, `email`, `phone` |
| `Number` | Số | `price`, `status`, `view`, `favorites`, `role`, `active`, `ice`, `sugar` |
| `Date` | Mốc thời gian | `timeStart`/`timeEnd` của `voucher.js`; và `createdAt`/`updatedAt` do timestamps sinh |
| `Array` | Mảng "lỏng", không định nghĩa phần tử | `voucher` của `order.js`; `user_ids` của `voucher.js` |
| `ObjectId` | `_id` của document khác → **quan hệ** | `categoryId`, `productId`, `userId`, `orderId`, `store` |

`yotea-be/src/models/voucher.js:26-33` là chỗ duy nhất dùng `Date` do người viết tự khai:

```js
    timeStart: {
        type: Date,
        required: true,
    },
    timeEnd: {
        type: Date,
        required: true,
    },
```

Kiểu `Date` cho phép truy vấn "voucher nào còn hiệu lực hôm nay" bằng `$lte` / `$gte`.

> ⚠️ Đối chiếu ngay: `yotea-be/src/models/store.js` lại lưu giờ mở/đóng cửa chi nhánh
> (`timeStart`/`timeEnd`) kiểu **`String`**, ví dụ `"07:00"`. Hệ quả: **không thể truy vấn "cửa hàng
> nào đang mở cửa lúc này"** — phải tải hết về rồi tự so chuỗi. Cùng dự án, cùng khái niệm "thời gian",
> hai kiểu khác nhau: ví dụ rất tốt về việc **chọn sai kiểu dữ liệu làm chết tính năng tương lai**.

Còn `ObjectId` + `ref` là cột sống của quan hệ — `yotea-be/src/models/product.js:30-34`:

```js
    categoryId: {
        type: ObjectId,
        ref: "Category",
        required: true,
    },
```

Đọc là: *"trường `categoryId` chứa `_id` của một document bên model `Category`, bắt buộc phải có."*
Nhờ `ref` mà sau này gọi được `.populate("categoryId")` để "nở" cái id 24 ký tự thành nguyên object
danh mục — nội dung của [Bài 10](10-quan-he-va-populate.md).

### 5.2. Vì sao dự án dùng `Number` cho `status` / `role` / `active`?

`yotea-be/src/models/user.js:54-61`

```js
    role: {
      type: Number,
      default: 0,
    },
    active: {
      type: Number,
      default: 1,
    },
```

`role` chỉ có hai giá trị (0 = Khách hàng, 1 = Admin), `active` cũng hai (0 = Khóa, 1 = Kích hoạt).
Đây rõ ràng là **hai biến boolean được lưu dưới dạng số**. Còn `status` của đơn hàng thì có tận 5
giá trị — `yotea-be/src/models/order.js:35-42`:

```js
    status: {
        type: Number,
        default: 0
    },
    voucher: {
        type: Array,
        default: []
    },
```

Bảng ý nghĩa (đối chiếu từ `yotea-fe/src/components/admin/OrderList.js` và `.../CartDetailPage.js`):

| Model.Trường | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| `Product.status` · `News.status` · `Slide.status` | Ẩn | Hiển thị | — | — | — |
| `Order.status` | Chờ xác nhận | Đã xác nhận | Đang giao hàng | Đã giao hàng | Đã hủy |
| `User.role` | Khách hàng | Admin | — | — | — |
| `User.active` | Khóa | Kích hoạt | — | — | — |

**Ưu điểm:** gọn, dễ sắp xếp (`_sort=status`); mở rộng dễ (mai thêm trạng thái thứ 6 chỉ cần thêm số
`5`); và vì `0` là falsy trong JS nên frontend viết được rất ngắn: `{item.role ? "Admin" : "Khách hàng"}`.

**Nhược điểm — nặng hơn ưu điểm:**

1. **Đọc code không hiểu gì.** Thấy `status === 2` phải đi lục frontend mới biết là "Đang giao hàng".
2. **Không ràng buộc giá trị.** Ai gửi `status: 99` lên Mongoose vẫn lưu — vì `99` đúng là `Number` hợp lệ. Giao diện hiển thị chuỗi rỗng, không ai biết vì sao.
3. **Trường chỉ 2 giá trị thì `Boolean` đúng hơn hẳn:** `active: { type: Boolean, default: true }` đọc phát hiểu ngay.
4. Mongoose có sẵn `enum` để khoá danh sách giá trị — dự án không dùng chỗ nào.

Cách chuẩn hơn nếu bạn tự viết (**đoạn này bạn tự viết, dự án KHÔNG có**):

```js
status: { type: Number, enum: [0, 1, 2, 3, 4], default: 0 },
active: { type: Boolean, default: true },
```

> ⚠️ Nhưng **đừng vội đi sửa dự án**. Frontend đang so `item.active ? ... : ...` và gửi `value={0}` /
> `value={1}` từ các thẻ `<option>`. Đổi kiểu ở model mà không đổi frontend là **gãy toàn hệ thống**.
> Refactor an toàn để dành cho [Bài 34](34-refactor-du-an.md).

### 5.3. `{ timestamps: true }` — hai trường miễn phí

Tham số thứ hai của `new Schema(...)` bảo Mongoose: *"tự thêm `createdAt` và `updatedAt` kiểu `Date`
vào mọi document, và tự cập nhật giùm tôi."*

| Trường | Được ghi khi nào | Sau đó |
|---|---|---|
| `createdAt` | Lần đầu document được tạo | **Không bao giờ đổi nữa** |
| `updatedAt` | Lần tạo, và **mỗi lần** `save()` / `findOneAndUpdate()` | Luôn là mốc sửa gần nhất |

**Chứng minh bằng dữ liệu thật.** Gọi `POST /api/category/:userId` với body vỏn vẹn
`{ "name": "Trà trái cây", "slug": "tra-trai-cay" }`, rồi mở Compass xem collection `categories`:

```json
{
  "_id": { "$oid": "6650a1f2c4e8b91234abcd01" },
  "name": "Trà trái cây",
  "slug": "tra-trai-cay",
  "createdAt": { "$date": "2026-08-15T09:12:03.517Z" },
  "updatedAt": { "$date": "2026-08-15T09:12:03.517Z" },
  "__v": 0
}
```

Bạn **chỉ gửi 2 trường**, document có tới **6**. Ba trường thêm miễn phí: `_id` (MongoDB tự sinh),
`createdAt`/`updatedAt` (do timestamps — lúc mới tạo hai mốc **bằng nhau y hệt**), và `__v` — bộ đếm
phiên bản của Mongoose, gần như không ai dùng. Chính vì thế `yotea-be/src/controllers/auth.js:47-49`
phải cố tình loại bỏ nó trước khi trả về client (đoạn destructuring bạn đã gặp ở [Bài 03](03-kien-thuc-nen.md)).

Gọi tiếp `PUT` sửa tên, xem lại sẽ thấy:

```json
  "createdAt": { "$date": "2026-08-15T09:12:03.517Z" },   ← GIỮ NGUYÊN
  "updatedAt": { "$date": "2026-08-15T14:38:41.902Z" },   ← ĐÃ NHẢY
```

**14/14 model của dự án đều bật `timestamps`**, và frontend sống dựa vào nó: trang thực đơn xếp "mới
nhất" bằng `?_sort=createdAt&_order=desc`; trang quản trị đơn hàng hiển thị "Đã xác nhận lúc …" bằng
`formatDate(order.updatedAt)`; danh sách bình luận cũng xếp theo `createdAt`. Thiếu một dòng này ở
model `Order` là **cả màn hình lịch sử đơn hàng không có ngày tháng để hiển thị**. Vì vậy hãy bật nó
cho **mọi** model bạn viết, kể cả khi chưa nghĩ ra dùng làm gì.

---

## 6. `schema.index({ "$**": "text" })` — text index

Dòng cuối trong bộ xương model. Cả **14/14 model** đều có — `yotea-be/src/models/product.js:43`:

```js
productSchema.index({'$**': 'text'});
```

### 6.1. Index là gì, text index khác gì?

Nghĩ tới quyển từ điển: không có bảng tra ở gáy sách, muốn tìm từ "trà" bạn phải lật từng trang. Đó là
cách MongoDB làm **khi không có index** (*collection scan*). **Index** chính là bảng tra đó — nhanh khi
**đọc**, nhưng mỗi lần **ghi** phải cập nhật thêm, và tốn bộ nhớ.

- Index **thường** (`unique: true` cũng tạo ra một cái) giúp tìm **khớp chính xác**: `slug === "tra-sua-tran-chau"`.
- Index **text** tách chuỗi thành **từng từ** rồi lập chỉ mục theo từ → tìm được "document nào **chứa** từ *trà*".

### 6.2. Vì sao là wildcard `'$**'`?

`$**` nghĩa là **"mọi trường kiểu chuỗi trong document, kể cả trường lồng bên trong"**. Cách thông
thường phải liệt kê tên trường (**đoạn dưới bạn tự viết để so sánh, dự án KHÔNG viết kiểu này**):

```js
productSchema.index({ name: "text", description: "text" });
```

Dùng `$**` thì khỏi liệt kê, thêm trường mới cũng tự được index. Tiện, nhưng **thô**: nó index cả những
trường bạn chẳng bao giờ muốn tìm. Lưu ý quan trọng: **MongoDB chỉ cho phép DUY NHẤT một text index
trên mỗi collection** — dự án đã "xài" suất đó cho wildcard, nên không thể thêm text index thứ hai.

### 6.3. Nó phục vụ chức năng nào? — ô tìm kiếm `?q=...`

`yotea-be/src/controllers/product.js:154-155`

```js
    } else if (item === "q" && query["q"]) {
      filter["$text"] = { $search: `"${query["q"]}"` };
```

Luồng đầy đủ:

```
Người dùng gõ "trà đào" vào ô tìm kiếm trên header
        ↓
Frontend gọi  GET /api/products?q=trà đào
        ↓
Controller thấy key "q"  →  filter.$text = { $search: "\"trà đào\"" }
        ↓
Product.find(filter)  →  MongoDB dùng TEXT INDEX để tra
        ↓
Trả về danh sách sản phẩm khớp
```

Nếu xoá dòng `productSchema.index({'$**': 'text'})`, truy vấn `$text` sẽ ném lỗi
`text index required for $text query` và chức năng tìm kiếm chết ngay. Cơ chế bộ lọc query này là nội
dung chi tiết của [Bài 09](09-bo-loc-query.md). Chú ý cặp nháy kép trong `` `"${query["q"]}"` ``: bọc
từ khoá trong `"..."` là cú pháp **cụm từ chính xác** của MongoDB — đòi cả cụm "trà đào", chứ không
phải "trà" HOẶC "đào".

### 6.4. Hạn chế — tiếng Việt là nạn nhân lớn nhất

> ⚠️ **Ba điểm yếu bạn phải biết:**
>
> 1. **Không hỗ trợ tiếng Việt.** Ngôn ngữ mặc định của text index là `english`; MongoDB **không có bộ
>    phân tích tiếng Việt** nên chỉ tách từ theo khoảng trắng. Hệ quả rất thật: gõ **"tra sua"** (không
>    dấu) sẽ **không** ra "Trà sữa", vì với MongoDB đó là hai từ khác nhau hoàn toàn. Người Việt tìm
>    kiếm thì phần lớn gõ không dấu — nên trải nghiệm tìm kiếm của Yotea thực tế khá tệ.
> 2. **Chỉ khớp trọn từ.** Tìm "sữa" ra "Trà sữa trân châu"; tìm "sư" thì không ra gì. Muốn khớp chuỗi
>    con phải dùng `$regex` — chậm hơn nhưng linh hoạt hơn.
> 3. **Index thừa gây lãng phí.** `$**` bật cho cả 14 model, kể cả `Rating`, `FavoritesProduct`,
>    `OrderDetail` — những model gần như **chỉ chứa số và ObjectId**, chẳng có chuỗi nào để index. Vẫn
>    tốn RAM, tốn ổ đĩa, làm mọi thao tác ghi chậm đi. Riêng `user.js:96` còn tệ hơn: wildcard index
>    đánh chỉ mục **cả trường `password`**. Về nguyên tắc bảo mật, mật khẩu không nên nằm trong bất kỳ
>    chỉ mục tìm kiếm nào.
>
> **Cách đúng cho tiếng Việt:** lưu thêm trường `searchName` đã bỏ dấu (`"tra sua tran chau"`) rồi
> index/regex trên đó; hoặc dùng công cụ chuyên tìm kiếm như Atlas Search / Elasticsearch.

---

## 7. Toàn cảnh 14 model của Yotea

| # | File model | Model đăng ký | Collection | Mô tả một câu |
|---|---|---|---|---|
| 1 | `category.js` | `Category` | `categories` | Danh mục sản phẩm (Trà sữa, Trà trái cây…) — chỉ có `name`, `slug`, `image` |
| 2 | `product.js` | `Product` | `products` | Sản phẩm bán ra: giá, ảnh, bộ đếm `view`/`favorites`, ref tới `Category` |
| 3 | `cateNews.js` | `CateNews` | `catenews` | Danh mục tin tức — chỉ có `name` (unique) và `slug` |
| 4 | `news.js` | `News` | `news` | Bài viết tin tức: tiêu đề, ảnh, mô tả, nội dung HTML, ref tới `CateNews` |
| 5 | `slider.js` | **`Slide`** | `slides` | Banner quay vòng trang chủ: tiêu đề, ảnh, link đích, trạng thái |
| 6 | `store.js` | `Store` | `stores` | Chi nhánh cửa hàng: địa chỉ, SĐT, giờ mở/đóng, mã nhúng Google Maps |
| 7 | `contact.js` | `Contact` | `contacts` | Phản hồi khách gửi từ trang Liên hệ, gắn với một `Store` |
| 8 | `user.js` | `User` | `users` | Tài khoản: email, mật khẩu băm, địa chỉ, `role`, `active` + method băm mật khẩu |
| 9 | `order.js` | `Order` | `orders` | Đơn hàng: người nhận, tổng tiền, trạng thái 0→4 |
| 10 | `orderDetail.js` | `OrderDetail` | `orderdetails` | Từng dòng trong đơn: sản phẩm, giá lúc mua, số lượng, % đá, % đường |
| 11 | `comment.js` | `Comment` | `comments` | Bình luận của user trên một sản phẩm |
| 12 | `rating.js` | `Rating` | `ratings` | Số sao user chấm cho một sản phẩm |
| 13 | `favoritesProduct.js` | `FavoritesProduct` | `favoritesproducts` | Bảng nối n-n: user nào thích sản phẩm nào (wishlist) |
| 14 | `voucher.js` | `Voucher` | `vouchers` | Mã giảm giá: code, số lượt, điều kiện, thời gian hiệu lực |

> 💡 **Bẫy đặt tên đáng nhớ:** file tên `slider.js` nhưng đăng ký `model("Slide", ...)` (`slider.js:25`)
> → collection thật là **`slides`**, không phải `sliders`. Mở Compass tìm mãi không thấy là vì vậy.
> Còn `yotea-be/src/models/news.js:3` đặt biến schema là `const categorySchema = ...` — copy-paste từ
> `category.js` mà quên đổi tên; `orderDetail.js:3` cũng vậy, biến tên là `orderSchema`. Code chạy đúng
> nhưng người đọc sau khốn khổ.

---

## 8. ⚠️ Ba chỗ tầng model làm chưa chuẩn

### 8.1. `orderDetail.js` thiếu hẳn `toppingId` / `sizeId` — dù frontend đang gửi lên

Đây là lỗi thú vị nhất của cả tầng model, và liên quan trực tiếp tới chức năng Topping bạn sắp làm.

`yotea-be/src/models/orderDetail.js:3-31`

```js
const orderSchema = new Schema(
  {
    orderId: {
      type: ObjectId,
      ref: "Order",
    },
    productId: {
      type: ObjectId,
      ref: "Product",
    },
    productPrice: {
      type: Number,
      required: true,
    },
    quantity: {
      type: Number,
      required: true,
    },
    ice: {
      type: Number,
      required: true,
    },
    sugar: {
      type: Number,
      required: true,
    },
  },
  { timestamps: true }
);
```

Đúng 6 trường. Không có `toppingId`, không có `sizeId`. Giờ nhìn sang frontend —
`yotea-fe/src/pages/user/cart/CheckoutPage.js:76-87`:

```js
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

Frontend gửi **10 trường**, backend khai báo **6**.

> ⚠️ **Mongoose chạy ở chế độ `strict` mặc định: mọi trường không có trong Schema đều bị ÂM THẦM LOẠI
> BỎ.** Không lỗi, không cảnh báo, không log. API trả `200 OK`, frontend tưởng lưu thành công, mà dữ
> liệu size/topping **bốc hơi hoàn toàn**.

Đây là loại bug khó chịu nhất trong nghề: mọi thứ trông như đang chạy tốt. Và nó rất phổ biến — người
mới hay hỏi *"em gửi lên rồi mà sao database không có?"* thì 9/10 lần là quên khai báo trường trong
Schema. Truy tiếp sẽ thấy `yotea-fe/src/pages/user/ProductDetailPage.js` khi thêm vào giỏ **cũng chưa
bao giờ set** `toppingId`/`sizeId`, nên bốn biến đó luôn `undefined`. Nói cách khác: **tính năng size
và topping đã được thiết kế nhưng bỏ dở giữa chừng, ở cả hai đầu.**

Đó chính xác là lý do bạn sẽ xây lại nó từ đầu, đúng cách, bắt đầu từ hôm nay. Tới
[Bài 10](10-quan-he-va-populate.md) bạn sẽ nối `Topping` vào `OrderDetail` để hoàn thiện mảnh ghép này.

### 8.2. `voucher.js` — model tồn tại nhưng không ai dùng

> ⚠️ Model `Voucher` khai báo đủ 8 trường, có `unique`, `uppercase`, `timestamps`, text index — nhưng
> **không có `controllers/voucher.js`, không có `routes/voucher.js`, và `app.js` không mount nó**. Bạn
> kiểm chứng ngay được: `yotea-be/src/app.js:7-20` import đúng 14 router, tuyệt nhiên không có
> `voucherRouter`. Bên frontend, tìm từ khoá `voucher` trong toàn bộ `yotea-fe/src` cho **0 kết quả**.
>
> → Collection `vouchers` sẽ **mãi mãi rỗng** — thậm chí MongoDB còn chưa tạo nó, vì collection chỉ
> sinh ra khi có document đầu tiên. Hai trường `Order.priceDecrease` và `Order.voucher` cũng nằm chờ
> cùng số phận: `CheckoutPage.js` chỉ gửi 7 trường, không có chúng. Đây là **tính năng bị bỏ dở rõ ràng
> nhất** của dự án, và cũng chính là đề tài của [Bài 35](35-do-an-cuoi-voucher.md).

### 8.3. `product.js` không ràng buộc `price >= 0`

Đã phân tích ở mục 3. Nhắc lại để nhớ: **model là hàng rào cuối cùng trước database.** Validate ở form
React chỉ ngăn được người dùng thật thà; ai mở Postman gọi thẳng API thì hàng rào duy nhất còn lại là
Schema. Bỏ trống nó nghĩa là bỏ trống luôn.

---

## 9. 🛠️ Tự tay làm — model `Topping` đầu tiên của bạn

> Mục tiêu phần này: cuối phần bạn có file **`yotea-be/src/models/topping.js`** do chính tay bạn viết,
> đúng chuẩn 4 phần của dự án, và đã kiểm chứng nó ghi được dữ liệu xuống MongoDB.

Yotea bán trà sữa mà **chưa có topping** — trân châu đen, thạch dừa, pudding trứng… Ta xây chức năng
đó, bắt đầu từ tầng thấp nhất: model.

### Bước 1 — Thiết kế trước, gõ sau

| Trường | Kiểu | Ràng buộc | Vì sao chọn như vậy |
|---|---|---|---|
| `name` | `String` | `required`, `trim` | Bắt chước `product.js:4-8`. `trim` để `"  Trân châu  "` không lưu kèm khoảng trắng thừa |
| `price` | `Number` | `required` | Giá cộng thêm khi khách chọn topping — phải là số để còn cộng vào tổng tiền |
| `image` | `String` | không bắt buộc | Ảnh minh hoạ, có cũng được không có cũng được — giống `category.js:15` |
| `status` | `Number` | `default: 0` | Giống `Product.status`: 0 = Ẩn, 1 = Hiển thị. Hết trân châu thì admin ẩn đi thay vì xoá |
| `slug` | `String` | `unique`, `lowercase` | Để có URL đẹp `/topping/tran-chau-duong-den`; `lowercase` chống trùng do khác hoa/thường |

### Bước 2 — Tạo file

Mở thư mục `yotea-be/src/models/`, tạo file mới tên **`topping.js`** (chữ `t` thường — dự án đặt tên
file model theo camelCase), rồi gõ:

```js
// yotea-be/src/models/topping.js  ← file MỚI, bạn tự tạo. Dự án gốc KHÔNG có file này.
import { Schema, model } from "mongoose";

const toppingSchema = new Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true,
    },
    price: {
      type: Number,
      required: true,
    },
    image: String,
    status: {
      type: Number,
      default: 0,
    },
    slug: {
      type: String,
      unique: true,
      lowercase: true,
    },
  },
  { timestamps: true }
);

toppingSchema.index({ "$**": "text" });

export default model("Topping", toppingSchema);
```

### Bước 3 — Giải thích từng lựa chọn

| Dòng bạn vừa gõ | Vì sao viết như vậy |
|---|---|
| `import { Schema, model }` | Chỉ cần 2 thứ. **Không** import `ObjectId` vì Topping chưa có quan hệ nào — tới [Bài 10](10-quan-he-va-populate.md) mới cần |
| `const toppingSchema = ...` | Đặt tên biến **đúng theo model**. Đừng lặp lại lỗi của `news.js:3` (đặt tên `categorySchema` trong file news) |
| `name: { required, trim }` | Không có tên thì không hiển thị được. `trim` xử lý khoảng trắng do người dùng gõ ẩu |
| `price: { type: Number }` | **Number** chứ không phải String — nếu để String thì `35000 + "5000"` ra `"350005000"` |
| `image: String` | Dạng rút gọn vì không cần ràng buộc. Thiếu ảnh thì frontend hiển thị ảnh mặc định |
| `status: { default: 0 }` | Dùng `Number` cho **nhất quán** với toàn dự án. Mặc định `0` (Ẩn) để topping mới tạo chưa hiện ra ngay |
| `slug: { unique, lowercase }` | Cố ý **chưa đặt `required`** — tới [Bài 08](08-slug-slugify.md) mới học cách sinh slug tự động bằng `slugify` |
| `{ timestamps: true }` | Để sau này `?_sort=createdAt&_order=desc` chạy được — y hệt 14 model kia |
| `toppingSchema.index({ "$**": "text" })` | Để ô tìm kiếm `?q=trân châu` chạy được ở [Bài 09](09-bo-loc-query.md) |
| `model("Topping", toppingSchema)` | Collection MongoDB sinh ra sẽ tên là **`toppings`** |

> ⚠️ **Bẫy `unique` + không `required`:** MongoDB coi trường **thiếu** là `null`. Vì `slug` có `unique`,
> nếu bạn tạo **hai** topping mà **cả hai đều không có slug**, document thứ hai sẽ dính
> `E11000 duplicate key error` — do hai giá trị `null` trùng nhau! Vì vậy trong bài này **mỗi lần tạo
> topping hãy tự gõ slug vào**. Từ [Bài 08](08-slug-slugify.md), `slugify` sẽ sinh slug tự động và bạn
> có thể thêm `required: true` cho yên tâm. (Cách chuyên nghiệp hơn là thêm `sparse: true` — bỏ qua
> document thiếu trường khi kiểm tra unique.)

### Bước 4 — Vì sao chưa gọi được API?

Bạn vừa đúc xong **cái khuôn**. Nhưng khuôn nằm im trong xưởng thì chưa ai ngoài đường mua được bánh:

```
[ Client / Postman ]
        ↓  ❌ chưa có
[ routes/topping.js ]      ← Bài 06
        ↓  ❌ chưa có
[ controllers/topping.js ] ← Bài 06
        ↓  ✅ ĐÃ CÓ (bạn vừa viết)
[ models/topping.js ]
        ↓
[ MongoDB · collection toppings ]
```

Gọi `GET http://localhost:8080/api/toppings` lúc này sẽ trả **404** — hoàn toàn bình thường, đừng hoảng.

---

## 10. ✅ Kiểm chứng kết quả

Chưa có route, nên ta kiểm chứng bằng cách **nhờ `app.js` tạo thử một bản ghi lúc khởi động**.

### Bước 1 — Thêm đoạn kiểm tra TẠM THỜI vào `app.js`

Mở `yotea-be/src/app.js`, thêm dòng import vào cuối khối import router (khoảng dòng 20), rồi thay khối
`mongoose.connect(...)` (dòng 45-48) bằng đoạn dưới. **Toàn bộ đoạn này bạn tự viết để kiểm tra, dự án
gốc không có** — nhớ xoá đi ở Bước 5:

```js
import Topping from "./models/topping";   // ← TẠM THỜI

mongoose
  .connect("mongodb://localhost:27017/yotea")
  .then(async () => {
    console.log("Connected to MongoDB");

    const topping = await new Topping({
      name: "  Trân châu đường đen  ",
      price: 10000,
      slug: "tran-chau-duong-den",
    }).save();

    console.log("ĐÃ TẠO TOPPING:", topping);
  })
  .catch((error) => console.log(error));
```

### Bước 2 — Chạy server

```bash
# đứng tại thư mục yotea-be
npm start
```

### Bước 3 — Đọc kết quả trong terminal

```
Connected to MongoDB
ĐÃ TẠO TOPPING: {
  name: 'Trân châu đường đen',
  price: 10000,
  status: 0,
  slug: 'tran-chau-duong-den',
  _id: new ObjectId("6650b3a1c4e8b91234abcd77"),
  createdAt: 2026-08-15T10:04:17.882Z,
  updatedAt: 2026-08-15T10:04:17.882Z,
  __v: 0
}
```

**Bốn thứ cần soi kỹ — chúng chứng minh Schema của bạn hoạt động đúng:**

| Quan sát | Chứng minh điều gì |
|---|---|
| `name` mất sạch khoảng trắng dù bạn gõ `"  Trân châu đường đen  "` | `trim: true` đang chạy |
| `status: 0` xuất hiện dù bạn **không hề gửi** trường này | `default: 0` đang chạy |
| Có `createdAt` và `updatedAt`, **giống hệt nhau** | `{ timestamps: true }` đang chạy, document mới tạo lần đầu |
| Có `_id` và `__v` | MongoDB và Mongoose tự thêm |

Mở **MongoDB Compass** → database `yotea` → bạn sẽ thấy collection **mới toanh tên `toppings`** chứa
đúng 1 document.

### Bước 4 — Thử phá cho nó lỗi (bước quan trọng nhất)

Sửa đoạn tạo topping thành thiếu `price` (`{ name: "Thạch dừa", slug: "thach-dua" }`) rồi chạy lại.
Terminal phải in:

```
ValidationError: Topping validation failed: price: Path `price` is required.
```

🎉 **Đây mới là bằng chứng cuối cùng.** Schema của bạn đang thật sự đứng gác: dữ liệu thiếu trường bắt
buộc bị chặn **trước khi** chạm tới MongoDB. Đó chính xác là toàn bộ lý do Mongoose tồn tại.

### Bước 5 — Dọn dẹp

**Xoá toàn bộ đoạn tạm** trong `app.js`, trả `mongoose.connect(...)` về đúng nguyên trạng
(`app.js:45-48`), xoá dòng `import Topping`. File `models/topping.js` thì **giữ lại** — nó là thành quả
của bạn, bài sau còn dùng. Muốn xoá document thử nghiệm thì vào Compass bấm biểu tượng thùng rác.

---

## 11. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `MongooseServerSelectionError: connect ECONNREFUSED` | MongoDB chưa chạy | PowerShell quyền admin, gõ `net start MongoDB` |
| ``ValidationError: Path `price` is required`` | Thiếu trường có `required: true` | Bổ sung trường — **đây là lỗi tốt**, chứng tỏ Schema đang bảo vệ bạn |
| `E11000 duplicate key error ... index: slug_1` | Trùng `slug`, hoặc tạo 2 document **cùng thiếu** `slug` | Đổi slug cho khác, hoặc xoá document cũ trong Compass |
| ``Cannot overwrite `Topping` model once compiled`` | Gọi `model("Topping", ...)` hai lần (thường do copy file model) | Mỗi tên model chỉ được đăng ký **một lần** trong toàn dự án |
| `Cast to Number failed for value "ba mươi nghìn"` | Gửi chuỗi không đổi được sang số vào trường `Number` | `"35000"` thì đổi được, `"ba mươi"` thì không — sửa dữ liệu client gửi |
| Lưu thành công nhưng **thiếu trường** trong database | Trường đó không có trong Schema → strict mode cắt bỏ | Bổ sung trường vào Schema. Đúng là bug `toppingId`/`sizeId` ở mục 8.1 |
| `text index required for $text query` | Truy vấn `$text` mà collection chưa có text index | Thêm `schema.index({ "$**": "text" })`, khởi động lại server |
| Sửa Schema xong mà **không thấy đổi gì** | Server chưa khởi động lại, hoặc index cũ còn trong MongoDB | `Ctrl + C` rồi `npm start`. Với index: xoá index cũ ở tab **Indexes** của Compass |

> 💡 **Mẹo về `unique`:** nó **không phải validator**, chỉ yêu cầu MongoDB tạo một unique index. Nên lỗi
> trùng không hiện dạng `ValidationError` đẹp đẽ mà là `E11000` thô. Và nếu collection **đã có sẵn dữ
> liệu trùng** từ trước, index sẽ **tạo thất bại trong im lặng** — ràng buộc coi như không tồn tại.
> Kiểm tra bằng tab **Indexes** trong Compass.

---

## 12. 📝 Bài tập

**Bài 1.** Đọc model `Rating` (`yotea-be/src/models/rating.js:1-20`) và chỉ ra **hai** ràng buộc quan
trọng đang bị thiếu, kèm hậu quả cụ thể của từng cái.

```js
import { Schema, model, ObjectId } from "mongoose";

const ratingSchema = new Schema({
    userId: {
        type: ObjectId,
        ref: "User"
    },
    ratingNumber: {
        type: Number,
        required: true
    },
    productId: {
        type: ObjectId,
        ref: "Product"
    }
}, { timestamps: true });

ratingSchema.index({'$**': 'text'});

export default model("Rating", ratingSchema);
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Thiếu thứ nhất: `ratingNumber` không có `min` / `max`.** Đánh giá sao lẽ ra chỉ 1→5, nhưng schema chỉ
nói "phải là số và bắt buộc". Gọi Postman với `{ "ratingNumber": 9999 }` là lưu được ngay → điểm trung
bình sao bị đẩy lên vô lý, giao diện vẽ 9999 ngôi sao hoặc vỡ layout. Sửa (**bạn tự viết, dự án chưa có**):

```js
ratingNumber: { type: Number, required: true, min: 1, max: 5 },
```

**Thiếu thứ hai: không có unique index kép `{ userId, productId }`.** Một người chấm sao cho cùng một
sản phẩm **vô số lần**, mỗi lần thêm một document. Ai muốn "dìm" đối thủ chỉ cần bấm 1 sao một nghìn lần.

```js
ratingSchema.index({ userId: 1, productId: 1 }, { unique: true });
```

**Điểm thưởng:** `userId` và `productId` đều **không `required`** → lưu được đánh giá "mồ côi", không
thuộc sản phẩm nào và không của ai. Model `FavoritesProduct` mắc y hệt bộ ba lỗi này.

</details>

**Bài 2.** Bổ sung cho model `Topping` của bạn hai trường mới: `description` (chuỗi, không bắt buộc) và
`quantity` (số lượng tồn kho, mặc định `0`, **không được âm**). Giải thích vì sao `quantity` cần `min`
còn `description` thì không.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Code **bạn tự viết thêm** vào `yotea-be/src/models/topping.js`, chèn giữa `image` và `status`:

```js
    image: String,
    description: String,
    quantity: {
      type: Number,
      default: 0,
      min: 0,
    },
```

- `description` là văn bản tự do, không có "giá trị sai" nào cả — khai báo rút gọn là đủ, giống `product.js:17`.
- `quantity` là **số lượng vật lý**: kho không thể có `-5` gói trân châu. `min: 0` biến điều vô lý đó
  thành `ValidationError` ngay tại tầng model, thay vì để nó âm thầm trôi vào database rồi làm sai báo
  cáo tồn kho.
- Đây chính xác là ràng buộc mà `Product.price` đang thiếu (mục 8.3) — bạn vừa làm tốt hơn dự án gốc.

Khởi động lại server rồi thử tạo topping với `quantity: -1` để tận mắt thấy lỗi.

</details>

**Bài 3.** Có người đề xuất: *"`Topping.status` chỉ có 2 giá trị Ẩn/Hiện, sao không đổi luôn sang
`Boolean` cho gọn?"* Nêu **hai lý do nên đổi** và **hai lý do không nên đổi**, rồi chọn một phương án và
bảo vệ nó.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**NÊN đổi:** (1) rõ nghĩa — `topping.isVisible === true` đọc phát hiểu, còn `status === 1` phải đi tra
tài liệu; (2) ràng buộc chặt hơn — `Boolean` chỉ nhận `true`/`false`, còn `Number` thì ai gửi `status: 7`
vẫn lưu được rồi giao diện không biết vẽ gì.

**KHÔNG nên đổi:** (1) tính nhất quán — cả 14 model của Yotea đều dùng `Number` cho `status`/`role`/
`active`, một mình Topping làm khác khiến người đọc phải dừng lại tự hỏi "sao chỗ này lại khác?";
(2) khó mở rộng — mai kia cần thêm trạng thái "Tạm hết hàng" (giá trị `2`) thì `Boolean` bí đường, phải
đổi kiểu và migrate toàn bộ dữ liệu cũ.

**Phương án nên chọn:** giữ `Number` **nhưng thêm `enum`** — lấy được cả hai (**bạn tự viết thêm**):

```js
status: { type: Number, enum: [0, 1], default: 0 },
```

Vẫn nhất quán với dự án, vẫn mở rộng được (sau này sửa thành `[0, 1, 2]`), mà `status: 7` giờ bị chặn
thẳng bằng `ValidationError`.

**Bài học lớn hơn:** khi làm trên codebase có sẵn, "đúng về lý thuyết" chưa phải lý do đủ để đổi — phải
cân thêm tính nhất quán và chi phí migrate. Đó cũng là tinh thần của [Bài 34](34-refactor-du-an.md).

</details>

---

## 📌 Tóm tắt

- **Schema là bản vẽ, Model là khuôn đúc, Document là sản phẩm đúc ra.** Nhớ ba từ này là hiểu cả tầng model.
- Mọi file model của Yotea theo **4 phần**: `import` → `new Schema({...}, { timestamps: true })` → `schema.index(...)` → `export default model("X", schema)`.
- Bảng option phải thuộc: `type`, `required`, `default`, `unique`, `trim`, `lowercase`, `min`, `ref`. Nhớ **`min` chỉ dùng cho `Number`/`Date`; với `String` phải là `minlength`** — dự án dính đúng bẫy này ở `user.js:14`.
- Dự án dùng **5 kiểu**: `String`, `Number`, `Date`, `Array`, `ObjectId`. `status`/`role`/`active` đều là `Number` — gọn và nhất quán, nhưng khó đọc và không ràng buộc được giá trị (nên thêm `enum`).
- `{ timestamps: true }` tặng miễn phí `createdAt`/`updatedAt`; **14/14 model đều bật**, và frontend sống nhờ chúng để sắp xếp.
- `schema.index({ "$**": "text" })` là **wildcard text index** phục vụ `?q=...` qua `filter.$text`. Nhược điểm lớn: **không hiểu tiếng Việt không dấu**, chỉ khớp trọn từ, và mỗi collection chỉ được **một** text index.
- **Mongoose strict mode âm thầm cắt bỏ mọi trường không khai báo trong Schema** — nguyên nhân khiến `toppingId`/`sizeId` mà `CheckoutPage.js` gửi lên bị bốc hơi.
- Bạn đã tự viết `yotea-be/src/models/topping.js` — viên gạch đầu tiên của chức năng Topping.

**Từ khoá tra cứu thêm:** `mongoose schema types`, `mongoose schema validation`, `mongoose timestamps`, `mongodb text index`, `mongodb wildcard index`, `mongoose strict mode`, `E11000 duplicate key error`

➡️ **Bài tiếp theo:** [06 — Vòng đời một request: Router → Controller → Model](06-vong-doi-mot-request.md) — bạn đã có khuôn đúc nhưng chưa ai gọi tới được. Bài sau ta lắp nốt route và controller, rồi lần đầu tiên gọi thành công `GET /api/toppings` bằng Postman.
