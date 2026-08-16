# Bài 01 — Tổng quan dự án Yotea & kiến trúc full-stack

> **Phần 0 · Khởi động** — Thời lượng ước tính: **~30 phút**
> ⬅️ Bài trước: [Mục lục](README.md) · Bài sau: [02 — Cài đặt môi trường và chạy dự án lần đầu](02-cai-dat-moi-truong.md) ➡️
> 🏠 [Mục lục](README.md)

---

## 🎯 Sau bài này bạn sẽ

- Giải thích được **full-stack** nghĩa là gì và Yotea gồm những mảnh nào ghép lại.
- Vẽ lại được sơ đồ đường đi của dữ liệu từ lúc người dùng bấm chuột đến lúc chữ hiện ra màn hình.
- Biết **file nào nằm ở đâu** và **thư mục nào chịu trách nhiệm gì** trong cả `yotea-be` lẫn `yotea-fe`.
- Lần theo được một chức năng thật (hiển thị danh sách sản phẩm) qua đủ 7 lớp của hệ thống.
- Đọc được tên 14 nhóm chức năng của dự án và biết mỗi nhóm giải quyết nghiệp vụ gì.

## 📋 Cần chuẩn bị

- Một trình soạn thảo code, khuyến nghị **Visual Studio Code**.
- Đã tải/clone dự án về máy (thư mục gốc chứa 2 thư mục con `yotea-be` và `yotea-fe`).
- Bài này **chưa cần chạy được dự án** — chỉ đọc và mở file. Việc cài đặt để ở [Bài 02](02-cai-dat-moi-truong.md).

---

## 1. "Full-stack" thực ra là gì?

Hãy tưởng tượng một **quán trà sữa ngoài đời**:

| Ngoài đời | Trong lập trình web | Trong dự án này |
|---|---|---|
| Quầy pha chế, menu treo tường, nhân viên order | **Frontend** — thứ khách nhìn thấy và bấm vào | `yotea-fe` |
| Bếp phía sau: nhận phiếu, kiểm tra nguyên liệu, quyết định làm hay từ chối | **Backend** — nơi xử lý nghiệp vụ và bảo mật | `yotea-be` |
| Kho nguyên liệu, sổ ghi đơn hàng | **Database** — nơi lưu dữ liệu lâu dài | **MongoDB** |
| Phiếu order chạy qua lại giữa quầy và bếp | **HTTP + JSON** — giao thức truyền tin | Các request `http://localhost:8080/api/...` |

Khách hàng **không bao giờ được vào bếp**. Muốn gì cũng phải qua nhân viên, viết phiếu,
đưa vào trong. Đó chính là lý do frontend không được nói chuyện thẳng với database —
mọi thứ phải đi qua backend để backend còn kiểm tra "bạn là ai, bạn có quyền không".

> 📖 **Thuật ngữ:** *full-stack* = làm cả "tầng trên" (frontend) lẫn "tầng dưới"
> (backend + database). Học full-stack nghĩa là bạn tự làm được trọn một sản phẩm.

### Ba tiến trình chạy song song

Khi dự án Yotea hoạt động, trên máy bạn có **ba chương trình chạy cùng lúc**:

```
┌───────────────────────────┐         ┌───────────────────────────┐         ┌──────────────────┐
│      TRÌNH DUYỆT          │         │      NODE.JS              │         │    MONGODB       │
│  ─────────────────────    │         │  ─────────────────────    │         │  ──────────────  │
│      yotea-fe             │ ──────▶ │      yotea-be             │ ──────▶ │   database       │
│  React 18 · Redux         │  HTTP   │  Express · Mongoose       │  driver │   "yotea"        │
│                           │  JSON   │                           │         │                  │
│  localhost:3000           │ ◀────── │  localhost:8080           │ ◀────── │  localhost:27017 │
└───────────────────────────┘         └───────────────────────────┘         └──────────────────┘
        Giao diện                          Nghiệp vụ + bảo mật                    Lưu trữ
     (chạy trên máy khách)                  (chạy trên server)                 (chạy trên server)
```

