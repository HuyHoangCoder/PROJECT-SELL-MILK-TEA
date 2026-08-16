# Bài 09 — Bộ lọc kiểu json-server: `_sort`, `_limit`, `_expand`, `_like`, `q`…

> **Phần 1 · Backend** — Thời lượng ước tính: **~100 phút**
> ⬅️ Bài trước: [08 — Slug thân thiện SEO với slugify](08-slug-slugify.md) · Bài sau: [10 — Quan hệ dữ liệu & `populate`](10-quan-he-va-populate.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Hiểu **json-server là gì** và vì sao backend Yotea phải viết tay lại đúng bộ quy ước query của nó.
- Đọc vanh vách hàm `list()` trong `yotea-be/src/controllers/product.js` — **từng khối một**.
- Dịch được **bất kỳ URL nào** thành object `filter` mà Mongoose thực sự nhận.
- Chỉ ra được đoạn code này bị **copy-paste 13 lần** ở đâu, và vì sao đó là vấn đề.
- Nhìn ra **6 lỗi/hạn chế thật** của đoạn code, kèm cách làm đúng.
- Tự viết hàm dùng chung `buildQuery()` và gắn nó vào chức năng **Topping** của bạn.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 08 — Slug thân thiện SEO với slugify](08-slug-slugify.md).
- Đã có **model / route / controller Topping** với đủ 5 thao tác CRUD + trường `slug`.
- MongoDB đang chạy; backend chạy được bằng `npm start` tại `yotea-be`; có Postman.

> 💡 **Mạch thực hành xuyên suốt:** ở [Bài 08](08-slug-slugify.md) bạn đã thêm trường `slug` cho
> Topping để URL đẹp hơn. **Bài này ta làm tiếp:** cho API Topping biết **lọc, sắp xếp, phân trang
> và tìm kiếm** — giống hệt API sản phẩm.

---

## 1. json-server là gì, và vì sao Yotea phải bắt chước nó?

Khi nhóm sinh viên bắt đầu làm Yotea, họ **chưa có backend**. Frontend cần dữ liệu để hiển thị, nên
họ dùng **json-server**.

> 📖 **Thuật ngữ:** *json-server* — thư viện npm biến **một file `db.json`** thành REST API đầy đủ
> trong 30 giây. Viết `{"products": [...]}` vào file, chạy `json-server db.json`, lập tức có
> `GET /products`, `POST /products`, `PUT /products/1`… mà không cần một dòng code backend.

Điểm hấp dẫn nhất của json-server là nó **tặng kèm luôn một bộ quy ước query rất mạnh**:

```
GET /products?_sort=price&_order=asc      ← sắp xếp
GET /products?_start=0&_limit=9           ← phân trang
GET /products?name_like=tra               ← tìm gần đúng
GET /products?price_gte=20000             ← lớn hơn hoặc bằng
GET /products?q=trà sữa                   ← tìm toàn văn
GET /products?_expand=categoryId          ← "nở" khoá ngoại thành object
```

Frontend Yotea đã được viết **bám chặt** vào bộ quy ước đó — `yotea-fe/src/api/product.js:8-17`:

```js
export const getAll = (
  start = 0,
  limit = 0,
  sort = "createdAt",
  order = "desc"
) => {
  let url = `/${DB_NAME}/?_expand=categoryId&_sort=${sort}&_order=${order}`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};
```

Đến lúc thay json-server bằng Express + MongoDB, nhóm đứng trước ngã ba đường:

| Lựa chọn | Việc phải làm | Hậu quả |
|---|---|---|
| **A.** Sửa frontend theo backend | Sửa lại **toàn bộ** 13 file trong `yotea-fe/src/api/` và các component gọi chúng | Rủi ro cao, dễ sót |
| **B.** Sửa backend theo frontend | Viết tay lại **đúng** protocol json-server trong controller | Frontend **không phải đổi một chữ nào** |

Nhóm chọn **B**. Kết quả là hàm `list()` — một "**bộ dịch**" nhận query string kiểu json-server rồi
phiên ra **toán tử MongoDB**:

```
Frontend                      list() (bộ dịch)                        MongoDB
────────────►   req.query   ────────────────────►   filter/sortOpt   ────────────►
?name_like=tra  (toàn chuỗi)                    { name: { $in: [/tra/i] } }
```

> 💡 Kỹ thuật này có tên là **Adapter** (bộ chuyển đổi): thay vì bắt hai bên nói cùng ngôn ngữ, ta
> viết một lớp phiên dịch ở giữa. Ý tưởng hoàn toàn hợp lý — chỉ là **cách hiện thực hoá** trong dự
> án này có nhiều chỗ vụng, ta sẽ chỉ ra ở mục 5 và 6.

---

## 2. Soi code thật — mổ hàm `list()` từng khối

Ta lấy `yotea-be/src/controllers/product.js` làm bản gốc, vì đây là bản đã chạy Prettier (2 space,
dễ đọc nhất) và là endpoint frontend gọi nhiều nhất. Toàn bộ hàm nằm ở **dòng 107 → 180**, chia
thành **5 khối**.

### 2.1. Khối 1 — Đọc `_expand`

`yotea-be/src/controllers/product.js:107-108`

```js
export const list = async (req, res) => {
  const populate = req.query["_expand"];
```

| Dòng | Ý nghĩa |
|---|---|
| 107 | Controller cho `GET /api/products` |
| 108 | Lấy giá trị tham số `_expand` trên URL, cất vào biến `populate` |

`_expand` được đọc **riêng ngay từ đầu** vì nó **không phải điều kiện lọc** — nó là chỉ thị "hãy nở
khoá ngoại này thành object đầy đủ", lát nữa sẽ được truyền thẳng vào `.populate(populate)`. Nếu URL
không có `_expand`, biến là `undefined`, và `.populate(undefined)` được Mongoose **bỏ qua yên lặng**
— nên không cần viết `if`.

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `_expand` **không** được tách theo dấu phẩy như `_sort`.
> Gõ `?_expand=categoryId,userId` khiến Mongoose đi tìm path tên đúng là `"categoryId,userId"` —
> không tồn tại → **không nở gì cả, cũng không báo lỗi**. Chi tiết `populate` học ở
> [Bài 10](10-quan-he-va-populate.md).

### 2.2. Khối 2 — Dựng `sortOpt` từ `_sort` + `_order`

`yotea-be/src/controllers/product.js:110-118`

