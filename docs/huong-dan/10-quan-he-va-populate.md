# Bài 10 — Quan hệ dữ liệu & `populate`

> **Phần 1 · Backend** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [09 — Bộ lọc kiểu json-server](09-bo-loc-query.md) · Bài sau: [11 — Mã hoá mật khẩu và xác thực bằng JWT](11-mat-khau-va-jwt.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu vì sao MongoDB **không có `JOIN`** như SQL, và hai cách người ta thay thế nó: **nhúng** (embed) và **tham chiếu** (reference).
- Đọc được **toàn bộ bản đồ quan hệ** của 14 model trong Yotea, biết model nào trỏ tới model nào.
- Khai báo được một trường tham chiếu: `type: ObjectId` + `ref: "TenModel"`, và biết chuỗi trong `ref` phải khớp với cái gì.
- Giải thích được `populate()` biến dữ liệu từ hình dạng nào sang hình dạng nào (có ví dụ JSON trước/sau).
- Hiểu cách dự án cho **client tự chọn** trường cần populate qua query `_expand`, và chuyện gì xảy ra khi client gõ sai tên trường.
- Nối được `Topping` (chức năng bạn đang tự xây từ Bài 05) vào `OrderDetail` và populate nó ra.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 09 — Bộ lọc kiểu json-server](09-bo-loc-query.md): bạn đã thêm được `_sort`, `_limit`, `q`… cho API Topping.
- Backend chạy được (`npm start` trong `yotea-be`, cổng 8080) và MongoDB đang bật.
- Có **Postman** và **MongoDB Compass** (cài ở [Bài 02](02-cai-dat-moi-truong.md)).

> 💡 **Mạch thực hành xuyên suốt:** ở bài trước bạn đã cho API Topping biết sắp xếp, giới hạn
> và tìm kiếm. **Bài này ta làm tiếp một việc quan trọng hơn: nối Topping vào đơn hàng**, để
> một dòng chi tiết đơn biết được khách đã chọn thêm trân châu hay thạch.

---

## 1. MongoDB không có JOIN — vậy nối dữ liệu kiểu gì?

### 1.1. Vấn đề đời thường

Bạn có hai quyển sổ:

- Sổ **Danh mục**: `1. Trà sữa`, `2. Trà trái cây`, `3. Cà phê`
- Sổ **Sản phẩm**: mỗi sản phẩm có tên, giá, và **thuộc danh mục nào**

Câu hỏi: khi in phiếu sản phẩm "Trà sữa trân châu đường đen", bạn muốn in kèm chữ
**"Trà sữa"** (tên danh mục). Có hai cách ghi:

| Cách | Cách ghi trong sổ Sản phẩm | Đời thường gọi là |
|---|---|---|
| **Nhúng** | Ghi thẳng `danh mục: Trà sữa` vào từng dòng sản phẩm | Chép lại cho tiện |
| **Tham chiếu** | Chỉ ghi `danh mục: số 1`, muốn biết tên thì lật sổ Danh mục | Ghi mã, tra sau |

Trong cơ sở dữ liệu SQL, "lật sổ" chính là câu lệnh `JOIN`. **MongoDB không có `JOIN`**
(nó có `$lookup` ở tầng aggregation, nhưng đó là chuyện khác và dự án này không dùng).
Vậy nên bạn phải chọn một trong hai cách trên ngay từ lúc thiết kế schema.

### 1.2. Cách 1 — Nhúng (embed)

Nhét luôn dữ liệu con vào trong document cha:

```json
{
  "_id": "6650a1f2c4e8b91234abcd01",
  "name": "Trà sữa trân châu đường đen",
  "price": 35000,
  "category": { "name": "Trà sữa", "slug": "tra-sua" }
}
```

| ✅ Ưu điểm | ❌ Nhược điểm |
|---|---|
| **Một truy vấn duy nhất** lấy đủ dữ liệu → rất nhanh | **Trùng lặp dữ liệu**: 200 sản phẩm cùng danh mục = 200 bản sao chữ "Trà sữa" |
| Dữ liệu con luôn "đi cùng" cha, không bao giờ mồ côi | Đổi tên danh mục ⇒ phải **sửa 200 document** |
| Hợp với dữ liệu **không dùng lại** (địa chỉ giao hàng của 1 đơn) | Document phình to; MongoDB giới hạn **16 MB / document** |

### 1.3. Cách 2 — Tham chiếu (reference)

Chỉ lưu **`_id`** của document bên kia:

```json
{
  "_id": "6650a1f2c4e8b91234abcd01",
  "name": "Trà sữa trân châu đường đen",
  "price": 35000,
  "categoryId": "6650a1f2c4e8b91234abcd99"
}
```

| ✅ Ưu điểm | ❌ Nhược điểm |
|---|---|
| **Một nguồn sự thật duy nhất**: đổi tên danh mục chỉ sửa 1 chỗ | Muốn có tên danh mục phải chạy **thêm truy vấn** |
| Document gọn nhẹ | Có thể **mồ côi**: xoá danh mục mà sản phẩm vẫn trỏ tới id đã chết |
| Dùng lại được cho nhiều nơi (1 danh mục ↔ N sản phẩm) | Không có ràng buộc khoá ngoại như SQL — MongoDB **không kiểm tra** id đó có tồn tại không |

### 1.4. Yotea chọn cách nào?

**Yotea chọn tham chiếu cho toàn bộ dự án.** Không có một chỗ nào nhúng document con.
Cụ thể có **11 trường khai đúng chuẩn `ObjectId` + `ref`** (trải trên 7 file model), cộng thêm
**1 quan hệ "trên giấy"** là `Order.userId` — chỉ là `String`, không có `ref` (xem mục 6.2).
Đây là lựa chọn hợp lý cho web bán hàng: danh mục, sản phẩm, người dùng đều là dữ liệu
**dùng lại nhiều lần** và **hay bị sửa tên**.

`populate()` của Mongoose chính là công cụ để "lật sổ" — thay id bằng object đầy đủ.

> 📖 **Thuật ngữ:** *populate* dịch thô là "làm đầy". Bạn đưa cho Mongoose một document
> đang cầm id trống rỗng, nó đi tra collection kia rồi **nhét object thật vào chỗ id đó**.

---

## 2. Bản đồ quan hệ toàn dự án Yotea

### 2.1. Bảng tra cứu 12 quan hệ (đã đối chiếu từng file model)

11 dòng đầu là tham chiếu thật (`ObjectId` + `ref`); dòng cuối chỉ là quan hệ trên giấy.