Thiếu một trong ba, web sẽ hỏng theo kiểu khác nhau:

| Tắt cái gì | Triệu chứng bạn sẽ thấy |
|---|---|
| Tắt MongoDB | Backend in lỗi `MongooseServerSelectionError`, mọi API trả lỗi |
| Tắt backend | Web mở được nhưng trắng trơn, Console báo `ERR_CONNECTION_REFUSED` |
| Tắt frontend | `localhost:3000` không vào được, nhưng gọi API bằng Postman vẫn chạy |

---

## 2. Soi cấu trúc thư mục thật

### 2.1. Nhìn từ trên xuống

```
PROJECT-SELL-MILK-TEA/
├── yotea-be/          ← Backend: API + database
│   ├── .babelrc
│   ├── package.json
│   └── src/
│       ├── app.js           ← điểm khởi động của backend
│       ├── routes/          ← 14 file: khai báo đường dẫn URL
│       ├── controllers/     ← 14 file: xử lý logic cho từng đường dẫn
│       ├── models/          ← 14 file: mô tả cấu trúc dữ liệu trong MongoDB
│       └── middlewares/     ← 1 file: kiểm tra đăng nhập & phân quyền
│
└── yotea-fe/          ← Frontend: giao diện người dùng
    ├── package.json
    ├── tailwind.config.js
    ├── public/
    └── src/
        ├── index.js         ← điểm khởi động của frontend
        ├── App.js           ← khai báo toàn bộ đường dẫn (route) của web
        ├── api/             ← 15 file: các hàm gọi sang backend
        ├── redux/           ← 13 file: kho dữ liệu dùng chung
        ├── pages/           ← các trang: user, admin, auth, layouts
        ├── components/      ← các mảnh giao diện dùng lại nhiều nơi
        └── utils/           ← hàm tiện ích: định dạng tiền, ngày, upload ảnh
```

### 2.2. Backend — mô hình MVC

Backend Yotea theo mô hình **MVC** (Model – View – Controller), nhưng vì đây là API
thuần nên **không có View**; thứ trả về cho client là JSON. Còn lại đủ ba lớp:

| Thư mục | Vai trò | Ví von |
|---|---|---|
| `routes/` | Khai báo **URL nào** ứng với **hàm nào**, và request đó phải đi qua **chốt kiểm tra nào** | Bảng chỉ dẫn ở cửa: "Đơn thêm sản phẩm → đưa cho bếp trưởng, phải có thẻ nhân viên" |
| `controllers/` | Chứa **logic xử lý**: đọc dữ liệu client gửi lên, gọi Model, trả kết quả | Đầu bếp thật sự làm việc |
| `models/` | Mô tả **cấu trúc dữ liệu** và nói chuyện với MongoDB | Công thức pha chế + kho nguyên liệu |
| `middlewares/` | Các **chốt kiểm tra** chạy trước controller | Bảo vệ kiểm tra thẻ ra vào |

Điểm quan trọng cần nhớ ngay từ bây giờ: **ba thư mục `routes`, `controllers`, `models`
có các file trùng tên nhau**. Ví dụ với sản phẩm, bạn sẽ thấy bộ ba:

```
yotea-be/src/routes/product.js       ← URL nào
yotea-be/src/controllers/product.js  ← làm gì
yotea-be/src/models/product.js       ← dữ liệu trông ra sao
```

Cứ nắm quy tắc này, muốn sửa chức năng nào bạn biết ngay phải mở 3 file nào.

### 2.3. Trái tim của backend: `app.js`

`yotea-be/src/app.js:22-42`

