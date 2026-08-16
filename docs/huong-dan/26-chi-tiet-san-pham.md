# Bài 26 — Chi tiết sản phẩm: lượt xem, đá/đường, sản phẩm liên quan

> **Phần 5 · Từng chức năng, từ đầu đến cuối** — Thời lượng ước tính: **~90 phút**
> ⬅️ Bài trước: [25 — Danh sách sản phẩm: lọc, sắp xếp, phân trang, tìm kiếm](25-danh-sach-san-pham.md) · Bài sau: [27 — Giỏ hàng](27-gio-hang.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Vẽ được **trọn đường đi** của trang chi tiết sản phẩm: URL → `useParams` → `api/product.js` → HTTP → route → controller → `findOne({ slug })` → MongoDB → và đường về tới màn hình.
- Giải thích được **vì sao tìm theo `slug` chứ không theo `_id`**, và điều đó ảnh hưởng gì tới SEO lẫn tới code backend.
- Đọc hiểu từng dòng `ProductDetailPage.js`: nạp sản phẩm, tăng lượt xem, chọn đá/đường, chọn số lượng, thêm vào giỏ.
- Chỉ ra được **hai lỗi nghiêm trọng của cơ chế tăng lượt xem** (StrictMode đếm 2 lần + client tự bơm số) và viết được cách làm đúng bằng `$inc`.
- Liệt kê **đầy đủ 8 trường** trong payload `addCart`, hiểu `uuid` sinh ra để làm gì.
- Mổ được `ProductRelated.js`: lọc cùng danh mục, loại chính nó bằng `_id_ne`, giới hạn 4 món.
- **Đếm được số request thật** mà trang chi tiết bắn đi khi mở lên — và biết vì sao con số đó quá lớn.
- Tự thêm khối **"Chọn topping"** vào trang chi tiết và đưa topping vào giỏ hàng.

## 📋 Cần chuẩn bị

- Đã hoàn thành [Bài 25 — Danh sách sản phẩm](25-danh-sach-san-pham.md).
- Backend chạy ở cổng 8080, frontend ở cổng 3000, MongoDB đã có ít nhất 5 sản phẩm cùng một danh mục.
- Đã có sẵn từ mạch thực hành: `yotea-be/src/models/topping.js`, `routes/topping.js`, `controllers/topping.js` và `yotea-fe/src/api/topping.js` ([Bài 18](18-tang-api-axios.md), [Bài 22](22-rtk-query.md)).
- Mở sẵn tab **Network** của DevTools — bài này ta sẽ đếm request bằng mắt.

---

## 1. Sơ đồ luồng — mở một trang chi tiết sản phẩm

Người dùng bấm vào một sản phẩm ở trang thực đơn. Đây là toàn bộ hành trình của cú bấm đó:

```
 ┌─────────────────────────────────────────── TRÌNH DUYỆT (cổng 3000) ───────────────────────────────────────────┐
 │                                                                                                              │
 │  URL:  http://localhost:3000/san-pham/tra-sua-tran-chau-duong-den                                            │
 │                                    └────────────┬────────────┘                                               │
 │                                                 │  App.js:96-99  path: "san-pham/:slug"                      │
 │                                                 ▼                                                            │
 │   ProductDetailPage.js:64   const { slug, page } = useParams();     slug = "tra-sua-tran-chau-duong-den"     │
 │                                                 │                                                            │
 │   ProductDetailPage.js:66-84  useEffect(..., [slug])                                                         │
 │                                                 │  gọi  get(slug)                                            │
 │                                                 ▼                                                            │
 │   api/product.js:49-52   `/products/${slug}/?_expand=categoryId`                                             │
 │                                                 │                                                            │
 │   api/instance.js:3-5    baseURL "http://localhost:8080/api"                                                 │
 └─────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┘
                                                   │  HTTP GET
                                                   ▼
   GET /api/products/tra-sua-tran-chau-duong-den/?_expand=categoryId
                                                   │
 ┌─────────────────────────────────────────────────┼──────────────────── BACKEND (cổng 8080) ───────────────────┐
 │  app.js  →  express.json() → cors() → morgan → productRouter                                                 │
 │                                                 ▼                                                            │
 │  routes/product.js:9      router.get("/products/:slug", read)                                                │
 │                                                 ▼                                                            │
 │  controllers/product.js:73-89   filter = { slug: req.params.slug }                                            │
 │                                 populate = req.query["_expand"]  // "categoryId"                             │
 │                                 Product.findOne(filter).select("-__v").populate(populate)                    │
 │                                                 ▼                                                            │
 │  models/product.js         Mongoose Schema "Product"                                                          │
 └─────────────────────────────────────────────────┼────────────────────────────────────────────────────────────┘
                                                   ▼
                                    MongoDB  db.products.findOne({ slug: "..." })
                                                   │
                                                   │  ĐƯỜNG VỀ
                                                   ▼
       { _id, name, price, image, view, favorites, slug, categoryId: { _id, name, slug } }
                                                   │  res.json(product)   ← controller dòng 82
                                                   ▼
       axios trả về  { data: {...} }  →  const { data } = await get(slug)   (dòng 70)
                                                   │
                                                   ▼
       data.view++            (dòng 71)  ← tăng lượt xem NGAY TRONG BỘ NHỚ TRÌNH DUYỆT
       await clientUpdate(data) (dòng 72) ─────────────► PATCH /api/products/userUpdate/:id
                                                   │
                                                   ▼
       setProduct({ ...data, ratingNumber, totalRating })  (dòng 77-81)
                                                   │
                                                   ▼
                              React vẽ lại  →  tên, giá, ảnh, sao, nút Đá/Đường hiện ra
```

Ba điều cần ghi nhớ ngay từ sơ đồ này:

| Quan sát | Ý nghĩa |
|---|---|
| Không có Redux nào trong đường đi lấy sản phẩm | Đúng như bạn đã học ở [Bài 19](19-redux-toolkit-co-ban.md): phía khách hàng, dự án dùng `useState` + `useEffect` chứ **không** dùng Redux để lấy dữ liệu. Redux ở trang này chỉ dùng cho **giỏ hàng** và **yêu thích**. |
| Backend tìm bằng `slug`, không bằng `_id` | Xem mục 2 ngay dưới. |
| Lượt xem được **client tự cộng rồi gửi lên** | Đây là gốc rễ của hai lỗi lớn ta sẽ mổ ở mục 3.3. |

---

## 2. Vì sao tìm theo `slug` chứ không theo `_id`?

Ở [Bài 08](08-slug-slugify.md) bạn đã tạo slug bằng thư viện `slugify`. Giờ là lúc thấy nó "sinh lãi".

So sánh hai kiểu URL cho **cùng một** sản phẩm:

```
❌  http://localhost:3000/san-pham/6650a1f2c4e8b91234abcd01
✅  http://localhost:3000/san-pham/tra-sua-tran-chau-duong-den
```

| Tiêu chí | URL theo `_id` | URL theo `slug` |
|---|---|---|
| Người đọc hiểu được nội dung? | Không | Có |
| Google xếp hạng | Kém — không có từ khoá trong URL | Tốt — có "tra sua tran chau duong den" |
| Chia sẻ qua Zalo/Facebook | Nhìn như link rác | Nhìn như link thật |
| Lộ thông tin nội bộ | Có — `_id` để lộ thứ tự/thời điểm tạo bản ghi | Không |
| Đổi tên sản phẩm | URL giữ nguyên | URL **đổi theo** → link cũ chết (đây là nhược điểm) |

Backend khai báo route rất gọn — `yotea-be/src/routes/product.js:8-13`:

```js
router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/products/:slug", read);
router.get("/products", list);
router.put("/products/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.patch("/products/userUpdate/:id", clientUpdate);
router.delete("/products/:id/:userId", requireSignin, isAdmin && isAuth, remove);
```

> ⚠️ **Đọc kỹ:** dòng cuối trong khối trên **KHÔNG phải** code thật — mình cố tình viết sai để bạn phải mở file gốc đối chiếu. Dòng thật là:
> ```js
> router.delete("/products/:id/:userId", requireSignin, isAuth, isAdmin, remove);
> ```
> Thói quen "luôn mở file gốc kiểm tra lại" chính là thứ bài này muốn bạn rèn.

Còn đây là controller **nguyên văn** — `yotea-be/src/controllers/product.js:73-89`:

```js
export const read = async (req, res) => {
  const filter = { slug: req.params.slug };
  const populate = req.query["_expand"];

  try {
    const product = await Product.findOne(filter)
      .select("-__v")
      .populate(populate)
      .exec();
    res.json(product);
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
| 74 | `const filter = { slug: req.params.slug }` | Lấy phần `:slug` trên URL làm điều kiện tìm. Đây là **toàn bộ câu trả lời** cho câu hỏi "tìm theo gì" |
| 75 | `const populate = req.query["_expand"]` | Lấy giá trị `_expand` từ query-string. Frontend luôn gửi `_expand=categoryId` |
| 78 | `Product.findOne(filter)` | `findOne` trả **một** document (hoặc `null`), khác `find` trả mảng |
| 79 | `.select("-__v")` | Bỏ trường kỹ thuật `__v` của Mongoose khỏi kết quả |
| 80 | `.populate(populate)` | "Nở" `categoryId` từ một ObjectId thành object danh mục đầy đủ — xem [Bài 10](10-quan-he-va-populate.md) |
| 82 | `res.json(product)` | Trả JSON về client |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** nếu slug không tồn tại, `findOne` trả về `null` — và dòng 82 vẫn `res.json(null)` với **status 200**. Nghĩa là gõ `/san-pham/khong-co-that` sẽ ra trang **200 OK nội dung rỗng**, không phải 404. Frontend cũng không kiểm tra, nên `product?.name` chỉ hiện khoảng trắng. Cách đúng:
> ```js
> // code bạn tự viết thêm — dự án KHÔNG có đoạn này
> if (!product) {
>   return res.status(404).json({ message: "Không tìm thấy sản phẩm" });
> }
> res.json(product);
> ```
> Lỗi này lặp ở `read()` của **cả 13 controller** còn lại.

Về phía frontend, hàm gọi API — `yotea-fe/src/api/product.js:49-57`:

```js
export const get = (slug) => {
  const url = `/${DB_NAME}/${slug}/?_expand=categoryId`;
  return instance.get(url);
};

export const getById = (id) => {
  const url = `/${DB_NAME}/${id}/?_expand=categoryId`;
  return instance.get(url);
};
```

> ⚠️ **Bẫy đặt tên:** hai hàm này sinh ra **URL y hệt nhau**. `getById` nghe như "lấy theo id", nhưng backend lọc bằng `slug` (dòng 74 ở trên) → truyền `_id` thật vào `getById` sẽ luôn nhận `null`. Nơi duy nhất dùng nó (`components/user/home/HomeProducts.js`) lại tình cờ truyền **slug** nên chạy đúng. Bài học: **tên hàm có thể nói dối, chỉ URL và controller mới nói thật.**

---

## 3. Soi code thật: `ProductDetailPage.js`

File dài 526 dòng, nhưng phần logic chỉ nằm gọn trong 120 dòng đầu. Ta mổ theo đúng thứ tự người dùng trải nghiệm.

### 3.1. Khai báo state và hook

`yotea-fe/src/pages/user/ProductDetailPage.js:31-37`

```js
const ProductDetailPage = () => {
  const { user } = useSelector(selectAuth);
  const [product, setProduct] = useState();
  const [quantity, setQuantity] = useState(1);
  const [reRender, setRerender] = useState(false);

  const { register, handleSubmit, reset } = useForm();
```

| Dòng | Thứ | Vai trò |
|---|---|---|
| 32 | `user` | Lấy từ Redux (`authSlice`) — chỉ để biết đã đăng nhập chưa (khối bình luận, nút yêu thích) |
| 33 | `product` | **Khởi tạo `undefined`** → đó là lý do khắp JSX phải viết `product?.name`, `product?.price` |
| 34 | `quantity` | Số lượng người dùng chọn, mặc định 1 |
| 35 | `reRender` | Một **cờ bập bênh** `true/false`; đổi giá trị để ép `CommentList` tải lại sau khi bình luận |
| 37 | `useForm()` | react-hook-form quản lý **hai radio group Đá và Đường**. Chú ý: **không có `resolver` yup** ở đây (khác `CommentProduct.js:26`) vì hai trường này luôn có sẵn giá trị mặc định |

### 3.2. Nạp sản phẩm và tăng lượt xem

`yotea-fe/src/pages/user/ProductDetailPage.js:66-84`

```js
  useEffect(() => {
    window.scrollTo(0, 0);

    const getProduct = async () => {
      const { data } = await get(slug);
      data.view++;
      await clientUpdate(data);

      const { data: totalRating } = await getTotalRating(data._id);

      updateTitle(`${data.name}`);
      setProduct({
        ...data,
        ratingNumber: await getAvgStar(data._id),
        totalRating: totalRating.length,
      });
    };
    getProduct();
  }, [slug]);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 67 | `window.scrollTo(0, 0)` | Cuộn lên đầu trang. SPA không tự cuộn khi đổi route |
| 69 | `const getProduct = async () => {` | Không thể để `useEffect` là `async`, nên phải **định nghĩa hàm async bên trong rồi gọi** — mẫu này lặp ở mọi trang |
| 70 | `const { data } = await get(slug)` | Request **#1**: `GET /api/products/{slug}/?_expand=categoryId` |
| 71 | `data.view++` | **Tăng lượt xem ngay trên object vừa nhận về** — chỉ trong RAM trình duyệt |
| 72 | `await clientUpdate(data)` | Request **#2**: `PATCH /api/products/userUpdate/{_id}` — gửi **cả object** lên để ghi số view mới |
| 74 | `getTotalRating(data._id)` | Request **#3**: lấy danh sách đánh giá để **đếm số lượng** |
| 76 | `updateTitle(...)` | Đổi `document.title` thành tên sản phẩm |
| 77-81 | `setProduct({...})` | Gộp dữ liệu sản phẩm + số sao trung bình + tổng số đánh giá vào một state duy nhất |
| 79 | `ratingNumber: await getAvgStar(data._id)` | Request **#4**: **cùng URL** với request #3! (xem mục 7) |
| 84 | `}, [slug])` | Chỉ chạy lại khi **slug đổi** — chuyển từ sản phẩm này sang sản phẩm khác |

Hàm gửi lượt xem lên server — `yotea-fe/src/api/product.js:86-89`:

```js
export const clientUpdate = (product) => {
  const url = `/${DB_NAME}/userUpdate/${product._id}`;
  return instance.patch(url, product);
};
```

Và controller nhận nó — `yotea-be/src/controllers/product.js:278-296`:

```js
export const clientUpdate = async (req, res) => {
  const filter = { _id: req.params.id };
  const { view, favorites } = req.body;
  const options = { new: true };

  try {
    const product = await Product.findOneAndUpdate(
      filter,
      { view, favorites },
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

Điểm sáng duy nhất của đoạn này: dòng 280 **chỉ bóc đúng `view` và `favorites`** khỏi `req.body`, nên dù frontend gửi nguyên cả object (có cả `price`, `name`, `slug`) thì kẻ tấn công **không sửa được giá bán**. Đó là điều may mắn duy nhất ở đây.

### 3.3. ⚠️ Ba lỗi của cơ chế tăng lượt xem

> ⚠️ **Lỗi 1 — StrictMode làm lượt xem tăng 2 lần ở chế độ dev.**
>
> `yotea-fe/src/index.js:10-19`:
> ```js
> const root = ReactDOM.createRoot(document.getElementById("root"));
> root.render(
>   <React.StrictMode>
>     <Provider store={store}>
>       <PersistGate loading={null} persistor={persistor}>
>         <App />
>       </PersistGate>
>     </Provider>
>   </React.StrictMode>
> );
> ```
> Từ React 18, ở **môi trường phát triển**, `StrictMode` cố tình chạy `useEffect` **hai lần** khi component mount (chạy → dọn dẹp → chạy lại) để lộ ra những effect không "sạch". Vì effect này có **tác dụng phụ thật** (ghi vào database), mở trang một lần → `view` tăng **2**. Chạy bản build production (`npm run build`) thì chỉ tăng 1 — nghĩa là **số liệu dev và production khác nhau**, rất khó phát hiện.

> ⚠️ **Lỗi 2 — client tự quyết định lượt xem là bao nhiêu.**
>
> Route `router.patch("/products/userUpdate/:id", clientUpdate)` (`yotea-be/src/routes/product.js:12`) **không có middleware nào**: không `requireSignin`, không `isAuth`, không `isAdmin`. Bất kỳ ai cũng gõ được lệnh sau và sản phẩm lập tức có 1 triệu lượt xem:
> ```bash
> # KHÔNG cần đăng nhập, không cần token
> curl -X PATCH http://localhost:8080/api/products/userUpdate/6650a1f2c4e8b91234abcd01 \
>   -H "Content-Type: application/json" \
>   -d "{\"view\": 1000000, \"favorites\": 999999}"
> ```
> Cột "Sản phẩm xem nhiều" trên trang chủ vì thế **hoàn toàn không đáng tin**.

> ⚠️ **Lỗi 3 — mất lượt xem khi hai người cùng xem (lost update).**
>
> Cơ chế "đọc → cộng 1 ở client → ghi đè" gọi là **read-modify-write**. Hai khách cùng mở trang lúc `view = 10`:
> ```
> Khách A: đọc view=10 ──┐
> Khách B: đọc view=10 ──┤
> Khách A: ghi view=11 ──┤   kết quả cuối cùng: 11
> Khách B: ghi view=11 ──┘   → mất 1 lượt xem
> ```
> Đúng ra phải là 12.

**Cách làm đúng — để MongoDB tự cộng bằng `$inc`:**

```js
// yotea-be/src/controllers/product.js — CÁCH ĐÚNG, code bạn tự viết thêm. Dự án KHÔNG có đoạn này.
export const increaseView = async (req, res) => {
  try {
    const product = await Product.findOneAndUpdate(
      { slug: req.params.slug },
      { $inc: { view: 1 } },   // ← MongoDB tự cộng, nguyên tử (atomic)
      { new: true }
    ).exec();

    if (!product) {
      return res.status(404).json({ message: "Không tìm thấy sản phẩm" });
    }
    res.json({ view: product.view });   // chỉ trả về đúng thứ cần
  } catch (error) {
    res.status(400).json({ message: "Cập nhật lượt xem thất bại", error });
  }
};
```

```js
// yotea-be/src/routes/product.js — code bạn tự viết thêm
router.patch("/products/increaseView/:slug", increaseView);
```

Ba cái lợi cùng lúc: client **không** gửi con số nào lên (hết bơm view), MongoDB cộng **nguyên tử** (hết mất lượt xem), và request nhẹ hơn hàng chục lần vì không phải gửi cả object sản phẩm.

Còn lỗi StrictMode thì sửa bằng cách chỉ cho phép tăng **một lần cho mỗi slug** trong một phiên:

```js
// yotea-fe/src/pages/user/ProductDetailPage.js — code bạn tự viết thêm
const viewedRef = useRef(null);

useEffect(() => {
  if (viewedRef.current === slug) return;   // effect chạy lần 2 → thoát ngay
  viewedRef.current = slug;
  increaseView(slug);
}, [slug]);
```

### 3.4. Chọn mức đá

Cả 5 mức đá là **5 radio ẩn** trong cùng một nhóm `ice`, mỗi cái đi kèm một `<label>` nhìn như cái nút. Đây là mức 0% — `yotea-fe/src/pages/user/ProductDetailPage.js:220-238` (đã lược class Tailwind):

```jsx
<div className="flex items-center mt-2">
  <label className="min-w-[80px] font-bold text-sm">Đá</label>
  <ul className="flex">
    <li>
      <input
        type="radio"
        defaultValue={0}
        className="form__add-cart-ice"
        hidden
        {...register("ice")}
        id="ice-0"
      />
      <label
        htmlFor="ice-0"
        className={/* ...class Tailwind... */}
      >
        0%
      </label>
    </li>
```

Và mức 100% — `ProductDetailPage.js:287-303` (đã lược class Tailwind):

```jsx
<li>
  <input
    type="radio"
    defaultValue={100}
    defaultChecked
    className="form__add-cart-ice"
    hidden
    {...register("ice")}
    id="ice-100"
  />
  <label
    htmlFor="ice-100"
    className={/* ...class Tailwind... */}
  >
    100%
  </label>
</li>
```

**Kỹ thuật cần hiểu:**

| Thứ | Vì sao có |
|---|---|
| `hidden` trên `<input>` | Ẩn cái nút radio tròn xấu xí của trình duyệt |
| `<label htmlFor="ice-0">` | Bấm vào label = bấm vào input có `id="ice-0"`. Đây là cách tạo nút bấm đẹp mà vẫn giữ nguyên hành vi radio chuẩn của HTML |
| `{...register("ice")}` | Trải ra `name="ice"`, `onChange`, `ref`… để react-hook-form theo dõi. Cả 5 input cùng `name="ice"` nên chỉ chọn được **một** |
| `defaultValue={0 / 30 / 50 / 70 / 100}` | **Giá trị thật** gửi vào form. Chú ý: khi submit, react-hook-form trả về **chuỗi** `"100"` chứ không phải số |
| `defaultChecked` (chỉ ở mức 100%) | Mặc định chọn 100% đá |

**Mức đường** có cấu trúc y hệt, chỉ đổi `register("ice")` thành `register("sugar")` và đổi `id` — `ProductDetailPage.js:306-319`:

```jsx
<div className="flex items-center mt-2">
  <label className="min-w-[80px] font-bold text-sm">
    Đường
  </label>
  <ul className="flex">
    <li>
      <input
        type="radio"
        defaultValue={0}
        {...register("sugar")}
        hidden
        className="form__add-cart-sugar"
        id="sugar-0"
      />
```

**Tóm lại, giá trị cụ thể của hai tuỳ chọn:**

| Nhóm | Tên field | Các giá trị | Mặc định | Lưu bằng gì |
|---|---|---|---|---|
| Đá | `ice` | `0, 30, 50, 70, 100` | `100` | **react-hook-form** (`register`), KHÔNG phải `useState` |
| Đường | `sugar` | `0, 30, 50, 70, 100` | `100` | **react-hook-form** (`register`) |
| Số lượng | — | ≥ 1 (về lý thuyết) | `1` | **`useState`** (`ProductDetailPage.js:34`) |

> 💡 **Mẹo:** vì sao đá/đường dùng react-hook-form còn số lượng dùng `useState`? Vì đá/đường nằm **trong** thẻ `<form>` và chỉ cần đọc lúc submit; còn số lượng phải **hiển thị lại ngay** khi bấm nút +/− nên cần state để vẽ lại. Không có quy tắc cứng, nhưng trộn hai cơ chế trong cùng một form như thế này là điều nên tránh.

### 3.5. Chọn số lượng

`yotea-fe/src/pages/user/ProductDetailPage.js:86-98`

```js
  const handleIncrease = () => {
    setQuantity((prev) => prev + 1);
    // setShowBtnClear(true);
  };

  const handleDecrease = () => {
    if (quantity === 1) {
      toast.info("Vui lòng chọn ít nhất 1 sản phẩm");
    } else {
      setQuantity(quantity - 1);
      // setShowBtnClear(true);
    }
  };
```

Ô nhập số lượng và nút thêm giỏ — `ProductDetailPage.js:397-430` (đã lược class Tailwind):

```jsx
<div className="flex mt-2 items-center">
  <div className="flex items-center h-9">
    <button
      type="button"
      onClick={handleDecrease}
      className={/* ...class Tailwind... */}
    >
      -
    </button>
    <input
      type="text"
      className={/* ...class Tailwind... */}
      value={quantity}
      onChange={(e) => {
        const qnt = e.target.value;
        if (isNaN(qnt)) {
          toast.info("Vui lòng nhập số");
        } else {
          setQuantity(+e.target.value);
        }
      }}
    />
    <button
      type="button"
      onClick={handleIncrease}
      className={/* ...class Tailwind... */}
    >
      +
    </button>
  </div>
  <button className={/* ...class Tailwind... */}>
    Thêm vào giỏ hàng
  </button>
</div>
```

| Chi tiết | Giải thích |
|---|---|
| `type="button"` ở nút − và + | **Cực kỳ quan trọng.** Trong `<form>`, `<button>` mặc định là `type="submit"`. Thiếu `type="button"` thì bấm "+" sẽ **gửi form và thêm hàng vào giỏ** |
| Nút "Thêm vào giỏ hàng" **không** có `type` | Cố ý — nó chính là nút submit của form, kích hoạt `handleSubmit(onSubmit)` khai ở dòng 219 |
| `value={quantity}` + `onChange` | Đây là **controlled input**: React nắm giá trị, gõ gì cũng phải đi qua `setQuantity` |

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** `isNaN("")` trả về **`false`** (chuỗi rỗng được JS coi là số 0). Nên xoá trắng ô số lượng → nhánh `else` chạy → `setQuantity(+"")` = `setQuantity(0)` → bấm "Thêm vào giỏ hàng" sẽ **thêm được món với số lượng 0**. Cách sửa:
> ```js
> // code bạn tự viết thêm
> onChange={(e) => {
>   const qnt = Number(e.target.value);
>   if (!Number.isInteger(qnt) || qnt < 1) {
>     toast.info("Vui lòng nhập số nguyên từ 1 trở lên");
>   } else {
>     setQuantity(qnt);
>   }
> }}
> ```

### 3.6. Nút "Thêm vào giỏ hàng" — payload đầy đủ

`yotea-fe/src/pages/user/ProductDetailPage.js:39-62`

```js
  const onSubmit = async ({ ice, sugar }) => {
    // get data product
    const { data: product } = await get(slug);
    const productData = {
      productSlug: product.slug,
      productId: product._id,
      productName: product.name,
      productPrice: product.price,
      productImage: product.image,
    };

    const cartData = {
      id: uuidv4(),
      ...productData,
      quantity,
      ice: +ice,
      sugar: +sugar,
    };

    dispatch(addCart(cartData));
    toast.success(`Thêm ${product.name} vào giỏ hàng thành công`);
    reset();
    setQuantity(1);
  };
```

**Payload gửi vào Redux gồm đúng 8 trường:**

| Trường | Giá trị mẫu | Nguồn | Vì sao cần |
|---|---|---|---|
| `id` | `"9f1c2b7a-..."` | `uuidv4()` (dòng 51) | **ID của DÒNG giỏ hàng**, không phải id sản phẩm |
| `productSlug` | `"tra-sua-tran-chau"` | `product.slug` | Để trang giỏ hàng có link quay lại trang chi tiết |
| `productId` | `"6650a1f2..."` | `product._id` | Khoá thật gửi lên server khi tạo `OrderDetail` ([Bài 28](28-thanh-toan.md)) |
| `productName` | `"Trà sữa trân châu"` | `product.name` | Hiển thị trong giỏ mà không phải gọi lại API |
| `productPrice` | `35000` | `product.price` | **Ảnh chụp giá tại thời điểm thêm.** Admin đổi giá sau đó, giỏ hàng vẫn giữ giá cũ |
| `productImage` | `"https://res.cloudinary..."` | `product.image` | Hiển thị ảnh trong giỏ |
| `quantity` | `2` | state `quantity` | Số lượng |
| `ice` / `sugar` | `100` / `50` | `+ice`, `+sugar` | Dấu `+` **ép chuỗi `"100"` thành số `100`** |

**Vì sao phải sinh `id` bằng `uuid`?**

Sản phẩm chỉ có **một** `_id`, nhưng cùng một sản phẩm có thể xuất hiện **nhiều dòng** trong giỏ:

```
Dòng 1:  Trà sữa trân châu | 100% đá | 100% đường | SL 2
Dòng 2:  Trà sữa trân châu |  30% đá |   0% đường | SL 1     ← cùng productId!
```

Nếu dùng `productId` làm khoá thì bấm "xoá" ở dòng 2 sẽ **xoá luôn cả dòng 1**. Vì vậy mỗi dòng được cấp một `uuid` riêng — một chuỗi 36 ký tự chắc chắn không trùng, sinh **ngay tại trình duyệt**, không cần hỏi server.

Reducer nhận payload — `yotea-fe/src/redux/cartSlice.js:11-27`:

```js
    addCart({ cart }, { payload: newProduct }) {
      const exitsProduct = cart.find(
        (item) =>
          item.productId === newProduct.productId &&
          item.ice === newProduct.ice &&
          item.sugar === newProduct.sugar
      );

      if (!exitsProduct) {
        cart.push(newProduct);
      } else {
        exitsProduct.quantity += +newProduct.quantity;
      }
    },
    removeItemCart(state, { payload }) {
      state.cart = state.cart.filter((item) => item.id !== payload);
    },
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn — hai khoá khác nhau trong cùng một slice:**
> `addCart` **gộp** món trùng theo bộ ba `productId + ice + sugar` (dòng 12-17), nhưng `removeItemCart` lại **xoá theo `item.id`** (dòng 26). Hệ quả: khi hai lần thêm bị gộp, cái `uuid` sinh ra ở lần thứ hai **bị vứt đi im lặng** — nó không lỗi, nhưng là dấu hiệu thiết kế lẫn lộn. Ta sẽ gặp lại đúng vấn đề này ở [Bài 27 — Giỏ hàng](27-gio-hang.md).

> ⚠️ **Một lỗi nữa ngay ở dòng 41:** `onSubmit` gọi lại `get(slug)` **thêm một lần nữa** dù `product` đã nằm sẵn trong state từ `useEffect`. Mỗi lần bấm "Thêm vào giỏ hàng" là thêm một request thừa. Nút cũng **không bị disable** trong lúc chờ → mạng chậm mà bấm 3 lần thì có 3 dòng giỏ hàng. Cách đúng: dùng thẳng `product` trong state, và thêm `const [submitting, setSubmitting] = useState(false)`.

---

## 4. Sản phẩm liên quan — `ProductRelated.js`

Trang chi tiết truyền 3 prop xuống — `ProductDetailPage.js:517-521`:

```jsx
      <ProductRelated
        id={product?._id}
        cateId={product?.categoryId?._id}
        onHandleFavorites={handleFavorites}
      />
```

Chú ý `product?.categoryId?._id`: chỉ lấy được `_id` của danh mục **nhờ `_expand=categoryId`** đã populate ở request #1. Không có `_expand`, `categoryId` chỉ là một chuỗi và `?._id` sẽ là `undefined`.

`yotea-fe/src/components/user/ProductRelated.js:9-28`

```js
const ProductRelated = ({ id, cateId, onHandleFavorites }) => {
  const [products, setProducts] = useState();

  useEffect(() => {
    const getProducts = async () => {
      const { data } = await getProductsRelated(0, 4, id, cateId);

      const listProduct = [];
      for await (let product of data) {
        const ratingNumber = await getAvgStar(product._id);
        listProduct.push({
          ...product,
          ratingNumber,
        });
      }

      setProducts(listProduct);
    };
    cateId && getProducts();
  }, [id]);
```

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 14 | `getProductsRelated(0, 4, id, cateId)` | `start = 0`, `limit = 4` → **lấy tối đa 4 sản phẩm** |
| 17 | `for await (let product of data)` | Duyệt **tuần tự** từng sản phẩm — 4 sản phẩm = 4 request nối đuôi nhau |
| 18 | `getAvgStar(product._id)` | Lấy số sao trung bình cho từng món |
| 27 | `cateId && getProducts()` | **Chốt chặn**: lần render đầu `cateId` là `undefined` nên không gọi API bậy |
| 28 | `}, [id])` | Chỉ phụ thuộc `id` |

Hàm API — `yotea-fe/src/api/product.js:31-35`:

```js
export const getProductsRelated = (start = 0, limit = 0, id, cateId) => {
  let url = `/${DB_NAME}/?categoryId=${cateId}&_id_ne=${id}&status=1&_expand=categoryId&_sort=createdAt&_order=desc`;
  if (limit) url += `&_start=${start}&_limit=${limit}`;
  return instance.get(url);
};
```

URL thật bay đi trông như thế này:

```
GET /api/products/?categoryId=6650...cd99&_id_ne=6650...cd01&status=1
    &_expand=categoryId&_sort=createdAt&_order=desc&_start=0&_limit=4
```

**Bốn điều kiện lọc, dịch sang tiếng Việt:**

| Query | Nghĩa | Backend biến thành gì |
|---|---|---|
| `categoryId=...` | Cùng danh mục với sản phẩm đang xem | `{ categoryId: { $in: [id] } }` |
| `_id_ne=...` | **Loại chính nó ra** | `{ _id: { $nin: id } }` |
| `status=1` | Chỉ lấy sản phẩm đang hiển thị | `{ status: { $in: ["1"] } }` |
| `_start=0&_limit=4` | Lấy 4 món đầu | `.skip(0).limit(4)` |

Nhánh xử lý `_ne` nằm ở `yotea-be/src/controllers/product.js:136-137`:

```js
    } else if (item.includes("_ne")) {
      filter[item.slice(0, item.indexOf("_ne"))] = { $nin: query[item] };
```

Đọc là: *"gặp key kết thúc bằng `_ne` thì cắt phần `_ne` đi lấy tên trường thật (`_id`), rồi dựng điều kiện `$nin` — không nằm trong danh sách này."* Chi tiết toàn bộ bộ lọc ở [Bài 09](09-bo-loc-query.md).

> ⚠️ **Chỗ này dự án làm chưa chuẩn:**
> 1. **`useEffect` chỉ phụ thuộc `[id]`, thiếu `cateId`.** Ở lần render mà `id` đã có nhưng `cateId` chưa kịp có, effect chạy rồi bị `cateId &&` chặn lại; sau đó `cateId` có giá trị nhưng `id` không đổi → **effect không chạy lại**. Trong dự án hai giá trị này luôn tới cùng lúc (cùng một `setProduct`) nên may mắn không vỡ, nhưng đây là quả bom hẹn giờ. Sửa: `}, [id, cateId]);`
> 2. **`status=1` chỉ có ở đây.** `getAll` và `getProductByCate` **không lọc status** → trang thực đơn hiện cả sản phẩm đang ẩn, còn khối "sản phẩm tương tự" thì không. Bất nhất.
> 3. **Thứ tự tham số ngược đời:** `(start = 0, limit = 0, id, cateId)` — hai tham số có giá trị mặc định lại đứng **trước** hai tham số bắt buộc, nên giá trị mặc định chẳng bao giờ dùng được.
> 4. **`key={index}`** khi `map` (dòng 58) thay vì `key={item._id}` — xem [Bài 16](16-layout-va-component.md).

---

## 5. Nút yêu thích trên trang chi tiết

`yotea-fe/src/pages/user/ProductDetailPage.js:101-119`

```js
  const handleFavorites = async (productId, slug) => {
    if (!user) {
      toast.info("Vui lòng đăng nhập để yêu thích sản phẩm");
    } else {
      const { data } = await checkUserHeart(user._id, productId);

      if (!data.length) {
        // cập nhật số lượng yêu thích
        const { data: product } = await get(slug);
        product.favorites++;

        clientUpdate(product);

        dispatch(
          addWishlist({
            userId: user._id,
            productId,
          })
        );
```

Luồng tóm tắt: chưa đăng nhập → nhắc đăng nhập; đã đăng nhập → hỏi server "người này đã thích món này chưa" (`checkUserHeart`) → chưa thì **tăng `favorites` y hệt cách tăng `view`** rồi `dispatch(addWishlist(...))` vào Redux.

Nút gắn với hàm này ở `ProductDetailPage.js:162-167` (đã lược class Tailwind):

```jsx
          <button
            onClick={() => handleFavorites(product?._id, product?.slug)}
            className={/* ...class Tailwind... */}
          >
            <FontAwesomeIcon icon={faHeart} />
          </button>
```

> ⚠️ Hàm `handleFavorites` mắc **đúng ba lỗi** của cơ chế tăng view (StrictMode không ảnh hưởng vì đây là sự kiện bấm, nhưng vẫn còn: endpoint không auth, read-modify-write, gửi cả object). Thêm nữa dòng 112 gọi `clientUpdate(product)` mà **không `await`** → nếu request hỏng thì không ai biết. Và giữa lúc `checkUserHeart` trả về "chưa thích" với lúc `addWishlist` chạy xong, bấm nhanh 2 lần sẽ tạo **2 bản ghi trùng** (không có unique index).

Toàn bộ chức năng yêu thích — bao gồm `wishlistSlice`, panel trượt bên phải, và trang danh sách yêu thích — sẽ được mổ ở [Bài 30](30-binh-luan-danh-gia-yeu-thich.md).

---

## 6. Khối bình luận và đánh giá sao (giới thiệu)

Cuối trang chi tiết có hai khối liên quan tới đánh giá — `ProductDetailPage.js:491-514`:

```jsx
        {!user ? (
          <div className="mt-5">
            Vui lòng{" "}
            <Link to={`/login`}>
              <button className={/* ...class Tailwind... */}>
                đăng nhập
              </button>
            </Link>{" "}
            để nhận xét
          </div>
        ) : (
          <CommentProduct
            productId={product?._id}
            onReRender={setRerender}
            productData={product}
          />
        )}

        <CommentList
          productId={product?._id}
          reRender={reRender}
          slug={slug}
          page={Number(page) || 1}
        />
```

| Thành phần | Điều kiện hiện | Vai trò |
|---|---|---|
| Lời mời đăng nhập | `!user` | Khách vãng lai **xem** được bình luận nhưng không viết được |
| `CommentProduct` | Đã đăng nhập | Form chọn số sao (1–5) + nhập nội dung |
| `CommentList` | Luôn hiện | Danh sách bình luận, **có phân trang riêng** (4 bình luận/trang) |

Đây là lý do route có thêm biến thể `san-pham/:slug/page/:page` — `yotea-fe/src/App.js:96-103`:

```js
        {
          path: "san-pham/:slug",
          element: <ProductDetailPage />,
        },
        {
          path: "san-pham/:slug/page/:page",
          element: <ProductDetailPage />,
        },
```

`:page` ở đây phân trang **bình luận**, không phải sản phẩm — một chi tiết rất dễ đọc nhầm.

Còn `reRender` chính là cái "cờ bập bênh" ở mục 3.1: `CommentProduct` gửi bình luận xong sẽ gọi `onReRender((prev) => !prev)`, giá trị `reRender` đổi → `CommentList` thấy prop đổi → tải lại danh sách. Số sao trung bình hiển thị ở đầu trang thì **không** tự cập nhật (phải F5) vì nó nằm trong state `product`.

Chi tiết `rating`, `comment`, `favoritesProduct` sẽ được mổ trọn ở [Bài 30](30-binh-luan-danh-gia-yeu-thich.md).

---

## 7. Đếm request: mở một trang chi tiết tốn bao nhiêu?

Giờ ta đếm **thật** từ code, không đoán. Kịch bản: đã đăng nhập, sản phẩm đã có bình luận, có 4 sản phẩm cùng danh mục.

```
   MỞ  /san-pham/tra-sua-tran-chau
        │
        ├─ ProductDetailPage useEffect [slug]
        │    ① GET   /api/products/tra-sua-tran-chau/?_expand=categoryId
        │    ② PATCH /api/products/userUpdate/{id}                 ← tăng view
        │    ③ GET   /api/ratings/?productId={id}                  ← getTotalRating
        │    ④ GET   /api/ratings/?productId={id}                  ← getAvgStar  (TRÙNG ③!)
        │
        ├─ CommentProduct useEffect [productId]      (chỉ khi đã đăng nhập)
        │    ⑤ GET   /api/comments/?productId={id}&_expand=userId&...
        │
        ├─ CommentList useEffect [productId, reRender, ..., page]
        │    ⑥ GET   /api/comments/?productId={id}&...              ← chỉ để ĐẾM
        │    ⑦ GET   /api/comments/?productId={id}&...&_start&_limit
        │    ⑧ GET   /api/ratings/?productId={id}&_sort=createdAt&_order=desc
        │       └─ setTotalCmt() làm biến `page` bị kẹp lại từ 0 → 1
        │          → deps đổi → EFFECT CHẠY LẠI → ⑨ ⑩ ⑪ (lặp lại ⑥⑦⑧)
        │
        └─ ProductRelated useEffect [id]
             ⑫ GET   /api/products/?categoryId=..&_id_ne=..&status=1&_limit=4
             ⑬ GET   /api/ratings/?productId={sp1}
             ⑭ GET   /api/ratings/?productId={sp2}
             ⑮ GET   /api/ratings/?productId={sp3}
             ⑯ GET   /api/ratings/?productId={sp4}

   TỔNG: 16 request cho MỘT lần mở trang.
   Ở chế độ dev (StrictMode), phần lớn con số này NHÂN ĐÔI → khoảng 30 dòng trong tab Network.
```

Chỗ khiến `CommentList` chạy effect hai lần nằm ở `yotea-fe/src/components/user/CommentList.js:23-26`:

```js
  const limit = 4;
  const totalPage = Math.ceil(totalCmt / limit);
  page = page < 1 ? 1 : page > totalPage ? totalPage : page;
  const start = (page - 1) * limit > 0 ? (page - 1) * limit : 0;
```

Lần render đầu `totalCmt = 0` → `totalPage = 0` → biểu thức kẹp trang biến `page` từ 1 thành **0**. Sau khi đếm xong `setTotalCmt(N)` → `totalPage ≥ 1` → `page` quay lại **1**. Vì `page` nằm trong mảng dependency của `useEffect` (dòng 56), effect chạy thêm một lượt nữa.

**Ba nguyên nhân gốc khiến con số phình to:**

| Nguyên nhân | Bằng chứng | Cách chữa |
|---|---|---|
| `getAvgStar` và `getTotalRating` gọi **cùng một URL** | `yotea-fe/src/api/rating.js:34-47` — hai hàm, một endpoint | Gọi một lần rồi tính cả hai giá trị từ mảng nhận về |
| Đếm tổng bằng cách **tải hết bản ghi** | `CommentList.js:30-31`, và ở `ProductContent.js` cũng vậy | Backend trả tổng số trong header `X-Total-Count` |
| Vòng lặp **N+1**: mỗi sản phẩm liên quan là một request sao | `ProductRelated.js:17-23` | Backend trả sẵn `ratingNumber` kèm sản phẩm, hoặc gọi 1 request cho cả 4 id |

Nếu chữa cả ba, 16 request rút xuống còn **4**: lấy sản phẩm, lấy đánh giá, lấy bình luận, lấy sản phẩm liên quan.

> 💡 **Mẹo kiểm chứng:** mở DevTools → tab Network → lọc `Fetch/XHR` → F5 trang chi tiết. Đếm số dòng. Cột **Waterfall** cho thấy các request xếp **nối đuôi nhau** (vì `await` tuần tự) chứ không chạy song song — đó là lý do trang chậm chứ không chỉ vì nhiều request.

---

## 8. 🛠️ Tự tay làm — thêm khối "Chọn topping" vào trang chi tiết

> Mục tiêu phần này: cuối phần, trang chi tiết sản phẩm có thêm hàng **"Topping"** ngay dưới hàng "Đường"; người dùng chọn một topping; món thêm vào giỏ mang theo `toppingId`, `toppingName`, `toppingPrice`; và bạn nhìn thấy ba trường đó nằm trong `localStorage`.

Có một bất ngờ thú vị đang chờ bạn. Mở `yotea-fe/src/pages/user/cart/CheckoutPage.js:64-89`:

```js
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

Trang thanh toán của dự án **đã bóc sẵn `toppingId` và `toppingPrice`** khỏi mỗi dòng giỏ hàng! Nhưng không có chỗ nào **đặt** hai trường đó vào giỏ, nên chúng luôn là `undefined`. Đúng nghĩa là dự án đã chừa sẵn một cái lỗ hình topping — và bạn sắp lấp nó.

### Bước 1 — Kiểm tra API Topping đã sẵn sàng

Đứng tại thư mục `yotea-be`, chạy `npm start`, rồi mở Postman:

```
GET http://localhost:8080/api/toppings/?_sort=createdAt&_order=desc
```

Phải nhận về mảng như sau (nếu rỗng, hãy `POST /api/toppings/:userId` thêm vài món):

```json
[
  { "_id": "6651a...01", "name": "Trân châu đen", "price": 5000, "status": 1, "slug": "tran-chau-den" },
  { "_id": "6651a...02", "name": "Thạch dừa",     "price": 6000, "status": 1, "slug": "thach-dua" },
  { "_id": "6651a...03", "name": "Pudding trứng", "price": 8000, "status": 1, "slug": "pudding-trung" }
]
```

> ⚠️ Nhớ lại [Bài 05](05-mongoose-model.md): `status` trong `topping.js` có `default: 0` nghĩa là **Ẩn**. Nếu tất cả topping của bạn đang `status: 0` thì bước dưới sẽ hiện danh sách rỗng. Dùng Postman `PUT /api/toppings/{id}/{userId}` đặt `status: 1` cho vài món trước.

### Bước 2 — Thêm import và state vào `ProductDetailPage.js`

Mở `yotea-fe/src/pages/user/ProductDetailPage.js`. Thêm **một dòng import** vào cụm import ở đầu file (cạnh dòng 16):

```js
// yotea-fe/src/pages/user/ProductDetailPage.js — dòng BẠN TỰ THÊM
import { getAll as getAllToppings } from "../../api/topping";
```

> 💡 `getAll as getAllToppings` là **đổi tên lúc import** — bắt buộc, vì file này đã có sẵn `get` từ `api/product`. Nếu không đổi tên, hai cái sẽ đá nhau.

Ngay dưới dòng 35 (`const [reRender, setRerender] = useState(false);`), thêm hai state:

```js
// code BẠN TỰ VIẾT THÊM — dự án gốc không có
  const [toppings, setToppings] = useState([]);        // danh sách topping lấy từ API
  const [selectedTopping, setSelectedTopping] = useState(null);  // topping đang chọn
```

| Lựa chọn | Vì sao |
|---|---|
| `useState([])` cho `toppings` | Khởi tạo **mảng rỗng** để `.map()` không nổ ở lần vẽ đầu tiên |
| `useState(null)` cho `selectedTopping` | `null` nghĩa là "không thêm topping" — đây là lựa chọn mặc định hợp lý |
| Lưu **cả object** topping chứ không chỉ id | Vì payload cần cả `name` và `price`; lưu object thì khỏi phải đi tìm lại |

### Bước 3 — Gọi API lấy danh sách topping

Thêm `useEffect` này **ngay dưới** `useEffect` nạp sản phẩm (tức sau dòng 84):

```js
// code BẠN TỰ VIẾT THÊM
  useEffect(() => {
    const fetchToppings = async () => {
      try {
        const { data } = await getAllToppings();
        // chỉ hiện topping đang bật (status = 1)
        setToppings(data.filter((item) => item.status === 1));
      } catch (error) {
        setToppings([]);
      }
    };
    fetchToppings();
  }, []);
```

| Chi tiết | Vì sao viết vậy |
|---|---|
| `}, [])` | Danh sách topping **không phụ thuộc sản phẩm** → chỉ tải một lần khi vào trang |
| `try/catch` | Backend chết thì trang chi tiết vẫn hiển thị bình thường, chỉ mất hàng topping |
| `.filter(item => item.status === 1)` | Lọc phía client cho đơn giản. Chuẩn hơn là để backend lọc: `getAllToppings()` gọi thẳng `?status=1` |

### Bước 4 — Vẽ khối chọn topping

Tìm thẻ đóng `</div>` của khối **Đường** (dòng 393 trong file gốc — ngay trước `<div className="border-b border-dashed pb-4 mt-6">` ở dòng 395). Chèn khối sau vào **giữa** hai chỗ đó, tức là topping nằm dưới Đường và trên phần số lượng:

```jsx
{/* ===== KHỐI TOPPING — code BẠN TỰ VIẾT THÊM ===== */}
<div className="flex items-start mt-2">
  <label className="min-w-[80px] font-bold text-sm pt-1">Topping</label>
  <ul className="flex flex-wrap">
    <li>
      <button
        type="button"
        onClick={() => setSelectedTopping(null)}
        className={`block cursor-pointer px-3 py-1 border-2 rounded-[4px] mr-1 mb-1 shadow-sm text-sm transition duration-300 hover:shadow-md ${
          selectedTopping === null
            ? "border-[#D9A953] text-[#D9A953] font-semibold"
            : "border-gray-300 text-gray-500"
        }`}
      >
        Không
      </button>
    </li>

    {toppings.map((topping) => (
      <li key={topping._id}>
        <button
          type="button"
          onClick={() => setSelectedTopping(topping)}
          className={`block cursor-pointer px-3 py-1 border-2 rounded-[4px] mr-1 mb-1 shadow-sm text-sm transition duration-300 hover:shadow-md ${
            selectedTopping?._id === topping._id
              ? "border-[#D9A953] text-[#D9A953] font-semibold"
              : "border-gray-300 text-gray-500"
          }`}
        >
          {topping.name} +{formatCurrency(topping.price)}
        </button>
      </li>
    ))}
  </ul>