```js
  let sortOpt = {};
  if (req.query["_sort"]) {
    const sortArr = req.query["_sort"].split(",");
    const orderArr = (req.query["_order"] || "").split(",");

    sortArr.forEach((sort, index) => {
      sortOpt[sort] = orderArr[index] === "desc" ? -1 : 1;
    });
  }
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 110 | `let sortOpt = {};` | Mặc định **không sắp xếp**. `.sort({})` = để MongoDB trả theo thứ tự tự nhiên |
| 111 | `if (req.query["_sort"])` | Chỉ làm việc khi URL **có** `_sort`. Không có `_sort` thì `_order` bị **bỏ qua hoàn toàn** |
| 112 | `._sort.split(",")` | `"view,createdAt"` → `["view","createdAt"]` — **hỗ trợ sắp xếp nhiều trường** |
| 113 | `(req.query["_order"] \|\| "").split(",")` | `\|\| ""` là lưới an toàn: thiếu `_order` thì dùng chuỗi rỗng, `"".split(",")` cho `[""]` — không nổ lỗi |
| 115 | `sortArr.forEach((sort, index) =>` | Duyệt từng trường **kèm chỉ số** `index` |
| 116 | `orderArr[index] === "desc" ? -1 : 1` | Ghép trường thứ `index` với chiều thứ `index`. `-1` giảm dần, `1` tăng dần |

Đây là chỗ đọc lướt rất dễ bỏ sót: `_sort` và `_order` được ghép **theo cặp đúng vị trí**.

```
_sort  =  view  ,  createdAt
            │         │
_order =  desc  ,    asc
            ▼         ▼
sortOpt = { view: -1, createdAt: 1 }
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** điều kiện là `=== "desc"` — **so sánh chuỗi tuyệt đối**.
> Gõ `_order=DESC`, `_order=descending`, hay quên luôn `_order` đều rơi vào nhánh `: 1` →
> **tăng dần**, và không có cảnh báo nào cho người gọi API biết họ gõ sai.

### 2.3. Khối 3 — Lấy `_start` / `_limit`

`yotea-be/src/controllers/product.js:120-123`

```js
  const start = req.query["_start"];
  const limit = req.query["_limit"];

  const filter = {};
```

| Dòng | Ý nghĩa |
|---|---|
| 120 | Bỏ qua bao nhiêu bản ghi đầu → sẽ vào `.skip()` |
| 121 | Lấy tối đa bao nhiêu bản ghi → sẽ vào `.limit()` |
| 123 | Khai báo object điều kiện lọc, **rỗng** — khối 4 sẽ nhồi dần vào đây |

Chú ý: giá trị lấy từ `req.query` **luôn là chuỗi** — `"9"` chứ không phải `9`. Hệ quả nói ở mục 6.5.

### 2.4. Khối 4 — Vòng lặp `queryArr.forEach` (trái tim của bài này)

`yotea-be/src/controllers/product.js:125-163`

```js
  const { _expand, _sort, _order, ...query } = req.query;
  const queryArr = Object.keys(query);
  queryArr.forEach((item) => {
    if (item.includes("like")) {
      const objectKey = item.slice(0, item.indexOf("_"));

      if (Object.hasOwn(filter, objectKey)) {
        filter[objectKey]["$in"].push(new RegExp(req.query[item], "i"));
      } else {
        filter[objectKey] = { $in: [new RegExp(req.query[item], "i")] };
      }
    } else if (item.includes("_ne")) {
      filter[item.slice(0, item.indexOf("_ne"))] = { $nin: query[item] };
    } else if (item.includes("_gte")) {
      const objectKey = item.slice(0, item.indexOf("_gte"));

      if (Object.hasOwn(filter, objectKey)) {
        filter[objectKey]["$gte"] = query[item];
      } else {
        filter[objectKey] = { $gte: query[item] };
      }
    } else if (item.includes("_lte")) {
      const objectKey = item.slice(0, item.indexOf("_lte"));

      if (Object.hasOwn(filter, objectKey)) {
        filter[objectKey]["$lte"] = query[item];
      } else {
        filter[objectKey] = { $lte: query[item] };
      }
    } else if (item === "q" && query["q"]) {
      filter["$text"] = { $search: `"${query["q"]}"` };
    } else {
      if (Object.hasOwn(filter, item)) {
        filter[item]["$in"].push(query[item]);
      } else {
        filter[item] = { $in: [query[item]] };
      }
    }
  });
```

**Ba dòng chuẩn bị:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 125 | `const { _expand, _sort, _order, ...query } = req.query;` | **Rest destructuring**: bóc 3 tham số "điều khiển" ra, phần **còn lại** gom vào `query`. Nói cách khác: *"loại `_expand`, `_sort`, `_order` khỏi danh sách điều kiện lọc"* |
| 126 | `const queryArr = Object.keys(query);` | Lấy **danh sách TÊN** tham số còn lại, ví dụ `["categoryId","status","_start","_limit"]` |
| 127 | `queryArr.forEach((item) => {` | Duyệt từng **tên**. `item` là **tên**, `query[item]` mới là **giá trị** |

> 💡 **Mẹo đọc code:** ba biến `_expand`, `_sort`, `_order` ở dòng 125 khai ra rồi **không dùng lại
> lần nào**. Chúng tồn tại **chỉ để bị loại** khỏi `...query`. Đây là *omit trick* — bạn đã gặp ở
> [Bài 03](03-kien-thuc-nen.md) khi loại `password` khỏi dữ liệu user trước lúc gửi về client.

**Sáu nhánh của chuỗi `if / else if`** — thứ tự kiểm tra **rất quan trọng**, nhánh đứng trước sẽ
"nuốt" mất tham số trước:

| # | Dòng | Điều kiện | Việc làm | Ví dụ |
|---|---|---|---|---|
| 1 | `:128-135` | tên **chứa** `"like"` | Cắt tên trường tại dấu `_` **đầu tiên** (`:129`), tạo `{ $in: [RegExp(v,"i")] }`. Nếu trường đã có thì **push thêm** vào mảng `$in` | `?name_like=tra` → `{ name: { $in: [/tra/i] } }` |
| 2 | `:136-137` | tên **chứa** `"_ne"` | Cắt tại `indexOf("_ne")`, **gán đè** `{ $nin: … }` | `?_id_ne=62496b` → `{ _id: { $nin: "62496b" } }` |
| 3 | `:138-145` | tên **chứa** `"_gte"` | Cắt tại `indexOf("_gte")`, thêm `$gte`. Có `Object.hasOwn` nên **gộp** được với `$lte` | `?price_gte=20000` → `{ price: { $gte: "20000" } }` |
| 4 | `:146-153` | tên **chứa** `"_lte"` | Y hệt nhánh 3 nhưng với `$lte` | `?price_lte=50000` → `{ price: { $lte: "50000" } }` |
| 5 | `:154-155` | tên **bằng đúng** `"q"` **và** giá trị khác rỗng | `filter["$text"] = { $search: '"…"' }` | `?q=trà sữa` → `{ $text: { $search: "\"trà sữa\"" } }` |
| 6 | `:156-162` | mọi trường hợp còn lại | Bọc giá trị trong `{ $in: [v] }` | `?status=1` → `{ status: { $in: ["1"] } }` |

Vài điểm cần soi kỹ:

- **Nhánh 1** dùng `new RegExp("tra", "i")` → `/tra/i`: khớp **bất cứ chuỗi nào có chứa** `tra`,
  **không phân biệt hoa thường** — chính là `LIKE '%tra%'` bên SQL.
- **Nhánh 2:** `"_id_ne".indexOf("_ne")` trả về **3** (dấu `_` ở vị trí 0 thuộc về `_id`, còn `_ne`
  bắt đầu ở vị trí 3) → `slice(0, 3)` cho đúng `"_id"`. Frontend dùng nó để lấy sản phẩm liên quan —
  cùng danh mục nhưng **không phải chính nó** (`yotea-fe/src/api/product.js:32`).
- **Nhánh 3 + 4** là chỗ **duy nhất** `Object.hasOwn` thật sự hữu ích: với
  `?price_gte=20000&price_lte=50000`, gặp `price_gte` trước tạo `{ price: { $gte: "20000" } }`, gặp
  `price_lte` sau thấy `price` đã có nên **thêm** `$lte` vào chính object đó → kết quả
  `{ price: { $gte: "20000", $lte: "50000" } }`.
  > 📖 **Thuật ngữ:** `Object.hasOwn(obj, key)` — hỏi *"object này có **tự nó** sở hữu thuộc tính
  > `key` không?"*. Bản thay thế hiện đại, an toàn hơn của `obj.hasOwnProperty(key)`; chỉ có từ
  > **Node 16.9** trở lên.
- **Nhánh 5** dùng `===` chứ không phải `includes`. Cặp dấu `"` bao ngoài trong `` `"${query["q"]}"` ``
  khiến chuỗi gửi đi là `"trà sữa"` **kèm ngoặc kép** ⇒ MongoDB hiểu là **tìm cụm từ chính xác**,
  không phải tìm "trà" HOẶC "sữa". `$text` chỉ chạy nhờ **text index** khai trong model —
  `yotea-be/src/models/product.js:43`:

  ```js
  productSchema.index({'$**': 'text'});
  ```

  `'$**'` là **wildcard**: đánh chỉ mục toàn văn cho **mọi trường chuỗi** của document.
- **Nhánh 6** bọc trong `$in: [ … ]` thay vì gán thẳng, với ý đồ **cộng dồn nhiều giá trị cho cùng
  một trường**. Ý đồ tốt, nhưng cách hiện thực không chạy — xem mục 6.6.

### 2.5. Khối 5 — Truy vấn Mongoose cuối cùng

`yotea-be/src/controllers/product.js:165-180`

```js
  try {
    const products = await Product.find(filter)
      .select("-__v")
      .populate(populate)
      .skip(start)
      .limit(limit)
      .sort(sortOpt)
      .exec();
    res.json(products);
  } catch (error) {
    res.status(400).json({
      message: "Không tìm thấy sản phẩm",
      error,
    });
  }
};
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 166 | `Product.find(filter)` | Dựng truy vấn với object điều kiện ráp ở khối 4 |
| 167 | `.select("-__v")` | **Loại** trường `__v` (số phiên bản nội bộ của Mongoose). Dấu `-` nghĩa là "bỏ" |
| 168 | `.populate(populate)` | "Nở" khoá ngoại theo `_expand`. `undefined` → bỏ qua |
| 169 | `.skip(start)` | Bỏ qua `_start` bản ghi đầu |
| 170 | `.limit(limit)` | Lấy tối đa `_limit` bản ghi. **`limit(0)` = không giới hạn** |
| 171 | `.sort(sortOpt)` | Sắp xếp theo `sortOpt` dựng ở khối 2 |
| 172 | `.exec()` | **Bấm nút chạy** — trước dòng này chưa hề có câu lệnh nào chạm tới MongoDB |
| 173 | `res.json(products)` | Trả mảng kết quả về client |
| 175-178 | `res.status(400).json({...})` | Có lỗi (ví dụ `_limit=abc` không ép được thành số) → trả 400 |

> 💡 **Điểm quan trọng:** `Product.find(...)` trả về một **Query object**, không phải dữ liệu. Nối
> `.select().populate().skip().limit().sort()` là để **mô tả** truy vấn — như viết đơn đặt hàng.
> Chỉ đến `.exec()` (hoặc `await`) đơn hàng mới được gửi đi. Kỹ thuật nối chuỗi này gọi là
> **method chaining**. Vì thế **thứ tự các `.select/.skip/.sort` không quan trọng**; nhưng thứ tự
> MongoDB **thực thi** thì cố định: `find` → `sort` → `skip` → `limit`.

---

## 3. 📖 BẢNG TRA CỨU ĐẦY ĐỦ — quy ước query → toán tử MongoDB

| Quy ước trên URL | Dòng xử lý | Toán tử / lệnh Mongo | URL ví dụ thật | Kết quả sinh ra |
|---|---|---|---|---|
| `_expand=<path>` | `:108`, `:168` | `.populate(path)` | `?_expand=categoryId` | `populate = "categoryId"` — **không** vào `filter` |
| `_sort=<field>` | `:111-118`, `:171` | `.sort({field: 1})` | `?_sort=price` | `sortOpt = { price: 1 }` |
| `_sort` + `_order=desc` | `:113`, `:116` | `.sort({field: -1})` | `?_sort=price&_order=desc` | `sortOpt = { price: -1 }` |
| `_sort=a,b` + `_order=x,y` | `:112-117` | `.sort({a: …, b: …})` | `?_sort=view,createdAt&_order=desc,asc` | `sortOpt = { view: -1, createdAt: 1 }` |
| `_start=<n>` | `:120`, `:169` | `.skip(n)` | `?_start=9` | `start = "9"` |
| `_limit=<n>` | `:121`, `:170` | `.limit(n)` | `?_limit=9` | `limit = "9"` (`0` = lấy tất) |
| `<field>_like=<v>` | `:128-135` | `$in` + `RegExp(v,"i")` | `?name_like=tra` | `{ name: { $in: [/tra/i] } }` |
| `<field>_ne=<v>` | `:136-137` | `$nin` | `?_id_ne=62496b4d` | `{ _id: { $nin: "62496b4d" } }` |
| `<field>_gte=<v>` | `:138-145` | `$gte` (≥) | `?price_gte=20000` | `{ price: { $gte: "20000" } }` |
| `<field>_lte=<v>` | `:146-153` | `$lte` (≤) | `?price_lte=50000` | `{ price: { $lte: "50000" } }` |
| `<f>_gte` **và** `<f>_lte` | `:141`, `:149` | `$gte` + `$lte` gộp | `?price_gte=20000&price_lte=50000` | `{ price: { $gte: "20000", $lte: "50000" } }` |
| `q=<từ khoá>` | `:154-155` | `$text` / `$search` | `?q=trà sữa` | `{ $text: { $search: "\"trà sữa\"" } }` |
| `<field>=<v>` (thường) | `:156-162` | `$in` | `?status=1` | `{ status: { $in: ["1"] } }` |

**Nghĩa của các toán tử MongoDB xuất hiện ở trên:**

| Toán tử | Đọc là | SQL tương đương |
|---|---|---|
| `$in: [a, b]` | Giá trị nằm **trong** danh sách | `WHERE field IN (a, b)` |
| `$nin: a` | Giá trị **không nằm trong** danh sách | `WHERE field NOT IN (a)` |
| `$gte: v` / `$lte: v` | Lớn hơn / nhỏ hơn **hoặc bằng** | `WHERE field >= v` / `<= v` |
| `$text: { $search: "…" }` | Tìm toàn văn theo text index | `WHERE MATCH(...) AGAINST(...)` |
| `RegExp(v,"i")` trong `$in` | Chứa chuỗi con, không phân biệt hoa/thường | `WHERE field LIKE '%v%'` |

---

## 4. Ba ví dụ chuyển đổi CHI TIẾT: URL → object cuối cùng

### 4.1. Ví dụ 1 — Xem thực đơn theo danh mục, sắp xếp theo giá tăng dần

Đây là URL **thật sự** trình duyệt bắn đi khi bạn vào trang một danh mục rồi chọn ô *"Thứ tự theo
giá: thấp → cao"*. Ô select đó có `value="price-asc"`
(`yotea-fe/src/components/user/FilterProduct.js:67`), được tách bằng dấu `-` thành `sort = "price"`,
`order = "asc"` rồi đưa vào `yotea-fe/src/api/product.js:19-29`:

```js
export const getProductByCate = (
  start = 0,
  limit = 0,
  sort = "createdAt",
  order = "desc",
  cateId
) => {
  let url = `/${DB_NAME}/?categoryId=${cateId}&_sort=${sort}&_order=${order}&_expand=categoryId`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};
```

Với `limit = 9` (`yotea-fe/src/components/user/ProductContent.js:26`) và đang ở trang 1
(`start = 0`), URL cuối cùng là:

```
GET http://localhost:8080/api/products/?categoryId=6249a1f2c4e8b91234abcd99&_sort=price&_order=asc&_expand=categoryId&_start=0&_limit=9
```

| Khối | Kết quả |
|---|---|
| 1 — `_expand` | `populate = "categoryId"` |
| 2 — `_sort`/`_order` | `sortArr = ["price"]`, `orderArr = ["asc"]` → `"asc" !== "desc"` → `sortOpt = { price: 1 }` |
| 3 — `_start`/`_limit` | `start = "0"`, `limit = "9"` |
| 4 — `forEach` | Sau khi bỏ `_expand`/`_sort`/`_order`, còn `["categoryId", "_start", "_limit"]` |

| `item` | Nhánh trúng | `filter` được thêm |
|---|---|---|
| `categoryId` | 6 (`else`) | `categoryId: { $in: ["6249a1f2c4e8b91234abcd99"] }` |
| `_start` | 6 (`else`) | `_start: { $in: ["0"] }` ← **rác** |
| `_limit` | 6 (`else`) | `_limit: { $in: ["9"] }` ← **rác** |

**Truy vấn cuối cùng:**

```js
Product.find({
  categoryId: { $in: ["6249a1f2c4e8b91234abcd99"] },
  _start: { $in: ["0"] },      // ← bị Mongoose 6 loại bỏ, xem mục 6.1
  _limit: { $in: ["9"] },      // ← bị Mongoose 6 loại bỏ
})
  .select("-__v")
  .populate("categoryId")
  .skip("0")
  .limit("9")
  .sort({ price: 1 })
  .exec();
```

Dịch sang tiếng Việt: *"lấy sản phẩm thuộc danh mục này, ẩn `__v`, nở `categoryId` thành object danh
mục đầy đủ, sắp theo giá tăng dần, bỏ 0 cái đầu, lấy 9 cái."*

### 4.2. Ví dụ 2 — Sản phẩm liên quan (dùng `_ne` và nhiều điều kiện)

URL do `getProductsRelated()` (`yotea-fe/src/api/product.js:31-35`) sinh ra, với `limit = 4`:

```
GET http://localhost:8080/api/products/?categoryId=6249a1&_id_ne=62496b&status=1&_expand=categoryId&_sort=createdAt&_order=desc&_start=0&_limit=4
```

Khối 1 → `populate = "categoryId"`. Khối 2 → `sortOpt = { createdAt: -1 }` (vì `"desc" === "desc"`).
Khối 3 → `start = "0"`, `limit = "4"`. Vòng `forEach` chạy trên 5 tên tham số:

| `item` | Kiểm tra | Nhánh | Đóng góp vào `filter` |
|---|---|---|---|
| `categoryId` | không chứa `like`, không chứa `_ne` | 6 | `categoryId: { $in: ["6249a1"] }` |
| `_id_ne` | không chứa `like`; **chứa `_ne`** ✔ | 2 | `_id: { $nin: "62496b" }` |
| `status` | — | 6 | `status: { $in: ["1"] }` |
| `_start` | — | 6 | `_start: { $in: ["0"] }` (rác) |
| `_limit` | — | 6 | `_limit: { $in: ["4"] }` (rác) |

```js
Product.find({
  categoryId: { $in: ["6249a1"] },
  _id: { $nin: "62496b" },
  status: { $in: ["1"] },
})
  .select("-__v")
  .populate("categoryId")
  .skip("0")
  .limit("4")
  .sort({ createdAt: -1 })
  .exec();
```

*"4 sản phẩm mới nhất, cùng danh mục, đang hiển thị, nhưng không phải chính sản phẩm đang xem."*

### 4.3. Ví dụ 3 — Tìm kiếm bằng `q` (tìm toàn văn)

`yotea-fe/src/api/product.js:37-47`:

```js
export const search = (
  start = 0,
  limit = 0,
  sort = "createdAt",
  order = "desc",
  keyword
) => {
  let url = `/${DB_NAME}/?_sort=${sort}&_order=${order}&q=${keyword}`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};
```

Gõ từ khoá `trà đào` vào ô tìm kiếm →

```
GET http://localhost:8080/api/products/?_sort=createdAt&_order=desc&q=trà đào&_start=0&_limit=9
```

`q` khớp nhánh 5 (`item === "q"` ✔ **và** `"trà đào"` khác rỗng ✔); `_start`, `_limit` rơi vào
nhánh 6 (rác). Kết quả:

```js
Product.find({
  $text: { $search: '"trà đào"' },
})
  .select("-__v")
  .populate(undefined)
  .skip("0")
  .limit("9")
  .sort({ createdAt: -1 })
  .exec();
```

Chú ý cặp ngoặc kép **bên trong** chuỗi: MongoDB nhận `"trà đào"` → tìm **đúng cụm từ** "trà đào"
đứng liền nhau, chứ không phải "sản phẩm nào có chữ trà **hoặc** chữ đào".

---

## 5. ⚠️ Đoạn code này bị copy-paste **13 lần**

Đây là bài học lớn nhất hôm nay, quan trọng hơn cả cú pháp. Tìm chuỗi
`const { _expand, _sort, _order, ...query } = req.query;` trong `yotea-be/src/` cho ra **13 kết quả**,
ở **13 controller khác nhau**:

| # | File | Dòng chứa `const { _expand, … }` | `export const list` bắt đầu ở |
|---|---|---|---|
| 1 | `yotea-be/src/controllers/category.js` | `:52` | `:34` |
| 2 | `yotea-be/src/controllers/cateNews.js` | `:51` | `:33` |
| 3 | `yotea-be/src/controllers/comment.js` | `:48` | `:30` |
| 4 | `yotea-be/src/controllers/contact.js` | `:48` | `:30` |
| 5 | `yotea-be/src/controllers/favoritesProduct.js` | `:48` | `:30` |
| 6 | `yotea-be/src/controllers/news.js` | `:52` | `:34` |
| 7 | `yotea-be/src/controllers/order.js` | `:48` | `:30` |
| 8 | `yotea-be/src/controllers/orderDetail.js` | `:48` | `:30` |
| 9 | `yotea-be/src/controllers/product.js` | `:125` | `:107` |
| 10 | `yotea-be/src/controllers/rating.js` | `:48` | `:30` |
| 11 | `yotea-be/src/controllers/slider.js` | `:48` | `:30` |
| 12 | `yotea-be/src/controllers/store.js` | `:48` | `:30` |
| 13 | `yotea-be/src/controllers/user.js` | `:130` | `:112` |

Trong `yotea-be/src/controllers/`, chỉ đúng **một** file **không** có `list()`: `auth.js`. Khoảng
**41 dòng logic × 13 file ≈ 530 dòng trùng lặp**. Khác biệt giữa các bản chỉ là tên Model, tên biến
kết quả, và câu thông báo lỗi (`"Không tìm thấy sản phẩm"` / `"Không tìm thấy slide"` /
`"Không tìm thấy chi nhánh"`…).

> ⚠️ **Chỗ này dự án làm chưa chuẩn — vi phạm nguyên tắc DRY.**
>
> 📖 **Thuật ngữ:** *DRY* = **D**on't **R**epeat **Y**ourself. Mỗi mẩu kiến thức trong hệ thống chỉ
> nên tồn tại ở **đúng một chỗ**.
>
> **Hậu quả cụ thể khi cần sửa một lỗi:** giả sử mai bạn phát hiện bug ở mục 6.1 và muốn vá bằng
> cách thêm hai chữ vào dòng destructure. Bạn phải mở **13 file**, sửa **13 chỗ**, và:
> - Chỉ cần **quên một file**, hệ thống chạy **không đồng nhất** — `GET /api/products` đúng nhưng
>   `GET /api/orders` sai. Loại bug này cực khó truy vì **không có thông báo lỗi nào**.
> - Người review PR phải đọc 13 diff giống hệt nhau → mỏi mắt → dễ duyệt bừa.
> - Viết unit test cũng phải nhân 13 lần.
>
> **Cách làm đúng:** tách **một** hàm dùng chung, ví dụ `utils/buildQuery.js`, rồi 13 controller
> cùng gọi nó. Sửa bug ⇒ sửa **một chỗ duy nhất**. Đó chính là việc bạn làm ở mục 7, và là bài tập
> lớn ở [Bài 34 — Refactor dự án](34-refactor-du-an.md).

---

## 6. ⚠️ Sáu hạn chế và lỗi thật của đoạn code

### 6.1. `_start` và `_limit` lọt vào `filter` — quả bom hẹn giờ

Dòng `:125` chỉ loại **3** tham số điều khiển, `_start` và `_limit` **không bị loại** → chúng đi
thẳng vào `forEach` → rơi vào nhánh 6 → sinh điều kiện lọc vô nghĩa
`{ _start: { $in: ["0"] }, _limit: { $in: ["9"] } }`.

Hiện tại **may mắn không hỏng** vì `yotea-be/package.json:21` khai `"mongoose": "^6.2.8"`, mà
**Mongoose 6 mặc định bật `strictQuery: true`** — tự động **vứt bỏ mọi điều kiện lọc trên trường
không có trong schema**. Model `Product` không có `_start`/`_limit` nên chúng bị dọn sạch.

> ⚠️ **Hậu quả nếu nâng cấp:** Mongoose **7 trở lên** đổi mặc định thành `strictQuery: false`. Ngày
> ai đó chạy `npm update mongoose`, hai điều kiện rác kia được gửi thật xuống MongoDB → **mọi
> endpoint danh sách có phân trang trả về mảng rỗng**. Website trắng trơn mà terminal không một
> dòng lỗi.
>
> **Cách sửa (một dòng):** `const { _expand, _sort, _order, _start, _limit, ...query } = req.query;`

### 6.2. Nhánh `*_like` cắt sai tên trường có dấu gạch dưới

Dòng `:129` dùng `item.slice(0, item.indexOf("_"))` — cắt tại dấu `_` **ĐẦU TIÊN**. Với `name_like`
thì đúng, nhưng:

| URL | Tên trường code cắt ra | Đáng lẽ phải là | Chuyện gì xảy ra |
|---|---|---|---|
| `?name_like=tra` | `"name"` ✔ | `"name"` | Đúng |
| `?full_name_like=hoang` | `"full"` ✘ | `"full_name"` | Lọc nhầm trường `full` (không tồn tại) |
| `?_id_like=6249` | `""` ✘ (**chuỗi rỗng!**) | `"_id"` | `indexOf("_")` = 0 → `slice(0,0)` = `""` |

Tệ hơn: điều kiện dòng `:128` là `item.includes("like")` — **chỉ tìm chữ `"like"`, không có gạch
dưới**. Nên trường nghiệp vụ nào có chữ `like` trong tên đều bị hiểu nhầm: `?dislike_count=3` bị lọc
thành trường `"dislike"` với RegExp `/3/i`; `?likes=10` có `indexOf("_")` = `-1` → `slice(0,-1)` →
lọc trường `"like"`.

Và nhánh `_ne` ngay bên dưới lại dùng chiến lược **khác** (`indexOf("_ne")`, chính xác hơn) —
**hai nhánh trong cùng một hàm dùng hai cách cắt chuỗi khác nhau**.

**Cách sửa đúng:** dùng `endsWith` và cắt theo độ dài hậu tố:

```js
if (item.endsWith("_like")) {
  const objectKey = item.slice(0, item.length - "_like".length);
}
```

### 6.3. Ghép `RegExp` trực tiếp từ input người dùng — rủi ro ReDoS

Dòng `:132` và `:134` đưa giá trị người dùng gõ **thẳng** vào `new RegExp()` mà **không escape**.

> 🔒 **Ghi chú bảo mật:** hai kiểu tấn công:
>
> 1. **Lách bộ lọc.** Gửi `?name_like=.*` — trong RegExp, `.*` nghĩa là "khớp mọi thứ" → bộ lọc mất
>    tác dụng hoàn toàn.
> 2. **ReDoS** (*Regular expression Denial of Service*). Gửi `?name_like=(a+)+$` — một mẫu regex
>    "thảm hoạ": với chuỗi đầu vào chỉ vài chục ký tự, engine phải thử **số tổ hợp tăng theo hàm
>    mũ**. Node.js chỉ có **một luồng chính (event loop)** → nó bận quay vòng với regex này thì
>    **toàn bộ server đứng hình**. Chỉ cần **một** request là đủ.
>
> **Cách sửa:** escape ký tự đặc biệt trước khi ghép — ta sẽ dùng đúng cách này ở mục 7.

### 6.4. `$text` cần text index và **không tìm được từ khoá một phần**

Hai điều kiện tiên quyết để `?q=` chạy:

1. Model phải có **text index**. Trong dự án mọi model đều có `schema.index({'$**': 'text'})`
   (ví dụ `yotea-be/src/models/product.js:43`). **Thiếu dòng này** → MongoDB ném lỗi
   `text index required for $text query` → controller trả `400`.
2. Từ khoá phải là **nguyên từ**, vì `$text` tìm theo **từ** (token), không phải **chuỗi con**:

| Bạn gõ | Với sản phẩm tên `"Trà sữa trân châu"` |
|---|---|
| `?q=trân châu` | ✅ Tìm thấy (đúng cụm từ) |
| `?q=trân` | ✅ Tìm thấy (đúng một từ) |
| `?q=chau` | ❌ Không thấy — thiếu dấu, khác token |
| `?q=tr` | ❌ Không thấy — `$text` không tìm chuỗi con |

Muốn "gõ tới đâu gợi ý tới đó" thì phải dùng `*_like` (RegExp), **không** dùng `q`. Ngoài ra khi tìm
bằng `$text`, thông thường ta muốn sắp theo **độ liên quan** (`{ score: { $meta: "textScore" } }`);
code ở đây chỉ sắp theo `_sort` trên URL — mà frontend luôn gửi `_sort=createdAt` → **kết quả tìm
kiếm được sắp theo ngày tạo, không theo mức độ khớp**.

### 6.5. `_start` / `_limit` là **chuỗi**, không phải số

`req.query` luôn trả về **chuỗi**, nên `.skip("0").limit("9")` đang nhận chuỗi. Mongoose ép kiểu
giúp nên bình thường vẫn chạy, nhưng:

| URL | Chuyện xảy ra |
|---|---|
| `?_limit=9` | Ép thành `9` → OK |
| `?_limit=abc` | Không ép được → Mongoose ném lỗi → rơi vào `catch` → **400 "Không tìm thấy sản phẩm"** (thông báo gây hiểu nhầm hoàn toàn) |
| `?_limit=-5` | `.limit(-5)` — hành vi không xác định |
| `?_limit=999999` | Không có trần → một request là **kéo cả database** về |

**Cách sửa:** ép kiểu và chặn biên ngay từ đầu, ta làm ở mục 7.

### 6.6. Query trùng key sinh mảng lồng mảng

Express dùng thư viện `qs` để phân tích query string. Gửi `?status=0&status=1` thì `req.query.status`
là **mảng** `["0","1"]`. Code làm `filter[item] = { $in: [query[item]] }` → kết quả
`{ status: { $in: [ ["0","1"] ] } }` — **một mảng chứa một mảng** → không khớp gì cả.

Trớ trêu: nhánh `Object.hasOwn(filter, item)` ở dòng `:157` được viết ra **chính là để xử lý tình
huống này**, nhưng vì `qs` đã gom sẵn thành mảng nên nhánh đó **gần như không bao giờ chạy**.

**Cách sửa:** dùng `[].concat(value)` — biến giá trị đơn thành mảng 1 phần tử, giữ nguyên mảng đã có.

---

## 7. 🛠️ Tự tay làm — tách `buildQuery()` dùng chung cho Topping

> **Mục tiêu phần này:** cuối phần bạn sẽ có file mới `yotea-be/src/utils/buildQuery.js` chứa **một**
> hàm dùng chung sửa được cả 6 lỗi ở mục 6; và API Topping của bạn hỗ trợ đầy đủ `_sort`, `_order`,
> `_start`, `_limit`, `_expand`, `q`, `*_like`, `*_ne`, `*_gte`, `*_lte`.

> ⚠️ **Nhắc lại quy tắc:** toàn bộ code dưới đây là **code bạn tự viết thêm**, dự án gốc **chưa có**.
> Tuyệt đối **không sửa** file có sẵn trong `yotea-be/` — ta chỉ **tạo file mới** và sửa **file
> topping do chính bạn tạo ra ở các bài trước**.

### Bước 1 — Tạo `src/utils/buildQuery.js`

Backend chưa có thư mục `utils`. Đứng tại thư mục `yotea-be`:

```bash
# đứng tại thư mục yotea-be
mkdir src/utils
```

Rồi tạo file `src/utils/buildQuery.js` với nội dung sau:

```js
// yotea-be/src/utils/buildQuery.js  ← file MỚI, bạn tự tạo (dự án gốc KHÔNG có file này)

// Escape ký tự đặc biệt của RegExp để tránh ReDoS / regex injection (mục 6.3)
const escapeRegExp = (value) =>
  String(value).replace(/[.*+?^${}()|[\]\\]/g, "\\$&");

// Gộp thêm toán tử vào điều kiện đã có của một trường, thay vì ghi đè
const mergeFilter = (filter, key, condition) => {
  filter[key] = { ...(filter[key] || {}), ...condition };
};

/**
 * Dịch req.query kiểu json-server sang các tham số Mongoose.
 * Trả về: { filter, sortOpt, start, limit, populate }
 */
export const buildQuery = (reqQuery = {}) => {
  // (1) _expand → .populate()
  const populate = reqQuery["_expand"];

  // (2) _sort + _order → .sort()
  const sortOpt = {};
  if (reqQuery["_sort"]) {
    const sortArr = String(reqQuery["_sort"]).split(",");
    const orderArr = String(reqQuery["_order"] || "").split(",");

    sortArr.forEach((field, index) => {
      const name = field.trim();
      if (name) sortOpt[name] = orderArr[index] === "desc" ? -1 : 1;
    });
  }

  // (3) _start / _limit → ép về SỐ và chặn biên (mục 6.5)
  const start = Math.max(Number(reqQuery["_start"]) || 0, 0);
  const rawLimit = Math.max(Number(reqQuery["_limit"]) || 0, 0);
  const limit = rawLimit > 100 ? 100 : rawLimit; // trần 100 bản ghi / request

  // (4) Phần còn lại → filter. Loại BỎ CẢ _start và _limit (sửa lỗi mục 6.1)
  const { _expand, _sort, _order, _start, _limit, ...query } = reqQuery;

  const filter = {};
  Object.keys(query).forEach((item) => {
    const value = query[item];

    if (item.endsWith("_like")) {
      // endsWith thay cho includes("like") — sửa lỗi mục 6.2
      const key = item.slice(0, item.length - "_like".length);
      mergeFilter(filter, key, { $regex: new RegExp(escapeRegExp(value), "i") });
    } else if (item.endsWith("_ne")) {
      const key = item.slice(0, item.length - "_ne".length);
      mergeFilter(filter, key, { $nin: [].concat(value) });
    } else if (item.endsWith("_gte")) {
      const key = item.slice(0, item.length - "_gte".length);
      mergeFilter(filter, key, { $gte: value });
    } else if (item.endsWith("_lte")) {
      const key = item.slice(0, item.length - "_lte".length);
      mergeFilter(filter, key, { $lte: value });
    } else if (item === "q") {
      // q rỗng thì BỎ QUA hẳn, không tạo điều kiện rác
      if (value) filter["$text"] = { $search: `"${value}"` };
    } else {
      // [].concat() xử lý cả trường hợp trùng key (mục 6.6)
      mergeFilter(filter, item, { $in: [].concat(value) });
    }
  });

  return { filter, sortOpt, start, limit, populate };
};

export default buildQuery;
```

**Vì sao hàm nhận `reqQuery` chứ không nhận cả `req`?** Vì như thế hàm **không phụ thuộc Express** —
bạn test được bằng cách gọi thẳng `buildQuery({ _sort: "price" })` mà không cần dựng server.

### Bước 2 — Dùng `buildQuery()` trong controller Topping

Mở `yotea-be/src/controllers/topping.js` (file bạn viết ở [Bài 06](06-vong-doi-mot-request.md), hoàn
thiện ở [Bài 07](07-crud-category.md)) và thay **toàn bộ** hàm `list` cũ bằng:

```js
// yotea-be/src/controllers/topping.js  ← file của BẠN, sửa lại hàm list
import Topping from "../models/topping";
import { buildQuery } from "../utils/buildQuery";   // ← thêm dòng import này

export const list = async (req, res) => {
  const { filter, sortOpt, start, limit, populate } = buildQuery(req.query);

  try {
    const toppings = await Topping.find(filter)
      .select("-__v")
      .populate(populate)
      .skip(start)
      .limit(limit)
      .sort(sortOpt)
      .exec();
    res.json(toppings);
  } catch (error) {
    res.status(400).json({
      message: "Không tìm thấy topping",
      error,
    });
  }
};
```

Hàm `list` từ **~50 dòng** rút xuống **18 dòng**, phần logic khó nhất chỉ còn **một dòng**. Giữ
nguyên `create`, `read`, `update`, `remove` bạn đã viết ở bài trước.

### Bước 3 — Bổ sung text index cho model Topping (để `?q=` chạy được)

Mở `yotea-be/src/models/topping.js` (file của bạn, tạo ở [Bài 05](05-mongoose-model.md)). Nếu chưa
có dòng text index, thêm **ngay trước** dòng `export default`:

```js
// yotea-be/src/models/topping.js  ← file của BẠN, thêm 1 dòng
toppingSchema.index({ "$**": "text" });

export default model("Topping", toppingSchema);
```

Đây chính là dòng mà mọi model của dự án đều có (`yotea-be/src/models/product.js:43`), và là điều
kiện bắt buộc để `$text` hoạt động.

### Bước 4 — Nạp dữ liệu mẫu

Chạy `npm start` tại `yotea-be`, rồi dùng Postman `POST` **5 topping** vào endpoint tạo topping của
bạn (`http://localhost:8080/api/toppings/...`):

```json
{ "name": "Trân châu đen",   "price": 5000 }
{ "name": "Trân châu trắng", "price": 6000 }
{ "name": "Thạch dừa",       "price": 7000 }
{ "name": "Pudding trứng",   "price": 10000 }
{ "name": "Kem cheese",      "price": 12000 }
```

> ⚠️ **Một cái bẫy đang chờ Topping của bạn:** trang thanh toán gửi lên `toppingId` và `toppingPrice`
> khi đặt hàng (`yotea-fe/src/pages/user/cart/CheckoutPage.js:73-74` và `:85-86`), nhưng schema
> `yotea-be/src/models/orderDetail.js:3-31` **chỉ khai 6 trường**: `orderId`, `productId`,
> `productPrice`, `quantity`, `ice`, `sugar` — **không có** `toppingId`/`toppingPrice`. Mongoose ở
> chế độ `strict` mặc định **âm thầm loại bỏ** trường lạ: HTTP vẫn `200`, không lỗi, nhưng dữ liệu
> topping **bốc hơi**. Ta sẽ nối quan hệ này cho đúng ở [Bài 10](10-quan-he-va-populate.md).

---

## 8. ✅ Kiểm chứng kết quả

```bash
# đứng tại thư mục yotea-be
npm start
```

Terminal phải hiện `Server is running on port 8080` và `Connected to MongoDB`. Giờ bắn lần lượt các
URL sau bằng Postman (method `GET`):

**8.1. Sắp xếp theo giá giảm dần**

```
GET http://localhost:8080/api/toppings?_sort=price&_order=desc
```

Phải trả về 5 topping, đắt nhất đứng đầu:

```json
[
  { "_id": "...", "name": "Kem cheese",      "price": 12000, "slug": "kem-cheese" },
  { "_id": "...", "name": "Pudding trứng",   "price": 10000, "slug": "pudding-trung" },
  { "_id": "...", "name": "Thạch dừa",       "price": 7000,  "slug": "thach-dua" },
  { "_id": "...", "name": "Trân châu trắng", "price": 6000,  "slug": "tran-chau-trang" },
  { "_id": "...", "name": "Trân châu đen",   "price": 5000,  "slug": "tran-chau-den" }
]
```

Đổi `_order=asc` → thứ tự phải **đảo ngược hoàn toàn**.

| # | URL | Kết quả phải thấy |
|---|---|---|
| 8.2 | `?_sort=price&_order=desc&_limit=2` | Đúng **2 phần tử**: Kem cheese, Pudding trứng |
| 8.3 | `?_sort=price&_order=desc&_start=2&_limit=2` | **2 phần tử tiếp theo** (tức "trang 2"): Thạch dừa, Trân châu trắng |
| 8.4 | `?q=trân châu` | **2 topping** có cụm từ "trân châu". Nếu nhận `400` kèm `text index required` → bạn chưa làm Bước 3 |
| 8.5 | `?name_like=chau` | Vẫn ra 2 topping trân châu nếu tên/slug chứa chuỗi `chau` — vì `_like` dùng RegExp trên chuỗi, không phải token |
| 8.6 | `?price_gte=6000&price_lte=10000` | **3 topping**: Trân châu trắng, Thạch dừa, Pudding trứng |

> 💡 **Mẹo debug:** muốn nhìn tận mắt `filter` mà `buildQuery()` sinh ra, thêm tạm
> `console.log(JSON.stringify(filter))` ngay sau dòng gọi `buildQuery(...)` rồi xem terminal backend.
> Đây là cách nhanh nhất để hiểu vì sao một URL không trả đúng dữ liệu.

---

## 9. 🐞 Lỗi thường gặp

| Thông báo lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `400 { message: "Không tìm thấy topping" }` khi gõ `?q=…` | Model chưa có text index | Thêm `toppingSchema.index({ "$**": "text" })` (Bước 3) rồi khởi động lại server |
| `MongoServerError: text index required for $text query` | Như trên, nhìn thấy trong terminal | Như trên. Vẫn lỗi thì xoá collection `toppings` để Mongoose dựng lại index |
| `?q=` (bỏ trống) trả về sai | Ở code gốc: `q=""` là falsy → rơi vào nhánh 6 → sinh `{ q: { $in: [""] } }` | `buildQuery()` của bạn đã xử lý: `q` rỗng thì bỏ qua hẳn |
| `CastError: Cast to Number failed for value "abc"` | Gõ `?_limit=abc` ở controller gốc | `buildQuery()` dùng `Number(...) \|\| 0` nên đã miễn nhiễm |
| Lọc `?price_gte=20000` không ra gì | `price` trong schema là `String` chứ không phải `Number` → so sánh chuỗi | Sửa kiểu dữ liệu trong model thành `Number` |
| `_order=DESC` mà vẫn sắp tăng dần | Code so sánh `=== "desc"`, phân biệt hoa/thường | Gõ chữ thường, hoặc thêm `.toLowerCase()` vào `buildQuery()` |
| `TypeError: Object.hasOwn is not a function` | Node cũ hơn 16.9 | Nâng Node lên ≥ 18 |
| `Cannot find module '../utils/buildQuery'` | Sai đường dẫn tương đối | Từ `src/controllers/` lùi một cấp là `src/`, nên đúng phải là `../utils/buildQuery` |
| Sửa `buildQuery.js` mà API không đổi | `nodemon` chưa nạp lại | Xem terminal có dòng `[nodemon] restarting` chưa; nếu không, gõ `rs` + Enter |

---

## 10. 📝 Bài tập

**Bài 1.** Cho URL sau, hãy viết ra **chính xác** `filter`, `sortOpt`, `start`, `limit`, `populate`
mà controller **gốc của dự án** (`yotea-be/src/controllers/product.js:107-180`) sinh ra:

```
GET /api/products?categoryId=6249a1&price_gte=20000&price_lte=50000&name_like=tra&_expand=categoryId&_sort=price&_order=desc&_start=9&_limit=9
```

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Duyệt lần lượt theo đúng thứ tự `Object.keys()`:

| `item` | Nhánh trúng | Đóng góp |
|---|---|---|
| `categoryId` | 6 | `categoryId: { $in: ["6249a1"] }` |
| `price_gte` | 3 | `price: { $gte: "20000" }` |
| `price_lte` | 4 — `filter` đã có `price` → **gộp** | `price: { $gte: "20000", $lte: "50000" }` |
| `name_like` | 1 (kiểm tra đầu tiên) | `name: { $in: [/tra/i] }` |
| `_start` | 6 | `_start: { $in: ["9"] }` ← rác |
| `_limit` | 6 | `_limit: { $in: ["9"] }` ← rác |

```js
filter = {
  categoryId: { $in: ["6249a1"] },
  price: { $gte: "20000", $lte: "50000" },
  name: { $in: [/tra/i] },
  _start: { $in: ["9"] },   // bị Mongoose 6 (strictQuery) loại bỏ
  _limit: { $in: ["9"] },   // bị Mongoose 6 (strictQuery) loại bỏ
};
sortOpt  = { price: -1 };
start    = "9";     // chuỗi, không phải số
limit    = "9";     // chuỗi
populate = "categoryId";
```

**Bẫy dễ sai:** nhiều bạn xếp `name_like` vào nhánh `_ne`, hoặc quên rằng nhánh `like` được kiểm tra
**trước tiên**. Cứ nhớ thứ tự: `like` → `_ne` → `_gte` → `_lte` → `q` → `else`.

</details>

**Bài 2.** Thêm vào `buildQuery()` hai quy ước json-server có nhưng dự án chưa làm: `_page=<n>`
(số trang, bắt đầu từ 1, dùng chung với `_limit` để tự tính `start`) và `*_gt` / `*_lt` (lớn hơn /
nhỏ hơn **nghiêm ngặt**). Chú ý `_page` cũng phải bị **loại khỏi `filter`**.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Sửa khối (3) và (4) trong `yotea-be/src/utils/buildQuery.js` (code bạn tự viết thêm):

```js
  // (3) _start / _limit / _page
  const rawLimit = Math.max(Number(reqQuery["_limit"]) || 0, 0);
  const limit = rawLimit > 100 ? 100 : rawLimit;

  const page = Math.max(Number(reqQuery["_page"]) || 0, 0);
  const start = page > 0
    ? (page - 1) * limit                                  // ưu tiên _page nếu có
    : Math.max(Number(reqQuery["_start"]) || 0, 0);

  // (4) nhớ loại thêm _page khỏi phần rest
  const { _expand, _sort, _order, _start, _limit, _page, ...query } = reqQuery;
```

Rồi thêm hai nhánh (đặt **sau** `_gte`/`_lte`):

```js
    } else if (item.endsWith("_gt")) {
      const key = item.slice(0, item.length - "_gt".length);
      mergeFilter(filter, key, { $gt: value });
    } else if (item.endsWith("_lt")) {
      const key = item.slice(0, item.length - "_lt".length);
      mergeFilter(filter, key, { $lt: value });
```

**Vì sao thứ tự quan trọng:** dùng `endsWith` thì hai nhánh **không đè nhau**, vì
`"price_gte".endsWith("_gt")` là `false` (kết thúc bằng `e`). Nhưng nếu bạn dùng `includes("_gt")`
như code gốc thì `price_gte` **sẽ bị nhánh `_gt` nuốt mất** — đúng bài học của mục 6.2.

Kiểm chứng: `GET /api/toppings?_sort=price&_order=asc&_page=2&_limit=2` phải trả về đúng 2 topping ở
vị trí thứ 3 và 4 (Thạch dừa, Pudding trứng).

</details>

**Bài 3.** *(Khó)* Frontend đang gọi API **hai lần** cho mỗi trang danh sách: một lần lấy dữ liệu
trang hiện tại, một lần lấy **toàn bộ** chỉ để **đếm tổng số** — xem hai hàm `getData` và
`getTotalProduct` trong `yotea-fe/src/components/user/ProductContent.js:31-67`. Hãy đề xuất cách sửa
**phía backend** để bỏ được request thứ hai.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

json-server thật giải quyết bằng **header `X-Total-Count`**. Ta bắt chước y hệt; vì đó là **header**
nên body vẫn là mảng thuần → frontend cũ **không cần sửa gì** vẫn chạy được.

Trong `list` của Topping (code bạn tự viết thêm):

```js
export const list = async (req, res) => {
  const { filter, sortOpt, start, limit, populate } = buildQuery(req.query);

  try {
    // Đếm tổng số bản ghi KHỚP ĐIỀU KIỆN (không tính skip/limit)
    const total = await Topping.countDocuments(filter).exec();

    const toppings = await Topping.find(filter)
      .select("-__v")
      .populate(populate)
      .skip(start)
      .limit(limit)
      .sort(sortOpt)
      .exec();

    res.set("X-Total-Count", total);
    res.json(toppings);
  } catch (error) {
    res.status(400).json({ message: "Không tìm thấy topping", error });
  }
};
```

**Hai điểm phải lưu ý:**

1. `countDocuments(filter)` dùng **cùng `filter`** nhưng **không** dùng `skip`/`limit` — nếu không
   thì tổng số sẽ luôn bằng số bản ghi của một trang.
2. `app.js` đang bật `app.use(cors())` mở cho mọi origin, nhưng mặc định trình duyệt **chỉ cho
   JavaScript đọc vài header cơ bản**. Muốn frontend đọc được `X-Total-Count`, server phải khai thêm
   `Access-Control-Expose-Headers`. Đây là cái bẫy rất hay gặp — bạn sẽ gặp lại ở
   [Bài 34](34-refactor-du-an.md).

**Vì sao đây là cải tiến lớn?** Cách hiện tại gọi `getAll(0, 0)` tức `_limit=0` = **lấy toàn bộ** sản
phẩm chỉ để lấy con số `data.length`. Với 10 sản phẩm thì không sao; với 10.000 sản phẩm thì mỗi lần
mở trang là kéo về vài MB JSON rồi vứt đi. `countDocuments` chỉ trả về **một con số**.

</details>

---

## 📌 Tóm tắt

- Backend Yotea **bắt chước protocol của json-server** để frontend đã viết sẵn theo json-server không
  phải sửa một dòng nào khi chuyển sang API thật. Đây là mẫu thiết kế **Adapter**.
- Hàm `list()` gồm **5 khối**: đọc `_expand` → dựng `sortOpt` từ `_sort`/`_order` (ghép theo cặp chỉ
  số, hỗ trợ nhiều trường qua dấu phẩy) → lấy `_start`/`_limit` → **vòng `forEach` 6 nhánh** dựng
  `filter` → nối chuỗi `find().select().populate().skip().limit().sort().exec()`.
- Sáu nhánh xét theo đúng thứ tự `*_like` → `*_ne` → `*_gte` → `*_lte` → `q` → **`else`**; ánh xạ lần
  lượt sang `RegExp` trong `$in`, `$nin`, `$gte`, `$lte`, `$text/$search`, `$in`.
- `Product.find()` chỉ **mô tả** truy vấn; đến `.exec()` (hoặc `await`) mới thật sự chạm database.
- Đoạn code này bị **copy-paste sang 13 controller** (chỉ `auth.js` không có `list`) ≈ 530 dòng trùng
  lặp — vi phạm **DRY**: sửa một bug phải mở 13 file, sót một file là hệ thống chạy không đồng nhất.
- Sáu lỗi cần nhớ: `_start`/`_limit` lọt vào `filter` (chỉ sống nhờ `strictQuery` của Mongoose 6);
  `indexOf("_")` cắt sai tên trường có gạch dưới; `new RegExp()` từ input người dùng gây **ReDoS**;
  `$text` cần **text index** và không tìm được **từ khoá một phần**; `_start`/`_limit` là **chuỗi**;
  query trùng key sinh **mảng lồng mảng**.
- Cách chữa gọn nhất: tách **một** hàm `buildQuery(req.query)` trả về
  `{ filter, sortOpt, start, limit, populate }` — bạn đã tự viết nó và gắn vào API Topping.

**Từ khoá tra cứu thêm:** `json-server query params`, `mongoose query chaining`, `mongodb $text search`,
`mongodb wildcard text index`, `strictQuery mongoose 6`, `ReDoS regular expression`, `DRY principle`,
`X-Total-Count pagination header`

➡️ **Bài tiếp theo:** [10 — Quan hệ dữ liệu & `populate`](10-quan-he-va-populate.md) — bạn đã biết
`_expand` gọi tới `.populate()`, giờ ta xem `populate` thật sự làm gì bên trong, và **nối Topping với
OrderDetail** để sửa đúng cái bug dữ liệu bốc hơi vừa nhắc ở mục 7.