```js
const app = express();

// middleware
app.use(express.json());
app.use(cors());
app.use(morgan("tiny"));

app.use("/api", categoryRouter);
app.use("/api", productRouter);
app.use("/api", sliderRouter);
app.use("/api", storeRouter);
app.use("/api", cateNewsRouter);
app.use("/api", newsRouter);
app.use("/api", contactRouter);
app.use("/api", userRouter);
app.use("/api", authRouter);
app.use("/api", orderRouter);
app.use("/api", orderDetailRouter);
app.use("/api", cmtRouter);
app.use("/api", ratingRouter);
app.use("/api", favoritesRouter);
```

**Đọc từng dòng:**

| Dòng | Code | Ý nghĩa |
|---|---|---|
| 22 | `const app = express()` | Tạo ra ứng dụng web. Từ đây `app` chính là server. |
| 25 | `app.use(express.json())` | Bật khả năng đọc dữ liệu JSON client gửi lên. Không có dòng này, `req.body` sẽ luôn là `undefined`. |
| 26 | `app.use(cors())` | Cho phép trình duyệt ở cổng 3000 gọi sang cổng 8080. Không có dòng này trình duyệt sẽ **chặn** mọi request. |
| 27 | `app.use(morgan("tiny"))` | In log mỗi request ra terminal, cực hữu ích khi debug. |
| 29–42 | `app.use("/api", xxxRouter)` | Gắn 14 nhóm route vào server, tất cả đều có tiền tố `/api`. |

Chính vì dòng 29–42 mà mọi địa chỉ API của dự án đều bắt đầu bằng `/api`:
`http://localhost:8080/api/products`, `http://localhost:8080/api/signin`…

Ngay dưới đó là phần kết nối database và bật server:

`yotea-be/src/app.js:44-52`

```js
// connect db
mongoose
  .connect("mongodb://localhost:27017/yotea")
  .then(() => console.log("Connected to MongoDB"))
  .catch((error) => console.log(error));

// connect
const PORT = process.env.PORT || 8080;
app.listen(PORT, () => console.log(`App is running on port: ${PORT}`));
```

> ⚠️ **Chỗ này dự án làm chưa chuẩn:** địa chỉ database bị **viết cứng** (hardcode) ngay
> trong code. Chuẩn công nghiệp là đưa vào biến môi trường (file `.env`) để mỗi môi
> trường — máy bạn, máy đồng nghiệp, server thật — dùng một địa chỉ khác nhau mà không
> phải sửa code. Ta sẽ sửa đúng chỗ này ở [Bài 34](34-refactor-du-an.md).

### 2.4. Trái tim của frontend: `index.js` và `App.js`

`yotea-fe/src/index.js:10-19`

```js
const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <Provider store={store}>
      <PersistGate loading={null} persistor={persistor}>
        <App />
      </PersistGate>
    </Provider>
  </React.StrictMode>
);
```

Đọc từ ngoài vào trong như bóc củ hành:

| Lớp | Nhiệm vụ |
|---|---|
| `createRoot(document.getElementById("root"))` | Tìm thẻ `<div id="root">` trong `public/index.html` — toàn bộ web sẽ được vẽ vào đây |
| `<React.StrictMode>` | Chế độ kiểm tra nghiêm ngặt khi lập trình (khiến một số hàm chạy 2 lần — đừng hoảng) |
| `<Provider store={store}>` | Đưa **kho dữ liệu Redux** xuống cho mọi component bên trong dùng chung |
| `<PersistGate>` | Chờ đọc xong dữ liệu đã lưu trong trình duyệt (giỏ hàng, phiên đăng nhập) rồi mới vẽ giao diện |
| `<App />` | Ứng dụng thật sự — chứa toàn bộ bảng đường dẫn |

Còn `App.js` là **bản đồ đường đi của cả website**: mỗi địa chỉ trên thanh URL ứng với
một component nào. Ví dụ một mẩu trong đó:

`yotea-fe/src/App.js:64-84`

```js
const router = createBrowserRouter([
  {
    path: "",
    element: <WebsiteLayout />,
    children: [
      {
        path: "",
        element: <HomePage />,
      },
      {
        path: "gioi-thieu",
        element: <AboutPage />,
      },
      {
        path: "thuc-don",
        element: <ProductPage />,
      },
```