</div>
{/* ===== HẾT KHỐI TOPPING ===== */}
```

| Chi tiết | Vì sao **bắt buộc** phải vậy |
|---|---|
| `type="button"` | 🔴 **Quan trọng nhất.** Khối này nằm **trong** thẻ `<form>` (mở ở dòng 219). Thiếu `type="button"`, bấm chọn topping sẽ **submit form và thêm hàng vào giỏ** ngay lập tức |
| `key={topping._id}` | Khoá duy nhất cho mỗi phần tử của `.map()` |
| `selectedTopping?._id === topping._id` | Dấu `?.` phòng khi `selectedTopping` là `null` (lựa chọn "Không") |
| `formatCurrency(topping.price)` | Dùng lại hàm có sẵn ở `yotea-fe/src/utils/index.js` (đã import ở dòng 17) → hiện `5.000 ₫` |
| Dùng `<button>` + `useState` thay vì radio + `register` | Đề bài yêu cầu lưu bằng `useState`; cách này cũng tránh được lỗi `reset()` làm bay lựa chọn (xem Bước 6) |

### Bước 5 — Đưa topping vào payload `addCart`

Sửa hàm `onSubmit` (dòng 39-62). Đây là **phiên bản đầy đủ sau khi sửa** — ba dòng mới được đánh dấu:

```js
// yotea-fe/src/pages/user/ProductDetailPage.js — PHIÊN BẢN SAU KHI BẠN SỬA
  const onSubmit = async ({ ice, sugar }) => {
    // get data product
    const { data: product } = await get(slug);
    const productData = {
      productSlug: product.slug,
      productId: product._id,
      productName: product.name,
      productPrice: product.price,
      productImage: product.image,
    };

    const cartData = {
      id: uuidv4(),
      ...productData,
      quantity,
      ice: +ice,
      sugar: +sugar,
      toppingId: selectedTopping?._id || "",        // ← MỚI
      toppingName: selectedTopping?.name || "",     // ← MỚI
      toppingPrice: selectedTopping?.price || 0,    // ← MỚI
    };

    dispatch(addCart(cartData));
    toast.success(`Thêm ${product.name} vào giỏ hàng thành công`);
    reset();
    setQuantity(1);
    setSelectedTopping(null);                       // ← MỚI: trả nút về "Không"
  };