| Model nguồn | Trường | Kiểu | Trỏ tới model | Vị trí trong source |
|---|---|---|---|---|
| `Product` | `categoryId` | ObjectId (**required**) | `Category` | `yotea-be/src/models/product.js:30-34` |
| `News` | `category` | ObjectId | `CateNews` | `yotea-be/src/models/news.js:27-30` |
| `Contact` | `store` | ObjectId (**required**) | `Store` | `yotea-be/src/models/contact.js:20-24` |
| `Comment` | `userId` | ObjectId | `User` | `yotea-be/src/models/comment.js:4-7` |
| `Comment` | `productId` | ObjectId | `Product` | `yotea-be/src/models/comment.js:12-15` |
| `Rating` | `userId` | ObjectId | `User` | `yotea-be/src/models/rating.js:4-7` |
| `Rating` | `productId` | ObjectId | `Product` | `yotea-be/src/models/rating.js:12-15` |
| `FavoritesProduct` | `userId` | ObjectId | `User` | `yotea-be/src/models/favoritesProduct.js:4-7` |
| `FavoritesProduct` | `productId` | ObjectId | `Product` | `yotea-be/src/models/favoritesProduct.js:8-11` |
| `OrderDetail` | `orderId` | ObjectId | `Order` | `yotea-be/src/models/orderDetail.js:5-8` |
| `OrderDetail` | `productId` | ObjectId | `Product` | `yotea-be/src/models/orderDetail.js:9-12` |
| `Order` | `userId` | **`String` — KHÔNG có `ref`** ⚠️ | (User, quan hệ ngầm) | `yotea-be/src/models/order.js:4-6` |

Các model **hoàn toàn độc lập**, không tham chiếu ai và không ai tham chiếu tới:
`Slide`, `Voucher`. Còn `Category`, `CateNews`, `Store`, `User` là **đích đến** của quan hệ
chứ bản thân chúng không trỏ đi đâu.

### 2.2. Sơ đồ quan hệ (mermaid)

```mermaid
erDiagram
    Category  ||--o{ Product          : "categoryId (required)"
    CateNews  ||--o{ News             : "category"
    Store     ||--o{ Contact          : "store (required)"

    User      ||--o{ Comment          : "userId"
    User      ||--o{ Rating           : "userId"
    User      ||--o{ FavoritesProduct : "userId"

    Product   ||--o{ Comment          : "productId"
    Product   ||--o{ Rating           : "productId"
    Product   ||--o{ FavoritesProduct : "productId"
    Product   ||--o{ OrderDetail      : "productId"

    Order     ||--o{ OrderDetail      : "orderId"
    User      ..o{  Order             : "userId (String - KHONG ref)"
```

Ký hiệu `||--o{` đọc là **"một tới nhiều"**: một `Category` có nhiều `Product`.
Đường đứt nét `..o{` giữa `User` và `Order` là quan hệ **chỉ tồn tại trên giấy** —
xem mục 6.2.

### 2.3. Sơ đồ dạng text (dễ chép vào vở)

```
                         +-----------+                +-----------+
                         | Category  |                | CateNews  |
                         +-----+-----+                +-----+-----+
                               | 1                          | 1
                               | categoryId (required)      | category
                               v N                          v N
   +---------+  N        +----------+  N   +------------------+   +--------+
   | Comment |<----------|  Product |----->| FavoritesProduct |   |  News  |
   +---------+ productId +----+-----+      +--------+---------+   +--------+
        ^ userId              | productId           | userId
        |                     v N                   |
   +----+----+           +---------+                |
   |  User   |<----------| Rating  |                |
   +----+----+  userId   +---------+                |
        |  ^                                        |
        |  +----------------------------------------+
        |
        | userId : String  (KHONG PHAI ObjectId --> KHONG populate duoc)
        v
   +---------+  1        N  +---------------+  productId
   |  Order  |------------->|  OrderDetail  |-------------> Product
   +---------+   orderId    +---------------+

   +----------+  1      N  +----------+          +--------+  +---------+
   |  Store   |----------->| Contact  |          | Slide  |  | Voucher |
   +----------+   store    +----------+          +--------+  +---------+
                                                   << doc lap, khong quan he >>
```

---

## 3. Soi code thật: khai báo một tham chiếu

### 3.1. Trường `categoryId` của Product

`yotea-be/src/models/product.js:30-34`

```js
    categoryId: {
        type: ObjectId,
        ref: "Category",
        required: true,
    },
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 30 | `categoryId: {` | Tên trường trong document. Đây là tên bạn sẽ gõ vào `_expand` sau này |
| 31 | `type: ObjectId,` | Kiểu dữ liệu là **ObjectId** (chuỗi 24 ký tự hex), không phải `String` |
| 32 | `ref: "Category",` | **Chìa khoá của cả bài này**: id ở trên thuộc collection của model tên `Category` |
| 33 | `required: true,` | Bắt buộc phải có — sản phẩm không được "vô danh mục" |
| 34 | `},` | Đóng khối |

`ObjectId` được lấy ra ngay dòng đầu file, `yotea-be/src/models/product.js:1`

```js
import { Schema, model, ObjectId } from "mongoose";
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** cách viết chính thống là
> `Schema.Types.ObjectId` (hoặc `mongoose.Schema.Types.ObjectId`). `mongoose.ObjectId`
> là một **alias** cũ, chạy được nhưng không phải cách tài liệu Mongoose khuyến nghị.
> Cả 7 file model có `ref` trong dự án đều viết theo kiểu này. Bạn cứ đọc hiểu, nhưng khi
> viết code mới hãy dùng `Schema.Types.ObjectId`.

### 3.2. Chuỗi trong `ref` phải khớp CHÍNH XÁC với tên đăng ký model

Đây là chỗ người mới sai nhiều nhất. Chuỗi `"Category"` ở dòng 32 **không phải** tên file,
**không phải** tên collection trong MongoDB, mà là **tham số thứ nhất của `model(...)`**:

`yotea-be/src/models/category.js:20-22`

```js
categorySchema.index({ "$**": "text" });

export default model("Category", categorySchema);
```

Ba cái tên khác nhau, đừng nhầm:

| Thứ | Giá trị trong Yotea | Do đâu mà có |
|---|---|---|
| Tên **file** | `category.js` | Bạn tự đặt |
| Tên **model** (dùng cho `ref`) | `"Category"` | Tham số 1 của `model("Category", …)` |
| Tên **collection** trong MongoDB | `categories` | Mongoose tự **viết thường + số nhiều** hoá tên model |