Nghĩa là: vào `localhost:3000/thuc-don` thì React sẽ vẽ `WebsiteLayout` (header + footer),
và **bên trong nó** vẽ `ProductPage`. Cơ chế layout lồng nhau này sẽ được mổ kỹ ở
[Bài 15](15-routing-v6.md).

---

## 3. 🛠️ Tự tay làm — lần theo một chức năng qua 7 lớp

> Mục tiêu phần này: cuối phần bạn sẽ tự chỉ được ngón tay vào **7 file** và nói
> "dữ liệu đi từ đây, qua đây, tới đây". Đây là kỹ năng quan trọng nhất khi nhận
> bàn giao một dự án lạ.

Chức năng ta chọn: **trang thực đơn hiển thị danh sách sản phẩm**.

### Bước 1 — Xác định URL người dùng truy cập

Người dùng vào `http://localhost:3000/thuc-don`.

### Bước 2 — Tìm xem URL đó vẽ component nào

Mở `yotea-fe/src/App.js`, dùng `Ctrl + F` tìm chuỗi `thuc-don`. Bạn sẽ thấy:

```js
{
  path: "thuc-don",
  element: <ProductPage />,
},
```

👉 **Lớp 1:** `yotea-fe/src/App.js` → chỉ tới `ProductPage`.

### Bước 3 — Mở component đó

Mở `yotea-fe/src/pages/user/ProductPage.js`. Điều bất ngờ: trang này **không hề tự gọi
API**. Nó chỉ dựng khung rồi giao việc lấy dữ liệu cho component con:

`yotea-fe/src/pages/user/ProductPage.js:3`

```js
import { getAll } from "../../api/product";
```

`yotea-fe/src/pages/user/ProductPage.js:40-44`

```jsx
<ProductContent
  getProducts={getAll}
  page={Number(page) || 1}
  url="thuc-don"
/>
```

Để ý dòng `getProducts={getAll}` — trang đang **truyền cả một hàm** xuống làm props.

👉 **Lớp 2:** trang chọn "cách lấy dữ liệu" rồi đưa cho component con.

### Bước 4 — Component con mới là nơi gọi API

Mở `yotea-fe/src/components/user/ProductContent.js`:

`yotea-fe/src/components/user/ProductContent.js:16`

```js
const ProductContent = ({ url, page, getProducts, parameter }) => {
```

`yotea-fe/src/components/user/ProductContent.js:30-37`

```js
  useEffect(() => {
    const getData = async () => {
      const { data } = await getProducts(
        start,
        limit,
        filter.sort,
        filter.order,
        parameter
      );
```

Nó gọi chính cái hàm được truyền vào, rồi cất kết quả bằng `useState` của riêng nó.

> 💡 **Kỹ thuật hay đáng học:** cùng một `ProductContent` được **ba trang khác nhau**
> dùng lại — `ProductPage` (tất cả sản phẩm), `ProductByCate` (theo danh mục),
> `ProductSearchPage` (theo từ khoá). Mỗi trang chỉ việc truyền vào một hàm lấy dữ liệu
> khác nhau. Đây gọi là **dependency injection** — thay vì viết ba component gần giống
> nhau, ta viết một cái và "tiêm" phần khác biệt vào từ ngoài.

👉 **Lớp 3:** component con gọi xuống tầng api.

> ⚠️ **Đừng nhầm:** dự án **có** file `yotea-fe/src/redux/productSlice.js` với đủ
> `createAsyncThunk` để lấy sản phẩm, nhưng **không trang nào dùng tới nó** — nó đã bị
> thay thế bởi cách gọi trực tiếp ở trên và bởi RTK Query trong trang quản trị. Đây là
> **code chết**. Bài học rút ra: đừng thấy một file trong `redux/` mà vội tin rằng dữ
> liệu chạy qua đó — hãy dùng `Ctrl + Shift + F` tìm xem ai thật sự import nó.