```

| Trường mới | Kiểu | Khi không chọn topping | Dùng ở đâu về sau |
|---|---|---|---|
| `toppingId` | chuỗi | `""` | `CheckoutPage.js:73` đã bóc sẵn — gửi lên `OrderDetail` |
| `toppingName` | chuỗi | `""` | Hiển thị trong giỏ hàng ([Bài 27](27-gio-hang.md)) mà không phải gọi API |
| `toppingPrice` | số | `0` | Cộng vào tổng tiền |

> 💡 Vì sao dùng `""` và `0` chứ không phải `undefined`? Vì `JSON.stringify` **xoá sạch** các trường có giá trị `undefined`. Redux-persist lưu bằng JSON, nên `toppingId: undefined` sẽ **biến mất** khỏi localStorage, gây khó khi debug.

### Bước 6 — Sửa khoá gộp món trong `cartSlice.js`

Đây là bước dễ bị bỏ quên nhất. Hiện tại `addCart` gộp món trùng theo `productId + ice + sugar`. Nghĩa là:

```
Thêm: Trà sữa, 100% đá, 100% đường, TRÂN CHÂU
Thêm: Trà sữa, 100% đá, 100% đường, PUDDING
→ Bị gộp thành 1 dòng số lượng 2, và topping của lần thêm thứ hai BỐC HƠI!
```

Mở `yotea-fe/src/redux/cartSlice.js` và thêm **một dòng** vào điều kiện `find`:

```js
// yotea-fe/src/redux/cartSlice.js — dòng cuối trong find() là BẠN TỰ THÊM
    addCart({ cart }, { payload: newProduct }) {
      const exitsProduct = cart.find(
        (item) =>
          item.productId === newProduct.productId &&
          item.ice === newProduct.ice &&
          item.sugar === newProduct.sugar &&
          item.toppingId === newProduct.toppingId   // ← MỚI
      );
```

> ⚠️ Đây là lần đầu ta **sửa file có sẵn của dự án** trong mạch thực hành. Hãy `git diff` để thấy rõ mình đã đổi gì; nếu sau này muốn quay lại nguyên bản, `git checkout -- src/redux/cartSlice.js`.

---

## 9. ✅ Kiểm chứng kết quả

### 9.1. Nhìn thấy request lấy topping

Đứng tại `yotea-fe`, chạy:

```bash
npm start
```

Mở `http://localhost:3000/san-pham/<slug-sản-phẩm-của-bạn>`, mở DevTools → **Network** → lọc `Fetch/XHR`. Ngoài các request đã đếm ở mục 7, phải có thêm một dòng:

| Cột | Giá trị mong đợi |
|---|---|
| Name | `toppings/?_sort=createdAt&_order=desc` |
| Status | `200` |
| Request URL | `http://localhost:8080/api/toppings/?_sort=createdAt&_order=desc` |

### 9.2. Nhìn thấy khối topping

Dưới hàng "Đường" phải xuất hiện hàng "Topping" gồm nút **Không** và các nút topping kèm giá:

```
Đá      [0%] [30%] [50%] [70%] [100%]
Đường   [0%] [30%] [50%] [70%] [100%]
Topping [Không] [Trân châu đen +5.000 ₫] [Thạch dừa +6.000 ₫] [Pudding trứng +8.000 ₫]
```

Bấm vào một topping → nút đó đổi sang viền vàng `#D9A953`, các nút khác trở lại xám. **Trang không được reload** — nếu trang nhảy lên đầu hoặc có toast "Thêm ... vào giỏ hàng" thì bạn đã quên `type="button"`.

### 9.3. Kiểm chứng bằng Redux state trong localStorage — bước quan trọng nhất

1. Chọn: 50% đá, 30% đường, topping **Pudding trứng**, số lượng **2** → bấm **Thêm vào giỏ hàng**.
2. DevTools → tab **Application** → **Local Storage** → `http://localhost:3000` → khoá **`persist:root`**.
3. Bạn thấy một chuỗi JSON dài. Copy giá trị của khoá `cart` ra rồi format lại, hoặc nhanh hơn: dán lệnh này vào **Console**:

```js
JSON.parse(JSON.parse(localStorage.getItem("persist:root")).cart).cart
```

Kết quả phải là:

```json
[
  {
    "id": "0d3f9c8e-7b31-4a2f-9c1d-5e6f7a8b9c0d",
    "productSlug": "tra-sua-tran-chau-duong-den",
    "productId": "6650a1f2c4e8b91234abcd01",
    "productName": "Trà sữa trân châu đường đen",
    "productPrice": 35000,
    "productImage": "https://res.cloudinary.com/levantuan/image/upload/...",
    "quantity": 2,
    "ice": 50,
    "sugar": 30,
    "toppingId": "6651a...03",
    "toppingName": "Pudding trứng",
    "toppingPrice": 8000
  }
]
```

**Danh sách phải soát đủ 11 trường** — 8 trường gốc + 3 trường topping bạn vừa thêm.

4. Thêm **cùng sản phẩm, cùng đá/đường** nhưng đổi topping sang **Trân châu đen** → mảng phải có **2 phần tử**, mỗi phần tử một `toppingId` khác nhau. Nếu chỉ thấy 1 phần tử với `quantity` cộng dồn thì Bước 6 chưa được làm.
5. **F5 trang** → chạy lại lệnh Console trên → dữ liệu vẫn còn nguyên. Đó chính là `redux-persist` bạn đã học ở [Bài 21](21-redux-persist.md) đang làm việc.

---

## 10. 🐞 Lỗi thường gặp

| Thông báo / hiện tượng | Nguyên nhân | Cách sửa |
|---|---|---|
| Bấm nút topping thì trang nhảy lên đầu, hiện toast "Thêm ... thành công" | Quên `type="button"` → nút mặc định là submit của `<form>` | Thêm `type="button"` cho **mọi** nút bên trong form |
| `Cannot read properties of undefined (reading 'map')` | `useState()` rỗng thay vì `useState([])` cho `toppings` | Khởi tạo bằng mảng rỗng |
| Hàng Topping hiện ra nhưng **không có nút nào** ngoài "Không" | Mọi topping đang `status: 0` (mặc định là Ẩn) | Postman `PUT /api/toppings/{id}/{userId}` đặt `status: 1`, hoặc tạm bỏ `.filter(...)` |
| `404 Not Found` khi gọi `/api/toppings` | `DB_NAME` trong `api/topping.js` không khớp route backend | Mở `yotea-be/src/routes/topping.js` đối chiếu (`/toppings` hay `/topping`) |
| `Identifier 'get' has already been declared` | Import `get` từ `api/topping` mà file đã có `get` của `api/product` | Đổi tên: `import { getAll as getAllToppings }` |
| Vào trang chi tiết thấy `view` tăng **2** mỗi lần | `StrictMode` chạy `useEffect` hai lần ở chế độ dev | Đây là hành vi đúng của React; muốn hết thì phải sửa theo mục 3.3 |
| Trang chi tiết trắng trơn, không lỗi đỏ | Slug không tồn tại → backend trả `200` kèm `null` → `product` là `null` | Kiểm tra slug; sửa backend trả 404 (mục 2) |
| `toppingId` biến mất khỏi localStorage | Gán `undefined` — `JSON.stringify` xoá trường `undefined` | Dùng `|| ""` và `|| 0` |
| Thêm hai topping khác nhau nhưng giỏ chỉ có 1 dòng | Chưa sửa khoá gộp trong `cartSlice.js` | Làm Bước 6 |
| `Warning: Each child in a list should have a unique "key" prop` | Thiếu `key` khi `.map()` | Thêm `key={topping._id}` |

---

## 11. 📝 Bài tập

**Bài 1.** Hiển thị **tạm tính** ngay phía trên nút "Thêm vào giỏ hàng": `(giá sản phẩm + giá topping) × số lượng`, định dạng tiền Việt.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Không cần thêm state mới — mọi thứ cần thiết đã có trong `product`, `selectedTopping`, `quantity`. Đây là **giá trị dẫn xuất** (derived value), chỉ cần tính lúc render:

```jsx
{/* code BẠN TỰ VIẾT THÊM — đặt ngay trên nút "Thêm vào giỏ hàng" */}
<p className="mt-2 text-xl font-semibold">
  Tạm tính:{" "}
  {formatCurrency(
    ((product?.price || 0) + (selectedTopping?.price || 0)) * quantity
  )}
</p>
```

> 💡 **Nguyên tắc vàng của state:** cái gì **tính ra được** từ state có sẵn thì **đừng** tạo state mới. Nếu bạn tạo `const [totalPrice, setTotalPrice] = useState(0)` rồi phải nhớ `setTotalPrice` ở cả `handleIncrease`, `handleDecrease`, `onChange` và `setSelectedTopping` — chỉ cần quên một chỗ là hiện sai tiền. Nhìn lại dòng 396 của file gốc: dự án từng thử làm kiểu đó rồi **comment lại bỏ đi**.

Lưu ý: con số này mới chỉ đúng ở trang chi tiết. Muốn tổng tiền toàn giỏ cũng cộng topping thì phải sửa công thức `reduce` trong `CartPage.js` và `CheckoutPage.js` — việc đó để dành cho [Bài 27](27-gio-hang.md) và [Bài 28](28-thanh-toan.md).

</details>

**Bài 2.** Viết lại cơ chế **tăng lượt xem** cho đúng: backend tự cộng bằng `$inc`, frontend không gửi con số nào, và ở chế độ dev cũng chỉ tăng đúng 1 cho mỗi lần mở trang.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Backend — thêm controller** (`yotea-be/src/controllers/product.js`, code bạn tự viết thêm):

```js
export const increaseView = async (req, res) => {
  try {
    const product = await Product.findOneAndUpdate(
      { slug: req.params.slug },
      { $inc: { view: 1 } },
      { new: true }
    ).exec();

    if (!product) {
      return res.status(404).json({ message: "Không tìm thấy sản phẩm" });
    }
    res.json({ view: product.view });
  } catch (error) {
    res.status(400).json({ message: "Cập nhật lượt xem thất bại", error });
  }
};
```

**Backend — thêm route** (`yotea-be/src/routes/product.js`). ⚠️ Phải đặt **trước** `router.get("/products/:slug", read)`? Không — đây là `PATCH` còn kia là `GET` nên không đụng nhau; nhưng nó **phải khác** `/products/userUpdate/:id` để không nhập nhằng:

```js
import { clientUpdate, create, increaseView, list, read, remove, update } from "../controllers/product";

router.patch("/products/increaseView/:slug", increaseView);
```

**Frontend — thêm hàm API** (`yotea-fe/src/api/product.js`):

```js
export const increaseView = (slug) => {
  const url = `/${DB_NAME}/increaseView/${slug}`;
  return instance.patch(url);      // KHÔNG gửi body — server tự cộng
};
```

**Frontend — chống StrictMode chạy 2 lần** (`ProductDetailPage.js`):

```js
import { useRef } from "react";   // thêm vào import có sẵn

const viewedRef = useRef(null);

useEffect(() => {
  if (viewedRef.current === slug) return;   // lượt chạy thứ 2 thoát ngay
  viewedRef.current = slug;
  increaseView(slug).catch(() => {});       // lỗi lượt xem không được làm hỏng trang
}, [slug]);
```

Và **xoá** hai dòng 71-72 (`data.view++` và `await clientUpdate(data)`) khỏi effect nạp sản phẩm.

**Vì sao `useRef` chứ không phải `useState`?** Vì đổi `useState` sẽ làm component vẽ lại; `useRef` giữ giá trị qua các lần render mà **không** kích hoạt render — đúng nhu cầu "ghi nhớ đã làm rồi".

Vẫn còn một điểm chưa hoàn hảo: người dùng F5 100 lần vẫn tăng 100 view. Muốn chặt chẽ hơn thì backend phải ghi nhận IP hoặc session — vượt ngoài phạm vi giáo trình, nhưng bạn nên biết đó là cách các trang thật làm.

</details>

**Bài 3.** Ở mục 4 ta đã nói `useEffect` của `ProductRelated` thiếu `cateId` trong mảng dependency. Hãy **tái hiện** lỗi đó (làm nó vỡ thật), rồi sửa.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

**Tái hiện:** tách `setProduct` trong `ProductDetailPage` thành hai lần cập nhật để `id` tới trước `cateId`:

```js
// code TẠM để tái hiện lỗi — nhớ xoá sau khi thử xong
setProduct({ ...data, categoryId: undefined });   // id có, cateId chưa có
setTimeout(() => setProduct({ ...data }), 1000);  // 1 giây sau cateId mới tới
```

Kết quả: khối "Sản phẩm tương tự" **trống vĩnh viễn**. Lý do:

| Lần render | `id` | `cateId` | Effect chạy? | Kết quả |
|---|---|---|---|---|
| 1 | `undefined` | `undefined` | Có (mount) | `cateId &&` chặn → không gọi API |
| 2 | `6650...` | `undefined` | Có (`id` đổi) | `cateId &&` chặn → không gọi API |
| 3 | `6650...` | `6650...` | **KHÔNG** (`id` không đổi) | Mãi mãi trống |

**Sửa:**

```js
    cateId && getProducts();
  }, [id, cateId]);     // ← thêm cateId
```

**Quy tắc rút ra:** mọi giá trị bên ngoài mà thân `useEffect` **đọc tới** đều phải có mặt trong mảng dependency. Bật ESLint rule `react-hooks/exhaustive-deps` để máy nhắc giúp bạn — dự án Yotea vi phạm quy tắc này ở **hơn 10 chỗ**, ta sẽ dọn ở [Bài 34](34-refactor-du-an.md).

</details>

---

## 📌 Tóm tắt

- Trang chi tiết đi theo đường: `URL /san-pham/:slug` → `useParams` → `api/product.js get(slug)` → `GET /api/products/:slug/?_expand=categoryId` → `routes/product.js:9` → `controllers/product.js:73-89` → `Product.findOne({ slug })` → MongoDB, rồi `res.json` → axios → `setProduct` → React vẽ lại.
- Dùng **slug** thay vì `_id` để URL đọc được, tốt cho SEO và không lộ id nội bộ — thành quả của [Bài 08](08-slug-slugify.md).
- **Đá và đường** là hai nhóm radio ẩn (`0/30/50/70/100`, mặc định `100`) do **react-hook-form** quản; **số lượng** do `useState` quản.
- Payload `addCart` có **8 trường**, trong đó `id` sinh bằng **uuid** để phân biệt các *dòng giỏ hàng* — không phải để phân biệt sản phẩm.
- Cơ chế tăng lượt xem của dự án sai ở **ba tầng**: StrictMode đếm 2 lần, endpoint `PATCH /products/userUpdate/:id` **không có auth**, và read-modify-write làm mất lượt xem. Cách đúng gọn hơn nhiều: `{ $inc: { view: 1 } }` ở backend.
- `ProductRelated` lọc **cùng `categoryId`**, loại chính nó bằng **`_id_ne` → `$nin`**, `status=1`, `_limit=4`.
- Mở một trang chi tiết bắn đi **khoảng 16 request** (dev còn gần gấp đôi) — do URL trùng lặp, đếm bằng cách tải hết, và vòng lặp N+1.
- Bạn đã nối Topping vào luồng mua hàng: `api/topping.js` → `useState` → khối chọn → payload `addCart` → `localStorage`.

**Từ khoá tra cứu thêm:** `React StrictMode double render`, `MongoDB $inc atomic update`, `read-modify-write race condition`, `uuid v4`, `controlled input React`, `react-hook-form register radio`, `N+1 query problem`, `useEffect dependency array`

➡️ **Bài tiếp theo:** [27 — Giỏ hàng](27-gio-hang.md) — cái mảng vừa nằm trong `localStorage` sẽ hiện lên thành bảng có nút tăng giảm và tổng tiền; và bạn sẽ thấy vì sao `addCart` gộp theo một khoá còn `removeItemCart` xoá theo khoá khác lại là một quả bom thật sự.