Sai một ký tự — ví dụ `ref: "category"` (chữ c thường) — thì lúc populate Mongoose sẽ ném
`MissingSchemaError: Schema hasn't been registered for model "category"`.

Có một cái bẫy thật trong dự án: file `yotea-be/src/models/slider.js:25` đăng ký model tên
**`"Slide"`** chứ không phải `"Slider"` → collection trong MongoDB là **`slides`**. Nếu sau
này bạn cần `ref` tới nó, phải viết `ref: "Slide"`.

### 3.3. Ba kiểu đặt tên trường tham chiếu trong cùng một dự án

| Model | Trường | Nhận xét |
|---|---|---|
| `Product` | `categoryId` | Có hậu tố `Id` — rõ ràng nhất, đúng quy ước |
| `News` | `category` | **Không** có hậu tố `Id` → dễ tưởng là object |
| `Contact` | `store` | **Không** có hậu tố `Id` |

`yotea-be/src/models/news.js:27-30`

```js
    category: {
        type: ObjectId,
        ref: "CateNews"
    },
```

`yotea-be/src/models/contact.js:20-24`

```js
    store: {
        type: ObjectId,
        ref: "Store",
        required: true,
    },
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** ba cách đặt tên cho cùng một loại việc. Hệ quả trực tiếp:
> frontend phải nhớ gọi `_expand=categoryId` cho sản phẩm nhưng `_expand=store` cho liên hệ và
> `_expand=category` cho tin tức — sai một chữ là API trả lỗi. Cách làm đúng: **thống nhất
> một quy ước** cho cả dự án (khuyên dùng hậu tố `Id`).

### 3.4. Bảng nối n–n: `FavoritesProduct`

Model này thú vị vì nó **chỉ toàn khoá ngoại** — nó tồn tại chỉ để nối `User` với `Product`
(danh sách yêu thích).

`yotea-be/src/models/favoritesProduct.js:3-12`

```js
const schema = new Schema({
    userId: {
        type: ObjectId,
        ref: "User"
    },
    productId: {
        type: ObjectId,
        ref: "Product"
    }
}, { timestamps: true });
```

`Comment` và `Rating` cũng có đúng cặp `userId` + `productId` như vậy, chỉ khác là có thêm
phần nội dung (`content` cho Comment, `ratingNumber` cho Rating).

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** cả ba model trên **thiếu unique index kép**
> `{ userId, productId }`. Nghĩa là một người có thể "thích" cùng một sản phẩm 50 lần, hoặc
> chấm sao cùng một sản phẩm nhiều lần. Cách đúng:
> `schema.index({ userId: 1, productId: 1 }, { unique: true })`.

---

## 4. `populate()` làm gì — nhìn dữ liệu trước và sau

### 4.1. Trước khi populate

Gọi `GET http://localhost:8080/api/products` (không kèm `_expand`), một phần tử trả về có
dạng như sau (ví dụ minh hoạ, id là giả):

```json
{
  "_id": "6650a1f2c4e8b91234abcd01",
  "name": "Trà sữa trân châu đường đen",
  "image": "https://res.cloudinary.com/.../tra-sua.png",
  "price": 35000,
  "status": 1,
  "view": 128,
  "favorites": 3,
  "categoryId": "6650a1f2c4e8b91234abcd99",
  "slug": "tra-sua-tran-chau-duong-den",
  "createdAt": "2026-08-01T02:11:07.104Z",
  "updatedAt": "2026-08-12T09:40:55.980Z"
}
```

Để ý `"categoryId"` — nó chỉ là **một chuỗi id trơ trọi**. Frontend muốn hiện chữ
"Trà sữa" thì phải tự gọi thêm một request nữa tới `/api/categories/...`. Rất phiền.

### 4.2. Sau khi populate

Gọi `GET http://localhost:8080/api/products?_expand=categoryId`:

```json
{
  "_id": "6650a1f2c4e8b91234abcd01",
  "name": "Trà sữa trân châu đường đen",
  "image": "https://res.cloudinary.com/.../tra-sua.png",
  "price": 35000,
  "status": 1,
  "view": 128,
  "favorites": 3,
  "categoryId": {
    "_id": "6650a1f2c4e8b91234abcd99",
    "name": "Trà sữa",
    "slug": "tra-sua",
    "image": "https://res.cloudinary.com/.../cate.png",
    "createdAt": "2026-07-20T01:00:00.000Z",
    "updatedAt": "2026-07-20T01:00:00.000Z",
    "__v": 0
  },
  "slug": "tra-sua-tran-chau-duong-den",
  "createdAt": "2026-08-01T02:11:07.104Z",
  "updatedAt": "2026-08-12T09:40:55.980Z"
}
```

**Khác biệt duy nhất:** trường `categoryId` từ **chuỗi** biến thành **object đầy đủ**.
Nhờ vậy frontend viết thẳng `product.categoryId.name` là ra chữ "Trà sữa".

> ⚠️ **Cái bẫy lớn nhất của populate:** cùng một trường `categoryId` mà lúc là chuỗi, lúc là
> object — tuỳ vào việc request có `_expand` hay không. Nếu code frontend viết
> `product.categoryId.name` mà API lại không populate, bạn sẽ nhận `undefined`; ngược lại nếu
> so sánh `item.userId === user._id` trong khi `userId` đã bị populate thành object thì phép
> so sánh **luôn sai**. Bug kiểu này có thật trong Yotea — ta sẽ mổ ở
> [Bài 30](30-binh-luan-danh-gia-yeu-thich.md).

### 4.3. Bên trong, populate chạy mấy truy vấn?

`populate()` **không phải JOIN**. Mongoose làm thế này:

```
Bước 1: db.products.find({})                       -> 20 sản phẩm, mỗi cái có categoryId
Bước 2: gom tất cả categoryId lại, loại trùng      -> [id1, id2, id3]
Bước 3: db.categories.find({ _id: { $in: [...] }}) -> 3 danh mục
Bước 4: Mongoose ghép trong bộ nhớ Node.js         -> gắn object vào từng sản phẩm
```

Tức là **2 lượt đi về database**, không phải 21 lượt (Mongoose đã gom id lại giúp bạn).
Nhưng vẫn là 2 chứ không phải 1 — populate **không miễn phí**. Đây là cái giá của việc
chọn "tham chiếu" thay vì "nhúng".

---

## 5. Cách Yotea cho client tự chọn trường cần populate: `_expand`

### 5.1. Soi controller

`yotea-be/src/controllers/orderDetail.js:15-28`

