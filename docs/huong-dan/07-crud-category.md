# Bài 07 — CRUD trọn vẹn với Category + test bằng Postman

> **Phần 1 · Backend** — Thời lượng ước tính: **~75 phút**
> ⬅️ Bài trước: [06 — Vòng đời một request](06-vong-doi-mot-request.md) · Bài sau: [08 — Slug thân thiện SEO với slugify](08-slug-slugify.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Ánh xạ được: CRUD ↔ HTTP method ↔ hàm Mongoose ↔ tên hàm controller trong Yotea.
- Đọc hiểu **từng dòng** cả 5 hàm trong `yotea-be/src/controllers/category.js`.
- Phân biệt `find`/`findOne`/`findById`, `updateOne`/`findOneAndUpdate`, `deleteOne`/`findOneAndDelete`.
- Giải thích được vì sao thiếu `{ new: true }` thì API cập nhật trả về **dữ liệu cũ**.
- Dùng **Postman**: tạo Collection, gửi body JSON, đọc status code, lưu request dùng lại.
- Tự viết **đủ 5 hàm CRUD** cho chức năng Topping của bạn và test hết bằng Postman.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 06 — Vòng đời một request](06-vong-doi-mot-request.md).
- MongoDB đang chạy, `yotea-be` khởi động được bằng `npm start` ([Bài 02](02-cai-dat-moi-truong.md)).
- **Postman** bản Desktop ([postman.com/downloads](https://www.postman.com/downloads/)).
- Trong database có ít nhất một tài khoản **admin** (`role: 1`) để lấy token.

> 💡 Ở [Bài 06](06-vong-doi-mot-request.md) bạn đã dựng `routes/topping.js` + `controllers/topping.js`
> với **đúng một** endpoint `GET /api/toppings` và mount vào `app.js`. Bài này ta làm tiếp:
> bổ sung **4 thao tác còn lại** để Topping có CRUD trọn vẹn, rồi test tử tế bằng Postman.

---

## 1. CRUD là gì?

**CRUD** = **C**reate (thêm) · **R**ead (đọc) · **U**pdate (sửa) · **D**elete (xoá) — 4 việc mà
mọi phần mềm quản lý dữ liệu đều phải làm. Như một quán trà sữa với cuốn thực đơn: thêm món
mới, cho khách xem, đổi giá, gạch món ngừng bán. 90% code backend của một web bán hàng chỉ là
CRUD lặp lại cho từng loại dữ liệu.

Chữ **R** thực tế tách làm **hai** việc: đọc **một** bản ghi và đọc **danh sách**. Nên trong
Yotea (và hầu hết dự án Express) có **5** hàm controller chứ không phải 4:

| Việc | HTTP method | Đường dẫn trong Yotea | Hàm Mongoose | Tên hàm controller |
|---|---|---|---|---|
| **C** — Thêm mới | `POST` | `/api/category/:userId` | `new Model(...).save()` | `create` |
| **R** — Đọc **1** bản ghi | `GET` | `/api/category/:slug` | `findOne(filter)` | `read` |
| **R** — Đọc **danh sách** | `GET` | `/api/category` | `find(filter)` | `list` |
| **U** — Cập nhật | `PUT` | `/api/category/:id/:userId` | `findOneAndUpdate(...)` | `update` |
| **D** — Xoá | `DELETE` | `/api/category/:id/:userId` | `findOneAndDelete(filter)` | `remove` |

Bộ tên `create / read / list / update / remove` **lặp lại y hệt ở cả 14 controller** của dự án.
Nắm được `category.js` là bạn đọc hiểu luôn `product.js`, `slider.js`, `news.js`…

> 💡 **Mẹo nhớ:** tên hàm là **`remove`** chứ không phải `delete`, vì `delete` là **từ khoá có
> sẵn** của JavaScript (`delete obj.key`) — không đặt tên hàm trùng được.

---

## 2. Soi code thật: `controllers/category.js`

### 2.1. Trước hết, xem bảng route

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
| 8 | `router.post("/category/:userId", …, create)` | Thêm danh mục. 3 middleware chặn trước: phải có token, token khớp `:userId`, phải là admin |
| 9-10 | `router.get(…, read)` / `router.get("/category", list)` | Hai API đọc, **không** middleware → ai cũng gọi được |
| 11-12 | `router.put` / `router.delete` | `:id` là id danh mục, `:userId` là id người đang đăng nhập |
| 14 | `router.param("userId", userById)` | Hễ URL có `:userId` thì chạy `userById` trước, gắn kết quả vào `req.profile` |

Router này được mount ở `yotea-be/src/app.js:29` bằng `app.use("/api", categoryRouter)`
([Bài 04](04-express-va-appjs.md)) — nên đường dẫn thật là `/api/category`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dòng 2 import `getProductByCate`, nhưng đọc hết 151
> dòng `controllers/category.js` thì **không hề có** export nào tên như vậy. Biến này nhận
> `undefined` và không dùng ở đâu — code thừa. Không sập server chỉ vì Babel biên dịch ESM
> sang CommonJS; chạy ESM thuần sẽ lỗi ngay lúc khởi động.

### 2.2. `create` — thêm mới

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
| 6 | `req.body.slug = slugify(req.body.name)` | Tự sinh slug từ tên — client **không cần** gửi slug ([Bài 08](08-slug-slugify.md)) |
| 9 | `await new CategoryModel(req.body).save()` | **Hai bước gộp một** — xem ngay dưới |
| 10 | `res.json(category)` | Trả về document **vừa lưu** (đã có `_id`, `createdAt`, `updatedAt`) |
| 12-15 | `res.status(400).json({…})` | Mã 400 kèm thông báo tiếng Việt và object lỗi gốc |

Dòng 9 làm hai việc:

```js
// đoạn này chỉ để minh hoạ — dự án viết gộp một dòng
const category = new CategoryModel(req.body);  // ① tạo document trong RAM, CHƯA chạm database
await category.save();                          // ② ghi xuống MongoDB, CHỜ xong mới đi tiếp
```

Bước ① là lúc **Mongoose kiểm tra Schema**: thiếu `name`? sai kiểu? có trường lạ? Vì có bước
①, nếu bạn gửi `{"name": "Trà sữa", "abc": 123}` thì `abc` **bị âm thầm loại bỏ** — Mongoose
mặc định chạy chế độ `strict`, chỉ giữ trường có khai báo trong Schema.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dòng 6 nằm **ngoài** `try`. Client gửi body thiếu
> `name` → `slugify(undefined)` ném lỗi mà **không ai bắt** → Express 4 không xử lý được async
> rejection → **Node kill cả process, server sập**. Cách đúng: đưa dòng 6 vào trong `try`. Mổ
> kỹ ở [Bài 08](08-slug-slugify.md).

### 2.3. `read` — đọc một bản ghi

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
| 20 | `const filter = { slug: req.params.slug }` | **Tìm theo `slug`, KHÔNG phải `_id`** — đây là câu trả lời cho "dự án tìm theo gì" |
| 21 | `req.query["_expand"]` | Nếu URL có `?_expand=xxx` thì "nở" trường quan hệ đó ra |
| 24 | `.findOne(filter)` | Lấy **document đầu tiên** khớp filter |
| 24 | `.select("-__v")` | Bỏ trường kỹ thuật `__v` (dấu `-` = "loại trường này") |
| 24 | `.populate(populate).exec()` | `undefined` thì Mongoose bỏ qua; `.exec()` là nút "chạy thật" |
| 25 | `res.json(category)` | Trả về **một object**, không phải mảng |

Vì sao tìm theo `slug`? Vì URL trên trình duyệt là `/danh-muc/tra-sua` chứ không phải
`/danh-muc/6650a1f2c4e8b91234abcd01`. Frontend lấy luôn cái đuôi trong URL đem hỏi backend.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** không tìm thấy thì `findOne` trả `null`, và
> `res.json(null)` gửi đi với status **200 OK**. Chuẩn REST phải là **404 Not Found**. Hậu
> quả: frontend tưởng gọi thành công rồi nổ `Cannot read properties of null` khi vẽ giao diện.
> Sửa: `if (!category) return res.status(404).json({ message: "Không tìm thấy danh mục" });`

### 2.4. `list` — đọc danh sách

Hàm này **dài 85 dòng** (34→118) vì gánh cả "bộ máy lọc" mô phỏng json-server. Bài này chỉ
nhìn phần thực thi truy vấn; khúc lọc dành trọn cho [Bài 09](09-bo-loc-query.md).

`yotea-be/src/controllers/category.js:92-111`

```js
    try {
        const categories = await CategoryModel
            .find(filter)
            .select("-__v")
            .populate(populate)
            .skip(start)
            .limit(limit)
            .sort(sortOpt)
            .exec();

        const newCates = [];
        for await (let cate of categories) {
            const products = await Product.find({ categoryId: cate._id }).exec();
            const { _doc } = cate;
            newCates.push({
                ..._doc,
                products
            })
        }
        res.json(newCates);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 94 | `.find(filter)` | Trả về **MẢNG** mọi document khớp. `find({})` = lấy tất |
| 97-99 | `.skip()` `.limit()` `.sort()` | Bỏ qua N bản ghi đầu · lấy tối đa N bản ghi · sắp xếp |
| 102-110 | vòng `for await` | **Chỉ `category.js` mới có**: mỗi danh mục được nhét thêm mảng sản phẩm thuộc nó |
| 111 | `res.json(newCates)` | Trả mảng đã nhồi thêm `products` |

Nhờ vòng lặp này mà `GET /api/category` trả về mỗi danh mục **kèm luôn** sản phẩm bên trong —
trang chủ Yotea dùng đúng dữ liệu đó.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** đây là **truy vấn N+1** — 1 lần lấy danh mục + N lần
> lấy sản phẩm, chạy **tuần tự**. 10 danh mục là 11 lượt đi/về database. Cách đúng dùng
> `$lookup` (aggregation) hoặc virtual populate. Ngoài ra `cate._doc` là **thuộc tính riêng
> tư** của Mongoose, đúng ra phải viết `cate.toObject()`.

### 2.5. `update` — và bí mật của `{ new: true }`

`yotea-be/src/controllers/category.js:120-137`

```js
export const update = async (req, res) => {
    const filter = { _id: req.params.id };
    const update = {
        ...req.body,
        slug: slugify(req.body.name)
    };
    const options = { new: true };

    try {
        const category = await CategoryModel.findOneAndUpdate(filter, update, options).exec();
        res.json(category);
    } catch (error) {
        res.status(400).json({
            message: "Cập nhật danh mục thất bại",
            error
        });
    }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 121 | `const filter = { _id: req.params.id }` | **Sửa thì tìm theo `_id`**, không phải slug (khác `read`!) |
| 122-125 | `{ ...req.body, slug: slugify(…) }` | Trải toàn bộ dữ liệu client gửi lên, **rồi ghi đè** `slug` bằng slug sinh từ tên mới |
| 126 | `const options = { new: true }` | ⭐ Ngôi sao của bài này |
| 129 | `findOneAndUpdate(filter, update, options)` | Ba tham số: **tìm ai · sửa gì · tuỳ chọn** |

#### Không có `{ new: true }` thì sao?

`findOneAndUpdate` mặc định trả về **bản ghi TRƯỚC khi sửa** — phản trực giác, nhưng đó là
hành vi kế thừa từ lệnh `findAndModify` của MongoDB. Giả sử danh mục đang là
`{ name: "Trà sữa" }` và bạn gửi lên `{ name: "Trà sữa nóng" }`:

| Option | Database sau lệnh | API trả về cho client |
|---|---|---|
| `{ new: true }` (dự án dùng) | `name = "Trà sữa nóng"` ✅ | `{ name: "Trà sữa nóng" }` ✅ **đúng ý** |
| Không truyền options | `name = "Trà sữa nóng"` ✅ | `{ name: "Trà sữa" }` ❌ **bản CŨ!** |

Điểm chết người: **database vẫn được sửa đúng trong cả hai trường hợp**, chỉ có **phản hồi**
là sai. Nên bug này rất khó phát hiện — bạn bấm Lưu, giao diện vẫn hiện tên cũ, tưởng "lưu
không được", F5 lại thì thấy tên mới. Kiểu bug làm người ta mất cả buổi chiều.

> 💡 **Khi nào bạn *muốn* bản cũ?** Khi cần ghi log kiểu "giá đổi từ X thành Y" — lúc đó bỏ
> `{ new: true }` để lấy giá cũ, còn giá mới đã có sẵn trong `req.body`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `options` chỉ có `{ new: true }`, **thiếu
> `runValidators: true`**. Mặc định Mongoose **KHÔNG chạy validate Schema** khi
> `findOneAndUpdate` — bạn hoàn toàn có thể `PUT` một danh mục với `name` rỗng, điều mà lúc
> `create` sẽ bị chặn ngay. Cách đúng: `{ new: true, runValidators: true }`.

### 2.6. `remove` — xoá

`yotea-be/src/controllers/category.js:139-151`

```js
export const remove = async (req, res) => {
    const filter = { _id: req.params.id };

    try {
        const category = await CategoryModel.findOneAndDelete(filter).exec();
        res.json(category);
    } catch (error) {
        res.status(400).json({
            message: "Xóa danh mục không thành công",
            error
        });
    }
};
```

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 140 | `const filter = { _id: req.params.id }` | Xoá theo `_id` |
| 143 | `findOneAndDelete(filter)` | Xoá document đầu tiên khớp filter, và **trả về chính nó** |
| 144 | `res.json(category)` | Client biết mình vừa xoá cái gì |

Đây là **xoá cứng** (hard delete) — dữ liệu biến mất, không khôi phục được. Nhiều hệ thống
thật dùng **xoá mềm**: chỉ set `deletedAt` rồi lọc bỏ khi query.

> ⚠️ **Chỗ này dự án làm chưa chuẩn — LỖI NẶNG NHẤT của bài:** API xoá danh mục **không hề
> kiểm tra** xem còn sản phẩm nào thuộc danh mục đó hay không.
>
> Nhìn `yotea-be/src/models/product.js:30-34`:
>
> ```js
>     categoryId: {
>         type: ObjectId,
>         ref: "Category",
>         required: true,
>     },
> ```
>
> Mỗi sản phẩm **bắt buộc** trỏ tới một danh mục. Nhưng khi bạn xoá danh mục "Trà sữa",
> MongoDB **không** tự xoá hay cập nhật các sản phẩm đang trỏ tới nó (khác hẳn SQL có khoá
> ngoại `ON DELETE CASCADE`). Kết quả: hàng chục sản phẩm thành **"mồ côi"** — `categoryId`
> vẫn giữ một id trỏ vào hư không.
>
> **Hậu quả nhìn thấy trên giao diện:** frontend gọi `GET /api/products?_expand=categoryId`,
> `populate` không tìm ra danh mục nên gán `categoryId = null`. Ở
> `yotea-fe/src/components/admin/ListProduct.js:121`:
>
> ```jsx
>               {item.categoryId?.name}
> ```
>
> Nhờ dấu `?.` nên trang không sập, nhưng **cột "Danh mục" hiện trống trơn**. Tệ hơn, ở
> `yotea-fe/src/pages/user/ProductDetailPage.js:180` link breadcrumb thành
> `/danh-muc/undefined` — bấm vào ra trang trắng.
>
> **Cách làm đúng** (chọn 1 trong 3): (1) **chặn xoá** nếu
> `Product.countDocuments({ categoryId })` lớn hơn 0 — xem Bài tập 3; (2) **xoá dây chuyền**
> (nguy hiểm, phải hỏi lại người dùng); (3) **xoá mềm**. Yotea không làm cái nào. Nói tiếp về
> `populate` ở [Bài 10](10-quan-he-va-populate.md).

---

## 3. Các hàm Mongoose hay bị nhầm

| Hàm | Trả về | Ghi chú |
|---|---|---|
| `find(filter)` | **Mảng** (rỗng `[]` nếu không có gì) | Dùng cho `list` |
| `findOne(filter)` | **Một object** hoặc `null` | Lọc theo điều kiện bất kỳ — Yotea dùng `{ slug }` cho `read` |
| `findById(id)` | **Một object** hoặc `null` | **Chỉ tìm theo `_id`**; viết tắt của `findOne({ _id: id })`, dùng ở `userById` |
| `updateOne(filter, update)` | **Báo cáo** `{ matchedCount, modifiedCount }` — **KHÔNG có dữ liệu** | Khi chỉ cần biết "sửa xong chưa" |
| `findOneAndUpdate(filter, update, options)` | **Document** (cũ hay mới tuỳ `{ new: true }`) | API phải trả dữ liệu về → Yotea dùng cái này |
| `deleteOne(filter)` | `{ deletedCount: 1 }` — **KHÔNG có dữ liệu** | Xoá xong là thôi |
| `findOneAndDelete(filter)` | **Document vừa bị xoá** | Cần trả lại "cái vừa xoá" → Yotea dùng cái này |

> 💡 **Quy luật đặt tên dễ nhớ:** hàm bắt đầu bằng **`findOneAnd…`** thì **trả về document**.
> Hàm bắt đầu bằng **`update`/`delete`** chỉ trả về **con số thống kê**.

**`.exec()` để làm gì?** `find()` trả về một **Query** (bản thiết kế câu truy vấn), chưa chạy.
`.exec()` biến nó thành **Promise thật**. Không gọi `.exec()` thì `await` vẫn chạy được vì
Query có sẵn `.then()`. Yotea luôn gọi `.exec()` — thói quen tốt, stack trace khi lỗi rõ hơn nhiều.

---

## 4. Vì sao mọi hàm đều `try/catch` và trả `res.status(400)`?

Cả 5 hàm ở trên có **chung một khuôn**: ① chuẩn bị `filter`/`update`/`options` → ② `try` gọi
database → ③ `res.json(kết quả)` → ④ `catch` thì `res.status(400).json({ message, error })`.

**Vì sao bắt buộc `try/catch`?** Vì `await` gặp lỗi thì **ném exception**. Trong hàm `async`,
exception không bắt sẽ thành **unhandled promise rejection**. Express 4 **không** bắt được
loại lỗi này: nhẹ thì request treo tới khi timeout, nặng thì (Node ≥ 15, tức mọi máy hiện nay)
**Node giết luôn process** → server sập, mọi người dùng khác chết theo.

Một truy vấn database có thể lỗi vì rất nhiều lý do đời thường:

| Tình huống | Lỗi Mongoose ném ra |
|---|---|
| Gửi thiếu trường `name` (đang `required`) | `ValidationError: Path 'name' is required` |
| Tạo danh mục trùng slug (đang `unique`) | `MongoServerError: E11000 duplicate key error` |
| Truyền `:id` không phải chuỗi 24 ký tự hex | `CastError: Cast to ObjectId failed` |
| MongoDB chưa bật | `MongooseServerSelectionError` |

**Vì sao là 400?** Nhắc lại [Bài 03](03-kien-thuc-nen.md): **4xx = lỗi tại client**, **5xx =
lỗi tại server**. Đa số lỗi trên đúng là do client gửi sai → 400 hợp lý.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** dự án gộp **tất cả** vào 400. Nhưng
> `MongooseServerSelectionError` (MongoDB chết) rõ ràng là lỗi **của server**, phải trả 500 —
> trả 400 khiến người dùng ngồi sửa dữ liệu trong khi vấn đề thật là database chưa bật. Ngoài
> ra `error` được trả **nguyên xi** cho client, lộ tên collection, tên trường, đường dẫn file
> trên server. Bàn tiếp ở [Bài 33](33-ra-soat-bao-mat.md), [Bài 34](34-refactor-du-an.md).

> ⚠️ **Backend gần như KHÔNG validate gì ngoài Schema.** Toàn bộ "kiểm tra dữ liệu đầu vào"
> của Yotea chỉ có ràng buộc trong file model (`required`, `unique`, `type`) — không hề có
> `express-validator`, `joi` hay `yup` ở backend. Hệ quả rất thật: `name` có thể là `"   "`
> (toàn khoảng trắng — `required` vẫn cho qua vì chuỗi khác rỗng); `price` sản phẩm có thể là
> **số âm** (Schema không có `min: 0`); `ratingNumber` có thể là **1000 sao**
> (`yotea-be/src/models/rating.js`). Và với `findOneAndUpdate` thì **ngay cả ràng buộc Schema
> cũng không chạy** vì thiếu `runValidators`.
>
> Frontend có validate bằng `react-hook-form` + `yup` ([Bài 28](28-thanh-toan.md)), nhưng
> **validate phía client chỉ để trải nghiệm đẹp** — ai cũng mở Postman gọi thẳng API được.
> **Quy tắc vàng: không bao giờ tin dữ liệu từ client.**

---

## 5. Postman — cẩm nang cho người mới

Postman là "trình duyệt dành cho API". Trình duyệt thường chỉ gửi được `GET`; Postman gửi được
đủ `POST`, `PUT`, `DELETE`, kèm body JSON và header token.

**Tạo Collection `Yotea`:** mở Postman → thanh trái chọn tab **Collections** → bấm **+**
(hoặc **Create Collection**) → gõ **`Yotea`** rồi Enter. Chuột phải vào `Yotea` → **Add
folder** → tạo 2 thư mục con `Auth` và `Category` cho gọn.

**Tạo một request:** chuột phải thư mục `Category` → **Add request** → đặt tên có nghĩa
(`GET danh sách danh mục`) → chọn **method** ở ô dropdown bên trái, gõ **URL** vào ô dài bên
cạnh → bấm **Send** (nút xanh) → **bấm `Ctrl + S` để lưu**. Không lưu thì đóng tab là mất, lần
sau phải gõ lại từ đầu.

**Gửi body JSON** (bắt buộc với `POST`/`PUT`), đúng 3 thao tác:

1. Chọn tab **Body** (ngay dưới ô URL).
2. Chọn ô tròn **raw**.
3. Dropdown bên phải (mặc định `Text`) → đổi thành **JSON**.

Postman sẽ tự thêm header `Content-Type: application/json` — đúng thứ mà
`app.use(express.json())` ở `yotea-be/src/app.js:25` chờ để bóc body thành `req.body`.

> 💡 **Lỗi kinh điển của người mới:** quên đổi `Text` → `JSON`. Khi đó `express.json()` không
> xử lý, `req.body` thành `{}` rỗng, backend báo "Thêm danh mục thất bại" mà bạn không hiểu vì
> sao. Nhớ: **raw + JSON**.

**Đọc status code ở đâu?** Sau khi **Send**, nhìn **góc trên bên phải khung Response**:
`Status: 200 OK · Time: 43 ms · Size: 512 B`.

| Bạn thấy | Nghĩa là | Làm gì tiếp |
|---|---|---|
| `200 OK` | Thành công | 🎉 |
| `400 Bad Request` | Body sai / thiếu / id sai định dạng | Đọc `message` trong response, sửa body |
| `401 Unauthorized` | Chưa gửi token, token sai hoặc hết hạn | Đăng nhập lại lấy token mới |
| `404 Not Found` | Gõ sai URL (thiếu `/api`? `category` viết thành `categories`?) | Soi lại URL |
| `Error: connect ECONNREFUSED` | Backend chưa chạy | Về terminal `yotea-be` chạy `npm start` |

### 5.1. Đủ 5 request mẫu với Category

Bật backend trước. Mở terminal, **đứng tại thư mục `yotea-be`**, chạy `npm start`. Thấy
`App is running on port: 8080` và `Connected to MongoDB` là ổn.

#### Request 0 (bắt buộc trước) — Đăng nhập lấy token

Ba API ghi dữ liệu đều bị chặn bởi `requireSignin, isAuth, isAdmin`, nên cần **token** và
**`_id` của tài khoản admin**.

```
POST http://localhost:8080/api/signin      ·      Body → raw → JSON
```

```json
{ "email": "admin@gmail.com", "password": "123456" }
```

Response mẫu:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NjUwYTFmMmM0ZThiOTEyMzRhYmNkNzcifQ.xxxxx",
  "user": { "_id": "6650a1f2c4e8b91234abcd77", "email": "admin@gmail.com", "role": 1, "active": 1 }
}
```

**Chép ra giấy nháp 2 thứ:** `token` và `user._id` (chính là **`:userId`** trên URL — ở đây
`6650a1f2c4e8b91234abcd77`). Cách gắn token: tab **Authorization** → **Type** chọn **Bearer
Token** → dán chuỗi token vào ô **Token** (dán **không kèm** chữ `Bearer`, Postman tự thêm).

> 🔒 **Ghi chú bảo mật:** token hết hạn sau **3 giờ** (`expiresIn: "3h"` tại
> `yotea-be/src/controllers/auth.js:53`). Hôm sau mở Postman thấy `401` thì đăng nhập lại.

#### Request 1 — **CREATE**

```
POST http://localhost:8080/api/category/6650a1f2c4e8b91234abcd77
Authorization → Bearer Token → <token>      ·      Body → raw → JSON
```

```json
{ "name": "Trà trái cây", "image": "https://res.cloudinary.com/demo/tra-trai-cay.png" }
```

Response — **`200 OK`**:

```json
{
  "name": "Trà trái cây",
  "slug": "tra-trai-cay",
  "image": "https://res.cloudinary.com/demo/tra-trai-cay.png",
  "_id": "6650b3a1c4e8b91234abce01",
  "createdAt": "2026-08-15T03:12:44.812Z",
  "updatedAt": "2026-08-15T03:12:44.812Z",
  "__v": 0
}
```

Để ý: bạn **không gửi** `slug` mà nó vẫn có (dòng 6 của controller sinh ra); `_id`,
`createdAt`, `updatedAt` do MongoDB/Mongoose tự thêm. **Chép lại `_id`** cho request 4 và 5.

#### Request 2 — **READ** (theo slug, không cần token)

```
GET http://localhost:8080/api/category/tra-trai-cay
```

Response — **`200 OK`**, đúng object trên nhưng **không còn `__v`** (công của `.select("-__v")`).

**Thử sai cho vui:** đổi URL thành `/api/category/khong-ton-tai` → bạn nhận **`200 OK`** với
nội dung đúng một chữ `null`. Chính là lỗi đã nêu ở mục 2.3.

#### Request 3 — **LIST**

```
GET http://localhost:8080/api/category
```

Response — **`200 OK`**:

```json
[
  {
    "_id": "6650b3a1c4e8b91234abce01",
    "name": "Trà trái cây",
    "slug": "tra-trai-cay",
    "createdAt": "2026-08-15T03:12:44.812Z",
    "updatedAt": "2026-08-15T03:12:44.812Z",
    "products": []
  }
]
```

Mảng `products` rỗng vì danh mục mới toanh — vòng `for await` ở dòng 102-110 đã chạy và nhét vào.

#### Request 4 — **UPDATE**

```
PUT http://localhost:8080/api/category/6650b3a1c4e8b91234abce01/6650a1f2c4e8b91234abcd77
Authorization → Bearer Token → <token>      ·      Body → raw → JSON
```

```json
{ "name": "Trà trái cây tươi", "image": "https://res.cloudinary.com/demo/tra-trai-cay-v2.png" }
```

Response — **`200 OK`**:

```json
{
  "_id": "6650b3a1c4e8b91234abce01",
  "name": "Trà trái cây tươi",
  "slug": "tra-trai-cay-tuoi",
  "createdAt": "2026-08-15T03:12:44.812Z",
  "updatedAt": "2026-08-15T03:20:07.455Z",
  "__v": 0
}
```

Tên **mới** hiện ra ngay — `{ new: true }` đang làm việc. `updatedAt` đã đổi, `createdAt` giữ nguyên.

#### Request 5 — **DELETE** (không cần Body)

```
DELETE http://localhost:8080/api/category/6650b3a1c4e8b91234abce01/6650a1f2c4e8b91234abcd77
Authorization → Bearer Token → <token>
```

Response — **`200 OK`** trả về chính document vừa bị xoá. Gọi **lần thứ hai** cùng URL đó →
vẫn `200 OK` nhưng nội dung là `null` (không còn gì để xoá).

---

## 6. 🛠️ Tự tay làm — CRUD trọn vẹn cho Topping

> Mục tiêu: cuối phần này `controllers/topping.js` của bạn có **đủ 5 hàm**
> `create / read / list / update / remove`, `routes/topping.js` có **đủ 5 route**, và bạn đã tự
> chạy đủ 5 request Postman.

**Nhắc lại mạch bài:** [Bài 05](05-mongoose-model.md) bạn tự viết `yotea-be/src/models/topping.js`;
[Bài 06](06-vong-doi-mot-request.md) bạn tự viết `routes/topping.js` + `controllers/topping.js`
với đúng một endpoint `GET /api/toppings` và mount vào `app.js`. Bài này hoàn thiện nốt 4 thao tác còn lại.

> 📖 **Nhắc lại model của bạn** (bài 05 bạn khai báo hơi khác cũng không sao, miễn có `name` và `price`):
>
> ```js
> // yotea-be/src/models/topping.js  ← file BẠN TỰ TẠO ở Bài 05, dự án gốc KHÔNG có
> const toppingSchema = new Schema(
>   { name: { type: String, required: true },
>     price: { type: Number, required: true },
>     image: String,
>     status: { type: Number, default: 0 } },
>   { timestamps: true }
> );
> ```
>
> Model này **chưa có `slug`** — đó là việc của [Bài 08](08-slug-slugify.md). Nên controller
> hôm nay **không gọi `slugify`**, và `read` sẽ tìm theo `_id`.

### Bước 1 — Thêm hàm `create`

Mở `yotea-be/src/controllers/topping.js` (bạn đã tạo ở bài 06), thêm hàm này **bên trên** hàm `list` sẵn có:

```js
// yotea-be/src/controllers/topping.js  ← code BẠN TỰ VIẾT, dự án gốc không có file này
export const create = async (req, res) => {
  try {
    const topping = await new Topping(req.body).save();
    res.json(topping);
  } catch (error) {
    res.status(400).json({ message: "Thêm topping thất bại", error });
  }
};
```

So với `create` của category: **không có dòng `slugify`**, nên cũng không dính bug "slugify
ngoài try/catch". Bài 08 sẽ thêm slug **đúng cách**.

### Bước 2 — Thêm hàm `read` (theo `_id`)

```js
// yotea-be/src/controllers/topping.js  ← code BẠN TỰ VIẾT
export const read = async (req, res) => {
  const filter = { _id: req.params.id };

  try {
    const topping = await Topping.findOne(filter).select("-__v").exec();

    if (!topping) {
      return res.status(404).json({ message: "Không tìm thấy topping" });
    }

    res.json(topping);
  } catch (error) {
    res.status(400).json({ message: "Không tìm thấy topping", error });
  }
};
```

Ta **cố tình làm tốt hơn dự án**: kiểm tra `null` và trả **404** thay vì `200 null` — đúng
cách sửa lỗi đã nêu ở mục 2.3.

### Bước 3 — Thêm hàm `update` (nhớ `{ new: true }`)

```js
// yotea-be/src/controllers/topping.js  ← code BẠN TỰ VIẾT
export const update = async (req, res) => {
  const filter = { _id: req.params.id };
  const update = req.body;
  const options = { new: true, runValidators: true };

  try {
    const topping = await Topping.findOneAndUpdate(filter, update, options).exec();

    if (!topping) {
      return res.status(404).json({ message: "Không tìm thấy topping" });
    }

    res.json(topping);
  } catch (error) {
    res.status(400).json({ message: "Cập nhật topping thất bại", error });
  }
};
```

Ta thêm cả `runValidators: true` — thứ mà `category.js:126` còn thiếu. Nhờ nó, gửi lên
`{"price": "rẻ lắm"}` sẽ bị chặn bằng `CastError` thay vì lọt vào database.

### Bước 4 — Thêm hàm `remove`

```js
// yotea-be/src/controllers/topping.js  ← code BẠN TỰ VIẾT
export const remove = async (req, res) => {
  const filter = { _id: req.params.id };

  try {
    const topping = await Topping.findOneAndDelete(filter).exec();

    if (!topping) {
      return res.status(404).json({ message: "Không tìm thấy topping để xoá" });
    }

    res.json(topping);
  } catch (error) {
    res.status(400).json({ message: "Xoá topping không thành công", error });
  }
};
```

### Bước 5 — Khai báo đủ 5 route

Mở `yotea-be/src/routes/topping.js` và sửa thành:

```js
// yotea-be/src/routes/topping.js  ← code BẠN TỰ VIẾT, dự án gốc không có file này
import { Router } from "express";
import { create, list, read, remove, update } from "../controllers/topping";

const router = Router();

router.post("/toppings", create);
router.get("/toppings/:id", read);
router.get("/toppings", list);
router.put("/toppings/:id", update);
router.delete("/toppings/:id", remove);

export default router;
```

Ba điều đáng nói:

1. **Thứ tự route quan trọng.** Ở đây hai path khác nhau rõ ràng nên không sao. Nhưng nếu bạn
   thêm `/toppings/search` thì nó **bắt buộc** phải nằm trước `/toppings/:id`, không thì
   Express hiểu `search` là một cái `:id`.
2. Ta dùng **số nhiều** `/toppings` cho chuẩn REST, khác `/category` số ít của dự án.
3. **Chưa gắn middleware phân quyền** — cố ý. [Bài 12](12-phan-quyen-middleware.md) mới khoá
   các route ghi bằng `requireSignin`/`isAuth`/`isAdmin`. Giờ để trống cho dễ test.

> ⚠️ Nhớ kiểm tra `yotea-be/src/app.js` đã có `import toppingRouter from "./routes/topping";`
> và `app.use("/api", toppingRouter);` (bạn thêm ở bài 06). Thiếu là mọi request đều 404.

---

## 7. ✅ Kiểm chứng kết quả

Đứng tại thư mục `yotea-be`, khởi động lại server:

```bash
npm start
```

Trong Postman tạo thư mục `Topping` trong Collection `Yotea`, rồi chạy **lần lượt** 5 request
(URL đầy đủ là `http://localhost:8080` + đường dẫn):

| # | Request | Kết quả **phải** thấy |
|---|---|---|
| 1 | `POST /api/toppings` — Body raw JSON `{"name": "Trân châu đen", "price": 5000, "status": 1}` | `200 OK`, JSON có `_id`, `createdAt`, `updatedAt`. **Chép lại `_id`** |
| 2 | `GET /api/toppings/<_id vừa chép>` | `200 OK`, đúng object đó, **không có** `__v` |
| 3 | `GET /api/toppings` | `200 OK`, một **mảng** chứa topping vừa tạo |
| 4 | `PUT /api/toppings/<_id>` — Body `{"price": 7000}` | `200 OK`, `price` là **7000** (không phải 5000!), `updatedAt` đã đổi |
| 5 | `DELETE /api/toppings/<_id>` | `200 OK`, trả về object vừa xoá |

Response mẫu cho request 1:

```json
{
  "name": "Trân châu đen",
  "price": 5000,
  "status": 1,
  "_id": "6650c0d2c4e8b91234abcf10",
  "createdAt": "2026-08-15T04:01:22.104Z",
  "updatedAt": "2026-08-15T04:01:22.104Z",
  "__v": 0
}
```

Sau khi xong, chạy lại **request 3** → phải nhận mảng rỗng `[]`. Chạy lại **request 5** lần
nữa → phải nhận **`404`** với `{"message": "Không tìm thấy topping để xoá"}` (nhờ bạn đã kiểm
tra `null`; viết kiểu dự án thì sẽ trả `200 null`).

**Bài kiểm tra cuối:** gửi `POST /api/toppings` với body **thiếu `price`** —
`{ "name": "Thạch dừa" }` → phải nhận **`400 Bad Request`**, `"message": "Thêm topping thất
bại"`, và trong `error` có chữ `Path 'price' is required`. Đó là Schema đang bảo vệ bạn.

---

## 8. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `Could not get response — ECONNREFUSED` | Backend chưa chạy | Terminal ở `yotea-be`, chạy `npm start` |
| `404 Not Found` kèm HTML dài dòng | Sai URL: thiếu `/api`, sai tên resource, sai method | Đối chiếu file route. Nhớ `/api/category` (số ít) nhưng `/api/products` (số nhiều) |
| `400` + `"Thêm danh mục thất bại"` mà body nhìn đúng | Quên đổi Body từ `Text` sang **JSON** → `req.body` rỗng | Body → raw → dropdown chọn **JSON** |
| `400` + `CastError: Cast to ObjectId failed` | `:id` trên URL không phải chuỗi 24 ký tự hex | Chép lại đúng `_id` từ response trước |
| `400` + `E11000 duplicate key error ... slug` | Tạo danh mục trùng tên → trùng slug (`unique: true`) | Đổi tên khác, hoặc xoá bản ghi cũ |
| `401 Unauthorized` | Chưa gắn token / token hết hạn (3 giờ) / dán kèm chữ `Bearer` | Đăng nhập lại, dán **chỉ** chuỗi token vào ô Bearer Token |
| `400` + `"Bạn không có quyền truy cập"` | `:userId` trên URL khác `_id` trong token (`isAuth` chặn) | Dùng đúng `user._id` trả về lúc signin |
| `401` + `"Bạn không phải là Admin"` | Tài khoản có `role = 0` | Đăng nhập bằng tài khoản `role = 1` |
| Server **tắt hẳn**, terminal in `slugify: string argument expected` | Gửi `POST/PUT category` thiếu `name` → lỗi ngoài `try` | Luôn gửi `name`. Bug thật của dự án, xem mục 2.2 |
| Sửa xong nhưng API trả **dữ liệu cũ** | Thiếu `{ new: true }` trong `findOneAndUpdate` | Thêm option `{ new: true }` |
| `TypeError: Topping is not a constructor` | Quên `import Topping from "../models/topping";` đầu controller | Thêm dòng import |

---

## 9. 📝 Bài tập

**Bài 1.** Đoạn sau là `update` của slider (`yotea-be/src/controllers/slider.js:106-120`). Nó
khác `update` của category ở **đúng một điểm** nào, và vì sao?

```js
export const update = async (req, res) => {
    const filter = { _id: req.params.id };
    const update = req.body;
    const options = { new: true };

    try {
        const slider = await Slider.findOneAndUpdate(filter, update, options).exec();
        res.json(slider);
    } catch (error) {
        res.status(400).json({
            message: "Cập nhật slide thất bại",
            error
        });
    }
};
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Khác ở dòng gán `update`:

- Category (`category.js:122-125`): `{ ...req.body, slug: slugify(req.body.name) }` — trải body rồi **ghi đè `slug`**.
- Slider (`slider.js:108`): `req.body` **nguyên xi**.

Lý do: model `Slide` (`yotea-be/src/models/slider.js`) **không có trường `slug`** — banner
không cần URL thân thiện, người dùng không bao giờ vào `/slide/abc`. Đó cũng là lý do
`yotea-be/src/controllers/slider.js:1` **không import `slugify`**, còn `category.js:1` thì có.
Bốn controller sinh slug: `category`, `product`, `cateNews`, `news`.

**Lợi ích phụ:** vì không gọi `slugify`, `update` của slider **không dính bug crash server**
khi thiếu `name` — toàn bộ logic nằm gọn trong `try`.

</details>

**Bài 2.** Bạn gọi `PUT /api/category/<id>/<userId>` với body sau. Dự đoán **chính xác**
document trong database sau lệnh, và giải thích.

```json
{
  "name": "Cà phê",
  "_id": "111111111111111111111111",
  "createdAt": "2000-01-01T00:00:00.000Z",
  "soLuongBan": 999
}
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

| Trường gửi lên | Số phận | Vì sao |
|---|---|---|
| `name: "Cà phê"` | ✅ Được cập nhật | Có trong Schema |
| *(controller tự thêm)* `slug` | ✅ `"Ca-phe"` → lưu thành `ca-phe` | `category.js:124` sinh ra, `lowercase: true` hạ chữ |
| `_id` | ❌ Bị MongoDB từ chối | `_id` là **bất biến** (immutable) |
| `createdAt` | ⚠️ **Có thể bị ghi đè!** | Nó **có** trong Schema (do `{ timestamps: true }`), Mongoose không chặn |
| `soLuongBan: 999` | ❌ Bị loại âm thầm | Không có trong Schema, `strict: true` mặc định loại bỏ |

**Điểm nguy hiểm là `createdAt`.** Dự án lấy nguyên `...req.body` không lọc, nên client có thể
sửa ngày tạo — làm hỏng mọi thống kê và mọi truy vấn `_sort=createdAt`.

**Cách làm đúng:** lập **allow-list**, chỉ nhận đúng những trường được phép sửa:

```js
// đoạn này BẠN TỰ VIẾT để sửa lỗi — dự án hiện KHÔNG có
const { name, image } = req.body;
const update = { name, image, slug: slugify(name) };
```

Bạn sẽ áp dụng kỹ thuật này khi refactor ở [Bài 34](34-refactor-du-an.md).

</details>

**Bài 3.** (nâng cao) Sửa hàm `remove` của **category** để nó **chặn** việc xoá danh mục đang
còn sản phẩm. Viết ra file nháp — **đừng sửa file dự án**.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```js
// đoạn này BẠN TỰ VIẾT ở file nháp — KHÔNG sửa vào dự án
export const remove = async (req, res) => {
  const filter = { _id: req.params.id };

  try {
    const soSanPham = await Product.countDocuments({ categoryId: req.params.id }).exec();

    if (soSanPham > 0) {
      return res.status(400).json({
        message: `Danh mục còn ${soSanPham} sản phẩm, không thể xoá.`,
      });
    }

    const category = await CategoryModel.findOneAndDelete(filter).exec();

    if (!category) {
      return res.status(404).json({ message: "Không tìm thấy danh mục" });
    }

    res.json(category);
  } catch (error) {
    res.status(400).json({ message: "Xóa danh mục không thành công", error });
  }
};
```

Thuận lợi: `controllers/category.js:3` **đã sẵn** `import Product from "../models/product"`
(dùng cho vòng lặp trong `list`), không cần thêm import. `countDocuments` nhanh hơn
`find().length` rất nhiều vì đếm ngay trong database, không kéo dữ liệu về Node.

Nhớ chữ `return` trước `res.status(400)` — thiếu nó thì code **vẫn chạy tiếp** xuống dòng xoá,
gây lỗi "Cannot set headers after they are sent". Đây đúng là bug có thật ở hàm `signup`
(`yotea-be/src/controllers/auth.js`), ta gặp lại ở [Bài 11](11-mat-khau-va-jwt.md).

**Áp dụng cho Topping:** sau [Bài 10](10-quan-he-va-populate.md), khi Topping đã nối với
`OrderDetail`, bạn sẽ phải làm đúng việc này cho `remove` của topping — không cho xoá một
topping đã nằm trong đơn hàng cũ.

</details>

---

## 📌 Tóm tắt

- **CRUD** tách thành **5 hàm** trong code: `create` · `read` · `list` · `update` · `remove` — bộ tên lặp lại ở cả 14 controller của Yotea.
- Ánh xạ cần thuộc: `POST → new Model().save()` · `GET một → findOne` · `GET danh sách → find` · `PUT → findOneAndUpdate` · `DELETE → findOneAndDelete`.
- Yotea cho `read` tìm theo **`slug`** (thân thiện SEO) nhưng `update`/`remove` tìm theo **`_id`**.
- **`{ new: true }`** bắt `findOneAndUpdate` trả về bản **MỚI**; thiếu nó thì database vẫn đúng nhưng API trả **bản cũ** — bug cực khó phát hiện.
- Hàm `findOneAnd…` trả về **document**; `updateOne`/`deleteOne` chỉ trả về **con số thống kê**.
- Mọi controller bọc **`try/catch`** vì `await` lỗi mà không bắt sẽ **giết cả process Node**; dự án trả tất cả về `res.status(400)`.
- **⚠️ Xoá danh mục không kiểm tra ràng buộc** → sản phẩm "mồ côi", `populate` trả `null`, cột Danh mục trên trang admin hiện trống.
- **⚠️ Backend không validate gì ngoài Schema**, và `findOneAndUpdate` còn bỏ qua cả Schema vì thiếu `runValidators: true`.
- Postman: **Body → raw → JSON**, token ở tab **Authorization → Bearer Token**, status code ở **góc trên bên phải** khung Response, nhớ `Ctrl + S` để lưu request.

**Từ khoá tra cứu thêm:** `CRUD REST API`, `mongoose findOneAndUpdate new true`, `mongoose runValidators`, `mongoose strict mode`, `findOne vs findById`, `deleteOne vs findOneAndDelete`, `postman collection tutorial`, `orphaned records`

➡️ **Bài tiếp theo:** [08 — Slug thân thiện SEO với slugify](08-slug-slugify.md) — bạn đã thấy dòng `req.body.slug = slugify(req.body.name)` xuất hiện 2 lần trong bài này mà chưa hiểu nó làm gì. Bài sau mổ xẻ nó, chỉ ra vì sao "Đá Xay" biến thành "DJa-Xay", và bạn sẽ tự thêm trường `slug` cho Topping.
