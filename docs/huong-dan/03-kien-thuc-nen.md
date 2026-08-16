# Bài 03 — Kiến thức nền tối thiểu: ES6, async/await, HTTP/REST, MongoDB

> **Phần 0 · Khởi động** — Thời lượng ước tính: **~60 phút**
> ⬅️ Bài trước: [02 — Cài đặt môi trường và chạy dự án lần đầu](02-cai-dat-moi-truong.md) · Bài sau: [04 — Mổ xẻ `app.js`](04-express-va-appjs.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Đọc trôi chảy 10 cú pháp JavaScript hiện đại xuất hiện dày đặc trong Yotea.
- Hiểu **bất đồng bộ** (asynchronous) là gì, và vì sao gần như mọi hàm trong dự án đều có chữ `async`.
- Nắm được HTTP: method, status code, headers, body, query string, path params.
- Phân biệt được **REST API** với cách gọi API tuỳ tiện.
- Hiểu MongoDB lưu dữ liệu kiểu gì và khác gì so với cơ sở dữ liệu bảng (SQL).

## 📋 Cần chuẩn bị

- Đã chạy được dự án ở [Bài 02](02-cai-dat-moi-truong.md).
- Biết JavaScript ở mức cơ bản: biến, hàm, `if`, `for`, mảng, object.

> 💡 Bài này là **từ điển tra cứu**. Đọc một lượt cho quen mặt, đừng cố học thuộc.
> Sau này gặp cú pháp lạ ở các bài sau, quay lại đây tra.

---

## 1. Mười cú pháp JavaScript bạn sẽ gặp ở mọi file

### 1.1. `import` / `export` — chia code thành nhiều file

Thay vì nhét tất cả vào một file khổng lồ, ta chia nhỏ rồi nối lại.

Có **hai kiểu export**:

```js
// export mặc định — MỖI FILE CHỈ ĐƯỢC 1 CÁI
export default model("Product", productSchema);
// khi import: tên gì cũng được, KHÔNG có ngoặc nhọn
import Product from "../models/product";
import BatKyTenNao from "../models/product";   // vẫn chạy!

// export có tên — bao nhiêu cũng được
export const create = async (req, res) => { ... };
export const list = async (req, res) => { ... };
// khi import: PHẢI đúng tên, CÓ ngoặc nhọn
import { create, list } from "../controllers/product";
```

Nhìn vào dự án, `yotea-be/src/routes/product.js:1-4`:

```js
import { Router } from "express";
import { clientUpdate, create, list, read, remove, update } from "../controllers/product";
import { userById } from "../controllers/user";
import { isAdmin, isAuth, requireSignin } from "../middlewares/checkAuth";
```

**Đọc từng dòng:**

| Dòng | Ý nghĩa |
|---|---|
| 1 | Lấy **một phần** tên là `Router` ra khỏi thư viện `express` |
| 2 | Lấy 6 hàm từ file controller. Dấu `../` nghĩa là "lùi ra thư mục cha rồi đi tiếp" |
| 4 | Lấy 3 middleware. Không có `.js` ở cuối — Node tự thêm |

> 💡 **Mẹo phân biệt:** thấy **có ngoặc nhọn** `{ }` là export có tên → phải gõ đúng tên.
> Thấy **không có ngoặc nhọn** là export mặc định → đặt tên gì cũng được.

### 1.2. Arrow function — viết hàm ngắn gọn

Ba cách viết dưới đây **tương đương nhau**:

```js
// cách cũ
function tinhTong(a, b) {
  return a + b;
}

// arrow function đầy đủ
const tinhTong = (a, b) => {
  return a + b;
};

// arrow function rút gọn: 1 dòng thì bỏ luôn { } và return
const tinhTong = (a, b) => a + b;
```

Trong dự án, `yotea-fe/src/utils/index.js:14-15`:

```js
export const formatCurrency = (currency) =>
  currency.toLocaleString("it-IT", { style: "currency", currency: "VND" });
```

Đây là arrow function rút gọn: nhận vào một con số, trả về chuỗi tiền tệ kiểu `35.000 ₫`.
Không có `{ }` nghĩa là **tự động return**.

### 1.3. Destructuring — bóc tách dữ liệu

Kỹ thuật quan trọng nhất cần nắm, vì nó có mặt ở **mọi file** của dự án.

```js
const user = { email: "a@gmail.com", password: "123", role: 1 };

// cách cũ
const email = user.email;
const role = user.role;

// destructuring — lấy 2 thứ trong 1 dòng
const { email, role } = user;
```

Với **mảng**, dùng ngoặc vuông và lấy theo **thứ tự**:

```js
const [dau, thu2] = ["trà sữa", "trà đào", "cà phê"];
// dau = "trà sữa", thu2 = "trà đào"
```

Destructuring còn dùng được **ngay đầu hàm để bóc dữ liệu client gửi lên** —
`yotea-be/src/controllers/auth.js:33`:

```js
const { email, password } = req.body;
```

Nghĩa là: từ dữ liệu client gửi lên, lấy ra hai trường `email` và `password`.

**Đổi tên khi bóc tách** bằng dấu hai chấm:

```js
const { data: productsData } = await getAll(start, limit);
// lấy thuộc tính "data", nhưng gọi nó là "productsData"
```

Vì sao phải đổi tên? Vì trong `yotea-fe/src/redux/productSlice.js` có **hai lần** lấy
`data` trong cùng một hàm — không đổi tên sẽ trùng biến.

**Bóc tách lồng nhau** — đoạn khó nhất dự án, `yotea-be/src/controllers/auth.js:47-49`:

```js
const {
  _doc: { password: hashed_password, __v, ...rest },
} = user;
```

Đọc từ ngoài vào trong:

| Bước | Diễn giải |
|---|---|
| `_doc:` | Đi vào thuộc tính `_doc` của `user` (nơi Mongoose cất dữ liệu thật) |
| `password: hashed_password` | Lấy `password` ra, đặt tên là `hashed_password` |
| `__v` | Lấy trường `__v` ra (trường kỹ thuật của Mongoose) |
| `...rest` | **Tất cả phần còn lại** gom vào biến `rest` |

Mục đích thật sự: **tách `password` và `__v` ra khỏi dữ liệu**, để `rest` chỉ còn thông
tin an toàn gửi về cho client. Đây là một mẹo rất hay — hãy nhớ nó.

### 1.4. Spread `...` và Rest `...` — cùng ký hiệu, hai công dụng

**Rest** — gom phần còn lại lại (như ví dụ trên):

```js
const { password, ...rest } = user;   // rest = mọi thứ TRỪ password
```

**Spread** — trải một object/mảng ra:

```js
const update = {
  ...req.body,                  // trải hết dữ liệu client gửi lên
  slug: slugify(req.body.name), // rồi thêm/ghi đè trường slug
};
```

Đoạn trên lấy từ `yotea-be/src/controllers/product.js` (hàm `update`). Với mảng:

```js
state.value = [...state.value, payload];   // mảng cũ + phần tử mới
```

> 💡 **Cách nhớ:** `...` nằm ở **vế trái** dấu `=` (lúc bóc tách) là **rest** — *gom vào*.
> `...` nằm ở **vế phải** (lúc tạo giá trị mới) là **spread** — *trải ra*.

### 1.5. Template literal — nối chuỗi bằng dấu backtick

```js
// cách cũ, dễ sai dấu cách
const url = "/products/" + id + "/" + userId;

// template literal — dùng dấu ` (phím bên trái số 1)
const url = `/products/${id}/${userId}`;
```

Trong `yotea-fe/src/api/product.js:14`:

```js
let url = `/${DB_NAME}/?_expand=categoryId&_sort=${sort}&_order=${order}`;
```

Với `DB_NAME = "products"`, `sort = "createdAt"`, `order = "desc"`, chuỗi tạo ra là:

```
/products/?_expand=categoryId&_sort=createdAt&_order=desc
```

### 1.6. Tham số mặc định — giá trị dự phòng

```js
const getAll = (start = 0, limit = 0) => { ... };

getAll();        // start = 0,  limit = 0
getAll(10);      // start = 10, limit = 0
getAll(10, 5);   // start = 10, limit = 5
```

Dự án dùng chiêu này ở một chỗ rất "cao tay" —
`yotea-fe/src/api/product.js:59`:

```js
export const add = (product, { token, user } = isAuthenticate()) => {
```

Nghĩa là: *"nếu người gọi không truyền tham số thứ hai, hãy tự chạy `isAuthenticate()`
để lấy token và user từ localStorage."*

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** nếu người dùng **chưa từng đăng nhập**, hàm
> `isAuthenticate()` sẽ nổ lỗi vì `localStorage` chưa có dữ liệu. Xem
> [Bài 18](18-tang-api-axios.md) và [Bài 33](33-ra-soat-bao-mat.md).

### 1.7. Optional chaining `?.` — hỏi trước khi lấy

```js
errors.fullName.message     // 💥 nổ nếu errors.fullName là undefined
errors.fullName?.message    // ✅ trả về undefined, không nổ
```

Xuất hiện khắp các form trong dự án, ví dụ `yotea-fe/src/pages/user/cart/CheckoutPage.js`:

```jsx
<div className="text-sm mt-0.5 text-red-500">
  {errors.fullName?.message}
</div>
```

Nếu trường "Họ tên" chưa có lỗi thì `errors.fullName` là `undefined`; nhờ `?.` nên
React chỉ hiển thị rỗng thay vì làm sập cả trang.

### 1.8. Object shorthand và computed key

**Shorthand** — tên biến trùng tên thuộc tính thì viết một lần:

```js
const email = "a@gmail.com";
const user = { email: email };  // cách cũ
const user = { email };          // shorthand — y hệt
```

**Computed key** — tên thuộc tính là một biến, đặt trong ngoặc vuông.
`yotea-fe/src/redux/rootReducer.js:31-35`:

```js
[productApi.reducerPath]: productApi.reducer,
[sliderApi.reducerPath]: sliderApi.reducer,
[cateProductApi.reducerPath]: cateProductApi.reducer,
```

Vì `productApi.reducerPath` có giá trị `"productApi"`, dòng đầu tương đương:

```js
productApi: productApi.reducer,
```

Viết theo kiểu computed key để nếu sau này đổi tên `reducerPath`, code vẫn đúng —
không phải sửa hai chỗ.

### 1.9. Các phương thức mảng phải thuộc lòng

| Phương thức | Làm gì | Trả về |
|---|---|---|
| `map()` | Biến đổi **từng** phần tử | Mảng mới **cùng độ dài** |
| `filter()` | Giữ lại phần tử thoả điều kiện | Mảng mới **ngắn hơn hoặc bằng** |
| `find()` | Tìm phần tử **đầu tiên** thoả điều kiện | **Một** phần tử, hoặc `undefined` |
| `reduce()` | Gộp cả mảng thành **một** giá trị | Bất kỳ kiểu gì |
| `forEach()` | Chạy qua từng phần tử để làm việc gì đó | **`undefined`** (không trả về gì) |

Cả 5 đều có mặt trong `yotea-fe/src/redux/cartSlice.js`:

```js
// find — tìm sản phẩm đã có trong giỏ chưa (dòng 12)
const exitsProduct = cart.find(
  (item) =>
    item.productId === newProduct.productId &&
    item.ice === newProduct.ice &&
    item.sugar === newProduct.sugar
);

// filter — xoá một món khỏi giỏ (dòng 26)
state.cart = state.cart.filter((item) => item.id !== payload);
```

Và `reduce` để tính tổng tiền, trong `yotea-fe/src/pages/user/cart/CheckoutPage.js`:

```js
return cart.reduce((total, cart) => {
  return total + cart.productPrice * cart.quantity;
}, 0);
```

Đọc là: *"bắt đầu từ `0`, cộng dồn `giá × số lượng` của từng món."*

> ⚠️ **Bẫy chết người với `forEach`:** `forEach` **không biết chờ** `await`. Đoạn code
> đặt hàng trong `CheckoutPage.js` mắc đúng lỗi này. Ta sẽ mổ xẻ ở
> [Bài 28 — Thanh toán](28-thanh-toan.md).

### 1.10. Toán tử `||` và `&&` dùng như câu lệnh rẽ nhánh

```js
const PORT = process.env.PORT || 8080;
```

Đọc: *"lấy `process.env.PORT`; nếu nó rỗng/undefined thì dùng `8080`."*
(`yotea-be/src/app.js:51`)

```js
userId: (user && user._id) || "",
```

Đọc: *"nếu có `user` thì lấy `user._id`; không thì dùng chuỗi rỗng."* Đây là cách dự án
cho phép **khách vãng lai đặt hàng** mà không cần tài khoản.

Trong JSX, `&&` dùng để **hiển thị có điều kiện**:

```jsx
{user && (
  <div className="col-span-12 mb-3 flex items-center">
    <input type="checkbox" ... />
  </div>
)}
```

Đọc: *"chỉ vẽ ô checkbox này khi đã đăng nhập."*

---

## 2. Bất đồng bộ: Promise và async/await

### 2.1. Vấn đề cần giải quyết

Truy vấn database mất 50 mili-giây. Gọi API mất 300 mili-giây. Nếu JavaScript **đứng
chờ**, cả server sẽ đơ, không phục vụ được ai. Nên nó chọn cách: *"cứ giao việc đi, khi
nào xong thì báo tôi."* Đó là **bất đồng bộ**.

Ví von: bạn gọi một ly trà sữa. Nhân viên **không** bắt bạn đứng chết trân ở quầy —
họ đưa số thứ tự (đó là **Promise**) rồi phục vụ khách tiếp theo. Khi trà xong, họ gọi số.

### 2.2. Promise: `.then()` / `.catch()`

`yotea-be/src/app.js:45-48`:

```js
mongoose
  .connect("mongodb://localhost:27017/yotea")
  .then(() => console.log("Connected to MongoDB"))
  .catch((error) => console.log(error));
```

| Phần | Ý nghĩa |
|---|---|
| `.connect(...)` | Bắt đầu kết nối, **trả về ngay** một Promise (chưa xong việc) |
| `.then(...)` | Chạy khi **thành công** |
| `.catch(...)` | Chạy khi **thất bại** |

### 2.3. async/await: viết bất đồng bộ mà nhìn như đồng bộ

`await` nghĩa là *"chờ Promise này xong rồi mới chạy dòng tiếp"*. Chỉ dùng được bên
trong hàm có gắn từ khoá `async`.

`yotea-be/src/controllers/product.js:35-45`:

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
| 35 | `async (req, res) =>` | Hàm bất đồng bộ, nhận request và response |
| 36 | `req.body.slug = slugify(...)` | Tạo slug từ tên sản phẩm |
| 38 | `try {` | Bắt đầu vùng "có thể lỗi" |
| 39 | `await new Product(...).save()` | **Chờ** lưu xuống MongoDB xong |
| 40 | `res.json(product)` | Trả sản phẩm vừa lưu về cho client |
| 41 | `} catch (error) {` | Nếu bất kỳ dòng nào trong `try` nổ, nhảy vào đây |
| 42 | `res.status(400).json({...})` | Trả mã lỗi 400 kèm thông báo |

> 📖 **Thuật ngữ:** `try...catch` là "lưới an toàn". Không có nó, một lỗi nhỏ trong
> truy vấn database sẽ làm **sập cả server**. Đây là lý do **mọi** controller trong dự
> án đều bọc `try...catch` — hãy bắt chước thói quen này.

### 2.4. Bảng đối chiếu nhanh

| `.then()` | `async/await` |
|---|---|
| `getAll().then(res => ...)` | `const res = await getAll()` |
| `.catch(err => ...)` | `try { } catch (err) { }` |
| Lồng nhiều tầng thì rối như tổ chim | Đọc từ trên xuống như văn xuôi |

Dự án dùng **cả hai**, có chỗ trộn lẫn — ví dụ `yotea-fe/src/redux/authSlice.js`:

```js
return update(dataAuth).then(async () => {
  const { data: { password, ...rest } } = await get(dataAuth._id);
  return rest;
});
```

Vừa `.then()` vừa `await` trong cùng một biểu thức. Chạy được, nhưng khó đọc.
Cách viết gọn hơn:

```js
await update(dataAuth);
const { data: { password, ...rest } } = await get(dataAuth._id);
return rest;
```

---

## 3. HTTP và REST API

### 3.1. Một request gồm những gì?

```
POST /api/products/6650a1f2c4e8b91234abcd01?_expand=categoryId   HTTP/1.1
│    │                                       │
│    └── path (đường dẫn)                    └── query string
└── method

Headers:
  Content-Type: application/json          ← đang gửi dữ liệu kiểu JSON
  Authorization: Bearer eyJhbGciOi...     ← tấm vé chứng minh đã đăng nhập

Body:
  { "name": "Trà sữa matcha", "price": 40000 }
```

| Thành phần | Vai trò | Lấy ở backend bằng |
|---|---|---|
| **Method** | Định "loại hành động" | `router.get()`, `router.post()`… |
| **Path params** | Phần biến trong đường dẫn (`:id`) | `req.params.id` |
| **Query string** | Tuỳ chọn sau dấu `?` | `req.query._sort` |
| **Headers** | Thông tin phụ trợ | `req.headers.authorization` |
| **Body** | Dữ liệu chính khi thêm/sửa | `req.body` |

### 3.2. Bốn method chính

| Method | Dùng khi | Ví dụ trong Yotea |
|---|---|---|
| `GET` | **Đọc** dữ liệu | `GET /api/products` — lấy danh sách sản phẩm |
| `POST` | **Tạo mới** | `POST /api/signin` — đăng nhập |
| `PUT` / `PATCH` | **Cập nhật** (PUT thay toàn bộ, PATCH sửa một phần) | `PATCH /api/products/userUpdate/:id` — tăng lượt xem |
| `DELETE` | **Xoá** | `DELETE /api/products/:id/:userId` |

### 3.3. Status code — bảng tra cứu

| Mã | Nhóm | Nghĩa | Gặp ở đâu trong Yotea |
|---|---|---|---|
| **200** | ✅ Thành công | OK | Mọi API chạy đúng |
| **400** | ❌ Lỗi phía client | Dữ liệu gửi lên sai | `res.status(400)` — thấy ở mọi `catch` |
| **401** | ❌ Lỗi phía client | Chưa đăng nhập / không đủ quyền | `isAdmin` trả về khi `role = 0` |
| **404** | ❌ Lỗi phía client | Không tìm thấy đường dẫn | Gõ sai URL API |
| **500** | 💥 Lỗi phía server | Backend nổ | Bug trong controller |

> 💡 **Mẹo nhớ:** **4xx là "lỗi tại bạn"** (client gửi sai), **5xx là "lỗi tại tôi"**
> (server hỏng). Debug 4xx thì soi lại dữ liệu gửi lên; debug 5xx thì soi terminal backend.

### 3.4. REST là gì?

**REST** là một bộ **quy ước đặt tên** để API dễ đoán. Nguyên tắc cốt lõi: URL mô tả
**tài nguyên** (danh từ), method mô tả **hành động** (động từ).

```
✅ Chuẩn REST                        ❌ Không chuẩn
GET    /api/products                 GET /api/layDanhSachSanPham
POST   /api/products                 POST /api/themSanPhamMoi
GET    /api/products/:id             GET /api/timSanPhamTheoId?id=5
PUT    /api/products/:id             POST /api/capNhatSanPham
DELETE /api/products/:id             GET  /api/xoaSanPham?id=5
```

Yotea theo REST **khá sát**, xem `yotea-be/src/routes/product.js:8-13`:

```js
router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/products/:slug", read);
router.get("/products", list);
router.put("/products/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.patch("/products/userUpdate/:id", clientUpdate);
router.delete("/products/:id/:userId", requireSignin, isAuth, isAdmin, remove);
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** thấy `:userId` nằm trên đường dẫn không? Đó là
> **id của người đang đăng nhập** bị nhét vào URL. REST chuẩn sẽ lấy thông tin này từ
> **token trong header**, không phải từ URL. Vì sao cách này vừa xấu vừa nguy hiểm —
> phân tích ở [Bài 12](12-phan-quyen-middleware.md) và [Bài 33](33-ra-soat-bao-mat.md).

---

## 4. MongoDB trong 10 phút

### 4.1. Khác gì cơ sở dữ liệu bảng?

| SQL (MySQL, SQL Server) | MongoDB | Trong Yotea |
|---|---|---|
| Database | Database | `yotea` |
| Table (bảng) | **Collection** | `products`, `users`, `orders` |
| Row (dòng) | **Document** | Một sản phẩm cụ thể |
| Column (cột) | **Field** | `name`, `price`, `slug` |
| Khoá chính `id` | **`_id`** (ObjectId, tự sinh) | `6650a1f2c4e8b91234abcd01` |
| Phải khai báo cấu trúc trước | **Không bắt buộc** — Mongoose tự thêm lớp kiểm tra | Các file trong `models/` |

Một document trông y như object JavaScript:

```json
{
  "_id": ObjectId("6650a1f2c4e8b91234abcd01"),
  "name": "Trà sữa trân châu đường đen",
  "price": 35000,
  "categoryId": ObjectId("6650a1f2c4e8b91234abcd99"),
  "slug": "tra-sua-tran-chau-duong-den",
  "view": 128,
  "createdAt": "2026-08-15T09:12:00.000Z"
}
```

Chính vì giống JavaScript đến thế nên MongoDB rất hợp với Node.js — dữ liệu đi từ
database lên tới React mà **không cần chuyển đổi** kiểu.

### 4.2. `_id` và ObjectId

Mỗi document tự động có `_id` kiểu **ObjectId** — chuỗi 24 ký tự hex, đảm bảo không
bao giờ trùng. Trong Yotea, các model tham chiếu nhau qua `_id`:

`yotea-be/src/models/product.js` (trích):

```js
categoryId: {
    type: ObjectId,
    ref: "Category",
    required: true,
},
```

Đọc là: *"trường `categoryId` chứa `_id` của một document bên collection `Category`."*
Nhờ khai báo `ref` mà sau này ta gọi được `.populate()` để "nở" cái id đó thành object
đầy đủ. Chi tiết ở [Bài 10](10-quan-he-va-populate.md).

### 4.3. Mongoose là gì?

MongoDB thô cho phép lưu **bất cứ thứ gì** — hôm nay `price` là số, mai ai đó lưu
`price: "ba mươi lăm nghìn"` cũng không ai cản. Rất dễ loạn.

**Mongoose** là thư viện đứng giữa Node.js và MongoDB, thêm hai thứ quý giá:

1. **Schema** — bản khai báo "sản phẩm phải có `name` là chuỗi, `price` là số, thiếu là không cho lưu".
2. **API dễ dùng** — `Product.find()`, `Product.findOneAndUpdate()` thay vì câu lệnh MongoDB thô.

```js
// Mongoose (dự án dùng)
const products = await Product.find({ status: 1 }).limit(10).exec();

// MongoDB thuần
db.collection("products").find({ status: 1 }).limit(10).toArray();
```

---

## 5. 🛠️ Tự tay làm — luyện tay 15 phút

> Mục tiêu: tự tay chuyển đổi qua lại giữa cú pháp cũ và mới, để lát nữa đọc code dự án
> không bị khựng.

Tạo file nháp `luyen-tap.js` ở bất cứ đâu **ngoài** thư mục dự án, rồi chạy `node luyen-tap.js`.

### Bước 1 — Luyện destructuring

```js
const donHang = {
  _id: "abc123",
  customerName: "Nguyễn Văn A",
  phone: "0912345678",
  totalPrice: 150000,
  status: 0,
};

// TODO 1: lấy customerName và totalPrice ra 2 biến riêng
// TODO 2: lấy mọi thứ TRỪ _id vào biến tên là thongTin
// TODO 3: lấy customerName ra nhưng đặt tên biến là tenKhach

console.log(customerName, totalPrice);
console.log(thongTin);
console.log(tenKhach);
```

<details>
<summary>💡 Xem lời giải</summary>

```js
const { customerName, totalPrice } = donHang;
const { _id, ...thongTin } = donHang;
const { customerName: tenKhach } = donHang;
```

</details>

### Bước 2 — Luyện phương thức mảng

```js
const gioHang = [
  { id: 1, ten: "Trà sữa trân châu", gia: 35000, soLuong: 2 },
  { id: 2, ten: "Trà đào cam sả",    gia: 40000, soLuong: 1 },
  { id: 3, ten: "Matcha latte",      gia: 45000, soLuong: 3 },
];

// TODO 1: in ra mảng chỉ chứa TÊN các món
// TODO 2: lọc ra các món có giá trên 38000
// TODO 3: tìm món có id = 2
// TODO 4: tính tổng tiền cả giỏ
// TODO 5: xoá món có id = 1 khỏi giỏ
```

<details>
<summary>💡 Xem lời giải</summary>

```js
const tenCacMon = gioHang.map((item) => item.ten);
const monDat = gioHang.filter((item) => item.gia > 38000);
const monCanTim = gioHang.find((item) => item.id === 2);
const tongTien = gioHang.reduce((tong, item) => tong + item.gia * item.soLuong, 0);
const gioMoi = gioHang.filter((item) => item.id !== 1);

console.log(tongTien);  // 275000
```

Câu 4 và câu 5 chính là logic thật của `CheckoutPage.js` và `cartSlice.js`.

</details>

### Bước 3 — Luyện async/await

```js
// Hàm giả lập gọi API, mất 1 giây mới xong
const goiAPI = (ten) =>
  new Promise((resolve) => setTimeout(() => resolve(`Đã lấy ${ten}`), 1000));

// TODO: viết hàm layDuLieu() gọi lần lượt goiAPI("sản phẩm") rồi goiAPI("danh mục"),
//       in kết quả từng cái, và bọc try/catch
```

<details>
<summary>💡 Xem lời giải</summary>

```js
const layDuLieu = async () => {
  try {
    const kq1 = await goiAPI("sản phẩm");
    console.log(kq1);

    const kq2 = await goiAPI("danh mục");
    console.log(kq2);
  } catch (error) {
    console.log("Có lỗi:", error);
  }
};

layDuLieu();
```

Chạy mất **2 giây** vì hai lệnh chờ nhau. Muốn chạy song song còn 1 giây:

```js
const [kq1, kq2] = await Promise.all([goiAPI("sản phẩm"), goiAPI("danh mục")]);
```

Kỹ thuật `Promise.all` này chính là thứ sẽ sửa được bug đặt hàng ở
[Bài 28](28-thanh-toan.md).

</details>

---

## 6. ✅ Kiểm chứng kết quả

Đọc đoạn code thật sau — trích từ `yotea-fe/src/redux/productSlice.js:9-19` — và trả
lời 5 câu hỏi bên dưới **không cần chạy thử**:

```js
export const getProducts = createAsyncThunk(
  "product/getProducts",
  async ({ start, limit }) => {
    const { data } = await getAll();
    const totalProduct = data.length;

    const { data: productsData } = await getAll(start, limit);

    return { totalProduct, productsData };
  }
);
```

1. `{ start, limit }` ở tham số là kỹ thuật gì?
2. Vì sao `getAll()` được gọi hai lần?
3. `const { data: productsData }` nghĩa là gì?
4. `return { totalProduct, productsData }` dùng cú pháp rút gọn nào?
5. Hàm này chạy mất bao lâu nếu mỗi lần gọi API mất 200 ms?

<details>
<summary>💡 Xem đáp án</summary>

1. **Destructuring trên tham số** — hàm nhận vào một object, bóc luôn hai trường `start` và `limit`.
2. Lần đầu **không truyền tham số** → lấy **toàn bộ** sản phẩm để **đếm tổng** (phục vụ phân trang). Lần hai truyền `start`/`limit` → lấy đúng **một trang**.
3. Lấy thuộc tính `data` và **đổi tên** thành `productsData`, tránh trùng với biến `data` ở trên.
4. **Object shorthand** — viết đủ sẽ là `{ totalProduct: totalProduct, productsData: productsData }`.
5. Khoảng **400 ms**, vì hai lệnh `await` chạy **nối tiếp**.

**Câu hỏi thưởng:** cách gọi 2 lần này có tệ không? Có — tải **toàn bộ** sản phẩm chỉ
để đếm là rất lãng phí khi dữ liệu lớn. Cách chuẩn là backend trả về tổng số trong
header hoặc trong body. Ta sẽ bàn ở [Bài 25](25-danh-sach-san-pham.md).

> ⚠️ **Ghi chú trung thực:** đoạn code trên là code thật trong dự án, nhưng thực tế
> **không trang nào gọi tới nó** — `productSlice.js` đã bị thay thế bởi cách gọi axios
> trực tiếp và bởi RTK Query. Ta vẫn dùng nó làm bài đọc hiểu vì nó gói gọn rất nhiều
> cú pháp cần học. Chuyện "code chết" này sẽ được nói kỹ ở [Bài 20](20-async-thunk.md).

</details>

---

## 7. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Cannot read properties of undefined (reading 'x')` | Truy cập thuộc tính của biến `undefined` | Dùng `?.` hoặc kiểm tra `if (obj)` trước |
| `x is not a function` | Import sai kiểu (nhầm default với named) | Kiểm tra lại có/không có `{ }` khi import |
| `await is only valid in async functions` | Dùng `await` trong hàm không có `async` | Thêm `async` vào trước tham số hàm |
| `Unexpected token 'export'` | Chạy file ES6 bằng `node` thuần | Backend phải chạy qua `babel-node` (dùng `npm start`) |
| Hàm trả về `undefined` bất ngờ | Arrow function có `{ }` mà quên `return` | Thêm `return`, hoặc bỏ `{ }` để tự động return |
| `forEach` với `await` không chờ | `forEach` không hỗ trợ bất đồng bộ | Dùng `for...of` hoặc `Promise.all` |

---

## 8. 📝 Bài tập

**Bài 1.** Đoạn code sau lấy từ `yotea-be/src/middlewares/checkAuth.js`. Giải thích
`req.profile` và `req.auth` từ đâu mà có, và dòng `if (!status)` kiểm tra điều gì.

```js
export const isAuth = (req, res, next) => {
  const status = req.profile._id == req.auth._id;

  if (!status) {
    res.status(400).json({ message: "Bạn không có quyền truy cập" });
  } else {
    next();
  }
};
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

- `req.auth` do middleware `requireSignin` (thư viện `express-jwt`) gắn vào sau khi
  **giải mã token** thành công. Nó chứa `_id` của người **thật sự đang đăng nhập**.
- `req.profile` do hàm `userById` gắn vào (qua `router.param("userId", userById)`).
  Nó chứa user tra được từ **`:userId` trên URL**.
- Dòng `if (!status)` kiểm tra: *"id trong token có khớp id trên URL không?"* Nếu không
  khớp nghĩa là ai đó đang dùng token của mình để thao tác trên tài khoản người khác.
- `next()` nghĩa là "qua chốt, đi tiếp tới middleware/controller kế".

Để ý dấu `==` (hai bằng) chứ không phải `===`. Ở đây là **cố ý**, vì `req.profile._id`
là **ObjectId** còn `req.auth._id` là **chuỗi**; `==` sẽ tự chuyển kiểu giúp. Tuy chạy
đúng nhưng đây là cách viết dễ gây hiểu nhầm — chuẩn hơn là
`req.profile._id.toString() === req.auth._id`.

</details>

**Bài 2.** Viết lại đoạn `.then()` sau bằng `async/await` cho dễ đọc
(trích `yotea-fe/src/redux/authSlice.js`):

```js
return update(dataAuth).then(async () => {
  const { data: { password, ...rest } } = await get(dataAuth._id);
  return rest;
});
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```js
await update(dataAuth);

const {
  data: { password, ...rest },
} = await get(dataAuth._id);

return rest;
```

Ngắn hơn, phẳng hơn, dễ đặt breakpoint hơn. Ý nghĩa: cập nhật thông tin user xong thì
**tải lại** dữ liệu mới nhất từ server, loại bỏ trường `password` trước khi đưa vào Redux.

</details>

**Bài 3.** Với URL sau, hãy chỉ ra đâu là path params, đâu là query string, và backend
sẽ đọc từng phần bằng biến nào:

```
GET http://localhost:8080/api/products/?categoryId=6650abc&_sort=price&_order=asc&_start=0&_limit=8
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

- **Path**: `/api/products/` — không có path param nào (khớp `router.get("/products", list)`).
- **Query string**: mọi thứ sau dấu `?`. Backend đọc bằng `req.query`:

| Key | `req.query` | Ý nghĩa |
|---|---|---|
| `categoryId` | `req.query.categoryId` | Chỉ lấy sản phẩm thuộc danh mục này |
| `_sort` | `req.query["_sort"]` | Sắp xếp theo trường `price` |
| `_order` | `req.query["_order"]` | Tăng dần |
| `_start` | `req.query["_start"]` | Bỏ qua 0 bản ghi đầu |
| `_limit` | `req.query["_limit"]` | Lấy tối đa 8 bản ghi |

Đây chính xác là request mà trang thực đơn bắn đi khi bạn lọc theo danh mục và sắp
xếp theo giá. Cách backend bóc tách đống query này là nội dung của
[Bài 09](09-bo-loc-query.md).

</details>

---

## 📌 Tóm tắt

- **Destructuring** là cú pháp quan trọng nhất cần nắm — nó có mặt ở gần như mọi file của dự án.
- `...` ở vế trái `=` là **rest** (gom vào), ở vế phải là **spread** (trải ra).
- **Mọi** thao tác với database/API đều bất đồng bộ → luôn có `async` + `await` + `try...catch`.
- HTTP request gồm 5 phần: method, path params (`req.params`), query string (`req.query`), headers, body (`req.body`).
- Status code: **2xx thành công · 4xx client gửi sai · 5xx server hỏng**.
- **REST** = URL là danh từ (tài nguyên), method là động từ (hành động).
- MongoDB lưu **document** giống hệt object JavaScript; **Mongoose** thêm lớp Schema để kiểm soát cấu trúc.
- `forEach` **không** chờ `await` — bẫy này gây bug thật trong dự án ([Bài 28](28-thanh-toan.md)).

**Từ khoá tra cứu thêm:** `ES6 destructuring`, `spread operator`, `async await javascript`, `HTTP status codes`, `RESTful API`, `MongoDB document model`, `Mongoose schema`

➡️ **Bài tiếp theo:** [04 — Mổ xẻ `app.js`: Express, middleware, CORS, morgan](04-express-va-appjs.md) — bắt đầu đi sâu vào backend, hiểu từng dòng của file khởi động server.