```js
export const read = async (req, res) => {
    const filter = { _id: req.params.id };
    const populate = req.query["_expand"];

    try {
        const orderDetail = await OrderDetail.findOne(filter).select("-__v").populate(populate).exec();
        res.json(orderDetail);
    } catch (error) {
        res.status(400).json({
            message: "Không tìm thấy đơn hàng",
            error
        });
    }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 16 | `const filter = { _id: req.params.id };` | Điều kiện tìm: lấy đúng document có `_id` nằm trên URL |
| 17 | `const populate = req.query["_expand"];` | **Đọc thẳng chuỗi client gửi lên** trong query `?_expand=...` |
| 20 | `.findOne(filter)` | Tìm 1 document |
| 20 | `.select("-__v")` | Bỏ trường kỹ thuật `__v` khỏi kết quả |
| 20 | `.populate(populate)` | Nở id thành object — **tên trường do client quyết định** |
| 20 | `.exec()` | Thực thi query, trả về Promise |
| 21 | `res.json(orderDetail)` | Trả JSON về client |

Hàm `list` cũng làm y hệt, chỉ khác là nối thêm cả chuỗi lọc/sắp xếp của [Bài 09](09-bo-loc-query.md).

`yotea-be/src/controllers/orderDetail.js:30-31`

```js
export const list = async (req, res) => {
    const populate = req.query["_expand"];
```

`yotea-be/src/controllers/orderDetail.js:88-97`

```js
    try {
        const ordersDetail = await OrderDetail
            .find(filter)
            .select("-__v")
            .populate(populate)
            .skip(start)
            .limit(limit)
            .sort(sortOpt)
            .exec();
        res.json(ordersDetail);
```

Đây là **chuỗi phương thức (method chaining)**: mỗi lệnh trả về chính đối tượng Query nên nối
tiếp được. Query chỉ thật sự chạy khi gặp `.exec()`.

> 💡 **Mẹo:** khuôn `const populate = req.query["_expand"]` rồi `.populate(populate)` xuất hiện
> **nguyên xi ở cả 14 controller** của dự án (`product.js:75` và `:108`, `order.js`,
> `comment.js`, `user.js`…). Học một lần, đọc được cả 14 file.

### 5.2. Frontend gọi như thế nào

`yotea-fe/src/api/orderDetail.js:10-13`

```js
export const get = (orderId) => {
  const url = `/${DB_NAME}/?orderId=${orderId}&_expand=productId`;
  return instance.get(url);
};
```

Với `DB_NAME = "orderDetail"`, URL thật là:

```
/orderDetail/?orderId=6650...&_expand=productId
```

Đọc là: *"cho tôi mọi dòng chi tiết thuộc đơn hàng này, và nhớ nở `productId` ra thành sản phẩm đầy đủ."*

Tương tự, trang thực đơn dùng `yotea-fe/src/api/product.js:14`

```js
  let url = `/${DB_NAME}/?_expand=categoryId&_sort=${sort}&_order=${order}`;
```

### 5.3. Nếu client truyền tên trường sai thì sao?

Đây là câu hỏi phải trả lời cho rõ, vì `_expand` là **chuỗi do người dùng nhập**, backend
không kiểm tra gì cả. Dự án dùng Mongoose **6.2.8** (`yotea-be/package.json:21`), hành vi như sau:

| Trường hợp | Ví dụ | Mongoose làm gì |
|---|---|---|
| **Không truyền `_expand`** | `GET /api/orderDetail` | `populate(undefined)` — Mongoose **bỏ qua lặng lẽ**, trả về id nguyên dạng. Không lỗi |
| **Tên trường không có trong schema** | `?_expand=abc` | **Ném lỗi** `Cannot populate path 'abc' because it is not in your schema. Set the 'strictPopulate' option to false to override.` |
| **Trường có thật nhưng không có `ref`** | `?_expand=quantity` | Mongoose **bỏ qua lặng lẽ** — trả về giá trị gốc, không lỗi, không populate |
| **Tên trường đúng và có `ref`** | `?_expand=productId` | Populate thành công ✅ |

Ở trường hợp thứ hai, lỗi bị `try…catch` của controller bắt được, nên client nhận về **HTTP 400**:

```json
{
  "message": "Không tìm thấy đơn hàng",
  "error": { "name": "MongooseError", "message": "Cannot populate path `abc` ..." }
}
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** thông báo lỗi hoàn toàn lạc đề — người dùng gõ sai
> `_expand` mà server báo *"Không tìm thấy đơn hàng"*, lại còn **ném nguyên object lỗi nội bộ
> ra ngoài** (lộ cấu trúc schema cho kẻ tấn công). Cách đúng: kiểm tra `_expand` với một
> **danh sách trắng** các trường cho phép, rồi trả `400` kèm thông báo đúng nghĩa:
>
> ```js
> // đoạn này bạn tự viết thêm, dự án chưa có
> const ALLOWED_EXPAND = ["orderId", "productId"];
> const populate = ALLOWED_EXPAND.includes(req.query["_expand"])
>   ? req.query["_expand"]
>   : "";
> ```

---

## 6. `Order` và `OrderDetail` — vì sao phải tách hai collection?

### 6.1. Một đơn hàng có nhiều dòng chi tiết

Nhìn tờ hoá đơn giấy ở quán trà sữa:

```
+-------------------------------------------------+
| DON HANG #A123                                  |  <-- phan NAY chi co 1
| Khach: Nguyen Van A - 0912345678                |      => collection "orders"
| Dia chi: 12 Le Loi, Q1                          |
| Tong tien: 110.000d                             |
+-------------------------------------------------+
| 1. Tra sua tran chau  x2  35.000  da 50% duong 30% |  <-- phan NAY co N dong
| 2. Tra dao cam sa     x1  40.000  da 70% duong 50% |      => collection "orderdetails"
+-------------------------------------------------+
```

Nếu nhét cả hai vào một collection, bạn sẽ phải lặp lại tên khách, số điện thoại, địa chỉ ở
**mỗi dòng sản phẩm**. Tách ra là đúng: phần đầu hoá đơn vào `Order`, mỗi dòng hàng vào một
document `OrderDetail` mang `orderId` trỏ ngược về đơn cha.

### 6.2. Đọc schema `Order` — và một điểm bất thường

`yotea-be/src/models/order.js:3-6`

```js
const orderSchema = new Schema({
    userId: {
        type: String,
    },
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `userId` của `Order` là **`String`**, không phải
> `ObjectId`, và **không có `ref`**. Hệ quả rất cụ thể:
>
> - **Không thể** gọi `GET /api/orders?_expand=userId` để lấy thông tin người đặt — Mongoose
>   sẽ bỏ qua lặng lẽ và trả về chuỗi id như cũ (trường hợp 3 ở bảng mục 5.3).
> - Frontend buộc phải lọc đơn của mình bằng `?userId=<chuỗi>` rồi tự gọi thêm API user nếu cần.
> - Khách vãng lai (chưa đăng nhập) được lưu `userId: ""` — chuỗi rỗng, không phải `null`
>   (xem `yotea-fe/src/pages/user/cart/CheckoutPage.js:52`). Với `ObjectId` thì chuỗi rỗng sẽ
>   bị Mongoose từ chối, nên đây có lẽ là lý do tác giả chọn `String`.
>
> **Cách làm đúng:** để `userId: { type: ObjectId, ref: "User", default: null }` rồi cho phép
> `null` với khách vãng lai. Khi đó `_expand=userId` sẽ chạy được như 11 quan hệ còn lại.

### 6.3. Đọc schema `OrderDetail`

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

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 3 | `const orderSchema = ...` | ⚠️ Biến tên `orderSchema` nhưng đây là file `orderDetail.js` — copy-paste sót từ `order.js` |
| 5-8 | `orderId` ref `"Order"` | Trỏ ngược về đơn hàng cha |
| 9-12 | `productId` ref `"Product"` | Trỏ tới sản phẩm được mua |
| 13-16 | `productPrice` | **Ảnh chụp giá** tại thời điểm mua — sau này sản phẩm tăng giá thì đơn cũ vẫn đúng |
| 17-20 | `quantity` | Số lượng |
| 21-24 | `ice` | Phần trăm đá: 0 / 30 / 50 / 70 / 100 |
| 25-28 | `sugar` | Phần trăm đường: 0 / 30 / 50 / 70 / 100 |
| 30 | `{ timestamps: true }` | Tự sinh `createdAt` / `updatedAt` |

`yotea-be/src/models/orderDetail.js:35`

```js
export default model("OrderDetail", orderSchema);
```

→ tên model là `"OrderDetail"`, collection trong MongoDB là **`orderdetails`**.

### 6.4. Frontend đọc chi tiết đơn bằng `_expand`

Trang admin xem chi tiết một đơn, `yotea-fe/src/pages/admin/cart/CartDetailPage.js:27-31`

```js
    const getOrderDetail = async () => {
      const { data } = await getOrderById(id);
      setOrderDetail(data);
    };
    getOrderDetail();
```

`getOrderById` chính là hàm `get` ở `yotea-fe/src/api/orderDetail.js:10-13` đã xem ở mục 5.2 —
nó gắn sẵn `_expand=productId`. Nhờ vậy, khi render, code truy cập thẳng vào object sản phẩm:

`yotea-fe/src/pages/admin/cart/CartDetailPage.js:161-176`

```jsx
                {orderDetail?.map((item, index) => (
                  <tr className="border-b" key={index}>
                    <td>{++index}</td>
                    <td className="py-2 flex items-center">
                      <img
                        src={item.productId.image}
                        className="w-10 h-10 object-cover"
                        alt=""
                      />
                      <div className="pl-3">
                        <Link
                          to={`/san-pham/${item.productId.slug}`}
                          className="text-blue-500"
                        >
                          {item.productId.name}
                        </Link>
                      </div>
```

`item.productId.image`, `item.productId.slug`, `item.productId.name` — cả ba dòng này
**chỉ chạy được vì đã populate**. Bỏ `_expand=productId` đi là trang admin trắng xoá kèm lỗi
`Cannot read properties of null (reading 'image')`.

### 6.5. Đơn hàng được tạo ra sao — và vì sao có thể sinh ra đơn rỗng

`yotea-fe/src/pages/user/cart/CheckoutPage.js:60-89`

```js
    const { data } = await addOrder(orderData);
    const orderId = data._id;

    // save order detail
    cart.forEach(
      async ({
        productId,
        productPrice,
        sizeId,
        sizePrice,
        quantity,
        ice,
        sugar,
        toppingId,
        toppingPrice,
      }) => {
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
      }
    );
```

Luồng thật sự là:

```
Request 1:  POST /api/orders          -> tra ve _id cua don
Request 2:  POST /api/orderDetail     (mon thu 1)
Request 3:  POST /api/orderDetail     (mon thu 2)
...
Request N+1: POST /api/orderDetail    (mon thu N)
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn — KHÔNG có transaction:**
> `Order` và các `OrderDetail` được tạo bằng **N+1 request riêng lẻ**, không nằm trong một
> giao dịch (transaction) nào. Trong toàn bộ `yotea-be/src` không hề có `startSession()` hay
> `startTransaction()`. Hậu quả rất thật:
>
> - Tạo `Order` xong, **rớt mạng** ở món đầu tiên → database còn lại một **đơn hàng rỗng**:
>   có tên khách, có tổng tiền 110.000đ, nhưng **không có món nào**. Không có cơ chế nào
>   dọn dẹp nó.
> - `cart.forEach(async …)` **không biết chờ `await`** (nhớ bẫy ở [Bài 03](03-kien-thuc-nen.md)),
>   nên ba dòng `dispatch(finishOrder())`, `toast.success(...)`, `navigate("/thank-you")` chạy
>   **ngay lập tức**. Khách đóng tab kịp lúc → chi tiết đơn mất mà vẫn thấy chữ "Đặt hàng thành công".
> - Xoá đơn (`DELETE /api/orders/:id`) **không xoá** các `OrderDetail` con → chi tiết mồ côi
>   nằm lại vĩnh viễn.
>
> **Cách làm đúng:** hoặc gộp thành **một endpoint duy nhất** `POST /api/orders` nhận cả mảng
> món rồi tạo trong một transaction, hoặc tối thiểu dùng
> `await Promise.all(cart.map(item => addOrderDetail(...)))` và bọc `try…catch` để biết mà báo lỗi.
> Ta sẽ sửa ở [Bài 28](28-thanh-toan.md) và [Bài 34](34-refactor-du-an.md).

### 6.6. ⚠️ Bug thật: `toppingId` và `sizeId` bị Mongoose âm thầm nuốt

Nhìn kỹ lại đoạn `CheckoutPage.js:76-87` ở trên: frontend gửi lên **10 trường**:

```
orderId, productId, productPrice, sizeId, sizePrice, quantity, ice, sugar, toppingId, toppingPrice
```

Nhưng schema `OrderDetail` (`yotea-be/src/models/orderDetail.js:3-31`) chỉ khai báo **6 trường**:

```
orderId, productId, productPrice, quantity, ice, sugar
```

| Trường FE gửi | Có trong schema? | Kết cục |
|---|---|---|
| `orderId` | ✅ | Lưu |
| `productId` | ✅ | Lưu |
| `productPrice` | ✅ | Lưu |
| `quantity` | ✅ | Lưu |
| `ice` | ✅ | Lưu |
| `sugar` | ✅ | Lưu |
| `sizeId` | ❌ | **Bị vứt bỏ, không báo lỗi** |
| `sizePrice` | ❌ | **Bị vứt bỏ, không báo lỗi** |
| `toppingId` | ❌ | **Bị vứt bỏ, không báo lỗi** |
| `toppingPrice` | ❌ | **Bị vứt bỏ, không báo lỗi** |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** Mongoose mặc định chạy ở chế độ **`strict: true`** —
> gặp trường **không có trong schema** thì nó **im lặng loại bỏ**, `res.json()` trả về 200 OK
> như không có chuyện gì. Không exception, không cảnh báo, không dòng log nào.
> Đây là kiểu bug khó chịu nhất: **mọi thứ trông như đang chạy đúng**.
>
> **Cách phát hiện sớm:** bật `new Schema({...}, { strict: "throw" })` — khi đó Mongoose sẽ
> **ném lỗi** thay vì nuốt, và bạn biết ngay ở lần test đầu tiên.

**Tự kiểm chứng bằng MongoDB Compass — làm ngay 3 phút:**

1. Mở **MongoDB Compass**, kết nối `mongodb://localhost:27017`, chọn database **`yotea`**.
2. Mở Postman, gửi request sau (chú ý: cố tình gửi kèm `toppingId`):

   ```
   POST http://localhost:8080/api/orderDetail
   Content-Type: application/json
   ```
   ```json
   {
     "orderId": "6650a1f2c4e8b91234abcd11",
     "productId": "6650a1f2c4e8b91234abcd01",
     "productPrice": 35000,
     "quantity": 2,
     "ice": 50,
     "sugar": 30,
     "toppingId": "6650a1f2c4e8b91234abcdff",
     "toppingPrice": 10000
   }
   ```
3. Đọc **response** trong Postman: bạn nhận HTTP **200** kèm document mới — nhưng **không có**
   `toppingId` và `toppingPrice` trong đó.
4. Sang Compass, mở collection **`orderdetails`**, sắp xếp theo `createdAt` giảm dần, mở
   document vừa tạo. **Đếm số trường**: chỉ có `_id`, `orderId`, `productId`, `productPrice`,
   `quantity`, `ice`, `sugar`, `createdAt`, `updatedAt`, `__v`. Hai trường topping **đã biến mất
   hoàn toàn**, không hề chạm tới ổ đĩa.

Đó chính là lý do bài này chọn Topping làm bài thực hành: **dự án đã có sẵn chỗ trống**, việc
của bạn là lấp nó.

---

## 7. 🛠️ Tự tay làm — nối Topping vào chi tiết đơn hàng

> Mục tiêu phần này: cuối phần, `OrderDetail` sẽ có thêm trường `toppingId` tham chiếu tới
> model `Topping` mà bạn đã viết ở [Bài 05](05-mongoose-model.md), và bạn gọi được
> `GET /api/orderDetail?_expand=toppingId` để thấy topping nở ra thành object đầy đủ.

Nhắc lại mạch: Bài 05 bạn tạo `models/topping.js` → Bài 06 tạo route + controller → Bài 07
làm đủ 5 thao tác CRUD → Bài 08 thêm `slug` → Bài 09 thêm bộ lọc query. **Bài này ta nối nó
với `OrderDetail`.**

### Bước 1 — Kiểm tra tên model Topping

Mở lại `yotea-be/src/models/topping.js` (file **bạn tự tạo** ở Bài 05) và tìm dòng cuối cùng.
Nó phải là:

```js
// yotea-be/src/models/topping.js — dòng cuối, code BẠN đã viết ở Bài 05
export default model("Topping", toppingSchema);
```

Chuỗi `"Topping"` này chính là thứ ta sắp gõ vào `ref`. Nếu bạn lỡ đặt là `"toppings"` hay
`"Toppings"` thì **sửa lại cho khớp** trước khi đi tiếp.

### Bước 2 — Thêm trường `toppingId` vào `models/orderDetail.js`

Mở `yotea-be/src/models/orderDetail.js`. Sau khối `productId` (kết thúc ở dòng 12), **chèn thêm**
khối dưới đây:

```js
// yotea-be/src/models/orderDetail.js — đoạn BẠN tự thêm, dự án gốc KHÔNG có
    toppingId: {
      type: ObjectId,
      ref: "Topping",
    },
    toppingPrice: {
      type: Number,
      default: 0,
    },
```

Sau khi chèn, phần đầu schema của bạn trông như sau (so lại cho chắc):

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
    toppingId: {          // <-- MỚI
      type: ObjectId,
      ref: "Topping",
    },
    toppingPrice: {       // <-- MỚI
      type: Number,
      default: 0,
    },
    productPrice: {
      ...
```

Ba điều cần để ý:

| Điều | Vì sao |
|---|---|
| `ObjectId` đã được import sẵn ở dòng 1 của file | Không cần thêm dòng `import` nào |
| **Không** đặt `required: true` | Khách hoàn toàn có thể mua trà sữa **không topping** |
| `toppingPrice` có `default: 0` | Không topping thì phụ thu bằng 0, khỏi phải kiểm tra `undefined` |

> 💡 **Mẹo:** đây là lần **đầu tiên và duy nhất** trong giáo trình bạn được sửa file có sẵn của
> `yotea-be`. Nếu muốn giữ dự án gốc nguyên vẹn để đối chiếu, hãy `git stash` hoặc tạo nhánh
> riêng: đứng ở thư mục gốc repo, gõ `git checkout -b thuc-hanh-topping`.

### Bước 3 — Khởi động lại server

Backend chạy bằng `nodemon` nên thường tự nạp lại. Nhưng **schema Mongoose chỉ được đọc một lần
lúc khởi động**, nên cứ tắt hẳn rồi bật lại cho chắc:

```bash
# đứng tại thư mục yotea-be
# nhấn Ctrl + C để tắt, rồi:
npm start
```

Terminal phải in ra:

```
App is running on port: 8080
Connected to MongoDB
```

### Bước 4 — Tạo một topping để lấy `_id`

Dùng API Topping bạn đã làm ở Bài 07:

```
POST http://localhost:8080/api/toppings
Content-Type: application/json
```
```json
{ "name": "Trân châu đen", "price": 10000 }
```

Chép lấy `_id` trong response, ví dụ `66b0c1aa2f9d4e0012ab34cd`.

### Bước 5 — Tạo một dòng chi tiết đơn CÓ topping

```
POST http://localhost:8080/api/orderDetail
Content-Type: application/json
```
```json
{
  "orderId": "<_id của một Order có sẵn>",
  "productId": "<_id của một Product có sẵn>",
  "productPrice": 35000,
  "quantity": 2,
  "ice": 50,
  "sugar": 30,
  "toppingId": "66b0c1aa2f9d4e0012ab34cd",
  "toppingPrice": 10000
}
```

> 💡 Chưa có `orderId` / `productId` thật? Mở Compass, vào collection `orders` và `products`,
> chép đại một `_id` bất kỳ. Nhớ rằng MongoDB **không kiểm tra** id đó có thật hay không.

---

## 8. ✅ Kiểm chứng kết quả

**Kiểm chứng 1 — trường mới đã được lưu.** Response của bước 5 phải **có** hai trường mới:

```json
{
  "_id": "66b0c2f1...",
  "orderId": "6650a1f2c4e8b91234abcd11",
  "productId": "6650a1f2c4e8b91234abcd01",
  "toppingId": "66b0c1aa2f9d4e0012ab34cd",
  "toppingPrice": 10000,
  "productPrice": 35000,
  "quantity": 2,
  "ice": 50,
  "sugar": 30,
  "createdAt": "2026-08-15T03:20:11.402Z",
  "updatedAt": "2026-08-15T03:20:11.402Z",
  "__v": 0
}
```

Nếu `toppingId` **vẫn biến mất** → schema chưa được nạp lại, quay về Bước 3.

**Kiểm chứng 2 — populate một trường.** Gọi:

```
GET http://localhost:8080/api/orderDetail?_expand=toppingId
```

`toppingId` phải nở thành object:

```json
[
  {
    "_id": "66b0c2f1...",
    "toppingId": {
      "_id": "66b0c1aa2f9d4e0012ab34cd",
      "name": "Trân châu đen",
      "price": 10000,
      "slug": "tran-chau-den",
      "createdAt": "2026-08-15T03:15:00.000Z",
      "updatedAt": "2026-08-15T03:15:00.000Z",
      "__v": 0
    },
    "productPrice": 35000,
    "quantity": 2,
    "ice": 50,
    "sugar": 30
  }
]
```

**Kiểm chứng 3 — populate hai trường một lúc.** Mongoose nhận **chuỗi nhiều tên cách nhau bởi
dấu cách**. Trong URL, dấu cách phải viết là `%20`:

```
GET http://localhost:8080/api/orderDetail?_expand=productId%20toppingId
```

Kết quả: **cả** `productId` **và** `toppingId` cùng nở thành object. Đây là tính năng có sẵn của
Mongoose mà dự án gốc chưa bao giờ khai thác — bạn vừa mở khoá nó bằng đúng 4 dòng schema.

**Kiểm chứng 4 — Compass.** Mở collection `orderdetails`, tìm document vừa tạo. Bạn phải thấy
`toppingId` với biểu tượng kiểu **ObjectId** (Compass hiển thị chữ `ObjectId('...')`), không
phải chuỗi thường. Nếu nó hiện dạng `"66b0c1aa..."` có ngoặc kép, nghĩa là bạn khai `type: String`
nhầm — sửa lại thành `ObjectId`.

---

## 9. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `MissingSchemaError: Schema hasn't been registered for model "Topping"` | Chuỗi trong `ref` không khớp tham số 1 của `model(...)`, hoặc file model chưa từng được `import` ở đâu | Mở `models/topping.js`, đối chiếu `model("Topping", …)`; đảm bảo `routes/topping.js` có import model (mount ở `app.js`) |
| `Cannot populate path 'toppingId' because it is not in your schema` | Quên lưu file, hoặc server chưa restart nên schema cũ vẫn nằm trong bộ nhớ | Ctrl+C rồi `npm start` lại |
| `Cast to ObjectId failed for value "abc" at path "toppingId"` | Gửi lên một chuỗi không phải 24 ký tự hex | Chép đúng `_id` từ Compass / từ response tạo topping |
| Trường mới **không xuất hiện** trong response, không báo lỗi | Mongoose `strict: true` nuốt trường lạ — schema chưa có trường đó | Kiểm tra lại chính tả tên trường (`toppingId` chứ không phải `toppingID`) |
| `Cannot read properties of null (reading 'name')` ở frontend | Populate ra `null` vì id trỏ tới document **đã bị xoá** | Dùng optional chaining `item.toppingId?.name`; về lâu dài nên chặn xoá topping đang được dùng |
| `populate()` trả về id nguyên dạng, không lỗi gì | Trường có tồn tại nhưng **thiếu `ref`** (đúng trường hợp `Order.userId`) | Thêm `ref: "TenModel"` vào schema |
| `MongooseServerSelectionError` | Chưa bật MongoDB | Chạy `net start MongoDB` (PowerShell quyền admin) |

---

## 10. 📝 Bài tập

**Bài 1.** Không chạy code, hãy trả lời: gọi `GET http://localhost:8080/api/orders?_expand=userId`
sẽ nhận được gì? Giải thích dựa trên schema.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Nhận về danh sách đơn hàng **bình thường**, `userId` vẫn là **chuỗi id trơ trọi**, và
**không có lỗi nào**.

Lý do: `yotea-be/src/models/order.js:4-6` khai `userId: { type: String }` — trường **có tồn tại**
trong schema (nên không dính lỗi `strictPopulate`), nhưng **không có `ref`** nên Mongoose không
biết phải tra ở collection nào. Trong mã nguồn Mongoose 6, đoạn xử lý populate gặp trường hợp
"không xác định được model đích" sẽ **`continue`** — tức là bỏ qua lặng lẽ.

Đây là kiểu lỗi nguy hiểm: frontend viết `order.userId.fullName` sẽ nhận `undefined` mà không
hề có manh mối nào từ phía server. Muốn sửa triệt để, phải đổi schema sang
`userId: { type: ObjectId, ref: "User", default: null }` — nhưng như vậy phải xử lý luôn trường
hợp khách vãng lai đang lưu chuỗi rỗng `""`.

</details>

**Bài 2.** Hãy cho `Topping` một quan hệ ngược lại: mỗi topping thuộc về **một nhóm topping**
(`CateTopping`), ví dụ "Trân châu", "Thạch", "Kem". Viết schema cho model mới và sửa
`models/topping.js` để tham chiếu tới nó. Sau đó gọi API để populate ra.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Tạo file mới (**bạn tự viết, dự án không có**):

```js
// yotea-be/src/models/cateTopping.js  ← file MỚI, bạn tự tạo
import { Schema, model } from "mongoose";

const cateToppingSchema = new Schema(
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
  },
  { timestamps: true }
);

cateToppingSchema.index({ "$**": "text" });

export default model("CateTopping", cateToppingSchema);
```

Rồi thêm vào `yotea-be/src/models/topping.js` (file bạn viết ở Bài 05):

```js
// đoạn BẠN tự thêm
    cateToppingId: {
      type: ObjectId,
      ref: "CateTopping",
    },
```

Nhớ bổ sung `ObjectId` vào dòng import:

```js
import { Schema, model, ObjectId } from "mongoose";
```

Kiểm chứng: `GET http://localhost:8080/api/toppings?_expand=cateToppingId`.

**Chú ý quan trọng:** model `CateTopping` **phải được nạp vào bộ nhớ** trước khi populate,
nếu không sẽ dính `MissingSchemaError`. Cách chắc ăn nhất là làm luôn route + controller cho nó
(như Bài 06) và mount vào `app.js`; khi đó `import` dây chuyền sẽ tự nạp model.

Với schema này bạn đã có một chuỗi quan hệ 3 tầng:
`OrderDetail → Topping → CateTopping`. Mongoose populate lồng nhau được:

```js
// đoạn tham khảo, dự án chưa có
OrderDetail.find({}).populate({
  path: "toppingId",
  populate: { path: "cateToppingId" },
});
```

</details>

**Bài 3.** Controller hiện tại nhận `_expand` từ client mà không kiểm tra gì. Hãy viết lại hàm
`read` của `controllers/orderDetail.js` sao cho: (a) chỉ cho phép populate các trường hợp lệ,
(b) nếu client gõ sai thì trả về **400 kèm thông báo đúng nghĩa** thay vì "Không tìm thấy đơn hàng".

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```js
// yotea-be/src/controllers/orderDetail.js — bản viết lại, code BẠN tự viết
const ALLOWED_EXPAND = ["orderId", "productId", "toppingId"];

export const read = async (req, res) => {
  const filter = { _id: req.params.id };
  const expand = req.query["_expand"];

  // (a) tách chuỗi thành mảng rồi lọc theo danh sách trắng
  const fields = (expand || "").split(" ").filter(Boolean);
  const invalid = fields.filter((f) => !ALLOWED_EXPAND.includes(f));

  // (b) báo lỗi đúng nghĩa
  if (invalid.length) {
    return res.status(400).json({
      message: `Không thể mở rộng trường: ${invalid.join(", ")}`,
      allowed: ALLOWED_EXPAND,
    });
  }

  try {
    const orderDetail = await OrderDetail.findOne(filter)
      .select("-__v")
      .populate(fields.join(" "))
      .exec();

    if (!orderDetail) {
      return res.status(404).json({ message: "Không tìm thấy chi tiết đơn hàng" });
    }

    res.json(orderDetail);
  } catch (error) {
    res.status(500).json({ message: "Lỗi máy chủ" });
  }
};
```

Bốn cải tiến so với bản gốc:

1. **Danh sách trắng** — client không thể dò cấu trúc schema bằng cách thử `_expand` lung tung.
2. **Thông báo lỗi đúng nghĩa** — không còn "Không tìm thấy đơn hàng" khi thật ra là gõ sai query.
3. **Có `return`** trước mỗi `res` — tránh đúng lỗi "thiếu return" mà dự án mắc ở
   `controllers/auth.js` (xem [Bài 11](11-mat-khau-va-jwt.md)).
4. **Không ném object `error` ra client** — chỉ log nội bộ, tránh lộ thông tin. Chi tiết ở
   [Bài 33](33-ra-soat-bao-mat.md).

Lưu ý `fields.join(" ")` khi mảng rỗng sẽ cho chuỗi `""` — Mongoose coi đó là **giá trị không
truthy** nên bỏ qua populate, đúng như mong muốn.

</details>

---

## 📌 Tóm tắt

- MongoDB **không có `JOIN`**. Hai lựa chọn thay thế: **nhúng** (nhanh, nhưng trùng lặp và khó sửa)
  và **tham chiếu** (gọn, một nguồn sự thật, nhưng phải truy vấn thêm). **Yotea chọn tham chiếu
  cho cả 12 quan hệ.**
- Khai báo tham chiếu = `type: ObjectId` + `ref: "TenModel"`. Chuỗi trong `ref` phải khớp **chính xác**
  tham số thứ nhất của `model("TenModel", schema)` — không phải tên file, không phải tên collection.
- `populate()` biến một **chuỗi id** thành **object đầy đủ**, bằng cách chạy thêm **một truy vấn**
  gom id rồi ghép trong bộ nhớ Node — không phải JOIN thật.
- Yotea để **client tự chọn** trường cần populate qua `_expand`: `const populate = req.query["_expand"]`
  rồi `.populate(populate)`, lặp lại y hệt ở cả 14 controller.
- Tên trường sai ⇒ Mongoose 6 ném `Cannot populate path ... because it is not in your schema` (controller
  trả 400 với thông báo lạc đề). Không truyền `_expand`, hoặc trường thiếu `ref` ⇒ **bỏ qua lặng lẽ**.
- `Order` (đầu hoá đơn) và `OrderDetail` (từng dòng hàng) tách hai collection, nối bằng `orderId`.
  Nhưng chúng được tạo bằng **N+1 request không transaction** → có thể sinh ra **đơn hàng rỗng**.
- Mongoose chế độ `strict` mặc định **âm thầm vứt bỏ** trường không có trong schema — đó là lý do
  `toppingId`/`sizeId` mà `CheckoutPage.js` gửi lên **chưa bao giờ được lưu**. Kiểm chứng bằng Compass:
  document trong `orderdetails` không hề có hai trường đó.

**Từ khoá tra cứu thêm:** `mongoose populate`, `mongoose ref ObjectId`, `embedding vs referencing mongodb`,
`strictPopulate mongoose 6`, `mongoose strict mode`, `mongodb transactions session`, `$lookup aggregation`

➡️ **Bài tiếp theo:** [11 — Mã hoá mật khẩu và xác thực bằng JWT](11-mat-khau-va-jwt.md) — dữ liệu đã
nối được với nhau rồi, giờ đến lúc hỏi: *ai được phép đọc nó?* Ta sẽ mổ xẻ cách Yotea băm mật khẩu
(và vì sao cách đó chưa an toàn), rồi phát tấm vé JWT cho người đăng nhập.