### Bước 5 — Xem tầng api tạo ra request gì

Mở `yotea-fe/src/api/product.js`:

`yotea-fe/src/api/product.js:9-18`

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

Kết hợp với `yotea-fe/src/api/instance.js` (đặt `baseURL` là `http://localhost:8080/api`),
request thật sự bay đi là:

```
GET http://localhost:8080/api/products/?_expand=categoryId&_sort=createdAt&_order=desc
```

👉 **Lớp 4:** trình duyệt bắn HTTP request sang backend. **Đây chính là ranh giới
giữa frontend và backend.**

### Bước 6 — Backend nhận request

Mở `yotea-be/src/routes/product.js`:

`yotea-be/src/routes/product.js:8-13`

```js
router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);
router.get("/products/:slug", read);
router.get("/products", list);
router.put("/products/:id/:userId", requireSignin, isAuth, isAdmin, update);
router.patch("/products/userUpdate/:id", clientUpdate);
router.delete("/products/:id/:userId", requireSignin, isAuth, isAdmin, remove);
```

`GET /products` khớp dòng thứ ba → chạy hàm `list`.

👉 **Lớp 5:** router chọn đúng controller.

### Bước 7 — Controller truy vấn database

Mở `yotea-be/src/controllers/product.js`, tìm `export const list`. Cuối hàm là:

```js
const products = await Product.find(filter)
  .select("-__v")
  .populate(populate)
  .skip(start)
  .limit(limit)
  .sort(sortOpt)
  .exec();
res.json(products);
```

`Product` chính là model, `import` từ `../models/product`.

👉 **Lớp 6:** controller gọi model.
👉 **Lớp 7:** model (`yotea-be/src/models/product.js`) nói chuyện với MongoDB, lấy dữ liệu
về, controller đóng gói thành JSON `res.json(products)` và trả ngược lại đúng con
đường vừa đi.

### Bước 8 — Vẽ lại sơ đồ bằng chính tay bạn

Lấy giấy bút (hoặc mở file text), viết lại chuỗi 7 lớp trên **mà không nhìn tài liệu**:

```
URL /thuc-don
  → App.js                    (bản đồ route)
  → ProductPage.js            (trang, truyền hàm getAll xuống)
  → ProductContent.js         (component con, gọi API trong useEffect)
  → api/product.js            (dựng URL + axios)
        ═══════ HTTP ═══════▶
  → routes/product.js         (khớp GET /products → list)
  → controllers/product.js    (dựng filter, truy vấn)
  → models/product.js         (Mongoose Model)
  → MongoDB
```

Viết được trơn tru nghĩa là bạn đã nắm xương sống của dự án.

> 💡 **Một điều quan trọng cần nhớ ngay:** phía khách hàng, dự án **hầu như không dùng
> Redux để lấy dữ liệu**. Sản phẩm, tin tức, cửa hàng… đều được gọi thẳng bằng axios rồi
> giữ trong `useState` của component. Redux chỉ được dùng cho **3 thứ**: đăng nhập
> (`authSlice`), giỏ hàng (`cartSlice`) và yêu thích (`wishlistSlice`) — đúng những dữ
> liệu cần dùng chung ở nhiều nơi và cần sống sót qua F5. Nắm được ranh giới này, bạn
> sẽ không mất công đi tìm dữ liệu ở chỗ nó không hề có.

---

## 4. ✅ Kiểm chứng kết quả

Bạn hoàn thành bài này khi trả lời được **không cần mở tài liệu**:

| Câu hỏi | Đáp án đúng |
|---|---|
| Muốn sửa **giao diện** trang thực đơn thì vào thư mục nào? | `yotea-fe/src/pages/user/` |
| Muốn thêm một **API mới** thì phải sửa ít nhất mấy file, những file nào? | 3 file: `routes/`, `controllers/`, `models/` (thêm 1 dòng `app.use` nếu là router mới) |
| Frontend gọi backend qua địa chỉ nào? | `http://localhost:8080/api/...`, khai báo trong `yotea-fe/src/api/instance.js` |
| Vì sao mọi API đều có `/api` ở đầu? | Vì `app.js` mount router bằng `app.use("/api", xxxRouter)` |
| `middlewares/checkAuth.js` dùng để làm gì? | Kiểm tra đã đăng nhập chưa và có phải admin không, chạy **trước** controller |

Và mở được đủ 7 file trong phần "Tự tay làm" mà không phải tìm mò.

---

## 5. 🐞 Lỗi thường gặp

| Vấn đề | Nguyên nhân | Cách xử lý |
|---|---|---|
| Mở `yotea-be/src/app.js` thấy `import` mà nhớ Node.js phải dùng `require` | Dự án dùng **Babel** để dịch cú pháp ES6 sang cú pháp Node hiểu được (xem file `.babelrc`) | Bình thường, cứ viết `import` theo dự án |
| Không tìm thấy thư mục `views/` dù được nói là MVC | Đây là **API thuần**, không render HTML, nên không có View | Đúng như thiết kế, không thiếu gì cả |
| Tìm mãi không thấy file `.env` | Dự án **chưa** dùng biến môi trường, mọi cấu hình đều hardcode | Ghi nhớ để sửa ở [Bài 34](34-refactor-du-an.md) |
| Thấy `node_modules` nặng vài trăm MB và cũng nằm trong git | Thư viện tải về, lẽ ra phải bỏ qua bằng `.gitignore` | Không đụng vào thư mục này, không bao giờ sửa file trong đó |
| Thấy `yotea-fe/src/api/product.js` vừa có `getAll` vừa có `productApi` | Dự án dùng **song song 2 cách** lấy dữ liệu: axios thủ công và RTK Query | Sẽ được giải thích ở [Bài 22](22-rtk-query.md) |

---

## 6. 📝 Bài tập

**Bài 1.** Dự án có 14 nhóm chức năng ở backend. Hãy mở thư mục `yotea-be/src/routes/`,
liệt kê đủ 14 file và ghi cạnh mỗi file một câu mô tả nghiệp vụ mà nó phụ trách.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

| File route | Nghiệp vụ |
|---|---|
| `auth.js` | Đăng ký, đăng nhập, kiểm tra mật khẩu |
| `users.js` | Quản lý tài khoản người dùng (admin CRUD) |
| `category.js` | Danh mục sản phẩm (Trà sữa, Trà trái cây…) |
| `product.js` | Sản phẩm — trung tâm của web bán hàng |
| `slider.js` | Ảnh banner chạy ở đầu trang chủ |
| `store.js` | Hệ thống cửa hàng / chi nhánh |
| `cateNews.js` | Danh mục bài viết |
| `news.js` | Bài viết / tin tức |
| `contact.js` | Liên hệ khách gửi về |
| `order.js` | Đơn hàng (thông tin người nhận, tổng tiền, trạng thái) |
| `orderDetail.js` | Chi tiết từng dòng sản phẩm trong đơn |
| `comment.js` | Bình luận dưới sản phẩm |
| `rating.js` | Đánh giá sao |
| `favoritesProduct.js` | Sản phẩm yêu thích |

**Điểm cộng nếu bạn nhận ra:** trong `yotea-be/src/models/` có **14 model** nhưng
`routes/` chỉ có **14 file cho 14 nhóm khác**. Cụ thể `models/voucher.js` tồn tại nhưng
**không có** `routes/voucher.js` lẫn `controllers/voucher.js` — nghĩa là chức năng mã
giảm giá mới làm được phần dữ liệu, chưa có API. Đây chính là đề tài của
[Bài 35 — Đồ án cuối khoá](35-do-an-cuoi-voucher.md).

</details>

**Bài 2.** Lặp lại bài tập "lần theo các lớp" nhưng với chức năng **danh sách tin tức**
(URL `/tin-tuc`). Ghi ra đủ tên các file bạn phải mở.

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

```
URL /tin-tuc
  1. yotea-fe/src/App.js                      → tìm path "tin-tuc" → <NewsPage />
  2. yotea-fe/src/pages/user/NewsPage.js      → truyền getNews={getAll} xuống
  3. yotea-fe/src/components/user/NewsContent.js  → gọi API trong useEffect
  4. yotea-fe/src/api/news.js                 → sinh URL GET /api/news?...
     ═════════════════ HTTP ═════════════════
  5. yotea-be/src/routes/news.js              → router.get("/news", list)
  6. yotea-be/src/controllers/news.js         → export const list
  7. yotea-be/src/models/news.js              → model News → MongoDB
```

Bạn có nhận ra **cấu trúc lặp lại y hệt** với sản phẩm không — kể cả mẹo truyền hàm
`getAll` xuống component con qua props? Đó không phải trùng hợp: đây là điểm mạnh của
việc code theo quy ước — học một lần, áp dụng cho cả 14 chức năng.

</details>

**Bài 3.** (nâng cao) Mở `yotea-be/src/routes/product.js` và `yotea-be/src/routes/news.js`.
Route nào bắt buộc đăng nhập, route nào ai cũng gọi được? Dựa vào đâu mà bạn biết?

<details>
<summary>💡 Xem gợi ý & lời giải</summary>

Nhìn vào **các tham số đứng giữa** đường dẫn và hàm xử lý:

```js
router.get("/products", list);                                              // ai cũng gọi được
router.post("/products/:userId", requireSignin, isAuth, isAdmin, create);   // phải đăng nhập + là admin
```

`requireSignin`, `isAuth`, `isAdmin` chính là **middleware** — các chốt kiểm tra chạy
tuần tự trước `create`. Chốt nào không cho qua thì request dừng ngay tại đó, `create`
không bao giờ được chạy.

Quy luật chung của dự án: **đọc (GET) thì tự do, ghi (POST/PUT/DELETE) thì phải là admin.**
Riêng `PATCH /products/userUpdate/:id` (tăng lượt xem, lượt thích) là ngoại lệ có chủ ý —
khách vãng lai cũng phải làm được. Chi tiết ở [Bài 12](12-phan-quyen-middleware.md).

</details>

---

## 📌 Tóm tắt

- Yotea gồm **3 tiến trình chạy song song**: React (:3000) → Express (:8080) → MongoDB (:27017).
- Frontend **không bao giờ** nói chuyện thẳng với database; mọi thứ đi qua backend bằng HTTP + JSON.
- Backend theo mô hình **MVC không View**: `routes` (URL nào) → `controllers` (làm gì) → `models` (dữ liệu ra sao), cộng thêm `middlewares` làm chốt kiểm tra.
- Ba thư mục `routes` / `controllers` / `models` có **file trùng tên nhau** — nhớ quy tắc này là biết ngay phải mở file nào khi sửa chức năng.
- Frontend chia thành `pages` (trang) · `components` (mảnh dùng lại) · `api` (gọi backend) · `redux` (dữ liệu dùng chung) · `utils` (tiện ích).
- Mọi API đều có tiền tố `/api` vì `app.js` mount router bằng `app.use("/api", xxxRouter)`.
- Kỹ năng cần luyện: **lần theo một chức năng qua đủ 7 lớp** — dùng được cho mọi dự án sau này.

**Từ khoá tra cứu thêm:** `MERN stack`, `client-server architecture`, `REST API`, `MVC pattern`, `separation of concerns`

➡️ **Bài tiếp theo:** [02 — Cài đặt môi trường và chạy dự án lần đầu](02-cai-dat-moi-truong.md) — cài Node, MongoDB và nhìn thấy trang chủ Yotea hiện ra trên máy bạn.
